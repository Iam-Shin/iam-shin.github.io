---
layout: post
title: "VirtualBox Setting I"
date: 2024-07-15 12:00:00 +0900
categories: [virtualbox, vm]
tags: [virtualbox, vm]
---



# VirtualBox 설치 및 VM host 클라우드 구성

----------------

## 가상화 설치전 준비

### 이걸 하는 이유

배정받은 4대의 머신이 있는데, 이들간의 네트웍이 열려있지 않음.  
방화벽 해제 요청을 하였으나, 작업이 지연됨.  

그래서 4대중 1대에 virtual box를 설치하고,  
그 안에서 클러스터를 구축하라고 지시 받음.   
황망하고, 황당하고, 착찹한 심정.  

### T-cloudbiz 호스트 구성

|자산 |호스트명   |모델명                    |SN     |OS           |계정정보            |공인IP           |스펙                      |비고  
|SKT |엠론팀 |Mron-NN01	Dell R630 |8273R42 |CentOS 7.9  |root /  |175.126.58.201 |16 cores, 128 gb,    1tb |
|SKT |엠론팀 |Mron-EN01	Dell R730 |5ZWSB22 |CentOS 7.9  |root /  |175.126.58.202 |48 cores,  64 gb,  215gb |
|SKT |엠론팀 |Mron-DN01	Dell R730 |FJSV032 |CentOS 7.9  |root /  |175.126.58.203 |32 cores, 128 gb,  215gb |virtualBox 설치 하겠음.6/12
|SKT |엠론팀 |Mron-DN04	Dell R730 |6VTGY42 |CentOS 7.9  |root /  |175.126.58.204 |32 cores, 128 gb,  215gb |엠론 설치하겠음.

