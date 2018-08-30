---
layout: post
title:  "Docker storage driver"
date:   2018-08-30 12:00:00
author: Dark
categories: Docker
tags: docker
---

Storage driver 를 효율적으로 사용하기 위해서는 Docker 가 image 를 build 하고 저장하는 방법과 이러한 image 들을 container 들에서 어떻게 사용되고 있는지 아는 것이 중요합니다.
이 정보를 사용하여 application 의 데이터를 유지하고 성능 문제를 피할 수 있는 가장 좋은 방법을 에 대한 정보를 얻을 수 있습니다.

Storage driver 는 Container 의 쓰기 가능한 Layer 에 데이터를 생성할 수 있습니다. 파일들은 container 가 중지된 이후에는 유지되지 않으며 read 와 write 의 속도가 낮습니다. Volume 을 사용하여 데이터를 보존하고 성능을 향상시키는 방법을 알아봅시다.

### Images and layers

Docker 이미지는 일련의 Layer 들로 구성됩니다. 각각의 Layer 는 Dockerfile 의 명령어들에 의해 생성됩니다.
각 Layer 의 가장 마지막의 Layer 를 제외하고는 읽기 전용입니다. 다음의 Dockerfile 을 참조하십시오: 

{% highlight bash %}
FROM ubuntu:15.04
COPY . /app
RUN make /app
CMD python /app/app.py
{% endhighlight %}

이 Dockerfile 은 4 개의 layer 를 생성하는 command 를 포함하고 있습니다.  
FROM 문은 `ubuntu:15.04` image 로부터 layer 를 생성하는 것으로 시작합니다.  
COPY 문은 Docker client 의 현재 디렉토리 경로의 파일들을 추가합니다.  
RUN 문은 make command 를 사용하여 application 을 빌드합니다.  
CMD 문은 마지막 layer 는 container 내부에서 어떤 command 를 실행할지 지정합니다.  

각각의 layer 는 이전 layer 로부터 차이점의 집합입니다.
Layer 들은 서로 층을 이루며 쌓여져 있습니다.
Docker 는 새로운 container 를 생성할 때, 기본적인 layer 위에 새로운 writable layer 를 추가합니다.
이 Layer 는 종종 “container layer” 라고 합니다. 
실행중인 컨테이너에 대한 모든 변경 사항(새로운 파일 작성, 기존 파일 수정 및 삭제)은 `Thin R/W layer == container layer` 에 기록됩니다.
아래의 다이어그램은 Ubuntu 15.04 image 를 기반으로 한 container 를 보여줍니다.

<img src="/assets/post/container-layers.png" title="Container layers">

Storage driver 는 이러한 layer 들과 상호 작용하는 방법에 대한 세부적인 부분을 처리합니다. 
다양한 Storage driver 를 사용할 수 있으며 상황에 따라 장점과 단점을 가지고 있습니다.

### Reference  
[docker-storage-driver]

[docker-storage-driver]:      https://docs.docker.com/storage/storagedriver