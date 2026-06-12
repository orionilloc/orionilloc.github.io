---
layout: post
title: "Ansible Linux Sandbox - Part 3"
date: 2026-06-08 11:00:00 -0500
categories: [Homelab]
tags: [aws, ansible, terraform, linux, ssm, roles, playbooks, openscap]
media_subpath: /assets/img/AnsibleSandbox3/
#image: AnsibleSandbox3FrontImage.png
---

By the end of Part 2, I had a functional lab: six nodes, all SSM-connected, no public IPs, with Ansible running over S3-backed session transport. The `inventory.ini` looked clean enough, but didn't stand when needing to terminate and replace resources through terraform.

Every EC2 instance gets a new instance ID on recreation. That ID is what the SSM connection plugin uses as `ansible_host`. So after a  `terraform apply` with some sort of replace argument tacked on, some of the entries in `inventory.ini` could be inaccurate. I was manually copying a few instance IDs out of the AWS CLI output and pasting them back into a flat file. Definitely not a great idea long-term!

```ini
[al2023]
node-al2023 ansible_host=i-0a1b2c3d4e5f60001 ansible_user=ec2-user
```

That `i-0a1b2c3d4e5f60001` is stale the moment the instance is replaced. This is the kind of thing that feels manageable until it isn't, and it's exactly the problem dynamic inventory exists to solve.

## Switching to the `aws_ec2` Plugin

The `amazon.aws` collection ships an inventory plugin called `aws_ec2` that queries the AWS API directly and builds your inventory at runtime. There's no static file and no manual updates. When Terraform recreates a node, the next `ansible-inventory --list` reflects it automatically.

The plugin is configured via a YAML file. 

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2

regions:
  - us-east-1

filters:
  tag:AnsibleGroup:
    - al2023
    - debian
    - ubuntu
    - fedora
    - rhel
    - opensuse

groups:
  linux: "'AnsibleGroup' in (tags | default({}))"

keyed_groups:
  - key: tags.AnsibleGroup
    prefix: ""
    separator: ""

compose:
  ansible_host: instance_id
  ansible_connection: "'aws_ssm'"
  ansible_aws_ssm_region: "'us-east-1'"
  ansible_aws_ssm_bucket_name: "'ansible-linux-sandbox-terraform-state'"
  ansible_aws_ssm_plugin_prefix: "'ansible-transport/'"
  ansible_remote_tmp: "'/tmp/.ansible/tmp'"
  ansible_shell_type: "'sh'"
  ansible_python_interpreter: "'/usr/bin/python3'"
  ansible_user: >-
    {'al2023': 'ec2-user', 'debian': 'admin', 'ubuntu': 'ubuntu',
     'fedora': 'fedora', 'rhel': 'ec2-user', 'opensuse': 'ec2-user'}[tags.AnsibleGroup]

hostnames:
  - tag:Name
  - instance-id
```

A few things worth explaining here.

The `filters` block limits the query to instances with an `AnsibleGroup` tag matching one of the six known values. This is more explicit than filtering by project tag: only the nodes that are supposed to be in the inventory are pulled in.

The `keyed_groups` block is doing what the group headers in `inventory.ini` used to do manually. Each instance has an `AnsibleGroup` tag set in `main.tf`. The plugin reads that tag and places the instance into a group of the same name. A node tagged `AnsibleGroup: al2023` lands in the `al2023` group automatically.

```hcl
# main.tf (excerpt)
tags = { Name = "${var.project_name}-AL2023-Managed", AnsibleGroup = "al2023" }
```

The `groups` block creates a parent `linux` group containing any instance that has the `AnsibleGroup` tag present. This mirrors the `[linux:children]` block from the static inventory.

The `compose` block is doing more work here than it might appear. In v2, the SSM connection variables (`ansible_connection`, `ansible_aws_ssm_region`, `ansible_aws_ssm_bucket_name`, and so on) lived in the `[linux:vars]` block of the static inventory. With dynamic inventory there is no equivalent, so every connection variable moves into `compose` and gets evaluated per host at runtime.

`ansible_user` uses a dictionary lookup against `tags.AnsibleGroup` rather than a chain of conditionals. The tag value is the key, the default user for that distribution is the value. It is more readable and adding a new distribution means adding one entry to the dictionary, not another branch to a conditional chain.

One cosmetic issue that showed up early: without the `hostnames` block, hosts were displaying as raw private IP addresses in Ansible output.

```
changed: [ip-10-0-2-215.ec2.internal]
```

Not useful when you're looking at six nodes and trying to figure out which one just failed. Adding `hostnames` to the plugin config fixes this by using the `Name` tag value instead. This one took some digging through the plugin documentation to land on.

```yaml
hostnames:
  - tag:Name
  - instance-id
