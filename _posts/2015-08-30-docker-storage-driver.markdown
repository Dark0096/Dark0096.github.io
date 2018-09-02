---
layout: post
title:  "Docker storage driver"
date:   2018-08-30 12:00:00
author: Dark
categories: Docker
tags: docker
---

Storage driver 를 효율적으로 사용하기 위해서는 Docker 가 image 를 build 하고 저장하는 방법과 이러한 image 들을 container 들에서 어떻게 사용되고 있는지 아는 것이 중요합니다.
이 정보를 사용하여 application 의 데이터를 유지하고 성능 문제를 피할 수 있는 가장 좋은 방법에 대한 정보를 얻을 수 있습니다.

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
<code>  
FROM 문은 `ubuntu:15.04` image 로부터 layer 를 생성하는 것으로 시작합니다.<br/>  
COPY 문은 Docker client 의 현재 디렉토리 경로의 파일들을 추가합니다.<br/>
RUN 문은 make command 를 사용하여 application 을 빌드합니다.<br/>
CMD 문은 마지막 layer 는 container 내부에서 어떤 command 를 실행할지 지정합니다.
</code>  

각각의 layer 는 이전 layer 로부터 차이점의 집합입니다.
Layer 들은 서로 층을 이루며 쌓여져 있습니다.
Docker 는 새로운 container 를 생성할 때, 기본적인 layer 위에 새로운 writable layer 를 추가합니다.
이 Layer 는 종종 “container layer” 라고 합니다. 
실행중인 컨테이너에 대한 모든 변경 사항(새로운 파일 작성, 기존 파일 수정 및 삭제)은 `Thin R/W layer == container layer` 에 기록됩니다.
아래의 다이어그램은 Ubuntu 15.04 image 를 기반으로 한 container 를 보여줍니다.

<img src="https://raw.githubusercontent.com/Dark0096/Dark0096.github.io/master/assets/post/container-layers.png" title="Container layers">

Storage driver 는 이러한 layer 들과 상호 작용하는 방법에 대한 세부적인 부분을 처리합니다. 
다양한 Storage driver 를 사용할 수 있으며 상황에 따라 장점과 단점을 가지고 있습니다.


### Container and layers
Container 와 image 사이의 주된 차이점은 가장 상위에 writable layer 입니다.
Container 로의 새로운 데이터 또는 기존의 데이터를 수정하는 모든 쓰기는 이 writable layer 에 저장됩니다. 
Container 가 삭제되었을 때, writable layer 또한 삭제됩니다. 
기본 이미지는 변경되지 않습니다.

각각의 Container 는 자신만의 writable container layer 를 가지고 있기 때문에, 모든 변경분은 이 writable container layer 에만 저장됩니다.
여러 Container 들은 동일한 기본 image 에 대한 접근을 공유할 수 있지만 자체 데이터 상태를 가질 수 있습니다. 
아래 다이어그램은 동일한 Ubuntu 15.04 이미지를 공유하는 여러 컨테이너를 보여줍니다. 
<img src="https://raw.githubusercontent.com/Dark0096/Dark0096.github.io/master/assets/post/sharing-layers.png" title="Container layers">

`Note: 똑같은 데이터에 대한 공유 액세스를 위해 여러 개의 이미지가 필요한 경우 이 데이터를 Docker 볼륨에 저장하고 컨테이너에 마운트하십시오.`

Docker 는 Storage driver 를 사용하여 image layer 들과 writable container layer 의 내용을 관리합니다. 
각각의 Storage driver 는 구현을 다르게 처리하지만 모든 driver 는 stackable image layer 와 copy-on-write (CoW) 전략을 사용합니다.

### Container size on disk
실행 중인 Container 의 대략적인 사이즈를 보기 위해서 `docker ps -s` command 를 사용할 수 있습니다. `docker ps` command 를 실행했을때와는 다르게 두 개의 size 관련 Column 을 확인할 수 있습니다.

