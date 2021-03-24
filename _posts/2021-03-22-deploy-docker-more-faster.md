---
title: 자주 변경되는 도커 이미지 빠르게 배포하기 (feat. 이미지 사이즈)
author: daru
date: 2021-03-23 13:14:13
categories: [devops,docker]
tags: [docker]
---

프로젝트에서 사용하는 워커가 도커로 관리되고 있지만 외부 라이브러리를 사용하기 위해서 `PHP`를 혼용해서 사용하고 있습니다.

문제는 해당 라이브러리의 버전이 주기적으로 자주 업데이트되는데 빌드 후에 모든 워커에 배포까지 약 3시간 정도 걸립니다.

(인터넷 속도가 느린 환경이라 어쩔수 없는 경우입니다. 고치는것도 불가능합니다.ㅠㅠ)

위의 이유로 최대한 이미지 사이즈를 작게 만들어 빠르게 배포될 수 있는 이미지를 만드는 것이 중요해졌습니다.

## Docker multi-stage 이용하기
`Docker 17.05`에서 도입된 `multi-staging`는 전에 있었던 [`Builder pattern`](https://blog.alexellis.io/mutli-stage-docker-builds/)의 대체로 이미지의 용량을 줄이기 위해 출시되었습니다.

### Builder Pattern
`Builder Pattern`에 대해서 간략하게 요약하자면 다음과 같습니다.(`Go`,`Java` 등의 상황에서)
- 애플리케이션을 빌드용 이미지를 하나 만듭니다.
- 빌드용 이미지를 이용해 컨테이너를 하나 실행합니다.
- 해당 이미지에서 빌드한 어플리케이션을 추출합니다.(`docker copy`)
- `alpine` 이미지 등 배포용 이미지에 추출한 어플리케이션을 복사합니다.
- 배포용 이미지를 이용해 컨테이너를 생성합니다.

해당 방식의 문제점은 여러 도커 파일을 관리 해야하며, `docker copy` 등을 위해서 명령어도 관리를 해야한다는 점이 있습니다.

위의 범용적인 `Builder Pattern`을 도커의 `multi-stage`를 사용함으로써 같은 효과를 낼 수 있게 되었습니다.

```Dockerfile
FROM golang:1.7.3
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html  
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/alexellis/href-counter/app .
CMD ["./app"]  
```

### 파이썬에서는 어떻게 사용할까
`pip install -user`를 통해 특정 유저에게만 패키지를 설치하고 해당 패키지를 옮기는 방법

```Dockerfile
FROM python:3.7-slim AS compile-image
RUN apt-get update
RUN apt-get install -y --no-install-recommends build-essential gcc

COPY requirements.txt .
RUN pip install --user -r requirements.txt

COPY setup.py .
COPY myapp/ .
RUN pip install --user .

FROM python:3.7-slim AS build-image
COPY --from=compile-image /root/.local /root/.local

# Make sure scripts in .local are usable:
ENV PATH=/root/.local/bin:$PATH
CMD ['myapp']
```

`virtualenv`를 이용하는 방법

```Dockerfile
FROM python:3.7-slim AS compile-image
RUN apt-get update
RUN apt-get install -y --no-install-recommends build-essential gcc

RUN python -m venv /opt/venv
# Make sure we use the virtualenv:
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY setup.py .
COPY myapp/ .
RUN pip install .

FROM python:3.7-slim AS build-image
COPY --from=compile-image /opt/venv /opt/venv

# Make sure we use the virtualenv:
ENV PATH="/opt/venv/bin:$PATH"
CMD ['myapp']
```

등 다양한 방법이 있습니다.

### multi-stage를 사용해볼까?
도커의 `multi-stage`를 사용하면 쓸데 없는 의존성, 파일 등을 제외하고 빌드할 수 있어서 이미지 용량 감소에 있어 좋아보였습니다.

또한 패키지의 의존성만 변경되거나, 소스코드만 변경되면 다른쪽 이미지를 다시 빌드할 필요 없이 바뀐 부분만 다시 빌드하면 되기때문에 빌드 속도 또한 올릴수 있습니다.

다만 프로젝트를 `pip install -e`로 설치해서 사용하며, 내부에서 `PHP` 스크립트를 호출해서 사용하는데 파이썬 `site-package`를 뜯어보니 복사해서 설치하는 것이 아니라 링킹 수준의 설치였습니다.

> [Deploy your project in “development mode”, such that it’s available on sys.path, yet can still be edited directly from its source checkout.](https://setuptools.readthedocs.io/en/latest/setuptools.html?highlight=develop%20mode#development-mode)
>
> 레거시 설치 방법이라 지금 당장 건드리기엔 너무 위험이 커서 건드리지 않을것 같습니다.
> 물론 `package_data`를 지정해서 넘길수 있지만 프로젝트 특성상 다른 컴포넌트들도 있기에 불가능할 것 같습니다.
>
> 결국에는 경로 문제가 발생할것 같아 `multi-stage`를 사용하지 않기로 했습니다.
> PHP쪽에는 가능할 것 같아 적용하고 있습니다.

### Layer를 바꾸기
`multi-stage`를 사용할수 없게되면서 다른 방법을 찾아봐야 했습니다.

처음부터 다시 시작하자라는 생각에 `Dockerfile`을 뜯어보니 굉장히 자주 바뀌는 부분이 꽤나 윗부분에 있었습니다.

#### 도커 이미지에서 순서?
도커 이미지는 유니온 마운트를 사용하여 순차적으로(레이어별로) 마운트하며 컨테이너에서 빌드하며 이미지를 작성해 나갑니다.

![docker-layers.jpg](https://www.baeldung.com/wp-content/uploads/2020/11/docker-layers.jpg)

효과적인 배포를 위해 레이어 별로 바뀌지 않은 이미지 ID는 캐쉬하여 사용하는 특성이 있습니다.
결국 아래 부분의 이미지에서 변경 사항이 있으면 윗 부분의 변경 사항까지 모두 다시 빌드해야 합니다.

> 유니온 마운트란
> 여러 파일 시스템을 하나의 파일 시스템처럼 마운트하는 기법을 뜻합니다.
>
> `OverlayFS`, `AUFS` 등이 있으며 `OverlayFS2`가 가장 대중적이며 커널4.0에 기본적으로 포함되어 있습니다.
> 중요한점은 밑의 디렉토리는 항상 불변의 상태를 유지하며, 읽기 전용으로 사용됩니다.
>
> 만약 밑의 디렉터리들에 파일 변경사항이 발생할 경우, 가장 윗쪽의 디렉토리에 그 변경 사항을 기록합니다.
> 더 자세한 정보는 [alice_k106님의 블로그 : 네이버 블로그](https://blog.naver.com/alice_k106/221530340759)를 참고해주세요.

#### 해결 방안
위에서 설명한 내용에 따라 자주 바뀌는 레이어는 도커 이미지 레이어 상단에 배치하는게 이득이라고 알 수 있었습니다.

프로젝트에서는 `Python` 코드보다는 `PHP` 의존성을 바꾸는 경우가 훨씬 많아 `PHP` 의존성 설치쪽을 `Dockerfile` 아래에 두었습니다.

### Dockerfile을 바꾸자
기존의 `Dockerfile`은 대략 이렇게 생겼습니다.


```Dockerfile
FROM python:3.7-slim

# apt install php ~ 기타 의존성

ADD php/src /app
RUN composer install

ADD python/src /app
RUN pip install


CMD ["python", "/app.py"]
```

`multi-stage` + 도커 레이서 순서 변경은 대략적으로 아래와 같이 바꾸었습니다.

```Dockerfile
FROM composer as builder

COPY composer.json /app
RUN composer install  \
    --ignore-platform-reqs \
    --no-ansi \
    --no-autoloader \
    --no-interaction \
    --no-scripts

FROM python:3.7-slim

# apt install php ~ 기타 의존성

ADD python/src /app

RUN pip install

ADD php/src /app

COPY --from=builder /app /app/vendor

CMD ["python", "/app.py"]
```

### 외부 CI에서는?
외부 CI에서는 `Dockerfile`을 빌드할 때 캐시가 날아가서 다시 빌드하는 경우가 많습니다.

도커에서는 빌드할 때 강제로 `--cache-from` 플래그를 사용하여 캐시를 사용하도록 할 수 있습니다.

이를 이용하여 CI 서버에서도 캐시를 사용하여 `Dockerfile`을 빌드할 수 있습니다.

`gitlab`의 경우 [해당 문서](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#how-docker-caching-works)를 읽어보시면 먼저 `docker pull`을 한 후 `docker build --cache-from ~`을 사용하도록 권장하고 있습니다. 

해당 기능을 사용하여 약 빌드 시간을 90% 단축했습니다.

```
#23 [builder 2/3] COPY ~
#23 sha256:e92607a9f0cc713e0704959f3c0e13b3bdca48ce8726cd883e1d6240dc3a130d
#23 CACHED
#24 [builder 3/3] RUN ~
#24 sha256:7156c7ce6ea40bdea42f4d10e1ae377c8e473aa7f9ed0dc7655e6684d52f47c4
#24 CACHED
#14 [stage-1  9/17] ADD ~
#14 sha256:3c726008c41a90f1d3a5b4b6704955911d7cf56eb17b1cd4edd8e3f3be5ead67
#14 CACHED
#15 [stage-1 10/17] ADD ~
#15 sha256:6e13475d1e9c3fd8a23c0508bee1feea11e0890a4d0aa58ae60012ff4657babf
#15 CACHED
#17 [stage-1 12/17] RUN ~
#17 sha256:e970a0155a8ce4b97826d0117e6d503cf20b12f9969bf0a9392f32edb89c47e1
#17 CACHED
...
#28 DONE 32.1s
```

**기존 빌드 시간**
![before.png](/assets/img/posts/deploy-docker-more-faster/before.png)

**캐시 + `multi-stage` + 레이어 빌드 시간**
![after.png](/assets/img/posts/deploy-docker-more-faster/after.png)


### 결론
도커의 이미지 구성이나 `multi-stage`를 어렴풋이 알고는 있었지만, 직접 사용해서 프로덕션에 적용해 보는것은 처음이였습니다.

덕분에, 도커 이미지 레이어, 도커 캐시 등의 원리를 이해할 수 있는 시간이였습니다.

배포 시간은 측정이 불가능해서 아쉽게도 측정하지 못했지만, 캐시를 충분히 사용하기 때문에 빠르게 배포되는 것을 눈으로 확인했습니다.


#### 출처
- [Use multi-stage builds \| Docker Documentation](https://docs.docker.com/develop/develop-images/multistage-build/)
- [Multi-stage builds #2: Python specifics—virtualenv, –user, and other methods](https://pythonspeed.com/articles/multi-stage-docker-python/)
- [alice_k106님의 블로그 : 네이버 블로그](https://blog.naver.com/alice_k106/221530340759)
- [Use Docker to build Docker images \| GitLab](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#how-docker-caching-works)

