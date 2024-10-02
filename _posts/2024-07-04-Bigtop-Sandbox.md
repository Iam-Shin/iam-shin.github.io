
---
layout: post
title: "Bigtop Sandbox"
date: 2024-07-04 12:00:00 +0900
categories: [bigdata]
tags: [hadoop, bigtop, sandbox]
---

# bigtop sandbox 먼저 해 보자.  

- 참고1. bigtop sandbox 가이드 - 이거 따라함.  
[컨플루언스](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=70256303)  

- 참고2. bigtop 공식 다운로드  
[아파치 공식다운로드](https://bigtop.apache.org/download.html)  

- 참고3. 도커 bigtop/sandbox repo  
[도커 repo](https://hub.docker.com/r/bigtop/sandbox/tags/)  

- 참고4. Github bigtop공식 repo  
[깃헙](https://github.com/apache/bigtop/blob/master/README.md)

- 참고5. 아파치 Jira 공식 issue  
[지라](https://issues.apache.org/jira/projects/BIGTOP/issues/BIGTOP-3902?filter=allopenissues)


## 1. ubuntu에 docker 설치  
```
# 시스템 패키지 목록을 업데이트
sudo apt-get update

# 필수 패키지 설치
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# Docker의 공식 GPG 키를 추가
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Docker 저장소를 추가
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 패키지 목록을 다시 업데이트
sudo apt-get update

# Docker 엔진 설치
sudo apt-get install -y docker-ce

# Docker 서비스를 시작하고 부팅 시 자동으로 시작하도록 설정
sudo systemctl start docker
sudo systemctl enable docker
```


## 2. 빅탑 프로젝트 다운로드
[아파치 공식다운로드](https://bigtop.apache.org/download.html)  
여기 list 중에서.
3.1.1 까지는 sandbox가 있고, 
3.2.0 부터는 sandbox가 없다.  
가이드 문서는 bigtop-1.2.1 설치. 
나는 3.1.1 설치.
```
# wget https://archive.apache.org/dist/bigtop/bigtop-3.1.1/bigtop-3.1.1-project.tar.gz
# tar zxvf bigtop-3.1.1-project.tar.gz
# cd bigtop-3.1.1/docker/sandbox
```

## 3. 빌드
Build a Hadoop HDFS, Hadoop YARN, and Spark on YARN sandbox image
```
# ./build.sh -a bigtop -o ubuntu-22.04 -c "hdfs, yarn, spark"   <-- 내 utuntu 버전에 맞게 고쳐봄.
```

## 4. 일단 도커 컴포즈가 없다고 오류남. 
```
./build.sh: line 25: docker-compose: command not found
```

## 5. 그리고, README.md 내용을 봤더니,
> Currently supported OS list:  
>  * centos-6  
>  * debian-8  
>  * ubuntu-16.04   

아놔...

## 6. 우선. 도커 컴포즈 설치해 본다.
```
# 최신 버전 변수 설정
COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep -Po '"tag_name": "\K.*\d')

echo $COMPOSE_VERSION
v2.28.1

# Docker Compose 바이너리 다운로드
sudo curl -L "https://github.com/docker/compose/releases/download/$COMPOSE_VERSION/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 실행 권한 부여
sudo chmod +x /usr/local/bin/docker-compose

# 심볼릭 링크 생성 (선택 사항)
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# Docker Compose 버전 확인
docker-compose --version
```

## 7. 다시 빌드
```
# ./build.sh -a bigtop -o ubuntu-22.04 -c "hdfs, yarn, spark"

# 에러
failed to solve: bigtop/puppet:1.4.0-ubuntu-22.04: failed to resolve source metadata for docker.io/bigtop/puppet:1.4.0-ubuntu-22.04: docker.io/bigtop/puppet:1.4.0-ubuntu-22.04: not found

# 다시 
# ./build.sh -a bigtop -o ubuntu-16.04 -c "hdfs, yarn, spark"   <-- 버전 원래대로.

-------------------------------------------------
Image bigtop/sandbox:1.4.0-ubuntu-16.04_hdfs_yarn_spark built
-------------------------------------------------
# 성공
```

## 8. 그런데 가이드 문서 밑에 보니 
Change the repository of packages 이런게 있다... ㅎ
```
export REPO=http://repos.bigtop.apache.org/releases/1.2.1/debian/8/x86_64
./build.sh -a bigtop -o ubuntu-16.04 -c "hdfs, yarn, ignite"

# 버전 바꿔서 다시 해본다.
export REPO=http://repos.bigtop.apache.org/releases/3.1.1/ubuntu/16.04/x86_64
echo $REPO
http://repos.bigtop.apache.org/releases/3.1.1/ubuntu/16.04/x86_64

./build.sh -a bigtop -o ubuntu-16.04 -p 3.1.1 -c "hdfs, yarn, ignite"   <-- -p prefix추가 

# 오류
failed to solve: bigtop/puppet:3.1.1-ubuntu-16.04: failed to resolve source metadata for docker.io/bigtop/puppet:3.1.1-ubuntu-16.04: docker.io/bigtop/puppet:3.1.1-ubuntu-16.04: not found
```

## 9. 도커 허브에서 bigtop/puppet를 찾아보니.. ㅎㅎ
https://hub.docker.com/r/bigtop/puppet

3.1.1-ubuntu-18.04 이건 있고,  
3.2.1-ubuntu-22.04 이건 있는데,  
3.1.1-ubuntu-16.04 이건 없고,  
3.1.1-ubuntu-22.04 이것도 없다. ㅎㅎㅎ  
딱 필요한것만 없다능...  

3.1.1-ubuntu-20.04 이건 있네... 
docker.io/bigtop/puppet:3.1.1-ubuntu-20.04: docker.io/bigtop/puppet


## 10. 그렇다면... 이건 되나?
```
export REPO=http://repos.bigtop.apache.org/releases/3.1.1/ubuntu/20.04/x86_64
echo $REPO
./build.sh -a bigtop -o ubuntu-20.04 -p 3.1.1 -c "hdfs, yarn, spark" 

-------------------------------------------------
Image bigtop/sandbox:3.1.1-ubuntu-20.04_hdfs_yarn_spark built
-------------------------------------------------
# 성공
```

이것도 된건가?
```
export REPO=http://repos.bigtop.apache.org/releases/3.1.1/ubuntu/20.04/x86_64
echo $REPO
./build.sh -a bigtop -o ubuntu-20.04 -p 3.1.1 -c "hdfs, yarn, spark, hive, trino" 

-------------------------------------------------
Image bigtop/sandbox:3.1.1-ubuntu-20.04_hdfs_yarn_spark_hive_trino built
-------------------------------------------------
# 성공
```

## 11. 위에서 생성된 이미지명 넣어서
```
BIGTOP=$(docker run -d -p 50070:50070 -p 8088:8088 bigtop/sandbox:3.1.1-ubuntu-20.04_hdfs_yarn_spark_hive_trino)
docker exec -ti $BIGTOP hdfs   <--- 이건 컨테이너 내부에있는 명령어를 실행하는것 (결국 뭐 이것저것 해봐도 잘 안됨. 이미지가 잘 생성되지 않은듯 함.)
docker exec -ti $BIGTOP yarn
docker exec -ti $BIGTOP spark
docker exec -ti $BIGTOP hive 
docker exec -ti $BIGTOP trino

#오류남.
# docker exec -ti $BIGTOP hdfs
OCI runtime exec failed: exec failed: unable to start container process: exec: "hdfs": executable file not found in $PATH: unknown

```


## 12. 여기까지 되면, 도커 이미지를 만드는것 까지 된것임.
확인 및 실행.
```
# 로컬 도커이미지 repo에서 확인
docker images

# 실행중인 컨테이너 확인
docker ps -a

# 특정 컨테이너 확인
docker inspect --format='{{.Config.Image}}' <container_id_or_name>


# 도커 컨테이너 실행
docker-compose up -d
#또는
docker run -d --name my_sandbox_container $ACCOUNT/sandbox:$TAG

# 실행중지
docker stop <container_id_or_name>

# 모든 컨테이너 중지
docker stop $(docker ps -q)

# 컨테이너 속으로 콘솔 로그인
docker exec -it <container_id_or_name> /bin/bash
#또는
docker exec -it <container_id_or_name> /bin/sh

```

## 13. 결국
```
BIGTOP=$(docker run -d -p 50070:50070 -p 8088:8088 bigtop/sandbox:3.1.1-ubuntu-20.04_hdfs_yarn_spark_hive_trino)
docker exec -ti $BIGTOP hdfs   <--- 이건 컨테이너 내부에있는 명령어를 실행하는것 (결국 뭐 이것저것 해봐도 잘 안됨. 이미지가 잘 생성되지 않은듯 함.)
```
잘안됨.


## 이건 nginx 여담.   
nginx 도커 이미지속에 콘솔 로그인 하면.  
```
docker exec -it nginx-server /bin/bash
```
/etc/nginx/conf.d/default.conf 파일 있음.  
cat default.conf 해서 보면.  
아래 구문 있음.  
```
 location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
```	
여기서 /usr/share/nginx/html/index.htm  
파일 수정하면, 변경된 페이지로 서비스됨.  
그런데 vi,vim,nano가 없어서 편집이 쉽지 않음.  
cat, echo, printf, sed, awk 이런거 이용해야해. ㅎㅎ  
내부에서 바로 설치해도 됨.  
```
apt-get update && apt-get install vim -y
```
도커 컨테이너 재시작 안해도 바로 적용됨.   
아니, 오히려 컨테이너 재시작 하면 원복되서 안될 듯.
