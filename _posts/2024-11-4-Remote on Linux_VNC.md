---
layout: post
title: 리눅스에서의 원격지원-VNC
date: 2024-11-4
categories: [linux]
tags: [linux]
---

# 1. 들어가며
나는 노트북으로 작업하기보단 데스크탑으로 작업하는것을 선호한다. 노트북이 있음에도 모니터 크기나 키보드, 마우스가 불편한 점이 있어 학교 실습실의 데스크탑을 사용하곤 했다. 실습실 컴퓨터와 우리 집의 데스크탑사이에서 작업물을 공유하기위해 카톡을 쓰기도 하고 github에서 private용의 Repo를 쓰기도 했으나 전자의 경우 매번 프로젝트를 압축-네이밍-압축해제 하는 불편함, 후자의 경우 매번 github authenticate관련 설정의 불편함이 있었으며 공통적으로 서로 다른 컴퓨터 환경으로 인한 의존성, IDE 설치의 불편함이 존재했다.

그래서 실습실 데스크탑의 경우, 환경의 HW적 편리함이 좋았기에 **Chrome 원격데스크탑(=CRD)**과 같은 방법으로 github 인증이나 다른 컴퓨터환경으로 인한 불편함을 없앨 수 있었다.

| 항목      | 우리집 데스크탑     | 실습실 데스크탑         | 노트북|실습실 데스크탑 with Chrome remote desktop
| --------- | ---------- | -------------- |-|-|
| HW환경의 익숙함 | **100%**        | 80%     | 40%|70%|
| 작업물 공유방법       | **주 환경** | 카카오톡 | git|**유사 주 환경**|
| 전반적 편리함  | **100%**        | 40%        |60%|**90%**|
| 요약    | **완벽함**       | HW환경은 좋으나 SW적 환경이 나빠 간단한 코딩만 가능        |SW적 환경은 좋으나 HW환경이 불편함| SW적 환경은 완벽하고 HW환경이 우리집이 아니므로 약간 부족함




<p> <img src="https://github.com/user-attachments/assets/eca8bc11-d4cb-48a8-b485-76ba431b788e
" alt="이미지 없음" width="800" height="300" /> </p>
_Chrome Remote Desktop (=CRD) 사용 화면_



# 2. 발단
윈도우 환경에서 CRD는 잘 동작했으나 리눅스를 메인 OS로 삼고 사용하면서 문제가 발생하였다. 
우선 첫번째로, 리눅스를 사용하며 오픈소스 생태계에 관심을 가지며 오픈소스 개발자들의 의견에 관심을 가지게 되며 구글이나 마이크로소프트같이 사용자 몰래 정보를 캐가는 빅테크의 SW를 대체할만한 SW를 찾아보면서 VSCode를 VScodium으로 대체하고 Chrome을 FireFox로 대체하게 되었다. 

두번째로, Linux에서 CRD를 사용하는게 제법 복잡하다. 애초에 CRD를 접하게 된 계기가 윈도우 자체 기능으로 원격지원을 시도하던 중 너무 복잡하여 쉽고 간단한 방법을 찾다가 CRD를 사용하게 된 것이니만큼 중요한 이유였으며, 첫번째 이유로 인해 Chrome조차도 FireFox로 대체하였기때문에 자연스럽게 CRD를 배제하고 다른 방법을 찾게 되었다.

## 2-1 VNC (Virtual Network Computing)
리눅스 환경에서의 원격데스크탑을 위해 이것저것 찾아보던중 VNC를 알게되었다. 

