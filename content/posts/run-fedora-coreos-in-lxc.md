+++
draft = false
authors = ["kuba86"]
date = "2023-04-30"
lastmod = "2023-05-03"
images = ["/posts/run-fedora-coreos-in-lxc.png"]
title = "How to run Fedora CoreOS in lxc"
description = "Easy to follow instructions on how to add Fedora CoreOS image to lxc and how to launch it as VM in lxc"
tags = [
    "lxc",
    "fedora",
    "coreos",
    "containers",
    "linux",
]
aliases = []
+++
![How to run Fedora CoreOS in lxc](/posts/run-fedora-coreos-in-lxc.png)

## Intro

Fedora CoreOS advertises itself as container first OS, immutable, auto-updating, scalable, and secure. As I am beginning to learn Kubernetes, it looks like a perfect match. Regardless of if you prefer Fedora CoreOS, Flatcar Container Linux, or anything else, you first want to play with it in a local environment. In 2022 I switched from Windows 11 to Fedora, and I was searching for a lightweight solution to run Virtual Machines and Container OS - lxc / lxd feels like a great alternative to Oracle VirtualBox. Installing Ubuntu, Fedora or Alpine Linux is easy with lxc, however, what do you do if you want to run something different? Below you will find easy to follow instructions on how to add Fedora CoreOS image to lxc and how to launch it as VM in lxc.

