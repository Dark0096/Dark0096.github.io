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
  Removing intermediate container a07b694759ba
  Successfully built dbf995fc07ff
  Successfully tagged acme/my-final-image:1.0
  {% endhighlight %}  
  </li>
  <li>
  이미지의 크기를 확인하십시오:
  {% highlight bash %}
  $ docker image ls
  
  REPOSITORY                         TAG                     IMAGE ID            CREATED             SIZE
  acme/my-final-image                1.0                     dbf995fc07ff        58 seconds ago      103MB
  acme/my-base-image                 1.0                     bd09118bcef6        3 minutes ago       103MB
  {% endhighlight %}
  </li>
  <li>
  각 이미지를 구성하는 레이어를 확인하십시오:
  
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

모든 레이어는 두 번째 이미지의 맨 위 레이어를 제외하고 동일하다는 점에 유의하십시오. 
다른 모든 레이어는 두 이미지간에 공유되며 /var/lib/docker/ 에 한 번만 저장됩니다. 
새로운 레이어는 파일을 변경하지 않고 명령만 실행하기 때문에 실제로 아무런 공간도 차지하지 않습니다.

`Note: docker 히스토리 출력의 <missing> 행은 해당 계층이 다른 시스템에서 작성되었으며 로컬에서 사용할 수 없음을 나타냅니다. 이것은 무시해도 됩니다.`

### Copying makes containers efficient
컨테이너를 시작할 때 thin writable container layer 가 다른 레이어 위에 추가됩니다.
컨테이너가 파일 시스템을 변경하면 여기에 저장됩니다.
컨테이너가 변경하지 않는 파일은 이 쓰기 가능한 레이어에 복사되지 않습니다.
즉, thin writable container layer 는 최소한의 사이즈만을 갖고 있습니다.

컨테이너의 기존 파일이 수정되면 저장소 드라이버는 쓰기시 복사 작업을 수행합니다.
관련된 구체적인 단계는 특정 저장 장치 드라이버에 따라 다릅니다.
기본 `aufs` driver 와 `overlay` 및 `overlay2` driver 의 경우 copy-on-write 작업은 다음과 같은 대략적인 순서를 따릅니다.

<ul>
  <li>
  이미지 레이어를 검색하여 업데이트할 파일을 찾습니다.
  이 프로세스는 최신 레이어에서 시작하여 한 번에 기본 레이어로 한 레이어씩 작업합니다.
  결과가 발견되면 향후 작업 속도를 높이기 위해 캐시에 추가됩니다.
  </li>
  <li>
  찾은 파일의 첫 번째 복사본에 대해 copy_up 작업을 수행하여 파일을 컨테이너의 쓰기 가능한 계층에 복사합니다.
  </li>
  <li>
  이 파일 복사본을 수정하면 컨테이너는 하위 계층에 있는 파일의 읽기 전용 복사본을 볼 수 없습니다.
  </li>
</ul>

Btrfs, ZFS 및 다른 드라이버는 copy-on-write 를 다르게 처리합니다.
이 드라이버의 방법에 대해서는 나중에 자세히 설명합니다.

많은 양의 데이터를 쓰는 컨테이너는 그렇지 않은 컨테이너보다 더 많은 공간을 소비합니다.
이는 대부분의 쓰기 작업이 컨테이너의 thin writable 한 최상위 영역에서 새로운 공간을 사용하기 때문입니다.

`Note: 쓰기가 많은 응용 프로그램의 경우 컨테이너에 데이터를 저장하면 안됩니다. 대신 Docker 볼륨을 사용하십시오. Docker 볼륨은 실행중인 컨테이너와 독립적이며 I/O 에 효율적으로 설계되었습니다. 또한 볼륨을 컨테이너간에 공유할 수 있으며 컨테이너의 쓰기 가능한 레이어 크기를 늘리지는 않습니다.`

