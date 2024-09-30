---
layout: post
title: "VirtualBox Setting II"
date: 2024-07-15 12:00:00 +0900
categories: [virtualbox, vm]
tags: [virtualbox, vm]
---


# 가상머신 설치 (minimal로 install된 CentOS-7 가상머신 이미지를 import 하는 방법)

### 다운로드 받은 ova 파일 import  
다운로드 : https://sourceforge.net/projects/osimages/  
자세한 내용은 다음을 참조하십시오. https://en.wikipedia.org/wiki/Open_Virtualization_Format#Industry_support   
사용자 : root / Osimages123!  
ip주소 : 172.23.251.73
hostname : localhost.localdomain   -> 변경 dn01-vm1


```shell
[vboxadm@mron-dn01 ~]$ VBoxManage import /home/vboxadm/CentOS-7-x64.ova
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Interpreting /home/vboxadm/CentOS-7-x64.ova...
OK.
Disks:
  vmdisk1       20      1346502656      http://www.vmware.com/interfaces/specifications/vmdk.html#streamOptimized       CentOS-7-x64-disk1.vmdk 641146368       -1

Virtual system 0:
 0: Suggested OS type: "RedHat_64"
    (change with "--vsys 0 --ostype <type>"; use "list ostypes" to list all possible values)
 1: Suggested VM name "CentOS-7-x64"
    (change with "--vsys 0 --vmname <name>")
 2: Suggested VM group "/"
    (change with "--vsys 0 --group <group>")
 3: Suggested VM settings file name "/home/vboxadm/VirtualBox VMs/CentOS-7-x64/CentOS-7-x64.vbox"
    (change with "--vsys 0 --settingsfile <filename>")
 4: Suggested VM base folder "/home/vboxadm/VirtualBox VMs"
    (change with "--vsys 0 --basefolder <path>")
 5: Description "OS: CentOS 7 Core x64
Login: root
Password: Osimages123!"
    (change with "--vsys 0 --description <desc>")
 6: Number of CPUs: 1
    (change with "--vsys 0 --cpus <n>")
 7: Guest memory: 512 MB
    (change with "--vsys 0 --memory <MB>")
 8: Network adapter: orig VM Network, config 3, extra type=Bridged
 9: Floppy
    (disable with "--vsys 0 --unit 9 --ignore")
10: CD-ROM
    (disable with "--vsys 0 --unit 10 --ignore")
11: SCSI controller, type LsiLogic
    (change with "--vsys 0 --unit 11 --scsitype {BusLogic|LsiLogic}";
    disable with "--vsys 0 --unit 11 --ignore")
12: IDE controller, type PIIX4
    (disable with "--vsys 0 --unit 12 --ignore")
13: IDE controller, type PIIX4
    (disable with "--vsys 0 --unit 13 --ignore")
14: Hard disk image: source image=CentOS-7-x64-disk1.vmdk, target path=CentOS-7-x64-disk1.vmdk, controller=11;port=0
    (change target path with "--vsys 0 --unit 14 --disk path";
    change controller with "--vsys 0 --unit 14 --controller <index>";
    change controller port with "--vsys 0 --unit 14 --port <n>";
    disable with "--vsys 0 --unit 14 --ignore")
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Successfully imported the appliance.

[vboxadm@mron-dn01 ~]$
[vboxadm@mron-dn01 ~]$ VBoxManage list vms
"dn01-vm1" {d97ea83e-5898-4050-8dfc-26adaa9927b0}
"CentOS-7-x64" {fb934577-6c9e-4d5e-875f-06405d8346a4}
[vboxadm@mron-dn01 ~]$ ls
CentOS-7-x64.ova  CentOS-7-x86_64-Minimal-2009.iso  CentOS-7-x86_64-NetInstall-2009.iso  VirtualBoxVMs  VirtualBox VMs

[vboxadm@mron-dn01 ~]$ cd "VirtualBox VMs"
[vboxadm@mron-dn01 VirtualBox VMs]$ ls
CentOS-7-x64
[vboxadm@mron-dn01 VirtualBox VMs]$ cd CentOS-7-x64/
[vboxadm@mron-dn01 CentOS-7-x64]$ ls
CentOS-7-x64-disk1.vmdk  CentOS-7-x64.vbox  CentOS-7-x64.vbox-prev
[vboxadm@mron-dn01 CentOS-7-x64]$ cd ..
[vboxadm@mron-dn01 VirtualBox VMs]$ ls
CentOS-7-x64
[vboxadm@mron-dn01 VirtualBox VMs]$ cd ..

[vboxadm@mron-dn01 ~]$ VBoxManage startvm CentOS-7-x64
Waiting for VM "CentOS-7-x64" to power on...
VBoxManage: error: The virtual machine 'CentOS-7-x64' has terminated unexpectedly during startup with exit code 1 (0x1)
VBoxManage: error: Details: code NS_ERROR_FAILURE (0x80004005), component MachineWrap, interface IMachine
[vboxadm@mron-dn01 ~]$ VBoxManage startvm CentOS-7-x64 --type headless
Waiting for VM "CentOS-7-x64" to power on...
VM "CentOS-7-x64" has been successfully started.
[vboxadm@mron-dn01 ~]$

[vboxadm@mron-dn01 Logs]$ VBoxManage guestproperty get CentOS-7-x64 "/VirtualBox/GuestInfo/Net/0/V4/IP"
No value set!

```

