---
layout: post
title:  "kops state store"
date:   2018-11-30 01:00:00
author: Dark
categories: kubernetes kops
tags: kubernetes kops
---

해당 Blog post 는 kops github 의 Doc 을 번역 + 보충 설명을 통하여 state store 를 설명하고 있습니다.

# State Store

kops 에는 state store 라는 개념이 존재합니다. 이곳에는 저희들이 kops 를 이용하여 생성한 cluster 의 설정 정보가 저장됩니다. 
이 state 값을 기반으로 최초 클러스터를 생성할 때 뿐만 아니라, 운영중인 kubernetes cluster 를 재구성할 수 있습니다.
kops 는 내부적으로 VFS 를 사용하여 state store 를 저장하도록 구현되어 있습니다. 이는 File system 을 abstraction 한 것으로 다양한 구현체를 통하여 여러 File system 을 사용할 수 있습니다. 다음의 state store 들은 지원되고 있는 시스템들을 나타냅니다:

<ul>
    <li>Amazon AWS S3 (s3://)</li>
    <li>local filesystem (file://)</li>
    <li>Digital Ocean (do://)</li>
    <li>MemFS (memfs://)</li>
    <li>Google Cloud (gs://)</li>
    <li>Kubernetes (k8s://)</li>
    <li>OpenStack Swift (swift://)</li>
    <li>AliCloud (oss://)</li>
</ul>

이 state store 는 단지 파일들입니다. 이에 따라, git 과 같은 version control system 에서 관리해도 무방합니다.

# {statestore}/config

state store 에서 가장 중요한 파일 중 하나는 config file 입니다. 이 파일은 kubernetes cluster 의 주요한 설정 값들 (instance types, zones, etc) 을 저장하고 있습니다. 
아래와 같이 kops 를 통하여 cluster 를 생성한다는 명령어를 요청한다고 가정해봅시다. 이후에 kops 는 명시한 command 의 옵션들을 기반으로 state store config 내에 그 상태를 저장하게 됩니다. 

{% highlight bash %}
$ kops create cluster --node-size=m4.large
{% endhighlight %}

위와 같은 command 를 실행할 경우 node size 를 m4.large 로 option 을 주었기 때문에 state store configuration 에는 아래와 같은 한 줄이 추가될 것입니다.
{% highlight bash %}
NodeMachineType: m4.large
{% endhighlight %}

kops command line tool 을 이용할 때에 사용되는 option 들은 사용자들이 좀 더 쉽게 cluster 를 구축할 수 있도록 제공되는 short-cut 이라고 볼 수 있습니다.
이러한 option 들은 기존의 state store 에 저장된 설정 값에 병합되는 구조로 관리됩니다.
만일 아직 command line tool 에서 제공되지 않는 option 들을 설정하고 싶거나 text 기반의 설정을 좀 더 선호하신다면, `kops edit cluster` command 를 통하여 직접 수정할 수 있습니다.
configuration 은 병합되기 때문에 클러스터를 재구성할 때에 변경된 인수를 재지정할 수 있습니다. 예를 들어, dry run 이후에 클러스터를 다시 생성하면 됩니다.


# Moving state between S3 buckets
state store 를 다른 s3 bucket 으로 쉽게 이동시킬 수 있습니다. Single cluster 를 처리하는 방법은 아래와 같습니다:

<ol>
    <li>${OLD_KOPS_STATE_STORE}/${CLUSTER_NAME} 에서 이동시키고자 하는 새로운 state store ( ${NEW_KOPS_STATE_STORE}/${CLUSTER_NAME} ) 로 하위의 모든 파일들을 이동시키빈다. 이때에는 aws s3 sync 나 기타 유사한 tool 을 이용하여 이동시킵니다.</li>
    <li>KOPS_STATE_STORE 환경 변수를 새로운 S3 bucket 을 바라보도록 업데이트합니다.</li>
    <li>`kops edit cluster ${CLUSTER_NAME}` 또는 cluster manifest yaml file 을 직접 수정합니다. 그리고, NEW_KOPS_STATE_STORE 을 보고 있는 .spec.configBase 을 업데이트하세요.</li>
    <li>이제 `kops update cluster ${CLUSTER_NAME} --yes`  실행하여 변경된 클러스터 정보를 반영할 수 있습니다. 새롭게 생성된 노드들은 새로 설정한 NEW_KOPS_STATE_STORE bucket 으로부터 의존성이 있는 파일들을 찾아오게 될 것입니다. 이제 기존의 bucket 의 파일들은 안전하게 제거해도 됩니다.</li>
    <li>이전하고자 하는 cluster 들은 모두 위와 같은 방법으로 이동시킵니다.</li>
</ol>

# State store configuration
state store 를 설정하는데에는 여러 가지 방법이 존재합니다. 설정이 적용되는 데에는 우선순위가 존재하며 이는 아래와 같습니다:

command line argument --state s3://yourstatestore  
environment variable export KOPS_STATE_STORE=s3://yourstatestore  
config file $HOME/.kops.yaml  
config file $HOME/.kops/config  

Configuration file example:  
$HOME/.kops/config might look like this:

```
kops_state_store: s3://yourstatestore
```

# Cross Account State-store (AWS)
클러스터를 생성하기 위해 kops 를 실행하는 엔티티가 state store bucket 의 소유자와 동일한 계정에 있지 않은 상황이 있습니다.
이 경우 권한을 명시적으로 부여해야합니다: 
`s3:getBucketLocation` - kops를 실행중인 ARN.

다음 정책을 사용하여 Cross account 간의 state store 를 가져와 사용할 수 있습니다:
```
{
    "Id": "123",
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "123",
            "Action": [
                "s3:GetBucketLocation"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:s3:::state-store-bucket",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::123456789:user/kopsuser"
                ]
            }
        }
    ]
}
```

### Reference
[kops state store]

[kops state store]:      https://github.com/kubernetes/kops/blob/master/docs/state.md