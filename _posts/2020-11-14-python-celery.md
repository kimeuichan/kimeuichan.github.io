---
title: Python Celery란 
author: daru
date: 2020-11-14 21:00:00
categories: [programming,python]
tags: [python,async,celery]
---


> 해당 글은 `celery`를 이용하여 코드를 잘 작성하는것에 비중을 둔 글이 아니라 
>
> 간단한 `celery` 소개 및 특징과 `broker`에 대해 공부하기 위해 작성한 글입니다.


`Celery`는 python 동시성 프로그래밍에서 가장 많이 사용되는 방법 중 하나입니다.
몇 가지 설정만 한다면 간단하게 python 코드를 실행하는 `worker`를 만들 수 있습니다.

`Celery`는 `task`를 `broker`를 통해 전달하고 `worker`가 처리하는 구조입니다.

공식 홈페이지에서 `celery`의 정의를 찾던 중 가장 핵심 적인 문구를 하나 찾았습니다.

**실시간 처리에 중점을두고 작업 예약을 지원하는 작업 큐입니다.**

물론 스케줄링을 지원하지만 실시간 처리에 중점을 두고 있습니다.

## Celery 구성 요소
`Celery`의 구성요소는 크게 3가지로 구분됩니다.
- Broker: task(message)를 `worker`에게 전달하는 역할

- Client: task를 생성하는 역할
- Worker: task를 실행하고 처리하는 역할


## Celery 특징
- `Cleient`나 `Worker`는 연결 유실이나 실패에 대해 자동으로 재시도 합니다. 또한 `Broker`, `Worker`를 HA 구성하여 고가용성이 뛰어납니다.
- 단일 `Celery` 프로세스는 최적화 설정(RabbitMQ, librabbitmq 등)을 통해 밀리 초 정도의 대기 시간으로 분당 수백만 개의 작업을 처리 할 수 있을 정도로 빠릅니다.
- `Celery`는 확장성이 매우 뛰어나 거의 모든 부분을 커스텀하여 사용할 수 있습니다. (사용자 지정 풀 구현, serializer, compression schemes, logging, schedulers, consumers, producers, broker transports 등.)


## Celery Broker
`broker`는 `client`와 `worker` 사이에서 메시지 전달을 중개하는 역할을 합니다.
`5.0` 버전 기준으로 다음과 같은 `broker`를 지원합니다.
- RabbitMQ
- Redis
- Amazon SQS(모니터링, 원격 제어 지원X)
- Zookeeper(실험적)

해당 글에서는 일반적으로 많이 쓰는 `RabbitMQ`와 `Redis`를 다룰 예정입니다.

### RabbitMQ vs Redis

#### RabbitMQ
- `RabbitMQ`의 인증 및 인가 방식을 그대로 사용할 수 있습니다.
- 매우 효율적이고 광범위하게 배포 및 테스트 된 메시지 브로커이지만 더 많은 내구성과 안정성을 위해 조정하면 성능에 영향을 미칠 수 있습니다.
- 고급 라우팅 요구 사항이있는 엔터프라이즈 메시징을 제공하는데 더 적합합니다.