### Windows11 호스트 os VirtualBox 설정 변경
Guest 와 통신할 네트웍 설정. 

파일 > 도구 > Network Manager > Host-only Networks 탭 선택
만들기+ 선택하여 추가. 
이름은 적당히. 
하단 어댑터 탭에서. 수동설정. ipv4 주소 입력. 
DHCP 서버 탭에서. Enable Serveer 선택 해제.


### Windows11 에서 게스트 os 설정 변경
Windows11의 GUI 환경에서, 다운받은 게스트 centos7을 접속하여
몇가지 설정을 바꾸고, 다시 export 하여, 
dn01 에서 import 하자. 

호스트명 변경 localdomain.localhost -> dn01-vm1
```shell
#CentOS 7
[root@localhost ~]# hostnamectl set-hostname dn01-vm1
[root@localhost ~]# yum install net-tools
```

호스트 전용 네트웍 설정
참고. https://blog.naver.com/tequini/220977723865
https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=tawoo0&logNo=221606425141
```shell
[root@dn01-vm1 ~]# cd /etc/sysconfig/network-scripts/
[root@dn01-vm1 ~]# cp ifcfg-ens160 ifcfg-enp0s8
[root@dn01-vm1 ~]# vi ifcfg-enp0s8
[root@dn01-vm1 ~]# systemctl restart network
[root@dn01-vm1 ~]# ifconfig
```


dn01에서 실행.
```shell
[root@mron-dn01 home]# VBoxManage hostonlyifs create

[root@mron-dn01 home]# VBoxManage list hostonlyifs -l
Name:            vboxnet0
GUID:            786f6276-656e-4074-8000-0a0027000000
DHCP:            Disabled
IPAddress:       192.168.56.1
NetworkMask:     255.255.255.0
IPV6Address:
IPV6NetworkMaskPrefixLength: 0
HardwareAddress: 0a:00:27:00:00:00
MediumType:      Ethernet
Wireless:        No
Status:          Down
VBoxNetworkName: HostInterfaceNetworking-vboxnet0

# CentOS-7-x64 상세 정보 확인. nic 여러개 나옴. 
[vboxadm@mron-dn01 home]$ VBoxManage showvminfo CentOS-7-x64
...
NIC 1:                       MAC: 0800274ADF79, Attachment: Bridged Interface 'eno1', Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
NIC 2:                       disabled
NIC 3:                       disabled
NIC 4:                       disabled
NIC 5:                       disabled
NIC 6:                       disabled
NIC 7:                       disabled
NIC 8:                       disabled
...

# nic 2 를 hostonly 로 설정. 
[root@mron-dn01 home]# VBoxManage modifyvm CentOS-7-x64 --nic2 hostonly

# CentOS-7-x64 상세 정보 확인. nic 2 연결. 
[vboxadm@mron-dn01 home]$ VBoxManage showvminfo CentOS-7-x64
NIC 2:                       MAC: 080027A345C9, Attachment: Host-only Interface '', Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none

# host의 hostonly 네트워크에 vm의 nic 2를 연결
[vboxadm@mron-dn01 home]$ VBoxManage modifyvm CentOS-7-x64 --hostonlyadapter2 eno2
VBoxManage: warning: Interface "eno2" is of type bridged

# 시작
[vboxadm@mron-dn01 home]$ VBoxManage startvm CentOS-7-x64 --type headless
Waiting for VM "CentOS-7-x64" to power on...
VM "CentOS-7-x64" has been successfully started.

[vboxadm@mron-dn01 home]$ VBoxManage list runningvms -l


#끄기 (상태 저장하고 끄기)
[vboxadm@mron-dn01 home]$  VBoxManage controlvm CentOS-7-x64 savestate
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%

#끄기 (저장 안하고 끄기)
[vboxadm@mron-dn01 home]$  VBoxManage controlvm CentOS-7-x64 poweroff
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%

#등록된 vm을 끈 다음에. 지우고자 할때 (이렇게 하면, 기본폴더 하위에 있는 vb관련 폴더와 파일 다 지워짐.)
VBoxManage unregistervm UUID1또는VMname --delete

```

