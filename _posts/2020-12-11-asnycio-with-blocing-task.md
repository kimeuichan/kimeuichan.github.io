---
title: Asyncio에서 동기 라이브러리 사용하기(feat.태스크와 코루틴)
author: daru
date: 2020-12-10 15:45:00
categories: [programming,python]
tags: [python,concurrency,asyncio]
---

회사에서 많은 양의 API를 처리해야하는 서버를 구축해야하는 경우가 생겼습니다.

프로젝트 셋업에 있어 빠른 처리를 위해 `Sanic` 프레임워크를 사용하기로 했습니다.

다만, [`celery`](https://docs.celeryproject.org/en/stable/index.html)를 호출해야하는 경우가 생겨 `asyncio`와 호환성을 찾다가
결국 발견하지 못해 `asyncio`를 이용한 해결방안을 포스팅합니다.

**찾아본 방법**
- `celery` 자체에서 `asyncio` [호환](https://stackoverflow.com/a/43325237/5944655)
  - 5 버전에서도 결국 지원을 하지 않는다.
- [`celery-pool-asyncio`](https://github.com/kai3341/celery-pool-asyncio)를 사용
  - 작동은 하지만, redis를 이용한 result backend를 [지원하지 않아](https://github.com/kai3341/celery-pool-asyncio/issues/30) 제외


## Sanic이란

[`Sanic`](https://sanic.readthedocs.io/en/latest/)은 파이썬 `asyncio`를 이용한 웹 프레임워크로 속도에 매우 초점을 맞춘 프레임 워크입니다.

비동기 기반으로 작동하며, `uvloop`를 사용할 수 있어 기본 파이선 `eventloop`를 활용한 것보다 좋은 성능을 끌어 낼 수 있습니다.


### Sanic의 단점
`asnycio`를 사용한 만큼 성능도 좋지만 단점이 여실히 들어나는 경우가 있는데 바로 `blocking` 함수를 호출한 경우 입니다.

`asnycio`와 마찬가지로 `sanic`에서도 `blocing` 함수를 사용하면 해당 쓰레드가 멈추고(단일 쓰레드인 경우) 다른 응답을 처리하지 못하는 경우가 생깁니다.

따라서, 사용하는 함수가 모두 `non-blocking` 함수여야 성능에 도움이 단점이 있습니다.

> 최근에는 `asyncio`를 이용한 라이브러리가 많았지만 몇 년전만 해도 그렇지 않았던 것으로 알고 있습니다.
> 최근 [asyncio에 대한 회의 - 영록이 홈페이지](http://youngrok.com/asyncio%EC%97%90%20%EB%8C%80%ED%95%9C%20%ED%9A%8C%EC%9D%98)을 재밌게 읽었습니다.
> 

## 해결법
해결법은 `loop.run_in_executor` 함수를 사용해서 `awaitable` 객체를 만들어 해결하려 했습니다.

해당 방법을 증명하기 위해 실험을 시도했는데 그 중에서 나온 삽질 경험을 적어보았습니다.


### 삽질
다음과 같은 간단한 코드를 작성하여 테스트 해보려 했습니다.

```python
import asyncio
import os
from concurrent.futures.thread import ThreadPoolExecutor

import requests


async def async_sleep():
    await asyncio.sleep(0.5)

    print(f"in async sleep task in {os.getpid()}")


def sync_cpu_bound_task():
    result = sum(i * i for i in range(10 ** 8))
    print(f"in heavy cpu bound in {os.getpid()}")

    return result


async def next_job():
    print(f"in coroutine task {os.getpid()}")


async def main():
    _loop = asyncio.get_event_loop()

    await _loop.run_in_executor(
        None,
        sync_cpu_bound_task
    )

    await next_job()
    await async_sleep()

if __name__ == '__main__':
    asyncio.run(main())
    
# 예상 출력은 다음과 같았습니다.
# in coroutine task ~
# in async sleep task in ~
# in heavy cpu bound in ~

# 실제 출력 값
# in heavy cpu bound in
# in coroutine task ~
# in async sleep task in ~
```


하지만 예상과는 다르게 첫번째 줄에서부터 바로 `blocking` 함수가 되버렸습니다.

Executor Pool 문제인가 싶어 `ProcessPool` 버전으로 재작성하였습니다.

```python
async def main():
    _loop = asyncio.get_event_loop()

    await _loop.run_in_executor(
        None,
        sync_cpu_bound_task
    )

    await next_job()
    await async_sleep()
    
#in heavy cpu bound in 10503
#in coroutine task 10503
#in async sleep task in 10503
#elapsed time: 9.846224784851074
```

결과는 보기좋게 제 예상과 빗나갔습니다.

둘다 같이 `await`을 쓸 수 있는 오브젝트지만, 다른 방식으로 작동하는 것 같아 예전에 썻던 [포스팅](https://kimeuichan.github.io/posts/python-asyncio/)을 다시 정독을 했습니다.

> asyncio의 핵심은 task(coroutine)를 관리이다.
> 


```python
print(next_job())
print(asyncio.create_task(next_job()))

#<coroutine object next_job at 0x10d7f6048>
#<Task pending coro=<next_job() running at /Users/gim-uichan/study/python/async-sync-test.py:28>> in coroutine task 10360
```

결국 태스크로 등록이 되기전까지는 의미가 없다는걸 느끼게 되었습니다. 

그래서 다음과 같이 다시 수정하여 테스트해보았고 의미있는 결과를 얻었습니다.


```python
    t1 = _loop.run_in_executor(
        None,
        sync_cpu_bound_task
    )

    t2 = asyncio.create_task(next_job())
    t3 = asyncio.create_task(async_sleep())

    await t1
    await t2
    await t3
    
#in coroutine task 10501
#in async sleep task in 10501
#in heavy cpu bound in 10501
#elapsed time: 9.368857145309448
```

보시는 바와 같이 실행 순서가 예상에 출력되었습니다.

## Task? Coroutine?
`Task`과 `Coroutine`은 뭐가 다르기에 다르게 동작하는지 궁금해졌습니다.

### [Coroutine](https://docs.python.org/ko/3/library/asyncio-task.html#coroutines)
공식문서에 아주 중요한 문구가 문구가 있습니다.

**코루틴을 기다리기. 다음 코드 조각은 1초를 기다린 후 《hello》를 인쇄한 다음 또 2초를 기다린 후 《world》를 인쇄합니다.**

실제로 코루틴을 `await` 한다고 해서 무조건 비동기적으로 코드가 실행되는 것이 아닙니다.

태스크로 등록이 되어야지만 비동기로 실행이 가능해집니다.


### [Task](https://docs.python.org/ko/3/library/asyncio-task.html#task-object)
`Task`는 `Future`의 한 종류로, 이벤트 루프에서 코루틴을 실행하는데 사용합니다.

만약 코루틴이 `Future`를 기다리고 있다면, 태스크는 코루틴의 실행을 일시 중지하고 `Future`의 완료를 기다립니다.
그동안 이벤트 루프는 다른 태스크, 콜백을 실행하거나 IO 연산을 수행합니다.(협업 스케줄링)

`Future`가 완료되면, 감싸진 코루틴의 실행이 다시 시작됩니다.

**이벤트 루프는 한 번에 하나의 `Task`를 실행합니다.**

### [Futrue](https://docs.python.org/ko/3/library/asyncio-future.html#asyncio.Future)
Future는 비동기 연산의 최종 결과를 나타냅니다.

Future는 어웨이터블 객체입니다. 코루틴은 결과나 예외가 설정되거나 취소될 때까지 `Future` 객체를 기다릴 수 있습니다.

일반적으로 퓨처는 저수준 콜백 기반 코드(예를 들어, `asyncio` 트랜스포트를 사용하여 구현된 프로토콜에서)가 고수준 `async/await `코드와 상호 운용되도록 하는 데 사용됩니다.

> 어웨이터블 객체란(Awaitable)
> `await` 표현식을 사용할 수 있는 객체로 `__await__` 메서드를 가진 객체나, 코루틴 등이 있습니다.


### 정리
![task-future-coroutine.png](/assets/img/posts/asnycio-with-blocing-task/task-coroutine-future.png)
> 해당 정리는 제가 그린 그림임으로 틀릴 수 있습니다.
> 잘못된 부분은 메일로 보내주시면 바로 수정하겠습니다.

#### 번외

**문득 cpu-bound task를 멀티프로세스로 실행시키면 어떻게 될지 궁금해졌습니다.**

```python
async def main():
    _loop = asyncio.get_event_loop()

    with ProcessPoolExecutor() as pool:
        t1 = _loop.run_in_executor(
            pool,
            sync_cpu_bound_task
        )

        t2 = asyncio.create_task(next_job())
        t3 = asyncio.create_task(async_sleep())

        await t1
        await t2
        await t3
        
#in coroutine task 11067
#in async sleep task in 11067
#in heavy cpu bound in 11068
#elapsed time: 10.087301969528198
```
cpu-bound task에 process context swicthing 때문인지 크게 다른점은 없었습니다.

다른점은 `os.getpid()`를 통해 확인할수 있듯이 다른 process로 돌아간다는 점이였습니다.

**`Sanic`에서 `Task`을 사용한 방법과 코루틴을 이용한 방법에 성능적 차이가 있을지 궁금했졌습니다.**
> 테스트 환경
>
> Macbook pro 2018 15 inch
>
> 2.6 GHz 6코어 Intel Core i7
>
> 16GB 2400 MHz DDR4
>
> Mojave Python 3.7
>
> Jmeter

```python
import asyncio

from sanic import Sanic
from sanic.response import text

app = Sanic(__name__)


async def sleep_task():
    await asyncio.sleep(3)


@app.route("/coroutine")
async def coroutine(req):
    await sleep_task()

    return text("hello world")


@app.route("/task")
async def task(req):
    t = asyncio.create_task(sleep_task())
    await t
    return text("hello world")


app.run("0.0.0.0", port=8080, debug=True)

```
**sleep 함수** 결과

*코루틴 결과*

![sanic-sleep-coroutine.png](/assets/img/posts/asnycio-with-blocing-task/sanic-sleep-coroutine.png)

*태스크 결과*

![sanic-sleep-task.png](/assets/img/posts/asnycio-with-blocing-task/sanic-sleep-task.png)


```python
import asyncio

from aiohttp import ClientSession
from sanic import Sanic
from sanic.response import text

app = Sanic(__name__)


async def sleep_task():
    await asyncio.sleep(3)


async def network_task():
    async with ClientSession() as session:
        await session.get("https://wtfismyip.com/json")


@app.route("/coroutine")
async def coroutine(req):
    await network_task()

    return text("hello world")


@app.route("/task")
async def task(req):
    t = asyncio.create_task(network_task())
    await t
    return text("hello world")


app.run("0.0.0.0", port=8080, debug=True)

```

**aiohttp GET** 결과

*코루틴 결과*

![sanic-http-coroutine.png](/assets/img/posts/asnycio-with-blocing-task/sanic-http-coroutine.png)

*태스크 결과*

![sanic-http-task.png](/assets/img/posts/asnycio-with-blocing-task/sanic-http-task.png)


## Celery와 같이 쓰기
이제 해당 포스팅의 주제인 `celery`와 같이 써보겠습니다.

`celery`의 설정 별로 물론 결과가 달라질 수 있으나 일반적으로 많이 쓰는 `rabbitmq` **브로커**와 `redis` **result backend**를 사용했습니다.

`worker`에서는 간단하지만 cpu-bound인 팩토리얼을 돌려봤습니다.

```python
import math

from celery import Celery


celery_task = Celery(broker="amqp://guest:guest@localhost:5672", backend="redis://localhost:6379/0")


@celery_task.task
def heavy_task():
    return math.factorial(20)
```

jmeter를 통해 100 threads X 5번 을 실행하여 결과를 내보았습니다.


### Executor에서 실행하기
파이썬 asyncio 중 [`loop.run_in_executor(pool, func)`](https://docs.python.org/ko/3/library/asyncio-eventloop.html#asyncio.loop.run_in_executor)라는 함수가 있습니다.

해당 함수는 `blocing` 함수를 `awaitable`하게 돌릴 수 있도록 task로 만들어주는 함수입니다.

또한, 어느 `pool`에서 돌릴건지도 설정이 가능하며 loop내의 기본 executor 또는 thread executor, process executor 등을 지원합니다.

해당 함수를 이용해서 코드는 다음과 같습니다.

```python
@app.route("/task")
async def task(req):
    celery_result = heavy_task.delay()
    loop = asyncio.get_event_loop()
    result = await asyncio.wait_for(loop.run_in_executor(None, celery_result.get), timeout=None)

    return text(f"hello world: {result}")
```

하지만, 해당 기본 executor나 thread pool, process pool에서 돌려도 동작하지 않았습니다.

대부분 `run_in_executor` 내부에서 어떠한 로직때문인지는 모르겠지만 `celery` 내부 동작 코드가 작동하지 않았습니다.

아래와 같은 exception이 발생하며 출력되며 에러율은 16%로 기록되었습니다.

- `redis.exceptions.InvalidResponse: Protocol Error: b'date_done": "2020-12-10T14:29:20.335394", "task_id": "41614cdf-5b70-443a-915f-6f0cb9683fd7"}'`
- `redis.exceptions.InvalidResponse: Protocol Error: b'CCESS", "result": 2432902008176640000, "traceback": null, "children": [], "date_done": "2020-12-10T14:29:16.887731", "task_id": "d6c483af-611e-468e-8fb2-036c6d5ee0bf"}'`
- `redis.exceptions.InvalidResponse: Protocol Error: b'\x00\x00...\x00\x00*3'`

문제는 exception만 발생한게 아니라 서버가 멈춰버리는 경우가 있어 500번의 요청을 모두 처리하지 못했습니다.

기다리면 처리 갯수는 올라갔으나, 의미가 없다고 판단해 모두 스톱하였습니다.

![sanic-celery-run-in-executor.png](/assets/img/posts/asnycio-with-blocing-task/sanic-celery-run-in-executor.png)


### `awaitable`을 이용해 돌리기
효율이 가장 좋을거라고 판단했던 `run_in_executor`를 쓰지 못하게 되어, 꿩 대신 닭이라고

코루틴을 이용해서 처리라도 해보자라는 생각이 들어 코루틴으로 작성하였습니다.

**Coroutine**

가장 기본적인 코루틴을 이용하도록 다음과 같이 작성해서 테스트 해보았습니다.
```python
@app.route("/coroutine")
async def coroutine(req):
    celery_result = heavy_task.delay()

    async def _celery_await():
        return celery_result.get()

    celery_result = await _celery_await()
    return text(f"hello world: {celery_result}")
```

결과는 다음과 같습니다.
![sanic-celery-coroutine-fast.png](/assets/img/posts/asnycio-with-blocing-task/sanic-celery-coroutine-fast.png)


신기하게도 아주 가끔 블록킹 되듯이 서버가 느려지는 경우가 있는 것을 확인했습니다.
![sanic-celery-coroutine-slow.png](/assets/img/posts/asnycio-with-blocing-task/sanic-celery-coroutine-slow.png)


**Task**

위의 코루틴 예제를 이용하여 태스크를 만들고 실행하도록 작성하였습니다.

태스크를 만드는데 간단한 몇가지 방법이 있는데 제가 사용한 방법은 아래 두가지입니다.
- `wait_for`
- `create_task`

**wait_for** 버전

[`wait_for(aw, timeout, *, loop=None)`](https://docs.python.org/ko/3/library/asyncio-task.html#asyncio.wait_for)의 함수의 경우 공식홈페이지에 다음과 같이 적혀있습니다.
> aw가 코루틴이면 자동으로 태스크로 예약됩니다.

```python
@app.route("/task")
async def task(req):
    celery_result = heavy_task.delay()

    async def test():
        return celery_result.get()

    result = await asyncio.wait_for(test(), timeout=None)

    return text(f"hello world: {result}")
```


막상 돌려보니 코루틴 버전과 크게 퍼포먼스 차이가 나지않았습니다.
![sanic-celery-wait-for.png](/assets/img/posts/asnycio-with-blocing-task/sanic-celery-wait-for.png)


왜 이러한 결과가 나타났는지 알아보기위해 `wait_for` 함수를 찾아보았습니다. 그리고 아래와 같은 코드를 발견하였습니다.

```python
    if timeout is None:
        return await fut
```

`wait_for`을 호출해 태스크를 만들 때는 `timeout` 파라미터를 주어야지만 태스크로 동작한다는 것을 알게되었습니다.

다음과 같이 소스코드를 수정한 후 다시 테스트 해보았습니다.

```python
@app.route("/task")
async def task(req):
    celery_result = heavy_task.delay()

    async def test():
        return celery_result.get()

    result = await asyncio.wait_for(test(), timeout=2)

    return text(f"hello world: {result}")
```

일반 코루틴 보다 빠른 속도를 확인 할 수 있었습니다.

![sanic-celery-wait-for-update.png](/assets/img/posts/asnycio-with-blocing-task/sanic-celery-wait-for-update.png)


**create_task** 버전

[`create_task(coro, *, name=None)`](https://docs.python.org/ko/3/library/asyncio-task.html#asyncio.create_task)는 가장 간단하게 태스크를 만들 수 있는 간단한 함수입니다.

```python
@app.route("/task")
async def task(req):
    celery_result = heavy_task.delay()

    async def test():
        return celery_result.get()

    result = await asyncio.create_task(test())

    return text(f"hello world: {result}")
```

`wait_for` 함수보다 조금 더 좋은 속도가 나왔습니다.

> 아마 로직이 조금 덜 타기 때문이지 않을까라고 예상하고있습니다.

![sanic-celery-create-task.png](/assets/img/posts/asnycio-with-blocing-task/sanic-celery-create-task.png)


**Non-blocking**

논외로 그냥 일반 함수로 실행하면 어떻게 될지 궁금하여 실행해보았습니다.

```python
@app.route("/blocking")
async def blocking(req):
    celery_result = heavy_task.delay()
    result = celery_result.get()
    return text(f"hello world: {result}")
```

코루틴 버전과 거의 동일한 속도가 나오는 것을 확인할 수 있엇습니다.

![sanic-celery-blocking.png](/assets/img/posts/asnycio-with-blocing-task/sanic-celery-blocking.png)


> 제일 빠를 거라고 예상했던 `loop.run_in_executor` 함수가 동작하지 않아 매우 아쉬웠습니다.
> result backed 따라 다른가 싶어 `redis`, `rpc` 두 가지 모두 테스트하였으나 모두 동작하지 않았습니다.
> `redis` 보다 `rpc`가 조금 속도면에서 우세하였으나 많은 요청량에서 에러율이 올라가는 상황이 있었습니다.
> `loop.run_in_executor`가 돌지 않는 이유와 `rpc`에서 많은 요청은 에러율이 올라가는지는 나중에 다뤄보도록 하겠습니다.

## 결론
웹 서버 등(IO 작업이 많은 앱)에서 `blocking` 함수를 실행하고 싶은 경우에는 `run_in_executor`나 `create_task` 함수를 통해 

최소한이라도 태스크 객체를 만들어 돌리는 것이 일반 `Non-blocking` 함수를 사용하는 것보다 효율이 좋다.


#### 참고 자료
- [코루틴과 태스크 — Python 3.9.1rc1 문서](https://docs.python.org/ko/3/library/asyncio-task.html)
- [용어집 — Python 3.9.1rc1 문서](https://docs.python.org/ko/3/glossary.html#term-awaitable)
- [python - What is the overhead of an asyncio task? - Stack Overflow](https://stackoverflow.com/a/55766474/5944655)