## Content
1. [Download Fedora CoreOS image](#download-fedora-coreos-image)
1. [Create metadata.yaml file](#create-metadatayaml-file)
1. [Import Fedora CoreOS image to lxc](#import-fedora-coreos-image-to-lxc)
1. [Create Butane config and convert it to Ignition config](#create-butane-config-and-convert-it-to-ignition-config)
1. [Create and launch Fedora CoreOS in lxc](#create-and-launch-fedora-coreos-in-lxc)
1. [End](#end)

## Download Fedora CoreOS image
First, head over to [Fedora CoreOS download page](https://fedoraproject.org/coreos/download) and download QEMU image. It should be under `Bare Metal & Virtualized` -> `Virtualized` -> `Fedora CoreOS QEMU qcow2.xz`. Once you download it and verified it, you can extract the archive, so we will get `qcow2` image. Command:

```shell
unxz fedora-coreos-37.20230401.3.0-qemu.x86_64.qcow2.xz
```

## Create metadata.yaml file
Before we can import the image, we need to provide some information in a `metadata.yaml` file and create a `tar` archive of the file. The `metadata.yaml` file has only two required fields: `architecture` and `creation_date`. This information can be easily extracted from file name of the image we downloaded. 

given:
```
fedora-coreos-37.20230401.3.0-qemu.x86_64.qcow2.xz
```

our file will look like:
```yaml
architecture: x86_64
creation_date: 1680307200
properties:
  description: Fedora CoreOS
  os: fedora-coreos
  release: 37.20230401.3.0
```

We can save it as `metadata.yaml` then, `tar` the file with command:

```shell
tar cf metadata.tar metadata.yaml
```

## Import Fedora CoreOS image to lxc
Now that we have `metadata.tar` file and `fedora-coreos-37.20230401.3.0-qemu.x86_64.qcow2` image we can import it to lxc with command:
```
lxc \
    image \
    import \
    metadata.tar \
    fedora-coreos-37.20230401.3.0-qemu.x86_64.qcow2 \
    --alias fedora-coreos/37.20230401.3.0
```

Once the import is done, you should see a message with the imported image fingerprint:
```
Image imported with fingerprint: def3bfeef0c58d2d4820151f670931ae9755630671e5c2e69327227c44276cfa
```

To list local images in lxc run:
```
lxc image list
```

The result:
```
+-------------------------------+--------------+--------+---------------+--------------+-----------------+-----------+-------------------------------+
|             ALIAS             | FINGERPRINT  | PUBLIC |  DESCRIPTION  | ARCHITECTURE |      TYPE       |   SIZE    |          UPLOAD DATE          |
+-------------------------------+--------------+--------+---------------+--------------+-----------------+-----------+-------------------------------+
| fedora-coreos/37.20230401.3.0 | def3bfeef0c5 | no     | Fedora CoreOS | x86_64       | VIRTUAL-MACHINE | 1568.20MB | Apr 29, 2023 at 11:04am (UTC) |
+-------------------------------+--------------+--------+---------------+--------------+-----------------+-----------+-------------------------------+
```

To get more info about the image:
```
lxc image info fedora-coreos/37.20230401.3.0
```

The result:
```
Fingerprint: def3bfeef0c58d2d4820151f670931ae9755630671e5c2e69327227c44276cfa
Size: 1568.20MB
Architecture: x86_64
Type: virtual-machine
Public: no
Timestamps:
    Created: 2023/04/01 00:00 UTC
    Uploaded: 2023/04/29 11:04 UTC
    Expires: never
    Last used: never
Properties:
    os: fedora-coreos
    release: 37.20230401.3.0
    description: Fedora CoreOS
Aliases:
    - fedora-coreos/37.20230401.3.0
Cached: no
Auto update: disabled
Profiles:
    - default
```

## Create Butane config and convert it to Ignition config

[Butane](https://coreos.github.io/butane/) config is a human-readable file that customizes Fedora CoreOS deployment. Once we prepare Butane config yaml file, we will convert it to [Ignition](https://coreos.github.io/ignition/) config, which is machine-readable json file. The online documentation is well written, so if you need to [add files](https://coreos.github.io/butane/examples/#files), change machine name, or [autostart a process](https://coreos.github.io/butane/examples/#systemd-units) - read the docs 😀

```
fedora-coreos-lxc.yaml
```

```yaml
variant: fcos
version: 1.5.0
passwd:
  users:
  - name: core
    ssh_authorized_keys:
    - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... main
    - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... backup
storage:
  files:
  - path: /etc/hostname
    mode: 0644
    contents:
      inline: fcos-lxc
```
Convert Butane config to Ignition config: `butane --pretty --strict fedora-coreos-lxc.yaml > fedora-coreos-lxc.ign`

```
fedora-coreos-lxc.ign
```

```json
{
  "ignition": {
    "version": "3.4.0"
  },
  "passwd": {
    "users": [
      {
        "name": "core",
        "sshAuthorizedKeys": [
          "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... main",
          "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... backup"
        ]
      }
    ]
  },
  "storage": {
    "files": [
      {
        "path": "/etc/hostname",
        "contents": {
          "compression": "",
          "source": "data:,fcos-lxc"
        },
        "mode": 420
      }
    ]
  }
}
```

## Create and launch Fedora CoreOS in lxc

`lxc` has `launch` command that will create and start our VM. I installed `lxd` / `lxc` via `snap` which complicates passing files from host. To pass our Ignition config we have to add `-c raw.qemu` and `-c raw.apparmor`. Adding `/var/lib/snapd/hostfs` as prefix to paths is required due to how `snap` work. 

```bash
lxc launch fedora-coreos/37.20230401.3.0 fcos-lxd \
    --vm \
    --config limits.cpu=1 \
    --config limits.memory=4GiB \
    -c raw.qemu="-fw_cfg name=opt/com.coreos/config,file=/var/lib/snapd/hostfs/home/kuba/Downloads/fedora-coreos/fedora-coreos-lxc.ign" \
    -c raw.apparmor="/var/lib/snapd/hostfs/home/kuba/Downloads/fedora-coreos/fedora-coreos-lxc.ign r,"
```

to see info about available instances:

```
lxc list
```

the result:

```
+----------+---------+-----------------------+-----------------------------------------------+-----------------+-----------+
|   NAME   |  STATE  |         IPV4          |                     IPV6                      |      TYPE       | SNAPSHOTS |
+----------+---------+-----------------------+-----------------------------------------------+-----------------+-----------+
| fcos-lxd | RUNNING | 10.160.159.200 (eth0) | fd42:6c0f:ca67:774b:216:3eff:fefa:33b7 (eth0) | VIRTUAL-MACHINE | 0         |
+----------+---------+-----------------------+-----------------------------------------------+-----------------+-----------+
```

to connect via ssh:

```
ssh core@10.160.159.200

```

```
The authenticity of host '10.160.159.200 (10.160.159.200)' can't be established.
ED25519 key fingerprint is SHA256:w7Lv1RQiR/lQAGp3TOq37jnhHnKaSLbU6uvh3sgJn+8.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.160.159.200' (ED25519) to the list of known hosts.
Fedora CoreOS 37.20230401.3.0
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/tag/coreos

[core@fcos-lxd ~]$ 
```

# End

This is it! 💪 We added Fedora CoreOS image to `lxc`, created `metadata.yaml` file, customized deployment with `Butane` and `Ignition` config. 