네트워크에 사용중인 ip 확인
```shell
nmap -sP 192.168.1.0/24   --> 설치 필요.

#ip-chk.sh 작성
#!/bin/bash
for ip in $(seq 1 254); do
  ping -c 1 192.168.1.$ip | grep "64 bytes" & 
done
wait
```

CentOS-7-x64-v2  
root  
vbox123!  


빅데이터 기술에 나온 내용을 토대로, 재 설정. 

- 기존 설정내용 지우고, 호스트의 host-only 네트워크 설정 하는 방법. dhcp 서버 포함. 101~104.

```shell  
[vboxadm@mron-dn01 ~]$ VBoxManage hostonlyif create
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Interface 'vboxnet1' was successfully created
[vboxadm@mron-dn01 ~]$ VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0

[vboxadm@mron-dn01 ~]$ VBoxManage dhcpserver add --ifname vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0 --lowerip 192.168.56.101 --upperip 192.168.56.200 --enable
VBoxManage: error: DHCP server already exists 
[vboxadm@mron-dn01 ~]$ VBoxManage dhcpserver add --ifname vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0 --lowerip 192.168.56.101 --upperip 192.168.56.109 --enable
VBoxManage: error: DHCP server already exists
[vboxadm@mron-dn01 ~]$ VBoxManage dhcpserver remove --ifname vboxnet0

[vboxadm@mron-dn01 ~]$ VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0
[vboxadm@mron-dn01 ~]$ VBoxManage dhcpserver add --ifname vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0 --lowerip 192.168.56.101 --upperip 192.168.56.109 --enable
[vboxadm@mron-dn01 ~]$
```

- 이번에는 호스트의 NAT Network 를 설정하는 방법.  dhcp 포함. 10~14.

```shell  
[vboxadm@mron-dn01 ~]$ VBoxManage dhcpserver remove --netname DN01_NatNetwork
[vboxadm@mron-dn01 ~]$ VBoxManage natnetwork add --netname DN01_NatNetwork --network "10.0.2.0/24" --enable
[vboxadm@mron-dn01 ~]$ VBoxManage natnetwork modify --netname DN01_NatNetwork --dhcp on
[vboxadm@mron-dn01 ~]$ VBoxManage dhcpserver add --netname DN01_NatNetwork --ip 10.0.2.2 --netmask 255.255.255.0 --lowerip 10.0.2.11 --upperip 10.0.2.19 --enable
[vboxadm@mron-dn01 ~]$
```

- Nat Network 설정 정보 보기.   
```shell  
VBoxManage list natnets
# 또는	
VBoxManage natnetwork list --netname <network-name>
VBoxManage natnetwork list --netname DN01_NatNetwork
```

- host-only 네크워크 설정 정보 보기.  
```shell
VBoxManage list hostonlyifs
# 또는
VBoxManage showvminfo <interface-name>
VBoxManage showvminfo vboxnet0
```

