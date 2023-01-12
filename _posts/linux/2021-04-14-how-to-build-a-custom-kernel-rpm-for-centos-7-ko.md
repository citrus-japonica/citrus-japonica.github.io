---
layout    : post
title     : "How to Build a Custom Kernel RPM for CentOS 7"
date      : 2021-04-14
lastupdate: 2021-04-14
categories: linux
---

CentOS 에 Kubernetes 을 설치한 뒤 cilium plugin 을 설치하려 하였으나 cilium pod 가 CrashLoopBackOff 에 빠지는 문제가 있었습니다. cilium 의 [System Requirements](https://docs.cilium.io/en/stable/operations/system_requirements/) 를 통해, CentOS 7의 커널 버전은 3.10.0-1160.el7.x86_64로 ’>= 4.9.17’ 을 충족하지 못하는 것을 확인하였습니다. 또한 cilium 이 요구하는 몇 가지 커널 설정 옵션들이 존재하지 않거나 비활성화 되어 있음을 확인하였습니다. cilium 을 위해 반드시 활성화되어야 한다는 옵션들은 다음과 같습니다.

```
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_NET_CLS_BPF=y
CONFIG_BPF_JIT=y
CONFIG_NET_CLS_ACT=y
CONFIG_NET_SCH_INGRESS=y
CONFIG_CRYPTO_SHA1=y
CONFIG_CRYPTO_USER_API_HASH=y
CONFIG_CGROUPS=y
CONFIG_CGROUP_BPF=y
```

현 호스트 커널의 설정은 `/boot/config-$(uname -r)` 파일에서 확인할 수 있습니다. 아래 예제는 `grep`을 이용하여 ‘BPF’ 라는 이름이 들어간 모든 설정을 확인하는 방법입니다.

``` bash
[root@localhost ~]# cat /boot/config-$(uname -r) | grep BPF
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT_ALWAYS_ON=y
CONFIG_NETFILTER_XT_MATCH_BPF=m
CONFIG_NET_CLS_BPF=m
CONFIG_BPF_JIT=y
CONFIG_HAVE_EBPF_JIT=y
CONFIG_BPF_EVENTS=y
CONFIG_BPF_KPROBE_OVERRIDE=y
```

당장 `CONFIG_CGROUP_BPF` 가 존재하지 않는 것을 볼 수 있습니다. 이처럼 비슷한 방법으로 옵션들이 빠짐없이 적용되어 있는지 살펴봅시다. 만약 그렇지 않다면 우리는 설정을 수정하여 커널을 재빌드해야 합니다.

# Ready

## Software Collections

처음엔 [How to Build a Custom Kernel RPM for CentOS 8](/blog/how-to-build-a-custom-kernel-rpm-for-centos-8-ko/) 의 복사/붙여넣기 같지만 과정을 따르면 ‘큰’ 차이가 하나 있습니다. 바로 `devtoolset-9`에 대한 의존성입니다. 현재 CentOS 7 2009의 gcc 버전은 4.8.5 입니다.

``` bash
[root@localhost ~]# gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/lto-wrapper
Target: x86_64-redhat-linux
Configured with: ../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-linker-hash-style=gnu --enable-languages=c,c++,objc,obj-c++,java,fortran,ada,go,lto --enable-plugin --enable-initfini-array --disable-libgcj --with-isl=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/isl-install --with-cloog=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/cloog-install --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
Thread model: posix
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
```

반면 devtoolset-9-gcc 의 버전은 9.3.1 입니다.

``` bash
[root@localhost ~]# scl enable devtoolset-9 bash
[root@localhost ~]# gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/opt/rh/devtoolset-9/root/usr/libexec/gcc/x86_64-redhat-linux/9/lto-wrapper
Target: x86_64-redhat-linux
Configured with: ../configure --enable-bootstrap --enable-languages=c,c++,fortran,lto --prefix=/opt/rh/devtoolset-9/root/usr --mandir=/opt/rh/devtoolset-9/root/usr/share/man --infodir=/opt/rh/devtoolset-9/root/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-shared --enable-threads=posix --enable-checking=release --enable-multilib --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-gcc-major-version-only --with-linker-hash-style=gnu --with-default-libstdcxx-abi=gcc4-compatible --enable-plugin --enable-initfini-array --with-isl=/builddir/build/BUILD/gcc-9.3.1-20200408/obj-x86_64-redhat-linux/isl-install --disable-libmpx --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
Thread model: posix
gcc version 9.3.1 20200408 (Red Hat 9.3.1-2) (GCC)
```

scl 는 [Software Collections](https://www.softwarecollections.org/en/) 를 줄인 단어입니다. 이것은 ‘chroot’ 를 이용한 격리를 이용하여 마치 일종의 툴체인을 제공하는 기술이라 보면 됩니다. 그러나 `$PS1` 이 구분되지 않아 내가 호스트인지 chroot 인지 구분이 어렵고 지원하는 clang 버전도 낮은 등 문제가 많았습니다. scl repo 에서 원하는 패키지 탐색할 시간에 개발 환경이 구성된 ‘컨테이너’에 호스트 볼륨 붙여 소스 코드를 빌드하는 것이 더 현명할 것 같습니다.

## ⚠️ Warning
- 아래 과정에서 root 및 non-root 계정을 번갈아가며 사용하기 때문에 주의가 필요합니다. root 로 실행해야 하는 경우 `# as root`, 일반 유저 계정으로 실행해야 하는 경우 `# as non-root` 라고 명시하였습니다.
- 1-2 주 전 마친 작업을 작성하려 하니 어느새 버전이 업데이트되어 명령을 그대로 실행할 수 없게 되었습니다. 😭 예를 들어 5.11.11 에서 5.11.12 로 변경된 경우 해당 숫자만 적절히 교체하여 주시기 바랍니다.

## HW Specification

8 Core/16 GB MEM/64 GB HDD 를 할당한 CentOS 8 가상 머신에서 작업을 진행하였습니다. CPU와 MEM의 경우 커널 빌드가 끝난 후 필요에 따라 줄일 수 있지만 HDD의 경우 커널 빌드를 위해 ‘/home’ 파티션에는 최소 25 GB 의 여유 용량이 필요합니다.

## 유저 계정 생성
[I Need to Build a Custom Kernel](https://wiki.centos.org/HowTos/Custom_Kernel) 에서는 root 를 이용한 작업을 권장하지 않고 있습니다. 일반 유저 계정이 없다면 다음과 같이 생성합니다.

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
newt-devel numactl-devel openssl-devel pciutils-devel perl-ExtUtils-Embed \
perl-Carp perl-devel perl-generators perl-interpreter python3-devel python3-docutils \
rsync xmlto xz-devel zlib-devel centos-release-scl-rh
```

``` bash
# as root 
yum install -y devtoolset-9-gcc devtoolset-9-binutils devtoolset-9-runtime \
scl-utils python3 python-devel
```

## Import SRPMS
``` bash
# as non-root 
rpm -i https://elrepo.org/linux/kernel/el7/SRPMS/kernel-ml-5.11.11-1.el7.elrepo.nosrc.rpm 2>&1 | grep -v exist
cd ~/rpmbuild/SOURCES
curl https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.11.11.tar.xz -o linux-5.11.11.tar.xz
cd ~/rpmbuild/SPECS
rpmbuild -bp --target=$(uname -m) kernel-ml-5.11.spec
```
Elrepo 의 SRPMS 를 이용하여 커널 RPM 빌드가 가능한 상태를 만들어 줍니다. SRPMS 를 import 해도 커널 소스는 직접 [kernel.org](https://www.kernel.org/)로 부터 가져와야 합니다.

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

저는 debug kernel 이 필요하지 않아 옵션으로 제외하였습니다. 필요한 경우 이를 해제하면 됩니다. 빌드 시 문제가 발생하면 리다이렉션된 `build-err.log`를 확인합니다. 에러가 없다면 다음과 같은 출력으로 종료됩니다.

```
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-5.11.11-1.local.el7.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-devel-5.11.11-1.local.el7.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-headers-5.11.11-1.local.el7.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/perf-5.11.11-1.local.el7.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/python-perf-5.11.11-1.local.el7.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-tools-5.11.11-1.local.el7.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-tools-libs-5.11.11-1.local.el7.x86_64.rpm
Wrote: /home/student/rpmbuild/RPMS/x86_64/kernel-ml-tools-libs-devel-5.11.11-1.local.el7.x86_64.rpm
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.lbqSKi
```
생성된 rpm 파일들을 확인합니다.

``` bash
[student@localhost x86_64]$ ls -l
total 83104
-rw-rw-r--. 1 student student 65559236 Apr  6 21:55 kernel-ml-5.11.11-1.local.el7.x86_64.rpm
-rw-rw-r--. 1 student student 14312360 Apr  6 21:55 kernel-ml-devel-5.11.11-1.local.el7.x86_64.rpm
-rw-rw-r--. 1 student student  1509764 Apr  6 21:55 kernel-ml-headers-5.11.11-1.local.el7.x86_64.rpm
-rw-rw-r--. 1 student student   246076 Apr  6 21:55 kernel-ml-tools-5.11.11-1.local.el7.x86_64.rpm
-rw-rw-r--. 1 student student   129036 Apr  6 21:55 kernel-ml-tools-libs-5.11.11-1.local.el7.x86_64.rpm
-rw-rw-r--. 1 student student   106220 Apr  6 21:55 kernel-ml-tools-libs-devel-5.11.11-1.local.el7.x86_64.rpm
-rw-rw-r--. 1 student student  2536608 Apr  6 21:55 perf-5.11.11-1.local.el7.x86_64.rpm
-rw-rw-r--. 1 student student   683716 Apr  6 21:55 python-perf-5.11.11-1.local.el7.x86_64.rpm
```

CentOS 7의 경우 kernel 하나만 설치해도 이를 사용할 수 있습니다. 다만 CentOS 8 과는 달리 grub 이 설치한 커널을 기본으로 사용하지 않기 때문에 이를 변경할 필요가 있습니다.

``` bash
# as root 
yum install -y kernel-ml-5.11.11-1.local.el7.x86_64.rpm
grub2-set-default 0
reboot
```

# 마무리

빌드 시간은 CentOS 7,8 둘 다 15-20 분 정도 소요되었습니다. 다소 오래 걸리지만 실패 없이 최소한의 노력으로 커스텀 커널을 빌드하는 좋은 방법입니다. cilium 또한 이상없이 잘 설치되는 것을 확인하였습니다.

``` bash
[root@localhost ~]# cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)
[root@localhost ~]# uname -r
5.11.11-1.local.el7.x86_64
[root@localhost ~]# kubectl get po -n kube-system
NAME                                            READY   STATUS    RESTARTS   AGE
cilium-operator-654456485c-r8pd7                1/1     Running   0          106s
cilium-xpprx                                    1/1     Running   0          106s
coredns-f9fd979d6-j6d42                         1/1     Running   0          11m
coredns-f9fd979d6-tlg97                         1/1     Running   0          11m
etcd-localhost.localdomain                      1/1     Running   0          11m
kube-apiserver-localhost.localdomain            1/1     Running   0          11m
kube-controller-manager-localhost.localdomain   1/1     Running   0          11m
kube-proxy-52jwm                                1/1     Running   0          11m
kube-scheduler-localhost.localdomain            1/1     Running   0          11m
```

# References

- [Elrepo](http://elrepo.org/tiki/HomePage)
- [I Need the Kernel Source](https://wiki.centos.org/HowTos/I_need_the_Kernel_Source)
- [I Need to Build a Custom Kernel](https://wiki.centos.org/HowTos/Custom_Kernel)
- [kernel.org](https://www.kernel.org/)
- [Software Collections](https://www.softwarecollections.org/en/)
- [SRPMS](https://elrepo.org/linux/kernel/el8/SRPMS/)
- [System Requirements (cilium)](https://docs.cilium.io/en/stable/operations/system_requirements/)