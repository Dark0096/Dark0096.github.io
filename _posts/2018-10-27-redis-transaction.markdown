---
layout: post
title:  "Redis transactions (번역)"
date:   2018-10-27 20:00:00
author: Dark
categories: Redis
tags: redis
---

## Transactions
Redis 에서 `MULTI, EXEC, DISCARD, WATCH` 는 Transaction 의 기반이 되는 Command 들이다. 이들은 한 단계로 Command 들을 그룹 단위로 실행할 수 있으며, 두 가지의 중요한 점들을 보장한다:

<ol>
  <li>Transaction 에서 모든 Command 들은 순차화되어 순서대로 실행된다. 다른 사용자의 요청은 기존에 실행되고 있던 Redis transaction 중간에 실행될 수 없다. 즉, Redis 에서 Transaction 은 단일 isolated 작업으로써 실행된다는 점이 보장된다.</li>
  <li>
  Redis transaction 은 모든 Command 들을 처리 또는 수행하지 않음으로써 Atomic 을 보장한다. `EXEC` Command 는 Transaction 의 실행 시작을 의미한다.
  client 가 `MULTI` Command 를 호출하기 이전에 서버에 대한 연결이 끊어지면 Transaction context 내의 작업들이 하나도 수행되지 않는다. 대신 `EXEC` Command 가 호출이 된다면, 모든 작업들은 수행된다.
  append-only 파일을 사용할 때 Redis 는 단일 write (2) 시스템 호출을 사용하여 디스크에 Transaction 을 기록한다.
  그러나 Redis server 가 고장나거나 system 관리자에 의해 종료되는 발생하기 힘든 경우가 일어났을 때에는 일부 작업들만이 기록될 수도 있다.
  Redis 는 재시작할 때에 이러한 비정상적인 상황들을 감지하고, error 와 함께 종료된다.
  redis-check-aof tool 을 사용하면 server 가 다시 시작될 수 있도록 부분적으로 기록된 작업들을 지움으로써 error 상황을 해결할 수 있다.
  </li>
</ol>

version 2.2 부터 Redis 는 `check-and-set (CAS)` 작업과 아주 유사한 방법으로 optimistic locking 방식을 위에서 언급한 두 가지 외에도 보장한다. 이에 대한 자세한 내용은 페이지의 하단에 설명되어 있다.

### Usage

Redis 는 `MULTI` Command 를 사용함으로써 Transaction 의 시작을 표현할 수 있다. 이 Command 를 호출하면 Redis 로부터 항상 OK 응답을 전달받는다. 이 시점에서 사용자는 여러 Command 들을 실행할 수 있다.
이러한 Command 들을 실행하기 이전에, Redis 는 이들을 queue 에 넣는다. 모든 Command 들이 `EXEC` Command 가 호출되는 순간 모두 실행된다.

대신 `DISCARD` Command 를 호출하면 Transaction Queue 가 비워지고 Transaction 이 종료됩니다.

하단의 예제는 key foo 와 bar 의 값을 atomic 하게 증가시킨다.

{% highlight bash %}
## 정상적인 Transaction 처리
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
## Transaction 처리 결과 확인
> GET foo
"1"
> GET bar
"1"
## DISCARD 를 통해 Transaction 내의 모든 작업들을 Queue 에서 제거
> MULTI
OK
> INCR foo
QUEUED
> DISCARD
OK
## 값이 바뀌지 않음
> GET foo
"1"
{% endhighlight %}

위의 Session 에서 볼 수 있듯이 `EXEC` Command 의 응답은 배열 형태로 오게 된다. 각각의 응답은 Transaction 내에 하나의 Command 의 응답과 같다. 이 응답의 순서는 Transaction Command 의 순서와 동일하다.

`MULTI` Command 이후 모든 Command 들은 `QUEUED` string 응답 값을 반환하게 된다. ( Redis 프로토콜의 관점에서 상태 응답으로 전송 됨 )

`EXEC` Command 가 호출되면 Queue 에 존재하던 Transaction Command 들은 실행되도록 예약된다.

### Errors inside a transaction
Transaction 중에 두 종류의 Command 오류가 발생할 수 있다:

<ol>
    <li>
    `EXEC` Command 를 실행하기 이전에 Queue 에 적재하는 도중 실패하는 경우가 있다.
    실패하는 경우는 Command 가 문법적으로 잘못된 경우 ( wrong number of arguments, wrong command name, ... ) 또는 메모리 부족과 같은 심각한 상태인 경우( Redis server 가 `maxmemory` 지시자를 사용하여 memory limit 설정이 된 경우 ) 에 발생할 수 있다.
    </li>
    <li>
    `EXEC` Command 를 실행한 이후에 실패하는 경우가 있다.
    잘못된 명령어 ( string value 에 list 에게만 실행할 수 있는 명령을 호출한 경우 ) 를 호출한 경우에 발생할 수 있다.
    </li>
</ol>

