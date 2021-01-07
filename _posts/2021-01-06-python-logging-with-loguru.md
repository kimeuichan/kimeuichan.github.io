---
title: loguru를 사용하여 python 로깅 쉽게하기
author: daru
date: 2021-01-06 15:45:00
categories: [programming,python]
tags: [python]
---

종종 파이썬으로 코딩을 하게되면 무의식적으로 `print()` 문을 사용하는 경우가 있습니다.

개인적인 사이드 프로젝트에서는 크게 문제되지 않지만 회사, 팀 프로젝트 단위에서는 필요한 데이터를 `print()`문 하나에 의존해 남기는 것은 매우 위험하다고 생각합니다.

`print()`는 단지 `stdout`을 통해 모니터에 출력되기만 하는것이며 다음에 필요할 때 찾기가 매우 까다롭기 때문입니다.

그래서 파이썬에서는 공식적으로 [logging 라이브러리](https://docs.python.org/ko/3/library/logging.html)가 존재합니다.

하지만 `level`, `filter`, `handler`라는 개념 등이 존재하며 잘 사용하기 위해서는 까다로운 설정이 필요하다고 생각합니다.


## loguru란
`loguru`란 python 기반의 사용할 수 있는 [로깅 오픈 소스](https://github.com/Delgan/loguru)입니다.

대표적인 특징은 아래와 같습니다.

### 특징
- 설정 없이 바로 사용 가능
- `handler`, `formatter`, `filter`를 하나의 함수에서 정의할 수 있음
- 회전 / 보존 / 압축을 사용할 수 있는 간편한 파일 로깅
- 중괄호 스타일을 사용한 최신 문자열(f-string과 비슷한) 서식
- 스레드 또는 메인 내에서 발생하는 예외 처리
- 색상으로 예쁜 로깅
- 비동기식, 스레드 안전, 다중 프로세스에 안전한 로깅

등과 여러 특징이 있습니다. 자세한 기능은 [공식 홈페이지](https://github.com/Delgan/loguru#features)에 좀 더 자세하게 설명되어 있습니다.

### 사용해보기

**기본 로깅**
```python
from loguru import logger
logger.debug("hello world")
```

`loguru`는 [기본 설정](https://github.com/Delgan/loguru#ready-to-use-out-of-the-box-without-boilerplate)이 되어 있어 다음과 같이 문법으로도 간단하게 로깅할 수 있습니다.

또한 다른 방법으로 로깅을 하고 싶다면 [`add()`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.add) 함수를 사용하여 다른 핸들러를 추가할 수 있습니다.

**파일에 특정 주기별 주기별 로깅하기**

```python
logger.add("file_1.log", rotation="12:00")  # 매일 12시에 새로운 로그 파일 생성
logger.add("file_X.log", retention="10 days")  # 10일 후에 제거
```

`loguru`는 `rotation`과 `retention` 기능을 이용해 특정 주기별로 로깅을 남길 수 있습니다.

```python
logger.add("test_{time}", rotation="12:00", retention="10 days", compression="zip")
```

다음과 같은 코드는 로깅할 때 파일 이름은 `test_2021-01-05_16-51-05_359452` 의 형식이 됩니다.

또한, 매일 12시에 새로운 파일이 생기며 해당 파일은 10일간 유지되며 로깅이 끝난 파일의 경우 `zip`으로 압축됩니다. (`"gz"`, `"bz2"`, `"xz"`, `"lzma"`, `"tar"`, `"tar.gz"`, `"tar.bz2"`, `"tar.xz"`, `"zip"` 등의 형식도 지원합니다.)


자세한 파라미터 정보는 [여기](https://loguru.readthedocs.io/en/stable/api/logger.html#file)를 참고하시면 좋습니다.

**필터링**

로그를 남기다보면 어떤 메세지는 특정 핸들러에 남기고 싶지 않은 경우가 있습니다.

그런 경우 `loguru`의 `filter`와 `bind`를 사용하면 됩니다.

```python
logger.add("special.log", filter=lambda record: "special" in record["extra"])
logger.debug("This message is not logged to the file")
logger.bind(special=True).info("This message, though, is logged to the file!")
```
`filter`에 `callable`이 들어간다면 `record`를 받아 사용할 수 있습니다.

 위의 코드와 같이 사용할 수 있으며, `record`는 [다양한 속성](https://loguru.readthedocs.io/en/stable/api/logger.html#record)이 있습니다.

`bind`는 해당 메시지의 `extra`에 특정 값을 종속적으로 묶을수 있으며 바인딩된 `logger`를 반환합니다. 따라서 다음과 같이 사용할 수도 있습니다.

```python
some_logger = logger.bind(special=True)
```

다음과 같이 사용하면 `filter` 조건을 만족해서 `special.log`에 적재되게 됩니다.

**새로운 레벨 만들기**

`loguru`는 독특하게 레벨이라는 개념을 만들 수 있습니다. 새로운 레벨을 만들고 로깅을 하는 방법은 다음과 같습니다.

기존 `loguru`에서 사용하는 레벌 개념은 [다음](https://loguru.readthedocs.io/en/stable/api/logger.html#levels)과 같으며, 표준 로거와 같은 동일한 수준으로 로깅과 연결됩니다.

```python
new_level = logger.level("SNAKY", no=38, color="<yellow>", icon="🐍")
logger.log("SNAKY", "Here we go!")
```

**메시지 포맷 정하기**

`loguru`또한 기본 로깅 라이브러리처럼 메시지 포맷을 정할 수 있습니다.

```python
logger.add("file.log", format="{time:YYYY-MM-DD at HH:mm:ss} | {level} | {message}")
```

또한, 색상 마크 업 등을 지원합니다.


`loguru`는 그 외에도 표준 로깅 라이브러리에 전파할 수 있는 기능 등 다양한 기능을 지원합니다.

### 프로덕션에 적용하기
해당 라이브러리를 `production` 프로젝트에 적용해보았습니다.

처음 프로젝트에 도입할 때 다음과 같은 형식으로 적용하였습니다.

```python
if ENV == "production":
  logger.add(...)
elif ENV == "staging":
  logger.add(...)
```

프로젝트는 [`Flask`와 비슷한 설정](https://flask.palletsprojects.com/en/1.1.x/config/#development-production)을 사용햇습니다.

아쉬운점은 프로젝트 도입 초기에 [`configure`](https://loguru.readthedocs.io/en/stable/api/logger.html#loguru._logger.Logger.configure)함수를 발견하지 못하고 바쁘게 프로젝트를 진행하여 위와 같이 각각의 환경에 설정을 하게 됐습니다.

코드 리팩토링 할 때 설정 파일을 다음과 같이 리팩토링 하려 합니다.

```python
class Config(object):
    DEBUG = False
    TESTING = False
    DATABASE_URI = 'sqlite:///:memory:'
    LOGURU_SETTINGS = {}


class ProductionConfig(Config):
    DATABASE_URI = 'mysql://user@localhost/foo'
    LOGURU_SETTINGS = {
        "handler": [
            dict(sink=sys.stderr, format="[{time}] {message}"),
            dict(sink="file.log", enqueue=True, serialize=True),
        ],
        "levels": []
        ...
    }


class StagingConfig(Config):
    DEBUG = True
    LOGURU_SETTINGS = {
        "handler": [
            dict(sink=sys.stdout, format="[{time}] {message}"),
        ],
        "levels": []
        ...
    }


if ENV == "production":
  config = ProductionConfig()
elif ENV == "staging":
  config = StagingConfig()


logger.configure(**config.LOGURU_SETTINGS)
```


### 번외
`production`에 적용을 하니 라이브러리 단 설정 때문에 `stderr`에 무조건 로깅이 되는 상황이 벌어졌습니다.

따라서 다음과 같은 설정을 적용했습니다.

```python
# ...

if ENV == "production":
  config = ProductionConfig()
elif ENV == "staging":
  config = StagingConfig()


logger.remove()  # 기존 모든 로깅 핸들러를 제거
logger.configure(**config.LOGURU_SETTINGS)
```

`loguru`를 이용해 간단하게 로깅하는 법을 포스팅해봤습니다. 

`loguru`는 한글 문서가 부족하다는 점이 있지만 사용자 편의 기능을 많이 지원하며 문서화 또한 간단하게 잘 되어 있습니다.
또한, 간단하게 logging 시스템을 쉽게 적용할 수 있다는 장점이 있어 한번쯤 사용해보시면 좋을 것 같습니다.



#### 참고
- [GitHub - Delgan/loguru: Python logging made (stupidly) simple](https://github.com/Delgan/loguru)
- [Table of contents — loguru documentation](https://loguru.readthedocs.io/)
- [[Python] log management loguru library - custom log rotation and compression - Programmer Sought](https://programmersought.com/article/59471895100/)
(https://stackoverflow.com/a/55766474/5944655)