`size`: 각 container 의 writable layer 에 사용되는 데이터의 양 (on disk) 

`virtual size`: Container 가 사용하는 읽기 전용 이미지 데이터와 컨테이너의 쓰기 가능한 레이어 크기에 사용되는 데이터의 양. 여러 container 들은 일부 또는 모든 read-only 이미지 데이터를 공유할 수 있습니다.
                동일한 이미지에서 시작된 두 개의 컨테이너는 읽기 전용 데이터의 100% 를 공유하는 반면 서로 다른 이미지를 가진 두 개의 컨테이너는 공통된 레이어를 공유합니다. 그러므로, virtual size 를 계산할 수 없습니다. 이것은 잠재적으로 중요하지 않은 총 디스크 사용량을 조금 높게 계산합니다.

모든 실행 중인 container 들의 총 disk 사용량은 각 container 의 크기와 virtual size 값들의 일부 조합입니다.

만일 여러 container 들이 동일한 이미지로부터 시작되었다면, 이 container 들의 디스크 total size 는 `container 들의 size + one image size (virtual size-size)` 이다.
    
위의 계산 공식은 컨테이너가 디스크 공간을 차지할 수 있는 다음과 같은 추가 방법을 계산하지 않았습니다:

<ul>
  <li>json-file logging driver 를 사용하는 경우 로그 파일에 사용되는 디스크 공간. Container 가 대량의 로깅 데이터를 생성하고 로그 로테이션이 구성되어 있지 않으면 이 작업이 중요하지 않을 수 있습니다.</li>
  <li>Container 가 사용하는 Volume 및 Mount.</li>
  <li>일반적으로 크기가 작은 container 구성 파일에 사용되는 디스크 공간.</li>
  <li>디스크에 기록 된 메모리 ( swapping 이 활성화 된 경우).</li>
  <li>실험적인 Checkpoints/restore 기능을 사용하는 경우.</li>
</ul>

### The copy-on-write (CoW) strategy
copy-on-write 는 파일을 공유하고 복사하여 최대한의 효율성을 높이는 전략입니다.
파일 또는 디렉토리가 이미지의 하위 레이어에 존재하고 다른 레이어 (쓰기 가능한 레이어 포함)에 읽기 액세스가 필요한 경우 기존 파일을 사용합니다.
처음으로 다른 레이어가 파일을 수정해야 할 때 (이미지를 만들거나 컨테이너를 실행할 때) 파일은 해당 레이어에 복사되고 수정됩니다.
이렇게하면 I/O 및 각 후속 레이어의 크기가 최소화됩니다.
이러한 장점은 아래에서 더 자세히 설명됩니다.

공유로 인하여 더 작은 이미지를 만들 수 있게 됩니다.
docker pull 을 사용하여 저장소에서 이미지를 다운받거나 아직 로컬에 존재하지 않는 이미지에서 컨테이너를 만들 때 각 레이어가 별도로 Docker 의 로컬 저장 영역에 저장됩니다 (일반적으로 /var/lib/docker). 
다음 예제에서 이러한 레이어가 표시되는지 확인할 수 있습니다.
{% highlight bash %}
$ docker pull ubuntu:15.04

15.04: Pulling from library/ubuntu
1ba8ac955b97: Pull complete
f157c4e5ede7: Pull complete
0b7e98f84c4c: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:5e279a9df07990286cce22e1b0f5b0490629ca6d187698746ae5e28e604a640e
Status: Downloaded newer image for ubuntu:15.04
{% endhighlight %}
이 layer 들 각각은 Docker 호스트의 로컬 스토리지 영역 내부의 자체 디렉토리에 저장됩니다.
파일 시스템의 레이어를 검사하려면 /var/lib/docker/\<storage-driver>/layers/의 내용을 나열하십시오.
이 예제에서는 default storage driver 인 aufs 를 사용합니다.

