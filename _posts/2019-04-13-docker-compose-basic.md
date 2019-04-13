---
layout: post
title: Docker Compose 는 뭘까?
tags:
  - docker
---

Docker 를 활용해서 개발 머신의 환경에 관계 없이 실행 환경을 선택할 수 있었고 ([OSX 에서 Python LightGBM 패키지 설치때 고생한 경험](https://chang12.github.io/osx-lightgbm-2.0.11/)), 비개발자 팀원이 개발 환경을 구성하지 않고도 프로그램을 실행할 수 있게 (`docker load` -> `docker run`) 도울 수 있었습니다. Docker Compose 라는 것도 있다는데 뭔지 궁금해집니다.

## Docker Compose?

[Overview of Docker Compose](https://docs.docker.com/compose/overview/) 문서의 문장을 보면, YAML 파일로 설정을 기술한 뒤 단일 명령어로 여러 container 를 실행할 수 있게 해주는 것 같습니다.

> Compose is a tool for defining and running **multi-container** Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with **a single command**, you create and start all the services from your **configuration.**

관련하여 [스택 오버플로우의 What's the difference between docker compose and kubernetes? 글의 답변](https://stackoverflow.com/a/47537046) 도 유익했습니다. Docker Compose (single-host & multi-container), Docker Swarm (multi-host & multi-container 를 orchestration), Kubernetes (Google 이 만든 Docker Swarm 의 대체재) 로 각각의 층위를 간단히 이해할 수 있었습니다.

## 예제 : Get started with Docker Compose

[Get started with Docker Compose](https://docs.docker.com/compose/gettingstarted/) 를 따라가봤는데 담백하고 훌륭한 예제였습니다. Redis container 와 이를 접근하는 Flask app container 를 단일 `docker-compose` 명령어로 실행할 수 있었습니다. 따라가는 도중에 들었던 몇가지 의문들을 정리해봤습니다.

## `redis` service 는 어떤 명령어를 실행할까?

`docker inspect redis:alpine` 명령어의 출력을 확인해보니, Config.Cmd 가 `redis-server` 입니다. `docker inspect docker-compose-basic_web` 에는 Dockerfile 에 적은대로 `["python", "app.py"]` 로 나옵니다.

## Flask app 에서 어떻게 redis container 에 접속 했을까?

[Networking in Compose](https://docs.docker.com/compose/networking/) 문서에 잘 정리되어있습니다. `docker-compose up` 으로 service 들을 띄울 때, `Creating network "docker-compose-example_default" with the default driver` 라고 콘솔에 출력됬었습니다. 이렇게 만들어진 network 에, service 이름을 host 로 container 들이 접속하므로, Flask app 코드에서 `cache = redis.Redis(host='redis', port=6379)` 로 redis 에 접속할 수 있게 됩니다.

## Ctrl+C 와 `docker-compose down` 의 차이는?

`docker-compose up` 중 Ctrl+C 하는건 `docker-compose stop` 와 동등하게 service 를 stop 하게 됩니다. 이에 비해 `docker-compose down` 의 경우 container 까지 제거합니다.

## 레퍼런스

* [Overview of Docker Compose](https://docs.docker.com/compose/overview/)
* [스택 오버플로우의 What's the difference between docker compose and kubernetes? 글의 답변](https://stackoverflow.com/a/47537046)
* [Get started with Docker Compose](https://docs.docker.com/compose/gettingstarted/)
* [Networking in Compose](https://docs.docker.com/compose/networking/)