- 설정 결과 보기. dhcp.  
```shell
[vboxadm@mron-dn01 ~]$ VBoxManage list dhcpservers
NetworkName:    HostInterfaceNetworking-vboxnet0
Dhcpd IP:       192.168.56.1
LowerIPAddress: 192.168.56.101
UpperIPAddress: 192.168.56.104
NetworkMask:    255.255.255.0
Enabled:        Yes
Global Configuration:
    minLeaseTime:     default
    defaultLeaseTime: default
    maxLeaseTime:     default
    Forced options:   None
    Suppressed opts.: None
        1/legacy: 255.255.255.0
Groups:               None
Individual Configs:   None

NetworkName:    DN01_NatNetwork
Dhcpd IP:       10.0.2.2
LowerIPAddress: 10.0.2.10
UpperIPAddress: 10.0.2.14
NetworkMask:    255.255.255.0
Enabled:        Yes
Global Configuration:
    minLeaseTime:     default
    defaultLeaseTime: default
    maxLeaseTime:     default
    Forced options:   None
    Suppressed opts.: None
        1/legacy: 255.255.255.0
Groups:               None
Individual Configs:   None
[vboxadm@mron-dn01 ~]$
```

ssh 네트워크가 자주 끊어지는 현상. 확인중.  

```shell
[root@mron-dn01 ~]# ps -ef | grep virtualbox
...
vboxadm  32200 32146  0 16:46 ?        00:00:00 /usr/lib/virtualbox/VBoxNetDHCP --comment HostInterfaceNetworking-vboxnet0 --config /home/vboxadm/.config/VirtualBox/HostInterfaceNetworking-vboxnet0-Dhcpd.config --log /home/vboxadm/.config/VirtualBox/HostInterfaceNetworking-vboxnet0-Dhcpd.log
...

# 설정 확인. 
vi /home/vboxadm/.config/VirtualBox/HostInterfaceNetworking-vboxnet0-Dhcpd.config

# 설정 안에서 lease 관련 내용 확인.

# 내용 확인.
vi /home/vboxadm/.config/VirtualBox/HostInterfaceNetworking-vboxnet0-Dhcpd.leases
<?xml version="1.0"?>
<Leases version="1.0">
  <Lease mac="08:00:27:7f:28:13" network="0.0.0.0" state="acked">
    <Address value="192.168.56.101"/>
    <Time issued="1718783211" expiration="600"/>
  </Lease>
</Leases>

# expiration="600" --> 600초. 10분?  너무 짧다. 

# dhcpserver 설정 지우고 다시 잡아야함. 30일=2,592,000초
VBoxManage dhcpserver add --ifname vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0 --lowerip 192.168.56.101 --upperip 192.168.56.104 --lease-time 3600 --enable
```


mron-dn01 의 hosts  
```shell
[root@mron-dn01 ~]# vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

175.126.58.201  Mron-NN01
175.126.58.202  Mron-EN01
175.126.58.203  Mron-DN01
175.126.58.204  Mron-DN04

192.168.56.102  dn01-vm1
192.168.56.103  dn01-vm2
192.168.56.104  dn01-vm3
192.168.56.105  dn01-vm4
```

vm 4대 생성하고, 이미지에 호스트명 맞춰놓음. (네트워크 ip 고정ip으로 변경필요.)  
```shell
[vboxadm@mron-dn01 ~]$ ssh root@dn01-vm1
[root@dn01-vm1 ~]# exit
logout
Connection to dn01-vm1 closed.
[vboxadm@mron-dn01 ~]$ ssh root@dn01-vm2
[root@dn01-vm2 ~]# exit
logout
Connection to dn01-vm2 closed.
[vboxadm@mron-dn01 ~]$ ssh root@dn01-vm3
[root@dn01-vm3 ~]# exit
logout
Connection to dn01-vm3 closed.
[vboxadm@mron-dn01 ~]$ ssh root@dn01-vm4
[root@dn01-vm4 ~]# exit
logout
Connection to dn01-vm4 closed.
[vboxadm@mron-dn01 ~]$
```

