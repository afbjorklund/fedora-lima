# Lima

See <https://github.com/lima-vm/lima>

Requires Lima version 0.7.0 or later

## Background

Podman will by default create virtual machines using [Fedora CoreOS](https://getfedora.org/en/coreos/).

But it is also possible to create virtual machines using [Fedora Cloud](https://cloud.fedoraproject.org)...

## Usage

Run `limactl start fedora-podman.yaml` to create a Lima instance named "fedora-podman".

To open a shell, run `limactl shell fedora-podman bash` or `LIMA_INSTANCE=fedora-podman lima`.

### user

limactl shell fedora-podman podman

```console
$ export LIMA_INSTANCE=fedora-podman
$ lima podman version
...
$ lima podman info | grep rootless
    rootless: true
```

The remote socket is listening as well, if needed:

```console
$ lima systemctl --user status podman.socket
...
$ lima podman --remote version
Client
...

Server
...
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
...
$ lima sudo podman info | grep rootless
    rootless: false
```

The remote socket is listening as well, if needed:

```console
$ lima sudo systemctl --system status podman.socket
...
$ lima sudo podman --remote version
Client
...

Server
...
$ lima sudo podman --remote info | grep -A2 remoteSocket
  remoteSocket:
    exists: true
    path: /run/podman/podman.sock
```

## Root

By default, podman will use user unless switching socket:

`export CONTAINER_HOST=unix:/run/podman/podman.sock`

But the system service requires root, for instance `sudo`.

`Error: unable to connect to Podman socket: Get "http://d/v3.4.0/libpod/_ping": dial unix /run/podman/podman.sock: connect: permission denied`

Alternatively, one can use a root-equivalent group:

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

## Processes

```text
systemd─┬─NetworkManager───2*[{NetworkManager}]
        ├─2*[agetty]
        ├─auditd───{auditd}
        ├─chronyd
        ├─dbus-broker-lau───dbus-broker
        ├─lima-guestagent───4*[{lima-guestagent}]
        ├─sshd───sshd───sshd─┬─bash───pstree
        │                    └─2*[sshfs───3*[{sshfs}]]
        ├─sssd─┬─sssd_be
        │      └─sssd_nss
        ├─systemd───(sd-pam)
        ├─systemd-homed
        ├─systemd-hostnam
        ├─systemd-journal
        ├─systemd-logind
        ├─systemd-oomd
        ├─systemd-resolve
        ├─systemd-udevd
        └─systemd-userdbd───3*[systemd-userwor]
```

## Examples

[examples/fedora.yaml](examples/fedora.yaml) runs containerd (not podman) with fedora

[examples/podman.yaml](examples/podman.yaml) runs podman with ubuntu (not fedora)
