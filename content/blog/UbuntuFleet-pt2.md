---
title: Managing a Macbook Air Fleet with Ubuntu 24.04 LTS and Ansible, pt.2
date: 2025-09-30
draft: true
showTOC: true
weight: 1
disableShare: true
---

## Managing Devices with Ansible

Great, the fleet is running Ubuntu 24.04 and connected to the network. Using an inventory list of serial numbers, we can view the status of our fleet by running an ansible ping command. 

For those unfamiliar, Ansible is an open source automation tool for managing repeatable tasks. 

```sh
$ ansible ubuntu-fleet -m ping -i inventory.ini

FAKE8C33FF01.local | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.12",
        "changed": false,
        "ping" : "pong"
    }
}
FAKE8C33FF02.local | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.12",
        "changed": false,
        "ping" : "pong"
    }
}
...
```

### inventory.ini
```toml
[ubuntu-fleet]
FAKE8C33FF01.local
FAKE8C33FF02.local
FAKE8C33FF03.local
FAKE8C33FF04.local
FAKE8C33FF05.local
FAKE9D44AA10.local
FAKE9D44AA11.local
FAKE9D44AA12.local
FAKE9D44AA13.local
FAKE9D44AA14.local

```

We've now arrived at the most difficult part, but most fun in my opinion. Ansible is a very powerful tool, however its only as good as the playbooks that you are using. I think this is a good spot to stop for a part 2 as this post is getting kinda long.

In part 2, I will cover managing 


For example, here are a few of the playbooks I used to configure the student laptops.


```sh
- fleet
 - playbooks
    - facetimehd.yml
    - install_edr.yml
    - desktop.yml
 - student.yml
```


### facetimehd.yml
```yml

```



## Final Thoughts, Future Plans

### Agent v. Agentless

One change that I think would be relativly simple would be to use a orchestration framework that utilizes agents. For example, utilizing something like [SaltStack](https://docs.saltproject.io/en/latest/topics/about_salt_project.html#about-salt) I think would make for an easier setup.

### Clever + Authd?

One big improvement that I think could really make a difference has to deal with a recent addition to Ubuntu konw as [authd](https://documentation.ubuntu.com/authd/stable-docs/). According to Canonical, authd is 'an authentication service that integrates with cloud identity providers like Microsoft Entra ID or Google IAM.' Think of something like Single Sign On (SSO) for Ubuntu GDM.


In K12, a popular IAM service is known as **Clever**. Students sign into Clever and gain access to all district apps/services via the portal. If someone (me?) could write the identity broker for Clever with [QR code sign-in](https://support.clever.com/hc/s/articles/360024681032?language=en_US), that would be a game changer.

Things to think about…


![](Fleet/img/clever_webcam.webp)

Was this the best option? Probably not. Will we end up switching to Chromebooks at some point in the future? Yeah, most likely. Do I regret my decision? No! This was an amazing learning opportunity as I got to mess around with Linux, orchestration and device managment while providing a cost-effective solution for student devices.