{% highlight bash %}
$ ls /var/lib/docker/aufs/layers
1d6674ff835b10f76e354806e16b950f91a191d3b471236609ab13a930275e24
5dbb0cbe0148cf447b9464a358c1587be586058d9a4c9ce079320265e2bb94e7
bef7199f2ed8e86fa4ada1309cfad3089e0542fec8894690529e4c04a7ca2d73
ebf814eccfe98f2704660ca1d844e4348db3b5ccc637eb905d4818fbfb00a06a
The directory names do not correspond to the layer IDs (this has been true since Docker 1.10).
{% endhighlight %}

자 이제 두 개의 Dockerfile 이 있다고 상상해보겠습니다. 첫 번째 이미지를 사용하여 acme/my-base-image:1.0 이라는 이미지를 만듭니다.

{% highlight bash %}
FROM ubuntu:16.10
COPY . /app
{% endhighlight %}
두 번째 것은 acme/my-base-image:1.0을 기반으로 하지만 몇 가지 추가 레이어가 있습니다:

{% highlight bash %}
FROM acme/my-base-image:1.0
CMD /app/hello.sh
{% endhighlight %}
두 번째 이미지에는 첫 번째 이미지의 모든 레이어와 CMD 명령이 포함 된 새 레이어 및 읽기/쓰기 컨테이너 레이어가 포함됩니다. 
Docker 는 이미 첫 번째 이미지의 모든 레이어를 가지고 있으므로 다시 가져올 필요가 없습니다. 두 이미지는 공통된 레이어를 공유합니다.

두 Dockerfile 에서 이미지를 빌드하는 경우 docker image ls 및 docker history 명령을 사용하여 공유 레이어의 암호화 ID가 동일한 지 확인할 수 있습니다.

