---
layout: post
title: "Ambari Bigtop Hadoop Install"
date: 2024-07-19 12:00:00 +0900
categories: [openvpn, vpn]
tags: [openvpn, vpn]
---

# bigtop + ambari 조합으로 hadoop ecosystem 관리.

### 0. 따라한다.
https://www.bearpooh.com/178

### 0-1. 일산서버 ISBIG 설치.
- mron-en01을 Bigtop + Ambari 빌드/배포/Master로 사용.
- ansible이 mron-dn04(mron master)에 있지만, ambari 빌드/배포 위해서 mron-en01에도 설치.
- git은 mron-en01만 설치.(소스 받아서 빌드하기 위함)

### 0-2. 버전.
- 리눅스 CentOS 7 필수.  다른 os버전은 호환성 안좋고 어렵단다.
- Bigtop 3.2.1 필수. (git branch 3.2) 그보다 상위버전은 ambari가 포함되지 않는다. git "branch 3.2" 최신 이므로 3.2.1 임.
- ambari-server 2.7.5.0-0
- java openJDK 1.8 
- python 2.7.5. 필수. 다른거 안되.
- maven 3.6.3이 아카이브에 없어서. 3.9.4로 대체. 별 상관은 없을것 같다.  --> 그런데, ambari-pkg 빌드 하다가 오류났다. 이 문제 같다.. maven 3.6.3 다시 설치해본다. 잘 넘어 갔다.
- maven 3.6.3 필수.
- Bigtop 3.2가 지원하는 하둡 ecosystem application 버전은 링크참조 : [컨플루언스](https://cwiki.apache.org/confluence/display/BIGTOP/Bigtop+3.2.1+Release)
- Bigtop 3.2 안에 Ambari 2.7.5 포함. 
- Bigtop 3.2.1 포함 버전.
```
bigtop 3.2.1 stack includes the following components
alluxio                        2.8.0
bigtop-ambari-mpack            2.7.5
bigtop-groovy                  2.5.4
bigtop-jsvc                    1.2.4
bigtop-utils                   3.1.0
flink                          1.15.3
gpdb                           5.28.5
hadoop                         3.3.5
hbase                          2.4.13
hive                           3.1.3
kafka                          2.8.1
livy                           0.7.1
oozie                          5.2.1
phoenix                        5.1.2
solr                           8.11.2
spark                          3.2.1
tez                            0.10.1
ycsb                           0.17.0
zeppelin                       0.10.1
zookeeper                      3.5.9
```

 
### 1. ambari 
- 호튼웍스가 클라우데라에 합병되어 무료 사용 불가  
- apache 에서는 명맥만 유지하는중.  
- 설치가 어렵고, repo 구성이 실패할수도 있다.  
- 해결법  
	- ambari 직접 빌드 및 설치  
	- apache bigtop 활용 - 이 방법 진행.  

### 2. centOs 7 가상환경 생성. (ISBIG은 가상환경 설정 skip!!)
- centos 7.9.2009 minimal (skip)  
- 권장 램 4G 이상. hdd 80~100G 이상. cpu 2core 이상  (skip)  
- NAT Network 설정 enp0s3, 호스트 전용 어댑터 enp0s8 설정.  (skip)  
- vm 호스트명, ntp, partition 설정(skip)  
- 계정설정. ambari 생성 (관리자지정. - 옵션확인)   
``` 
// t-doop 4개 호스트 모두 각각 생성. (root계정)
# adduser ambari
# passwd ambari         -- ambari123!
# usermod -aG wheel ambari
```
(wheel 그룹에 속한 사용자에게 sudo 권한 부여됨.)  

```
// t-doop 4개 호스트 모두 각각 생성. (root계정)
# adduser smartbig
# passwd smartbig         -- smartbig123!
```
(wheel 그룹에 속한 사용자에게 sudo 권한 부여됨.)  



### 3. centOs 7 기본설정 
- 사용자 계정 권한 변경 (sudo를 비번없이 실행하기위함.) 
- 하둡관련계정(=ambari) sudo 권한 설정.
```
// t-doop 4개 호스트 모두 각각 수정. (root계정)
# visudo
======================================================
## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL

# 아래의 코드 추가----------
# add t-doop
ambari  ALL=(ALL)       ALL
ambari-qa  ALL=(ALL)       ALL
mapred  ALL=(ALL)       ALL
hdfs    ALL=(ALL)       ALL
yarn    ALL=(ALL)       ALL
spark   ALL=(ALL)       ALL
hive    ALL=(ALL)       ALL
kafka   ALL=(ALL)       ALL
tez     ALL=(ALL)       ALL
trino   ALL=(ALL)       ALL
zookeeper       ALL=(ALL)       ALL

-----------------------
...
(중략)
...

## Allows people in group wheel to run all commands
%wheel  ALL=(ALL)       ALL

# 아래의 코드 주석 제거.---------
## Same thing without a password
%wheel  ALL=(ALL)       NOPASSWD: ALL
-----------------------

이후. ambari 계정으로 작업한다.
# su ambari
$
```

- wheel 그룹에 하둡관련계정(=ambari, hdfs, ... 등등 위에서 추가한 계정) 추가
```
// t-doop 4개 호스트 모두 각각 수정. (ambari계정)
$ sudo vi /etc/group
wheel:x:10:ambari,ambari-qa,mapred,hdfs,yarn,spark,hive,kafka,tez,trino,zookeeper

...

# 아래 내용 추가.----------------------
# hadoop 그룹을 추가하고, 그룹에 속할 계정 미리 추가.
hadoop:x:1002:ambari,ambari-qa,mapred,hdfs,yarn,spark,hive,kafka,tez,trino,zookeeper,hadmin,huser
```

- hdfs 일반계정 생성. - 관리자와 사용자 분리하여 각각 계정 생성.
```
// ambari@mron-en01 에서, ansible로 한번에 생성. (이하 ansible 명령은 ambari@mron-en01 계정으로 실행)
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo useradd hadmin'
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo useradd huser'

// 또는
// t-doop 4개 호스트 모두 각각 수정. (ambari계정)
$ sudo useradd hadmin
$ sudo useradd huser
```

- yum 기본설정 및 패키지 설치.
```
// 이건 ansible 안되넹...
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo yum update -y'
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo yum install epel-release -y'
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo yum install yum-utils'
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo yum-complete-transaction'

// 또는
// t-doop 4개 호스트 모두 각각 수정. (ambari계정)
$ sudo yum update                  # 최신으로 업데이트 해버림
$ sudo yum check-update            # 패키지 목록은 확인하고 최신상태로 유지하지만, 업데이트 하지 않고 차이점 보여줌. 
$ sudo yum install epel-release    # epel repo 활성화
$ sudo yum install yum-utils       # yum 확장도구 설치
$ sudo yum-complete-transaction    # 정상확인
```

- 네트워크 설정 변경 - 호스트전용 네트워크 고정ip화 (skip)  
- 호스트명 변경, 호스트 파일 변경 (skip)  
- ssh 설정 변경.- 암호화키는 가상머신 복제후. (skip)  
- 방화벽 설정변경
```
// t-doop 4개 호스트 모두 요청에 의해 신매니저님이 해제 해주심.
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo systemctl status firewalld'
```
- ntp 설정
```
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo yum -y install ntp'
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sudo systemctl enable ntpd;sudo systemctl enable chronyd;sudo systemctl start ntpd;sudo systemctl start chronyd;sudo systemctl status ntpd;timedatectl set-ntp y;timedatectl;'
```

- SELinux 설정 해제(접근제어) - 민감한건데 필수인것 같아서 고침.
호스트간 접근제어 보안 정책을 비활성화 시켜서 원활한 진행.
```
// t-doop 4개 호스트 모두 실행.

$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sestatus'   # 상태확인. 4개호스트 모두 활성화네.
mron-en01 | CHANGED | rc=0 >>
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31

$ sudo getenforce   # 상태 확인 Enforcing(활성.강제적용), permissions(활성이지만강제적용안됨. 로깅기록되지만차단안됨), disabled(비활성)
$ sudo setenforce 0    # 일시 비활성화.
$ sudo sed -i.bak 's/SELINUX=enforcing/#SELINUX=enforcing\nSELINUX=disabled\n/' /etc/selinux/config   # 영구 비활성화.
$ sudo cat /etc/selinux/config   # 확인.
$ sudo ansible -i /etc/ansible/hosts all -m shell -a 'sestatus'   # 확인.
# 또는 sudo getenforce

permissions   # 활성이지만강제아님 상태
# disabled는 재부팅 해야되는것 같고, permissions도 괜찮겠다...
```

- mask(권한박탈) - 민감한것 같아서 (skip)  
- 절전모드 해제, 시스템자원 제한 설정, locale 설정 - 민감한것 같아서 (skip)  


### 4. CentOs 추가설정
- 필수 yum 패키지 설치 (python은 반드시 2.7.5: 이 이상은 ambari 설치시 인식안됨.)
```
// Bigtop 소스코드 받아오기 위한 git (이건 ambari build 위함. en01에만 설치)
$ sudo yum install -y git

// t-doop 4개 호스트 모두 설치.
# centos yum 설치에 필요한 rpm 파일을 생성하기 위한 빌드 도구
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y rpm-build'
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y gcc'  
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y gcc-c++'

# 에러내용.
Transaction check error:
  file /usr/include/rpcsvc/nis.h from install of glibc-headers-2.17-326.el7_9.3.x86_64 conflicts with file from package libnsl-devel-1.2.0-5.1.x86_64
  ... 이하 여러건. 

# 에러나면, 아래와 같이 libnsl 지우고, gcc, gcc-c++ 재설치
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum remove libnsl-devel-1.2.0-5.1.x86_64'
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y gcc'
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y gcc-c++'


# Ambari의 동작에 필요한 파이썬 개발도구
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y python2'
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y python2-devel'
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y python-pip'

# Javascript 패키지 관리자 설치
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install -y npm'
```

- openJDK 1.8 설치
```
// t-doop 4개 호스트 모두 설치.

$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum update'
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'sudo yum install java-1.8.0-openjdk-devel.x86_64'

# 각 호스트에 들어가서 환경변수 수정
$ sudo vi /etc/bashrc
# 내용추가 
export JAVA_HOME=/usr/lib/jvm/java
export PATH=$JAVA_HOME:$PATH

$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'source /etc/bashrc'
$ sudo ansible -i /etc/ansible/hosts exp-en01 -m shell -a 'java -version'
```

- maven 설치, 설정 (일단, ambari master인 en01만 하자)  
 maven 3.6.3 찾기 까다로워. 3.9.4로 대체.(결국 지우고 3.6.3으로 설치)
```
// 호스트 4대. 다 설치 되있음.
$ sudo yum install wget

# 다운로드 
$ sudo wget https://www.apache.org/dist/maven/maven-3/3.9.4/binaries/apache-maven-3.9.4-bin.tar.gz -P /tmp

# 압축해제, 폴더이동, 링크생성
$ sudo tar xf /tmp/apache-maven-3.9.4-bin.tar.gz -C /opt
$ sudo ln -s /opt/apache-maven-3.9.4 /opt/maven


# 추가. ambari-pkg 빌드도중 오류발생하여, maven 버전 다시 맞춰본다.
$ sudo wget https://archive.apache.org/dist/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz -P /tmp
$ sudo tar xf /tmp/apache-maven-3.6.3-bin.tar.gz -C /opt
$ sudo rm /opt/maven
$ sudo ln -s /opt/apache-maven-3.6.3 /opt/maven



# 환경변수. 파일 하단에 추가 (파일이 없어서 새로만드네.)
$ sudo vi /etc/profile.d/maven.sh
export M2_HOME=/opt/maven 
export MAVEN_HOME=/opt/maven 
export PATH=$M2_HOME/bin:$PATH

# 스크립트에 실행 권한 부여
$ sudo chmod +x /etc/profile.d/maven.sh

# 환경변수에 maven 경로 설정 등록
$ source /etc/profile.d/maven.sh

# maven 버전확인
$ mvn -version
 
```


### ----------> 이 윗부분까지의 작업은 모든 호스트에 동등하게 설치되어야 한다.(일부 빌드 관련 application 제외)


### 5. Bigtop 설치와 Ambari 빌드
- Ambari 직접빌드? - 번거로움, 저장소누락, 의존성문제
- Bigtop 도입 + mpack
- Bigtop 설치
	- git clone 또는 zip 파일 다운로드.
	소스파일 다운로드 : https://archive.apache.org/dist/bigtop/bigtop-3.2.0/bigtop-3.2.0-project.tar.gz
	git : git clone -b branch-3.2 https://github.com/apache/bigtop.git 
	- gradlew take --all 확인.
```
[ambari@mron-en01 ~]$ pwd
/home/ambari

[ambari@mron-en01 ~]$ git clone https://github.com/apache/bigtop.git    # 실수였다. bigtop 3.3은 ambari를 지원하지 않는다. 
Cloning into 'bigtop'...
remote: Enumerating objects: 49735, done.
remote: Counting objects: 100% (690/690), done.
remote: Compressing objects: 100% (360/360), done.
remote: Total 49735 (delta 333), reused 543 (delta 244), pack-reused 49045
Receiving objects: 100% (49735/49735), 48.90 MiB | 15.57 MiB/s, done.
Resolving deltas: 100% (26327/26327), done.
[ambari@mron-en01 ~]$ ll
total 4
drwxrwxr-x. 18 ambari ambari 4096 Jul 11 13:49 bigtop

# 최신버전 3.3(ambari 없음) 실수. 다시해야해. 
[ambari@mron-en01 ~]$ mv bigtop bigtop-3.3
[ambari@mron-en01 ~]$ git clone -b branch-3.2 https://github.com/apache/bigtop.git   # 3.2 branch 다운로드
[ambari@mron-en01 ~]$ mv bigtop bigtop-3.2
[ambari@mron-en01 ~]$ ln -s /home/ambari/bigtop-3.2/ bigtop

# 빌드 시작.
[ambari@mron-en01 ~]$ cd bigtop
[ambari@mron-en01 ~]$ ./gradlew task --all > bigtop-3.2_gradle_task.txt
... 
생략
...
Rules
-----
Pattern: clean<TaskName>: Cleans the output files of a task.
Pattern: build<ConfigurationName>: Assembles the artifacts of a configuration.
Pattern: upload<ConfigurationName>: Assembles and uploads the artifacts belonging to a configuration.

BUILD SUCCESSFUL in 15s
1 actionable task: 1 executed
[ambari@mron-en01 bigtop]$

# 05.bigtop-3.2_gradle빌드.log 참고.

```
	
- bigtop에서 Ambari 빌드 (1~2시간 걸림)
```
$ cd /home/ambari/bigtop

# Ambari 설치파일 빌드 (server, agent) .. 오 된다. 
$ ./gradlew task ambari-pkg  # 아!!!! 빡쳐. 30분 돌고 빌드 실패. 아래로그 참조.
# maven 버전 바꾸고 다시 실행 했더니, 아까 중단됐던 71% 지점에서 다시 진행하네?? 

# 정상 종료.
Executing(--clean): /bin/sh -e /var/tmp/rpm-tmp.b4Gj8n
+ umask 022
+ cd /home/ambari/bigtop-3.2/build/ambari/rpm//BUILD
+ rm -rf apache-ambari-2.7.5-src
+ exit 0

BUILD SUCCESSFUL in 52m 40s
7 actionable tasks: 7 executed
[ambari@mron-en01 bigtop]$



# Ambari mpack 파일 빌드
$ ./gradlew task bigtop-ambari-mpack-pkg

# Ambari mapack을 설치하기 위한 bigtop-utils 빌드
$ ./gradlew task bigtop-utils-pkg

```


- 아!!!! 빡쳐. 30분 돌고 빌드 실패.
```
생략
...
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/commons/commons-collections4/4.2/commons-collections4-4.2.jar (753 kB at 8.9 MB/s)
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for Ambari Main 2.7.5.0.0:
[INFO]
[INFO] Ambari Main ........................................ SUCCESS [03:11 min]
[INFO] Apache Ambari Project POM .......................... SUCCESS [  0.008 s]
[INFO] Ambari Web ......................................... SUCCESS [02:16 min]
[INFO] Ambari Views ....................................... SUCCESS [01:55 min]
[INFO] Ambari Admin View .................................. SUCCESS [01:02 min]
[INFO] ambari-utility ..................................... SUCCESS [02:15 min]
[INFO] ambari-metrics ..................................... SUCCESS [  0.374 s]
[INFO] Ambari Metrics Common .............................. SUCCESS [  6.512 s]
[INFO] Ambari Metrics Hadoop Sink ......................... SUCCESS [  4.065 s]
[INFO] Ambari Metrics Flume Sink .......................... SUCCESS [  1.549 s]
[INFO] Ambari Metrics Kafka Sink .......................... SUCCESS [  1.962 s]
[INFO] Ambari Metrics Storm Sink .......................... SUCCESS [  4.552 s]
[INFO] Ambari Metrics Collector ........................... FAILURE [01:01 min]
[INFO] Ambari Metrics Monitor ............................. SKIPPED
[INFO] Ambari Metrics Grafana ............................. SKIPPED
[INFO] Ambari Metrics Host Aggregator ..................... SKIPPED
[INFO] Ambari Metrics Assembly ............................ SKIPPED
[INFO] Ambari Service Advisor ............................. SKIPPED
[INFO] Ambari Server ...................................... SKIPPED
[INFO] Ambari Functional Tests ............................ SKIPPED
[INFO] Ambari Agent ....................................... SKIPPED
[INFO] ambari-logsearch ................................... SKIPPED
[INFO] Ambari Logsearch Appender .......................... SKIPPED
[INFO] Ambari Logsearch Config Api ........................ SKIPPED
[INFO] Ambari Logsearch Config JSON ....................... SKIPPED
[INFO] Ambari Logsearch Config Solr ....................... SKIPPED
[INFO] Ambari Logsearch Config Zookeeper .................. SKIPPED
[INFO] Ambari Logsearch Config Local ...................... SKIPPED
[INFO] Ambari Logsearch Log Feeder Plugin Api ............. SKIPPED
[INFO] Ambari Logsearch Log Feeder Container Registry ..... SKIPPED
[INFO] Ambari Logsearch Log Feeder ........................ SKIPPED
[INFO] Ambari Logsearch Web ............................... SKIPPED
[INFO] Ambari Logsearch Server ............................ SKIPPED
[INFO] Ambari Logsearch Assembly .......................... SKIPPED
[INFO] Ambari Logsearch Integration Test .................. SKIPPED
[INFO] ambari-infra ....................................... SKIPPED
[INFO] Ambari Infra Solr Client ........................... SKIPPED
[INFO] Ambari Infra Solr Plugin ........................... SKIPPED
[INFO] Ambari Infra Manager ............................... SKIPPED
[INFO] Ambari Infra Assembly .............................. SKIPPED
[INFO] Ambari Infra Manager Integration Tests ............. SKIPPED
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  12:22 min
[INFO] Finished at: 2024-07-11T15:46:17+09:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-dependency-plugin:3.6.0:copy-dependencies (default) on project ambari-metrics-timelineservice: Excluding every artifact inside 'test' resolution scope means excluding everything: you probably want includeScope='compile', read parameters documentation for detailed explanations -> [Help 1]
# --->  ./build/ambari/rpm/BUILD/apache-ambari-2.7.5-src/ambari-metrics/ambari-metrics-timelineservice/pom.xml 문서 확인해 봐도. compile 로 잘 셋팅 되 있음. 
# maven 버전이 문제 인것 같아 3.9.4 -> 3.6.3 으로 재설치 해본다. 3.6.3 으로 재실행 했더니 넘어갔다. ㅎ..

[ERROR]
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR]
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
[ERROR]
[ERROR] After correcting the problems, you can resume the build with the command
[ERROR]   mvn <args> -rf :ambari-metrics-timelineservice
error: Bad exit status from /var/tmp/rpm-tmp.P7nJId (%build)

    Bad exit status from /var/tmp/rpm-tmp.P7nJId (%build)

RPM build errors:

> Task :ambari-rpm FAILED

FAILURE: Build failed with an exception.

* Where:
Script '/home/ambari/bigtop-3.2/packages.gradle' line: 529

* What went wrong:
Execution failed for task ':ambari-rpm'.
> Process 'command 'rpmbuild'' finished with non-zero exit value 1

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 14m 21s
6 actionable tasks: 6 executed
[ambari@mron-en01 bigtop]$

```

- maven을 3.6.3 으로 다시 설치해봄. 정상 처리되면서 넘어감. 

- 빌드 되면 rpm 파일 생성. 확인 (4개 rpm 확인)
```
[ambari@mron-en01 output]$ cd bigtop/output
[ambari@mron-en01 output]$ ls
ambari  bigtop-ambari-mpack  bigtop-utils
[ambari@mron-en01 output]$ find | grep .noarch.rpm
./bigtop-ambari-mpack/noarch/bigtop-ambari-mpack-2.7.5.0-1.el7.noarch.rpm
./ambari/noarch/ambari-server-2.7.5.0-1.el7.noarch.rpm
./ambari/noarch/ambari-agent-2.7.5.0-1.el7.noarch.rpm
./bigtop-utils/noarch/bigtop-utils-3.2.1-1.el7.noarch.rpm
```

- rpm 파일 복사 (빌드가 오래걸렸으므로, 그 결과물을 host에 별도 보관.)
- 참고. bigtop 3.3.0 부터는 ambari가 지원제외됨. 3.2.1은 정상지원.
    그런데, ambari 신규버전은 bigtop 3.3을 지원한다고함.
	그래서, bigtop3.3 부터는 ambari 2.8을 단독 설치하면 사용가능할 것이라고 예상함.


### 6. CentOs 7 가상이미지 복제와 Ambari 설치
- 가상머신 복제 master, worker 차등.
- 가상머신 네트워크 설정, 복제 옵션설정, 복제 반복
- 복제후 네트워크 설정변경 hostname, ip
- Master에서 ssh 설정변경
- root, ambari 계정의 ssh-key 생성(ssh-keygen 명령), ssh-key 복사. 비번없이 ssh접속 확인.
- id_rsa 파일을 저장소 또는 vm호스트에 보관. - 추후 ambari 설치시 업로드용.
- Worker에서 ssh 설정. ssh-key 모든노드에 서로 복사. 반복. 비번없이 ssh접속 확인.
```
# (모두 ambari계정)
# 각자의 콘솔에서 각자의 ssh 키 생성  (질문에 다 그냥 엔터) (root는 다 되있네)
ssh-keygen -t rsa -b 4096 -C "ambari@mron-en01"
ssh-keygen -t rsa -b 4096 -C "ambari@mron-nn01"
ssh-keygen -t rsa -b 4096 -C "ambari@mron-dn01"
ssh-keygen -t rsa -b 4096 -C "ambari@mron-dn04"

# ssh 공개키 배포 (4개 호스트에서 아래 4줄을 모두 실행)(비밀번호 입력)
ssh-copy-id ambari@mron-nn01
ssh-copy-id ambari@mron-en01
ssh-copy-id ambari@mron-dn01
ssh-copy-id ambari@mron-dn04

# 자기 자신의 host에도 해 줘야 ansible 등 여러 명령이 자신에게도 먹힘.
[root@mron-en01 ansible]# ssh-copy-id root@mron-en01

# 확인
[ambari@mron-en01]$ ssh mron-dn01
[ambari@mron-dn01]$ 
```

- mysql 설치(생략). ambari 설치시 postgreSQL을 자동설치 한다는데...
- hive 설치할거면 mysql 필요한데. 그냥 그때가서 하자.
- Ambari 설치. 
	- Master : Ambari server/agent, Bigtop Utils, Bigtop Ambari mpack 설치.
	- Worker : Ambari agent만 설치.
- Worker에 Ambari agent 설치.
```
# (ambari계정)
# en01에서 각 호스트에 ambari agent rpm 복사 
$ ansible -i /etc/ansible/hosts all -m copy -a "src=/home/ambari/bigtop/output/ambari/noarch/ambari-agent-2.7.5.0-1.el7.noarch.rpm dest=/home/ambari/"

$ ansible -i /etc/ansible/hosts all -m shell -a "sudo yum -y install /home/ambari/ambari-agent-2.7.5.0-1.el7.noarch.rpm"

모두 Complete!
```
- Master에 Ambari, bigtop, mpack 설치.
```
# 빌드된 설치파일 복사.
[ambari@mron-en01 output]$ pwd
/home/ambari/bigtop/output
[ambari@mron-en01 output]$ find . -iname "*.noarch.rpm" -exec cp -- "{}" /home/ambari/ \;

# (ambari 계정)
-- $ sudo yum -y install /home/ambari/ambari-agent-2.7.5.0-1.el7.noarch.rpm   --> Master의 agent는 앞에서 설치함.
$ sudo yum -y install /home/ambari/ambari-server-2.7.5.0-1.el7.noarch.rpm
$ sudo yum -y install /home/ambari/bigtop-utils-3.2.1-1.el7.noarch.rpm
$ sudo yum -y install /home/ambari/bigtop-ambari-mpack-2.7.5.0-1.el7.noarch.rpm

모두 Complete!
```

- ambari 상세 설정 수정 (질문 답변 참고)
```
$ sudo ambari-server setup

[ambari@mron-en01 ~]$ sudo ambari-server setup
Using python  /usr/bin/python
Setup ambari-server
Checking SELinux...
SELinux status is 'enabled'
SELinux mode is 'permissive'
WARNING: SELinux is set to 'permissive' mode and temporarily disabled.
OK to continue [y/n] (y)? y                                             -------> 답변
Customize user account for ambari-server daemon [y/n] (n)? y              -------> 답변
Enter user account for ambari-server daemon (root):ambari              -------> 답변
Adjusting ambari-server permissions and ownership...
Checking firewall status...
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
==============================================================================
Enter choice (1): 1                                                         -------> 답변
To download the Oracle JDK and the Java Cryptography Extension (JCE) Policy Files you must accept the license terms found at http://www.oracle.com/technetwork/java/javase/terms/license/index.html and not accepting will cancel the Ambari Server setup and you must install the JDK and JCE files manually.
Do you accept the Oracle Binary Code License Agreement [y/n] (y)? y                     -------> 답변
Downloading JDK from http://public-repo-1.hortonworks.com/ARTIFACTS/jdk-8u112-linux-x64.tar.gz to /var/lib/ambari-server/resources/jdk-8u112-linux-x64.tar.gz
jdk-8u112-linux-x64.tar.gz... 100% (174.7 MB of 174.7 MB)
Successfully downloaded JDK distribution to /var/lib/ambari-server/resources/jdk-8u112-linux-x64.tar.gz
Installing JDK to /usr/jdk64/
Successfully installed JDK to /usr/jdk64/
Downloading JCE Policy archive from http://public-repo-1.hortonworks.com/ARTIFACTS/jce_policy-8.zip to /var/lib/ambari-server/resources/jce_policy-8.zip

Successfully downloaded JCE Policy archive to /var/lib/ambari-server/resources/jce_policy-8.zip
Installing JCE policy...
Check JDK version for Ambari Server...
JDK version found: 8
Minimum JDK version is 8 for Ambari. Skipping to setup different JDK for Ambari Server.
Checking GPL software agreement...
GPL License for LZO: https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html
Enable Ambari Server to download and install GPL Licensed LZO packages [y/n] (n)? n                -------> 답변
Completing setup...
Configuring database...
Enter advanced database configuration [y/n] (n)? n                                            -------> 답변
Configuring database...
Default properties detected. Using built-in database.
Configuring ambari database...
Checking PostgreSQL...
Running initdb: This may take up to a minute.
Initializing database ... OK


About to start PostgreSQL
Configuring local database...
Configuring PostgreSQL...
Restarting PostgreSQL
Creating schema and user...
done.
Creating tables...
done.
Extracting system views...
ambari-admin-2.7.5.0.0.jar

Ambari repo file doesn't contain latest json url, skipping repoinfos modification
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.
[ambari@mron-en01 ~]$

```

- mpack 추가 설치
```
[ambari@mron-en01 ~]$ sudo ambari-server install-mpack --mpack=/usr/lib/bigtop-ambari-mpack/bgtp-ambari-mpack-1.0.0.0-SNAPSHOT-bgtp-ambari-mpack.tar.gz --verbose

...
INFO: Loading properties from /etc/ambari-server/conf/ambari.properties
INFO: Symlink: /var/lib/ambari-server/resources/stacks/BGTP/1.0/blueprints
INFO: Symlink: /var/lib/ambari-server/resources/stacks/BGTP/1.0/configuration
INFO: Symlink: /var/lib/ambari-server/resources/stacks/BGTP/1.0/hooks
...
INFO: about to run command: chmod  -R  0755 /var/lib/ambari-server/resources/stacks
INFO:
process_pid=26050
INFO: about to run command: chown  -R -L ambari /var/lib/ambari-server/resources/stacks
INFO:
process_pid=26051
INFO: about to run command: chmod  -R  0755 /var/lib/ambari-server/resources/extensions
...
#( 아래 세줄 나오면 정상이랜다.)
INFO: about to run command: chown  -R -L ambari /var/lib/ambari-server/resources/mpacks/cache
INFO:
process_pid=26066
INFO: about to run command: chown  -R -L ambari /var/lib/ambari-server/resources/dashboards
INFO:
process_pid=26067
INFO: Management pack bgtp-ambari-mpack-1.0.0.0-SNAPSHOT successfully installed! Please restart ambari-server.
INFO: Loading properties from /etc/ambari-server/conf/ambari.properties
Ambari Server 'install-mpack' completed successfully.
```

- postgreSQL 설치, jdbc 드라이버 설정 (ambari가 postgreSQL을 사용하고, Hive관련 postgreSQL 부분은 추후 mariaDB로 변경하자)
```
[root@mron-en01 ~]# su postgres
bash-4.2$ psql --u postgres
could not change directory to "/root"
psql (9.2.24)
Type "help" for help.

postgres=# help
You are using psql, the command-line interface to PostgreSQL.
Type:  \copyright for distribution terms
       \h for help with SQL commands
       \? for help with psql commands
       \g or terminate with semicolon to execute query
       \q to quit
postgres=# create user hive password 'hive123!'
postgres-# create database hive
postgres-# \q

bash-4.2$ exit
[root@mron-en01 ~]# sudo vi /var/lib/pgsql/data/pg_hba.conf
# hive 계정 추가
# add hive by henry 20240715
local  all  hive,ambari,mapred md5
host  all   hive,ambari,mapred 0.0.0.0/0  md5
host  all   hive,ambari,mapred ::/0 md5
[root@mron-en01 ~]# sudo systemctl restart postgresql

[root@mron-en01 ~]# sudo systemctl status postgresql
● postgresql.service - PostgreSQL database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-07-15 13:49:16 KST; 7s ago
  Process: 26456 ExecStop=/usr/bin/pg_ctl stop -D ${PGDATA} -s -m fast (code=exited, status=0/SUCCESS)
  Process: 25823 ExecReload=/usr/bin/pg_ctl reload -D ${PGDATA} -s (code=exited, status=0/SUCCESS)
  Process: 26467 ExecStart=/usr/bin/pg_ctl start -D ${PGDATA} -s -o -p ${PGPORT} -w -t 300 (code=exited, status=0/SUCCESS)

# jdbc 드라이버 설치.
[root@mron-en01 ~]# wget https://jdbc.postgresql.org/download/postgresql-42.7.3.jar
[root@mron-en01 ~]# cp /root/postgresql-42.7.3.jar /home/ambari/
[root@mron-en01 ~]# chown ambari:ambari /home/ambari/postgresql-42.7.3.jar
[root@mron-en01 ~]# su ambari 또는 exit (ambari 계정으로 돌아가라고)

# 실행
[ambari@mron-en01 ~]$ cd
[ambari@mron-en01 ~]$ ambari-server setup --jdbc-db=postgres --jdbc-driver=postgresql-42.7.3.jar

Using python  /usr/bin/python
Setup ambari-server
Copying postgresql-42.7.3.jar to /var/lib/ambari-server/resources/postgresql-42.7.3.jar
Creating symlink /var/lib/ambari-server/resources/postgresql-42.7.3.jar to /var/lib/ambari-server/resources/postgresql-jdbc.jar
If you are updating existing jdbc driver jar for postgres with postgresql-42.7.3.jar. Please remove the old driver jar, from all hosts. Restarting services that need the driver, will automatically copy the new jar to the hosts.
JDBC driver was successfully initialized.
Ambari Server 'setup' completed successfully.
[root@mron-en01 ambari]#

```

- 모든워커 ambari agent 시작
```
# 모든 ambari-agent 확인 (다 죽어있겠지)
ansible -i /etc/ansible/hosts all -m shell -a "sudo ambari-agent status"

# 켜기
ansible -i /etc/ansible/hosts all -m shell -a "sudo ambari-agent start"

# 다시 확인 (켜져있음)
ansible -i /etc/ansible/hosts all -m shell -a "sudo ambari-agent status"
```


- ambari server 시작
```
# ambari-server 확인 (꺼저있겠지)
[ambari@mron-en01 ~]$ sudo ambari-server status

# 켜기
[ambari@mron-en01 ~]$ sudo ambari-server restart

Using python  /usr/bin/python
Restarting ambari-server
Ambari Server is not running
Ambari Server running with administrator privileges.
Organizing resource files at /var/lib/ambari-server/resources...
Ambari database consistency check started...
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start.........
Server started listening on 8080

# 근데, 8080이네.... 이거 80으로 바꿔야겠음.
# 설정파일 열었는데, 아래 항목이 없어서 맨 아래에 추가하고, 80 셋팅.
[ambari@mron-en01 ~]$ sudo vi /etc/ambari-server/conf/ambari.properties
client.api.port=80

[ambari@mron-en01 ~]$ sudo ambari-server start[or restart]

# 80 으로 셋팅하면, 오류나고, 8080으로 하면 오류 안남. 

# 80 설정. (오류메세지 참고.)
...
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start..............
DB configs consistency check: no errors and warnings were found.
ERROR: Exiting with exit code -1.
REASON: Ambari Server java process has stopped. Please check the logs for more information.


# 8080 설정. (성공)
...
Waiting for server start.............
Server started listening on 8080
DB configs consistency check: no errors and warnings were found.
Ambari Server 'start' completed successfully.
```

- 80포트 사용시 오류 
```
2024-07-15 14:30:56,777 ERROR [main] AmbariServer:1120 - Failed to run the Ambari Server
java.net.SocketException: Permission denied
        at sun.nio.ch.Net.bind0(Native Method)
        at sun.nio.ch.Net.bind(Net.java:433)
        at sun.nio.ch.Net.bind(Net.java:425)
        at sun.nio.ch.ServerSocketChannelImpl.bind(ServerSocketChannelImpl.java:223)
        at sun.nio.ch.ServerSocketAdaptor.bind(ServerSocketAdaptor.java:74)
        at org.eclipse.jetty.server.ServerConnector.openAcceptChannel(ServerConnector.java:339)
        at org.eclipse.jetty.server.ServerConnector.open(ServerConnector.java:307)
        at org.eclipse.jetty.server.AbstractNetworkConnector.doStart(AbstractNetworkConnector.java:80)
        at org.eclipse.jetty.server.ServerConnector.doStart(ServerConnector.java:235)
        at org.eclipse.jetty.util.component.AbstractLifeCycle.start(AbstractLifeCycle.java:68)
        at org.eclipse.jetty.server.Server.doStart(Server.java:395)
        at org.eclipse.jetty.util.component.AbstractLifeCycle.start(AbstractLifeCycle.java:68)
        at org.apache.ambari.server.controller.AmbariServer.run(AmbariServer.java:574)
        at org.apache.ambari.server.controller.AmbariServer.main(AmbariServer.java:1114)

```
- 일단 ambari-server 는 8080 으로 기동.

- 해결책(?) - 해결됨. 
```
# 포트 리디렉션
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

# 혹시 DB 접속하려면
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3307
```
- (참고) 포트 리디렉션 삭제
```
sudo iptables -L INPUT --line-numbers
iptables -D INPUT <line-number>   # 1번 삭제후 2번은 자동으로 1번이 됨.
```



- 외부에서 http://mron-en01:80/ 접근시 8080포트에서 서비스중인 ambari 웹 페이지 열린다. 
- ambari 계정 admin / admin
- 아씨. 좀 감동적임. 넘흐 힘드네.



### 7. Ambari 설정과 hadoop, spark 설치.
- ambari 웹페이지 접속. 포트 포워딩방법. (skip)
- ambari 접속과 agent 구성. (skip)
- 웹ui 포트변경 /etc/ambari-server/conf/ambari.properties . 재시작. (skip)
- admin/admin 로그인. 
- 클러스터 이름부여 (= isbig), BGTP 스택 확인및 설정
- 클러스터 호스트의 FQDNs 입력 : 그냥 host명 넣었다. 저장시 오류 메세지 뜬다. continue 했다. 
- ambari-server 호스트의 ambari계정의 ras-id 파일을 갖고있다가 입력. ssh user는 root -> ambari 로 변경
- 다음단계로 가면. 모든 호스트 체크한다.
```
# 전체적으로 문제 없는데, mron 작동중인 server에서 java 관련 문제를 warnning 한다. 
Process Issues (2)
The following processes should not be running
Process
/home/eva/mron/bin/jdk1.8.0_102...			Running on 1 host
/home/eva/mron/bin/java8/bin/ja...			Running on 1 host

# 기분이 거시기 하므로, 엠론 관련 서비스 다 내리고, 다시 체크해 보자. 
eva@mron-dn04]$ /home/eva/mron/bin/shell/stopAll.sh
eva@mron-dn04]$ /home/eva/mron/WEB/apache-tomcat-8.0.36/bin/shutdown.sh

# 다 내리고 다시 host 체크하면. clear 됨. 
# 일단 하둡 설치 끝날때 까지 그냥 꺼놓자.
```
 
- 설치 도중. hive 설치시 metastore db 설정. mron-dn01 서버에 설치.
- new mysql 옵션 있는데, 
existing mysql/mariaDB 를 선택함. (아래와 같은 메세지 나옴.)
```
To use MySQL with Hive, you must download the https://dev.mysql.com/downloads/connector/j/ from MySQL. Once downloaded to the Ambari Server host, run:
ambari-server setup --jdbc-db=mysql --jdbc-driver=/path/to/mysql/mysql-connector-java.jar
```

- 이걸 위해서 hive 설치예정 서버에 mariaDB 설치함.
- 설치 과정은. mron mariadb 설치 과정과 동일. ver 10.6.4
``` 
[root@mron-dn01 bin]# mysqladmin -u root password 'root123'
[root@mron-dn01 bin]# mysql -u root -p

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 5
Server version: 10.6.4-MariaDB MariaDB Server
```

- mariadb 설치 정보
```
--basedir=/usr/local/mariadb-ticat 
--datadir=/mdata 
--defaults-file=/etc/my.cnf
```

-- 유저셋팅
```
use mysql;
select host, user, password from user;

create database hive;

CREATE USER 'hive'@'%' IDENTIFIED BY 'hive123!';
CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive123!';
CREATE USER 'hive'@'mron-dn01' IDENTIFIED BY 'hive123!';
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'%' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'localhost' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'mron-dn01' WITH GRANT OPTION;

CREATE USER 'root'@'%' IDENTIFIED BY 'root123';
CREATE USER 'root'@'localhost' IDENTIFIED BY 'root123';
CREATE USER 'root'@'mron-dn01' IDENTIFIED BY 'root123';
GRANT ALL PRIVILEGES ON mron.* TO 'root'@'%' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON mron.* TO 'root'@'localhost' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON mron.* TO 'root'@'mron-dn01' WITH GRANT OPTION;

flush PRIVILEGES;

```


- mariadb connector j 3.4.0와 3.1.0이 java8 에서 build 되지 않았다. java9 이상 지원 인것 같다. 
- 그래서, mron 설치 lib 폴더에 있는  mysql-connector-java-5.1.40.jar 를 사용해 보자.
```
[ambari@mron-en01 ~]$ ambari-server setup --jdbc-db=mysql --jdbc-driver=/home/ambari/mysql-connector-java-5.1.40.jar
...
Using python  /usr/bin/python
Setup ambari-server
Ambari-server setup is run with root-level privileges, passwordless sudo access for some commands commands may be required
Copying /home/ambari/mysql-connector-java-5.1.40.jar to /var/lib/ambari-server/resources/mysql-connector-java-5.1.40.jar
Creating symlink /var/lib/ambari-server/resources/mysql-connector-java-5.1.40.jar to /var/lib/ambari-server/resources/mysql-connector-java.jar
If you are updating existing jdbc driver jar for mysql with mysql-connector-java-5.1.40.jar. Please remove the old driver jar, from all hosts. Restarting services that need the driver, will automatically copy the new jar to the hosts.
JDBC driver was successfully initialized.
Ambari Server 'setup' completed successfully.
[ambari@mron-en01 ~]$

```

- 설정 완료되고, db접근 테스트 OK. 드디어 database 탭을 넘어갔다.

- directories 탭 인데, 여러가지 경로설정 화면인데, 
그냥 기본값으로 가자. (J)

- account 항목에는 아무것도 안나오네,

- allconfigurations 탭에서, 각 appli 마다 세부 항목 설정화면인데, 대부분 기본값 유지.
- hadoop.proxyuser.* 항목에 값 넣으라는데, * 넣어도 된단다. (블로그 가이드)

- allconfigurations 탭 넘어가서, next 하면 review 나옴.
```
Admin Name : admin

Cluster Name : isbig

Total Hosts : 4 (4 new)

Repositories:

redhat7 (BGTP-1.0):
http://repos.bigtop.apache.org/releases/3.2.1/centos/7/$basearch
Services:

HDFS
DataNode : 2 hosts
NameNode : mron-dn04
SNameNode : mron-dn01
YARN + MapReduce2
NodeManager : 2 hosts
ResourceManager : mron-dn04
Tez
Clients : 2 hosts
Hive
Metastore : mron-dn01
HiveServer2 : mron-dn01
WebHCat Server : mron-nn01
Database : Existing MySQL / MariaDB Database
ZooKeeper
Server : 3 hosts
Ambari Metrics
Metrics Collector : mron-en01
Grafana : mron-en01
Kafka
Broker : mron-nn01
Spark
History Server : mron-dn01
Thrift Server : 2 hosts
```

- deploy 하기 전에. generate blueprint 누르면, 설정내용 json 파일이 zip 으로 다운로드 됨.
- 배포 진행됨. 약 20분? 완료. 


### ambari Log
ambari-server : /var/log/ambari-server
ambari-agent : 각 호스트 /var/log/ambari-agent

task log
stderr:   /var/lib/ambari-agent/data/errors-229.txt
stdout:   /var/lib/ambari-agent/data/output-229.txt

### 일단 여기까지. 기본적인 셋팅은 되었고 'hdfs dfs command' 도 작동한다.


### spark thrift server start 오류 관련하여
- spark thrift server 를 수동 실행 해 보았다.
```
spark-sql --master yarn --deploy-mode client --conf spark.sql.hive.thriftServer.singleSession=true --class org.apache.spark.sql.hive.thriftserver.HiveThriftServer2
```

- 오류메세지
```

24/07/25 16:09:49 ERROR SparkContext: Error initializing SparkContext.
org.apache.hadoop.security.AccessControlException: Permission denied: user=ambari, access=WRITE, inode="/user":hdfs:hdfs:drwxr-xr-x

```

- 해결방안
```
# hdfs 접속
hdfs dfs -ls /

# /user 디렉토리 확인
hdfs dfs -ls /

drwxr-xr-x   - hdfs   hdfs            0 2024-07-25 15:53 /user

# 계정변경 
[ambari@mron-dn01 conf]$ su root
Password:
[root@mron-dn01 conf]# su hdfs
[hdfs@mron-dn01 conf]$

# 권한변경
[hdfs@mron-dn01 conf]$ hdfs dfs -chmod 775 /user
[hdfs@mron-dn01 conf]$ hdfs dfs -chown -R ambari:hdfs /user
[hdfs@mron-dn01 conf]$ hdfs dfs -mkdir /user/ambari
[hdfs@mron-dn01 conf]$ hdfs dfs -chown ambari:hdfs /user/ambari
[hdfs@mron-dn01 conf]$ hdfs dfs -ls /user
Found 7 items
drwxr-xr-x   - ambari hdfs          0 2024-07-25 16:17 /user/ambari
drwxrwx---   - ambari hdfs          0 2024-07-25 15:53 /user/ambari-qa
drwxr-xr-x   - ambari hdfs          0 2024-07-16 16:14 /user/ams
drwxr-xr-x   - ambari hdfs          0 2024-07-16 16:15 /user/hcat
drwxr-xr-x   - ambari hdfs          0 2024-07-25 15:53 /user/hdfs
drwxr-xr-x   - ambari hdfs          0 2024-07-17 15:39 /user/hive
drwxrwxr-x   - ambari hdfs          0 2024-07-16 16:21 /user/spark

# 다시 ambari로..
[hdfs@mron-dn01 conf]$ exit
exit
[root@mron-dn01 conf]# su ambari
[ambari@mron-dn01 conf]$

# 다시 spark thrift server 를 수동실행 해보면 되는데... 
```
- ambari web에서 해봤는데 안됨.  
- spark thriftserver start 못함.


### 참고. connection refused 오류발생.
- ambari web ui 에서 service를 start 할때 jmx, restAPI, uri 등등 접속이 안된다는 메세지가 종종 있음.  
- nginx 또는 nc로 포트를 열어 테스트 해보면, 접속은 가능한 상태임.  
- 방화벽 또는 네트워크 문제로 포트가 막혀있지 않다는 것.  
- 그렇다면 접속이 안되는 것은 해당 서비스가 로딩되지 않았다고 간주해야 함.  
- namenode ha 구성에 실패하고 dn01이 namenode 역할을 못하게 되면서, 연결된 서비스들이 start 못하게 됨.   



### Ambari에서 journalnode 설치하기. (namenode HA 구성. 1번째 시도)
- HDFS 메뉴에서 오른쪽 Action 클릭하면, 
"Enable Namenode HA" 메뉴 있음. (ㅜㅜ)

- Manual Steps Required: Create Checkpoint on NameNode
```
1. Login to the NameNode host mron-dn04.

2. Put the NameNode in Safe Mode (read-only mode):
# sudo su hdfs -l -c 'hdfs dfsadmin -safemode enter'

3. Once in Safe Mode, create a Checkpoint:
# sudo su hdfs -l -c 'hdfs dfsadmin -saveNamespace'

4.You will be able to proceed once Ambari detects that the NameNode is in Safe Mode and the Checkpoint has been created successfully.

If the Next button is enabled before you run the "Step 4: Create a Checkpoint" command, it means there is a recent Checkpoint already and you may proceed without running the "Step 4: Create a Checkpoint" command.
```
- 여기서 Next 하면 HA 구성 시작함. 막대기 쭉쭉~~

- Manual Steps Required: Initialize JournalNodes
```
1. Login to the NameNode host mron-dn04.

2. Initialize the JournalNodes by running:
# sudo su hdfs -l -c 'hdfs namenode -initializeSharedEdits'

3. You will be able to proceed once Ambari detects that the JournalNodes have been initialized successfully.
```

- sudo su hdfs -l -c 'hdfs namenode -initializeSharedEdits' 실행결과.
- 이 결과는 (/etc/security/clientKeys/all.jks noSearch 문제를 제외하고는,) 성공할때의 결과와 log 메세지가 같게 끝난다. 
- 참고. https://www.youtube.com/watch?v=TCdxK8C1Nn0  2:40 지점.
```
[root@mron-dn04 conf]# sudo su hdfs -l -c 'hdfs namenode -initializeSharedEdits'
WARNING: HADOOP_NAMENODE_OPTS has been replaced by HDFS_NAMENODE_OPTS. Using value of HADOOP_NAMENODE_OPTS.
24/07/23 15:40:30 INFO namenode.NameNode: STARTUP_MSG:
/************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = mron-DN04/192.168.0.4
STARTUP_MSG:   args = [-initializeSharedEdits]
STARTUP_MSG:   version = 3.3.5
STARTUP_MSG:   classpath = /etc/hadoop/conf:/usr/lib/hadoop/lib/json-smart-2.4.7.jar:/usr/lib/hadoop/lib/accessors-smart-2.4.7.jar:/usr/lib/hadoop/lib/jaxb-impl-2.2.3-1.jar:/usr/lib/hadoop/lib/animal-snif  
<<...생략...>>  ib/protobuf-java-2.5.0.jar:/usr/lib/tez/lib/RoaringBitmap-0.7.45.jar:/usr/lib/tez/lib/slf4j-api-1.7.30.jar:/usr/lib/tez/lib/snappy-java-1.1.8.4.jar:/usr/lib/tez/lib/tez.tar.gz:/etc/tez/conf
STARTUP_MSG:   build = https://github.com/apache/bigtop.git -r 4a34226ec01a894fd96cc00c052d96e61673c60e; compiled by 'jenkins' on 2023-08-02T05:53Z
STARTUP_MSG:   java = 1.8.0_112
************************************************************/
24/07/23 15:40:30 INFO namenode.NameNode: registered UNIX signal handlers for [TERM, HUP, INT]
24/07/23 15:40:30 INFO namenode.NameNode: createNameNode [-initializeSharedEdits]
24/07/23 15:40:31 WARN web.URLConnectionFactory: Cannot load customized ssl related configuration. Fallback to system-generic settings.
java.nio.file.NoSuchFileException: /etc/security/clientKeys/all.jks
        at sun.nio.fs.UnixException.translateToIOException(UnixException.java:86)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:102)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:107)
        at sun.nio.fs.UnixFileSystemProvider.newByteChannel(UnixFileSystemProvider.java:214)
        at java.nio.file.Files.newByteChannel(Files.java:361)
        at java.nio.file.Files.newByteChannel(Files.java:407)
        at java.nio.file.spi.FileSystemProvider.newInputStream(FileSystemProvider.java:384)
        at java.nio.file.Files.newInputStream(Files.java:152)
        at org.apache.hadoop.security.ssl.ReloadingX509TrustManager.loadTrustManager(ReloadingX509TrustManager.java:131)
        at org.apache.hadoop.security.ssl.ReloadingX509TrustManager.<init>(ReloadingX509TrustManager.java:79)
        at org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory.createTrustManagersFromConfiguration(FileBasedKeyStoresFactory.java:131)
        at org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory.init(FileBasedKeyStoresFactory.java:292)
        at org.apache.hadoop.security.ssl.SSLFactory.init(SSLFactory.java:180)
        at org.apache.hadoop.hdfs.web.SSLConnectionConfigurator.<init>(SSLConnectionConfigurator.java:50)
        at org.apache.hadoop.hdfs.web.URLConnectionFactory.getSSLConnectionConfiguration(URLConnectionFactory.java:100)
        at org.apache.hadoop.hdfs.web.URLConnectionFactory.newDefaultURLConnectionFactory(URLConnectionFactory.java:79)
        at org.apache.hadoop.hdfs.server.common.Util.<clinit>(Util.java:76)
        at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getStorageDirs(FSNamesystem.java:1665)
        at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getNamespaceEditsDirs(FSNamesystem.java:1710)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.getConfigurationWithoutSharedEdits(NameNode.java:1332)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.initializeSharedEdits(NameNode.java:1373)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.createNameNode(NameNode.java:1752)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.main(NameNode.java:1841)
24/07/23 15:40:31 INFO common.Util: Assuming 'file' scheme for path /hadoop/hdfs/namenode in configuration.
24/07/23 15:40:31 INFO common.Util: Assuming 'file' scheme for path /hadoop/hdfs/namenode in configuration.
24/07/23 15:40:31 WARN namenode.FSNamesystem: Only one image storage directory (dfs.namenode.name.dir) configured. Beware of data loss due to lack of redundant storage directories!
24/07/23 15:40:31 WARN namenode.FSNamesystem: Only one namespace edits storage directory (dfs.namenode.edits.dir) configured. Beware of data loss due to lack of redundant storage directories!
24/07/23 15:40:31 INFO common.Util: Assuming 'file' scheme for path /hadoop/hdfs/namenode in configuration.
24/07/23 15:40:31 WARN common.Storage: set restore failed storage to true
24/07/23 15:40:31 INFO namenode.FSEditLog: Edit logging is async:true
24/07/23 15:40:31 INFO namenode.FSNamesystem: KeyProvider: null
24/07/23 15:40:31 INFO namenode.FSNamesystem: fsLock is fair: true
24/07/23 15:40:31 INFO namenode.FSNamesystem: Detailed lock hold time metrics enabled: false
24/07/23 15:40:31 INFO namenode.FSNamesystem: fsOwner                = hdfs (auth:SIMPLE)
24/07/23 15:40:31 INFO namenode.FSNamesystem: supergroup             = hdfs
24/07/23 15:40:31 INFO namenode.FSNamesystem: isPermissionEnabled    = true
24/07/23 15:40:31 INFO namenode.FSNamesystem: isStoragePolicyEnabled = true
24/07/23 15:40:31 INFO namenode.FSNamesystem: Determined nameservice ID: isbig
24/07/23 15:40:31 INFO namenode.FSNamesystem: HA Enabled: true
24/07/23 15:40:31 INFO blockmanagement.HeartbeatManager: Setting heartbeat recheck interval to 30000 since dfs.namenode.stale.datanode.interval is less than dfs.namenode.heartbeat.recheck-interval
24/07/23 15:40:31 INFO common.Util: dfs.datanode.fileio.profiling.sampling.percentage set to 0. Disabling file IO profiling
24/07/23 15:40:31 INFO blockmanagement.DatanodeManager: dfs.block.invalidate.limit : configured=1000, counted=60, effected=1000
24/07/23 15:40:31 INFO blockmanagement.DatanodeManager: dfs.namenode.datanode.registration.ip-hostname-check=true
24/07/23 15:40:31 INFO blockmanagement.BlockManager: dfs.namenode.startup.delay.block.deletion.sec is set to 000:00:00:00.000
24/07/23 15:40:31 INFO blockmanagement.BlockManager: The block deletion will start around 2024 Jul 23 15:40:31
24/07/23 15:40:31 INFO util.GSet: Computing capacity for map BlocksMap
24/07/23 15:40:31 INFO util.GSet: VM type       = 64-bit
24/07/23 15:40:31 INFO util.GSet: 2.0% max memory 26.3 GB = 539.1 MB
24/07/23 15:40:31 INFO util.GSet: capacity      = 2^26 = 67108864 entries
24/07/23 15:40:31 INFO blockmanagement.BlockManager: Storage policy satisfier is disabled
24/07/23 15:40:31 INFO blockmanagement.BlockManager: dfs.block.access.token.enable = true
24/07/23 15:40:31 INFO blockmanagement.BlockManager: dfs.block.access.key.update.interval=600 min(s), dfs.block.access.token.lifetime=600 min(s), dfs.encrypt.data.transfer.algorithm=null
24/07/23 15:40:31 INFO block.BlockTokenSecretManager: Block token key range: [0, 1073741823)
24/07/23 15:40:31 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.threshold-pct = 0.99
24/07/23 15:40:31 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.min.datanodes = 0
24/07/23 15:40:31 INFO blockmanagement.BlockManagerSafeMode: dfs.namenode.safemode.extension = 30000
24/07/23 15:40:31 INFO blockmanagement.BlockManager: defaultReplication         = 3
24/07/23 15:40:31 INFO blockmanagement.BlockManager: maxReplication             = 50
24/07/23 15:40:31 INFO blockmanagement.BlockManager: minReplication             = 1
24/07/23 15:40:31 INFO blockmanagement.BlockManager: maxReplicationStreams      = 2
24/07/23 15:40:31 INFO blockmanagement.BlockManager: redundancyRecheckInterval  = 3000ms
24/07/23 15:40:31 INFO blockmanagement.BlockManager: encryptDataTransfer        = false
24/07/23 15:40:31 INFO blockmanagement.BlockManager: maxNumBlocksToLog          = 1000
24/07/23 15:40:31 INFO namenode.FSDirectory: GLOBAL serial map: bits=29 maxEntries=536870911
24/07/23 15:40:31 INFO namenode.FSDirectory: USER serial map: bits=24 maxEntries=16777215
24/07/23 15:40:31 INFO namenode.FSDirectory: GROUP serial map: bits=24 maxEntries=16777215
24/07/23 15:40:31 INFO namenode.FSDirectory: XATTR serial map: bits=24 maxEntries=16777215
24/07/23 15:40:31 INFO util.GSet: Computing capacity for map INodeMap
24/07/23 15:40:31 INFO util.GSet: VM type       = 64-bit
24/07/23 15:40:31 INFO util.GSet: 1.0% max memory 26.3 GB = 269.6 MB
24/07/23 15:40:31 INFO util.GSet: capacity      = 2^25 = 33554432 entries
24/07/23 15:40:31 INFO namenode.FSDirectory: ACLs enabled? true
24/07/23 15:40:31 INFO namenode.FSDirectory: POSIX ACL inheritance enabled? true
24/07/23 15:40:31 INFO namenode.FSDirectory: XAttrs enabled? true
24/07/23 15:40:31 INFO namenode.NameNode: Caching file names occurring more than 10 times
24/07/23 15:40:31 INFO snapshot.SnapshotManager: Loaded config captureOpenFiles: false, skipCaptureAccessTimeOnlyChange: false, snapshotDiffAllowSnapRootDescendant: true, maxSnapshotLimit: 65536
24/07/23 15:40:31 INFO snapshot.SnapshotManager: SkipList is disabled
24/07/23 15:40:31 INFO util.GSet: Computing capacity for map cachedBlocks
24/07/23 15:40:31 INFO util.GSet: VM type       = 64-bit
24/07/23 15:40:31 INFO util.GSet: 0.25% max memory 26.3 GB = 67.4 MB
24/07/23 15:40:31 INFO util.GSet: capacity      = 2^23 = 8388608 entries
24/07/23 15:40:31 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.window.num.buckets = 10
24/07/23 15:40:31 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.num.users = 10
24/07/23 15:40:31 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.windows.minutes = 1,5,25
24/07/23 15:40:31 INFO namenode.FSNamesystem: Retry cache on namenode is enabled
24/07/23 15:40:31 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis
24/07/23 15:40:31 INFO util.GSet: Computing capacity for map NameNodeRetryCache
24/07/23 15:40:31 INFO util.GSet: VM type       = 64-bit
24/07/23 15:40:31 INFO util.GSet: 0.029999999329447746% max memory 26.3 GB = 8.1 MB
24/07/23 15:40:31 INFO util.GSet: capacity      = 2^20 = 1048576 entries
24/07/23 15:40:31 INFO common.Storage: Lock on /hadoop/hdfs/namenode/in_use.lock acquired by nodename 9294@mron-DN04
24/07/23 15:40:31 INFO namenode.FSImage: No edit log streams selected.
24/07/23 15:40:31 INFO namenode.FSImage: Planning to load image: FSImageFile(file=/hadoop/hdfs/namenode/current/fsimage_0000000000000104925, cpktTxId=0000000000000104925)
24/07/23 15:40:31 INFO namenode.FSImageFormatPBINode: Loading 8642 INodes.
24/07/23 15:40:31 INFO namenode.FSImageFormatPBINode: Successfully loaded 8642 inodes
24/07/23 15:40:31 INFO namenode.FSImageFormatPBINode: Completed update blocks map and name cache, total waiting duration 1ms.
24/07/23 15:40:31 INFO namenode.FSImageFormatProtobuf: Loaded FSImage in 0 seconds.
24/07/23 15:40:31 INFO namenode.FSImage: Loaded image for txid 104925 from /hadoop/hdfs/namenode/current/fsimage_0000000000000104925
24/07/23 15:40:31 INFO namenode.FSNamesystem: Need to save fs image? false (staleImage=false, haEnabled=true, isRollingUpgrade=false)
24/07/23 15:40:31 INFO namenode.NameCache: initialized with 7 entries 3065 lookups
24/07/23 15:40:31 INFO namenode.FSNamesystem: Finished loading FSImage in 286 msecs
24/07/23 15:40:31 WARN common.Storage: set restore failed storage to true
24/07/23 15:40:31 INFO namenode.FSEditLog: Edit logging is async:true
24/07/23 15:40:31 WARN web.URLConnectionFactory: Cannot load customized ssl related configuration. Fallback to system-generic settings.
java.nio.file.NoSuchFileException: /etc/security/clientKeys/all.jks
        at sun.nio.fs.UnixException.translateToIOException(UnixException.java:86)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:102)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:107)
        at sun.nio.fs.UnixFileSystemProvider.newByteChannel(UnixFileSystemProvider.java:214)
        at java.nio.file.Files.newByteChannel(Files.java:361)
        at java.nio.file.Files.newByteChannel(Files.java:407)
        at java.nio.file.spi.FileSystemProvider.newInputStream(FileSystemProvider.java:384)
        at java.nio.file.Files.newInputStream(Files.java:152)
        at org.apache.hadoop.security.ssl.ReloadingX509TrustManager.loadTrustManager(ReloadingX509TrustManager.java:131)
        at org.apache.hadoop.security.ssl.ReloadingX509TrustManager.<init>(ReloadingX509TrustManager.java:79)
        at org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory.createTrustManagersFromConfiguration(FileBasedKeyStoresFactory.java:131)
        at org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory.init(FileBasedKeyStoresFactory.java:292)
        at org.apache.hadoop.security.ssl.SSLFactory.init(SSLFactory.java:180)
        at org.apache.hadoop.hdfs.web.SSLConnectionConfigurator.<init>(SSLConnectionConfigurator.java:50)
        at org.apache.hadoop.hdfs.web.URLConnectionFactory.getSSLConnectionConfiguration(URLConnectionFactory.java:100)
        at org.apache.hadoop.hdfs.web.URLConnectionFactory.newDefaultURLConnectionFactory(URLConnectionFactory.java:91)
        at org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager.<init>(QuorumJournalManager.java:193)
        at org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager.<init>(QuorumJournalManager.java:126)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
        at org.apache.hadoop.hdfs.server.namenode.FSEditLog.createJournal(FSEditLog.java:1864)
        at org.apache.hadoop.hdfs.server.namenode.FSEditLog.initJournals(FSEditLog.java:299)
        at org.apache.hadoop.hdfs.server.namenode.FSEditLog.initJournalsForWrite(FSEditLog.java:264)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.initializeSharedEdits(NameNode.java:1383)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.createNameNode(NameNode.java:1752)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.main(NameNode.java:1841)
24/07/23 15:40:32 INFO namenode.FileJournalManager: Recovering unfinalized segments in /hadoop/hdfs/namenode/current
24/07/23 15:40:32 INFO namenode.FileJournalManager: Finalizing edits file /hadoop/hdfs/namenode/current/edits_inprogress_0000000000000104926 -> /hadoop/hdfs/namenode/current/edits_0000000000000104926-0000  000000000105695
24/07/23 15:40:32 WARN web.URLConnectionFactory: Cannot load customized ssl related configuration. Fallback to system-generic settings.
java.nio.file.NoSuchFileException: /etc/security/clientKeys/all.jks
        at sun.nio.fs.UnixException.translateToIOException(UnixException.java:86)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:102)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:107)
        at sun.nio.fs.UnixFileSystemProvider.newByteChannel(UnixFileSystemProvider.java:214)
        at java.nio.file.Files.newByteChannel(Files.java:361)
        at java.nio.file.Files.newByteChannel(Files.java:407)
        at java.nio.file.spi.FileSystemProvider.newInputStream(FileSystemProvider.java:384)
        at java.nio.file.Files.newInputStream(Files.java:152)
        at org.apache.hadoop.security.ssl.ReloadingX509TrustManager.loadTrustManager(ReloadingX509TrustManager.java:131)
        at org.apache.hadoop.security.ssl.ReloadingX509TrustManager.<init>(ReloadingX509TrustManager.java:79)
        at org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory.createTrustManagersFromConfiguration(FileBasedKeyStoresFactory.java:131)
        at org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory.init(FileBasedKeyStoresFactory.java:292)
        at org.apache.hadoop.security.ssl.SSLFactory.init(SSLFactory.java:180)
        at org.apache.hadoop.hdfs.web.SSLConnectionConfigurator.<init>(SSLConnectionConfigurator.java:50)
        at org.apache.hadoop.hdfs.web.URLConnectionFactory.getSSLConnectionConfiguration(URLConnectionFactory.java:100)
        at org.apache.hadoop.hdfs.web.URLConnectionFactory.newDefaultURLConnectionFactory(URLConnectionFactory.java:91)
        at org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager.<init>(QuorumJournalManager.java:193)
        at org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager.<init>(QuorumJournalManager.java:126)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
        at org.apache.hadoop.hdfs.server.namenode.FSEditLog.createJournal(FSEditLog.java:1864)
        at org.apache.hadoop.hdfs.server.namenode.FSEditLog.initJournals(FSEditLog.java:299)
        at org.apache.hadoop.hdfs.server.namenode.FSEditLog.initJournalsForWrite(FSEditLog.java:264)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.copyEditLogSegmentsToSharedDir(NameNode.java:1438)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.initializeSharedEdits(NameNode.java:1402)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.createNameNode(NameNode.java:1752)
        at org.apache.hadoop.hdfs.server.namenode.NameNode.main(NameNode.java:1841)
24/07/23 15:40:32 INFO client.QuorumJournalManager: Starting recovery process for unclosed journal segments...
24/07/23 15:40:32 INFO client.QuorumJournalManager: Successfully started new epoch 1
24/07/23 15:40:32 INFO namenode.RedundantEditLogInputStream: Fast-forwarding stream '/hadoop/hdfs/namenode/current/edits_0000000000000104926-0000000000000105695' to transaction ID 104926
24/07/23 15:40:32 INFO namenode.FSEditLog: Started a new log segment at txid 104926
24/07/23 15:40:32 INFO namenode.FSEditLog: Starting log segment at 104926
24/07/23 15:40:33 INFO namenode.FSEditLog: Ending log segment 104926, 105695
24/07/23 15:40:33 INFO namenode.FSEditLog: logSyncAll toSyncToTxId=105695 lastSyncedTxid=105695 mostRecentTxid=105695
24/07/23 15:40:33 INFO namenode.FSEditLog: Done logSyncAll lastWrittenTxId=105695 lastSyncedTxid=105695 mostRecentTxid=105695
24/07/23 15:40:33 INFO namenode.FSEditLog: Number of transactions: 770 Total time for transactions(ms): 28 Number of transactions batched in Syncs: 0 Number of syncs: 1 SyncTimes(ms): 16
24/07/23 15:40:33 INFO namenode.NameNode: SHUTDOWN_MSG:
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at mron-DN04/192.168.0.4
************************************************************/
[root@mron-dn04 conf]#
```
- 이 다음에, zookeeper server 와 namenode service 가 차례대로 start 되야 하는데. (계속 뺑뺑이...)
- 원래는 start 되는 시간 오래 걸리지 않음.
- zookeeper server 가 시작되지 못하는 것으로 추측됨. 


### namenode Enable HA 구성 시작(2번째 시도)
- 전체 패키지 삭제, ambari db 삭제후 다시 설치후, HA 구성 재시도
- 참고  
https://www.hadooplessons.info/2017/12/enabling-namenode-ha-using-apache-ambari-hdpca.html#google_vignette  
https://www.youtube.com/watch?v=TCdxK8C1Nn0  
https://docs.hortonworks.com/HDPDocuments/Ambari-2.7.3.0/managing-high-availablility/content/amb_configuring_namenode_high_availability.html   -> 이 주소는 없어짐.

- dn04에서 command 실행   
참고 사이트를 보고 진행도중. namenode 서버에서 직접 실행해야하는 콘솔 command가 있다. 그대로 실행하면 next 버튼 활성화됨.  
```
sudo su hdfs -l -c 'hdfs dfsadmin -safemode enter'

sudo su hdfs -l -c 'hdfs dfsadmin -saveNamespace'
```

- 설치 진행.  
약간 시간이 걸린다. (5분?) 충분히 기다려라. 
	- 모든 서비스 중지  
	- 지정된 호스트에 추가 네임노드 설치  
	- 지정된 호스트에 저널 노드를 설치합니다.  
	- Namdenode HA에 필요한 속성으로 구성을 수정합니다.  
	- 저널 노드 시작  
	- 2차 네임노드 비활성화  
	
- dn04에서 다시 command 실행.
```
sudo su hdfs -l -c 'hdfs namenode -initializeSharedEdits'

# 오류발생
24/07/25 16:48:56 WARN web.URLConnectionFactory: Cannot load customized ssl related configuration. Fallback to system-generic settings.
java.nio.file.NoSuchFileException: /etc/security/clientKeys/all.jks

```

- key 설정 안하는걸로 설정 변경 (혹시모르니, 4개 호스트 모두에게 반영하자. )
```
sudo su hdfs -l -c 'hdfs getconf -confKey hadoop.security.ssl.enabled'
실행시, 키가 없다고 나오거나, true 면 false로 만들어 주자

vi /etc/hadoop/conf/hdfs-site.xml
# 추가
    <property>
      <name>hadoop.security.ssl.enabled</name>
      <value>false</value>
    </property>
```
- 다시 확인  
```
[root@mron-dn04 conf]# sudo su hdfs -l -c 'hdfs getconf -confKey hadoop.security.ssl.enabled'
false

```

- 다시 command 실행.
```
sudo su hdfs -l -c 'hdfs namenode -initializeSharedEdits'


# 아래 오류는 다시 나오지만...
java.nio.file.NoSuchFileException: /etc/security/clientKeys/all.jks
        at sun.nio.fs.UnixException.translateToIOException(UnixException.java:86)

# 아래와 같이 포멧변경 하겠냐고 나옴. 
        at org.apache.hadoop.hdfs.server.namenode.NameNode.main(NameNode.java:1841)
Re-format filesystem in QJM to [192.168.0.1:8485, 192.168.0.3:8485, 192.168.0.4:8485] ? (Y or N)  Y ---> 했는데 오류.


# 아래 오류 메세지
24/07/25 17:04:42 ERROR namenode.NameNode: Could not initialize shared edits dir
org.apache.hadoop.hdfs.qjournal.client.QuorumException: Could not format one or more JournalNodes. 1 exceptions thrown:
192.168.0.4:8485: Directory /hadoop/hdfs/journal/isbig is in an inconsistent state: Can't format the storage directory because the current directory is not empty.

# 혹시나 싶어 계정 바꿔봐도 오류 동일함.
[root@mron-dn04 conf]# su hdfs
[hdfs@mron-dn04 conf]$ sudo su hdfs -l -c 'hdfs namenode -initializeSharedEdits'
```

- 디렉토리가 비어있지 않다고 해서. 비워보자.(이건 hdfs의 디렉토리가 아닌, os의 디렉토리이다. 그래서 모든 journal node에서 이걸 수행하자.) 
```
[hdfs@mron-dn04 conf]$ sudo su hdfs -c "rm -rf /hadoop/hdfs/journal/isbig/*"

[hdfs@mron-dn04 conf]$ sudo su hdfs -c "ls /hadoop/hdfs/journal/isbig"
[hdfs@mron-dn04 conf]$
# 3개 호스트 모두 비웠음. 
```

- 명령 다시 실행. (dn04)
```
sudo su hdfs -l -c 'hdfs namenode -initializeSharedEdits'
```

- 성공인지 오류인지 모르겠음 (결국 7/23에 도달했던 과정까지 다시 온건데.)
```
# 이 워닝은 계속 나오지만,
24/07/25 17:23:38 WARN web.URLConnectionFactory: Cannot load customized ssl related configuration. Fallback to system-generic settings.
java.nio.file.NoSuchFileException: /etc/security/clientKeys/all.jks

# 아래처럼 메세지 나옴. 
24/07/25 17:23:38 INFO client.QuorumJournalManager: Starting recovery process for unclosed journal segments...
24/07/25 17:23:38 INFO client.QuorumJournalManager: Successfully started new epoch 1                             --->  성공인건가?
24/07/25 17:23:38 INFO namenode.RedundantEditLogInputStream: Fast-forwarding stream '/hadoop/hdfs/namenode/current/edits_0000000000000108503-0000000000000108503' to transaction ID 108503
24/07/25 17:23:38 INFO namenode.FSEditLog: Started a new log segment at txid 108503
24/07/25 17:23:38 INFO namenode.FSEditLog: Starting log segment at 108503
24/07/25 17:23:38 INFO namenode.FSEditLog: Ending log segment 108503, 108503
24/07/25 17:23:38 INFO namenode.FSEditLog: logSyncAll toSyncToTxId=108503 lastSyncedTxid=108503 mostRecentTxid=108503
24/07/25 17:23:38 INFO namenode.FSEditLog: Done logSyncAll lastWrittenTxId=108503 lastSyncedTxid=108503 mostRecentTxid=108503
24/07/25 17:23:38 INFO namenode.FSEditLog: Number of transactions: 1 Total time for transactions(ms): 1 Number of transactions batched in Syncs: 0 Number of syncs: 1 SyncTimes(ms): 44
24/07/25 17:23:38 INFO namenode.NameNode: SHUTDOWN_MSG:
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at mron-DN04/192.168.0.4
************************************************************/
```
- 여기까지는 all.jks 문제를 제외하고, 결과는 성공 메세지와 동일하게 출력되었다. 

- 일단 next 해 보겠다.   
  start Components 과정.... (7/25 17:25 시작.) 오래걸린다. 3년 기다릴 각오하고 기다리자. 중단하면 x된다.  
> Please wait while NameNode HA is being deployed.  
  이 메세지로 하루종일 뺑뺑이...지난번에 여기서 중단했었다.. 결국 '엣지 오브 투모로우'  
- 원래는 오래 걸리지 않음.   

### 뒤늦게 다른 ssh 세션에서 all.jks 파일 생성. (4개 호스트 모두 실행했음)
```
[root@mron-en01 etc]# mkdir -p /etc/security/clientKeys/

[root@mron-en01 ~]# keytool -genkeypair -alias hadoop -keyalg RSA -keystore /etc/security/clientKeys/all.jks -keysize 2048
Enter keystore password: 123qwe
Re-enter new password:
What is your first and last name?
  [Unknown]:  ambari
What is the name of your organizational unit?
  [Unknown]:  ambari-unit
What is the name of your organization?
  [Unknown]:  ambari-org
What is the name of your City or Locality?
  [Unknown]:  Seoul
What is the name of your State or Province?
  [Unknown]:  Pangyo
What is the two-letter country code for this unit?
  [Unknown]:  KR
Is CN=ambari, OU=ambari-unit, O=ambari-org, L=Seoul, ST=Pangyo, C=KR correct?
  [no]:  Yes

Enter key password for <hadoop>
        (RETURN if same as keystore password):

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore /etc/security/clientKeys/all.jks -destkeystore /etc/security/clientKeys/all.jks -deststoretype pkcs12".
[root@mron-en01 ~]#
```

- 다시 초기화 명령 실행해봄. 
```
sudo su hdfs -l -c 'hdfs namenode -initializeSharedEdits'
```
 
- 이번에는 all.jks 오류는 발생하지 않았지만. 이러고 끝났다.  
```
STARTUP_MSG:   build = https://github.com/apache/bigtop.git -r 4a34226ec01a894fd96cc00c052d96e61673c60e; compiled by 'jenkins' on 2023-08-02T05:53Z
STARTUP_MSG:   java = 1.8.0_112
************************************************************/
24/07/26 09:58:45 INFO namenode.NameNode: registered UNIX signal handlers for [TERM, HUP, INT]
24/07/26 09:58:45 INFO namenode.NameNode: createNameNode [-initializeSharedEdits]
24/07/26 09:58:45 ERROR namenode.NameNode: No shared edits directory configured for namespace null namenode null
24/07/26 09:58:45 INFO namenode.NameNode: SHUTDOWN_MSG:
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at mron-EN01/192.168.0.2
************************************************************/
```
- 오히려 이게 잘못된 실패 메세지임. 

- 결국 ambari web 에서 계속 돌고있던 뺑뺑이는 그대로임.
- 여기서 또 멈추면, ambari web 에서는 시스템을 지울수 없는데... 하..  

- ambari-shell 설치하여 shell에서 zookeeper 실행해 보려고 했는데,  
  ambari-shell 을 또 빌드해야 되고, git에서 받은 소스 빌드 하려는데 소스는 sequenceiq에 의존하고 있고,
  sequenceiq는 페이지 폐쇄되서 링크도 없고, 
- 그래서 포기. 여기서 다시 중단 해야 할듯.
- HA 구성 skip 해라 (J)


### 포기하는 심정으로 snamenode를 move 해보았다.
- 최초 설치 dn04(namenode), dn01(snamenode) -> HA 구성 dn04(nn1), dn01(nn2)  , dn01(snamenode 제거)
- 그런데, 원인은 모르게 실패하였다. 
- dn01의  snamenode 서비스를 제거하기 위해 ambari web에서 move 해봤음 dn01 -> nn01로
- 결과적으로는 nn01로 이동 실패.
- All commponent 재시작 해봤는데, HA구성한 namenode 2개 (dn04, dn01) 모두 정상 기동 되었음. 
- 그외에 다른 서비스들도 ambari web에서 수동으로 start 해봤더니, 일부 정상 기동 되었음.



### tez web-ui 설치방법 (설치는 ambari 계정으로 한다.)
- ambari 2.7 이후부터는 tez web-ui가 패키지에서 제외 되었다.
- tomcat과 tez web-ui 를 별도 설치해야 한다.  
- 반드시 resourcemanager, timelineserver가 설치된 호스트에 tez web-ui를 설치해라.
```
현재 isbig 환경에서는 ambari 기반으로 구성했더니, 
timelineserver가 패키지에 없었고, 그래서 별도로 
# yarn timelineserver &
명령어로 구동한다.

yarn-site.xml 파일은 timelineserver와 ambari의 yarn 설정에서 공유되는데, 
ambari에서 이 파일을 좀 이상하게 관리를 해서,
timeline-server 관련한 몇가지 설정이 'localhost'로 고정되어, 수정할수도 없고, 
yarn-site.xml 파일을 강제로 수정해도, 다시 'localhost'로 원복되는 현상이 있었다.

timeline-server가 127.0.0.1:8188 로 서비스 되니,
외부 서버에서 접근이 안되더라. 미치겠음.  
그래서, tez web-ui도 동일 서버에 설치하고 서비스를 제공받도록 한다.
```

- 아래부터는 tez web-ui 설치  
참고 https://tez.apache.org/tez-ui.html  
참고 https://www.programmersought.com/article/68765041730/  

- 문서 해석.
```
1. Tez는 yarn timeline server를 application history 저장소로 사용한다. 
2. Tez는 대부분의 lifecycle 정보를 이 히스토리 저장소에 저장한다. 
3. tez-site.xml에 설정추가.
tez.history.logging.service.class = org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService
```


- tomcat download 및 압축해제
```
[ambari@mron-en01 ~]$ wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.93/bin/apache-tomcat-9.0.93.tar.gz
[ambari@mron-en01 ~]$ tar -zxvf apache-tomcat-9.0.93.tar.gz

[ambari@mron-en01 ~]$ sudo mv apache-tomcat-9.0.93 /opt/
[ambari@mron-en01 ~]$ cd /opt/
[ambari@mron-en01 ~]$ sudo ln -s ./apache-tomcat-9.0.93 tomcat

[ambari@mron-en01 ~]$ sudo chown ambari:ambari -R /opt/tomcat
[ambari@mron-en01 ~]$ sudo chown ambari:ambari -R /opt/apache-tomcat-9.0.93

[ambari@mron-en01 ~]$ sudo chmod 777 /opt/tomcat/bin/startup.sh
[ambari@mron-en01 ~]$ sudo chmod 777 /opt/tomcat/bin/shutdown.sh

```

- 포트변경 (8080 -> 18081)
```
[ambari@mron-en01 ~]$ sudo vi /opt/tomcat/conf/server.xml
    <Connector port="18081" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               maxParameterCount="1000"
               />

```

- tez-ui 다운로드
확인 : https://repository.apache.org/content/repositories/releases/org/apache/tez/tez-ui/

```
[ambari@mron-en01 ~]$ cd 
[ambari@mron-en01 ~]$ wget https://repository.apache.org/content/repositories/releases/org/apache/tez/tez-ui/0.10.1/tez-ui-0.10.1.war

# tomcat이 실행된 상태에서 war 파일을 복사. 자동으로 압축 풀림.
[ambari@mron-en01 ~]$ cp ./tez-ui-0.10.1.war /opt/tomcat/webapps/tez-ui.war

```

- tez-ui.war 압축 풀기위하여 잠시 startup
```
[ambari@mron-dn04 ~]$ /opt/tomcat/bin/startup.sh
```


- 설정변경
configs.js 파일 열어서 주석 제거 및 호스트, 포트 설정
```
[ambari@mron-en01 ~]$ vi /opt/tomcat/webapps/tez-ui/config/configs.js

ENV = {
  hosts: {
    /*
     * Timeline Server Address:
     * By default TEZ UI looks for timeline server at http://localhost:8188, uncomment and change
     * the following value for pointing to a different address.
     */
    timeline: "http://mron-dn04:8188",     ---> 주석제거 및 서버 변경

    /*
     * Resource Manager Address:
     * By default RM REST APIs are expected to be at http://localhost:8088, uncomment and change
     * the following value to point to a different address.
     */
    rm: "http://mron-dn04:8088",       ---> 주석제거

    /*
     * Resource Manager Web Proxy Address:
     * Optional - By default, value configured as RM host will be taken as proxy address
     * Use this configuration when RM web proxy is configured at a different address than RM.
     */
    //rmProxy: "http://mron-dn04:8088",
  },

```

- ambari tez config 에서 아래 내용 추가 하면 tez-site.xml에 추가됨.
```
tez.history.logging.service.class = org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService
tez.tez-ui.history-url.base = http://mron-dn04:18081/tez-ui/
```

- ambari yarn conf 에서 아래 내용 추가. yarn-site.xml에 추가됨.
```
yarn.timeline-service.enabled = true
yarn.timeline-service.hostname = mron-dn04
yarn.timeline-service.http-cross-origin.enabled = true
yarn.resourcemanager.system-metrics-publisher.enabled = true
```

- tez-ui tomcat on/off
```
# 켜기
/opt/tomcat/bin/startup.sh
# 끄기
/opt/tomcat/bin/shutdown.sh
```

- ambari tez만 재시작.

- 웹 브라우저에서 http://mron-dno4:18081/tez-ui  연결하면 tez-DAG 웹페이지 나온다.
  - 그러나, timeline server 미설지 이므로. loading... 으로만 나옴.
  - 페이지 하단에 timeline 연결안된다는 경고 나옴. 



### yarn timeline-server 구동.

- 이 경로로 실행하는 방법 있음. 
```
% /usr/lib/hadoop-yarn/sbin/yarn-daemon.sh start timelineserver

# 오류발생.
ERROR: Cannot execute /usr/lib/hadoop-yarn/sbin/../libexec/yarn-config.sh.

# 환경변수 설정후 재실행
% HADOOP_HOME=/usr/lib/hadoop
% /usr/lib/hadoop-yarn/sbin/yarn-daemon.sh start timelineserver


# 같은오류발생.
ERROR: Cannot execute /usr/lib/hadoop-yarn/sbin/../libexec/yarn-config.sh.

```

- 필요한 경로에 softlink 생성하고 다시 실행해보자. (여기 중요!!!!!)
```
[root@mron-dn04 hadoop-yarn]# cd /usr/lib/hadoop-yarn
[root@mron-dn04 hadoop-yarn]# ln -s /usr/lib/hadoop/libexec libexec

# 다시실행
[root@mron-dn04 hadoop-yarn]# /usr/lib/hadoop-yarn/sbin/yarn-daemon.sh start timelineserver

# 실행성공.
```


- ambari에는 timeline-server 관련 설정이 없어서. 수동 실행. 끄기는 kill -9
```
[ambari@mron-dn04 ~]# yarn timelineserver &
[1] 11827
[root@mron-dn04 ~]# 24/08/07 14:02:55 INFO applicationhistoryservice.ApplicationHistoryServer: STARTUP_MSG:
/************************************************************
STARTUP_MSG: Starting ApplicationHistoryServer
STARTUP_MSG:   host = mron-DN04/192.168.0.4
STARTUP_MSG:   args = []
STARTUP_MSG:   version = 3.3.5
STARTUP_MSG:   classpath = /etc/hadoop/conf:/usr
``` 

- 그런데 이상하게, timelineserver를 실행하면 나오는 pid가 11827 applicationHistoryServer의 pid와 같다. 
```
[root@mron-dn04 ~]# jps
14504 Jps
16717 NameNode
9262 JobHistoryServer
11827 ApplicationHistoryServer
2199 ResourceManager
```


- 작동 확인 및 포트 확인
```
# pid 확인
[root@mron-dn04 ~]# ps -ef | grep timeline
root     11827 25845  2 11:02 pts/0    00:00:19 /usr/jdk64/jdk1.8.0_112/bin/java -Dproc_timelineserver 
...

# port 확인
[root@mron-dn04 ~]# netstat -plnt | grep 11827
tcp        0      0 127.0.0.1:10200         0.0.0.0:*               LISTEN      11827/java
tcp        0      0 127.0.0.1:8188          0.0.0.0:*               LISTEN      11827/java
```
- 10200, 8188 모두 127.0.0.1 로 실행되고 있다. (그래서 외부접근이 안됨.)

- 그런데 또 죽이면 timelineserver 가 죽었다고 나옴. 아놔..
```
[root@mron-dn04 ~]# kill -9 11827
[root@mron-dn04 ~]# 
16717 NameNode
9262 JobHistoryServer
2199 ResourceManager
15580 Jps
[1]+  Killed                  yarn timelineserver
```


- yarn-site.xml 수정.  (왜 localhost 인거지?)
- 이거 수정해도 ambari 에서 yarn 재시작하면 다시 localhost 됨. 미침.
```
# 파일 확인
[root@mron-dn04 /]# find . -type f -name "*-site.xml" | xargs grep -Hi -a3 "yarn.timeline-service.address"
./etc/hadoop/conf.empty/yarn-site.xml-    <property>
./etc/hadoop/conf.empty/yarn-site.xml:      <name>yarn.timeline-service.address</name>
./etc/hadoop/conf.empty/yarn-site.xml-      <value>localhost:10200</value>
./etc/hadoop/conf.empty/yarn-site.xml-    </property>

# 수정
[root@mron-dn04 /]# vi ./etc/hadoop/conf.empty/yarn-site.xml

	<property>
	  <name>yarn.timeline-service.address</name>
	  <value>mron-dn04:10200</value>
	</property>

    <property>
      <name>yarn.timeline-service.webapp.address</name>
      <value>mron-dn04:8188</value>
    </property>

    <property>
      <name>yarn.timeline-service.webapp.https.address</name>
      <value>mron-dn04:8190</value>
    </property>	

```


- timelineserver가 localhost 로 기동되는것 같다. 
- dn04 에서는 응답이 오는데, en01 에서는 응답 없음. 
- 위에서 언급한 127.0.0.1:8188 문제. (이 응답이 timeline에서 리턴하는것은 확실하다. 프로세스 죽이면 응답 없음.)
```
# 응답있음. 호스트 자체 호출. 
[ambari@mron-dn04 config]$ curl http://localhost:8188/ws/v1/timeline/
{"About":"Timeline API","timeline-service-version":"3.3.5","timeline-service-build-version":"3.3.5 from 4a34226ec01a894fd96cc00c052d96e61673c60e by jenkins source checksum b1afcccc971a0c8843187d1443dadeb","timeline-service-version-built-on":"2023-08-02T06:00Z","hadoop-version":"3.3.5","hadoop-build-version":"3.3.5 from 4a34226ec01a894fd96cc00c052d96e61673c60e by jenkins source checksum 6bbd9afcf4838a0eb12a5f189e9bd7","hadoop-version-built-on":"2023-08-02T05:53Z"}

# 응답없음.
[root@mron-en01 ~]#  curl http://localhost:8188/ws/v1/timeline/
curl: (7) Failed connect to localhost:8188; Connection refused

```

- 아래 설정을 모두 0.0.0.0으로 변경 하고, 재시작 했더니, 외부에서 접속 가능해짐. 
- 그러나, ambari에서 재시작 할 경우 다시 localhost 로 원복될 가능성 있음.
```
# 검색
[root@mron-dn04 /]# find . -type f -name "*-site.xml" | xargs grep -Hi "localhost"
./etc/hadoop/conf.empty/yarn-site.xml:      <value>localhost:10200</value>
./etc/hadoop/conf.empty/yarn-site.xml:      <value>localhost:8188</value>
./etc/hadoop/conf.empty/yarn-site.xml:      <value>localhost:8190</value>
[root@mron-dn04 /]# vi /etc/hadoop/conf.empty/yarn-site.xml

# 모두 배포
[root@mron-dn04 conf]# sudo scp yarn-site.xml root@mron-nn01:/etc/hadoop/conf/
[root@mron-dn04 conf]# sudo scp yarn-site.xml root@mron-en01:/etc/hadoop/conf/
[root@mron-dn04 conf]# sudo scp yarn-site.xml root@mron-dn01:/etc/hadoop/conf/
```

- en01 에서 접속 가능해짐.
```
[ambari@mron-en01 root]$ curl http://mron-dn04:8188/ws/v1/timeline/
{"About":"Timeline API","timeline-service-version":"3.3.5","timeline-service-build-version":"3.3.5 from 4a34226ec01a894fd96cc00c052d96e61673c60e by jenkins source checksum b1afcccc971a0c8843187d1443dadeb","timeline-service-version-built-on":"2023-08-02T06:00Z","hadoop-version":"3.3.5","hadoop-build-version":"3.3.5 from 4a34226ec01a894fd96cc00c052d96e61673c60e by jenkins source checksum 6bbd9afcf4838a0eb12a5f189e9bd7","hadoop-version-built-on":"2023-08-02T05:53Z"}

```

### timeline-server 오류 확인
- tez url 호출시 오류. http://mron-dn04:18081/tez-ui/ 

- 크롭 개발자 도구로 확인해본 결과. 아래 링크를 호출하고자 하나, 연결 안됨.
http://localhost:8188/ws/v1/timeline/TEZ_DAG_ID?limit=11&_=1723428161187

- 호출경로가 localhost 인데, 이러면 당연히 연결 안될것이고. 이걸 mron-dn04로 바꿔야 하는데, 어디서 바꾸지?

- yarn 오류 메세지 확인. http://mron-dn04:8088/logs/hadoop-yarn-resourcemanager-mron-dn04.out
```
# 1
Aug 12, 2024 9:46:21 AM com.sun.jersey.api.core.servlet.WebAppResourceConfig init
INFO: Scanning for root resource and provider classes in the Web app resource paths:
  /WEB-INF/lib
  /WEB-INF/classes
Aug 12, 2024 9:46:21 AM com.sun.jersey.server.impl.application.WebApplicationImpl _initiate
INFO: Initiating Jersey application, version 'Jersey: 1.19.4 05/24/2017 03:20 PM'
Aug 12, 2024 9:46:21 AM com.sun.jersey.server.impl.application.RootResourceUriRules <init>
SEVERE: The ResourceConfig instance does not contain any root resource classes.
24/08/12 09:46:21 WARN ContextHandler.cluster: unavailable
com.sun.jersey.api.container.ContainerException: The ResourceConfig instance does not contain any root resource classes.
        at com.sun.jersey.server.impl.application.RootResourceUriRules.<init>(RootResourceUriRules.java:99)
        at com.sun.jersey.server.impl.application.WebApplicationImpl._initiate(WebApplicationImpl.java:1359)

# 2		
24/08/12 09:46:21 WARN server.HttpChannel: /app/v1/services
javax.servlet.ServletException: javax.servlet.ServletException: 
API-Service==com.sun.jersey.spi.container.servlet.ServletContainer@8899a5c2{jsp=null,order=-1,inst=true,async=true,src=EMBEDDED:null,STARTED}
        at org.eclipse.jetty.server.handler.HandlerCollection.handle(HandlerCollection.java:162)	
		
# 3		
Caused by: com.sun.jersey.api.container.ContainerException: The ResourceConfig instance does not contain any root resource classes.
        at com.sun.jersey.server.impl.application.RootResourceUriRules.<init>(RootResourceUriRules.java:99)
```


- jersey 위치 확인
```
[root@mron-dn04 /]# find . -type f -name "*jersey*"
./usr/lib/hadoop/lib/jersey-core-1.19.4.jar
./usr/lib/hadoop/lib/jersey-json-1.20.jar
./usr/lib/hadoop/lib/jersey-server-1.19.4.jar
./usr/lib/hadoop/lib/jersey-servlet-1.19.4.jar
./usr/lib/hive/lib/jersey-hk2-2.34.jar
./usr/lib/hive-hcatalog/share/webhcat/svr/lib/jersey-core-1.19.4.jar
./usr/lib/hive-hcatalog/share/webhcat/svr/lib/jersey-json-1.19.4.jar
./usr/lib/hive-hcatalog/share/webhcat/svr/lib/jersey-servlet-1.19.4.jar
./usr/lib/zookeeper/contrib/rest/lib/jersey-client-1.1.5.1.jar
./usr/lib/zookeeper/contrib/rest/lib/jersey-core-1.1.5.1.jar
./usr/lib/zookeeper/contrib/rest/lib/jersey-json-1.1.5.1.jar
./usr/lib/zookeeper/contrib/rest/lib/jersey-server-1.1.5.1.jar
./usr/lib/hadoop-yarn/lib/jersey-client-1.19.4.jar
./usr/lib/hadoop-yarn/lib/jersey-guice-1.19.4.jar
./usr/lib/hadoop-hdfs/lib/jersey-core-1.19.4.jar
./usr/lib/hadoop-hdfs/lib/jersey-json-1.20.jar
./usr/lib/hadoop-hdfs/lib/jersey-server-1.19.4.jar
./usr/lib/hadoop-hdfs/lib/jersey-servlet-1.19.4.jar
./usr/lib/tez/lib/jersey-client-1.19.jar
./usr/lib/tez/lib/jersey-json-1.19.jar
./usr/lib/hbase/hbase-shaded-jersey-4.1.1.jar
./usr/lib/hbase/lib/hbase-shaded-jersey-4.1.1.jar
./usr/lib/hbase/lib/jersey-json-1.20.jar

```





### 참고. spark thrift 서버 구동 안될때. (실패)
- 오류 메세지에 따라. 파일 copy.(4개 호스트에 모두 scp 한다)
```
[root@mron-en01 ~]# find /var/lib/ambari-server/resources/ -name alert_spark_thrift_port.py
/var/lib/ambari-server/resources/mpacks/bgtp-ambari-mpack-1.0.0.0-SNAPSHOT/stacks/BGTP/1.0/services/SPARK/package/scripts/alerts/alert_spark_thrift_port.py
/var/lib/ambari-server/resources/common-services/SPARK/1.2.1/package/scripts/alerts/alert_spark_thrift_port.py
[root@mron-en01 ~]# scp /var/lib/ambari-server/resources/mpacks/bgtp-ambari-mpack-1.0.0.0-SNAPSHOT/stacks/BGTP/1.0/services/SPARK/package/scripts/alerts/alert_spark_thrift_port.py root@mron-en01:/var/lib/ambari-agent/cache/host_scripts/
alert_spark_thrift_port.py   

sudo chown -R ambari:ambari /var/lib/ambari-agent/cache/
sudo chmod -R 755 /var/lib/ambari-agent/cache/      
```
- ambari web ui 에서 다시 thrift server start 해 본다.




### 참고. 설정 값 WARNING 변경 (시도했으나, 결국은 기본값으로)
```
* Services > MapReduce2 > Configs 
- General
Map Memory    : 18432 MB (기본값) -> Values greater than 5120MB are not recommended
Reduce Memory : 18432 MB (기본값) -> Values greater than 5120MB are not recommended
- Advanced mapred-site
AppMaster Memory : 18432 MB (기본값) -> Values greater than 5120MB are not recommended

그래서 바꾸고 저장했더니, 

- General
Map Memory    : 5120  MB (권장값) -> Value is less than the recommended default of 18432
Reduce Memory : 5120  MB (권장값) -> Value is less than the recommended default of 18432
- Advanced mapred-site
AppMaster Memory : 5120 MB (기본값) -> Value is less than the recommended default of 18432

ㅋㅋㅋ 어쩌라고!!

* Services > YARN > Configs
- Scheduler
Maximum Container Size (Memory) : 55296 MB (기본값) -> Values greater than 5120MB are not recommended
Minimum Container Size (Memory) : 18432 MB (기본값) -> Values greater than 5120MB are not recommended

변경

Maximum Container Size (Memory) : 18432 MB (기본값의 1/3) -> Value is less than the recommended default of 55296
Minimum Container Size (Memory) :  5120 MB (권장값) -> Value is less than the recommended default of 18432


역시나 어쩌라고!!
```
- 결국은 그냥 기본값 그대로 유지.



### 참고. ambari에서 bigtop으로 설치한 appl 버전 입력기능.
- Cluster Admin > Stack and Versions 화면 Versions 탭에서   
"Manage Versions" 버튼 누르면 Ambari Admin 모드로 들어감.
- 오른쪽 "Register" 버튼 눌러서 version 등록할 수 있는데...





### 참고. journalnode 별도 설치 (이짓을 먼저 했는데, 이거 아님. Ambari에서 제공되는 기능있음.Enable Namenode HA) - 삽질한건 그냥 참고.
- journalnode 를 설치하는 것은. 단순히 프로그램 설치가 아니라. namenode를 HA 구성으로 전환하는것이다. 
- 개념과 설정이 복잡하다. 
- journalnode 는 오류를 빠르게 복구하기 위한, namenode 운영 데이터 값을 반복적으로 상시 보관하고, 긴급하게 active/standby namenode를 전환한다. 
- 3개 이상 호스트에 설치해야함.
- hdfs-site.xml 에 설정추가 (ambari HDFS config에서 추가가능)
- 모든 호스트에 변경된 hdfs-site.xml 반영
```
# vi /etc/hadoop/conf/hdfs-site.xml

<configuration>
  <!-- NameService ID 설정 -->
  <property>
    <name>dfs.nameservices</name>
    <value>mycluster</value>
  </property>

  <!-- NameNode ID 설정 -->
  <property>
    <name>dfs.ha.namenodes.mycluster</name>
    <value>nn1,nn2</value>
  </property>

  <!-- NameNode 호스트와 포트 설정 -->
  <property>
    <name>dfs.namenode.rpc-address.mycluster.nn1</name>
    <value>nn1-host:8020</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.mycluster.nn2</name>
    <value>nn2-host:8020</value>
  </property>

  <!-- NameNode HTTP 주소 설정 -->
  <property>
    <name>dfs.namenode.http-address.mycluster.nn1</name>
    <value>nn1-host:50070</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.mycluster.nn2</name>
    <value>nn2-host:50070</value>
  </property>

  <!-- Shared edits directory 설정 -->
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://journalnode1:8485;journalnode2:8485;journalnode3:8485/mycluster</value>
  </property>

  <!-- JournalNode 디렉토리 설정 -->
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/path/to/journalnode/edits</value>
  </property>

  <!-- HA 자동 페일오버 설정 -->
  <property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>

  <!-- Zookeeper 설정 -->
  <property>
    <name>ha.zookeeper.quorum</name>
    <value>zk1:2181,zk2:2181,zk3:2181</value>
  </property>
</configuration>

```

- journalnode 시작
- 설치를 결정한 3대의 host에서 실행
```
# root 계정으로
/usr/lib/hadoop/sbin/hadoop-daemon.sh start journalnode

WARNING: Use of this script to start HDFS daemons is deprecated.
WARNING: Attempting to execute replacement "hdfs --daemon start" instead.
WARNING: /var/run/hadoop/root does not exist. Creating.
WARNING: /var/log/hadoop/root does not exist. Creating.
```
- 3.1 nameNode 포맷
- 한 번만 실행해야 합니다. NameNode가 이미 포맷된 경우 이 단계를 건너뜁니다.
```
hdfs namenode -format
```

- 3.2 JournalNode와 연동된 NameNode 초기화
- 한 NameNode에서만 실행합니다:
```
[root@mron-dn04 ~]# hdfs namenode -initializeSharedEdits
```

- 3.3 Active/Standby NameNode 시작
- 각 NameNode에서 다음 명령어로 NameNode를 시작합니다 (ambari에서 restart)
```
/usr/lib/hadoop/sbin/hadoop-daemon.sh start namenode

# 실행 안됐네.
[root@mron-en01 pam.d]# /usr/lib/hadoop/sbin/hadoop-daemon.sh start namenode
WARNING: Use of this script to start HDFS daemons is deprecated.
WARNING: Attempting to execute replacement "hdfs --daemon start" instead.
WARNING: HADOOP_NAMENODE_OPTS has been replaced by HDFS_NAMENODE_OPTS. Using value of HADOOP_NAMENODE_OPTS.
[root@mron-en01 pam.d]# ps -ef | grep namenode
root     17923 42947  0 09:53 pts/1    00:00:00 grep --color=auto namenode
[root@mron-en01 pam.d]#

```

- 3.4 Zookeeper와 Failover Controller(FEC) 시작
- 각 NameNode에서 다음 명령어로 Zookeeper Failover Controller를 시작합니다
```
# 이건 deprecated 되었단다. 
/usr/lib/hadoop/sbin/hadoop-daemon.sh start zkfc

# 이걸로 실행.
sudo hdfs --daemon start zkfc
# 오류발생
ERROR: Cannot set priority of zkfc process 10037
# 해결하려면 파일열어서 내용추가
sudo vi /etc/security/limits.conf
  hdfs  -  nofile  32768
  hdfs  -  nproc   32768
  
# pam_limits.so 내용추가 
cd /etc/pam.d
su, sshd, login 파일에 내용 추가
--------------------------
# add 20240726
session required pam_limits.so
--------------------------

# 결론. 동일오류 발생.

```

- 4. 확인
- NameNode와 JournalNode가 제대로 작동하는지 확인합니다.
- 4.1 HDFS 상태 확인
- HDFS 클러스터의 상태를 확인합니다:
```
hdfs dfsadmin -report
```

- 4.2 JournalNode 웹 UI 확인
- 각 JournalNode의 웹 UI에 접근하여 상태를 확인합니다:
```
http://<journalnode-hostname>:8480
```

- 위 단계들을 따르면 JournalNode가 설치되고 활성화되며, NameNode HA 구성이 완료됩니다. 이 과정을 통해 Hadoop 클러스터의 고가용성을 확보할 수 있습니다.





### 참고. t-cloudbiz 환경에서 웹서버 운영을 위한 사전조사
- 외부에서 확인. 3개 포트만 허용되었나 봄.
```
$> nmap mron-en01
Starting Nmap 7.80 ( https://nmap.org ) at 2024-07-15 14:23 KST
Nmap scan report for mron-en01 (175.126.58.202)
Host is up (0.011s latency).
rDNS record for 175.126.58.202: mron-EN01
Not shown: 997 filtered ports
PORT    STATE  SERVICE
22/tcp  open   ssh
80/tcp  closed http
443/tcp closed https

Nmap done: 1 IP address (1 host up) scanned in 4.97 seconds
```

- nginx 설치 (4개 호스트 모두 설치. 80 포트 서비스 확인- 정상 접속됨.)
```
# nginx 설치/on/off/상태확인.
yum install nginx
vi /usr/share/nginx/html/index.html
systemctl start nginx
systemctl status nginx
systemctl stop nginx

# nginx 설정. 
/etc/nginx/nginx.conf
```
- nginx를 설치하여 웹서버 접근 가능성 체크.  
80포트, 8080 포트 확인.  => 80 포트만 외부에서 접속 가능.





### 참고. t-cloudbiz 4개 호스트 ip주소 목록
```
[root@mron-dn04 ~]# ansible -i /etc/ansible/hosts all -m shell -a 'ifconfig | grep inet'
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see
details
mron-dn04 | CHANGED | rc=0 >>
        inet 175.126.58.204  netmask 255.255.255.224  broadcast 175.126.58.223
        inet6 fe80::a183:b95b:4042:bc16  prefixlen 64  scopeid 0x20<link>
        inet 192.168.0.4  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::a076:dcef:b375:554a  prefixlen 64  scopeid 0x20<link>
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
mron-en01 | CHANGED | rc=0 >>
        inet 175.126.58.202  netmask 255.255.255.224  broadcast 175.126.58.223
        inet6 fe80::167c:3b7c:6e9d:6adb  prefixlen 64  scopeid 0x20<link>
        inet 192.168.0.2  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::e466:96a4:d345:3505  prefixlen 64  scopeid 0x20<link>
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
mron-dn01 | CHANGED | rc=0 >>
        inet 175.126.58.203  netmask 255.255.255.224  broadcast 175.126.58.223
        inet6 fe80::4a86:cc65:ff63:112  prefixlen 64  scopeid 0x20<link>
        inet 192.168.0.3  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::8900:553e:e31c:9ddc  prefixlen 64  scopeid 0x20<link>
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        inet 192.168.56.1  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::800:27ff:fe00:0  prefixlen 64  scopeid 0x20<link>
mron-nn01 | CHANGED | rc=0 >>
        inet 175.126.58.201  netmask 255.255.255.224  broadcast 175.126.58.223
        inet6 fe80::d6b1:3978:fa14:74be  prefixlen 64  scopeid 0x20<link>
        inet 192.168.0.1  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::b7b0:fea7:a7a5:74fa  prefixlen 64  scopeid 0x20<link>
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
[root@mron-dn04 ~]#
```
