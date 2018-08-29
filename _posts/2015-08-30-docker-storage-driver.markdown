---
layout: post
title:  "Docker storage driver"
date:   2018-08-30 12:00:00
author: Dark
categories: Docker
tags: docker
---

To use storage drivers effectively, it’s important to know how Docker builds and stores images, how these images are used by containers. You can use this information to make informed choices about the best way to persist data from your applications and avoid performance problems along the way.
Storage drivers allow you to create data in the writable layer of your container. The files won’t be persisted after the container stops, and both read and write speeds are low.
Learn how to use volumes to persist data and improve performance.

### Images and layers

Docker 이미지는 일련의 Layer 들로 구성됩니다. 각각의 Layer 는 Dockerfile 의 명령어들에 의해 생성됩니다.
각 Layer 의 가장 마지막의 Layer 를 제외하고는 읽기 전용입니다. 다음의 Dockerfile 을 참조하십시오: 

{% highlight bash %}
FROM ubuntu:15.04
COPY . /app
RUN make /app
CMD python /app/app.py
{% endhighlight %}

### Reference  
[docker-storage-driver]

[docker-storage-driver]:      https://docs.docker.com/storage/storagedriver