---
layout: post
title:  "[AWS] EC2 메모리 이슈"
date:   2022-04-26 18:25:01 +0900
categories: Devlife
---

# [AWS] EC2 프리티어 메모리 이슈

## 이슈의 배경
현재 AWS EC2를 사용하여 spring boot 공부를 하고 있다.  
git으로 이미 설정된 spring boot파일을 루트 디렉토리의 /var/www에 clone해서 EC2 서버에 배포하는 방식으로 진행중이다.  
따라서 java 11, git을 서버에 설치해준 뒤 /var/www에 git clone 해주었다.  
내 경우에는 해당 프로젝트 디렉토리가 udemy_spring_study_project여서 이 곳으로 cd한 뒤, ./gradlew build를 통해 빌드를 하면 되었는데, 이 과정에서 EC2의 메모리 이슈가 발생했다.  

따로 에러 메시지가 뜨지 않고, building의 진행사항 중 7%가 되면 compileJava라는 문구만 출력되어있는 상태로 먹통이 되었다.  
처음에는 연결되어있는 네트워크의 문제인가 싶어 장소를 옮겨보기도, 핸드폰의 핫스팟을 이용해보기도 했지만 변함이 없었다. windows환경에서 진행중이기에 winSCP를 사용중인데 마냥 기다리다가 소프트웨어적인 연결 중단이라며 puTTY와 winSCP의 접속이 끊기기도 했다.  
(이전에 APM을 수동설치했을 때도 한참 걸린 경험이 있어 정말 오래 걸리는 과정인가 싶어 45분 넘게 켜두었다가 노트북이 종료되기도 했었다.)  

sudo의 문제인가 싶어 sudo도 붙여보고, apt-get update, apt-get upgrade도 모두 시도해보았다.  
그런데 두번째로 위의 update도 정상작동을 시작하다가 Hit11쯤 부터 먹통이 되어버리더라.  
이 때부터 단순 build의 문제가 아니라고 느껴져 AWS의 내 인스턴스의 모니터링 탭을 통해 cpu점유율 등을 확인했다.
<img src='/assets/img/docs/memory_build_issue3.png'>
먹통이 될 때의 시간을 확인하면 대체로 100%에 가까운 점유율을 보이더라. (반응이 느린건지 뭔지 모르겠지만 사용률이 높지 않은 경우도 가끔 보였다.)  
관련된 사항을 구글링해보고 질문도 해 본 결과 EC2 인스턴스-t2.micro-의 메모리가 정말 micro해서 발생하는 문제라고 한다. (1GiB)  


## 해결 방안: swap영역을 이용한 메모리 확장
puTTY로 EC2 해당 인스턴스에 접속한 뒤 다음 코드들을 입력해준다.  
```
sudo dd if=/dev/zero of=/swapfile bs=128M count=16   
                            #dd 명령어를 통해 루트에 swapfile 생성(bs: block size, count: block의 수)  
                            #128MB x 15 = 2GB 크기의 파일을 생성함
                            #해당 크기는 AWS EC2 프리티어 권장 swap 공간

sudo chmod 600 /swapfile    #해당 swapfile에 읽기, 쓰기 권한 생성

sudo mkswap /swapfile       #리눅스 swap영역 설정

sudo swapon /swapfile       #swap 메모리 활성화

sudo vim /etc/fstab 
                    #파일이 vim에디터로 열리면 /swapfile swap swap defaults 0 0 을 추가하고 :wq로 나오기.
                    #부팅할 때마다 자동으로 swapfile을 활성화 하도록 해당 코드를 추가한 것

free                        #swap 메모리 확인하기
```


<img src='/assets/img/docs/memory_build_issue.png'>
<img src='/assets/img/docs/memory_build_issue1.png'>
swap 메모리 설정 뒤, update upgrade가 잘 작동하고,
<img src='/assets/img/docs/memory_build_issue2.png'>
기존 목적이었던 build도 성공적으로 빠르게 완료되었다.

## Swap이란?
- RAM 용량이 가득 차 데이터가 손실되거나 에러가 발생하는 것을 방지하기 위한 메모리 영역
- 하드 디스크의 메모리를 RAM처럼 교환하여(swap) 사용하도록 해 시스템 메모리 공간 부족을 방지함

[출처][(https://it-serial.tistory.com/entry/Linux-Swap-%ED%8C%8C%ED%8B%B0%EC%85%98%EC%9D%B4%EB%9E%80-CPU-RAM-%ED%95%98%EB%93%9C-%EB%94%94%EC%8A%A4%ED%81%AC-%E2%91%A0)](https://it-serial.tistory.com/entry/Linux-Swap-%ED%8C%8C%ED%8B%B0%EC%85%98%EC%9D%B4%EB%9E%80-CPU-RAM-%ED%95%98%EB%93%9C-%EB%94%94%EC%8A%A4%ED%81%AC-%E2%91%A0)


