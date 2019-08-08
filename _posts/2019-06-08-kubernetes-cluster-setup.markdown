---
layout: post
title:  "kubernetes cluster set up"
date:   2018-11-30 01:00:00
author: Dark
categories: kubernetes
tags: kubernetes
---

Hello Kubernetes

이번 시간에는 Kubernetes cluster 를 세팅하고, hello docker image 를 포함한 pod 를 신규로 구성한 Cluster 에 배포해보도록 하겠습니다.

kops 는 kubernetes operations 를 의미하며, Production level 의 kubernetes cluster 를 손쉽게 구성하고 운영하는데 도움을 주는 tool 입니다. 
kops 는 3 가지 종류의 Resource 를 생성 및 제거 등 다양한 Command 를 실행할 수 있도록 지원합니다.

1. Cluster
2. instancegroup
3. secret

Resource 생성시에는 미리 생성해놓은 yml 파일을 사용하거나 stdin 을 이용할 수 있습니다.
Detail 한 사용 정보는 아래의 경로를 확인해주세요.  
[kops command docs 바로가기]

우선 kubernetes cluster 를 생성하기 위해 필요한 tool 들을 설치하여야 합니다.

```
$ brew install kops
$ brew install awscli
```

Kubernetes cluster 를 AWS 환경에서 구성하고 다양한 테스트를 진행해보고자 하였습니다. 
이때에 Local 에서 AWS 환경에 접근할 수 있는 계정을 별도로 생성하고, 아래와 같은 IAM Group 을 생성 후 Binding 하였습니다.
<img src="https://raw.githubusercontent.com/Dark0096/Dark0096.github.io/master/assets/post/2019-06-08-iam-group.png" title="IAM Group">

# S3 state store setting

state store 로 S3 를 사용하기 위해서는 Bucket 을 생성해야합니다. 

```
$ aws s3api create-bucket --bucket kubernetes-dark-cluster --region ap-northeast-2 --create-bucket-configuration LocationConstraint=ap-northeast-2
```

업데이트된 내용들을 version 관리할 수 있도록 bucket versioning option 을 enable 처리합니다. 

```
$ aws s3api put-bucket-versioning --bucket kubernetes-dark-cluster --versioning-configuration Status=Enabled
```

kops 에서 state store 의 의미가 무엇인지 파악하고 싶으신 경우 아래의 블로그 글을 참조해주세요.
[kops state store란?]

# kops create cluster
kops 는 Resource 를 생성하고 이에 대한 정보를 파일로 저장하여 diff 를 기반으로 동작합니다. 
이때에 파일을 Local 에서 저장할 수 있지만 AWS S3 와 같은 다른 state store 를 이용할 수도 있습니다.

```
$ export KOPS_CLUSTER_NAME=imesh.k8s.local
$ export KOPS_STATE_STORE=s3://kubernetes-dark-cluster
```

아래의 Command 를 실행하면 kops 는 stdin 으로 입력된 정보를 기반으로 cluster 를 생성하게 됩니다. 이 정보는 KOPS_STATE_STORE 환경 변수에 지정한 system 에 저장되게 됩니다.  
 
```
$ kops create cluster --node-count=1 --node-size=t2.medium --master-size=t2.medium --zones='ap-northeast-2a' --name=$KOPS_CLUSTER_NAME
```

구조는 아래와 같습니다.
<img src="https://raw.githubusercontent.com/Dark0096/Dark0096.github.io/master/assets/post/2019-06-08-kops-state.png" title="IAM Group">

필요한 경우 ssh key 를 생성합니다.
```
$ ssh-keygen -t rsa -b 4096
```

kops 를 통해 secret key 를 생성합니다.

```
$ kops create secret --name imesh.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub
```

kops cluster 를 실제로 생성합니다.

```
$ kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
```

# Kubernetes dashboard 추가하기

kubernetes cluster 를 운영할 때에 UI 를 통하여 전반적인 상황을 보고 싶은 요구가 자연스럽게 발생합니다. 이때에는 별도의 dashboard 를 pod 으로 등록하여 사용할 수 있습니다.
해당 pod 은 별도의 설정을 하지 않는 경우 `kube-system` namespace 에 속하게 됩니다.

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

Dashboard 는 Authentication 이 되어야 제대로 이용할 수 있다. 그렇다면 어떻게 Authentication 을 처리할 수 있을까?  
이 부분을 처리하기 위해서는 kops 를 통하여 token 을 획득하고 활용하여야만 한다. 

```
$ kops get secrets admin --type secret -oplaintext
```


### Reference
[kops command docs 바로가기]  
[how to create kubernetes cluster]  
[kubernetes dashboard]  
[kops state store란?]

[kops command docs 바로가기]:              https://github.com/kubernetes/kops/tree/master/docs/cli
[how to create kubernetes cluster]:      https://medium.com/containermind/how-to-create-a-kubernetes-cluster-on-aws-in-few-minutes-89dda10354f4
[kubernetes dashboard]:                  https://github.com/kubernetes/dashboard#kubernetes-dashboard
[kops state store란?]:                   https://dark0096.github.io/kubernetes/kops/2018/11/30/kubernetes-state-store.html