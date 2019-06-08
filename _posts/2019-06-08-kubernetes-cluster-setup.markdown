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

kops 는 kubernetes ops 를 의미하며, Production level 의 kubernetes cluster 를 손쉽게 구성하고 운영하는데 도움을 주는 tool 입니다. 
kops 는 3 가지 종류의 Resource 를 생성 및 제거 등 다양한 Command 를 실행할 수 있도록 지원합니다.

1. Cluster
2. instancegroup
3. secret

Resource 생성시에는 미리 생성해놓은 yml 파일을 사용하거나 stdin 을 이용할 수 있습니다.

Detail 한 사용 정보는 아래의 경로에서 자세히 설명하고 있습니다.  
[kops command docs 바로가기]

우선 kops 를 설치하여야 합니다.

brew install kops
<img src="https://raw.githubusercontent.com/Dark0096/Dark0096.github.io/master/assets/post/2019-06-08-iam-group.png" title="IAM Group">
 

Kubernetes cluster 를 AWS 환경에서 구성하고 다양한 테스트를 진행해보고자 하였다. 

### Reference
[kops command docs 바로가기]

[kops command docs 바로가기]:      https://github.com/kubernetes/kops/tree/master/docs/cli