Client 들은 `EXEC` Command 를 호출하기 이전에 queued command 의 응답 값을 확인함으로써 첫번째 종류의 에러들을 감지할 수 있다:
Command 의 응답 값으로 `QUEUED` 가 온 경우에는 성공적으로 처리되었다고 보면 된다. 그렇지 않은 경우에는 Error 를 응답 값으로 받게 된다. Transaction Command 를 Redis Server 의 Queue 에 적재하는 동안에 에러가 발생하게 되면 대부분의 Client 들은 `DISCARD` Command 를 통해 Transaction Command 들을 모두 중단할 수 있다.

하지만, Redis 2.6.5 이후부터는 Server 에서 명령이 누적되는 동안 오류가 있음을 기억한다. 이는 `EXEC` 실행시 Transaction 을 거부한다는 오류를 반환하고 자동으로 `DISCARD` 처리한다.

Redis 2.6.5 이전에는 Client 가 오류 응답에 관계없이 `EXEC` 호출한 경우에는 Transaction 내의 Command 들 중 성공적으로 처리할 수 있는 일부분의 Command 만 실행되었다. 새로운 동작으로 인해 Transaction 을 파이프라인과 섞는 것이 훨씬 간단해 졌기 때문에 전체 트랜잭션을 한꺼번에 보내고 나중에 모든 응답을 한 번에 읽을 수 있다.

`EXEC` 이후 발생한 오류는 특별한 방법으로 처리되지 않습니다. 트랜잭션 중에 일부 명령이 실패하더라도 다른 모든 명령들이 실행됩니다.

이것은 프로토콜 수준에서 더 명확하다. 다음 예제에서는 구문이 올바르더라도 실행될 때 한 명령이 실패한다:

{% highlight bash %}
### Start transaction
> MULTI
+OK
> SET a 5
+QUEUED
> GET a
+QUEUED
### Wrong command ( only use to list type )
> LPOP a
+QUEUED
> SET a 4
+QUEUED
> GET a
+QUEUED
### End transaction
> EXEC
1) OK
2) "5"
3) (error) WRONGTYPE Operation against a key holding the wrong kind of value
4) OK
5) "4"
{% endhighlight %}

`명령이 실패하더라도 Queue 의 다른 모든 명령이 처리된다. Redis 는 명령 처리를 중단하지 않는다.`

telnet과 함께 유선 프로토콜을 다시 사용하는 또 다른 예는 구문 오류가 최대한 빨리보고되는 방법을 보여준다.

{% highlight bash %}
MULTI
+OK
INCR a b c
-ERR wrong number of arguments for 'incr' command
{% endhighlight %}

이번에는 잘못된 INCR 명령이 구문 오류로 인해 Redis queue 에 저장되지 않는다.

### Why Redis does not support roll backs?
Relational 데이터베이스에 대한 배경 지식이 있는 경우, Redis 의 Transaction 처리가 Command 가 일부 실패할 수 있지만 롤백하지 않고 나머지 트랜잭션을 실행한다는 점이 이상해 보일 수 있다.

그러나 이러한 Redis Transaction 처리 방법이 장점이 되기도 한다:

Redis 명령은 잘못된 구문으로 호출된 경우에만 실패 할 수 있으며 (Queue 에서 문제가 감지되지 않음) 또는 잘못된 데이터 유형에 대한 요청이 실패할 수 있다. 즉, 실패한 명령은 프로그래밍 오류의 결과이며, 대부분 개발 단계에서 발견될 수 있는 종류의 오류이므로 Production 에 발생할 일은 거의 발생하지 않는다.
Redis 는 rollback 할 필요가 없기 때문에 내부적으로 단순화되고 빨라졌다.
Redis 의 Transaction 처리에 대한 논쟁은 일반적인 사용자가 이러한 동작 방식을 이해하지 못하고 있을 경우 버그가 발생할 수 있다는 점이지만, 일반적으로 rollback 이 프로그래밍 오류로부터 보호할 수는 없다.
예를 들어 Command 가 특정 key 를 1에서 2로 증가시키거나 잘못된 key 를 증가시키면 rollback mechanism 이 도움이 되지 않는다.
누구도 프로그래머의 실수를 막을 수 없으며, Redis 명령이 실패하는 데 필요한 오류의 종류가 프로덕션 환경에 들어가기가 쉽지 않기 때문에 오류에 대한 롤백을 지원하지 않는 보다 간단하고 빠른 방법을 선택했다.

### Discarding the command queue
`DISCARD` 는 Transaction 을 중단시키기 위해 사용될 수 있다. 이 경우 명령이 실행되지 않고 연결 상태가 정상으로 복원된다.

{% highlight bash %}
> SET foo 1
OK
> MULTI
OK
> INCR foo
QUEUED
> DISCARD
OK
> GET foo
"1"
{% endhighlight %}

### Optimistic locking using check-and-set
`WATCH` 는 Redis 트랜잭션에 CAS (Check-and-Set) 동작을 제공하는 데 사용된다.

`WATCH` 된 키는 변경 사항을 감지하기 위해 모니터 된다. `EXEC` 명령 전에 하나 이상의 감시 키가 수정되면 전체 트랜잭션이 중단되고 `EXEC` 는 트랜잭션이 실패했음을 알리기 위해 Null 응답을 반환합니다.

