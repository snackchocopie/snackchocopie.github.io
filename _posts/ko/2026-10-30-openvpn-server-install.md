---
title: "[초보자용] AWS Lightsail + OpenVPN으로 나만의 VPN 서버 구축하기"
lang: ko-KR
permalink: /ko/openvpn-server-install
author: ramen
date: 2026-10-30 16:50:00 +0900
categories: [OpenVPN]
tags: [vpn, linux, aws, lightsail]
canonical_url: https://qiita.com/snackchocopie/items/50f08dd766ca7c61439b
---

![logo.jpeg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/36bfbdf0-6c3c-4ff6-98d8-574dc2ec2e68.jpeg)

# AWS Lightsail로 OpenVPN 서버 구축해보기 (기초편)

이번 글에서는 간단하게 따라 할 수 있는 VPN 서비스 구축 과정을 시연해 보겠습니다.

사용할 서버는 AWS의 **Lightsail**이며, VPN 서버는 **OpenVPN**을 사용합니다.  
기초적인 시연이므로 세부적인 보안 설정 등은 이번 글에서 다루지 않습니다.

## Lightsail이란?

Lightsail은 AWS에서 제공하는 심플한 VPS(가상 프라이빗 서버) 서비스입니다.  
월 고정 요금으로 이용할 수 있으며, 플랜마다 일정량의 데이터 전송량이 포함되어 있습니다.  
(포함된 전송량을 초과하면 추가 요금이 발생합니다.)

![1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/3458e2c5-78d5-4f62-a64d-fd5bd8e4cf56.png)

*AWS Lightsail 메인 화면*

현재 이미 1대의 서버를 이용 중이지만, 이와 별도로 새로운 서버를 생성해 보겠습니다.


## 서버 생성하기
![2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/0072d828-6c80-4ec6-b15b-6a4938828b71.png)

*플랫폼은 Operating System, Blueprint는 Ubuntu 24 선택*

이미지와 같이 플랫폼은 **Operating System**을 클릭하고, Blueprint는 **Ubuntu 24**를 선택합니다.

Blueprint란 AWS 내에서 미리 준비된 템플릿 같은 것이라고 생각하시면 됩니다.


__리전을 선택할 수 있도록 빨간색으로 표시한 링크를 클릭합니다.__


![3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/65002cff-35bb-4c7e-99a8-82c74d0b172c.png)

여기서는 미국 리전 중에서 **Oregon** 리전을 선택합니다.

![4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/1f219bef-3b50-4495-8efe-bab0d54b1099.png)

네트워크 유형은 IPv4와 IPv6를 모두 사용할 수 있는 **Dual-Stack**을 선택합니다.

IPv4 주소가 필요하지 않다면 **IPv6-only**를 선택해도 무방합니다.

참고로 IPv4 주소란 우리가 평소 자주 보는 `123.123.123.123`처럼 4개의 숫자 블록으로 구분된 주소입니다.  
반면 IPv6는 그보다 훨씬 길고 복잡한 `2001:0db8:85a3:0000:0000:8a2e:0370:7334`와 같은 형식의 주소입니다.

![5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/5d4d836e-beb2-485b-8044-18e9c5755d88.png)

인스턴스(서버를 부르는 단위)의 이름은 식별하기 쉬운 고유한 이름을 붙여줍니다.

스냅샷은 만일의 장애에 대비한 백업 이미지입니다.  
필요에 따라 체크해 주세요. 이번에는 간단한 시연이므로 체크하지 않습니다.

**태그**는 AWS 서비스 내에서 생성한 서버를 식별하고 관리하기 위한 라벨과 같은 것입니다.  
이번에는 국가를 구분하는 목적으로만 사용할 것이므로 `country` 태그 키만 설정합니다.

![6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/a17fd11c-a56f-4032-b14b-1c08689636b8.png)

인스턴스가 생성되면 자동으로 퍼블릭 IPv4 주소와 IPv6 주소가 할당됩니다.  

또한 Amazon의 프라이빗 네트워크에서 사용되는 프라이빗 IPv4 주소도 1개 생성됩니다.  
(현시점에서는 '이런 것이 존재한다' 정도로만 이해해 두면 충분합니다.)

본 글은 단순 시연용이므로 공개해도 특별히 문제없지만, 실제 운영 환경에서는 외부에 공개하지 않는 것을 강력히 권장합니다.  
(DDoS 공격의 표적이 될 수 있기 때문입니다.)


## IPv4 주소 고정하기 (어태치)

