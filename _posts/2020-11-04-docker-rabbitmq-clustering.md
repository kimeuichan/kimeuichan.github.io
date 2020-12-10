---
title: Docker RabibtMQ Clustering 하기
author: daru
date: 2020-11-04 20:00:00
categories: [devops]
tags: [rabbitmq,docker]
---

## Docker RabibtMQ Clustering 하기
---

### 왜 스케일 업 대신 클러스터링?
회사에서 `rabbitmq`를 사용하는데 해당 작업들이 매우 느려 `rabbitmq`의 메모리가 부족한
현상이 가끔 발견되었습니다. 따라서 스케일링과 클러스터링을 고려하게 되었습니다.

클러스터링을 하게된 4가지 이유가 아래와 같습니다.
1. 무중단 스케일링이 힘들다.
2. 유실된 메시지를 찾는것이 매우 힘들다.
3. 나중을 위해선 `HA` 구성을 해야 한다.
4. rabbitmq에서 clustering을 기본적으로 잘 지원


### Clustering 하기
회사에서는 `rabbitmq`와 해당 `rabbitmq` 체크를 위한 `rabbitmq-exporter`를 관리하기 위해 `docker-compose`로 컨테이너를 관리하고 있습니다.


#### Erlang cookie 맞추기
`rabbitmq`에서는 `Erlang Cookie`라는 값을 통하여 각 노드끼리 인증을 합니다.

해당 경로의 값을 `$HOME/.erlang.cookie` 복사하여 다른 노드에 같은 경로에 덮어씁니다.



#### Hostname Resolution
`rabbitmq`에서 `clustering`을 하기 위해 가장 중요한 것은 조인할 `node`를 디스커버리 하는 것입니다.
`/etc/hosts`에 조인할 `node`를 [등록해야 한다.](https://www.rabbitmq.com/clustering.html#hostname-resolution-requirement)

> `docker` 내부에서 `rabbitmq`를 사용중이라면 컨테이너 내부에서 수정을 해야합니다.

```
X.X.X.X   join-rabbitmq
```


그 다음 제대로 통신이 되기 위해서는 linux builtin `echo` 명령어를 사용하여 확인할 수 있습니다.

```shell
echo > /dev/tcp/join-rabbitmq/4639
```

아무런 메시지가 뜨지 않는다면 `tcp` 통신이 성공한 것입니다.
해당 명령어에서 오류가 난다면 클러스터링이 진행되지 않습니다.

#### Clustering
클러스터링 과정은 다음과 같습니다.
```shell
# 노드 중지
rabbitmqctl stop_app

# 클러스터 조인
rabbitmqctl join_cluster rabbit@join-rabbitmq

# 노드 시작
rabbitmqctl start_app

# 클러스터링 확인
rabbitmqctl cluster_status
```

##### 참고자료
[RabbitMQ 클러스터 구성하기 :: 조은우 개발 블로그](https://jonnung.dev/rabbitmq/2019/08/08/rabbitmq-cluster/)
