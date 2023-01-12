---
layout    : post
title     : "How to install the latest version of htop in CentOS"
date      : 2021-04-12
lastupdate: 2023-01-12
categories: linux
---

커널 빌드, ansible-playbook(okd, oVirt, kubespray), puppet(PackStack), devStack 등등 오래 걸리는 작업을 실행해놓고 뒤늦게 생각나는 것…

> 아… htop 설치 안 했네… XX 🤬

[htop](https://github.com/htop-dev/htop) 은 시스템 상에 동작하는 프로세스에 대한 CPU 및 MEM 점유율에 집중할 수 있는 모니터링 프로세스입니다. 전통적인 `top` 에 비해 조작 인터페이스도 쉽고, 무엇보다 컬러풀 해서 좋습니다. 😄 기본적으로 모든 쓰레드를 보여주는 것도 맘에 듭니다. 물론 이것이 리스트 개수를 크게 증폭시키지만 PageUp/Down 키로 넘기면서 충분히 볼 수 있는 수준입니다. 저는 MacOS 에도 `brew install htop` 을 설치하여 사용할 정도로 의존도가 높은 편입니다. 

Ubuntu 의 경우, `apt install -y htop` 한 줄로 설치해서 바로 사용할 수 있습니다. CentOS 의 경우 epel repo 를 활성화하면 `yum install -y htop` 을 이용해서 설치 가능합니다.

``` bash
yum install -y epel-release 
yum install -y htop 
```

그러나 여기서는 CentOS base repo 의 패키지들을 엎어 의존성을 자주 손상시키는 epel repo 를 사용하지 않고 직접 빌드하여 `htop` 을 설치하는 방법을 알아보겠습니다.

> 이하 과정은 root 권한으로 모두 실행하였습니다.

# Install dependencies
``` bash
yum install -y gcc make autoconf automake ncurses-devel
```

`gcc`, `make`, `autoconf`, `automake` 는 `htop` 빌드를 위한 도구이며, `ncurses-devel` 는 TUI(text user interface)를 위한 개발 라이브러리 입니다.

# Build and Install htop
이 글을 작성하는 시점을 기준으로 GitHub 에서 가장 최신 버전의 소스를 받아 빌드 및 설치 합니다. 버전은 3.0.5 로 올해 2021년 1월 12일에 릴리즈되었습니다. (블로그 마이그레이션 중인 2023-01-12 현재 버전은 3.2.1)

``` bash
cd /tmp
curl -sSL https://github.com/htop-dev/htop/archive/refs/tags/3.0.5.tar.gz | tar -xz
cd htop-3.0.5/
./autogen.sh && ./configure && make install
```

# Run
``` bash
htop
```

실행하면 다음과 같은 화면을 볼 수 있습니다.

<p align="center"><img src="/assets/img/linux/htop.png" width="90%" height="90%"></p>

빠져나오기 위해서는 `q` 를 누르면 됩니다.

`htop` 은 많은 기능을 가지고 있지만 저는 ‘F4 Filter’, ‘F6 SortBy’, ‘F9 Kill` 이 세 가지 위주로 사용하고 있습니다. 자세한 사용법 및 헤더 정보의 의미는 구글링을 통해서 쉽게 조회 가능하기 때문에 생략하겠습니다.

# References
- [htop](https://github.com/htop-dev/htop/)