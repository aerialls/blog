+++
title = "Ubuntu Server 22.04 image with Packer and Subiquity for Proxmox"
slug = "ubuntu-server-2204-image-packer-subiquity-for-proxmox"
date = "2022-04-30T19:00:00+02:00"
lastmod = "2022-04-30T19:00:00+02:00"
categories = ["automation"]
tags = ["ubuntu", "packer", "proxmox"]
+++

This article illustrates how to generate an Ubuntu Server 22.04 image template using Packer and only Subiquity on Proxmox.

## Subiquity

This new live system is based on cloud-init and uses a YAML file to fully automate the installation process. It differs on several points from the previous system.

- The syntax is much easier to understand (YAML vs debconf-set-selections format).
- It's possible to have hybrid situations where some sections can be interactive and others answered automatically from the configuration.

![Subiquity setup in Ubuntu Server 22.04](/images/ubuntu/subiquity.png)

## Proxmox

Packer requires a user account to perform actions on the Proxmox API. The following commands will create a new user account `packer@pve` with restricted permissions.

```bash
$ pveum useradd packer@pve
$ pveum passwd packer@pve
Enter new password: ****************
Retype new password: ****************
$ pveum roleadd Packer -privs "VM.Config.Disk VM.Config.CPU VM.Config.Memory Datastore.AllocateSpace Sys.Modify VM.Config.Options VM.Allocate VM.Audit VM.Console VM.Config.CDROM VM.Config.Network VM.PowerMgmt VM.Config.HWType VM.Monitor"
$ pveum aclmod / -user packer@pve -role Packer
```

{{< notice info >}}
Proxmox `pveum` is available through an SSH connection or from the web shell accessible under the node parameters on the UI.
{{< /notice >}}

