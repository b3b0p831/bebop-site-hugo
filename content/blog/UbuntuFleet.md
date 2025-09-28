---
title: Managing Macbook Air Fleet with Ubuntu 24.04 and Ansible
date: 2025-06-15
draft: true
showTOC: true
weight: 1
disableShare: true
---

![Ubuntu](Fleet/img/base.png)

## Time for a Change

As the IT Admin for a small school district, I’m responsible for all the tech onsite. Recently, I was tasked with refreshing the student laptop fleet—about 30–40 MacBook Airs (2017, A1466).

The obvious path seemed to be Chromebooks. They’re cheaper, cloud-managed, and popular in K-12. But when I compared the Airs to current Chromebooks, I was surprised at how well the Macs held up:

| Spec        | MacBook Air 2017 (A1466) | Dell 3120 Chromebook |
|-------------|--------------------------|----------------------|
| **CPU**     | Intel Core i5 (I5-5350U) | Intel® N100          |
| **Memory**  | 8 GB LPDDR3              | 4 GB LPDDR5          |
| **Storage** | 128 GB SSD               | 64 GB eMMC           |
| **OS**      | Monterey 12.7.6          | ChromeOS             |

The real issue was Monterey 12.7.6, it is no longer receiving security updates and considered end of life.

After talking with teachers and staff, and looking back at the most common problems I saw in my first year, I narrowed the refresh down to three requirements:

- **Basic web computing** (Google Apps, web, video, etc.)  
- **Must run state testing software** (standardized exams require specific apps)  
- **Shared device model** (any student can grab any student laptop, requires the use of webcam)   

As long as the new device satisfy these 3 requiremetns, we have freedom to expermiment with different operating systems.

## Same Device, New OS

Okay great, so we know the hardware isn't the issue here. The real issue lies with the OS, Monterey 12.7.6. 

Support for MacOS Monterey 12.7.6 was ended shortly after the release of MacOS Sequoia and now is considered EOL(end-of-life). This means that this version of macOS will no longer receive security updates, and many software vendors are likely to drop support in the near future.

My first choice for replacing macOS Monterey was [ChromeOS Flex](https://chromeos.google/products/chromeos-flex/). This is a version of ChromeOS released by google that is basically meant for this exact purpose. Old devices are losing support so you install ChromeOS flex and badda bing badda boom you have chromebooks

Perfect, right?

## Automating the Install

Out of the box, Ubuntu 24.04 doesn’t support the Broadcom Wi-Fi chipset or the FaceTime HD camera. The Wi-Fi problem was especially troublesome, since my goal was for each device to connect automatically right after autoinstall.

During my research on custom Linux ISOs, I came across [livefs-editor](https://github.com/mwhudson/livefs-editor). Livefs is a python tool used to make modified versions of distributed ISOs.


### Step 1: Create a base ISO
Include the packages you want on the target machine:

```sh
livefs Ubuntu-24.04.iso base.iso --install-packages openssh-server avahi-daemon python3 sssd
````

### Step 2: Add configuration files

After creating the base image, layer in configuration files and network settings:

```sh
livefs-edit base.iso fleet.iso \
--add-autoinstall-config 'local/path/to/user-data' \
--cp 'local/path/to/fleet.nmconnection' '$LAYERS[0]/etc/NetworkManager/system-connections/fleet.nmconnection'
```

### Example: Student autoinstall config

```yaml
#cloud-config
autoinstall:
  version: 1
  user-data:
    users:
      - name: fleet-adm
        gecos: "Fleet Admin"
        shell: /bin/bash
        groups: [sudo]
        lock_passwd: false
        passwd: <password hash here>
        ssh_authorized_keys:
          - <pub ssh key here>        

      - name: student
        gecos: "Student"
        shell: /bin/bash
        lock_passwd: false
        passwd: 'U6aMy0wojraho'

  storage:
    layout:
      name: lvm

  late-commands:
    # configure Wi-Fi connection “fleet”
    - chmod 600 /target/etc/NetworkManager/system-connections/fleet.nmconnection
    - chown root:root /target/etc/NetworkManager/system-connections/fleet.nmconnection
    # Hostname based on device serial #
    - curtin in-target --target=/target -- bash -c 'serial=$(cat /sys/class/dmi/id/product_serial | tr -cd "a-zA-Z0-9"); echo "$serial" > /etc/hostname'
    - curtin in-target --target=/target hostnamectl set-hostname "$(cat /etc/hostname)"
    - curtin in-target --target=/target -- bash -c 'echo "127.0.1.1 $(cat /etc/hostname)" >> /etc/hosts'
```

Once complete, the `fleet.iso` image can be flashed to a USB stick. When booted, it will automatically begin the install process and reboot once finished.


## Managing Devices with Ansible

Now that our devices are running the Custom Ubuntu Image, we can push the final configuration via Ansible.

Thanks to the autoinstall, the devices are discoverable via mDNS. We can view the status of our devices by running an ansible ping command like so

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


Here is what inventory.ini should look like
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

Now this is where everything falls apart. Ansible is not a Mobile Device Managment tool. It is an open source automation tool for managing repeatable tasks. This means we are lacking many features that you would expect like Remote Wipe, Remote Lock, etc. 

![Ubuntu](Fleet/img/ansible.png)


## Final Thoughts, Future Plans

Was this the best option? Probably not. Will we end up switching to Chromebooks at some point? Yeah, most likely. Do I regret my decision? No! This was an amazing learning opportunity as I got to mess around with Linux, orchestration and device managment while providing a cost-effective solution for student devices.

One promising recent addition to Ubuntu is [authd](https://documentation.ubuntu.com/authd/stable-docs/), an authentication service that integrates with cloud identity providers like Microsoft Entra ID or Google IAM.

In K12, a popular option is **Clever**, which works like Okta for schools. Students sign into Clever and gain access to all district apps/services via the portal. If someone (me?) could write the identity broker for Clever with [QR code sign-in](https://support.clever.com/hc/s/articles/360024681032?language=en_US), that would be a game changer.

Things to think about…



