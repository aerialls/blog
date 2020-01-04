+++
title = "Ubuntu 18.04 image with Packer and Ansible for Proxmox"
slug = "ubuntu-image-with-packer-and-ansible-for-proxmox"
date = "2019-12-15T19:15:00+01:00"
lastmod = "2020-01-03T23:17:00+01:00"
categories = ["automation"]
tags = ["ubuntu", "packer", "proxmox", "ansible"]
+++

{{< alert "info" >}}
Even if the article focuses on Ubuntu 18.04,  it has been tested successfully for Ubuntu 19.04 and Ubuntu 19.10.
{{< /alert >}}

If you read my previous post, time has passed since then. I have ditched my Raspberry Pi 3B+ and Hassbian for a dedicated Intel NUC with Proxmox and Hass.io [^1]. But this is not the subject of this post.

With my love to automate things, I was looking for a way to generate a base Ubuntu image for Proxmox to use it later on with Ansible for severals VMs (Home-Assistant is just one example).

# Introduction

[Packer](https://www.packer.io/) is here to help. Packer will generate a base image usable in Proxmox as a template. To make it easier, Packer has a Proxmox builder for six months now.

Packer will use the builder to contact Proxmox to start a new VM and launch the setup from the boot command. When the setup is done, it's possible to execute some additional tasks with several provisioners.

# Proxmox

The first thing is to create a dedicated user for Packer in Proxmox. In the datacenter view on your Proxmox web-interface, create a new role `Packer` with the following privileges (all privileges are required).

* Datastore.AllocateSpace
* VM.Audit
* VM.Allocate
* VM.Config.CDROM
* VM.Config.CPU
* VM.Config.Disk
* VM.Config.Memory
* VM.Config.Network
* VM.Config.Options
* VM.Config.HWType
* VM.PowerMgmt
* VM.Monitor
* Sys.Modify

![Packer role in Proxmox](/images/proxmox/packer-role.png)

After that, create a new Packer user. For me, I've created it inside the `Linux PAM standard authentication` realm so the username will be `packer@pve`.

![Packer user in Proxmox](/images/proxmox/packer-user.png)

And to finish, link the user and the role to the main path `/` in the `Permissions` section.

![Packer permission in Proxmox](/images/proxmox/packer-permission.png)

# Packer

Download the latest Ubuntu Server 18.04 from the [official website](http://cdimage.ubuntu.com/ubuntu/releases/18.04/release/ubuntu-18.04.3-server-amd64.iso). Put the image inside an `iso` folder in the local storage in your Proxmox cluster.

![Local storage in Proxmox](/images/proxmox/storage.png)

Packer needs a JSON configuration file to work with two main sections: builders and provisioners. Builders will be responsible to start the VM and launch the preseed configuration and provisioners to execute additional tasks into our running VM at the end before generating the template.

{{< gist aerialls 51808799cb2c6fe5fede9270066ffd98 >}}

The username and password for the Proxmox connection will be store in another JSON file (that you can gitignore).

```json
{
    "username": "packer@pve",
    "password": "fQk9f5Wd22aBgv"
}
```

The previous configuration will create a new VM template named `ubuntu-18.04` based on `ubuntu-18.04.3-server-amd64.iso` with a 20G disk. Proxmox will try to load and execute a preseed file from an HTTP endpoint (started by Packer during the provisioning).

The preseed file will be responsible to configure every aspects of your VM (from the keyboard to the disk partitions or even the user creation...). It's a text fle with several instructions to specify what need to be configured.

The `preseed.cfg` file needs to be locally inside the `config` directory (or you will need to update the field `http_directory` in the JSON configuration).

```
.
├── config
│   └── preseed.cfg
└── ubuntu.json
```

Here is an example of what the file looks like.

{{< gist aerialls 9761d2935644d7da2b19ef761ef9d162 >}}

If you wan to learn more about preseed files, Ubuntu has [a full documentation on this subject](https://help.ubuntu.com/lts/installation-guide/s390x/apb.html).

The process creation can be launch with the following command.

```bash
packer build -var-file=secrets.json ubuntu.json
```

At the end of the process, the VM template will be fully configured (packages are up to date, the LVM partition is created) and an user account will also be created (`madalynn`/`madalynn`).

# Ansible

At this step, the template can be used for new machines. Just clone the template, give it a name and voilà. However, the VM cannot be used in Ansible directly. Some extra configuration is needed to allow Ansible to connect to it with a SSH key and also to be able to run `sudo` commands without passwords.

This is where Packer provisioners will help. Provisioners are a way to execute commands inside our VM from different methods. Two provisioners will be use: `shell` and `ansible`. The first one will execute a script to edit the sudoer configuration for our new custom user (to please Ansible). The second will execute an Ansible playbook to finish the configuration.

```json
{
    ...
    "provisioners": [
        {
            "type": "shell",
            "execute_command": "echo 'madalynn' | {{ .Vars }} sudo -S -E bash '{{ .Path }}'",
            "script": "scripts/setup.sh"
        }
}
```

The command launch the script with `sudo` and give the user's password from the input. It's possible to use variables to simplify writing. The full list is available [in the documentation](https://www.packer.io/docs/provisioners/shell.html).

The bash script is very simple and will just put a new file inside `/etc/sudoers.d` to allow the user to use `sudo` commands without entering any password.

```bash
#!/bin/bash

set -eux

# Enable "madalynn" user sudo without password
echo "madalynn ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/madalynn
```

The second provisioner will launch an Ansible playbook to add the public key for the `madalynn` user and also to update all packages on the VM. It's possible to do the same with a bash script but it is much simple with a playbook (and the goal is to enable Ansible no?).

```json
{
    ...
    "provisioners": [
        ...
        {
            "type": "ansible",
            "playbook_file": "./ansible/provisioning.yml"
        }
}
```

The playbook file consists of two tasks. The first one update all packages to the latest version. The second one will push the public key used later by Ansible

```yaml
- hosts: all
  tasks:
    - name: Update all packages to the latest version
      apt:
        upgrade: dist
        autoremove: true
      become: true
    - name: Add public SSH key for the Madalynn user
      authorized_key:
        user: madalynn
        key: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIncmwjiCyDLFn515hRl4TgmKcoFyxnTkROKl3OI1jFB
        state: present
        exclusive: true
```

And that's it! VM generated from the template can be used directly in Ansible with the `madalynn` user.

 All files can be found in [aerialls/madalynn-packer](https://github.com/aerialls/madalynn-packer) GitHub repository.

[^1]: [Hassbian is now dead](https://www.home-assistant.io/blog/2019/10/26/rip-hassbian/) so a choice was to be made.
