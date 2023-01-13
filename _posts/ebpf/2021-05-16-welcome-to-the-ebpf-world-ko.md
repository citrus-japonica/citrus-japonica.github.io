---
layout    : post
title     : "eBPF/XDP (1) Welcome to the eBPF/XDP World"
date      : 2021-05-16
lastupdate: 2021-05-16
categories: ebpf
---

# Interest
[eBPF](https://ebpf.io/) (extended Berkeley Packet Filter)/[XDP](https://www.iovisor.org/technology/xdp) (eXpress Data Path) 를 알게된 것은 if (kakao) 2019 였습니다. 대부분의 article 에서는 kubernetes [CNI](https://github.com/containernetworking/cni) (Container Network Interface) 플러그인으로 [flannel](https://github.com/flannel-io/flannel), [calico](https://www.projectcalico.org/), [weavenet](https://github.com/weaveworks/weave) 의 설치를 안내하고 있습니다. [평범하게 성능이 우수하다는 이유](https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-updated-august-2020-6e1b757b9e49) 였습니다. 그러나 if (kakao) 2019 에서 카카오는 [cilium](https://cilium.io/) 을 사용 중이며, 이것이 eBPF/XDP 를 기반으로 동작한다는 것을 사실을 소개하였습니다. 저는 eBPF/XDP 가 리눅스 커널 프로그래밍으로 개발된다는 것을 알게 되면서 큰 관심을 가지게 되었습니다.

# Introduction
[CNCF 에서 올해 2021년의 주목할 기술 중에 ‘eBPF’ 를 꼽을 정도](https://about.gitlab.com/blog/2020/11/24/cncf-five-technologies-to-watch-in-2021/) 로 네트워크 가속 업계에서 상당한 관심을 보이고 있습니다. 이러한 eBPF/XDP 가 무엇인지 간략하게 소개하겠습니다.

## eBPF

<p align="center"><img src="https://ebpf.io/static/overview-3c0c9cd2010cb0b7fdc26e5e17d99635.png" width="90%" height="90%"></p>

eBPF 는 커널 변경 또는 모듈 로드 없이 리눅스 커널에서 프로그램을 실행할 수 있는 기술입니다. 즉, 커널을 동적으로 프로그래밍 할 수 있다는 뜻이 되겠습니다. eBPF 는 socket, TCP/IP stack, 네트워크 디바이스 드라이버와 같은 네트워크 관련 부분 뿐만 아니라 프로세스, 저장 장치, 원하면 사용자 정의 프로브를 만들어 후크 지점을 선정할 수 있습니다. 이들 후크 지점에서 이벤트가 발생하면 eBPF 프로그램이 자동 실행될 것입니다.

- Verifier는 악의적인 접근 또는 시스템 장애를 일으킬 수 있는 부분을 사전에 차단합니다.
- JIT (Just-in-Time) 은 eBPF 바이트 코드(ELF format)를 머신에 맞는 명령어 셋으로 변환합니다.
- Map 은 eBPF 프로그램 들과 유저 프로그램이 정보를 공유하기 위한 KVS (Key-Value Store) 입니다.

## XDP

<p align="center"><img src="https://www.iovisor.org/wp-content/uploads/sites/8/2016/09/xdp-packet-processing-768x420.png" width="90%" height="90%"></p>

XDP 는 리눅스 네트워크 스택보다 더 낮은 네트워크 드라이버 영역에서 작동하는 eBPF 프로그램입니다. 따라서 리눅스의 네트워크 스택을 전혀 통과하지 않은 상태에서 빠르게 패킷을 폐기하거나 리다이렉션 할 수 있는 [이점](https://blog.cloudflare.com/how-to-drop-10-million-packets/) 이 있습니다. 그러나 ingress traffic 만을 처리할 수 있다는 제약이 있습니다. 따라서, egress traffic 에 대한 정책을 요구하는 경우 XDP 가 아닌 socket-ebpf, tc-ebpf 를 사용해야 합니다.

# Digression
[DPDK](https://www.dpdk.org/) (Data Plane Development Kit) 는 eBPF/XDP 와는 달리 소위 ‘kernel bypass’ 를 통해 원하는 목적을 달성하기 위한 기술입니다. 상당 부분을 유저 영역에서 처리하게 되면서 유저-커널 영역 간 컨텍스트 스위칭 빈도수를 줄일 수 있습니다. 다만 오랜 세월동안 다져진 커널의 안정된 프로토콜 스택, 라우팅 정보 그리고 그것을 이용하는 도구를 모두 사용할 수 없기에 이것을 모두 구현해야 합니다. 반면 eBPF/XDP는 커널의 모든 기능을 사용하면서 원하는 기능의 프로그래밍이 가능합니다. 그러나 이것이 무조건 매력적인 방향으로만 동작하지 않는다는 것을 유념하시기 바랍니다. 시스템과 맞닿은 커널 영역에서 동작한다는 것은 그만큼 [보안에 더 많은 주의를 요구](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=BPF) 한다는 것입니다. 그리고 논쟁의 여지가 있으나 eBPF/XDP 가 언제나 뛰어난 성능을 보이는 것이 아닙니다. [[1](https://pchaigno.github.io/ebpf/2020/09/29/bpf-isnt-just-about-speed.html),[2](https://kinvolk.io/blog/2020/09/performance-benchmark-analysis-of-egress-filtering-on-linux/)] XDP 논문의 테스트 결과 모든 부분에서 DPDK 보다 낮은 성능을 보이고 있음을 확인할 수 있습니다. 비록 DPDK 보다 평균적으로 낮은 CPU 점유율을 가지고 있지만 DPDK 처럼 [CPU 100% 을 사용해야 최대 처리량을 달성](https://dl.acm.org/doi/10.1145/3281411.3281443) 할 수 있습니다. eBPF/XDP 진영에서는 이 performance gap 을 bridge 하기 위해서 많은 노력을 기울이고 있습니다. 저는 Smart NIC + XDP offload 가 DPDK의 성능을 능가할 수 있게 할 key 라고 생각합니다.

# Wrap Up
다음 포스트 eBPF/XDP (2) Samples on Virtual Network 에서는 eBPF/XDP 커뮤니터 진영에서 bible로 여기고 있는 [BPF and XDP Reference Guide](https://docs.cilium.io/en/stable/bpf/) 의 샘플 코드를 ‘직접’ 실행하고 분석하여 eBPF/XDP를 체득하도록 하겠습니다.

eBPF/XDP 에 대한 설명은 책 한 권에 써야 할 정도로 방대합니다. 따라서 부족한 설명은 추후 지속 업데이트할 예정입니다.

# References
- [calico](https://www.projectcalico.org/)
- [cilium](https://cilium.io/)
- [CNI](https://github.com/containernetworking/cni)
- [DPDK](https://www.dpdk.org/)
- [eBPF](https://ebpf.io/)
- [flannel](https://github.com/flannel-io/flannel)
- [weavenet](https://github.com/weaveworks/weave)
- [XDP](https://www.iovisor.org/technology/xdp)
