---
layout: post
title: Ansible Linux Sandbox - Part 1: From Inquiry to First Execution
date: 2026-06-06 11:00:00 -0500
categories: [Homelab]
tags: [aws, ansible, terraform, linux, ssm]
media_subpath: /assets/img/AnsibleSandbox1/
image: AnsibleSandbox1FrontImage.png
---

I was considering what else I should pivot to given my last foray into work. I was doing some consistent shell scripting in the background while staying busy with work and the holidays so I found time to finally start or pivot to sometthing different. In this case, I knew enough to see that reviewing Ansible would be worthwhile and I was already curious to see how organizations manage hundreds of nodes at scale. ansible felt like the right choice here esepcially since i had seen the limitations of using shell scripts for one off deployments or even across distributions so this was a natural topic to ponder. It almost feels quaint to think about it now. Honestly revisiting Jeff Geerling's Ansible for DevOps and his Covid-era livestream series. I completed about half of it before realizing I wasn;t too intersted in recreatinng the infrastructure using vagrant as is used in the book that he has. i suppose that can be of interst because then its all on your own machine but the difficulty here is that well I'd rather lean more into my cloud technology exposure
https://www.youtube.com/watch?v=goclfp6a2IQ&list=PL2_OBreMn7FqZkvMYt6ATmgC0KAGGJNAN



never having use vagrant althoiugh technically it would run fine on my 32 GB RAM Debian machine i figured why go through trouble of all that when i wanted to simulate a more releaistic cloud or production environement for myself.


besides the aws infrastructure we only really have ansibel.cfg inventoty.ini and a goofy ssh key which is really more so  a holdover but i wasnt particulary happy about implementing something where we keep a private key on our machine locally which felt more like a quick sloppy tutorial to demonstrate, hey look heres this feature now tear this piece fo trash down again! kind of youtube video yknow!

I intially started with four instances. imagining that would be enough for me to investigate. i think its importatn to articulate those specific files as well



towards end and after saying some of the specificstuff.i
it can be ideal for testing ad hoc paylods and quick one off playbooks with very brief steps. but then again  iwanted to have more which led me to what i will describe more of in part 2.
