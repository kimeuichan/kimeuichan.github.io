---
title: Python에서 MultiProcessing, MultiThread 다루기 with 삽질
author: daru
date: 2021-01-12 10:00:00
categories: [programming,python]
tags: [python,parallel]
---

회사에서 multi-thread 기반인 코드가 있었습니다.

`multiprocessing` 모듈로 바꾸는 과정 및 경험 그리고 결과를 공유하는 포스팅입니다.

## 왜 multiprocessing을 사용하나요?
Python에서 multi-thread를 지원을 하지만, GIL라는 거대한 제약 덕분에 생각하는 것 만큼의 성능을 내지 못하는 경우가 있습니다.

GIL에 대해서는 이미 포스팅을 해놓았기 때문에 자세한 설명은 [여기](https://kimeuichan.github.io/posts/python-gil/)를 참고해주세요.

multi-thread 코드가 정상적으로 작동하려면 GIL을 각 thread가 release 하는지 확인해야 하는데 애매한 경우가 많습니다.

이러한 경우, `multiprocessing` 모듈을 사용하면 GIL 제약 없이 성능을 끌어낼 수 있습니다.

### 멀티 쓰레드
기본적으로 사용하는 multi-thread 기반 코드는 다음과 같았습니다.

```python
import subprocess
import threading
import time


def fibo(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fibo(n - 1) + fibo(n - 2)


def task_func(index):
    print(f"start func {index}")
    return fibo(32)


if __name__ == '__main__':
    tasks = []

    for i in range(5):
        tasks.append(
            threading.Thread(target=task_func, args=(i,))
        )

    for task in tasks:
        task.start()

    for task in tasks:
        task.join()
```

실제 코드를 적을 순 없어 cpu-bound 태스크임을 고려해 피보나치 수열을 계산하는 함수를 사용하였습니다.

*실행결과*
```
start
start func 0
start func 1
start func 2
start func 3
start func 4
4.059529066085815
```


### 멀티 프로세스 코드
multi-thread 코드를 다음과 같이 `multiprocessing`을 사용하는 코드로 바꾸었습니다.

```python
import multiprocessing
import threading
import time


def fibo(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fibo(n - 1) + fibo(n - 2)


def task_func(index):
    print(f"start func {index}")
    return fibo(32)


if __name__ == '__main__':
    tasks = []

    for i in range(5):
        tasks.append(
            multiprocessing.Process(target=task_func, args=(str(i),))
        )

    for task in tasks:
        task.start()

    for task in tasks:
        task.join()
```

*실행결과*
```
start
start func 0
start func 1
start func 2
start func 3
start func 4
0.8843121528625488
```

## multi-thread vs multi-process
실행결과를 놓고 보앗을 때 약 4배 이상의 성능차이가 났습니다. 그렇다면 무조건 `multiprocessing`을 사용하는게 좋을까요?

프로세스와 쓰레드는 많은 차이점들이 있습니다만, 이 포스팅에서는 다루지 않겠습니다.

cpu-bound 태스크를 위해 예시로 만든 코드를 제거하고, 실제 태스크에서 쓰이는 태스크 코드를 비슷하게 작성하였습니다.


```python
import subprocess
import threading
import time


def task_func(index):
    print(f"start func {index}")
    process = subprocess.Popen(
        ["sleep", "2"],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
    )
    process.wait()


if __name__ == '__main__':
    start = time.time()
    print("start")
    tasks = []

    for i in range(5):
        tasks.append(
            threading.Thread(target=task_func, args=(i,))
        )

    for task in tasks:
        task.start()

    for task in tasks:
        task.join()

    end = time.time()
    print(end - start)
```

위와 같이 작성된 코드를 `multiprocessing` 기반으로 재작성 하였습니다.

```python
import multiprocessing
import subprocess
import time


def task_func(index):
    print(f"start func {index}")
    process = subprocess.Popen(
        ["sleep", "2"],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
    )
    process.wait()


if __name__ == '__main__':
    start = time.time()
    print("start")
    tasks = []

    for i in range(5):
        tasks.append(
            multiprocessing.Process(target=task_func, args=(str(i),))
        )

    for task in tasks:
        task.start()

    for task in tasks:
        task.join()

    end = time.time()
    print(end - start)
```

문득 성능차이가 궁금해져서 두 코드 모두 돌려보았습니다.

*multi-thread 실행결과*
```
start
start func 0
start func 1
start func 2
start func 3
start func 4
2.0173916816711426
```

*multi-process 실행결과*
```
start
start func 0
start func 1
start func 2
start func 3
start func 4
2.0356900691986084
```

제 예측과는 다르게 둘의 결과가 크게 차이가 없었습니다.


## 왜 큰 차이가 없을까?

`subprocess.py` 파일의 `process.wait` 함수를 분석을 해보았습니다.


```python
def _try_wait(self, wait_flags):
    """All callers to this function MUST hold self._waitpid_lock."""
    try:
        (pid, sts) = os.waitpid(self.pid, wait_flags)
    except ChildProcessError:
        # This happens if SIGCLD is set to be ignored or waiting
        # for child processes has otherwise been disabled for our
        # process.  This child is dead, we can't get the status.
        pid = self.pid
        sts = 0
    return (pid, sts)
```

`os.waitpid`는 OS별로 windows, posix 계열로 나뉘게 됩니다. 

내부를 계속 타고 들어가게되면 다음과 함수를 실행하는 구문이 있습니다.


> 제 분석이 틀렸을 수도 있습니다. 혹시 틀렸다 싶으면 github를 통해 이슈 등록해주시면 빠르게 수정하겠습니다.

여기서 `os.waitpid` 함수가 굉장히 중요한데 이 함수의 [내부 구현](https://github.com/python/cpython/blob/master/Modules/posixmodule.c#L8331)을 보면 다음과 같은 C 코드가 있습니다

```C
Py_BEGIN_ALLOW_THREADS
res = waitpid(pid, &status, options);
Py_END_ALLOW_THREADS
```

C 코드의 `Py_BEGIN_ALLOW_THREADS` 매크로는 파이썬에서 GIL 해제를 할 수 있는 매크로 입니다. 

반드시 `Py_END_ALLOW_THREADS` 매크로를 사용해야 합니다. 자세한 정보는 [공식 홈페이지](https://docs.python.org/3/c-api/init.html#releasing-the-gil-from-extension-code)에 있습니다.

`subprocess`를 통해 프로세스를 만들고 `wait`하는 과정은 GIL을 해제하기 때문에 크게 결과가 달라지지 않았습니다.

결국 GIL을 해제할 수 있는 경우에는 multi-processing과 multi-thread는 파이썬에서 성능차이가 나지 않는다는 점을 알 수 있었습니다.


## 느낀점

GIL을 공부하면서 알고 있엇던 내용을 실제 코드에서 적용할 수 있어 재밌었습니다.

Python에서 multi-thread 기반 코드를 작성하려면 해당 코드가 GIL을 해제하는지 확인하거나, 맘 편하게 `multiprocessing` 모듈을 사용하는게 좀 더 편할 것 같다고 느겼습니다.

또한, 실제 코드에 적용하기 전에 조금 더 테스트를 해보았으면 삽질 없이 쉬운 방법을 사용했을 것 같습니다.




## 참고
- [multithreading - Does using the subprocess module release the python GIL? - Stack Overflow](https://stackoverflow.com/questions/23369064/does-using-the-subprocess-module-release-the-python-gil)
- [Releasing the gil](https://thomasnyberg.com/releasing_the_gil.html#pure-python)
- [Initialization, Finalization, and Threads — Python 3.9.1 documentation](https://docs.python.org/3/c-api/init.html#releasing-the-gil-from-extension-code)
 



