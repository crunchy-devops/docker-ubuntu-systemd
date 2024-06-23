Docker Ubuntu Systemd
=====================

This Dockerfile can build containers capable to use systemd.

Branches
--------

This repository has multiple tags that relate to Ubuntu versions.

|Ubuntu Version|Docker image tag|
|------------------|--------------------|
|20.04 (focal)     |2004, focal         |
|22.04 (jammy)     |2204, jammy         |
|24.04 (noble)     |2404, noble, latest |

Manually starting
-----------------

```shell
docker run \
  --tty \
  --privileged \
  --volume /sys/fs/cgroup:/sys/fs/cgroup:ro \
  mullholland/docker-ubuntu-systemd
```