![RabbitMQ_ACK.PNG](https://ssup2.github.io/images/theory_analysis/RabbitMQ_ACK/RabbitMQ_ACK.PNG)

**RabbitMQ ACK**

**Message가 최소 한번 이상은 전달되는 것을 보장하는 성질**입니다. 이러한 성질을 `At Least Once`라고 명칭합니다.
송신자는 `Message`를 전송한 이후에 수신자로부터 `ACK`를 받지 못한다면, 수신자에게 `ACK`를 받을 때 까지 `Message`를 재전송합니다.
따라서 수신자가 `ACK`를 받지 못한다면 `Message`를 재전송합니다.
송신자가 `Message`를 처리한 다음 수신자에게 `ACK`를 전송하여도 일시적 `Network` 장애로 인하여 수신자에게 `ACK`가 전달 되지 않을 수 있습니다.
`ACK`를 받지 못한 송신자는 `Message`를 다시 수신자에게 전송할 수 있습니다. 즉 **수신자는 동일한 `Message`를 2번이상 받을 수 있습니다.**

따라서 celery에서는 `RabbitMQ`를 `broker`로 사용할 때 해당 메시지에 대해 [멱등성이 필요합니다.](https://docs.celeryproject.org/en/stable/glossary.html#term-idempotent)


#### Redis
- 시작이 빠르고 가볍고 빠른 브로커이지만 안정적인 전달을 지원하지 않습니다. 
- 시스템이 종료되는 경우 몇 분 동안 작업에 대한 정보를 손실하는 것이 중요하지 않은 애플리케이션에 대해 선택할 수 있습니다.
- `Redis` 자체가 메모리를 사용하여 저장하는 방법이라 메모리가 부족한 상황에서는 임의로 `key`가 삭제될 수 있습니다. `task`를 받아도 삭제 될 수 있습니다.
- [메시지 손실 방지 기법이 없습니다.](https://stackoverflow.com/a/50247277/5944655)


> `Redis`가 속도면에서 더 빠르다고는 하지만 엔터프라이즈 환경에서 다양한 라우팅, 메시지 유실 방지 등이 필요로 합니다.
> 
> 하지만 메시지가 유실되어도 상관이 없고 속도가 중요한 경우는 `Redis`가 더 효율적일 수 있습니다. 
> 상황에 맞춰 사용하는 것이 좋습니다.




## Celery Worker

### Starting the worker
`celery worker`는 같은 머신에서도 다른 `hostname`을 통해 여러 `worker`를 생성할 수 있습니다.

```sh
celery -A proj worker --loglevel=INFO --concurrency=10 -n worker1@%h
celery -A proj worker --loglevel=INFO --concurrency=10 -n worker2@%h
celery -A proj worker --loglevel=INFO --concurrency=10 -n worker3@%h
```

### Stopping the worker
`TERM` 시그날을 통해 워커를 종료할 수 있습니다.

종료가 시작되면 작업자는 실제로 종료되기 전에 현재 실행중인 모든 작업을 완료합니다. 작업이 중요한 경우 `KILL` signal을 보내기전 작업을 수행하기 전까지 기다려야합니다.

`worker`가 무한 루프 또는 이와 유사한 상황에 갇혀서 상당한 시간이 지난 후에도 종료되지 않는 경우 `KILL`신호를 사용하여 `worker`를 강제 종료 할 수 있습니다. 그러나 현재 실행중인 작업은 손실됩니다.

또한 프로세스가 `KILL`신호를 무시할 수 없지만 `children`(thread, proccess?)을 종료할 수 없습니다. 
이러한 경우에는 명령을 통해 수동으로 종료합니다.

```sh
pkill -9 -f 'celery worker'
# or
ps auxww | awk '/celery worker/ {print $2}' | xargs kill -9
```

### Restarting the worker
작업자를 다시 시작하려면 `TERM` 신호를 보내고 새 인스턴스를 시작 해야 합니다. 작업자를 관리하는 가장 쉬운 방법은 `celery multi`를 사용하는 것입니다.

```sh
celery multi start 1 -A proj -l INFO -c4 --pidfile=/var/run/celery/%n.pid
celery multi restart 1 --pidfile=/var/run/celery/%n.pid
```

프로덕션 환경에서는 `supervisor` 등(프로세스 관리 시스템)을 사용해 `worker`가 종료되지 않도록 관리해야 합니다.


------

회사에서도 `celery`를 통해 비동기 작업을 처리하고 있지만 특수한 방법으로 사용하다보니 종종 `message`가 사라지거나 `task`를 처리하는 중에 사라지는 경우가 종종 있었습니다. 또한 `celery`를 k8s로 관리하는데 `workload resource` 업데이트를 하면 `container`가 거의 곧 바로 죽어 작업을 처리하지 않고 사라지는 경우도 있엇습니다.
글을 작성하면서 배운 내용을 기반으로 `graceful shutdown`까지 적용해보려 합니다. 
> k8s에서 `Pod` 종료 이벤트의 순서는 `SIGTERM`을 1번 프로세스에 전송합니다. (`terminationGracePeriodSeconds` 30초)
>
> 유예 기간이 만료되어도 `Pod`이 삭제되지 않으면 `SIGKILL` 신호가 해당 프로세스로 전송되고 `Pod`가 API 서버에 삭제됩니다.
>
> 따라서, `terminationGracePeriodSeconds`을 적절하게 늘려주어 `graceful shutdown`이 되도록 해보려합니다.


## 참고자료
- [Brokers — Celery 5.0.2 documentation](https://docs.celeryproject.org/en/stable/getting-started/brokers/index.html#brokers)
- [CELERY에 대해(주의할 점) :: 쌀 팔다 개발자](https://daeguowl.tistory.com/157)
- [RabbitMQ ACK \| Ssup2 Blog](https://ssup2.github.io/theory_analysis/RabbitMQ_Ack/)
<https://tech.kakao.com/2018/12/24/kubernetes-deploy/>