Download the Ubuntu Server 22.04 ISO from [the image repository](http://releases.ubuntu.com/22.04/). At the time of writing, the latest version available was `ubuntu-22.04-live-server-amd64.iso`. Put the ISO inside the `local` storage under the `ISO image` category.

![Local storage in Proxmox](/images/proxmox/storage.png)

## Packer

### Builder configuration

Packer will execute a workflow to create a new template that can be used later on to bootstrap new VMs quickly with a pre-based configuration already applied.

Technically, Packer will start a VM in Proxmox, launch the installer from the boot command with a reference to the config file and convert the VM into a template at the end when the setup has been fully completed.

```json
{
  "builders": [{
    "type": "proxmox",
    "proxmox_url": "https://proxmox.madalynn.xyz/api2/json",
    "username": "{{ user `proxmox_username` }}",
    "password": "{{ user `proxmox_password` }}",
    "node": "proxmox",
    "network_adapters": [{
      "bridge": "vmbr0"
    }],
    "disks": [{
      "type": "scsi",
      "disk_size": "20G",
      "storage_pool": "local-lvm",
      "storage_pool_type": "lvm"
    }],
    "iso_file": "local:iso/ubuntu-22.04-live-server-amd64.iso",
    "unmount_iso": true,
    "boot_wait": "5s",
    "memory": 1024,
    "template_name": "ubuntu-22.04",
    "http_directory": "http",
    "boot_command": [
      "c",
      "linux /casper/vmlinuz --- autoinstall ds='nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/' ",
      "<enter><wait>",
      "initrd /casper/initrd<enter><wait>",
      "boot<enter>"
    ],
    "ssh_username": "madalynn",
    "ssh_password": "madalynn",
    "ssh_timeout": "20m"
  }]
}
```

The majority of the parameters are pretty straightforward to understand. The `ssh_timeout` will give time to the installer to download the latest security updates during the setup.

Launch Packer with the following command.

```bash
$ packer build -var-file=secrets.json ubuntu.json
```

{{< notice tip >}}
The `var-file` parameter gives the flexibility to extract secrets (like credentials) and dynamic parameters to use the workflow to build several Ubuntu images. The minimum required should include Proxmox credentials from the user created previously.

```json
{
  "proxmox_username": "packer@pve",
  "proxmox_password": "fQk9f5Wd22aBgv"
}
```
{{< /notice >}}

Packer will start a HTTP server from the content of the `http` directory (with the `http_directory` parameter). This will allow Subiquity to fetch the cloud-init files remotely.

{{< notice warning >}}
The live installer Subiquity uses more memory than debian-installer. The default value from Packer (`512M`) is not enough and will lead to weird kernel panic. Use `1G` as a minimum.

```text
---[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance. ]---
```

![Ubuntu kernel panic](/images/ubuntu/kernel-panic.png)
{{< /notice >}}

The boot command tells cloud-init to start and uses the `nocloud-net` data source to be able to load the `user-data` and `meta-data` files from a remote HTTP endpoint. The additional `autoinstall` parameter will force Subiquity to perform destructive actions without asking confirmation from the user.

```json
{
  ...
  "boot_command": [
    "c",
    "linux /casper/vmlinuz --- autoinstall ds='nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/' ",
    "<enter><wait>",
    "initrd /casper/initrd<enter><wait>",
    "boot<enter>"
  ],
  ...
}
```

### Provisioner configuration

Cloud-init will take care of everything else. However, Packer will assume the provisioning is complete as soon as it is able to connect to the virtual machine via SSH. But at this time, the setup process won't be fully done. Packer should be told to wait until cloud-init is fully done.

Technically speaking, the easiest solution is to wait until the `/var/lib/cloud/instance/boot-finished` file is present. Creating this file is the last thing cloud-init does. A bash script with a simple `while` will do the trick.

```json
{
  "provisioners": [{
    "type": "shell",
    "inline": [
      "while [ ! -f /var/lib/cloud/instance/boot-finished ]; do echo 'Waiting for cloud-init...'; sleep 1; done"
    ]
  }
```

Provisioners will be executed directly on the VM once the SSH connection is available. Packer supports [a lot of provisioners](https://www.packer.io/docs/provisioners). For instance, it's possible in this step to launch an Ansible playbook or to configure the Chef client.

## Cloud-init

As Subitiquy uses cloud-init, the configuration should be present in two files, `user-data` and `meta-data`. `user-data` is the main config file that Subitiquity and cloud-init will use for the provisioning. `meta-data` is an addition file that can host some additional metadata in the [EC2 metadata service format](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html).

{{< notice note >}}
The `meta-data` file can be empty (and will be for Proxmox) but must be present, otherwise cloud-init will not start correctly.
{{< /notice >}}

```text
http
├── meta-data
└── user-data
```

Cloud-init supports multiple formats for config files. YAML is the easiest to understand and will be used for the following snippets. Subiquity adds a new module `autoinstall` which hosts all the configuration necessary for the installation.

```yaml
#cloud-config
autoinstall:
  ...
```

{{< notice warning >}}
Unlike a classic cloud-init file, everything must be under the `autoinstall` key. The rest will be ignored.
{{< /notice >}}

The [official documentation](https://ubuntu.com/server/docs/install/autoinstall-reference) lists all available parameters that can be configured. The scope is reduced compared to what was possible with debian-installer but Subiquity gives the ability to use all other [cloud-init modules](https://cloudinit.readthedocs.io/en/latest/topics/modules.html) to compensate.

{{< notice note >}}
Under the hood, Subiquity will be able to handle some actions by itself (like partitioning) and will generate a cloud-init config file that will be executed after a reboot for the rest.
{{< /notice >}}

All "native" cloud-init modules must be under the `user-data` key. For instance, to use the [`write_files` cloud-init module](https://cloudinit.readthedocs.io/en/latest/topics/modules.html#write-files), the following configuration can be used.

```yaml
#cloud-config
autoinstall:
  ...
  user-data:
    write_files:
        - path: /etc/crontab
          content: 15 * * * * root ship_logs
          append: true
    ...
```

### Autoinstall configuration

Autoinstall is responsible for answering all questions asked during setup (keyboard layout, additional packages, ...). The scope is limited and additional workflows should be managed with cloud-init modules (see above).


```yaml
#cloud-config
autoinstall:
  version: 1
  locale: en_US
  keyboard:
    layout: fr
  ssh:
    install-server: true
    allow-pw: true
  packages:
    - qemu-guest-agent
```

The `qemu-guest-agent` package is needed for Packer to detect the IP address of the VM to perform the SSH connection. This will also enable Proxmox to display VM resources directly in the user interface.

![Proxmox VM meta-data](/images/proxmox/vm-metadata.png)

The VM will be configured in English with a French keyboard. The mapping keys correspond to settings in `/etc/default/keyboard`. See [its manual page](http://manpages.ubuntu.com/manpages/focal/en/man5/keyboard.5.html) for more details.

The SSH server is needed for remote connections from Packer. By default, it will try to connect using only a username and a password. This requires enabling the `allow-pw` parameter.

{{< notice tip >}}
Without `allow-pw`, the SSH server will only accept connections using certificates. Packer must be configured to do so by using [the `ssh_keypair_name` section](https://www.packer.io/docs/communicators/ssh/#ssh_keypair_name).
{{< /notice >}}

### Identity

In addition to the previous parameters, Subiquity is also able to create a user account during the provisioning with the `identity` section.

```yaml
#cloud-config
autoinstall:
  identity:
    hostname: ubuntu
    username: madalynn
    password: $6$xyz$1D0kz5pThgRWqxWw6JaZy.6FdkUCSRndc/PMtDr7hMK5mSw7ysChRdlbhkX83PBbNBpqXqef3sBkqGw3Rahs..
```

This section is also responsible for setting the hostname. As this VM only serves as the basis for the template, this has no importance and should be set during the final provisioning.

The previous block will create an user `madalynn` with `madalynn` as a password.

{{< notice tip >}}
It's possible to generate a unix encrypted password with the following command.

```text
$ openssl passwd -6 -salt xyz madalynn
```
{{< /notice >}}

To have more flexibility over how the account is created, it's possible to use the cloud-init module `users` instead.

```yaml
#cloud-config
autoinstall:
  ...
  user-data:
    users:
        - name: madalynn
          passwd: $6$xyz$1D0kz5pThgRWqxWw6JaZy.6FdkUCSRndc/PMtDr7hMK5mSw7ysChRdlbhkX83PBbNBpqXqef3sBkqGw3Rahs..
          groups: [adm, cdrom, dip, plugdev, lxd, sudo]
          lock-passwd: false
          sudo: ALL=(ALL) NOPASSWD:ALL
          shell: /bin/bash
          ssh_authorized_keys:
            - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJEXrziuUOCpWPvwOsGuF4K+aq1ufToGMi4ra/1omOZb
```

This module gives more parameters to configure: groups, shell binary, SSH authorized keys... The `sudo` parameter allows the usage of the `sudo` command without entering any password. This will be useful later to use, for instance, Ansible to complete the installation from the template.

{{< notice note >}}
It's not possible to use the cloud-init module `users` and the autoinstall `identity` block at the same time. Subiquity will discard the `users` module configuration if `identity` is present.
{{< /notice >}}

### Networking

Subiquity allows the usage of two pre-configured layouts, `lvm` and `direct`. By default, Subiquity will use `lvm` with a logical volume of 4G. The installer will not extend the partition to use the full capability of the volume group. It's also possible to configure the size of the swapfile on the filesystem (`0` to disable).

```yaml
#cloud-config
autoinstall:
  ...
  storage:
    layout:
      name: direct
    swap:
      size: 0
```

If `direct` is used, a single partition `/dev/sda2` will be created using the full disk.

```text
Filesystem                           Size  Used Avail Use% Mounted on
udev                                 1.9G     0  1.9G   0% /dev
tmpfs                                394M  696K  393M   1% /run
/dev/sda2                             20G  3.6G   15G  20% /
tmpfs                                2.0G     0  2.0G   0% /dev/shm
tmpfs                                5.0M     0  5.0M   0% /run/lock
tmpfs                                2.0G     0  2.0G   0% /sys/fs/cgroup
```

It's also possible to use a more complex configuration based on [curtin](https://curtin.readthedocs.io/en/latest/topics/storage.html) under the `config` key. This will be a requirement if the VM has more than one disk (pre-configured layouts won't work in this situation).

```yaml
#cloud-config
autoinstall:
  ...
  storage:
    config:
      - type: disk
        id: root-disk
        size: largest
      - type: partition
        id: boot-partition
        device: root-disk
        size: 10%
      - type: partition
        id: root-partition
        size: 20G
      - type: partition
        id: data-partition
        device: root-disk
        size: -1
```

The previous configuration will create three partitions on the largest drive.

- 10% for the boot partition.
- 20G for the root partition.
- The rest for the data partition.

These are the first steps to a custom layout. However, it's not enough and will require other steps (format, mount, ...).

{{< notice warning >}}
If a pre-configured layout is used, the custom config will be ignored.
{{< /notice >}}

## GitHub

The full configuration is available on GitHub in the [aerialls/madalynn-packer](https://github.com/aerialls/madalynn-packer) repository.