<ol>
  <li>새 디렉토리 cow-test/ 를 만들고 그 디렉토리로 변경하십시오.</li>
  <li>
  cow-test/ 에서 다음 내용으로 새 파일을 만듭니다:
  {% highlight bash %}
  #!/bin/sh
  echo "Hello world"
  {% endhighlight %}
  </li>
  <li>
  파일을 저장하고 실행 가능한 상태로 만듭니다:
  {% highlight bash %}
  $ chmod +x hello.sh
  {% endhighlight %}
  </li>
  <li>
  위의 첫 번째 Dockerfile 의 내용을 Dockerfile.base 라는 새 파일에 복사합니다.
  </li>
  <li>
  위의 두 번째 Dockerfile 의 내용을 Dockerfile 이라는 새 파일에 복사합니다.
  </li>
  <li>
  cow-test/ 디렉토리 내에서 첫 번째 이미지를 빌드하십시오. 
  Command 를 포함하는 것을 잊지 마십시오. 
  Docker 가 이미지에 추가해야 할 파일을 찾을 위치를 알려주는 PATH 가 설정됩니다.
  {% highlight bash %}
  $ docker build -t acme/my-base-image:1.0 -f Dockerfile.base .
  
  Sending build context to Docker daemon  4.096kB
  Step 1/2 : FROM ubuntu:16.10
   ---> 31005225a745
  Step 2/2 : COPY . /app
   ---> Using cache
   ---> bd09118bcef6
  Successfully built bd09118bcef6
  Successfully tagged acme/my-base-image:1.0
  {% endhighlight %}
  </li>
  <li>
  두 번째 이미지를 빌드합니다.
  {% highlight bash %}
  $ docker build -t acme/my-final-image:1.0 -f Dockerfile .
  
  Sending build context to Docker daemon  4.096kB
  Step 1/2 : FROM acme/my-base-image:1.0
   ---> bd09118bcef6
  Step 2/2 : CMD /app/hello.sh
   ---> Running in a07b694759ba
   ---> dbf995fc07ff
  {% endhighlight %}  
  </li>
  <li>
  Removing intermediate container a07b694759ba
  Successfully built dbf995fc07ff
  Successfully tagged acme/my-final-image:1.0
  Check out the sizes of the images:
  {% highlight bash %}
  $ docker image ls
  
  REPOSITORY                         TAG                     IMAGE ID            CREATED             SIZE
  acme/my-final-image                1.0                     dbf995fc07ff        58 seconds ago      103MB
  acme/my-base-image                 1.0                     bd09118bcef6        3 minutes ago       103MB
  {% endhighlight %}
  </li>
  <li>
  Check out the layers that comprise each image:
  
  {% highlight bash %}
  $ docker history bd09118bcef6
  IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
  bd09118bcef6        4 minutes ago       /bin/sh -c #(nop) COPY dir:35a7eb158c1504e...   100B                
  31005225a745        3 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
  <missing>           3 months ago        /bin/sh -c mkdir -p /run/systemd && echo '...   7B                  
  <missing>           3 months ago        /bin/sh -c sed -i 's/^#\s*\(deb.*universe\...   2.78kB              
  <missing>           3 months ago        /bin/sh -c rm -rf /var/lib/apt/lists/*          0B                  
  <missing>           3 months ago        /bin/sh -c set -xe   && echo '#!/bin/sh' >...   745B                
  <missing>           3 months ago        /bin/sh -c #(nop) ADD file:eef57983bd66e3a...   103MB      
  
  $ docker history dbf995fc07ff
  IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
  dbf995fc07ff        3 minutes ago       /bin/sh -c #(nop)  CMD ["/bin/sh" "-c" "/a...   0B                  
  bd09118bcef6        5 minutes ago       /bin/sh -c #(nop) COPY dir:35a7eb158c1504e...   100B                
  31005225a745        3 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
  <missing>           3 months ago        /bin/sh -c mkdir -p /run/systemd && echo '...   7B                  
  <missing>           3 months ago        /bin/sh -c sed -i 's/^#\s*\(deb.*universe\...   2.78kB              
  <missing>           3 months ago        /bin/sh -c rm -rf /var/lib/apt/lists/*          0B                  
  <missing>           3 months ago        /bin/sh -c set -xe   && echo '#!/bin/sh' >...   745B                
  <missing>           3 months ago        /bin/sh -c #(nop) ADD file:eef57983bd66e3a...   103MB  
  {% endhighlight %}
  </li>
</ol>

Notice that all the layers are identical except the top layer of the second image. All the other layers are shared between the two images, and are only stored once in /var/lib/docker/. The new layer actually doesn’t take any room at all, because it is not changing any files, but only running a command.

Note: The <missing> lines in the docker history output indicate that those layers were built on another system and are not available locally. This can be ignored.

Copying makes containers efficient
When you start a container, a thin writable container layer is added on top of the other layers. Any changes the container makes to the filesystem are stored here. Any files the container does not change do not get copied to this writable layer. This means that the writable layer is as small as possible.

When an existing file in a container is modified, the storage driver performs a copy-on-write operation. The specifics steps involved depend on the specific storage driver. For the default aufs driver and the overlay and overlay2 drivers, the copy-on-write operation follows this rough sequence:

Search through the image layers for the file to update. The process starts at the newest layer and works down to the base layer one layer at a time. When results are found, they are added to a cache to speed future operations.

Perform a copy_up operation on the first copy of the file that is found, to copy the file to the container’s writable layer.

Any modifications are made to this copy of the file, and the container cannot see the read-only copy of the file that exists in the lower layer.

Btrfs, ZFS, and other drivers handle the copy-on-write differently. You can read more about the methods of these drivers later in their detailed descriptions.

Containers that write a lot of data consume more space than containers that do not. This is because most write operations consume new space in the container’s thin writable top layer.

Note: for write-heavy applications, you should not store the data in the container. Instead, use Docker volumes, which are independent of the running container and are designed to be efficient for I/O. In addition, volumes can be shared among containers and do not increase the size of your container’s writable layer.

A copy_up operation can incur a noticeable performance overhead. This overhead is different depending on which storage driver is in use. Large files, lots of layers, and deep directory trees can make the impact more noticeable. This is mitigated by the fact that each copy_up operation only occurs the first time a given file is modified.

To verify the way that copy-on-write works, the following procedures spins up 5 containers based on the acme/my-final-image:1.0 image we built earlier and examines how much room they take up.

Note: This procedure doesn’t work on Docker for Mac or Docker for Windows.

From a terminal on your Docker host, run the following docker run commands. The strings at the end are the IDs of each container.
{% highlight bash %}
$ docker run -dit --name my_container_1 acme/my-final-image:1.0 bash \
  && docker run -dit --name my_container_2 acme/my-final-image:1.0 bash \
  && docker run -dit --name my_container_3 acme/my-final-image:1.0 bash \
  && docker run -dit --name my_container_4 acme/my-final-image:1.0 bash \
  && docker run -dit --name my_container_5 acme/my-final-image:1.0 bash

  c36785c423ec7e0422b2af7364a7ba4da6146cbba7981a0951fcc3fa0430c409
  dcad7101795e4206e637d9358a818e5c32e13b349e62b00bf05cd5a4343ea513
  1e7264576d78a3134fbaf7829bc24b1d96017cf2bc046b7cd8b08b5775c33d0c
  38fa94212a419a082e6a6b87a8e2ec4a44dd327d7069b85892a707e3fc818544
  1a174fc216cccf18ec7d4fe14e008e30130b11ede0f0f94a87982e310cf2e765
{% endhighlight %}
Run the docker ps command to verify the 5 containers are running.

{% highlight bash %}
CONTAINER ID      IMAGE                     COMMAND     CREATED              STATUS              PORTS      NAMES
1a174fc216cc      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_5
38fa94212a41      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_4
1e7264576d78      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_3
dcad7101795e      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_2
c36785c423ec      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_1
{% endhighlight %}
List the contents of the local storage area.
{% highlight bash %}
$ sudo ls /var/lib/docker/containers

1a174fc216cccf18ec7d4fe14e008e30130b11ede0f0f94a87982e310cf2e765
1e7264576d78a3134fbaf7829bc24b1d96017cf2bc046b7cd8b08b5775c33d0c
38fa94212a419a082e6a6b87a8e2ec4a44dd327d7069b85892a707e3fc818544
c36785c423ec7e0422b2af7364a7ba4da6146cbba7981a0951fcc3fa0430c409
dcad7101795e4206e637d9358a818e5c32e13b349e62b00bf05cd5a4343ea513
{% endhighlight %}
Now check out their sizes:

{% highlight bash %}
$ sudo du -sh /var/lib/docker/containers/*

32K  /var/lib/docker/containers/1a174fc216cccf18ec7d4fe14e008e30130b11ede0f0f94a87982e310cf2e765
32K  /var/lib/docker/containers/1e7264576d78a3134fbaf7829bc24b1d96017cf2bc046b7cd8b08b5775c33d0c
32K  /var/lib/docker/containers/38fa94212a419a082e6a6b87a8e2ec4a44dd327d7069b85892a707e3fc818544
32K  /var/lib/docker/containers/c36785c423ec7e0422b2af7364a7ba4da6146cbba7981a0951fcc3fa0430c409
32K  /var/lib/docker/containers/dcad7101795e4206e637d9358a818e5c32e13b349e62b00bf05cd5a4343ea513
{% endhighlight %}
Each of these containers only takes up 32k of space on the filesystem.

Not only does copy-on-write save space, but it also reduces start-up time. When you start a container (or multiple containers from the same image), Docker only needs to create the thin writable container layer.

If Docker had to make an entire copy of the underlying image stack each time it started a new container, container start times and disk space used would be significantly increased. This would be similar to the way that virtual machines work, with one or more virtual disks per virtual machine.

### Reference  
[docker-storage-driver]

[docker-storage-driver]:      https://docs.docker.com/storage/storagedriver