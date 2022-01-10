---
layout: post
title:  "Ubuntu 20.04를 VirtualBox에 설치하기"
date:   2021-07-01 16:00:00 +0900
categories: Platform
---

# Introduction
오픈 소스 프로젝트(특히 Cloud Native)를 구동하기 위해서는 Linux 환경이 필요한 경우가 많다. MacOS에 직접 설치를 해도 되지만, 테스트하고 지우기를 반복해야하기에 환경을 별도로 구성하는 것이 좋다. 이 튜토리얼에서는 Windows 10, MacOS에서 Ubuntu를 VirtualBox에 설치하고 간단하게 Image를 만들어(Export) 필요할때 바로바로 추가 설치 없이 복사(Import)해서 사용하는 방법을 설명해보겠다. 

> Note. 이 문서는 Windows에 맞춰서 작성되었으나 VirtualBox의 구성, VM에서 구동되는 메뉴는 모두 같다. 그래서 MacOS, Linux 등 다른 OS에서도 바로 사용할 수 있다. 


# Requirements
VirtualBox는 Memory와 Storage를 사용한다. CPU처럼 공유하는게 아니라 실제로 점유를 한다. 따라서 Local 환경에 여유 공간이 있어야 한다. 설치하려는 시스템마다 다르기는 하지만, 메모리 16G정도여야 이것저것 돌려보기 편할 것이다. 8G는 웹브라우져와 개발툴들을 같이 사용하기에는 좀 버거울 수 있다. 하드디스크또한 한 VM당 2~30G 정도(Linux)는 계산하는것이 적당할 것 같다. 

> Note. 클러스터를 구성하려면 Open Source는 보통 홀수로 구성을 해야하니 Desktop/Laptop 환경에 VM을 몇개 구성할지를 잘 생각해서 환경(Host)을 구성해보자.


# VirtualBox 설치하기
VirtualBox는 Oracle에서 제공한다. Windows 10 Pro 이상에서는 Hyper-V라는 Virtual 환경(Host)을 제공하고 있으나 Hyper-V는 Windows계열에서 밖에 사용을 하지 못하여 일반적으로 VirtualBox를 많이 선택한다. 

