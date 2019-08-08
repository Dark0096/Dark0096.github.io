---
layout: post
title:  "kafka cluster performance tuning"
date:   2019-06-17 23:00:00
author: Dark
categories: kafka
tags: kafka
---

# Performance 튜닝하기에 앞서

Kafka, Cassandra, Elasticsearch 와 같은 Stateful 한 System 을 운영한다면 OS 의 System resource 를 잘 사용할 수 있도록 신경써서 설정을 해주어야 합니다. 
이때에 보통 마주치는 것이 `Ulimit` (User limit) 입니다. `Ulimit` 은 실행 중인 Process 에게 얼마만큼의 system resource 를 최대로 사용하도록 허용하는지 그 값을 설정하고, 보여줄 수 있는 명령어입니다.
`ulimit -a` command 를 실행해보면 아래와 같이 Terminal 에 process 에게 얼마만큼의 Resource 를 할당해줄 수 있는지에 대해 확인할 수 있습니다.

```
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 31343
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 31343
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
``` 

만일 여러분이 Docker 를 사용하고 계시다면, Docker daemon 에 system resource 를 Container 의 Type 에 맞게 재설정하는 것이 필요합니다.
Docker daemon 에 설정한 값이 곧 Container 가 사용할 수 있는 system resource 와 동일하기 때문입니다. 물론, Container 의 resource 를 제한하지 않은 경우에서만 유용합니다.
이때에 Docker daemon 을 보통 `init process` 를 통하여 실행하게 되는데 이는 OS 의 version 에 따라 다른 `init process` 가 존재합니다.

각 `init process` 별로 어떻게 docker daemon 의 system resource 에 대해 설정할 수 있는지 확인해봅시다.

## Upstart (Ubuntu 14.x)

`/etc/init/docker.conf`

```
limit memlock unlimited unlimited
limit nofile unlimited unlimited
limit nproc unlimited unlimited 
limit rss unlimited unlimited
limit fsize unlimited unlimited
limit cpu unlimited unlimited
limit as unlimited unlimited
```

`sudo service docker restart`

`cat /proc/2238/limits`

### Reference
[Setting ulimits for docker process and containers in Ubuntu]  
[Kafka performance tuning guide]
[ulimit]

[Setting ulimits for docker process and containers in Ubuntu]:      http://tostr.pl/blog/setting-ulimits-for-docker-process-2/
[Kafka performance tuning guide]:                                   https://kafka.apache.org/documentation/#hwandos
[ulimit] https://linux.die.net/man/2/setrlimit