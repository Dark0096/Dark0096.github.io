---
layout: post
title:  "Spark data locality"
date:   2018-09-04 12:00:00
author: Dark
categories: Spark
tags: spark
---

Data Locality 는 Spark Job 의 퍼포먼스에 많은 영향을 주는 요소 중 하나입니다. 만일, 데이터가 작동하는 코드와 함께 있는 경우 계산이 빨라지는 경향이 있습니다. 하지만, 데이터와 코드가 분리되어 있으면 코드와 데이터가 서로 이동해야 합니다. 일반적으로 Serialized 코드가 다른 장소로 이동하는 것이 데이터를 이동시키는 것보다는 빠릅니다. 그 이유는 데이터의 양에 비해 코드의 양은 대부분 작기 때문입니다. Spark 은 Data Locality 의 일반적인 원리를 기반으로 Scheduling 을 하게 됩니다.
Data Locality 는 데이터를 처리하는 코드와 얼마나 가까운 거리에 있는지를 나타냅니다. 데이터의 현재 위치를 기반으로 Locality 는 여러 Level 로 정의됩니다. 가장 가까운 것에서 가장 먼 순으로 Level 들을 살펴보겠습니다:

<ol>
  <li>
  PROCESS_LOCAL<br/>
  데이터가 실행되고 있는 코드의 JVM 과 함께 위치해 있는 경우입니다. 가장 실행 속도가 빠른 Locality 입니다.
  </li> 
  <li>
  NODE_LOCAL<br/>
  데이터가 같은 노드에 있는 경우입니다. 예를 들어, 같은 노드에 HDFS 가 존재하는 경우 또는 Executor 가 있는 경우입니다. 해당 Locality 는 PROCESS_LOCAL 에 비하여 data 가 process 들 간에 이동해야하기 때문에 PROCESS_LOCAL 보다는 조금 느립니다.  
  </li>
  <li>
  NO_PREF<br/>
  데이터는 어느 곳에서나 똑같이 빠르게 액세스되며 지역 선호도가 없습니다.
  </li>
  <li>
  RACK_LOCAL<br/>
  데이터가 같은 Rack 의 서버에 존재합니다. Data 는 다른 서버에 존재하지만, 같은 Rack 에 있는 경우 네트워크 비용이 발생하게 되는데 이는 단순하게 Switch 쪽을 통한 Network 비용만이 발생합니다.
  </li>
  <li>
  ANY<br/>
  데이터가 같은 Rack 이 아닌 다른 네트워크 상에 존재하는 경우입니다. 가장 속도가 느린 Locality 입니다.
  </li>
</ol>

  Spark 은 모든 작업을 실행 속도가 가장 빠른 Locality 를 선택하여 Scheduling 하려고 노력하지만 이것이 매번 가능한 것은 아닙니다. Idle executor 에 아직 처리되지 않은 데이터가 있는 상황에서 Spark 는 Locality 를 낮추기 위해 전환합니다. 이때에 다음과 같은 두 가지 옵션이 있습니다. 
a) 데이터와 같이 위치하고 있는 서버의 사용량이 많은 CPU 가 이용 가능할때까지 대기하는 방법.
b) 새 작업을 즉시 실행할 수 있는 곳으로 데이터를 이동시키는 방법.

  Spark 이 일반적으로 하는 일은 기본 설정인 3초 동안 CPU 가 가용해지기를 기다리고 있는 것입니다. Timeout (3초) 이 만료되면, 데이터를 멀리 떨어져있는 CPU 로 이동시키기 시작합니다. 각 레벨 간의 폴백 대기 시간 초과는 개별적으로 또는 모두 함께 하나의 매개 변수로 구성 할 수 있습니다. 자세한 내용은 구성 페이지의 spark.locality 매개 변수를 참조하십시오. 작업이 길고 Locality 가 좋지 않은 경우 이러한 설정을 늘려야하지만 기본값은 대부분의 상황에 잘 작동합니다.
  
### Reference
[Spark official guide]

[Spark property]  

[Spark official guide]:      http://spark.apache.org/docs/latest/tuning.html#data-locality
[Spark property]:            http://spark.apache.org/docs/latest/configuration.html#scheduling