* 4대 모두 Intel® Xeon® 프로세서 E5-2600 v4 제품군
* 4대 모두 CentOS Linux release 7.9.2009 (Core)
* VirtualBox 호스트 cpu 요구사항중, SSE2 스펙이 있는데, 
  Mron-DN01 호스트에 설치된 Intel(R) Xeon(R) CPU E5-2640 v3 @ 2.60GHz 이 SSE2 스펙을 지원하는지 근거 불명.
  [참조](https://ark.intel.com/content/www/kr/ko/ark/products/92984/intel-xeon-processor-e5-2640-v4-25m-cache-2-40-ghz.html)  
  ::  명령 세트 확장 Intel® AVX2

  > AVX2가 SSE2를 포함하거나, 반대로 SSE2가 AVX2를 포함하는 것은 아닙니다. 이들은 각각 독립적인 명령어 세트 확장으로, 서로 다른 기능과 성능을 제공합니다. 그러나, AVX2는 SSE2보다 더 많은 연산을 수행할 수 있으며, 이로 인해 일부 애플리케이션에서는 AVX2가 SSE2보다 더 높은 성능을 제공할 수 있습니다.  
  > [참고](https://community.intel.com/t5/Intel-ISA-Extensions/SSE2-to-AVX2-performance-question/td-p/1101418?profile.language=ko)

### VirtualBox 클러스터 구성후, 설치해야되는 프로그램 (순서 참고.)
 설치시 Bigtop 참고.   
* Hadoop (HDFS, Yarn, MR)
* Hive + MariaDB
* Tez
* Trino
* Impala
* Kafka
* Zookeeper  
* 각 restAPI 작동 여부 확인. 

### VirtualBox 클러스터 구성시 게스트 배치 계획.

* Host OS 사양 : CPU 32 cores, RAM 128 gb, HDD 215gb
* 4대의 Guest VM 생성 예정
* 각 VM당 cpu 6 Core, RAM 24 gb, HDD 20 gb 예정


### VirtualBox 설치

* yum 설치. 추가 패키지 설치 오류로 실패했음. 

```shell
# Users of Oracle Linux / RHEL can add ​the Oracle Linux repo file to /etc/yum.repos.d/.
$> wget https://download.virtualbox.org/virtualbox/rpm/el/virtualbox.repo
$> mv virtualbox.repo /etc/yum.repos.d/
$> yum repolist

$> yum search VirtualBox
$> yum install VirtualBox-7.0.x86_64 
```

* 설치파일 다운로드 설치

```shell
# download link : VirtualBox for Oracle Linux 7/Red Hat Enterprise Linux 7/CentOS 7
$> wget https://download.virtualbox.org/virtualbox/7.0.18/VirtualBox-7.0-7.0.18_162988_el7-1.x86_64.rpm
$> rpm -ivh VirtualBox-7.0-7.0.18_162988_el7-1.x86_64.rpm

# 호환성 의존성 무시 강제 설치 
$> rpm -ivh --force --nodeps VirtualBox-7.0-7.0.18_162988_el7-1.x86_64.rpm   
```

### VirtualBox 라이센스

[VirtualBox 공식 라이센스 안내](https://www.virtualbox.org/wiki/GPLv3) :: VirtutalBox는 GPL v3 정책 준수.  
[GPL v3 정책 가이드](https://www.gnu.org/licenses/quick-guide-gplv3.html)

### VirtualBox 확장팩  

[원문](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/intro-installing.html)  

* 가상 USB 2.0(EHCI) 장치. 섹션 3.11.1, “USB 설정”을 참조하십시오.  
* 가상 USB 3.0(xHCI) 장치. 섹션 3.11.1, “USB 설정”을 참조하십시오.  
* VirtualBox 원격 데스크톱 프로토콜(VRDP) 지원. 원격 디스플레이(VRDP 지원)를 참조하십시오.  



* 호스트 웹캠 패스스루. 웹캠 패스스루 를 참조하십시오.  
* 인텔 PXE 부팅 ROM.  
* Linux 호스트에서 PCI 패스스루를 실험적으로 지원합니다. PCI 패스스루를 참조하십시오.  
* AES 알고리즘을 사용한 디스크 이미지 암호화. 디스크 이미지 암호화를 참조하십시오.  
* 결론 : 확장팩 없어도 됨.  

## 설치 작업 진행

Mron-DN01 호스트에 VirtualBox 7 설치하기로 결정.   
사용자 생성 vboxadm  
비번 

### 사용자 생성

```shell
[root@mron-dn01 ~]# adduser vboxadm
[root@mron-dn01 ~]# passwd vboxadm
[root@mron-dn01 ~]# vi /etc/group  <-- 아래와 같이 권한 편집  
vboxadm:x:1000:vboxadm,root   <-- su root 허용
```

vboxadm 사용자 생성 했으나, 설치시 root 권한 요구
```shell
[vboxadm@mron-dn01 ~]$ yum install VirtualBox-7.0.x86_64
Loaded plugins: fastestmirror
You need to be root to perform this command.
```

vboxadm에게 sudo 권한 주기. __(vboxadm 보안 조심!!)__
```shell
[root@mron-dn01 vboxadm]# visudo  <-- sudoer 설정파일 편집
vboxadm ALL=(ALL)       ALL  <-- 라인 추가
```

### virtualbox 7 설치

vboxadm 계정으로 virtualbox 7 설치
```shell
[vboxadm@mron-dn01 ~]$ sudo yum install VirtualBox-7.0.x86_64
```

설치 log는 yum_install_virtualbox7.log 참조.

### VirtualBoxMng

VirtualBox 명령어 라인 인터페이스. 

Guide doc url : (https://www.virtualbox.org/manual/ch08.html#vboxmanage-intro)  
pdf download : (https://download.virtualbox.org/virtualbox/7.0.18/UserManual.pdf)

> Oracle VM VirtualBox command-line interface. 
> 
> __Synopsis__  
> VBoxManage [-V | --version] [--dump-build-type] [-q | --nologo]
> [--settingspw=password] [--settingspwfile=pw-file] [@response-file]
> [[help] subcommand]  
>
> __Description__  
> The VBoxManage command is the command-line interface (CLI) for the Oracle VM VirtualBox 
> software. The CLI supports all the features that are available with the Oracle VM VirtualBox 
> graphical user interface (GUI). In addition, you can use the VBoxManage command to manage 
> the features of the virtualization engine that cannot be managed by the GUI.  
> Each time you invoke the VBoxManage command, only one command is executed. Note that
> some VBoxManage subcommands invoke several subcommands.  
> Run the VBoxManage command from the command line of the host operating system (OS) to
> control Oracle VM VirtualBox software.  
> The VBoxManage command is stored in the following locations on the host system: 
>
> VBoxManage 명령은 Oracle VM VirtualBox 소프트웨어용 CLI(명령줄 인터페이스)입니다.  
> CLI는 Oracle VM VirtualBox GUI(그래픽 사용자 인터페이스)에서 사용할 수 있는 모든 기능을 지원합니다.  
> 또한 VBoxManage 명령을 사용하여 GUI에서 관리할 수 없는 가상화 엔진의 기능을 관리할 수 있습니다.  
> VBoxManage 명령을 호출할 때마다 하나의 명령만 실행됩니다. 일부 VBoxManage 부속 명령은 여러 부속 명령을 호출합니다.  
> 호스트 운영 체제(OS)의 명령줄에서 VBoxManage 명령을 실행하여 Oracle VM VirtualBox 소프트웨어를 제어합니다.  
> VBoxManage 명령은 호스트 시스템의 다음 위치에 저장됩니다:  
> 
> • Linux: /usr/bin/VBoxManage  
> • Mac OS X: /Applications/VirtualBox.app/Contents/MacOS/VBoxManage  
> • Oracle Solaris: /opt/VirtualBox/bin/VBoxManage  
> • Windows: C:\Program Files\Oracle\VirtualBox\VBoxManage.exe  
>
> In addition to managing virtual machines (VMs) with this CLI or the GUI, you can use the
> VBoxHeadless CLI to manage VMs remotely.  
> The VBoxManage command performs particular tasks by using subcommands, such as list,
> createvm, and startvm. See the associated information for each VBoxManage subcommand.  
> If required, specify the VM by its name or by its Universally Unique Identifier (UUID).  
> Use the VBoxManage list vms command to obtain information about all currently registered
> VMs, including the VM names and associated UUIDs.  
> Note that you must enclose the entire VM name in double quotes if it contains spaces. 
> 
> 이 CLI 또는 GUI를 사용하여 VM(가상 머신)을 관리하는 것 외에도 VBoxHeadless CLI를 사용하여 VM을 원격으로 관리할 수 있습니다.  
> VBoxManage 명령은 list, createvm 및 startvm과 같은 하위 명령을 사용하여 특정 작업을 수행합니다.  
> 각 VBoxManage 하위 명령에 대한 연관된 정보를 참조하십시오.  
> 필요한 경우 이름 또는 UUID(Universally Unique Identifier)로 VM을 지정합니다.  
> VBoxManage list vms 명령을 사용하여 VM 이름 및 연결된 UUID를 포함하여 현재 등록된 모든 VM에 대한 정보를 가져옵니다.  
> 공백이 포함된 경우 전체 VM 이름을 큰따옴표로 묶어야 합니다.  



그런데 갑자기 베이그런트가 등장함...

### 베이그런트 개념

CentOS 가상화 솔루션 Vagrant 공식 홈페이지
https://app.vagrantup.com/centos/boxes/7

Vagrant 참고 블로그
https://www.44bits.io/ko/post/vagrant-tutorial
https://cafe-jun12.tistory.com/33
  
베이그런트는 독립적으로 사용되는 도구가 아니며, 가상 머신을 생성하거나 조작하는 기능을 직접 제공하지는 않습니다. 
베이그런트에는 프로바이더라는 개념이 있어서 버추얼박스VirtualBox, VMWare, 도커Docker, Hyper-V와 같은 도구들을 가상 머신을 관리하는 도구로 조합해서 사용할 수 있습니다.*

일단 베이그런트는 무시하고 VirtualBox만으로 해 본다. 

## Guest OS 설치 

CentOS 이미지 다운로드 공식 홈페이지
https://wiki.centos.org/Download.html  
https://www.centos.org/download/  
http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-2009.iso  


### CentOS 다운로드

NetInstall 버전 다운로드
```shell
[vboxadm@mron-dn01 ~]$ pwd
/home/vboxadm
[vboxadm@mron-dn01 ~]$ wget https://mirror.kakao.com/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-NetInstall-2009.iso
Saving to: ‘CentOS-7-x86_64-NetInstall-2009.iso’

100%[=========================================================================>] 602,931,200  110MB/s   in 5.3s

2024-06-17 14:00:15 (108 MB/s) - ‘CentOS-7-x86_64-NetInstall-2009.iso’ saved [602931200/602931200]
```

Minimal 버전 다운로드
```shell
[vboxadm@mron-dn01 ~]$ wget https://mirror.kakao.com/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-Minimal-2009.iso
Saving to: ‘CentOS-7-x86_64-Minimal-2009.iso’

100%[=======================================================================>] 1,020,264,448  111MB/s   in 8.9s

2024-06-17 14:01:14 (109 MB/s) - ‘CentOS-7-x86_64-Minimal-2009.iso’ saved [1020264448/1020264448]
```

# 가상머신 설치 (CentOS-7 설치 디스크를 cli에서 로딩할수 없어서 실패한 방법)
### VirtualBox 클러스터 구성시 게스트 배치 계획.

* Host OS 사양 : CPU 32 cores, RAM 128 gb, HDD 215gb
* 4대의 Guest VM 생성 예정
* 각 VM당 cpu 6 Core, RAM 24 gb, HDD 20 gb 예정


설치 명령 (Sample 1.)  
[참고 : 3.2. Unattended Guest Installation](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/basic-unattended.html)
```shell
VM_NAME="dn01-vm1"

# 가상 머신 생성 - 기본폴더 지정 안했더니, "VirtualBox VMs" 이렇게 만드네.. ㅜㅜ
VBoxManage createvm --name $VM_NAME --ostype "RedHat7_64" --register
Virtual machine 'dn01-vm1' is created and registered.
UUID: 76b8a345-aafb-4d3d-aa6b-21657d8beca4
Settings file: '/home/vboxadm/VirtualBoxVMs/dn01-vm1/dn01-vm1.vbox'

# 생성된 가상머신 조회
VBoxManage list vms -l

# 가상머신 삭제
VBoxManage unregistervm dn01-vm1 --delete

# 가상 머신 생성 2회차. (--basefolder 옵션과 생성된 Settings file 경로 주목.)
[vboxadm@mron-dn01 dn01-vm1]$ VBoxManage createvm --name $VM_NAME --basefolder=./VirtualBoxVMs --ostype "RedHat7_64" --register
shell-init: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
Virtual machine 'dn01-vm1' is created and registered.
UUID: 31e9cdff-8abc-45cf-9fd6-309c84f4f8a5
Settings file: '/home/vboxadm/.config/VirtualBox/VirtualBoxVMs/dn01-vm1/dn01-vm1.vbox'

# 가상머신 삭제 2회차.
VBoxManage unregistervm dn01-vm1 --delete


# 가상 머신 생성 3회차. (--basefolder 옵션과 생성된 Settings file 경로 주목.) 
# 생성에 성공하면, ~/VirtualBoxVMs 와 ~/.config 폴더가 생성됨. 
[vboxadm@mron-dn01 ~]$ VBoxManage createvm --name $VM_NAME --basefolder=/home/vboxadm/VirtualBoxVMs --ostype "RedHat7_64" --register

Virtual machine 'dn01-vm1' is created and registered.
UUID: d97ea83e-5898-4050-8dfc-26adaa9927b0
Settings file: '/home/vboxadm/VirtualBoxVMs/dn01-vm1/dn01-vm1.vbox'

# 혹시 오류나면 ostype 변경. (7 없다고 투덜댐.)
[vboxadm@mron-dn01 ~]$ VBoxManage modifyvm $VM_NAME --ostype "RedHat_64"


# OK. 이대로 진행.

# 메모리와 네트워크 설정
[vboxadm@mron-dn01 ~]$ VBoxManage modifyvm $VM_NAME --memory 24576 --nic1 nat

# 드라이브 설정-----start
# 가상 하드 드라이브 생성
[vboxadm@mron-dn01 ~]$ VBoxManage createhd --filename /home/vboxadm/VirtualBoxVMs/$VM_NAME/$VM_NAME.vdi --size 40000
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Medium created. UUID: 62ac5db3-ce83-4bc0-804a-5fbfb8e00841   --> 밑에서 이건 삭제

# 가상 하드 드라이브 삭제
vboxmanage closemedium disk <uuid> --delete

# vm 설정 명령들
[vboxadm@mron-dn01 dn01-vm1]$ ls
dn01-vm1.vbox  dn01-vm1.vbox-prev  dn01-vm1.vdi
[vboxadm@mron-dn01 dn01-vm1]$ rm -f dn01-vm1.vdi
[vboxadm@mron-dn01 dn01-vm1]$ VBoxManage createhd --filename /home/vboxadm/VirtualBoxVMs/$VM_NAME/$VM_NAME.vdi --size 40000
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...NS_ERROR_INVALID_ARG
VBoxManage: error: Failed to create medium
VBoxManage: error: Cannot register the hard disk '/home/vboxadm/VirtualBoxVMs/dn01-vm1/dn01-vm1.vdi' {f5da0654-0c62-48ac-80c4-2c2b98636491} because a hard disk '/home/vboxadm/VirtualBoxVMs/dn01-vm1/dn01-vm1.vdi' with UUID {62ac5db3-ce83-4bc0-804a-5fbfb8e00841} already exists
VBoxManage: error: Details: code NS_ERROR_INVALID_ARG (0x80070057), component VirtualBoxWrap, interface IVirtualBox
VBoxManage: error: Context: "RTEXITCODE handleCreateMedium(HandlerArg*)" at line 630 of file VBoxManageDisk.cpp
[vboxadm@mron-dn01 dn01-vm1]$ vboxmanage closemedium disk 62ac5db3-ce83-4bc0-804a-5fbfb8e00841 --delete
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
[vboxadm@mron-dn01 dn01-vm1]$ VBoxManage createhd --filename /home/vboxadm/VirtualBoxVMs/$VM_NAME/$VM_NAME.vdi --size 40000
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Medium created. UUID: f4e6b400-6579-4781-8e26-afb0f0a8302a



# SATA 컨트롤러 추가
[vboxadm@mron-dn01 ~]$ VBoxManage storagectl $VM_NAME --name "SATA Controller" --add sata --controller IntelAHCI

# 가상 하드 드라이브 연결
[vboxadm@mron-dn01 ~]$ VBoxManage storageattach $VM_NAME --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium /home/vboxadm/VirtualBoxVMs/$VM_NAME/$VM_NAME.vdi

# IDE 컨트롤러 추가
[vboxadm@mron-dn01 ~]$ VBoxManage storagectl $VM_NAME --name "IDE Controller" --add ide

# ISO 이미지 연결
[vboxadm@mron-dn01 ~]$ VBoxManage storageattach $VM_NAME --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium /home/vboxadm/CentOS-7-x86_64-NetInstall-2009.iso
# 드라이브 설정-----end

# VBoxManage 무인 설치
[vboxadm@mron-dn01 ~]$ VBoxManage unattended install $VM_NAME --iso=/home/vboxadm/CentOS-7-x86_64-NetInstall-2009.iso --user=tbizbig --full-user-name=tbizbig --install-additions --time-zone=KST
# 오류 발생.
VBoxManage: info: Preparing unattended installation of RedHat7_64 in machine 'dn01-vm1' (d97ea83e-5898-4050-8dfc-26adaa9927b0).
VBoxManage: error: The supplied ISO file does not contain an OS currently supported for unattended installation
VBoxManage: error: Details: code NS_ERROR_FAILURE (0x80004005), component UnattendedWrap, interface IUnattended, callee nsISupports
VBoxManage: error: Context: "Prepare()" at line 2359 of file VBoxManageMisc.cpp
# 자동설치 기능을 지원하지 않는다는거네... 그냥 곧바로 startvm 해야하나 봄.

# 가상 머신 시작
VBoxManage startvm $VM_NAME --type headless

Waiting for VM "dn01-vm1" to power on...
VM "dn01-vm1" has been successfully started.
# 헐... 순식간에 성공이라는데...? 그다음에 뭐 하지? guest os 내부는 설치도 안했는데...

```

결국 이 방법은 실패.
CentOS의 install iso 를 VM 에 드라이브 형태로 인식시킨후, start 하더라도, 
Xwindow 없이 터미널 CLI 에서는 
설치 프로세스에 진입하기 위한 GUI 초기화면을 로딩하지 못한다. 

방법.  
1. GUI 환경이 제공되는 host OS 에서 VirtualBox를 설치하여 CentOS의 vm을 생성하고, export 하여
그 ova 파일을 t-cloudbiz 환경으로 copy 하여 import 하는 방법.  

2. 초기 설치 완료된 CentOS vm 의 ova 파일을 제공하는 repository 사이트에서, 다운로드 받아서,
t-cloudbiz 환경에서 import 하는 방법. (다음 단계는 이 방법으로 실행함.)