hostonlyifs 에 구성된 hostonly network의 dhcp 서버에 vm의 hostonly 연결 nic 카드 고정 ip를 설정한다.  
```shell
VBoxManage dhcpserver modify --ifname vboxnet0 --mac-address=080027682496 --fixed-address=192.168.56.102
VBoxManage dhcpserver modify --ifname vboxnet0 --mac-address=080027ac772d --fixed-address=192.168.56.103
VBoxManage dhcpserver modify --ifname vboxnet0 --mac-address=080027f73c55 --fixed-address=192.168.56.104
VBoxManage dhcpserver modify --ifname vboxnet0 --mac-address=08002758af28 --fixed-address=192.168.56.105
```

VBoxManage controlvm dn01-vm1 reset

--------------------------------------------------------

1. NAT Network 생성  
먼저, NAT Network를 생성합니다. 예를 들어, NatNetwork1이라는 이름의 NAT Network를 생성하려면 다음 명령어를 사용합니다:  

```bash
VBoxManage natnetwork add --netname NatNetwork1 --network "10.0.3.0/24" --enable
VBoxManage natnetwork start --netname NatNetwork1
```
이 명령어는 NatNetwork1이라는 이름의 NAT Network를 10.0.3.0/24 네트워크 대역으로 생성하고 활성화합니다. (자동으로 DHCP 설정하고, dhcp 서버 만드는것 같다.)  

2. VM의 네트워크 어댑터 설정 변경  
이제 각 VM의 네트워크 어댑터를 NAT Network로 변경합니다. 예를 들어, VM1, VM2, VM3, VM4 네 대의 VM이 있다고 가정하겠습니다. 각 VM에 대해 다음 명령어를 실행합니다:  

```bash
VBoxManage modifyvm "dn01-vm1" --nic1 natnetwork --nat-network1 "NatNetwork1"
VBoxManage modifyvm "dn01-vm2" --nic1 natnetwork --nat-network1 "NatNetwork1"
VBoxManage modifyvm "dn01-vm3" --nic1 natnetwork --nat-network1 "NatNetwork1"
VBoxManage modifyvm "dn01-vm4" --nic1 natnetwork --nat-network1 "NatNetwork1"
```
여기서 --nic1은 첫 번째 네트워크 어댑터를 설정하는 옵션입니다. --nat-network1 옵션을 사용하여 첫 번째 네트워크 어댑터를 NatNetwork1으로 설정합니다.

----------------------------------------------------------

VBoxManage 7 버전에서는 --dhcp-option이 제거된 것으로 보입니다.  
대신, VBoxManage dhcpserver 명령어를 사용하여 DHCP 서버 설정을 변경할 수 있습니다. 이를 통해 특정 MAC 주소에 대해 고정 IP를 설정할 수 있습니다.  

NAT Network 설정 변경 방법  
NAT Network 생성:  
먼저, NAT Network를 생성합니다. 예를 들어, NatNetwork1이라는 이름의 NAT Network를 생성합니다.  

```bash
VBoxManage natnetwork add --netname NatNetwork1 --network "10.0.3.0/24" --enable
VBoxManage natnetwork start --netname NatNetwork1
```

DHCP 서버 설정:  
NAT Network의 DHCP 서버를 설정합니다.  
그런데, 아래 명령 안했는데, VBoxManage list dhcpservers 로 확인해보면 이미 dhcp가 설정되있다.  

```bash
VBoxManage dhcpserver add --netname NatNetwork1 --ip 10.0.3.3 --netmask 255.255.255.0 --lowerip 10.0.3.4 --upperip 10.0.3.254 --enable
```
여기서 10.0.3.3은 DHCP 서버의 IP 주소이고, 10.0.3.4부터 10.0.3.254까지의 범위는 DHCP가 할당할 IP 주소 범위입니다.

고정 IP 설정:
특정 VM의 MAC 주소에 대해 고정 IP를 설정합니다. 예를 들어, VM1의 MAC 주소가 0800279aa86b이고 고정 IP를 10.0.3.11으로 설정하려면 다음 명령어를 사용합니다.

