---
layout: default
title: About Citrus Japonica
---

# 김강희 (Kanghee Kim)
Computer Architecture and Network Researcher, Linux Certified Engineer, Software Developer, and

<p align="center"><img src="/assets/img/about/cleaner.gif"></p>

`Pump It Up EXPERT LV.1`

# Education

## 박사
인하대학교 전자공학과 2013.03.04 - 2019.08.24

## 석사
인하대학교 전자공학과 2011.02.28 - 2013.02.22

## 학사
인하대학교 전자공학과 2004.03.02 - 2011.02.24

# Skills

## Computer Architecture
Cache data prefetcher 

## Computer Network
Wireless sensor network (Zigbee, WBAN)

## Air traffic control
Radar(ASTERIX) & FPL(ICAO Doc.4444) data processing system

## Linux & Virtualization
Docker, Kubernetes, Ansible, eBPF/XDP

## Programming
C/C++, Qt, Python, Go

## Machine Vision
OpenCV, libtorch, OpenVINO

# Working Experience

## Machine Vision Application Development
(주) 블루타일랩 | 2022 - Current

Qt C++, C/C++, OpenCV C++, libtorch C++, OpenVINO C++, Various Camera SDK C++

머신비전 어플리케이션 개발 목표는 생산 라인 상의 검사 대상을 고해상도 카메라로 촬영하고, 컴퓨터비전 기법을 이용하여 획득한 영상 내 검사 대상을 찾아내거나 검사 대상의 결함을 검출하는 것입니다. 
현재 저는 회사 내에서 디스플레이 패널, FMM 인바 및 유리, 2차 전지 전극 등 다양한 소재에 대한 비전 SW 개발을 수행하고 있습니다. 
UI 개발 프레임워크롤 Qt C++ 를 사용하고, 머신비전을 위해 비전 라이브러리 (OpenCV), 머신러닝 라이브러리 (libtorch, OpenVINO), 다양한 카메라 라이브러리 등을 사용하고 있습니다.

## Healthcare Mobile Application Development
(주) 네오드림스 | 2021

- Java for Android
- Objective C, Swift, and SwiftUI for iOS

본 프로젝트는 웹앱 어플리케이션을 구현하는 것이 목표입니다. 개발 앱은 HTTP 서버와 통신하여 내장 웹 브라우저에 웹화면을 표시하고, REST API 를 이용하여 모바일 기기로 수집한 건강데이터를 전송합니다. 개발 참여 기간은 2022-06-01~2022-12-31 약 7개월입니다. 

### Android
건강데이터 전송 파트만을 담당하였습니다. REST API 통신을 위해 Retrofit2 를 이용하였습니다. 그리고 삼성 안드로이드 폰 대상으로 삼성헬스 앱의 데이터를 수집하기 위해 삼성헬스 SDK API 를 이용하였습니다. 

### iOS
모든 개발 파트를 담당하였습니다. REST API 통신을 위해 Alamofire 를 이용하였습니다. 그리고 비동기적으로 수집되는 건강앱의 데이터들을 병합하기 위해 RxSwift 를 사용하였습니다. 또한 사용자의 건강보험공단 데이터를 수집하기 위해 라온시큐어의 인증 API, 쿠콘의 스크랩 API 를 사용하였습니다.

## Air Traffic Controller Display Console Development
(주) 네오드림스 | 2018 - 2019

Qt C++, C/C++, Linux Network

본 프로젝트는 약 2년의 개발 기간에 걸쳐 항공관제용 관제사 현시 콘솔을 모의하기 위한 소프트웨어를 구현하는 것이 목표입니다. 해당 소프트웨어는 표준화된 항공정보, 항적, 비행계획의 현시 등을 주된 기능으로 하며, 주관기관에서 제시한 편의 기능, 확장기능 등의 추가 요구사항을 포함하고 있습니다. 개발 환경은 CentOS 리눅스 운영체제이며, GUI 구현 및 주관기관의 라이브러리와의 호환성을 위하여 사용언어는 Qt C++을 사용하였습니다. 저는 기능 요구사항 1차년도 32개 중 10개, 2차년도 29개 중 7개 총 28%의 참여율을 가졌습니다. 1차년도 참여 기간은 2018-10-22~2018-11-30 약 1개월이며, 2차년도 참여 기간은 2019-07-09~2019-10-30 약 3개월입니다.

