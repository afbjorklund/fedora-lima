# Lima

See <https://github.com/lima-vm/lima>

Requires Lima version 0.7.0 or later

## Usage
Run `limactl start fedora-podman.yaml` to create a Lima instance named "fedora-podman".

To open a shell, run `limactl shell fedora-podman bash` or `LIMA_INSTANCE=fedora-podman lima`.

### user

limactl shell fedora-podman podman

```console
$ export LIMA_INSTANCE=fedora-podman
$ lima podman info | grep rootless
    rootless: true
```

The remote socket is listening as well, if needed:

```console
$ lima systemctl --user status podman.socket
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
$ lima sudo podman info | grep rootless
    rootless: false
```

The remote socket is listening as well, if needed:

```console
$ lima sudo systemctl --system status podman.socket
...
$ lima sudo podman --remote info | grep -A2 remoteSocket
  remoteSocket:
    exists: true
    path: /run/podman/podman.sock
```

## Root

The system service requires root, for instance `sudo`.

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

