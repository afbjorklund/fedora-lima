# Fedora Lima

See <https://github.com/lima-vm/lima>

Requires Lima version 0.8.0 or later

![lime fedora](lime-fedora.png)

## Background

Podman is a Linux-only program, that creates Linux containers/images.

Therefore it needs to create a Linux virtual machine (VM), on other OS.

Podman v3 will by default create virtual machines using [Fedora CoreOS](https://getfedora.org/en/coreos/).

But it is also possible to create virtual machines using [Fedora Cloud](https://cloud.fedoraproject.org)...

### History

This can still be done "manually", and `podman` installed using `dnf`.

Podman version 1 (varlink) created virtual machines using [Fedora Linux](https://github.com/boot2podman/boot2podman-fedora-iso).

Doing an ISO would require writing a custom "kickstart" file, for Lima.

It would be a "live" image, similar to [alpine-lima](https://github.com/afbjorklund/alpine-lima). See the [fedora-live](https://github.com/afbjorklund/fedora-live) files.

## Usage

Run `limactl start fedora-podman.yaml` to create a Lima instance named "fedora-podman".

To open a shell, run `limactl shell fedora-podman bash` or `LIMA_INSTANCE=fedora-podman lima`.

### user

limactl shell fedora-podman podman

```console
$ export LIMA_INSTANCE=fedora-podman
$ lima podman version
Version:      3.4.0
API Version:  3.4.0
Go Version:   go1.16.8
Built:        Thu Sep 30 19:40:21 2021
OS/Arch:      linux/amd64
$ lima podman info | grep rootless
    rootless: true
```

The remote socket is listening as well, if needed:

```console
$ lima systemctl --user status podman.socket
...
$ lima podman --remote version
Client:
Version:      3.4.0
API Version:  3.4.0
Go Version:   go1.16.8
Built:        Thu Sep 30 19:40:21 2021
OS/Arch:      linux/amd64

Server:
Version:      3.4.0
API Version:  3.4.0
Go Version:   go1.16.8
Built:        Thu Sep 30 19:40:21 2021
OS/Arch:      linux/amd64
$ lima podman --remote info | grep -A2 remoteSocket
  remoteSocket:
    exists: true
    path: /run/user/1000/podman/podman.sock
```

### system

limactl shell fedora-podman sudo podman

```console
$ export LIMA_INSTANCE=fedora-podman
$ lima sudo podman version
Version:      3.4.0
API Version:  3.4.0
Go Version:   go1.16.8
Built:        Thu Sep 30 19:40:21 2021
OS/Arch:      linux/amd64
$ lima sudo podman info | grep rootless
    rootless: false
```

The remote socket is listening as well, if needed:

```console
$ lima sudo systemctl --system status podman.socket
...
$ lima sudo podman --remote version
Client:
Version:      3.4.0
API Version:  3.4.0
Go Version:   go1.16.8
Built:        Thu Sep 30 19:40:21 2021
OS/Arch:      linux/amd64

Server:
Version:      3.4.0
API Version:  3.4.0
Go Version:   go1.16.8
Built:        Thu Sep 30 19:40:21 2021
OS/Arch:      linux/amd64
$ lima sudo podman --remote info | grep -A2 remoteSocket
  remoteSocket:
    exists: true
    path: /run/podman/podman.sock
```

## Socket

The new version of lima automatically sets up unix sockets.

So you can connect directly to the VM, by setting a variable:

`export CONTAINER_HOST=unix://$DIR/sock/podman.sock`

And lima will make sure to tunnel this to the VM, using SSH.

You can use `limactl list`, to see the instance directory.

Normally it is located in a subdirectory, under `~/.lima`.

## Root

By default, podman will use user unless switching socket:

`export CONTAINER_HOST=unix:/run/podman/podman.sock`

But the system service requires root, for instance `sudo`.

`Error: unable to connect to Podman socket: Get "http://d/v3.4.0/libpod/_ping": dial unix /run/podman/podman.sock: connect: permission denied`

Alternatively, one can use a root-equivalent group: ([#6809](https://github.com/containers/podman/issues/6809))

```shell
groupadd -f -r podman

    mkdir -p /etc/systemd/system/podman.socket.d
    cat >/etc/systemd/system/podman.socket.d/override.conf <<EOF
[Socket]
SocketMode=0660
SocketUser=root
SocketGroup=podman
EOF
    systemctl daemon-reload
    echo "d /run/podman 0770 root podman" > /etc/tmpfiles.d/podman.conf
    systemd-tmpfiles --create
    systemctl restart podman.socket

usermod -aG podman $SUDO_USER
```

This is very similar to the "docker" group for docker.

| **WARNING** |
| ----------- |
| The podman group grants privileges equivalent to the root user. |

## Connections

To view the information about lima instances:

```console
$ limactl list
NAME             STATUS     SSH                ARCH      DIR
fedora-podman    Running    127.0.0.1:45007    x86_64    /home/anders/.lima/fedora-podman
```

Then, to add a user connection: (as default)

`podman system connection add --identity ~/.lima/_config/user --default lima ssh://127.0.0.1:45007`

Or, to add a system connection: (requires root)

`podman system connection add --identity ~/.lima/_config/user lima-root ssh://127.0.0.1:45007/run/podman/podman.sock`

Now there are two different podman connections:

```console
$??podman system connection list
Name        Identity                         URI
lima*       /home/anders/.lima/_config/user  ssh://anders@127.0.0.1:45007/run/user/1000/podman/podman.sock
lima-root   /home/anders/.lima/_config/user  ssh://anders@127.0.0.1:45007/run/podman/podman.sock
```

That can be used with the podman-remote client:

`podman --remote --connection lima`

`CONTAINER_CONNECTION=lima-root podman-remote`

## Packages

Lima starts with a Fedora Cloud .qcow2 installation.

* Kickstart: [fedora-cloud-base.ks](https://pagure.io/fedora-kickstarts/blob/f35/f/fedora-cloud-base.ks)

```kickstart
%packages --instLangs=en
@^cloud-server-environment
-kernel
kernel-core
-dracut-config-rescue
-plymouth
qemu-guest-agent
-firewalld
-geolite2-country
-geolite2-city
%end
```

* Comp: [comps-f35.xml.in](https://pagure.io/fedora-comps/blob/main/f/comps-f35.xml.in)

```xml
  <environment>
    <id>cloud-server-environment</id>
    <_name>Fedora Cloud Server</_name>
    <_description>A server install with components needed to run in a cloud environment.</_description>
    <grouplist>
      <groupid>cloud-bootloader-tools</groupid>
      <groupid>cloud-server</groupid>
      <groupid>core</groupid>
    </grouplist>
    <optionlist>
      ...
    </optionlist>
  </environment>
```

Then it adds the following RPMS, through cloud-init:

* `fuse-sshfs`

* `iptables`

## Processes

```text
systemd?????????NetworkManager?????????2*[{NetworkManager}]
        ??????2*[agetty]
        ??????auditd?????????{auditd}
        ??????chronyd
        ??????dbus-broker-lau?????????dbus-broker
        ??????lima-guestagent?????????4*[{lima-guestagent}]
        ??????sshd?????????sshd?????????sshd?????????bash?????????pstree
        ???                    ??????2*[sshfs?????????3*[{sshfs}]]
        ??????sssd?????????sssd_be
        ???      ??????sssd_nss
        ??????systemd?????????(sd-pam)
        ??????systemd-homed
        ??????systemd-hostnam
        ??????systemd-journal
        ??????systemd-logind
        ??????systemd-oomd
        ??????systemd-resolve
        ??????systemd-udevd
        ??????systemd-userdbd?????????3*[systemd-userwor]
```

## CentOS

[CentOS Stream](https://www.centos.org/centos-stream/) freezes an older version of Fedora.

It will eventually become Red Hat Enterprise Linux.

Except for the image URLs, it is same:

`limactl start centos-podman.yaml`

Fedora | CentOS |
------ | ------ |
fc28   | el8    |
fc34   | el9    |

## Rawhide

[Rawhide](https://docs.fedoraproject.org/en-US/releases/rawhide/) is the development version of Fedora.

It is "rollin', rollin', rollin", before Fedora releases.

Except for the image URLs, it is same:

`limactl start rawhide-podman.yaml`

## Comparison

### docker

See <https://docs.docker.com/engine/install/fedora/>

`limactl start fedora-docker.yaml`

`limactl shell fedora-docker docker`

`limactl shell fedora-docker sudo docker`

### nerdctl

`nerdctl` is a client for `containerd` and `buildkitd`

`limactl start fedora-nerdctl.yaml`

`limactl shell fedora-nerdctl nerdctl`

`limactl shell fedora-nerdctl sudo nerdctl`

### kubernetes

`limactl --debug start fedora-kubernetes.yaml`

`limactl shell fedora-kubernetes sudo kubectl`

### nomad

`limactl --debug start fedora-nomad.yaml`

`limactl shell fedora-nomad nomad`

## Examples

lima default:

[examples/ubuntu.yaml](https://github.com/lima-vm/lima/tree/master/examples/ubuntu.yaml) (the `default`) runs **containerd** with **ubuntu**

[examples/fedora.yaml](https://github.com/lima-vm/lima/tree/master/examples/fedora.yaml) runs containerd (not podman) with fedora

> "The goal of Lima is to promote containerd including nerdctl to Mac users"

Container engines:

[examples/docker.yaml](https://github.com/lima-vm/lima/tree/master/examples/docker.yaml) runs docker (not podman) with ubuntu (not fedora)

[examples/podman.yaml](https://github.com/lima-vm/lima/tree/master/examples/podman.yaml) runs podman with ubuntu (not fedora)

Container orchestration:

[examples/k8s.yaml](https://github.com/lima-vm/lima/tree/master/examples/k8s.yaml) runs kubernetes by using containerd with ubuntu

[examples/nomad.yaml](https://github.com/lima-vm/lima/tree/master/examples/nomad.yaml) runs nomad by using containerd with ubuntu

So we learn that:

1) containerd is the (new) standard container runtime

2) ubuntu is (still) the standard cloud linux distribution

Or something. ????