```bash
VBoxManage dhcpserver modify --netname NatNetwork1 --mac-address=0800279aa86b --fixed-address=10.0.3.11
VBoxManage dhcpserver modify --netname NatNetwork1 --mac-address=0800273fd4c1 --fixed-address=10.0.3.12
VBoxManage dhcpserver modify --netname NatNetwork1 --mac-address=08002727fb06 --fixed-address=10.0.3.13
VBoxManage dhcpserver modify --netname NatNetwork1 --mac-address=080027f29968 --fixed-address=10.0.3.14
```
전체 설정 스크립트
다음은 NAT Network를 생성하고, DHCP 서버를 설정하고, VM에 고정 IP를 설정하는 전체 스크립트입니다.

```bash
#!/bin/bash

# NAT Network 생성
VBoxManage natnetwork add --netname NatNetwork1 --network "10.0.3.0/24" --enable
VBoxManage natnetwork start --netname NatNetwork1

# DHCP 서버 설정
VBoxManage dhcpserver add --netname NatNetwork1 --ip 10.0.3.3 --netmask 255.255.255.0 --lowerip 10.0.3.4 --upperip 10.0.3.254 --enable

# 각 VM의 MAC 주소와 고정 IP 설정
VBoxManage dhcpserver modify --netname NatNetwork1 --mac-address=0800279aa86b --fixed-address=10.0.3.11
VBoxManage dhcpserver modify --netname NatNetwork1 --mac-address=0800273fd4c1 --fixed-address=10.0.3.12
VBoxManage dhcpserver modify --netname NatNetwork1 --mac-address=08002727fb06 --fixed-address=10.0.3.13
VBoxManage dhcpserver modify --netname NatNetwork1 --mac-address=080027f29968 --fixed-address=10.0.3.14
```
위 스크립트에서 각 VM의 MAC 주소와 고정 IP 주소를 적절히 변경합니다.

이렇게 설정하면 NAT Network의 DHCP 서버가 특정 MAC 주소에 대해 고정 IP를 할당하도록 설정됩니다. 이 방법을 통해 VM들이 항상 동일한 IP 주소를 받게 됩니다. 추가적인 질문이나 다른 설정에 대해 문의사항이 있다면 말씀해 주세요.

---------------
다음 각 vm 내부에서 ip 설정.

[root@dn01-vm1 network-scripts]# cd /etc/sysconfig/network-scripts

만약 네트워크 인터페이스의 uuid를 알려면, 
```
[root@dn01-vm1 network-scripts]# nmcli connection show
NAME                UUID                                  TYPE            DEVICE
Wired connection 1  e636b5e7-4f10-3989-8f26-3c33b7e20664  802-3-ethernet  enp0s3
Wired connection 3  8d146403-ba63-3562-aae7-bddec7bf69d7  802-3-ethernet  enp0s8
Wired connection 2  cd1e3b49-b3ff-325d-aede-facc13a4e114  802-3-ethernet  --
ens160              daf3361b-aaae-4d00-a1da-be2ee7024a8d  802-3-ethernet  --
```


[root@dn01-vm1 network-scripts]# ls
ifcfg-Wired_connection_1 
생성하여 아래 내용 기입

```
HWADDR=08:00:27:9A:A8:6B   ###여기
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
PEERDNS=no
PEERROUTES=no
IPV4_FAILURE_FATAL=no
NAME="Wired connection 1"    ###여기
UUID=e636b5e7-4f10-3989-8f26-3c33b7e20664   ###여기
DEVICE=enp0s3   ###여기
ONBOOT=yes
IPADDR=10.0.3.11    ###여기
NETMASK=255.255.255.0
GATEWAY=10.0.3.1    ###여기
DNS1=8.8.8.8    ###여기
```

각 VM에 대해 고유한 MAC 주소와 고정 IP를 예약하고, 게스트 OS 내에서도 각각의 IP 주소를 고정 IP로 설정합니다. 예를 들어, VM2, VM3, VM4에 대해서도 동일한 방법으로 설정합니다.

네트워크 재시작
```
sudo systemctl restart network
```
또는
```
sudo ifdown enp0s3
sudo ifup enp0s3
```

---------------