IPv6 주소와 달리 IPv4 주소는 서버를 중지하고 시작할 때마다 바뀔 수 있습니다.  
이는 IPv4 주소가 고갈되어 가는 주소 체계이기 때문에 AWS 내에서도 체계적으로 관리되고 있기 때문입니다.

IPv4를 소유(고정)하려면 이 IP를 고정해야 합니다.  
Lightsail에서는 이를 **고정 IP(Static IP)**라고 부릅니다.

![7.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/53b4a03b-2154-4a1f-9174-72d5afe8e0e6.png)
![8.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/2507d6aa-bd06-4a27-a7e8-f0c9dc580b24.png)

*Networking 탭에서 Attach Static IP 선택*


**Attach Static IP**를 클릭하고 원하는 이름을 지정한 뒤 확인을 누릅니다.

## 네트워크 트래픽 허용하기

![9.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/82255c72-19af-4441-9e41-38f0aff1fa2d.png)

**AWS(Lightsail)**는 기본적으로 **80번(HTTP에서 사용하는 포트)**과 **22번(SSH 원격 접속)** TCP 포트 외에는 모든 포트를 차단합니다.

일반적으로 우리가 이용하는 인터넷 트래픽의 상당수는 UDP를 사용하며, 이번에 사용할 OpenVPN도 기본적으로 UDP를 통해 통신합니다(변경 가능).

Networking 탭 하단에 있는 Firewall 규칙 설정 화면에서 UDP 프로토콜과 OpenVPN이 통신할 UDP 포트를 개방합니다.

참고로 위 화면은 시연용이므로 전체를 개방했습니다.  
실제 서비스에서는 모든 포트를 개방하는 것을 피하고, Firewall 규칙은 최소 권한 원칙을 기본으로 엄격하게 운영해야 한다는 점을 기억해 두세요.

![10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/a2e025a8-0832-4bb0-be9b-4b2483108954.png)

OpenVPN에서 사용하는 기본 UDP 포트는 **1194**이므로 이를 허용합니다.

또한 허용할 IPv4 및 IPv6 소스 IP도 아래와 같이 모두 개방합니다.
 
*UDP 1194 포트와 모든 소스 IP 허용*

## 서버 원격 접속용 SSH 키 발급

![22.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/d3ba9e64-37f6-4392-ad48-4ac47e95ef7e.png)

위 이미지의 섹션처럼 먼저 **SSH 키**를 다운로드합니다.  
(키 관리는 철저히 해주세요!)

## 서버에 접속하여 OpenVPN 서버 설치하기

![11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/1e94c329-f934-41f7-ad82-aa46b65bcdac.png)

별도의 SSH 접속 프로그램을 설치하지 않고, 브라우저를 통해 AWS의 SSH 환경을 사용하여 서버에 접속합니다.

Connect 탭으로 돌아가 **Connect using SSH** 버튼을 눌러 서버에 접속합니다.



## 오픈소스 OpenVPN-Install을 이용한 간편 설치

본격적으로 OpenVPN 서버를 설치해 보겠습니다.

OpenVPN을 처음부터 수동으로 설치하려고 하면 복잡한 환경 설정 때문에 트러블슈팅에 많은 시간을 빼앗기게 됩니다.  
이를 애초에 피하기 위해 사용하기 편한 오픈소스를 활용해 봅시다.

이번에 사용할 오픈소스 프로젝트는 **angristan 님의 openvpn-install**입니다.  
https://github.com/angristan/openvpn-install

요즘은 훌륭한 오픈소스가 매우 많기 때문에, 목적에 맞는 솔루션이 있다면 적극적으로 오픈소스를 활용하는 편이 좋다고 생각합니다.

### 패키지 매니저 업데이트하기

```shell
$ sudo apt update
```
### 파일 다운로드 및 실행 권한 부여

```shell
$ curl -O https://raw.githubusercontent.com/angristan/openvpn-install/master/openvpn-install.sh
$ chmod +x openvpn-install.sh
```

`chmod +x` 명령어는 해당 파일에 실행 권한을 부여한다는 의미입니다.

리눅스에서는 `curl` 등으로 외부에서 다운로드한 파일은 보안상의 이유로 기본적으로 실행 권한이 빠져 있으므로 실행 권한을 부여해 줍시다.  
실행 권한이 없으면 실행할 수 없습니다.


## openvpn-install.sh 인터랙티브 실행
![14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/3651e8ea-477b-4d26-aa48-973886eb4288.png)

위 이미지와 같이 실행 권한(`x`)이 부여되면 파일 색상이 초록색으로 표시됩니다.

