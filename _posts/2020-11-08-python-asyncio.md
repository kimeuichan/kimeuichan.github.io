---
title: Python에서 asyncio의 동작
author: daru
date: 2020-11-08 21:00:00
categories: [programming,python]
tags: [python,concurrency,asyncio]
---

[python - How does asyncio actually work? - Stack Overflow](https://stackoverflow.com/questions/49005651/how-does-asyncio-actually-work)

해당글을 번역한 글입니다.

파이썬에서 `asnyc`는 `coroutine`을 기반으로 동작합니다.
`coroutine`의 동작을 이해하기 전 `generator`의 개념을 알고가면 이해하기 쉽기 때문에 먼저 `generator` 개념 부터 설명하겠습니다.


## Generator
`generator`는 `yield` 문법을 사용하여 구현한 함수 형태입니다. 독특한 점은 `yield`를 통해 함수에서 다시 호출한 곳으로 제어권을 넘긴다는 점입니다.
```python
def test():
    yield 1
    yield 2
    yield 3
    yield 4
    
a = test()
print(next(a))
#1
print(next(a))
#2
print(next(a))
#3
print(next(a))
#4
print(next(a))
```


`generator`에서 `next()`를 호출하면 인터프리터가 `test`의 프레임을 로드하고 산출된 값을 반환합니다. 
`next()`를 다시 호출하면 `test` 프레임이 인터프리터 스택에 다시 로드되고 계속해서 다른 값을 산출합니다.

4번의 `next()`를 호출하면 `generator`는 `StopIteration` 예외을 발생시켜 `generator`가 끝났음을 알립니다.

### Generator 통신하기
`generator`은 `send()` 및 `throw()`의 두 가지 메서드를 사용하여 이들과 통신 할 수 있습니다.

```python
def test():
    a = yield 1
    print(a)
    yield 2

g = test()

print(next(a))
# 1
print(next(a))
# None
# 2
print(next(a))
# StopIteration

g = test()
print(next(a))
# 1
g.send(123)
# 123
# 2
```

`send`로 값을 보내면 `yield` 문법으로 값을 돌려받는 받습니다.

`g.throw()`는 `generator` 내부에서 `yield` 가 있는 위치에서 `Exception`을 던질 수 있도록 허용합니다. 

```python
def test():
    a = yield 1
    print(a)
    yield 2

g = test()

print(next(a))
# 1
g.throw(Exception())
# Traceback (most recent call last):
#   File "<stdin>", line 2, in <module>
#   File "<stdin>", line 2, in test
# Exception
```

### `Generator`로부터 반환값 받기
`generator`로부터 반환값을 받으려면 결과값은 `StopIteration` 예외 내부에서 값을 할당 받아야 한다.
나중에 예외에서 값을 복구하여 필요에 따라 사용할 수 있습니다.
```python
def test():
    yield 1
    return "hello"

g = test()

print(next(a))
# 1

try:
    print(next(a))
except StopIteration as e:
    e.value
    # hello
```

### 새로운 문법 `yield from`
`Python3.4`에서 [`yield from`](https://docs.python.org/3/reference/expressions.html#yield-expressions)이라는 새로운 문법이 소개 되었다.
`yield from`은 가장 안쪽의 `generator`에게 `next()`, `send()`, `throw()`를 넘겨주는 것을 할 수 있다.
만약 내부의 `generator`가 `return`했다면 `yield from`으로 인해 외부의 `generator`도 return한다
```python
def inner():
    a = yield 1

    print(a)

    return "done"


def outer():
    yield 2
    val = yield from inner()
    print(val)
    yield 4


g = outer()

print(next(g))
# 2
print(next(g))
# 1
g.send("hello")
# hello
# done
print(next(g))
#Traceback (most recent call last):
#  File "/Users/gim-uichan/study/python_async/test.py", line 26, in <module>
#    print(next(g))
#StopIteration: hi
```
## Coroutine
`yield from`문을 사용하여 `generator`에서 터널처럼 내부 `generator`를 사용할 수 있게 되었습니다.
데이터를 가장 안쪽에서 바깥 쪽 `generator`로 앞뒤로 전달합니다. 이것은 `generator`에게 새로운 의미를 낳았습니다.
우리가 알고 있는 `couroutine`과 가장 비슷한 개념이 생겼습니다.

`coroutine`은 실행 도중 일시 정지와 재 시작할 수 있는 함수입니다. 파이썬에서는 `async def`를 사용해서 만들 수 있습니다.
`generator`와 유사하게 `yield from`과 비슷한 `await`을 사용합니다.
Python 3.5에서 `async`, `await`이 소개되기 전 `generator`와 같은 방식으로 `coroutine`을 만들었습니다.(`await` 대신에 `yield from`)
`coroutine`은 `await coro`가 호출될 때마다 실행될 수 있도록 `__await __()`를 구현합니다.


### Coroutine chain
[18.5.3. Tasks and coroutines — Python 3.5.10 documentation](https://docs.python.org/3.5/library/asyncio-task.html#example-chain-coroutines)

```python
import asyncio

async def compute(x, y):
    print("Compute %s + %s ..." % (x, y))
    await asyncio.sleep(1.0)
    return x + y

async def print_sum(x, y):
    result = await compute(x, y)
    print("%s + %s = %s" % (x, y, result))

loop = asyncio.get_event_loop()
loop.run_until_complete(print_sum(1, 2))
loop.close()
```

`compute()`는 `print_sum()`에 연결되어있습니다. 
`print_sum()` 은 `compute()`가 끝나 결과를 반환할 때까지 기다립니다.

![tulip_coro.png](https://docs.python.org/3.5/_images/tulip_coro.png)


**Task**는 [`AbstractEventLoop.run_until_complete()`](https://docs.python.org/3.5/library/asyncio-eventloop.html#asyncio.AbstractEventLoop.run_until_complete) 메소드에서 `task`가 아닌 `coroutine` 객체를 통해 성성됩니다.


`asyncio` `coroutine`에서 중요한 2가지 object `tasks`, `futures`가 있습니다.

### Futures
[`futures`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Future)는 `__await__()` 메소드를 구현한 객체입니다. 그들의 임무는 특정 상태와 결과를 유지하는 것입니다. 상태는 다음 중 하나 일 수 있습니다.
- PENDING: `future`가 결과나 예외를 갖고 있지 않음.
- CANCELLED: `future`가 `fut.cancel()`에 의해 취소됨.
- FINISHED: 결과 값이 [`fut.set_result()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Future.set_exception)로 설정되거나 예외가 [`fut.set_exception()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Future.set_exception)로 설정되어 `future`가 종료됨.

`future`의 결과는 반환될 Python 객체거나 발생할 수 있는 예외입니다.

`future` 객체의 또 다른 중요한 기능은 [`add_done_callback()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Future.add_done_callback)이라는 메서드를 포함한다는 것입니다. 
이 메서드를 사용하면 예외가 발생 했든 완료 되었든 작업이 완료되는 즉시 `callback` 함수를 호출합니다.


### Tasks

[`task`](https://docs.python.org/3/library/asyncio-task.html#task) 객체는 `coroutine`을 감싸고 가장 안쪽 및 가장 바깥 쪽 `coroutine`과 통신하는 특별한 `futures`입니다. `coroutine`이 `future`를 기다릴 때마다 `future`는 작업으로 완전히 다시 전달되고 (`yield from` 과 같이) `task`은 이를 수신합니다.

`task`은 `future`에 `add_done_callback()`을 호출하여 자신을 `future`에 바인딩합니다. 
`future`가 예외를 던지거나 결과값을 반환하여 완료되면 `task`가 `future`에 바인딩했던 콜백이 호출되고 다시 실행하게됩니다.

## Asyncio
`asyncio` 내부에는 `task` 이벤트 루프가 있습니다. 이벤트 루프의 역할은 `task`이 준비 될 때마다 `task`을 호출하고 모든 `task`들이 단일 머신에서 돌아가도록 제어합니다.

이벤트 루프의 IO 부분은 [`select`](https://docs.python.org/3/library/select.html#module-select)라는 하나의 중요한 기능을 기반으로합니다. `Select`는 운영 체제 아래에서 구현된 블록킹 함수로, 소켓에서 들어 오거나 나가는 데이터를 대기 할 수 있습니다. 데이터가 수신되면 깨어나서 데이터를 수신한 소켓을 반환하거나 쓰기 준비가 된 소켓을 반환합니다.

`asyncio`를 통해 소켓을 통해 데이터를 받거나 보내려고 할 때 맨 처음 일어나는 일은 소켓에 즉시 읽거나 보낼 수 있는 데이터가 있는지 먼저 확인하는 것입니다. `.send()` 버퍼가 가득 차거나 `.recv()` 버퍼가 비어있는 경우 소켓은 `select` 함수에 등록됩니다 (단순히 목록 중 하나에 추가, recv의 경우 rlist, 송신의 경우 wlist). 함수는 해당 소켓에 연결된 새로 생성 된 `future` 객체를 기다립니다.

사용 가능한 모든 `task`가 `future`를 기다리고있을 때 이벤트 루프는 `select`를 호출하고 대기합니다. 소켓 중 하나에 들어오는 데이터가 있거나 전송 버퍼가 비워지면 `asyncio`는 해당 소켓에 연결된 `future` 객체를 확인하고 완료로 설정합니다.

이제 모든 마법이 일어납니다. `future`는 완료로 설정되고 `add_done_callback()`으로 이전에 추가 된 작업이 다시 살아 나고 가장 안쪽의 `coroutine`을 재개하는 `coroutine`에서 `.send()`를 호출합니다 (`await` 체인으로 인해). 근처 버퍼에서 새로 수신된 데이터가 유출되었습니다.

`recv()`의 경우 다시 메서드 체인 :

1. `select.select`가 대기합니다.
2. 데이터가있는 준비 소켓이 리턴됩니다.
3. 소켓의 데이터는 버퍼로 이동됩니다.
4. `future.set_result()`가 호출됩니다.
5. `add_done_callback()`으로 자신을 추가한 `task`가 깨어납니다.
6. `task`는 `coroutine`에서 `.send()`를 호출하여 가장 안쪽의 `coroutine`을 깨웁니다.
7. 데이터는 버퍼에서 읽고 겸손한 사용자에게 반환됩니다.

요약하면 `asyncio`는 기능을 일시 중지하고 다시 시작할 수 있는 `coroutine` 기능을 사용합니다. 가장 안쪽의 `coroutine`에서 바깥쪽으로 데이터를 앞뒤로 전달할 수있는 기능의 `yield from`를 사용합니다. IO가 완료되기를 기다리는 동안 (OS `select` 기능을 사용하여) 기능 실행을 중지하기 위해이 모든 것을 사용합니다.

그리고 무엇보다도 한 기능이 일시 중지 된 동안 다른 기능이 실행되고 `asyncio`와 교차로 실행될 수 있습니다.

## Event loop
`asyncio`의 핵심으로 `task`(`coroutine`)를 관리하는 전략이 핵심이다.

> 이벤트 루프는 프로그램 에서 이벤트 또는 메시지 를 기다렸다가 전달합니다. 내부 또는 외부 "이벤트 공급자"(일반적으로 이벤트가 도착할 때까지 요청을 차단 함) 에 요청한 다음 관련 이벤트 처리기를 호출 합니다 ( "이벤트 디스패치").

#### 핵심 기능
- 스케줄링
- 네트워크를 통한 데이터 전송
- DNS 쿼리
- OS signal 핸들링
- 서버와 커넥션을 만드는 편리한 추상화
- 비동기적 서브프로세스(코루틴을 뜻하는것일까?)
- 호출 등록, 실행 및 취소

> ~~macOS Python3.7 버전에서는 I/O Multiplexing을 [`select`](https://en.wikipedia.org/wiki/Select_(Unix))를 사용하는 것으로 보입니다.~~
>
> `select` 함수를 사용하는 것이 아니라 python `selector` 모듈에 환경에 맞 멀티 I/O 플렉싱 모델을 기반으로 사용합니다.
> ```python
> import asyncio
> print(asyncio.get_event_loop())
> # <_UnixSelectorEventLoop running=False closed=False debug=False>
> ```


#### 함께보면 좋을 자료
[파이썬 Asyncio 를 이해하기 위한 여정](https://hamait.tistory.com/834)
[Python의 asyncio를 직접 만들어보자 (3) - 이상한모임](https://blog.weirdx.io/post/56921)
[I/O Multiplexing(select, poll, epoll, kqueue) \| Mimul Tech log](https://www.mimul.com/blog/io-multiplexing/)
[Understanding event loop with Python \| by Ilya Pekelny | Medium](https://medium.com/@pekelny/fake-event-loop-python3-7498761af5e0)
[libuv-vs-libev.md · GitHub](https://gist.github.com/andreybolonin/2413da76f088e2c5ab04df53f07659ea)


#### 참고자료
[Python Generators/Coroutines/Async IO with examples \| by Alex Anto Navis L | Analytics Vidhya | Medium](https://medium.com/analytics-vidhya/python-generators-coroutines-async-io-with-examples-28771b586578)
[18.5.3. Tasks and coroutines — Python 3.4.10 documentation](https://docs.python.org/3.4/library/asyncio-task.html#asyncio.coroutine)
