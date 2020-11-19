---
title: Python의 GIL
author: daru
date: 2020-11-19 21:00:00
categories: [programming,python]
tags: [python,concurrency]
---


해당글은 [What Is the Python Global Interpreter Lock (GIL)? – Real Python](https://realpython.com/python-gil/)을 번역 및 정리한 포스팅입니다.


## GIL이란
Global Interpreter Lock의 약자입니다.
한 쓰레드만을 사용해 파이썬 인터프리터를 제어하게 하는 `mutex`(or `lock`) 입니다.

어느 시점에서든 하나의 스레드 만 실행 상태에있을 수 있습니다. GIL의 영향은 단일 스레드 프로그램을 실행하는 개발자에게 보이지 않지만 CPU 바인딩 및 다중 스레드 코드에서 성능 병목 현상이 될 수 있습니다.

GIL은 하나 이상의 CPU 코어가있는 다중 스레드 아키텍처에서도 한 번에 하나의 스레드 만 실행할 수 있도록 허용해 많은 파이썬 앱에 성능 이슈를 끼칩니다.


## 왜 GIL이 생겼을까
**Python은 메모리 관리를 위해 변수(container object)의 참조 횟수를 계산을 사용합니다.** 

즉, 파이썬에서 생성 된 객체에는 객체를 가리키는 참조 수를 카운팅하는 참조 변수가 있습니다. 이 카운트가 0에 도달하게 되면 객체가 메모리에서 해제됩니다.

```python
import sys


a = []
b = a

print(sys.getrefcount(a))
# 3
del b

print(sys.getrefcount(a))
# 2
```

위의 예제에서 빈 리스트 객체`[]`의 참조 횟수는 `a, b, sys.getrefcount()` 인수에서 사용하기 때문에 때문에 3입니다.

문제는 참조 카운트 변수를 두 쓰레드가 동시에 값을 늘리거나 줄이는 경쟁 조건으로부터 보호해야한다는 것입니다.
이런 일이 발생하면 메모리 릭 또는 해당 객체에 참조하는 동안 메모리에서 사라질 수 있습니다.

해당문제를 해결하고자 `Lock`을 추가할 경우 각 개체 또는 개체 그룹에 잠금을 추가하면 교착 상태를 유발할 수 있습니다.
또한 모든 변수에 `Lock` 만들어야 함으로 반복적인 잠금 및 해제 때문에 성능 저하 일으킬 수 있습니다.

이러한 문제를 해결하고자 Python은 인터프리터 자체에 대한 단일 잠금으로 Python 바이트 코드를 실행하려면 인터프리터 잠금(GIL)을 획득해야한다는 규칙을 추가했습니다. 
GIL은 교착 상태를 방지하고 (잠금이 하나뿐이므로) 성능 오버 헤드를 많이 발생시키지 않습니다. 하지만 모든 CPU 바인딩 Python 프로그램을 단일 스레드로 만들었습니다.


## 왜 GIL을 사용하는가
`Python`은 운영 체제에 쓰레드 개념이 없었던 시절부터 사용되었습니다. `Python`은 개발 속도를 높이기 위해 사용하기 쉽게 설계되었으며 점점 더 많은 개발자가 사용하기 시작했습니다.

- Python에 기존 C extension을 추가하기 쉬워졌습니다.
- Process context swtiching의 overhead가 사라져 단일 스레드 프로그램의 성능이 향상되었습니다.
- 쓰레드로부터 안전하지 않은 C 라이브러리는 통합하기가 더 쉬워졌습니다.


## 멀티 쓰레드를 사용하는 Python 프로그램에 미치는 영향
일반적인 Python 프로그램을 살펴보면 성능면에서 CPU 바인딩 된 프로그램과 `I/O` 바인딩 된 프로그램간에 차이가 있습니다.

CPU 바운드 프로그램은 CPU를 한계까지 밀어 붙이는 프로그램입니다. 행렬 곱셈, 검색, 이미지 처리 등과 같은 수학적 계산을 수행하는 프로그램이 포함됩니다.

`I/O` 바인딩 된 프로그램은 사용자, 파일, 데이터베이스, 네트워크 등에서 올 수있는 `I/O`을 기다리는 데 시간을 소비하는 프로그램입니다.
`I/O` 바인딩 된 프로그램은 CPU를 사용은 하지 않지만 프로그램 실행을 기다립니다.
예를 들어 사용자가 입력 프롬프트에 입력 할 내용을 생각하거나 데이터베이스 쿼리가 프로세스가 대표적입니다.

카운트 다운을 수행하는 간단한 CPU 바인딩 프로그램을 살펴 보겠습니다.

```python
import time

COUNT = 50000000


def countdown(n):
    while n>0:
        n -= 1


start = time.time()
countdown(COUNT)
end = time.time()

print('Time taken in seconds -', end - start)
# Time taken in seconds - 2.2837860584259033
```

이제 두 개의 스레드를 병렬로 사용하도록 코드를 수정 후 실행하면 다음과 같습니다.

```python
import time
from threading import Thread

COUNT = 50000000


def countdown(n):
    while n > 0:
        n -= 1


t1 = Thread(target=countdown, args=(COUNT // 2,))
t2 = Thread(target=countdown, args=(COUNT // 2,))

start = time.time()

t1.start()
t2.start()

t1.join()
t2.join()

end = time.time()

print('Time taken in seconds -', end - start)
# Time taken in seconds - 2.226407051086426
```


보시다시피 완료하는 데 거의 같은 시간이 걸립니다. 멀티 쓰레드 코드에서 GIL은 CPU 바운드 스레드가 병렬로 실행되는 것을 막습니다.

**GIL은 `I/O`를 기다리는 동안 스레드간에 잠금이 해제되기 때문에 `I/O` 바인딩 된 다중 스레드 프로그램의 성능에 큰 영향을주지 않습니다.**

> [multithreading - Why is a Python I/O bound task not blocked by the GIL? - Stack Overflow](https://stackoverflow.com/a/29270846/5944655)
> [Initialization, Finalization, and Threads — Python 3.9.0 documentation](https://docs.python.org/3/c-api/init.html#thread-state-and-the-global-interpreter-lock)


그러나 쓰레드가 전체적으로 CPU 바운드인 프로그램 (쓰레드를 사용하여 이미지를 부분적으로 처리하는 프로그램)은 잠금으로 인해 단일 스레드가 될뿐만 아니라 실행 시간도 증가합니다.

이 증가는 잠금에 의해 추가 된 오버 헤드 획득 및 해제의 결과입니다.

#### 번외
multi proccess, 4개의 쓰레드를 사용한 코드를 작성해 비교해보았습니다.(밑에 내용에 이미 포함되어 있엇네요.)

**multi process**
```python
import time
from multiprocessing import Process

COUNT = 50000000


def countdown(n):
    while n > 0:
        n -= 1


t1 = Process(target=countdown, args=(COUNT // 2,))
t2 = Process(target=countdown, args=(COUNT // 2,))

start = time.time()

t1.start()
t2.start()

t1.join()
t2.join()

end = time.time()

print('Time taken in seconds -', end - start)
# Time taken in seconds - 1.1550788879394531
```

**Nth thread**
macos Mojave Python 3.7 기준으로 흥미로운 결과가 나왔습니다. 4 쓰레드보단 1쓰레드가 빠르지만
1 쓰레드는 2 쓰레드보다 느리게 나왔습니다.

*1 thread*
2.202889919281006
2.2208809852600098
2.2509100437164307
2.1846389770507812
2.2133240699768066

avg: 2.214528799


*2 thread*
2.1756350994110107
2.184033155441284
2.1663358211517334
2.2320470809936523
2.2731218338012695

avg: 2.206234598


*4 thread*
2.1956586837768555
2.218824863433838
2.3495471477508545
2.388563871383667
2.186440944671631


avg: 2.267807102

> 구글링을 해봐도 구글링 실력이 부족한 탓인지 [stackoverflow에 질문을 남겼습니다](https://stackoverflow.com/questions/64911551/python-single-thread-is-slower-than-two-thread-in-cpu-bound)....


## GIL이 아직까지 남아있는 이유
`Python` 개발자는 이에 대해 많은 불만을 받고 있지만 `Python`만큼 인기있는 언어는 이전 버전의 비 호환성 문제를 일으키지 않고 GIL 제거만큼 중요한 변화를 가져올 수 없습니다.

GIL은 분명히 제거 될 수 있으며 이는 과거에 개발자와 연구원에 의해 여러 번 수행되었지만 이러한 시도는 GIL이 제공하는 솔루션에 크게 의존하는 기존 C extension을 손상 시켰습니다.

물론 GIL이 해결하는 문제에 대한 다른 솔루션이 있지만 그중 일부는 단일 스레드 및 다중 스레드 `I/O` 바인딩 프로그램의 성능을 저하시키거나 어려웠습니다. 

## Python 3에서 제거되지 않은 이유는?
Python 3은 처음부터 많은 기능을 시작할 수있는 기회를 가졌고 그 과정에서 기존 C extension 중 일부를 깨뜨려 Python 3에서 작동하도록 업데이트 및 이식했습니다. 그러나 안정성이 뛰어나지 않아 초기 버전의은 커뮤니티에서 채택 속도가 느렸습니다.

### 그럼에도 GIL이 함께 제거되지 않은 이유는?

GIL을 제거하면 단일 스레드 성능에서 Python2에 비해 Python3이 느려질 수 있으며 그 결과로 어떤 결과가 나올지 상상할 수 있습니다.
GIL의 단일 스레드 성능 이점에 대해 논쟁 할 수 없고, GIL을 제거하지 않았습니다.

#### Python 멀티 쓰레드를 사용하는데 가장 큰 문제점 및 개선책
CPU 바운드 쓰레드에서 GIL을 I/O 바운드 스레드가 얻을 수 있는 기회를주지 않음으로써 고갈시키는 것으로 입니다.

이를 해결하기 위해 Python은 일정 주기별로 스레드의 GIL을 강제로 릴리즈하는 메커니즘을 사용햇습니다. 
다른 thread가 GIL을 획득하지 않으면 동일한 스레드가 계속 사용할 수 있습니다. 

```python
import sys
# The interval is set to 100 instructions:
# sys.getcheckinterval()는 Python 3.7 기준으로 deprecated 되어 getswitchinterval()를 사용했습니다.
print(sys.getswitchinterval())
# 0.005
```

이 메커니즘의 문제는 대부분의 경우 CPU 바운드 스레드가 다른 스레드가 GIL을 획득하기 전에 GIL 자체를 다시 획득한다는 것입니다.

이 문제는 삭제 된 다른 스레드의 GIL 획득 요청 수를 확인하고 다른 스레드가 실행되기 전에 현재 스레드가 GIL을 다시 획득하지 못하도록 하는 메커니즘을 추가한 Python 3.2에서 수정되었습니다.


## Python의 GIL을 다루는 방법
GIL 때문에 성능 이슈 일으키는 경우 다음과 같은 방법을 시도할 수 있습니다.

multi processing : 가장 널리 사용되는 방법은 스레드 대신 여러 프로세스를 사용하는 multi processing 접근 방식을 사용하는 것입니다. 
각 Python 프로세스에는 자체 Python 인터프리터와 메모리 공간이 있으므로 GIL은 문제가되지 않습니다. Python에는 다음 `multiprocessing`과 같은 프로세스를 쉽게 만들 수 있는 모듈이 있습니다.

```python
from multiprocessing import Pool
import time

COUNT = 50000000


def countdown(n):
    while n > 0:
        n -= 1


pool = Pool(processes=2)

start = time.time()

r1 = pool.apply_async(countdown, [COUNT // 2, ])
r2 = pool.apply_async(countdown, [COUNT // 2, ])

pool.close()
pool.join()

end = time.time()

print('Time taken in seconds -', end - start)
# Time taken in seconds - 1.3406448364257812
```

멀티 스레드 버전에 비해 비약적으로 성능이 상당히 향상되지는 않았습니다.
Python GIL은 어려운 주제로 간주됩니다. 그러나 C 확장을 작성하거나 프로그램에서 CPU 바인딩 된 멀티 스레딩을 사용하는 경우에만 영향을 받습니다.

>  아마 프로세스 `context switching` 등의 오버헤드가 있기 때문일 것 같습니다. 또한 메모리 동기화 등에 좀 더 신경써야해서 더욱 어렵습니다.
>  다른 방법으로는 전에 포스팅했던 `asyncio`, `celery`등도 하나의 해결방법 입니다.


### 참조 링크
- [GlobalInterpreterLock - Python Wiki](https://wiki.python.org/moin/GlobalInterpreterLock)
- [파이썬 New GIL을 이해하기 : 네이버 블로그](https://m.blog.naver.com/PostView.nhn?blogId=parkjy76&logNo=30167429369&proxyReferer=https%3A%2F%2Fwww.google.com%2F)
- [왜 Python에는 GIL이 있는가 :: 개발새발로그](https://dgkim5360.tistory.com/entry/understanding-the-global-interpreter-lock-of-cpython)