```

Output reads as `ansible-lab-AL2023-Managed` instead of a private IP.

## Verifying the Plugin

Before running any playbooks, verify that the plugin is building the inventory correctly.

```bash
ansible-inventory -i inventory/aws_ec2.yml --graph
```

Expected output shows the group structure with all six hosts placed correctly. If a node is missing, the first thing to check is whether the SSM agent is running and whether the instance tags match the filter.

```bash
ansible-inventory -i inventory/aws_ec2.yml --list
```

The `--list` flag dumps the full JSON inventory including all host variables. This is where you verify that `ansible_host` is resolving to an instance ID and that `ansible_user` is set correctly per host.

## Role Structure

With dynamic inventory working, the next step was building out the Ansible roles. The v2 work used flat playbooks. Roles impose structure: tasks, defaults, handlers, and templates each live in their own directory, and the role is invoked by name from a playbook rather than having tasks inlined.

```
roles/
  universal-baseline/
    defaults/
      main.yml
    tasks/
      main.yml
      packages.yml
      system.yml
      users.yml
      updates.yml
    templates/
      motd.j2
    files/
      .vimrc
  os-hardening/
    defaults/
      main.yml
    handlers/
      main.yml
    tasks/
      main.yml
      sysctl.yml
      sshd.yml
      filesystem.yml
```

`tasks/main.yml` in each role is the orchestrator. It does not contain task logic directly: it imports the concern-based task files.

```yaml
# roles/universal-baseline/tasks/main.yml
- name: Apply package configuration
  ansible.builtin.import_tasks: packages.yml

- name: Apply system configuration
  ansible.builtin.import_tasks: system.yml

- name: Apply user configuration
  ansible.builtin.import_tasks: users.yml

- name: Apply system updates
  ansible.builtin.import_tasks: updates.yml
```

`defaults/main.yml` is data only. No task syntax, no conditionals. Just variable definitions that tasks reference.

```yaml
# roles/universal-baseline/defaults/main.yml
admin_group_map:
  RedHat: wheel
  Debian: sudo
  Suse: root

baseline_packages:
  - vim
  - git
  - wget

timezone: "America/New_York"

baseline_users:
  - name: harrycallahan
    uid: 1051
    shell: /bin/bash
    state: present
    is_admin: true

  - name: lab_user1
    uid: 1052
    groups: lab_users
    shell: /bin/bash
    state: present
    is_admin: false

baseline_groups:
  - name: lab_users
    gid: 2001
    state: present
```

The roles are invoked from two separate playbooks.

```yaml
# playbooks/site.yml
- name: Apply universal baseline to all Linux nodes
  hosts: linux
  become: true
  roles:
    - role: universal-baseline
```

```yaml
# playbooks/harden.yml
- name: Apply OS hardening to all Linux nodes
  hosts: linux
  become: true
  roles:
    - role: os-hardening
```

## Cross-Distro Debugging

This is where the actual work happened.

### The `ansible_user` Problem

The first run after switching to dynamic inventory failed immediately across every node.

```
fatal: [ip-10-0-2-247.ec2.internal]: FAILED! => {
  "msg": "The task includes an option with an undefined variable.
  The error was: 'ansible_user' is undefined."
}
```

In the static inventory, `ansible_user` was defined in the `[linux:vars]` block and inherited by every child group. The dynamic inventory plugin does not replicate that automatically. The `compose` block in `aws_ec2.yml` is what fills that gap, as shown above. Once that was in place, the failures went away.

### AL2023 and the curl Conflict

AL2023 ships with `curl-minimal` by default. Installing the full `curl` package conflicts with it at the package manager level.

```
package curl-minimal-8.3.0-1.amzn2023.0.2.x86_64 from amazonlinux
conflicts with curl provided by curl-8.17.0-1.amzn2023.0.1.x86_64
```

The packages task ended up fully distro-specific rather than using the abstract `package` module. The abstract module works well when package names are consistent across distributions, but this lab has enough per-distro variation: AL2023 needs `curl-minimal` instead of `curl`, Fedora requires `dnf5` rather than `dnf`, and the Debian family needs an explicit cache update before any package installation. A single generic task couldn't handle all of that cleanly.

```yaml
{% raw %}
# roles/universal-baseline/tasks/packages.yml
- name: Update apt cache on Debian family
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 0
  when: ansible_os_family == "Debian"

- name: Ensure baseline packages on Debian family
  ansible.builtin.apt:
    name: "{{ baseline_packages + ['curl'] }}"
    state: present
  when: ansible_os_family == "Debian"