VNC는 "원격 데스크톱 공유 시스템"으로 [RFB](https://en.wikipedia.org/wiki/RFB_(protocol)) 프로토콜을 기반으로 동작한다. 이 포스팅은 VNC의 사용법에 대해 초점을 두므로 자세한 설명은 삼가겠으나 RFB, 즉 "Remote Frame Buffer" 프로토콜은 GUI에 대한 원격 액세스를 가능하게 하는 프로토콜로 **프레임 버퍼** 수준에서 동작하므로 윈도우, macOS, X window system(리눅스의 GUI 프레임워크라고 생각하면 된다.) 을 가리지 않고 동일하게 동작한다. 굉장히 단순하게 모든 화면을 픽셀단위로 그대로 전달하기때문에 여러 GUI시스템에서도 동작하는것이다.


# 3. Remote Desktop on Linux Mint
그러면, 이제부터 내가 사용중인 설정을 공유하도록 하겠다. 준비사항은 다음과 같다.

1. Server : X11VNC 설치, 설정 / 공유기 포트포워딩 
2. Client : Remmina 설치 

## 3-1 Server Setting
서버에서는 [X11VNC](https://github.com/LibVNC/x11vnc)라는 프로그램을 설치하고 비밀번호와 편의를 위한 alias 등록 그리고 공유기 포트포워딩이 필요하다.



```bash
#vnc 설치
sudo apt install x11vnc

#vnc 암호 설정
x11vnc -storepasswd
```
위와같이 간단한 설정을 마치게 되면 아래 사진과 같이 ~/.vnc위치에 passwd와 xstartup이 생성된다. passwd는 암호화된 상태로 저장되며 xstartup에는 서버의 설정이 담겨있다.
<p> <img src="https://github.com/user-attachments/assets/49595afe-15dc-4003-9291-d0e151b09173
" alt="이미지 없음" width="1000" height="60" /> </p>


<p> <img src="https://github.com/user-attachments/assets/986af3b1-188b-46e1-91f1-449e14cba675
" alt="이미지 없음" width="1000" height="300" /> </p>

그리고 서버 실행과 관련하여 

```bash
#~/.bashrc
...
alias runremote='x11vnc -display :0 -rfbauth ~/.vnc/passwd -forever -loop'
...
```

와 같이 alias를 설정하여 x11vnc 서버를 열고있는데 의미하는 바는 아래와 같다. 

* -display :0  // x11vnc가 제어할 디스플레이, :0은 사용자가 로컬에서 보고있는 화면을 의미
* -rfbauth ~/.vnc/passwd //VNC연결시 사용할 비밀번호 지정
* -forever //클라이언트가 연결을 종료하더라도 VNC서버가 계속 실행되도록 설정
* -loop // VNC서버가 오류로 인해 중단되거나 클라이언트가 연결을 끊었을때 자동으로 서버를 재시작 하도록 설정

이제 공유기쪽 설정을 알아보겠다. 우선 공유기 설정을 해야하는 이유는 다음과 같다.


<p> <img src="https://github.com/user-attachments/assets/51826a20-95c1-4410-aeb5-72da2314563f
" alt="이미지 없음" width="1000" height="400" /> </p>

위 이미지처럼 네트워크 설정이 되어있는 경우, **Case1** 이라면 노트북(클라이언트)에서 서버(127.0.0.4)로 연결해야 한다면 127.0.0.4:5900 (5900포트는 RFB의 기본 포트)로 요청을 보내면 알아서 공유기가 라우팅을 해주지만, **Case2**의 경우 199.32.5.7:5900으로 요청을 보내야 하며, 이 경우 포트포워딩 설정이 되어있지 않다면 199.32.5.7:5900으로 가는 요청은 라우팅되지 못한다. 따라서 직접 공유기의 설정에서 포트포워딩을 해야 한다. 

어떻게 통신사 공유기 어드민 페이지에 접속하는가. 는 아무래도 포스팅의 범위를 벗어나는것 같고, 나의 경우 이런식으로 포트포워딩을 했다. 

<p> <img src="https://github.com/user-attachments/assets/5ea223df-2eaf-4df0-b0c4-96dd9fbee90d
" alt="이미지 없음" width="1000" height="400" /> </p>

## 3-2 Client Setting

클라이언트의 세팅은 다소 쉽다. 

우선 [Remmina](https://remmina.org/)라고하는 오픈소스 원격 데스크탑 클라이언트를 설치한다.

그리고 실행하여 아래와 같이 VNC를 선택하고 외부ip:5900포트를 입력하고 비밀번호를 입력하면...

<p> <img src="https://github.com/user-attachments/assets/83d5111d-14f1-4b75-8d8f-e4cc22c13125
" alt="이미지 없음" width="1000" height="400" /> </p>



<p> <img src="https://github.com/user-attachments/assets/d0aa96ac-e9d0-4873-b4be-eff741d53c15
" alt="이미지 없음" width="1000" height="400" /> </p>

위와같이 노트북에서 데스크탑의 화면을 불러올 수 있다.


# 4 결론
이러한 과정을 통해 노트북-데스크탑간의 화면공유가 가능하나, 몇가지 아쉬운점이 있다.

1. 화면 해상도 
 * 데스크탑은 32인치, 노트북은 13인치화면인데 화면공유를 하게되면 화면이 잘려나온다. 추가 설정을 통해 클라이언트에 맞게 화면을 설정할 수 있을것같은데 아직 찾아보지 못했다.

2. 연결이 불안정한 경우가 있다.
 * 카페나 학교같은곳에서 접속을 할 경우, 가끔 렉이걸려 화면의 변화가 클라이언트로 전송이 안되는 경우가 있다.

이러한 아쉬운점이 있으나 앞으로 좀 더 관련 설정과 정보를 찾아보며 문제점을 해결하고 싶다. 
이렇게 윈도우에서 사용하던 SW를 리눅스의 오픈소스기반 SW로 대체해 나가는 과정에서 여러 지식이 쌓이는것 같아 피곤할때도 있지만 기분이 좋다.

