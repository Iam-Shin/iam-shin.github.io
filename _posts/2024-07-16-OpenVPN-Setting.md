---
layout: post
title: "OpenVPN Setting"
date: 2024-07-19 12:00:00 +0900
categories: [infra]
tags: [openvpn, vpn]
---

## 외부 pc에서 vm에 직접 접속하기 위한 vpn 설치.
[서버설치] (https://itrooms.tistory.com/1040)  
[win client 설치] (https://dejavuqa.tistory.com/244)  
[linux client 설치] (https://blog.naver.com/ncloud24/221443379824)  

설치 dir   
[root@Mron-dn01 ~]# cd /etc/openvpn   
log dir  
[root@Mron-dn01 openvpn]# cd /var/log/openvpn  
conf 파일 위치
[root@Mron-dn01 openvpn]# ls /etc/openvpn/server.conf

작동확인.
```shell
[root@Mron-dn01 openvpn]# ps aux | grep openvpn
root      1811  0.0  0.0 112812   976 pts/1    S+   15:13   0:00 grep --color=auto openvpn
nobody   31147  0.0  0.0  77152  4112 ?        Ss   11:36   0:00 /usr/sbin/openvpn --status /run/openvpn-server/status-server.log --status-version 2 --suppress-timestamps --config /etc/openvpn/server.conf
[root@Mron-dn01 openvpn]# vi /run/openvpn-server/status-server.log  ---> 없다...
```

서버 설치과정 요약
```
IP address: AAA.BBB.CCC.DDD (외부 공인아이피를 입력합니다.)   ---> 175.126.58.203

What port do you want OpenVPN to listen to?    1) Default: 1194    2) Custom    3) Random [49152-65535]

Port choice [1-3]: 2 (디폴트 1194 포트 외에 다른 포트를 사용하셔도 됩니다.)  ---> 11949

UDP is faster. Unless it is not available, you shouldn't use TCP.    1) UDP    2) TCP Protocol [1-2]: 1

What DNS resolvers do you want to use with the VPN?
9) Google (Anycast: worldwide)
DNS [1-12]: 9

Enable compression? [y/n]: n

Customize encryption settings? [y/n]: n

Okay, that was all I needed. We are ready to setup your OpenVPN server now. 
You will be able to generate a client at the end of the installation. 
Press any key to continue...

Client name: dn01-vpn

Do you want to protect the configuration file with a password? (e.g. encrypt the private key with a password)    
1) Add a passwordless client    2) Use a password for the client 
Select an option [1-2]: 2 (1번을 선택하셔도 되지만 아무래도 비밀번호가 있는게 좋을것 같습니다.)


Do you want to protect the configuration file with a password?
(e.g. encrypt the private key with a password)
   1) Add a passwordless client
   2) Use a password for the client
Select an option [1-2]: 2
⚠️ You will be asked for the client password below ⚠️

* Using SSL: openssl OpenSSL 1.0.2k-fips  26 Jan 2017

* Using Easy-RSA configuration: /etc/openvpn/easy-rsa/vars

* The preferred location for 'vars' is within the PKI folder.
  To silence this message move your 'vars' file to your PKI
  or declare your 'vars' file with option: --vars=<FILE>
Generating a 256 bit EC private key
writing new private key to '/etc/openvpn/easy-rsa/pki/e2ea878e/temp.0d2121c1'
Enter PEM pass phrase: vbox123!
Verifying - Enter PEM pass phrase: vbox123!


-----

Notice
------
Keypair and certificate request completed. Your files are:
req: /etc/openvpn/easy-rsa/pki/reqs/dn01-vpn.req
key: /etc/openvpn/easy-rsa/pki/private/dn01-vpn.key
Using configuration from /etc/openvpn/easy-rsa/pki/e2ea878e/temp.b2fb8fae
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'dn01-vpn'
Certificate is to be certified until Sep 29 02:38:58 2026 GMT (825 days)

Write out database with 1 new entries
Data Base Updated

Notice
------
Certificate created at:
* /etc/openvpn/easy-rsa/pki/issued/dn01-vpn.crt

Notice
------
Inline file created:
* /etc/openvpn/easy-rsa/pki/inline/dn01-vpn.inline
Client dn01-vpn added.

The configuration file has been written to /root/dn01-vpn.ovpn.
Download the .ovpn file and import it in your OpenVPN client.
```

추가적으로 설정파일에 경로 확인후 수정필요 (꼭 확인할것!!)
```shell
# vi /etc/openvpn/server.conf

tls-crypt /etc/openvpn/tls-crypt.key
crl-verify /etc/openvpn/crl.pem
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server_TdzDNVDWkyKoDiFC.crt
key /etc/openvpn/server_TdzDNVDWkyKoDiFC.key
management localhost 7505                  <--- 추가. 접속 확인용 
```

ip 할당 테이블 파일 수정 ipp.txt
```shell
# vi /etc/openvpn/ipp.txt
dn01-vm1,10.8.0.2
dn01-vm2,10.8.0.3
dn01-vm3,10.8.0.4
dn01-vm4,10.8.0.5
```

server.conf  수정
```shell
# vi /etc/openvpn/server.conf 
ifconfig-pool-persist /etc/openvpn/ipp.txt   <-- 지정.
```

클라이언트 접속 확인할때는 
```shell
# telnet localhost 7505
status
```

시작
```
[root@Mron-dn01 log]# /usr/sbin/openvpn --status /run/openvpn-server/status-server.log --status-version 2 --suppress-timestamps --config /etc/openvpn/server.conf &
```

종료는
```
ps -ef | grep openvpn
kill -9 
```


프로세스 실행되었고, 포트 대기 확인되었지만, 접속 안됨. 
```shell
[root@Mron-dn01 log]# ps -ef | grep openvpn
nobody    2591  1125  0 16:03 pts/1    00:00:00 /usr/sbin/openvpn --status /run/openvpn-server/status-server.log --status-version 2 --suppress-timestamps --config /etc/openvpn/server.conf
root      2614  1125  0 16:04 pts/1    00:00:00 grep --color=auto openvpn
[root@Mron-dn01 ~]# netstat -plunt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:18649           0.0.0.0:*               LISTEN      26953/gmond
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1848/master
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      5031/sshd
tcp6       0      0 ::1:25                  :::*                    LISTEN      1848/master
tcp6       0      0 :::22                   :::*                    LISTEN      5031/sshd
udp        0      0 0.0.0.0:11949           0.0.0.0:*                           18454/openvpn    --->
udp        0      0 0.0.0.0:18649           0.0.0.0:*                           26953/gmond

```

11949 방화벽 열었음.
```shell
[root@Mron-dn01 openvpn-server]# firewall-cmd --add-port=11949/udp --permanent
success
[root@Mron-dn01 ~]# firewall-cmd --add-port=11949/tcp --permanent
success

[root@Mron-dn01 openvpn-server]# firewall-cmd --reload
success

```

nmap 으로 확인결과. 175.126.58.203 에 대해 udp 포트 11949가 열려있지 않으므로, 외부에서 접속 불가능함.  
---> 11949포트에 대한 udp 프로토콜 방화벽 해제를 요청해야함.(그냥 tcp로 하기로 결정. 밑에 기술)
```shell
[vboxadm@Mron-dn01 ~]$ nmap 175.126.58.203
Starting Nmap 6.40 ( http://nmap.org ) at 2024-06-27 16:24 KST
Nmap scan report for Mron-DN01 (175.126.58.203)
Host is up (0.0012s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
Nmap done: 1 IP address (1 host up) scanned in 0.12 seconds
[vboxadm@Mron-dn01 ~]$
```



### 윈도우 클라이언트 설치시
Mron-dn01 서버의 /root/dn01-vpn.ovpn 파일을 다운로드 하고.
윈도우용 openvpn 클라이언트 프로그램이 모두 설치되면.
바탕화면 오른쪽 하단에서 openvpn 아이콘 마우스 오른쪽 클릭하여.
import메뉴에 들어가서, 이 파일을 선택해 줘야 한다. 




### dn01-vm1에 클라이언트 설치

```shell
# yum list installed | grep <해당하는 패키지명>

# yum install openvpn

sftp
sftp root@192.168.56.1  <-- /root/dn01-vpn.ovpn 복사하기 위해
ls
cp /root/dn01-vpn.ovpn /etc/openvpn
cd /etc/openvpn
ls
openvpn --config /etc/openvpn/dn01-vpn.ovpn    <--- 이렇게 실행해야 비번 넣을수 있다. 
touch openvpn-client-start.sh
vi openvpn-client-start.sh    <--- openvpn --config /etc/openvpn/dn01-vpn.ovpn --daemon     --> 이렇게 했더니 비번 입력을 못해서 접속 안되네
ps -ef | grep openvpn-client-start.sh
ps -ef | grep open
```


vpn 연결된 후 10.8.0.2 아이피 있음. 
```shell
[root@dn01-vm1 ~]# ifconfig
enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.3.11  netmask 255.255.255.0  broadcast 10.0.3.255
        inet6 fe80::a00:27ff:fe9a:a86b  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:9a:a8:6b  txqueuelen 1000  (Ethernet)
        RX packets 87307  bytes 117804474 (112.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 21705  bytes 1566972 (1.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.102  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::e3ff:e015:f74f:75ae  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:68:24:96  txqueuelen 1000  (Ethernet)
        RX packets 14536  bytes 2599780 (2.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 12287  bytes 3870390 (3.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 55  bytes 4768 (4.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 55  bytes 4768 (4.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.8.0.2  netmask 255.255.255.0  destination 10.8.0.2       <----- openvpn 연결된후 
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 100  (UNSPEC)
        RX packets 4  bytes 406 (406.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4  bytes 294 (294.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### udp 에서 tcp 로 설정 변경
그런데, icbig 확인해보니. 다 tcp 로 통신하더라. 

그래서 udp -> tcp로 변경처리.

1. dn01-vpn.ovpn 파일 변경 (server, client 모두 /root/ 에 dn01-vpn.ovpn 파일을 넣었음.)
```
# vi /root/dn01-vpn.ovpn
proto udp
-->
proto tcp
```
이 파일을 각 클라이언트에 재배포 하던가, 
클라이언트에 들어가서 이 파일을 수정 하던가..


2. server.conf 파일 설정 변경.
```
# vi /etc/openvpn/server.conf
proto udp
-->
proto tcp
```

3. openvpn server 재시작.


### 여기서 부터는 dn01-vm1에 ansible이 모든 호스트에 접속가능해짐. 
dn01-vm1 에서 ansible로 작업함.
/root/dn01-vpn.ovpn 파일은 sftp로 다 복사해 놓음. 


### ansible로 vm 3대에 openvpn 설치. (결과적으로는 ansible로 설치 안됨.)
1. openvpn 설치
   
```shell
ansible -i /etc/ansible/hosts vm-all -m shell -a 'ls /root/dn01-vpn.ovpn'     --> 다 있음.
ansible -i /etc/ansible/hosts vm-all -m shell -a 'yum list installed | grep -Hi openvpn'    ---> vm1만 설치되있음.
ansible -i /etc/ansible/hosts vm-234 -m shell -a 'yum install -y epel-release'
ansible -i /etc/ansible/hosts vm-234 -m shell -a 'yum install -y openvpn' 

ansible -i /etc/ansible/hosts vm-all -m shell -a 'ls /etc/openvpn' 
ansible -i /etc/ansible/hosts vm-234 -m shell -a 'cp /root/dn01-vpn.ovpn /etc/openvpn' 
ansible -i /etc/ansible/hosts vm-all -m shell -a 'ls /etc/openvpn/dn01-vpn.ovpn' 

```
openvpn 설치할때 오류 났었는데, epel-release 설치 안되있어서, repository 없어서 그랬던거임. epel-release 먼저 설치.
>> 오류 No package openvpn available.Error: Nothing to donon-zero return code

2. openvpn 실행
   
```shell
openvpn --config /etc/openvpn/dn01-vm1.ovpn   <--- 이렇게 실행해야 비번 넣을수 있다. 
touch openvpn-client-start.sh
vi openvpn-client-start.sh    <--- openvpn --config /etc/openvpn/dn01-vm1.ovpn --daemon     --> 이렇게 했더니 비번 입력을 못해서 접속 안되네
ps -ef | grep openvpn-client-start.sh
ps -ef | grep open
```
 
3. openvpn 종료
   
```
ps -ef | grep openvpn
kill -9 <pid>
``` 

 
### 각 vm에 부팅시 openvpn 자동실행되어 접속 되도록 파일생성, 설정변경. 

서버에 openvpn-plugin-auth-pam.so 모듈이 설치되어 있음. 

```
[root@Mron-dn01 openvpn]# ls /usr/lib64/openvpn/plugins/openvpn-plugin-auth-pam.so
/usr/lib64/openvpn/plugins/openvpn-plugin-auth-pam.so
```


client 에서 dn01-vm1.ovpn 파일 신규 생성

```shell
# vi /etc/openvpn/dn01-vm1.ovpn
client
dev tun
proto tcp    ---> tcp로 변경
remote 175.126.58.203 11949
resolv-retry infinite
nobind
persist-key
persist-tun

# 인증서 파일 위치    
ca /etc/openvpn/ca.crt   ---> 서버의 인증서를 복사해서 경로에 배치

# 사용자 이름과 비밀번호 파일 참조
auth-user-pass /etc/openvpn/auth.txt

cipher AES-256-CBC
verb 3
```

client에서 auth.txt 설정

```
# vi /etc/openvpn/auth.txt
vboxadm
123vbox
```

서버의 ca.crt 인증서 복사

```shell
[root@Mron-dn01 openvpn]# scp /etc/openvpn/ca.crt root@dn01-vm1:/etc/openvpn/ca.crt
root@dn01-vm1's password:
ca.crt                                                                                         100%  668   694.3KB/s   00:00
[root@Mron-dn01 openvpn]# scp /etc/openvpn/ca.crt root@dn01-vm2:/etc/openvpn/ca.crt
root@dn01-vm2's password:
ca.crt                                                                                         100%  668   698.2KB/s   00:00
[root@Mron-dn01 openvpn]# scp /etc/openvpn/ca.crt root@dn01-vm3:/etc/openvpn/ca.crt
root@dn01-vm3's password:
ca.crt                                                                                         100%  668   693.0KB/s   00:00
[root@Mron-dn01 openvpn]# scp /etc/openvpn/ca.crt root@dn01-vm4:/etc/openvpn/ca.crt
root@dn01-vm4's password:
ca.crt                                                                                         100%  668   981.3KB/s   00:00
[root@Mron-dn01 openvpn]#
```


client.ovpn 템플릿 파일 복사.

```shell
[root@Mron-dn01 openvpn]# scp /etc/openvpn/client-template.txt root@dn01-vm1:/etc/openvpn/dn01-vm1.ovpn
root@dn01-vm1's password:
client-template.txt                                                                            100%  490   567.6KB/s   00:00
[root@Mron-dn01 openvpn]# scp /etc/openvpn/client-template.txt root@dn01-vm2:/etc/openvpn/dn01-vm2.ovpn
root@dn01-vm2's password:
client-template.txt                                                                            100%  490   585.6KB/s   00:00
[root@Mron-dn01 openvpn]# scp /etc/openvpn/client-template.txt root@dn01-vm3:/etc/openvpn/dn01-vm3.ovpn
root@dn01-vm3's password:
client-template.txt                                                                            100%  490   463.9KB/s   00:00
[root@Mron-dn01 openvpn]# scp /etc/openvpn/client-template.txt root@dn01-vm4:/etc/openvpn/dn01-vm4.ovpn
root@dn01-vm4's password:
client-template.txt                                                                            100%  490   543.8KB/s   00:00
```

