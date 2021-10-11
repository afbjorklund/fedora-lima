# Lima

See <https://github.com/lima-vm/lima>

## Usage
Run `limactl start fedora-podman.yaml` to create a Lima instance named "fedora-podman".

To open a shell, run `limactl shell fedora-podman bash` or `LIMA_INSTANCE=fedora-podman lima`.

### user

limactl shell fedora-podman podman

```console
$ limactl shell fedora-podman podman info | grep rootless
    rootless: true
```

### system

limactl shell fedora-podman sudo podman

```console
$ limactl shell fedora-podman sudo podman info | grep rootless
    rootless: false
```

## Examples

[examples/fedora.yaml](examples/fedora.yaml) runs containerd (not podman) with fedora

[examples/podman.yaml](examples/podman.yaml) runs podman with ubuntu (not fedora)

