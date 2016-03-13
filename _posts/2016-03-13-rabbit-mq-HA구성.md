---
layout: post
title: RabbitMQ HA구성
weight: 10
excerpt: "Rabbit MQ의 HA Clustering을 통해 서비스의 가용성을 높이는 방법에 대해서 알아본다."
tags: [rabbitMQ, oss]
modified: 2016-03-10
comments: true
category : rabbit
---


고가용성 Queue 구성
===================

기본적으로 Cluster구성을 하면 Queue는 단일노드에만 존재하고 복제되지 않는다.
(Exchange, Bindings는 복제됨)

옵션을 지정하여 *mirrored*된 Queue를 만들 수 있으며 하나의 *master*와 한개이상의 *slave*로 구성되며
현 *master*가 사라지면 가장 오래된 *slave*가 *master*로 승급한다.



미러링 구성
-----------

*policy*설정에 의해 미러링을 활성화 시킬 수 있다. (*policy*설정에 의해 특정 패턴의 Queue는 자동 미러링 되도록 설정 할 수 있다.)



### 미러링 종류(ha-mode) 3가지

all
: 클러스터내의 모든 노드를 미러링 함

exactly
: 특정 수 만큼의 노드만 미러링 함
: *ha-params*으로 *count*로 미러링 수 지정

: *count*가 총 노드스보다 많으면 *all*과 동일하게 동작
: 미러링된 노드가 죽었으면 *count*를 채우기 위해 다른 노드를 새 미러로 만듬

nodes
: 특정 이름의 노드들끼리 미러링함
: *ha-params*으로 노드이름(node names) 지정



### Queue master 위치

모든 Queue는 *Queue Master*라 불리는 기준 노드를 가지고 있다.
FIFO보장을 위해 모든 Queue는 master를 먼저 처리하고 다른 mirror들을 처리한다.


#### Queue master 위치 지정 방법

1. Queue를 선언할 때 (queue declare) *x-queue-master-locator* 사용
2. 설정파일의 *queue_master_locator* 키의 값 지정


##### 3가지 설정방법
min-masters
: 최소수의 master로 설정 (현재 가지고 있는 master queue가 가장 적은 노드를 새로운 queue의 마스터로)
: queue를 생성시의 연결이 서버별로 고르게 분상되지 않는다면 이 방법이 바람직해 보임

client-local
: Queue를 선언할때 연결된 client를 지정
: 별다른 설정을 안했다면 이 값이 Default
: queue를 생성시의 연결이 서버별로 고르게 분배된다면 이 방법을 쓰면 서버별로 고르게 master queue가 분배될 듯

random
 : 랜덤


##### 설정방법별 테스트


configuration 파일 (rabbitmq.config)열어 *queue_master_locator* 항목을 추가한다.

~~~
vi /etc/rabbitmq/rabbitmq.config
~~~

~~~~~
[
 {rabbit,
  [
   {queue_master_locator, <<"min-masters">>}
   
   ...

  ]}
].
~~~~~

cluster 노드들을 재기동하여 설정을 반영한다.

※ 주의 : json포맷의 설정이 잘못되거나 .으로 끝나지 않으면 기동시 오류 발생



하나의 서버에 접근하여 *queueDeclare*로 5개의 queue를 생성해보고 master위치지정 방법별로 어떻게 생성되는지 확인해본다.

~~~~
channel.queueDeclare("ha.queue1", false, false, false, null);
		
channel.queueDeclare("ha.queue2", false, false, false, null);
		
channel.queueDeclare("ha.queue3", false, false, false, null);

channel.queueDeclare("ha.queue4", false, false, false, null);
		
channel.queueDeclare("ha.queue5", false, false, false, null);
~~~~



[사진 master_queue_locator_result]








### 설정 샘플

*ha.*으로 시작하는 이름을 가진 Queue를 HA 구성하는 *ha-all*이름의 policy 설정


~~~
rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'
~~~


*two.*으로 시작하는 이름을 가진 Queue를 2개의 미러로 HA 구성하는 *ha-two*이름의 policy 설정
새로운 slave가 join할 때 마다 자동으로 Queue를 Sync하도록 설정

~~~
rabbitmqctl set_policy ha-two "^two\." \
   '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
~~~


*node.*으로 시작하는 이름을 가진 Queue를 rabbit@nodeA,rabbit@nodeB 노드간에 미러링 함

~~~
rabbitmqctl set_policy ha-nodes "^nodes\." \
   '{"ha-mode":"nodes","ha-params":["rabbit@nodeA", "rabbit@nodeB"]}'
~~~



### 새로운 Slave의 추가

slave 노드를 추가하는 것은 어느 때나 가능하지만 새로운 노드는 추가되기 이전의 Queue의 내용을 가지고 있지 못하다.

명시적으로 동기화(*synchronised*) 하는 것은 가능한데 그 순간에 Queue는 응답할 수 없는 상황이 일어나기 때문에
자연스럽게 동기화(*synchronised*) 될 수 있는 active queue가 되도록 하는게 더 좋다.


#### 한번에 동기화하는 Size 설정

위에 언급된 동기화시에 시간이 소요되어 응답 할 수 없는 상황이 일어나므로 RabbitMQ 3.6.0 이후 부터는
*policy*에 *ha-sync-batch-size* 라는 설정을 두어 나누어서 동기화하도록 성능이 향상되었다.


