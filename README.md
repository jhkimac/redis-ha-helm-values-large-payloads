# values.yaml for configuring redis-ha helm chart for large payloads

Reference [redis-ha chart](https://github.com/helm/charts/tree/master/stable/redis-ha) for installation details.

## Problems

Redis High Availability on k8s had issues of **NOREPLICAS: Not Enough Good Replicas To Write** because of the large payload size (up to 512MB per entry).

The replicas (slave redis) had alternating error messages including the following:

### Low somaxconn

```bash
# WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
```

By default, somaxconn is not whitelisted by k8s and thus cannot be enabled through the pod spec configuration.
This configuration needs to be added to initContainer of helm deployment.

### Transparent Huge Page (THP)

```bash
# WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled
```

This also needs to be added to initContainer of helm deployment.

### Client Output Buffer Limits

```bash
# Client id={some-id} addr={some addr} fd=533 name= age=127 idle=127 flags=S db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=13997 oll=1227 omem=192281275 events=rw cmd=psync scheduled to be closed ASAP for overcoming of output buffer limits.
```

Replication fails because of the output buffer limits, causing NOREPLICAS error in the master.
This configuration can be added to `redis.config` in helm deployment.

## initContainer settings

In order to trigger initContainer in the redis-ha StatefulSet, add the following in values.yaml.

```YAML
sysctlImage:
  enabled: true
  command:
    - /bin/sh
    - -c
    - |-
      install_packages systemd
      sysctl -w net.core.somaxconn=10000
      echo never > /host-sys/kernel/mm/transparent_hugepage/enabled
  mountHostSys: true
```

## redis.config && sentinel.config

```YAML
redis:
  config:
    min-slaves-to-write: 1
    repl-timeout: '60'
    client-output-buffer-limit: 'slave 836870912 836870912 0' # Can use 'slave 0 0 0' to remove the limit
sentinel:
  config:
    down-after-milliseconds: 30000
```
