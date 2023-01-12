---
layout    : post
title     : "How to Build a Custom Kernel RPM for CentOS 8"
date      : 2021-04-13
lastupdate: 2021-04-13
categories: linux
---

# Kernel RPM ?

여기서는 [Elrepo](http://elrepo.org/tiki/HomePage)의 [SRPMS](https://elrepo.org/linux/kernel/el8/SRPMS/)을 이용하여 CentOS 8 을 위한 커널 관련 패키지 rpm 들을 빌드하는 방법을 알아보겠습니다. CentOS 는 이 방법을 공식 문서를 통해 알려주고 있습니다. [I Need the Kernel Source](https://wiki.centos.org/HowTos/I_need_the_Kernel_Source) 을 참고하시기 바랍니다. 이 포스팅은 실질적으로 해당 문서를 기반으로 만들었습니다. 문서 앞 부분의 요약은 다음과 같습니다.

> 우리는 필요한 대부분을 지원한다. 뭐가 불만이길래 이걸 하려 하는가?

CentOS는 상당히 공격적인 붉은색의 바탕 경고문과 함께 커널 커스텀과 관련된 행위에 대한 모든 책임을 사용자에게 돌리고 있습니다. 어지간히 사용자들한테 시달렸나 봅니다. 저는 이들에게 요구할 것이 없기 때문에 그대로 작업을 진행합니다.

SRPMS 는 rpm 패키지를 배포하는 과정에서 발생한 원본 소스와의 차이 및 추가 파일들에 대한 정보를 포함하고 있습니다. 따라서 SRPMS 를 이용하면 사용자는 극히 일부만의 수정으로 원하는 목적을 달성할 수 있습니다. 또한 빌드 결과물이 rpm 이라는 것은 설치 이후 삭제 또한 용이하다는 것을 의미합니다. 설치한 커널에 문제가 있는 경우 언제든지 삭제를 통해 이전 상태로 복귀할 수 있습니다.

# Ready

## ⚠️ Warning
- 아래 과정에서 root 및 non-root 계정을 번갈아가며 사용하기 때문에 주의가 필요합니다. root 로 실행해야 하는 경우 `# as root`, 일반 유저 계정으로 실행해야 하는 경우 `# as non-root` 라고 명시하였습니다.
- 1-2 주 전 마친 작업을 작성하려 하니 어느새 버전이 업데이트되어 명령을 그대로 실행할 수 없게 되었습니다. 😭 예를 들어 5.11.11 에서 5.11.12 로 변경된 경우 해당 숫자만 적절히 교체하여 주시기 바랍니다.

## HW Specification
8 Core/16 GB MEM/64 GB HDD 를 할당한 CentOS 8 가상 머신에서 작업을 진행하였습니다. CPU와 MEM의 경우 커널 빌드가 끝난 후 필요에 따라 줄일 수 있지만 HDD의 경우 커널 빌드를 위해 ‘/home’ 파티션에는 최소 25 GB 의 여유 용량이 필요합니다.

## 유저 계정 생성
[I Need the Kernel Source](https://wiki.centos.org/HowTos/I_need_the_Kernel_Source) 에서는 root 를 이용한 작업을 권장하지 않고 있습니다. 일반 유저 계정이 없다면 다음과 같이 생성합니다.

``` bash
# as root 
useradd -m student
su - student
```

일반 유저 계정으로 전환되었다면 바로 다음 단계를 진행합니다.

## RPM Build Path
``` bash
# as non-root
mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
echo '%_topdir %(echo $HOME)/rpmbuild' > ~/.rpmmacros
```
rpm 빌드를 위한 디렉토리를 생성하고 빌드 path를 .rpmmacros 에 넣어주었습니다.

## Dependencies for Build Kernel
``` bash
# as root
yum install -y vim rpm-build asciidoc audit-libs-devel bc binutils-devel bison \
elfutils-devel flex gcc git java-devel libcap-devel make ncurses-devel net-tools \
newt-devel numactl-devel openssl-devel pciutils-devel perl-ExtUtils-Embed perl-Carp \
perl-devel perl-generators perl-interpreter python3-devel python3-docutils \
rsync xmlto xz-devel zlib-devel
```

## Import SRPMS
``` bash
# as non-root 
rpm -i https://elrepo.org/linux/kernel/el8/SRPMS/kernel-ml-5.11.11-1.el8.elrepo.nosrc.rpm 2>&1 | grep -v exist
cd ~/rpmbuild/SOURCES
curl https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.11.11.tar.xz -o linux-5.11.11.tar.xz
cd ~/rpmbuild/SPECS
rpmbuild -bp --target=$(uname -m) kernel-ml-5.11.spec
```

Elrepo 의 SRPMS 를 이용하여 커널 RPM 빌드가 가능한 상태를 만들어 줍니다. SRPMS 를 import 해도 커널 소스는 직접 [kernel.org](https://www.kernel.org/) 로 부터 가져와야 합니다.

## Motify Config File
우리가 커널 빌드마다 `make menuconfig` 를 통해 생성해 왔던 커널 빌드 설정 파일 `.config` 은 `~/rpmbuild/SOURCES/config-5.11.11-x86_64` 파일을 이용하여 생성됩니다.

``` bash
# as non-root
vim ~/rpmbuild/SOURCES/config-5.11.11-x86_64
```

편집기를 열어 우리가 원하는 설정이 정확히 입력이 되어 있는지 확인합니다.

## Assign Build ID
[I Need to Build a Custom Kernel](https://wiki.centos.org/HowTos/Custom_Kernel) 에서는 커스텀 커널이 호스트 커널과 동일한 버전을 가지지 않도록 buildid 를 할당하는 것을 권장하고 있습니다. 커널 버전 뒤에 ‘.local’ 을 붙이기 위해 다음의 작업을 진행합니다.

``` bash
# as non-root 
cd ~/rpmbuild/SPECS
vim kernel-ml-5.11.spec
```

편집기를 열면 다음과 같은 부분이 있습니다.

```
#define buildid .local
```

다음과 같이 수정합니다.

```
%define buildid .local
```

## Install pahole
만약 커널 수준 디버깅을 위해 `CONFIG_DEBUG_INFO_BTF`를 활성화하였다면 `pahole`이 필요합니다.

``` bash
# as root 
yum install -y cmake
cd /tmp
git clone https://github.com/acmel/dwarves
cd dwarves
mkdir build
cd build
cmake -D__LIB=lib ..
make install
cat << EOF > /etc/ld.so.conf.d/local.conf
/usr/local/lib
EOF
ldconfig
```

# Build Kernel
드디어 모든 준비 과정이 끝나고 커널을 빌드할 차례입니다.

``` bash
# as non-root
rpmbuild -bb --target=`uname -m` --with baseonly --without debug \
--without debuginfo kernel-ml-5.11.spec 2> build-err.log | tee build-out.log
```

저는 debug kernel 이 필요하지 않아 옵션으로 제외하였습니다. 필요한 경우 이를 해제하면 됩니다. 빌드 시 문제가 발생하면 리다이렉션된 `build-err.log` 를 확인합니다. 에러가 없다면 다음과 같은 출력으로 종료됩니다.

```
...
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-headers-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/perf-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/python3-perf-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-tools-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-tools-libs-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-tools-libs-devel-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/bpftool-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-core-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-devel-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-modules-5.11.11-1.el8.local.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-modules-extra-5.11.11-1.el8.local.x86_64.rpm
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.UL2cy0
```

생성된 rpm 파일들을 확인합니다.

``` bash
[student@localhost x86_64]$ cd ~/rpmbuild/RPMS/x86_64
[student@localhost x86_64]$ ls -l
total 98840
-rw-rw-r--. 1 student student  1381724 Apr  6 21:42 bpftool-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student    58276 Apr  6 21:42 kernel-ml-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student 37289640 Apr  6 21:42 kernel-ml-core-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student 14308676 Apr  6 21:42 kernel-ml-devel-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student  1456896 Apr  6 21:42 kernel-ml-headers-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student 36642548 Apr  6 21:42 kernel-ml-modules-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student  1143432 Apr  6 21:42 kernel-ml-modules-extra-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student   249540 Apr  6 21:42 kernel-ml-tools-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student    91120 Apr  6 21:42 kernel-ml-tools-libs-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student    61128 Apr  6 21:42 kernel-ml-tools-libs-devel-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student  7748712 Apr  6 21:42 perf-5.11.11-1.el8.local.x86_64.rpm
-rw-rw-r--. 1 student student   759196 Apr  6 21:42 python3-perf-5.11.11-1.el8.local.x86_64.rpm
```

이들 중에서 kernel, kernel-core, kernel-modules 3개를 설치하면 해당 커널을 설치 즉시 사용할 수 있습니다.

``` bash
yum localinstall -y kernel-ml-5.11.11-1.el8.local.x86_64.rpm kernel-ml-core-5.11.11-1.el8.local.x86_64.rpm kernel-ml-modules-5.11.11-1.el8.local.x86_64.rpm
```

CentOS 8의 경우 새로 설치한 커널은 grub 부트로더의 최상단에 위치하여 자동으로 선택되고 있기 때문에 추가 설정을 요구하지 않습니다.

이어서 CentOS 7 사용자를 위한 [How to Build a Custom Kernel RPM for CentOS 7](/blog/how-to-build-a-custom-kernel-rpm-for-centos-7-ko/) 을 진행하겠습니다.

# References

- [Elrepo](http://elrepo.org/tiki/HomePage)
- [I Need the Kernel Source](https://wiki.centos.org/HowTos/I_need_the_Kernel_Source)
- [I Need to Build a Custom Kernel](https://wiki.centos.org/HowTos/Custom_Kernel)
- [kernel.org](https://www.kernel.org/)
- [SRPMS](https://elrepo.org/linux/kernel/el8/SRPMS/)