Queue의 데이터량, tick message 시간, 네트워크 대역폭 등을 고려해야 한다.
마냥 size를 높이면 시간이 오래 소요되어 ticktime을 초과하여 네트워크 partition이 깨진 것으로 인지 할 수 있기 때문이다.


    ha-sync-batch-size가 50000이고 큐의 메시지가 건당 1kb라면 net_ticktime이 1초로 설정되어있다면
    네트워크는 50Mb/1초 의 성능을 커버할 수 있어야 한다.


    최소 BatchSize = 네트워크 대역폭 * ticktime / 메시지당 size
    
    ex) 30Mb/s * 2s / 1Kb  = 60,000 이하로 설정해야 함




#### 명시적인 Synchronisation 구성

명시적으로 *Sychronised*하는 방법은 2가지가 있다.


##### 자동으로 설정

*policy*설정에서 "ha-sync-mode":"automatic"로 설정


##### 수동으로 수행

*policy*설정에서 "ha-sync-mode":"manual"로 설정 or 없으면 Default로 수동

이 명령어를 통해 수행

~~~~
rabbitmqctl sync_queue name
~~~~

~~~~
rabbitmqctl cancel_sync_queue name
~~~~


#### 노드가 중지될 때 주의 사항

*master*와 *slave*들로 구성된 cluster를 차례로 중지시킬 때 master가 중지되면 slave가 동기화되어있는 상태라고 가정했을 때
다른 slave 노드들이 차례로 master로 승격된다. 그러다 마지막 노드가 중지될 때 마지막 node가 *durable* 이라면 마지막 노드가
다시 기동되었을 때 Queue안에 메시지들은 살아 있게 된다. (*durable*설정이 안되어있으면 날아감)

이 *slave*들이 다시 살아나도 각자 노드에서 가지고 있는 데이터는 삭제된다. 각자 가지고 있는 데이터들이 *master*를 통해 동기화된
데이터 임을 확신할 수 없으므로 빈상태로 다시 새로 가입된 cluster 노드처럼 동작하게 된다. (동기화를 향후 수행)

또한 *slave*가 동기화되지 않은 상태에서는 설정값에 따라서 master로 승격이 되지 않을 수도 있는데 그 방법은 아래와 같다.


##### Master로 승격 설정

*master*가 중지되고 *slave*가 새로운 *master*가 되는데 동기화상황에 따라서 처리가 다르다.


*ha-promote-on-shutdown* policy 설정값

always
: *slave*가 동기화가 안되어있더라도 무조건 다음 *slave*를 *master*로 승격시킴

when-synced
: *slave*가 동기화가 되어 있지 않다면 메시지 유실방지를 위해 fail over를 하지 않고 전체 Queue를 중지시킴



##### 모든 노드가 중지될 때 master 분실상황

모든 노드가 중지되어있는 동안 master를 분실 할 수 있는 가능성이 있다.
보통은 마지막 노드가 중지되고 마지막 노드가 재기동될 때 master가 되지만 *forget_cluster_node*명령어를 날리면
삭제된 노드를 master로 가지고 있던 slave 중에 하나가 master로 승격된다.

*forget_cluster_node*할 때만 master 승격이 이루어지므로 master를 분실한 상황에서는 slave를 시작하기 전에 
*forget_cluster_node*명령을 수행해야 한다. (그냥 slave를 시작하면 위에 주의사항에 언급된데로 queue데이터를 날리므로)



#### 미러 Queue의 구현과 의미

master와 slave들로 구성된 상황에서 *publish*를 제외한 모든 명령은 master로만 전달되고 master에서 slave들에게 broadcast한다.
그래서 미러 Queue에다 client가 consume하는 일은 사실 master에 consume하는 것이다.


##### slave가 죽었을시 알아야 할 점

사실 slave가 죽으면 master는 그냥 master로 남아있기 때문에 client가 fail신호를 보내거나 하진 않는다. 때문에 slave 문제를
바로 알아차릴 수 없으므로 tick메시지 시간동안 지연이 발생 할 수 있다.


##### master가 죽었을시 알아야 할 점

master가 죽으며 slave가 master로 승급하는데 이 때 일어나는 일들은

1. master가 죽으면 최신 동기화 했을 가능성이 높은 최신 slave가 master가 된다. 하지만 모든 slave가 동기화를 하기전에 죽으면 메시지는 날아갈 수 밖에없다.

2. 메시지를 client에 전달했지만 client가 전송한 ACK를 받지 못한 상황에서 (client-master구간에서 사라졌거나, master-slave구간에서 사라짐)
ACK를 못받은 메시지는 재전송 할 수밖에 없음

3. master가 변경되었으므로 *x-cancel-on-ha-failover=true* 설정 상황에서는 `basic.cancel`의 cancel notification을 받는다.

4. client는 재전송 때문에 이전 받았던 메시지를 또 받게 되며, client는 재전송이란 것을 인지해야 한다.


publish는 모든 master와 slave에 직접되므로 slave가 master가 되는동안 메시지 유실없이 slave에 잘 쌓인다.
또한 publisher confirm도 별문제없이 동작한다. (HA Queues는 publish과정에서는 특별히 고려할 사항없음)


*noAck=true* 인 경우는 consumer에게 메시지가 날라가는 상황에 broker가 갑자기 죽어버리면 메시자가 유실된다.
*Consumer Cancellation Notification*으로 broker에 이상이 생겼음을 인지 시키도록 하면 유용하며,
반드시 전송이 보장되어야 하는 경우 Ack를 반드시 받도록 *noAck=false*로 설정하는걸 권장한다.





