---
layout: post
title: AWS Keycloak Lab - Part 2
date: 2025-10-07 11:09:00 -500
categories: [Homelab]
tags: [aws,idp,cloud,keycloak,iam,terraform]     # TAG names should always be lowercase
media_subpath: /assets/img/Keycloak2/
image: Keycloak2FrontImage.jpg
---

## First Forays into Terraform

Previously, we successfully established the foundational governance perimeter-a secure, multi-account environment governed by IAM Identity Center and centralized CloudTrail. Now, we make the critical transition to using Infrastructure as Code (IaC) to declare what the environment should look like.

This phase required significant research and tooling adoption even before writing anything. I had to effectively scribble out and diagram what I wanted. Having zero prior exposure to Terraform, the entire codebase for this project began by following the official HashiCorp Terraform "Get Started" tutorial [here](https://developer.hashicorp.com/terraform/tutorials/aws-get-started). This ensured a foundational understanding based on authoritative quick-start documentation, rather than Youtuber/influencer content. There's nothing worse than a 15 minute video about a two paragraph concept in documentation. The tutorial is fairly basic but it forces you to start thinking about what resources you need to declare or specify in your architecture. It's not all default VPCs and security groups out here! I particularly enjoyed this part. It's quite fun and dreamy, honestly. The complexity would quickly ramp up as I moved from static resource deployment to storing/using dynamic secrets and handling script errors.

Also integral to this phase was acquiring a domain. I purchased a domain off Namecheap for, well, cheap. The domain I purchased, `tonymacaronis.net` is supposed to be for an imaginary and zany Italian-American restaurant chain. It turns out there's a Tony Macaroni franchise in Scotland. Please don't sue me.

Although it looks like this was a seamless process, it was very much not so, especially when I also had to initialize Terraform with my correct Development account and made goofy mistakes. I quickly recognized that throwing everything into just a `main.tf` would get overwhelming real fast and so I started parsing out sections as needed. I referred to some of the first building blocks, but quickly began to use resources like ChatGPT and Gemini. These platforms did not necessarily streamline the process, and would sometimes write faulty, weird, hard-coded values and also required lots of manual debugging after punching in `terraform plan` for the nth time. This could be a whole separate post, but Generative AI isn't always a panacea for bad craftsmanship. Fortunately, I know enough to know what I don't know.

I opted for a pragmatic, single-server monolithic architecture built around one EC2 instance to accelerate progress over seeking immediate perfection. This setup deliberately avoids managed services (like RDS, Fargate, or an ALB) to showcase full, low-level Infrastructure as Code (IaC) control while reducing cost for the moment. The stack is deployed on Amazon Linux, where Nginx is installed directly on the "bare metal" to act as the primary TLS terminator and reverse proxy. The core applications—Keycloak and its persistent backend, PostgreSQL—are deployed as containerized services managed by Docker Compose. Crucially, the system uses two separate EBS volumes: the root volume for the OS and Nginx, and an external EBS volume dedicated solely to PostgreSQL data persistence. The entire stack, including the deployment of Certbot for TLS certificates, is automated via a reasonably robust `user-data.sh` script, proving that a complete, functional identity gateway can be provisioned rapidly with a modest but rapidly expanding set of Terraform files. My Terraform files changed ***a lot***, but I'll try to enumerate and explain the design decisions that happened over time.

## Defining my Terraform Process

### networking.tf

The `networking.tf` file defines my custom VPC while ensuring external connectivity. Security group rules are specified later in `security.tf`. This is fairly barebones.

- Custom VPC: The `aws_vpc` resource creates a private, isolated network using the `10.0.0.0/16` CIDR block. Bigger is better!
- Internet Gateway (IGW): The `aws_internet_gateway` resource is provisioned and attached to the VPC, allowing traffic to flow in and out of our custom network.
- Public Subnet: The `aws_subnet` resource, created in the first Availability Zone `${var.region}a`, is configured with `map_public_ip_on_launch = true`.
- Routing: The `aws_route_table` directs all external traffic `0.0.0.0/0` to the IGW. The final `aws_route_table_association` links the public subnet to this routing table, making it a fully functional public network segment.
- Elastic IP (EIP) & Route 53: To ensure the public domain name (`auth.tonymacaronis.net`) is always resolved to the correct host regardless of instance changes, we allocate an Elastic IP (`aws_eip.keycloak_eip`) and assign it to the EC2 instance. The Route 53 record then targets this static EIP (`dns.tf`).

### dns.tf 

The `dns.tf` file ensures that our Keycloak instance has a fixed public address that is reliably resolved by our purchased domain and/or configured subdomains. This is essential for the IdP's public-facing nature and for the TLS certificate validation process.

- Zone Lookup: The data `aws_route53_zone` resource dynamically looks up the existing public hosted zone (like tonymacaronis.net) managed in AWS. This prevents hardcoding the Zone ID and makes the module reusable.
- Elastic IP (EIP): The `aws_eip` resource allocates a static IP address in AWS and attaches it to the provisioned EC2 instance. The EIP guarantees the public IP address won't change upon reboot or instance replacement, maintaining service continuity.
- A Record: The `aws_route53_record` resource creates the `auth` subdomain record, pointing its Type A value to the public IP of the Elastic IP resource. The low TTL of 300 seconds allows for quick DNS updates if maintenance is required.

### security.tf

The `security.tf` file handles both the EC2 instance key pair generation and the security group rules that define network access.

- Key Pair Generation: We use the tls_private_key resource to generate a new RSA key pair locally, register the public key with AWS (`aws_key_pair`), and save the private key (`keycloak-lab-key.pem`) to the local module path with restricted permissions (0400). This is crucial for initial SSH access.
- IP Discovery: The data "`http`" "`my_ip`" resource dynamically retrieves my current public IP address at plan time.
- Public Access: Ingress is allowed on ports 80 and 443 from `0.0.0.0/0` to allow client logins and, crucially, enable Certbot (the ACME client) to validate the domain and issue a public TLS certificate.
- Restricted SSH: The security group restricts SSH (Port 22) to only my dynamically discovered IP address. All outbound traffic is permitted by default.

> Restricting port 22 access to a single, dynamic public IP is a known Single Point of Failure (SPOF) for operational access. If my home IP changes or I need access from an unallowed location, I am locked out. For a production-ready environment, the solution would technically involve AWS Systems Manager (SSM) Session Manager, which completely eliminates the need for public SSH access. I have implemented this access option elsewhere, but I still wanted SSH around!
{: .prompt-warning }

### variables.tf and terraform.tfvars

These files decouple the infrastructure logic from the environment-specific values. The `variables.tf` file defines the schema, while the `terraform.tfvars` file provides the actual values used for deployment.

- Deployment Profile: We specify the AWS Region (us-east-1) and the local AWS CLI profile to ensure all resources are deployed to the correct account and region.
- Naming and Compute: The `project_name` (keycloak-lab) is used as a prefix for all created resources (VPC, Security Group, EIP, etc.) to aid in resource management and cleanup. The instance_type is set to t3.small for our SPOF launchpad.
- Public Access: Variables for the `domain_name` and `certbot_email` are critical, as they are used to configure Route 53 and to provision the public TLS certificate at runtime.

### secrets.tf 

This file is central to the project's security model, ensuring that sensitive data is never stored in plain text and is only retrieved at runtime by the intended EC2 instance. 

- Random Password Generation: We use the `random_password` resource for the PostgreSQL database password and the Keycloak administrative password. This guarantees high-entropy secrets that are unique to this deployment.
- SecureString Storage: All sensitive values (DB user/password, Keycloak admin user/password) are stored in AWS SSM Parameter Store as `SecureString` types.
- KMS Encryption: By setting `key_id = "alias/aws/ssm"`, the secrets are encrypted at rest using a default AWS KMS key managed by the SSM service. The EC2 instance will later use its IAM role (defined in `iam.tf`) to read and decrypt these secrets.

### outputs.tf 

The `outputs.tf` file defines the key pieces of information returned to the console after a successful `terraform apply`. This makes it easy to immediately access the newly provisioned resources.

- Public IP and URL: The public IP of the Keycloak instance (`ec2_public_ip`) is outputted, along with the expected final URL (`keycloak_url`).
- SSH Key Path: The path to the locally saved private key (`private_key_path`) is provided, which is necessary for the operator to SSH into the instance to perform any manual debugging or maintenance.

### main.tf

The `main.tf` file defines the core compute resource—the EC2 instance—and implements the architectural choice for data persistence, decoupling the PostgreSQL data from the operating system's root volume.

- Provider and AMI: The provider block configures the AWS profile and region. A data source dynamically fetches the latest Amazon Linux 2023 AMI, ensuring we always use a current, patched base image for the EC2 instance.
- EC2 Instance Definition: The `aws_instance` resource links together everything we've built so far: the security group, the key pair, the public subnet, and the desired instance type (t3.small). Crucially, it links to the IAM Instance Profile, enabling the Principle of Least Privilege (PoLP) for secret retrieval.
- Defining User Data: The entire application setup—from mounting the data volume to running Docker and configuring Nginx—is handled by the `user_data.sh` script. We use the templatefile function to inject dynamic variables (like the domain and project name) into the shell script at launch time, making the script portable and environment-aware.
- Data Decoupling: We create a separate, independent EBS volume (`aws_ebs_volume`) and use an `aws_volume_attachment` resource to attach it to the EC2 instance as the device `/dev/xvdf`. This guarantees that if the Keycloak EC2 instance is terminated, the PostgreSQL data is preserved and can be quickly remounted to a new instance.

## Conclusion

We have successfully laid the entire foundation. Every resource, from networking to security and compute, is declared in Terraform. This declarative approach, combined with our secrets management pipeline, makes the environment reproducible and secure.

The infrastructure is now built, but the Keycloak service is not yet running. The true nightmare of ensuring everything works—mounting the file system, installing Docker, fetching the SSM secrets, generating the SSL certificate, and finally locking down the service—all rests on the `user-data.sh` script. Getting this script right was a messy, multi-hour (multi-week) saga of failed launches and frustrating log review.

The next post is a deep technical dive into the `user-data.sh` script. We will walk line-by-line through the shell code that handles filesystem mounting, secure secret retrieval, Docker Compose orchestration, Nginx/Certbot automation, race condition navigation, and the final admin console lockdown.