- name: Ensure baseline packages on Fedora
  ansible.builtin.dnf5:
    name: "{{ baseline_packages + ['curl', 'vim-enhanced'] }}"
    state: present
  when: ansible_distribution == "Fedora"

- name: Ensure baseline packages on RedHat family
  ansible.builtin.dnf:
    name: "{{ baseline_packages + ['curl', 'vim-enhanced'] }}"
    state: present
  when: ansible_os_family == "RedHat" and ansible_distribution != "Amazon" and ansible_distribution != "Fedora"

- name: Ensure baseline packages on AL2023
  ansible.builtin.dnf:
    name: "{{ baseline_packages + ['curl-minimal', 'vim-enhanced'] }}"
    state: present
  when: ansible_distribution == "Amazon"

- name: Ensure baseline packages on SUSE family
  community.general.zypper:
    name: "{{ baseline_packages + ['curl'] }}"
    state: present
  when: ansible_os_family == "Suse"
{% endraw %}
```

This is also where the `ansible_distribution` vs `ansible_os_family` distinction matters in practice. AL2023's `ansible_os_family` resolves to `RedHat`. Without the explicit `ansible_distribution != "Amazon"` guard on the RedHat task, AL2023 would match it and try to install `curl` instead of `curl-minimal`, hitting the conflict. `ansible_distribution` resolves to `Amazon` for AL2023 specifically, which is what makes the per-distro targeting work correctly.

## The `admin_group_map` Pattern

Different distributions use different groups to grant sudo access: `wheel` on Red Hat derived systems, `sudo` on Debian and Ubuntu, and `root` on openSUSE. The approach that ended up in the role uses a dictionary in `defaults/main.yml` and a ternary expression in the task.

```yaml
# defaults/main.yml (excerpt)
admin_group_map:
  RedHat: wheel
  Debian: sudo
  Suse: root
```

```yaml
{% raw %}
# tasks/users.yml
- name: Ensure baseline groups exist
  ansible.builtin.group:
    name: "{{ item.name }}"
    state: "{{ item.state }}"
  loop: "{{ baseline_groups }}"

- name: Ensure baseline users are present
  ansible.builtin.user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    groups: "{{ admin_group_map[ansible_os_family] if item.is_admin else 'lab_users' }}"
    shell: "{{ item.shell }}"
    append: true
    state: "{{ item.state }}"
  loop: "{{ baseline_users }}"
{% endraw %}
```

The `ansible_os_family` value acts as the dictionary key. Admin users get the correct sudo group for their distribution. Non-admin users land in `lab_users`. Adding a new distribution means adding one entry to `admin_group_map` in `defaults/main.yml`, not modifying task logic.

This is a case where I needed help arriving at the pattern. The initial approach used chained conditionals and worked, but it was inelegant and hard to extend. Claude suggested the dictionary lookup approach.

## The os-hardening Role

The `os-hardening` role applies a subset of CIS benchmark controls that are meaningful in this environment. Before writing any tasks, I spent time with OpenSCAP and the [ComplianceAsCode content repository](https://github.com/complianceascode/content) to understand what these controls actually do at the system level. The benchmark content is machine-readable: you can trace a control from its XCCDF definition through to the remediation script and understand what is being changed and why. I've done a basic CIS Level 1 audit for an organization before and the gap between "controls on paper" and "controls verified on running systems" is not small. Having that reference made it easier to decide what was worth implementing briefly in the lab versus what only makes sense in a different operational context.

The full benchmark is described in its own documentation as a catalog rather than a checklist, and several controls either do not apply to cloud instances or conflict with how this lab is designed to operate.

### sysctl Parameters

`sysctl.yml` sets kernel parameters for network hardening: disabling IP forwarding, disabling acceptance of source-routed packets, enabling SYN cookie protection, and restricting ptrace scope and kernel pointer exposure. Parameters are written to `/etc/sysctl.d/99-hardening.conf` and reloaded immediately.

```yaml
# roles/os-hardening/tasks/sysctl.yml
- name: Harden kernel parameters
  ansible.posix.sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    sysctl_file: /etc/sysctl.d/99-hardening.conf
    state: present
    reload: true
  loop:
    - { name: 'net.ipv4.conf.all.accept_source_route', value: '0' }
    - { name: 'net.ipv4.conf.all.accept_redirects', value: '0' }
    - { name: 'net.ipv4.tcp_syncookies', value: '1' }
    - { name: 'net.ipv4.ip_forward', value: '0' }
    - { name: 'kernel.yama.ptrace_scope', value: '1' }
    - { name: 'kernel.kptr_restrict', value: '1' }
    - { name: 'kernel.randomize_va_space', value: '2' }
    - { name: 'fs.suid_dumpable', value: '0' }