현재 vbox 호스트의 설정 상태.
```
[vboxadm@mron-dn01 ~]$ VBoxManage list dhcpservers

NetworkName:    NatNetwork1
Dhcpd IP:       10.0.3.3
LowerIPAddress: 10.0.3.4
UpperIPAddress: 10.0.3.254
NetworkMask:    255.255.255.0
Enabled:        Yes
Global Configuration:
    minLeaseTime:     default
    defaultLeaseTime: default
    maxLeaseTime:     default
    Forced options:   None
    Suppressed opts.: None
        1/legacy: 255.255.255.0    #1 - Subnet Mask
        3/legacy: 10.0.3.1         #3 - Router (Default Gateway)
        6/legacy: 8.8.8.8          #6 - DNS Server
Groups:               None
Individual Configs:   None
[vboxadm@mron-dn01 ~]$
```

아래 표를 작성하기 위한 명령
```
[root@dn01-vm1 network-scripts]# nmcli connection show; ip addr;
```

|GUEST OS	|NAME                |UUID                                  |TYPE            |DEVICE	|IP				|MAC				|HOST-OS NET 
|dn01-vm1	|Wired connection 1  |e636b5e7-4f10-3989-8f26-3c33b7e20664  |802-3-ethernet  |enp0s3	|10.0.3.11		|08:00:27:9A:A8:6B	|NatNetwork1   
|dn01-vm1	|Wired connection 3  |8d146403-ba63-3562-aae7-bddec7bf69d7  |802-3-ethernet  |enp0s8	|192.168.56.102	|08:00:27:68:24:96	|vboxnet0      
|dn01-vm2	|Wired connection 1  |1e4ea849-8e20-3e95-8b2a-a14c9ab51338  |802-3-ethernet  |enp0s3	|10.0.3.12		|08:00:27:3f:d4:c1	|NatNetwork1   
|dn01-vm2	|Wired connection 3  |a8bdcfe7-ff9b-37a0-ab42-1798811f1304  |802-3-ethernet  |enp0s8	|192.168.56.103	|08:00:27:ac:77:2d	|vboxnet0      
|dn01-vm3	|Wired connection 1  |47f04164-face-36f5-977a-c987e911c9fe  |802-3-ethernet  |enp0s3	|10.0.3.13		|08:00:27:27:fb:06	|NatNetwork1   
|dn01-vm3	|Wired connection 3  |55787626-92b6-3135-a9b6-a355b02e2431  |802-3-ethernet  |enp0s8	|192.168.56.104	|08:00:27:f7:3c:55	|vboxnet0      
|dn01-vm4	|Wired connection 1  |a579cf1c-116b-3a99-add2-85d553ca56e4  |802-3-ethernet  |enp0s3    |10.0.3.14		|08:00:27:f2:99:68	|NatNetwork1   
|dn01-vm4	|Wired connection 3  |6d726677-1a84-3040-862f-f20a57dc1fd3  |802-3-ethernet  |enp0s8    |192.168.56.105	|08:00:27:58:af:28	|vboxnet0	   


모든 게스트에 시간 셋팅
```
$ timedatectl set-timezone Asia/Seoul
```

자신의 공인ip 알아내는법
```
curl ifconfig.me
```


