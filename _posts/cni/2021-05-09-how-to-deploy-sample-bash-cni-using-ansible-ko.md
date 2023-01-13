---
layout    : post
title     : "CNI (1) How to deploy sample bash CNI using ansible"
date      : 2021-05-09
lastupdate: 2021-05-09
categories: cni
---

Bash script 로 만든 간단한 CNI(container network interface)인 [cni-from-scratch](https://github.com/eranyanay/cni-from-scratch) 의 `my-cni-demo` 를 ansible을 이용하여 쉽게 배포할 수 있도록 만든 저의 작업 [ansible-sample-cni](https://github.com/citrus-japonica/ansible-sample-cni) 를 소개합니다.

# CNI
[CNI](https://github.com/containernetworking/cni) (container network interface)는 공통 인터페이스입니다. 즉, 어떠한 컨테이너 런타임을 사용하더라도 CNI를 통해 동일한 네트워크를 구성할 수 있습니다. 그리고 그 구현은 [plugins](https://github.com/containernetworking/plugins) 로 분리되어 있어 네트워크 기술 변화에 효과적으로 대응할 수 있습니다. CNI가 지원하는 컨테이너 런타임, 그리고 CNI 기본 제공 plugins 외에 3rd party plugins 는 CNI의 `README.md` 를 참고하시기 바랍니다.

# Ansible
[Ansible](https://github.com/ansible/ansible) 은 어플리케이션 및 설정 배포를 위한 자동화 시스템 도구입니다. 이것은 기본적으로 멱등성(idempotent)을 보장하도록 설계되어 있습니다. 여러 번 실행하더라도 playbook(YAML 정의서)에 선언한 바와 동일한 결과를 보여줄 수 있습니다. 그러나 언제나 이것이 보장되지 않는다는 것을 경계해야 합니다. 네트워크 문제로 패키지 및 파일 다운로드 프로세스가 지연되는 경우, 호스트 시스템 문제로 태스크 수행에 실패하는 경우 등 예기치 못한 장애가 발생하면 재실행에도 원하는 결과를 얻을 수 없습니다. 이 모든 것을 고려하여 playbook을 작성하는 것은 매우 고된 작업입니다.

Ansible은 agent를 설치해야 하는 다른 솔루션들과는 달리 ssh 원격 접속을 이용합니다. 따라서 매끄러운 자동화를 위하여 사용자의 password 입력을 요구하지 않는 [key-based authentication ssh](https://www.redhat.com/sysadmin/configure-ssh-keygen) 접속 환경을 구성하는 것이 매우 권장됩니다.

# ansible-sample-cni
이 포스팅에서는 playbook [deploy.yaml](https://github.com/citrus-japonica/ansible-sample-cni/blob/main/deploy.yaml) 을 중심으로 [cni-from-scratch](https://github.com/eranyanay/cni-from-scratch) 의 `my-cni-demo` 배포에 대하여 설명합니다. Playbook 은 다수의 play 정의를 가질 수 있습니다. 여기서는 단일 play 만을 사용하고 있습니다. [ansible-sample-cni](https://github.com/citrus-japonica/ansible-sample-cni) 의 Result를 도식화한 결과는 다음과 같습니다. 

<p align="center"><img src="/assets/img/cni/cni_cluster.png" width="90%" height="90%"></p>

## Play
``` yaml
---
- name: Deploy my-cni-demo using ansible
  hosts: all
  vars_files:
  - vars.yaml
```

이 부분은 play 전체를 대상으로 하여 생명주기를 가지고 있는 부분입니다. Playbook은 `vars_files` 를 이용하여 외부 파일로부터 파라미터화된 설정 변수를 얻을 수 있습니다. 이 프로젝트에서는 아래의 `vars.yaml` 파일 하나만을 사용하였습니다.

``` yaml
name: my-cni-demo
podcidr: 10.240.0.0/16
interface: ens32
bridge: cni0
nodeinfo:
- name: cni-master01
  ip: 192.168.100.100
  podcidr: 10.240.0.0/24
- name: cni-worker01
  ip: 192.168.100.101
  podcidr: 10.240.1.0/24
- name: cni-worker02
  ip: 192.168.100.102
  podcidr: 10.240.2.0/24
```

Play에서는 이 `vars.yaml` 의 데이터 “key” 값을 이용하여 “value” 값을 얻어 사용합니다. 아래 Tasks 에서 사용 예를 확인할 수 있습니다.

CNI는 클러스터 내의 모든 노드를 대상으로 배포되어야 합니다. 이는 calico, cilium 과 같은 3rd party plugin 들을 Daemonset으로 배포하는 것과 같습니다. 여기서 `all` 은 아래 inventory 파일에 정의한 key 값입니다. 이 `all` 에 호스트 이름을 순서대로 작성합니다.

``` ini
[all]
cni-master01
cni-worker01
cni-worker02
```

## Tasks
하나의 play는 다수의 tasks 를 정의할 수 있습니다. Task의 결과를 `register` 를 이용하여 저장한 뒤 이를 다음 task 에서 활용하는 방식을 취하지 않는 한 각 task 들은 play에 정의된 순서에 맞추어 독립적으로 동작합니다.

``` yaml
  tasks:
    - name: install dependencies
      apt:
        name:
          - bridge-utils
          - jq
        state: latest
```

1번 task는 의존성 패키지 설치입니다. 이것은 `apt install -y bridge-utils jq` 와 동일합니다. `bridge-utils` 는 리눅스 브리지 관리 도구 `brctl` 를 위한 것이며, `my-cni-demo` 스크립트에서 브릿지 `cni0` 를 생성하는 데에 사용합니다. `jq` 는 JSON 포맷을 다루기 위한 도구입니다. `my-cni-demo` 스크립트에서 `podcidr` 을 추출하는 데에 사용되고 있습니다.

``` yaml
    - name: allow forwarding for src-ips in podcidr
      iptables:
        chain: FORWARD
        source: "{% raw %}{{ podcidr }}{% endraw %}"
        jump: ACCEPT

    - name: allow forwarding for dst-ips in podcidr
      iptables:
        chain: FORWARD
        destination: "{% raw %}{{ podcidr }}{% endraw %}"
        jump: ACCEPT
``` 

2, 3번 task는 Pod CIDR에 포함되는 출발지 주소 및 목적지 주소를 가진 패킷을 통과시키기 위한 iptables 명령을 수행합니다. `{% raw %}{{ podcidr }}{% endraw %}` 을 `vars.yaml` 에 정의된 값으로 대체되면 다음 명령과 동일합니다.

``` bash
iptables -A FORWARD -s 10.240.0.0/16 -j ACCEPT
iptables -A FORWARD -d 10.240.0.0/16 -j ACCEPT
```

``` yaml 
    - name: allow outgoing internet
      iptables:
        table: nat
        chain: POSTROUTING
        source: "{% raw %}{{ item.podcidr }}{% endraw %}"
        jump: MASQUERADE
        out_interface: "!{% raw %}{{ bridge }}{% endraw %}"
      loop: "{% raw %}{{ nodeinfo }}{% endraw %}"
      when: ansible_hostname == item.name
```

4번 task는 외부 인터넷과의 통신을 위하여 내부 pod들이 서로 통신하는 브릿지(여기서는 `cni0`)를 제외하고 나가는 패킷을 인터페이스들이 masquerade(SNAT)하도록 iptables를 설정합니다. 해당 노드의 출발지 IP CIDR을 특정하기 위해서는 `vars.yaml` 의 `nodeinfo` 에서 정보를 추출해야 합니다. `loop` 에 `{% raw %}{{ nodeinfo }}{% endraw %}` 를 두면 각 nodeinfo 엔트리를 `item` 으로 접근할 수 있습니다. 호스트명 `ansible_hostname` 과 `item.name` 이 일치할 때만 task 를 실행하도록 설정하였고, 위 명령에 따라 `{% raw %}{{ item.podcidr }}{% endraw %}` 은 해당 노드의 CIDR로 대체됩니다. 예를 들어 호스트명이 cni-master01 이라면 다음과 같은 명령과 동일한 작업을 수행합니다.

``` bash
iptables -t nat -A POSTROUTING -s 10.240.0.0/24 ! -o cni0 -j MASQUERADE
``` 

``` yaml
    - name: allow communication across hosts
      command: >
          ip route add {% raw %}{{ item.podcidr }} via {{ item.ip }} dev {{ interface }}{% endraw %}
      ignore_errors: yes
      when: ansible_hostname != item.name
      loop: "{% raw %}{{ nodeinfo }}{% endraw %}"
```

5번 task는 노드를 건너 pod 간의 네트워크를 연결하기 위하여 라우팅 테이블 엔트리를 추가합니다. 만약 호스트 네트워크에 연결된 인터페이스 ens32가 수신한 패킷의 도착지 주소가 cni-worker01의 podcidr 10.240.1.0/24에 포함하는 경우 다음 hop 을 cni-worker01의 IP 주소로 설정하면 패킷은 cni-worker01로 건너갈 수 있게 되어 내부 pod에 도달할 수 있습니다.
라우팅 테이블 엔트리는 자신이 아닌 모든 노드를 대상으로 실시하여야 하므로 `ansible_hostname != item.name` 조건을 부여합니다. 따라서 cni-master01에서 실시된 task는 다음 명령과 동일합니다.

``` bash
ip route add 10.240.1.0/24 via 192.168.100.101 dev ens32 
ip route add 10.240.2.0/24 via 192.168.100.102 dev ens32 
```

``` yaml
    - name: ensure that /etc/cni/net.d exists
      file:
        state: directory
        recurse: yes
        path: /etc/cni/net.d

    - name: deploy configuration file
      template:
        src: 10-my-cni-demo.conf.j2
        dest: /etc/cni/net.d/10-my-cni-demo.conf
        owner: root
        group: root
        mode: 0644
```

6,7번 task는 CNI 설정파일 `10-my-cni-demo.conf` 를 모든 노드의 `/etc/cni/net.d` 경로에 추가합니다. 각 노드마다 podcidr 값을 다르게 부여하기 위해 jinja2 template을 사용합니다. Template 파일 `10-my-cni-demo.conf.j2` 은 다음과 같습니다.

``` 
{% raw %}
{
    "cniVersion": "0.3.1",
    "name": "{{ name }}",
    "type": "{{ name }}",
{% for node in nodeinfo %}
{% if node.name == ansible_hostname %}
    "podcidr": "{{ node.podcidr }}"
{% endif %}
{% endfor %}
}
{% endraw %}
```

이를 통해 해당 노드에 맞는 nodeinfo.podcidr를 추가한 `10-my-cni-demo.conf` 를 모든 노드에 배포할 수 있습니다. 위와 같이 jinja2에서 for loop와 if statement를 이용하는 방법에 대한 예제는 [[1](https://www.arctiq.ca/our-blog/2017/2/16/ansible-jinja-warrior-loop-variable-scope/)]을 참고바랍니다.

``` yaml
    - name: deploy binary file
      template:
        src: my-cni-demo.j2
        dest: /opt/cni/bin/my-cni-demo
        owner: root
        group: root
        mode: 0755
```

마지막 8번 task는 CNI 실행파일 `my-cni-demo` 를 `/opt/cni/bin` 경로에 추가합니다. `mode` 는 실행 권한을 포함하도록 0755(-rwxr-xr-x)로 설정하였습니다.

# Wrap Up
[cni-from-scratch](https://github.com/eranyanay/cni-from-scratch) 의 CNI를 수동 입력없이 쉽게 배포할 수 있도록 ansible로 자동화하였습니다. `vars.yaml` 만 오류없이 작성하면 몇 개의 노드가 추가되더라도 노드간 또는 내외부간 네트워크가 가능한 클러스터를 구축할 수 있습니다. 실습 환경에서 pod를 배포하고 연결을 테스트하는 방법은 [ansible-sample-cni](https://github.com/citrus-japonica/ansible-sample-cni) 의 Result를 참고바랍니다.