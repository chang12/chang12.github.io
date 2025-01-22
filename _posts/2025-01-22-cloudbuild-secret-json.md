---
layout: post
title: "json 으로 된 secret 의 특정 key 의 값을 cloud build 에서 env 로 주입"
tags: []
---

1. gcp 의 secret manager 에 secret 이 있고, json 이다.
2. secret 의 특정 key 의 값을 cloud build 에서 실행되는 program 에서 사용하고 싶다.
3. cloud build 에서 이를 native 하게 지원하진 않는다.
4. chatgpt 는 `jq` 로 parsing 하는 방법을 제안했다.
5. cloud build 에서 실행하려는 program 의 base docker image 가 python 이라, `jq` 를 추가로 설치해야 한다.
6. python image 이니 그냥 python code 로 parsing 하면 되겠다. 

```yaml
steps:
  - name: python:3.11
    ...
    entrypoint: bash
    args:
      - "-c"
      - |
        export KEY1=$(python -c "import os, json; print(json.loads(os.environ['SECRET'])['KEY1'])")
        export KEY2=$(python -c "import os, json; print(json.loads(os.environ['SECRET'])['KEY2'])")
        ...
    secretEnv:
      - SECRET
availableSecrets:
  secretManager:
    - versionName: projects/.../secrets/.../versions/latest
      env: 'SECRET'
...

```

7. program 에서 `KEY1`, `KEY2` 로 env variable 을 접근하여 사용하면 된다.