## 외부 pc에서 vm에 직접 접속하기 위한 vpn 설치.
[서버설치] (https://itrooms.tistory.com/1040)  
[win client 설치] (https://dejavuqa.tistory.com/244)  
[linux client 설치] (https://blog.naver.com/ncloud24/221443379824)  

- 설치 dir   
[root@mron-dn01 ~]# cd /etc/openvpn   
- log dir  
[root@mron-dn01 openvpn]# cd /var/log/openvpn  
- conf 파일 위치
[root@mron-dn01 openvpn]# ls /etc/openvpn/server.conf

작동확인.
```
[root@mron-dn01 openvpn]# ps aux | grep openvpn
root      1811  0.0  0.0 112812   976 pts/1    S+   15:13   0:00 grep --color=auto openvpn
nobody   31147  0.0  0.0  77152  4112 ?        Ss   11:36   0:00 /usr/sbin/openvpn --status /run/openvpn-server/status-server.log --status-version 2 --suppress-timestamps --config /etc/openvpn/server.conf
[root@mron-dn01 openvpn]# vi /run/openvpn-server/status-server.log  ---> 없다...
```

설치과정. : openvpn_설치과정.md 참조.


### 각 vm에 java 설치.
```
# yum install -y java-11-openjdk.x86_64
```

### ssh 키 공유로 서버간 password 없이 로그인 하기 설치.
1. ssh 키 생성
```
[root@dn01-vm1 ~]# ssh-keygen -t rsa -b 4096 -C "root@dn01-vm1"
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):    ---> 엔터
Enter passphrase (empty for no passphrase):    ---> 엔터
Enter same passphrase again:         ---> 엔터
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
42:1a:6c:27:83:3e:a5:52:31:76:b3:2f:a6:7f:53:9a root@dn01-vm1
The key's randomart image is:
+--[ RSA 4096]----+
|  + o            |
...
|    .. .         |
+-----------------+

[root@dn01-vm1 ~]# ls -al /root/.ssh/
total 12
drwx------. 2 root root   57 Jul  1 09:23 .
dr-xr-x---. 5 root root  196 Jun 28 16:20 ..
-rw-------. 1 root root 3243 Jul  1 09:23 id_rsa
-rw-r--r--. 1 root root  739 Jul  1 09:23 id_rsa.pub
-rw-r--r--. 1 root root  714 Jun 28 16:50 known_hosts

```

2. ssh 공개키 배포
```
ssh-copy-id root@dn01-vm2   ---> 복사할 target remote 호스트
ssh-copy-id root@dn01-vm3
ssh-copy-id root@dn01-vm4

[root@dn01-vm1 ~]# ssh-copy-id root@dn01-vm2
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@dn01-vm2's password:         ---> 접속하기위해 비번 입력

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@dn01-vm2'"
and check to make sure that only the key(s) you wanted were added.
```

2-1. 위의 방법이 안될때
```
# 공개 키 파일의 내용을 클립보드에 복사합니다:
cat ~/.ssh/id_rsa.pub

# 원격 서버에 접속하여 ~/.ssh/authorized_keys 파일에 붙여넣습니다:
ssh username@remote_host
mkdir -p ~/.ssh
echo "your_copied_public_key" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

3. 모든 호스트에서 각자의 ssh키 생성하고, remote를 향해 ssh 공개키 배포작업 반복
```
# 각자의 ssh 키 생성
ssh-keygen -t rsa -b 4096 -C "root@dn01-vm2"
ssh-keygen -t rsa -b 4096 -C "root@dn01-vm3"
ssh-keygen -t rsa -b 4096 -C "root@dn01-vm4"

# ssh 공개키 배포 (4개 호스트에서 아래 4줄을 모두 실행)
ssh-copy-id root@dn01-vm1
ssh-copy-id root@dn01-vm2
ssh-copy-id root@dn01-vm3
ssh-copy-id root@dn01-vm4

# dn01-vm1의 경우, 자기 자신의 host에도 해 줘야 ansible 명령이 자신에게도 먹힘.
[root@dn01-vm1 ansible]# ssh-copy-id root@dn01-vm1
```

4. 접속 테스트
```
[root@dn01-vm3 ~]# ssh root@dn01-vm1
[root@dn01-vm1 ~]# 
```

### dn01-vm3에서 고정ip가 아닌 다른ip가 셋팅되는 경우 발생
dn01-vm3 내부에 들어가서 dhcp 캐시를 지워본다.
```
[root@dn01-vm3 ~]# sudo dhclient -r
[root@dn01-vm3 ~]# sudo dhclient
```
빠져나와서,   
해당 vm 죽였다가, 다시 살려봄.  
해결.


### dn01-vm1 에서 ansible로 각 호스트 접근 되는지 확인.
정상작동함.
```
[root@dn01-vm1 ansible]# ansible -i /etc/ansible/hosts vm-all -m shell -a 'ls /root/'
```
위 3번에서 dn01-vm1이 자신에게 공개키 배포를 안했더니, 여기서 에러났음. 