해당 파일을 아래 명령어로 실행합니다.

```shell
$ sudo ./openvpn-install.sh interactive
```

위 명령어를 실행하면 아래와 같이 대화형으로 설치가 진행됩니다.

순차적으로 표시되는 질문은 대부분의 경우 기본값 그대로 Enter 키를 눌러 진행해도 문제없습니다.  
저도 기본적으로 기본값으로 설정했습니다.

아래는 제가 입력한 응답 내용입니다.

```shell
What IP version should clients use to connect to this server?
> 1 (IPv4)

It seems this server is behind NAT. What is its public IPv4 address or hostname?
> 35.88.182.243

What IP versions should VPN clients use?
> 3 (Dual-stack (IPv4 + IPv6))

IPv4 VPN subnet:
> 1 (Default: 10.8.0.0/24)

IPv6 VPN subnet:
> 1 (Default: fd42:42:42:42::/112)

Route client internet traffic through the VPN? [y/n]
> y

Allow VPN clients to access each other? [y/n]
> n

Allow VPN clients to access the server's local network? [y/n]
> n

What port do you want OpenVPN to listen to?
> 1 (Default: 1194)

What protocol do you want OpenVPN to use?
> 1 (UDP)

What DNS resolvers do you want to use with the VPN?
> 3 (Cloudflare)

Allow multiple devices per client? [y/n]
> n

Do you want to customize the tunnel MTU?
> 1 (Default: 1500)

Choose the authentication mode:
> 1 (PKI)

Do you want to customize encryption settings? [y/n]
> n

Tell me a name for the client.
> tester

How many days should the client certificate be valid for?
> 3650

Do you want to protect the configuration file with a password?
> 2 (Use a password for the client)
```

참고로 **Route client internet traffic through the VPN?** 항목은 반드시 **Y**를 선택해 주세요.

**N**으로 설정하면 모든 인터넷 트래픽이 VPN 서버를 경유하지 않게 되어 타국 서비스 이용에 제한이 생길 수 있습니다.


## VPN 서버 접속을 위한 사전 준비

대화형 모드의 응답이 모두 끝나면 스크립트 내에서 PKI 생성 및 필요한 일련의 처리가 자동으로 진행됩니다.

최종적으로 아래 이미지와 같이 `.ovpn` 파일이 생성됩니다.  
이 파일이 OpenVPN 서버에 접속하기 위한 중요한 설정 파일입니다.

이 파일을 우리가 사용 중인 로컬 컴퓨터로 다운로드해야 합니다.  
이를 위해 **SCP**라는 프로토콜을 이용합니다. SCP는 TCP 22번 포트의 SSH 통신을 활용한 파일 전송 프로토콜입니다.

![20.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/147888bb-45b1-4bd8-99c8-61c9f37bd63f.png)
![21.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/02a4a558-983f-4a96-a216-6ac431d3674e.png)

```powershell
scp -i C:\Users\me\aws\sshkey.pem ubuntu@35.88.182.243:/home/ubuntu/tester.ovpn C:\Users\me\aws\tester.ovpn
```

**Windows 11**에서는 기본적으로 SSH 및 SCP가 설치되어 있습니다.  
만약 설치되어 있지 않다면 별도로 설치해 주세요.

위 명령어를 본인 PC의 **PowerShell** 또는 **명령 프롬프트(CMD)**에서 실행합니다.

물론 해당 예시의 **서버 IP**, **사용자 이름**, **키 이름**은 모두 제가 사용한 샘플이므로 본인의 정보를 입력해 주세요.


## OpenVPN 클라이언트로 접속 테스트하기

![24.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/55efb6ac-6eeb-45ff-ad3e-3f6f76b29809.png)

본격적으로 **OpenVPN 클라이언트**를 설치하여 접속을 테스트해 봅시다.

OpenVPN 클라이언트는 OpenVPN 공식에서 제공하는 클라이언트 애플리케이션입니다.  
다른 서드파티 앱이 아닌 이 공식 클라이언트를 사용하는 것을 권장합니다.

앱을 설치했다면 방금 다운로드한 `.ovpn` 파일을 임포트하여 접속해 봅시다.

![25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/652fa2d3-50c6-45ae-8076-d7132d36eb59.png)
![26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/2018b3f7-97b9-4204-b21b-c9d5b801c20f.png)

무사히 접속이 완료된 것을 확인할 수 있습니다.

또한 [mylocation.org](https://mylocation.org)에서 위치 정보를 확인해 본 결과, 정상적으로 Oregon 리전으로 표시되는 것을 확인할 수 있습니다.

