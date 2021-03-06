---
layout: post
title: OSX 에서 Python LightGBM 패키지 설치때 고생한 경험
tags: [Docker]
---

오랜만에 Python 으로 작성된 프로젝트의 개발 환경을 설정할 일이 있었습니다. virtualenv 를 만들고, `pip -r requirements.txt` 로 의존하는 패키지들을 설치하고 ([lightgbm](https://pypi.org/project/lightgbm/) 포함), 예시 명령어를 실행하는 데 에러가 발생했습니다. `lightgbm` 을 초기화 하는 부분입니다.

```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/Users/fakenerd/.envs/env-with-lightgbm/lib/python3.6/site-packages/lightgbm/__init__.py", line 8, in <module>
    from .basic import Booster, Dataset
  File "/Users/fakenerd/.envs/env-with-lightgbm/lib/python3.6/site-packages/lightgbm/basic.py", line 32, in <module>
    _LIB = _load_lib()
  File "/Users/fakenerd/.envs/env-with-lightgbm/lib/python3.6/site-packages/lightgbm/basic.py", line 27, in _load_lib
    lib = ctypes.cdll.LoadLibrary(lib_path[0])
  File "/Library/Frameworks/Python.framework/Versions/3.6/lib/python3.6/ctypes/__init__.py", line 426, in LoadLibrary
    return self._dlltype(name)
  File "/Library/Frameworks/Python.framework/Versions/3.6/lib/python3.6/ctypes/__init__.py", line 348, in __init__
    self._handle = _dlopen(self._name, mode)
OSError: dlopen(/Users/fakenerd/.envs/env-with-lightgbm/lib/python3.6/site-packages/lightgbm/lib_lightgbm.so, 6): Library not loaded: /usr/local/opt/gcc/lib/gcc/7/libgomp.1.dylib
  Referenced from: /Users/fakenerd/.envs/env-with-lightgbm/lib/python3.6/site-packages/lightgbm/lib_lightgbm.so
  Reason: image not found
```

DLL, gcc, .so 등의 키워드를 보고나니 정신이 아득해집니다. 그래도 다시 추스리고 문제 해결에 돌입합니다. 환경은 아래와 같습니다.

* OSX 10.13.5 (High Sierra)
* Python 3.6.3
* lightgbm==2.0.11 (다른 패키지에서 `lightgbm<2.1,>=2.0.11` 를 요구)

## OSError, dlopen, Library not loaded

해당 경로를 찾아보니 `/usr/local/opt` 디렉토리 아래 `gcc` 디렉토리가 존재하지 않습니다. 의아해지면서, `OSError` 를 발생시킨 `dlopen` 함수가 궁금해집니다. [Python GitHub](https://github.com/python/cpython/blob/v3.6.3/Lib/ctypes/__init__.py#L133) 의 코드를 보니 `_ctypes` 모듈의 `dlopen` 함수입니다. python shell 을 실행해서 `_ctypes` 모듈의 파일 경로를 찾아봅니다.

```
$ python
Python 3.6.3 (v3.6.3:2c5fed86e0, Oct  3 2017, 00:32:08)
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import _ctypes
>>> _ctypes.__file__
'/Users/fakenerd/.envs/env-with-lightgbm/lib/python3.6/lib-dynload/_ctypes.cpython-36m-darwin.so'
>>>
```

또 다시 .so 파일을 만났습니다. 컴파일된 C 코드를 python 에서 사용하는 것 같습니다. 결국 .so 파일의 코드가 다른 .so 파일을 참조하는데, 참조되는 .so 파일이 있어야 할 경로에 존재하지 않아 에러가 난 것입니다. 왜 그 경로를 원하는지 알고 싶었으나... C 코드를 읽어야 하므로 포기했습니다.

## Microsoft/LightGBM #1369

다행히 동료분이 이미 맞닿뜨렸던 에러였고, [해결 방법이 적힌 GitHub issue](https://github.com/Microsoft/LightGBM/issues/1369) 를 공유해주셨습니다. PyPi 에 업로드 된 binary 를 빌드하는 환경과, OSX 에서 이를 사용하려는 개발자들의 환경 차이로 인한 문제였습니다. 자세히는 gcc 버전의 차이라고 합니다. issue 리포팅한 개발자는 lightgbm 소스 코드로 직접 빌드해서 해결했다고 합니다. 고민이 됩니다.

* `/usr/local/opt/gcc/lib/gcc/7/libgomp.1.dylib` 이 없는게 문제니까 넣어줘서 배포판의 환경을 맞춰준다.
* issue 에 적힌대로 소스 코드로 직접 빌드한다.

비단 `/usr/local/opt/gcc/lib/gcc/7/libgomp.1.dylib` 만의 문제가 아닐 것으로 보여, 전자를 택할 경우 고생길이 예상됩니다. 그래서 후자를 따르기로 했습니다.

## LightGBM build, install

gcc 버전이 문제라고 하니 현재 개발 머신의 gcc 버전을 확인합니다.

```
$ gcc --version
Configured with: --prefix=/Applications/Xcode.app/Contents/Developer/usr --with-gxx-include-dir=/usr/include/c++/4.2.1
Apple LLVM version 9.1.0 (clang-902.0.39.2)
Target: x86_64-apple-darwin17.6.0
Thread model: posix
InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
```

버전스러워 보이는 값이 `4.2.1` 와 `9.1.0` 이 있는데... gcc7 vs gcc8 을 논하던 상황에서 이상합니다. `4.2.1` 은 너무 낮고, [GCC Releases](https://www.gnu.org/software/gcc/releases.html) 을 보면 최신 버전이 8.3 인데 `9.1.0` 은 너무 높습니다. 혼란스러우니 homebrew 로 gcc 를 설치합니다. 최신 버전인 8 으로 설치합니다.

```
brew install gcc@8
```

[Microsoft/LightGBM GitHub repo 의 Releases](https://github.com/Microsoft/LightGBM/releases) 에서 [2.0.11 버전의 tar.gz 파일을](https://github.com/Microsoft/LightGBM/archive/v2.0.11.tar.gz) 다운로드 해서 압축을 푼 뒤 원하는 경로로 복사합니다. 그리고 [해결 방법이 적힌 GitHub issue](https://github.com/Microsoft/LightGBM/issues/1369) 의 안내를 따라 빌드합니다.

```
cd {path_to_LightGBM-2.0.11}
export CXX=g++-8 CC=gcc-8
mkdir build ; cd build
cmake ..
make -j
```

virtualenv 에서 lightgbm 을 install 합니다.

```
source ~/{path_to_my_virtualenv}/bin/activate
pip install --no-binary :all: lightgbm==2.0.11
```

확인합니다.

```
$ python
Python 3.6.3 (v3.6.3:2c5fed86e0, Oct  3 2017, 00:32:08)
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import lightgbm
>>>
```

## Docker

이러한 고충을 지인에게 털어놓으니 `애초에 OSX 에서 시도하지 말고, Docker 로 Linux 환경에서 실행했으면 문제 없었을 걸?` 이라는 조언을 들었습니다. 일견 분했지만 합당한 조언이었다고 생각합니다. 겸사겸사 Docker 공부의 첫 삽을 떠봤습니다. [What is Docker and How to Use it With Python (Tutorial)](https://djangostars.com/blog/what-is-docker-and-how-to-use-it-with-python/) 글을 맨 처음으로 봤는데 유익했습니다. 구글링 해가면서 Dockerfile 을 작성했습니다.

```
FROM python:3.7.2

# docker build 때는 requirements.txt 파일만 복사해서 패키지들 설치한 뒤 다시 requirements.txt 파일은 제거합니다.
# 이후 container 띄울 때 -v 옵션으로 프로젝트 경로를 마운트 합니다.
WORKDIR /foo/bar
COPY requirements.txt .
RUN pip install -r requirements.txt
RUN rm requirements.txt
```

Dockerfile 을 가지고 image 를 생성합니다.

```
docker build --tag your-tag-name .
```

Dockerfile 의 주석에 적었던 것 처럼, container 를 띄울 때는 프로젝트 경로를 마운트 합니다.

```
docker run -i -t --rm -v /path-to-foo-bar:/foo/bar your-tag-name /bin/bash
```

원하는 python 스크립트를 실행해서 lightgbm 이 무리없이 설치되어 사용되었음을 확인할 수 있었습니다.

## 마치며

고통스러웠지만, 다행히 OSX 에서 `lightgbm` 패키지를 사용할 수 있게 되었습니다. 겸사겸사 Docker 공부의 첫 삽을 뜨게 되어 유익했습니다.

## 레퍼런스

* [Microsoft/LightGBM/issues/1369](https://github.com/Microsoft/LightGBM/issues/1369)
* [What is Docker and How to Use it With Python (Tutorial)](https://djangostars.com/blog/what-is-docker-and-how-to-use-it-with-python/)

