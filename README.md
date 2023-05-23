Docker Ubuntu Systemd
=====================

This Dockerfile can build containers capable to use systemd.

Branches
--------

This repository has multiple tags that relate to Ubuntu versions.

|Ubuntu Version|Docker image tag|
|------------------|--------------------|
|18.04 (bionic)    |1804, bionic        |
|20.04 (focal)     |2004, focal         |
|22.04 (jammy)     |2204, jammy, latest |

Manually starting
-----------------

```shell
docker run \
  --tty \
  --privileged \
  --volume /sys/fs/cgroup:/sys/fs/cgroup:ro \
  mullholland/docker-ubuntu-systemd
```