```

Using `sysctl_file` writes changes to a dedicated drop-in file rather than modifying `/etc/sysctl.conf` directly. This is worth noting because it keeps hardening changes isolated and easy to audit. `ansible.posix.sysctl` also reports `changed` vs `ok` accurately, which matters for idempotency: unlike a `shell` module call, it can tell the difference between setting a value and confirming one that's already correct.

This ran cleanly across all five reachable nodes. The actual output is worth showing because it makes the scope of the changes concrete.

```
TASK [os-hardening : Harden kernel parameters] 
changed: [ansible-lab-Debian-Managed] => (item={'name': 'net.ipv4.conf.all.accept_source_route', 'value': '0'})
changed: [ansible-lab-Ubuntu-Managed] => (item={'name': 'net.ipv4.conf.all.accept_source_route', 'value': '0'})
changed: [ansible-lab-AL2023-Managed] => (item={'name': 'net.ipv4.conf.all.accept_source_route', 'value': '0'})
changed: [ansible-lab-SUSE-Managed] => (item={'name': 'net.ipv4.conf.all.accept_source_route', 'value': '0'})
changed: [ansible-lab-Fedora-Managed] => (item={'name': 'net.ipv4.conf.all.accept_source_route', 'value': '0'})
...
changed: [ansible-lab-AL2023-Managed] => (item={'name': 'kernel.randomize_va_space', 'value': '2'})
changed: [ansible-lab-Fedora-Managed] => (item={'name': 'fs.suid_dumpable', 'value': '0'})
```

RHEL dropped out before fact gathering completed with a `TargetNotConnected` error on that run. It reconnected cleanly on rerun: the SSM agent occasionally needs a moment after boot before sessions are accepted.

### sshd: A Service Name Problem

The `sshd.yml` task was written to stop and disable the SSH daemon. The connection transport is SSM, not SSH, but sshd is running on every node by default and leaving it enabled is not the right call.

The first run produced this:

```
TASK [os-hardening : Stop and disable sshd service]
fatal: [ansible-lab-Ubuntu-Managed]: FAILED! => {"changed": false, "msg": "Could not find the requested service sshd: host"}
fatal: [ansible-lab-Debian-Managed]: FAILED! => {"changed": false, "msg": "Unable to stop service sshd: Failed to stop sshd.service: Unit sshd.service not loaded.\n"}
changed: [ansible-lab-AL2023-Managed]
changed: [ansible-lab-Fedora-Managed]
changed: [ansible-lab-SUSE-Managed]
```

Debian and Ubuntu name the service `ssh`, not `sshd`. The fix is a single task with an inline ternary that resolves the correct name at runtime.

```yaml
# roles/os-hardening/tasks/sshd.yml
- name: Stop and disable sshd service
  ansible.builtin.service:
    name: "{{ 'ssh' if ansible_os_family == 'Debian' else 'sshd' }}"
    state: stopped
    enabled: false
```

AL2023, Fedora, and SUSE use `sshd` and changed cleanly.

`filesystem.yml` sets sticky bits on world-writable directories. Standard CIS control, no cross-distro issues.

### Firewalld: Scoped Out

The initial role included a task to remove SSH from firewalld's allowed services. That task failed across every node it reached.

```
TASK [os-hardening : Remove SSH service from firewalld]
fatal: [ansible-lab-AL2023-Managed]: FAILED! => {"changed": false, "msg": "Failed to import the required Python library (firewall) on ip-10-0-2-61.ec2.internal's Python /usr/bin/python3."}
fatal: [ansible-lab-Fedora-Managed]: FAILED! => {"changed": false, "msg": "Failed to import the required Python library (firewall) on ip-10-0-2-75.ec2.internal's Python /usr/bin/python3."}
fatal: [ansible-lab-SUSE-Managed]: FAILED! => {"changed": false, "msg": "Failed to import the required Python library (firewall) on ip-10-0-2-202's Python /usr/bin/python3."}
```

The `firewall` Python library was not present on any of the nodes. Installing it would have meant adding a pre-task to the role, and that led to a more useful question: what is firewalld actually protecting here? The managed nodes have no inbound rules in their security group outside of traffic from the control node. SSHd is disabled. There is no service exposed to the network that firewalld would need to gate. Adding firewalld management to the role would be complexity for its own sake. The task was removed and the README documents the reasoning.

## What's Next

Part 4 covers what happens when you try to enforce all of this in a CI/CD pipeline: GitHub Actions, OIDC authentication, ansible-lint, Terraform fmt and validate, and the real pipeline failures that came out of it.