## VirtualBox 다운로드/설치
[Oracle VirtualBox Download](https://www.virtualbox.org/wiki/Downloads)에서 환경에 맞는 VirtualBox를 다운로드를 받는다. 이 튜토리얼에서는 Windows에 설치할 것이디 VirtualBox x.x.x platform package -> Windows hosts 를 선택한다. VirtualBox가 설치 완료되면 확장팩(Extension Pack) 도 설치할 것이니 같이 받아 놓자.
 
> Note. 설치시 오류가 나오는 경우가 있다. Host의 보드에서 Virtual Environment/Machine 등 옵션이 꺼져 있는 경우가 있으니 제조사의 메뉴얼을 참조하여 BIOS에서 해당 옵션을 활성화  해주기 바란다. MacOS는 상관이 없었고 Windows계열은 근래(2021년) 나온 Desktop/Laptop에서는 Default로 활성화 되어 있는지 본 튜토리얼 작성시 Laptop(LG Gram)에서는 바로 설치가 되었다.  

![VirtualBox - download](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-oracle-site.png){: width=”500”}

## Extension 설치
위에서 VirtualBox 설치파일을 받을때 확장팩(Extension Pack)도 같이 받았을 것이다. VirtualBox의 설치가 완료되면 다운받은 파일을 더블클릭하면 VirtualBox에 자동으로 설치가 된다.  다운로드된 파일은 Oracle_VM_VirtualBox_Extension_Pack-6.1.30.vbox-extpack 와 같다. (파일내의 버전 부분만 틀릴 것이다.) 

이 확장팩은 Host의 자원을 공유할때 사용할 수 있다. 데이터 디렉토리등을 마운트 할때 사용하고 있다. 


# Ubuntu 설치하기
VirtualBox로 미리 만들어진 이미지를 다운로드 받아서 생성을 하는 방식도 있다. 하지만, 이 튜토리얼에서는 가장 최소상태로 설치를 하기 위해 Ubuntu ISO이미지를 가지고 설치를 진행해보겠다. 

## Ubuntu 20.04 Server 이미지 받기
Ubuntu 서버의 ISO 이미지 파일을 받아서 직접 설치할 것이다. Ubuntu 이미지는 [ubuntu download](https://ubuntu.com/download/server)에서 받을 수 있다. 

> Note. Ubuntu Desktop은 GUI 환경에서 구동되고 각종 GUI용 Software를 설치하기에 공간을 많이 차지한다. 특별한 경우가 아니면 Server를 최소로 설치하고 원하는 Package를 설치하여 사용하는 것을 권장한다. 

## Ubuntu 설치하기
이제 설치를 시작해보자. 화면 순서대로 VM을 생성하고, Ubuntu를 구동하여 설치를 진행하면 된다.

> Note. Ubuntu의 버전에 따라 화면이 구성과 Default 값등이 다를 수 있다. 그러니 아래 순서에 없는 부분은 화면의 내용을 잘 보고 선택하기를 바란다.

먼저 VirtualBox를 실행하고 “New”를 선택하면 아래와 같이 화면이 표시되고 Name, Linux, Ubuntu (64-bit)를 선택합니다. VM 파일이 관리될 디렉토리도 필요시에 변경합니다.

![VirtualBox - Create Virtual Machine](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-01.png){: width=”500”}

다음은 Memory Size를 결정한다. 이 설정은 설치 후에도 변경할 수 있다. 

![VirtualBox - Memory size](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-02.png){: width=”500”}

> Note. Memory는 구동되면 실제로 CPU와는 다르게 실제 메모리를 점유하기 Host의 실제 메모리의 여유를 확인하여 잘 설정해야한다. 

이제 Hard disk를 결정한다. 신규로 생성할 것이니 Default로 설정하자. 내용을 보면 Linux는 최소 10GB의 공간이 필요하다고 안내를 해주고 있다. 

![VirtualBox - Hard disk](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-03.png){: width=”500”}

Hard disk의 Type은 3가지를 선택할 수 있는데, 내용에도 표시된 것 처럼 다른 시스템에서도 사용할지에 따라 결정하면 된다. 우리는 Default로 설정하여 넘어가자.

![VirtualBox - hard disk file type](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-04.png){: width=”500”}

실제 파일이 위치할 공간을 확인한다. 디렉토리는 처음 시작할때 적용하였기 때문에 변경할 부분은 없을 것 이다. 여기서는 Hard disk의 크기를 지정한다. 한번 지정하면 변경할 수 없으니 Host의 하드디스크 공간과 설치/테스트할 Software의 Requirements를 고려야여 결정을 한다. 

![VirtualBox - Hard disk size](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-05.png){: width=”500”}

이제 기본적인 설정은 완료하였다. 생성된 Guest/VM가 VirtualBox Manager에 보일 것이다. 이제는 추가설정을 해보자. VM을 선택하고 Setting -> Storage을 선택하여 Ubuntu ISO 이미지를 추가하자. ISO 를 Storage에 CD-ROM처럼 등록을 한다. VM이 처음 구동할때 CD-ROM에 설치CD가 있는 것 처럼 인식하여 설치 프로세스를 시작한다. 

![VirtualBox - ISO](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-06.png){: width=”500”}

이제 VM을 시작을 해보자. 새로운 창이 나타나면서 잠시후에 Installer의 언어를 선택하는 화면이 표시된다. 원하는 언어를 선택한다. 

![Ubuntu - language](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-01.png){: width=”500”}

인스톨을 어떻게 진행할지 Menu가 나타난다. Default로 선택된 Install Ubuntu Server를 선택하여 Install을 시작한다. 

![Ubuntu - install menu](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-02.png){: width=”500”}

Server의 언어를 선택한다.

![Ubuntu - server language](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-03.png){: width=”500”}

설치시 Update를 할지를 선택한다. 우리는 빠르게 설치하기 위해 Continue without updating 을 선택한다. 커서의 이동은 화살표와 Tab을 이용하여 이동할 수 있다. 

![Ubuntu - update](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-04.png){: width=”500”}

Keyboard의 정보를 설정한다. 일반적으로 Default로 설정하면된다. 

![Ubuntu - keyboard configuration](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-05.png){: width=”500”}

Network의 설정을 할 수 있다. 혹시 VM Setting에서 Network를 추가하였거나 다른 Network 설정을 하고 싶으면 내용을 변경하면 된다. Network 설정은 구동후에도 /etc/netplan 에서도 변경이 가능하다. 

![Ubuntu - Network connections](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-06.png){: width=”500”}

Proxy를 설정한다. 방화벽내에서 설치할 경우 Ubuntu 의 Mirror 사이트에 접속을 못할 수 있는 경우에 설정을 하면된다. 일반적인 경우에는 설정할 필요가 없다.

![Ubuntu - Configure proxy](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-07.png){: width=”500”}

Ubuntu의 Mirror URL을 설정한다. 자동으로 근처의 Mirror 서버를 잡아준다.

![Ubuntu - Mirror address](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-08.png){: width=”500”}

Storage의 Partition 구성을 설정한다. 이번 설치에는 VM으로 간단하게 설치하는 것이니 Default로 설정된 것을 사용하자. 

![Ubuntu - Storage configuration](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-09.png){: width=”500”}

Storage의 설정은 중요하기 때문에 한번더 확인한다. Continue로 커서를 옮겨서 다음으로 진행을 하자.

![Ubuntu - Storage configuration confirm](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-10.png){: width=”500”}

서버의 기본 계정의 정보를 설정한다. Root 사용자는 직접 접근이 안되게 되어 있다. 이곳에서 생성되는 계정이 sudoer로 등록되어 관리할 수 있게 설정해준다. 서버의 이름과 서버에서 사용할 계정과 비밀번호를 입력한다.  

![Ubuntu - Profile setup](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-11.png){: width=”500”}

Server에 접근할때 일반적으로 SSH를 사용하기에 SSH를 설정을 한다. Install OpenSSH Server를 선택하여 설치가 되도록 한다. 물론 서버를 구동 후에 추가로 설치해도 상관 없다. 

![Ubuntu - SSH Setup](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-12.png){: width=”500”}

일반적으로 많이 사용하는 Software추가로 설치할지 선택을 할 수 있다. 우리는 보통 직접 설치하고 테스트를 하려는 것이니 선택하지 않고 넘어가겠다. 
![Ubuntu - Install anothers](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-13.png){: width=”500”}

Ubuntu Server 설치를 시작한다. 진행상황은 맨아래 커서의 움직으로 확인할 수 있다. 

![Ubuntu - Installing](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-14.png){: width=”500”}

Server 설치가 완료되었다. Reboot now를 선택하여 Server를 재시작한다. Reboot now를 누르면 경우에 따라 오류가 나오기도 하는데 Enter를 누르거나 반응이 없으며 해당 VM을 Power off 하고 다시 시작해도 상관은 없다. 설치가 완료되면 기존에 Storage에 연결한 ISO이미지는 unmount되는데 reboot후에 설치화면이 다시 나오면 종료후 Setting -> storage에서 ISO이미지를 제거하고 다시 시작을 하자.

![Ubuntu - Installed](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-15.png){: width=”500”}

Reboot가 진행되면 최초에 설정하는 정보들을 진행하느라 화면에 많은 정보가 지나간다. 정상적으로 완료가되면 login 화면이 나타난다. 이 화면이 나오면 설치가 정상적으로 완료된 것이다. 설치시 입력한 계정, 비밀번호로 login을 해보자. Network 설정을 추가하였는데 잘못될 경우 잠시 Network를 확인하느라 시간이 오래걸리는 경우가 있다. 이럴때는 기다리면 일단 부팅은 되니 부팅후에 Network 설정을 올바르게 변경을 하자. (netplan 참조)

![Ubuntu - Boot](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-ubuntu-install-16.png){: width=”500”}


참고로 VM 종료는 login후에 아래의 명령을 입력하면 해당 VM이 종료를 한다. Sudo 명령은 처음에 password를 물어보니 당황하지 말고 다시한번 Password를 입력하면 된다. 
```
$ sudo shutdown -h now
```

# VirtualBox의 Ubuntu를 백업(Export)/추가(Import)하기
기본으로 설치된 Ubuntu를 언제든 바로바로 추가하여 사용할수 있도록 이미지를 백업(Export)해보자. 이렇게 Export를 해두면 매번 Ubuntu를 설치하지 않고 Import만 해서 사용할수 있어서 여러번 설치하고 지우고를 반복할때 유용하게 사용할 수 있다.

VirtualBox Manager의 메뉴에서 File -> export  appliance를 선택한다. 그러면 아래와 같이 설치된 VM들의 리스트가 나오고 Export하고 싶은 VM을 선택하여 다음으로 넘어간다.

![VirtualBox - export vm](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-export-01.png){: width=”500”}

Export할 파일의 Format/Version과 위치를 선택한다. 

![VirtualBox - export vm - appliance settings](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-export-02.png){: width=”500”}

구성 정보를 확인하고 Export를 진행한다. 

![VirtualBox - export vm - system settings](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-export-03.png){: width=”500”}

Export의 진행은 아래와 같은 화면에서 진행상황을 확인할 수 있다. 

![VirtualBox - export vm - exporting appliance](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-export-04.png){: width=”200”}

Export가 완료되면 처음에 지정된 Export 디렉토리에 .ova 파일이 생성된 것을 확인할 수 있다. (Export시 맨처음 선택한 format에 따라 파일의 확장자는 다를 수 있다.) 


이제 생성된 ova를 이용하여 새로운 vm을 만들어보자. File -> import alliance를 선택한다. File에 OVA파일을 선택한다. 

![VirtualBox - import vm](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-import-01.png){: width=”500”}

그럼 Appliance의 정보를 확인할 수 있다. 이름, CPU수, Memory Size 등을 수정할 수 있다. 수정할 부분이 있으면 수정 후 다음을 진행한다. 

![VirtualBox - import vm - Appliance settings](/posts/assets/platform/virtualbox-ubuntu/images/virtualbox-import-02.png){: width=”500”}

Import가 완료되면 VirtualBox Manager에 새로 추가된 VM을 볼 수 있을 것이다. 이렇게 테스트시 필요한 VM을 한꺼번에 여러개 만들때 편리하게 이용할 수 있다. 


# Tips
VirtualBox가 만능이 아니다. Local(Desktop, Laptop)에서 돌아가도록 한 Software이기 때문에 서버에서 구동하는 VMWare나 다른 사용 Virtual Machine처럼 동작을 하지는 않는다. 그러니 사용을 하면서 안되는 부분은 공식 문서나 Issue문서를 확인하여 지원여부를 확인하기 바란다. (MacOS, Linux, Windows에서 조금씩 구동이 차이가 있는 것 같음.)

- Network의 Bridged Adapter는 Wifi를 선택하면 연결이 안된다. 문서를 찾아보면 Wifi의 Architecture상으로 Share가 안된다고 이야기하고 있다.

- MacOS에서의 경우 OS의 버전이 Upgrade되면서 종종 구동이 안되는 경우가 있다. OS가 Upgrade가 되면 바로 고쳐지지 않을 수 있으니 이점은 참조하기 바란다. VirtualBox 사용자가 많아서 해당 오류를 검색하면 고치는 방법/현재 상황등은 금방 검색이 가능할 것이다.
 
- VirtualBox는 Update가 많이 되는 편이다. 큰 버그가 없으면 매번 Update를 할 필요는 없으니 수정된 사항을 확인후에 자신에게 필요한 부분이 Update되었으면 진행하면 된다.


# Conclusions
VirtualBox를 이용하여 최소한의 테스트환경을 구성을 할 수 있다. 그러나 VirtualBox가 Host의 자원을 사용하기 때문에 개발을 같이 병행하기는 어렵고 Open Source Software를 설치/테스트를하고 바로 지워버리기에는 적당하다. 

좀 더 실제 환경과 비슷하게 구동하기 위해서는 Network 설정을 해야할 수도 있다. 본 튜토리얼은 가장 기본적인 구성을 한 것이니 각자의 상황에 맞는 부분을 검색하여 설정하기 바란다.


# References
[Oracle VirtualBox](https://www.virtualbox.org/)

[Install Ubuntu server](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview)