## Bluetooth Firmware Development
(주) 미로 | 2017 - 2017

C, TI BLE(Bluetooth Low Energy) Stack

본 프로젝트는 스마트폰 앱을 이용한 가습기 제어를 위한 IoT 시스템을 구현하는 것이 목표입니다. 소비자는 Wifi/블루투스 확장 모듈 중 어느 것을 선택하더라도 가습기의 슬롯에 이들을 장착하여 제어 명령을 가습기에 전달할 수 있습니다. (주) 미로는 2016년 12월 IoT 모듈 탑재 모델 출시 이후 매출이 증가하였고, 2018년 매출액은 약 207억으로 전년 대비 약 60% 증가하였습니다. 소프트웨어 개발 업무는 안드로이드 앱, Wifi 모듈 펌웨어, 블루투스 모듈 펌웨어가 있으며, 저는 그 중 블루투스 펌웨어 개발을 담당하였습니다.

# Projects

## 항공관제용 통합 정보처리 시스템 개발

2011 - 2014

Qt C++, C/C++, Linux Network

본 프로젝트의 목표는 매년 증가하는 국내외 항공교통량에 대응하여 더 안전한 항행을 가능하게 하는 최신 항공 관제시스템 관련 기술을 국내 연구진에 의해 개발하여 확보하는 것입니다. 이는 외산 장비 도입 비용 절감, 자율성 확보, 운용 유지비 절감 등의 기대 효과를 가지고 있습니다. 약 7년의 프로젝트 수행 기간 동안 주관 포함 12개 기관이 총 10개의 서브 시스템으로 구성된 항공관제 통합시스템 개발에 참여하였습니다. 저는 이 중 관제석 콘솔 개발에 3년 3개월 참여하였습니다.

## 그린 생체모방형 임플란티블 전자칩 및 U-생체정보처리 플랫폼 융합기술개발

2016 - 2019

C/C++

본 프로젝트는 체내 삽입형 의료기기의 핵심소자인 임플란터블 IC시스템, 플랫폼 구축을 통하여 생체내의 뉴런·미세조직 활동에 대한 대규모 채널 생체신호를 측정하는 시스템 개발을 목표로 합니다. 약 3년의 프로젝트 수행 기간 동안 무선 인체 통신 MAC 스케줄러 개발 및 최적화 기술을 연구하였습니다.

## 온라인 게임 QA용 무선 네트워크 환경 특성 연구

2011 - 2012

MFC C++

본 프로젝트는 노드 이동성, 긴 전송 지연, 높은 에러율 등 모바일 온라인 게임 네트워크가 가진 특성에 맞추어 처리 지연, QoS(Quality of Service) 감소, 트래픽 과다 등의 무선 네트워크 환경 장애 요인들을 효과적으로 해결하기 위한 연구를 목표로 합니다. 연구 내용은 WiFi/3G망에서의 온라인 게임 환경 특성 연구, 측정 기술 연구, 특성 모델링 연구, 에뮬레이션 및 시뮬레이션에 적용하는 연구를 포함하고 있습니다. 저는 이 중 측정 기술 연구에 참여하였습니다.

# Presentations

## Open Infrastructure Community Day Korea 2020

2020 - 2020

Docker, Kubernetes

제가 발표한 [오픈소스 Jitsi Meet와 함께 하는 오토스케일이 가능한 화상 회의 인프라 여정](https://youtu.be/fvks2fNsTuc)은 오픈소스 화상 회의 플랫폼 jitsi에 오토스케일을 적용하는 데 고려해야 할 점과 해결 과제들을 함께 공유하는 것을 목표로 합니다. 특히 스케일이 가능한 jitsi 구성 요소인 video bridge의 개수를 어떻게 관리할 것인지가 주요 과제입니다.

# Publications

## PMOP: Efficient Per-Page Most-Offset Prefetcher

2019

IEICE Transactions on Information and Systems, Volume E102.D, Issue 7, Pages 1271-1279, 2019

