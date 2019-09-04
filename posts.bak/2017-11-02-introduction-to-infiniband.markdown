---
layout: post
title:  "Introduction to InfiniBand for End Users 요약"
date:   2017-11-02 11:26:05 +0900
comments: true
categories: tech
---
읽고 요약중입니다. 문맥의 흐름이 원활하지 않습니다.
http://www.mellanox.com/pdf/whitepapers/Intro_to_IB_for_End_Users.pdf

### Chapter 1 - Basic Concepts

애플리케이션이 일반적으로 네트워크를 사용하기 위해서는 OS에 의존 할 수 밖에 없는데, 이는 네트워크 자원이 OS관리 하에 있기 때문이다. 반면에 Infiniband는 애플리케이션에게 메시징 서비스를 직접 제공 하기에, 기존처럼 OS 영역에서의 복잡한 네트워크 처리과정을 거치지 않아도 된다.

Infiniband는 서로 떨어진 애플리케이션의 가상 주소 영역간의 직접 연결을 제공하는데 이를 Channel이라고 부른다.

![Figure 1](/assets/images/ibintro_fig1.jpg)
<Figure 1 OS 관여없이 가상 주소 영역간의 직접 접근이 가능해진다>

이 Channel의 말단을 QP(Queue Pair)라 부르며 각 QP는 Send Queue와 Receive Queue로 구성된다. 애플리케이션에서 이 QP를 직접 가상 주소 영역에 올려 사용함으로써, 서로 다른 쪽의 애플리케이션의 가상 주소 영역에 직접 접근이 가능한데 이를 Channel I/O라 부른다.

Infiniband는 2가지 전송 방법(Semantic)을 제공하는데, 1. SEND/RECEIVE라고도 불리는 Channel Semantic과 2. RDMA READ/WRITE라고도 불리는 Memory Semantic이다.
* Channel Semantic의 경우 - 송신 측은 수신측의 버퍼나 구조체를 볼 수 없고, 메시지를 받는 쪽에서 자신의 구조체를 Receive Queue에 올려 메시지를 저장한다 - 즉 단순히 메시지를 보내고 받는다.
* Memory Semantic의 경우 - 송신 측이 RDMA READ나 RDMA WRITE를 사용하여 다른 쪽 애플리케이션의 가상 주소 영역에 읽기/쓰기를 수행한다.

InfiniBand Messaging Serivce 계층 중 하나인 Software Transport Interface(Figure 2 참조)가 QP를 포함하고 있으며, 또한 애플리케이션이 RDMA 메시징 서비스를 이용하기 위한 대부분의 방법이나 메커니즘들이 포함되어 있다.

![Figure 2](/assets/images/ibintro_fig2.jpg)
<Figure 2 InfiniBand Architecture는 애플리케이션에게 사용하기 쉬운 메시징 서비스를 제공한다. OSI 모형과 유사한 계층들을 포함하고 있다>

애플리케이션이 QP에 전송 요청하는 메시지를 Work Request(WR)라 하며, 이 메시지는 2**31 Bytes 내의 어느 사이즈도 가능하다.
전통적 TCP/IP에서는 메시지를 완성할때까지 구성하는 Byte를 스트리밍 받으면서 통신하는 반면, InfiniBand에서는 네트워크 대역폭을 탐지 후 최적화된 사이즈의 패킷들로 분리, 송수신 하여 메시지를 완성한다.

Verb은 Software Transport Interface에 정의되어 있는 액션, 즉 애플리케이션이 요청하는 액션을 의미한다. 추후 챕터에서 자세히 다룰 예정.

다음은 InfiniBand Architecture를 구성하는 하드웨어 요소들이다:
* HCA(Host Channel Adapter) - InfiniBand 네트워크의 말단으로서 서버 등에 설치되며, 주소 변환 메커니즘을 제공한다.
* TCA(Target Channel Adapter) - 임베디드 환경에서의 사용을 위한 특수한 버전의 Channel Adapter. 요즘 많이 쓰이진 않는다.
* Switch - 타 네트워크 스위치와 개념적으로는 유사하나, 패킷 드랍을 방지하는 기능 등 InfiniBand에 특수한 기능이 있다.
* Router -  많이 사용되지는 않으나, 큰 네트워크를 서브넷으로 쪼갤때 사용한다. 역으로 말하자면, 서브넷(참고로 InfiniBand Architecture의 기본 단위)을 아주 큰 사이즈의 네트워크로 확장 가능.
* Cable and Connector - 다양한 형태로 지원 된다. 물리적으로는 Copper or Fiber. Performance는 후술 혹은 링크(https://en.wikipedia.org/wiki/InfiniBand#Performance)

### Chapter 2 - InfiniBand for HPC
### Chapter 3 - InfiniBand for the Enterprise
### Chapter 4 - Designing with InfiniBand
### Chapter 5 - InfiniBand Architecture and Features
### Chapter 6 - Achieving an Interoperable Solution
### Chapter 7 - InfiniBand Performance Capabilities and Examples
### Chapter 8 - Into the Future

{% if page.comments %}
{% include disqus.html %}
{% endif %}