예를 들어, 키의 값을 원자적으로 1 씩 증가시킬 필요가 있다고 상상해보자 ( Redis 에는 INCR 이 없다고 가정해 보자 ).

첫 번째 시도는 다음과 같다.

{% highlight bash %}
val = GET mykey
val = val + 1
SET mykey $val
{% endhighlight %}

이것은 주어진 시간에 하나의 클라이언트가 작업을 수행하는 경우에만 안정적으로 작동한다. 여러 클라이언트가 거의 같은 시간에 키를 증가 시키려고하면 race condition 이 발생한다.
예를 들어 클라이언트 A와 B는 이전 값 (예 : 10)을 읽는다. 이 값은 클라이언트에서 모두 11로 증가하고 마지막으로 키 값으로 SET 된다. 따라서 최종 값은 12 대신 11이 된다.

`WATCH` 덕분에 우리는 문제를 아주 잘 다룰 수 있게 되었다:

{% highlight bash %}
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
{% endhighlight %}
위의 코드를 사용하여 Race condition 이 있고 다른 클라이언트가 WATCH 호출과 EXEC 호출 사이의 시간에 val 결과를 수정하면 Transaction 이 실패한다.

이 잠금 양식은 낙관적 잠금( Optimistic locking )이라고 하며 잠금의 매우 강력한 형태이다. 많은 경우, 여러 클라이언트가 서로 다른 키에 액세스하므로 충돌이 거의 발생하지 않는다. 일반적으로 작업을 반복할 가능성은 낮다.

## WATCH explained
`WATCH` 는 어떤 역할을 하는 Command 입니까? 이것은 `EXEC` Command 를 조건부처럼 실행할 수 있게 만드는 Command 이다: `WATCH` 하고 있는 key 의 값이 변경되지 않는다면, Transaction 을 실행하세요. (하지만 Transaction 을 중단하지 않고 동일한 클라이언트가 Transaction 을 변경할 수 있다.) 그렇지 않으면 Transaction 은 전혀 수행되지 않는다. ( TTL 이 설정된 key 에 대해 WATCH 를 한 경우, expire 이후에도 EXEC 는 계속 작동한다. )

`WATCH`는 여러 번 요청할 수 있다. 간단히 모든 `WATCH` 요청은 `EXEC` 가 요청되는 순간까지 `WATCH` 호출에서 시작하여 변경 사항을 감시하는 효과를 갖는다. 하나의 `WATCH` 에 여러 개의 키를 보낼 수도 있다. (WATCH key1 key2 ...)

`EXEC`가 호출되면 트랜잭션이 중단되었는지 여부에 관계없이 모든 키가 `UNWATCH` 가 된다. 또한 Client connection 이 닫히면 모든 키가 `UNWATCH`가 된다.
모든 `WATCH` 된 키를 해제하기 위해 인수없이 `UNWATCH` 명령을 사용할 수도 있습니다.

경우에 따라서는 몇몇 키의 값을 변경하기 위해 Transaction 을 수행해야할 수도 있기 때문에 낙관적이게 Lock 을 잡는 것은 유용하다. 키의 현재 값을 읽은 후에 더이상 `WATCH` 기능을 사용하고 싶지 않은 경우가 발생할 수 있다.
이런 일이 발생하면 `UNWATCH`를 호출하여 새로운 트랜잭션을 위해 연결을 자유롭게 사용할 수 있다.

### Using WATCH to implement ZPOP
Redis 가 지원하지 않는 새로운 Atomic 연산을 생성하는 데 WATCH 를 사용하는 방법을 보여주는 좋은 예는 ZPOP 를 구현하는 것이다. ZPOP 은 정렬된 집합의 더 낮은 점수로 Atomic 방식으로 요소를 팝하는 명령이다. 이것은 가장 간단한 구현이다.

{% highlight bash %}
WATCH zset
element = ZRANGE zset 0 0
MULTI
ZREM zset element
EXEC
{% endhighlight %}
`EXEC` 가 실패하면 (즉 Null 응답을 반환) 작업을 반복한다.

### Redis scripting and transactions
Redis script 는 정의상 Transaction 방식이므로 Redis Transaction 으로 수행 할 수 있는 모든 작업을 Script 로 수행 할 수 있다. 일반적으로 Script 는 더 간단하고 빠르다.

이 중복( Script 와 Transaction ) 은 이전에 Transaction 이 존재하였고, Scripting 이 Redis 2.6 에서 도입되었기 때문에 어쩔 수 없이 발생하게 되었다. 그러나 우리는 짧은 시간안에 Transaction 지원을 제거하지는 않을 것이다. Redis Scripting 을 사용하지 않고도 경쟁 조건을 피하는 것이 여전히 가능하며 Scripting 을 이용하는 것보다 Transaction 을 이용하는 것이 복잡성이 최소화되기 때문이다.

하지만, 가까운 시일 내에 모든 Redis 유저들이 Scripting 을 기반으로 Race condition 문제를 해결하고 있음을 확인하는 순간 우리는 Transaction 기능을 제거할 수도 있다.


### Reference
[Redis Transactions]

[Redis Transactions]:      https://redis.io/topics/transactions