copy_up 조작은 현저한 성능 오버 헤드를 초래할 수 있습니다. 이 오버 헤드는 사용중인 스토리지 드라이버에 따라 다릅니다. 
대용량 파일, 많은 레이어 및 딥 디렉토리 트리를 사용하면 영향을 더 두드러지게 만들 수 있습니다. 
이는 각 copy_up 조작이 주어진 파일이 처음 수정될 때만 발생한다는 사실에 의해 완화됩니다.

copy-on-write 가 작동하는 방식을 확인하기 위해 다음 절차에서는 앞에서 작성한 acme/my-final-image:1.0 이미지를 기반으로 5 개의 컨테이너를 회전시키고 그들이 차지하는 공간을 조사합니다.

`Note: This procedure doesn’t work on Docker for Mac or Docker for Windows.`

<ol>
  <li>
  Docker 호스트의 터미널에서 다음 도커 실행 명령을 실행합니다. 끝에있는 문자열은 각 컨테이너의 ID입니다.
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
  </li>
  <li>
  docker ps 명령을 실행하여 5 개의 컨테이너가 실행 중인지 확인하십시오:
  
  {% highlight bash %}
  CONTAINER ID      IMAGE                     COMMAND     CREATED              STATUS              PORTS      NAMES
  1a174fc216cc      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_5
  38fa94212a41      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_4
  1e7264576d78      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_3
  dcad7101795e      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_2
  c36785c423ec      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_1
  {% endhighlight %}
  </li>
  <li>
  로컬 저장 영역의 내용을 나열하십시오:
  {% highlight bash %}
  $ sudo ls /var/lib/docker/containers
  
  1a174fc216cccf18ec7d4fe14e008e30130b11ede0f0f94a87982e310cf2e765
  1e7264576d78a3134fbaf7829bc24b1d96017cf2bc046b7cd8b08b5775c33d0c
  38fa94212a419a082e6a6b87a8e2ec4a44dd327d7069b85892a707e3fc818544
  c36785c423ec7e0422b2af7364a7ba4da6146cbba7981a0951fcc3fa0430c409
  dcad7101795e4206e637d9358a818e5c32e13b349e62b00bf05cd5a4343ea513
  {% endhighlight %}
  </li>
  <li>
  이제 크기를 확인하십시오:
  
  {% highlight bash %}
  $ sudo du -sh /var/lib/docker/containers/*
  
  32K  /var/lib/docker/containers/1a174fc216cccf18ec7d4fe14e008e30130b11ede0f0f94a87982e310cf2e765
  32K  /var/lib/docker/containers/1e7264576d78a3134fbaf7829bc24b1d96017cf2bc046b7cd8b08b5775c33d0c
  32K  /var/lib/docker/containers/38fa94212a419a082e6a6b87a8e2ec4a44dd327d7069b85892a707e3fc818544
  32K  /var/lib/docker/containers/c36785c423ec7e0422b2af7364a7ba4da6146cbba7981a0951fcc3fa0430c409
  32K  /var/lib/docker/containers/dcad7101795e4206e637d9358a818e5c32e13b349e62b00bf05cd5a4343ea513
  {% endhighlight %}
  이 컨테이너들 각각은 파일 시스템상의 32k 공간만을 차지합니다.
  </li>
</ol>

copy-on-write 는 공간을 절약 할뿐만 아니라 시작 시간을 줄여줍니다.
container (또는 동일한 이미지에서 여러 컨테이너)를 시작하면 Docker 는 writable container layer 만 만들면됩니다.

Docker 가 새 컨테이너를 시작할 때마다 기본 이미지 스택의 전체 복사본을 만들어야하는 경우 컨테이너 시작 시간과 사용 된 디스크 공간이 크게 늘어납니다.
이는 가상 시스템이 작동하는 방식과 유사하며 가상 시스템마다 하나 이상의 가상 디스크가 있습니다.

### Reference  
[docker-storage-driver]

[docker-storage-driver]:      https://docs.docker.com/storage/storagedriver