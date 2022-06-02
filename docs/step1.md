# 그럴듯한 서비스 만들기

## 1단계 - 서비스 구성하기

### 요구사항
- [x] 웹 서비스를 운영할 네트워크 망 구성하기
- [x] 웹 애플리케이션 배포하기

#### 망 구성
- [x] VPC 생성
  - CIDR은 C class(x.x.x.x/24)로 생성.
- [x] Subnet 생성
  - [x] 외부망으로 사용할 Subnet: 64개씩 2개(AZ를 다르게 구성)
  - [x] 내부망으로 사용할 Subnet: 32개씩 1개
  - [x] 관리용으로 사용할 Subnet : 32개씩 1개
- [x] Internet Gateway 연결 
- [x] Route Table 생성
- [x] Security Group 설정
  - [x] 외부망
    - 전체 대역 : 8080 포트 오픈 
    - 관리망 : 22번 포트 오픈 
  - [x] 내부망 
    - 외부망 : 3306 포트 오픈 
    - 관리망 : 22번 포트 오픈
  - [x] 관리망
    - 자신의 공인 IP : 22번 포트 오픈
- [x] 서버 생성
  - [x] 외부망에 웹 서비스용도의 EC2 생성
  - [x] 내부망에 데이터베이스용도의 EC2 생성
  - [x] 관리망에 베스쳔 서버용도의 EC2 생성
  - [x] 베스쳔 서버에 Session Timeout 600s 설정
  - [x] 베스쳔 서버에 Command 감사로그 설정
#### 웹 애플리케이션 배포
- [x] 외부망에 웹 애플리케이션을 배포
- [x] DNS 설정

#### 망 구성
- VPC
  - 생성할 리소스: VPC 만 생성
  - 이름 태그: give928-vpc
  - IPv3 CIDR 블록: 192.168.1.0/24
- Subnet
  - 외부망1
    - 이름: give928-subnet-public01
    - 가용 영역: 아시아 태평양(서울) / ap-northeast-2a
    - IPv4 CIDR 블록: 192.168.1.0/26
  - 외부망2
    - 이름: give928-subnet-public02
    - 가용 영역: 아시아 태평양(서울) / ap-northeast-2c
    - IPv4 CIDR 블록: 192.168.1.64/26
  - 내부망
    - 이름: give928-subnet-private
    - 가용 영역: 아시아 태평양(서울) / ap-northeast-2c
    - IPv4 CIDR 블록: 192.168.1.128/27
  - 관리망
    - 이름: give928-subnet-manage
    - 가용 영역: 아시아 태평양(서울) / ap-northeast-2a
    - IPv4 CIDR 블록: 192.168.1.160/27
- Internet Gateway
  - 이름: give928-igw
- Route Table
  - 외부망
    - 이름: give928-rt-public
    - 라우팅
      - 192.168.1.0/24, local
      - 0.0.0.0/0, give928-igw
    - 서브넷 연결
      - give928-subnet-public01
      - give928-subnet-public02
      - give928-subnet-manage
  - 내부망
    - 이름: give928-rt-private
    - 라우팅
      - 192.168.1.0/24, local
    - 서브넷 연결
      - give928-subnet-private
- Security Group
  - 관리망
    - 이름: give928-sg-manage
    - 인바운드 규칙
      - 유형: SSH
      - 소스: 내 IP
    - 인바운드 규칙
      - 유형: 모든 ICMP - IPv4
      - 소스: 내 IP
  - 외부망
    - 이름: give928-sg-public
    - 인바운드 규칙
      - 유형: SSH
      - 소스: 관리망 보안 그룹
    - 인바운드 규칙
      - 유형: 사용자 지정 TCP, 8080
      - 소스: 0.0.0.0/0
  - 내부망
    - 이름: give928-sg-private
    - 인바운드 규칙
      - 유형: SSH
      - 소스: 관리망 보안 그룹
    - 인바운드 규칙
      - 유형: MYSQL
      - 소스: 외부망 보안 그룹
- 키 페어
  - 이름: give928-key
  - 유형: RSA
  - 프라이빗 키 파일 형식: .pem
- EC2
  - bastion server
    - 이름: give928-EC2-bastion
    - OS: Ubuntu Server 18.04 LTS (HVM), SSD Volume Type 64bit(x86)
    - 인스턴스 유형: t2.micro
    - 키 페어: give928-key
    - 네트워크 설정
      - VPC: give928-vpc
      - Subnet: give928-subnet-manage
      - 퍼블릭 IP 자동 할당: 활성화
      - 방화벽(보안 그룹): 기존 보안 그룹 선택, give928-sg-manage
    - 스토리지 구성: 1x8GiB gp3 루트 볼륨
  - web service1
    - 이름: give928-EC2-web01
    - OS: Ubuntu Server 18.04 LTS (HVM), SSD Volume Type 64bit(x86)
    - 인스턴스 유형: t2.micro
    - 키 페어: give928-key
    - 네트워크 설정
      - VPC: give928-vpc
      - Subnet: give928-subnet-public
      - 퍼블릭 IP 자동 할당: 활성화
      - 방화벽(보안 그룹): 기존 보안 그룹 선택, give928-sg-public
    - 스토리지 구성: 1x8GiB gp3 루트 볼륨
  - web service2
    - 이름: give928-EC2-web02
    - OS: Ubuntu Server 18.04 LTS (HVM), SSD Volume Type 64bit(x86)
    - 인스턴스 유형: t2.micro
    - 키 페어: give928-key
    - 네트워크 설정
      - VPC: give928-vpc
      - Subnet: give928-subnet-public
      - 퍼블릭 IP 자동 할당: 활성화
      - 방화벽(보안 그룹): 기존 보안 그룹 선택, give928-sg-public
    - 스토리지 구성: 1x8GiB gp3 루트 볼륨
  - database
    - 이름: give928-EC2-mysql
    - OS: Ubuntu Server 18.04 LTS (HVM), SSD Volume Type 64bit(x86)
    - 인스턴스 유형: t2.micro
    - 키 페어: give928-key
    - 네트워크 설정
      - VPC: give928-vpc
      - Subnet: give928-subnet-private
      - 퍼블릭 IP 자동 할당: 비활성화 -> 활성화(docker mysql pull)
      - 방화벽(보안 그룹): 기존 보안 그룹 선택, give928-sg-private
    - 스토리지 구성: 1x8GiB gp3 루트 볼륨
- DNS 설정
  - IP연결(A)

#### 서버 접속
```shell
# 터미널 접속한 후 앞 단계에서 생성한 key가 위치한 곳으로 이동한다.
$ chmod 400 [pem파일명]
$ ssh -i [pem파일명] ubuntu@[SERVER_IP]
```

#### 접근 통제
```shell
## Bastion Server에서 공개키를 생성합니다.
bastion $ ssh-keygen -t rsa
bastion $ cat ~/.ssh/id_rsa.pub

## 접속하려는 서비스용 서버에 키를 추가합니다.
$ vi ~/.ssh/authorized_keys

## Bastion Server에서 접속을 해봅니다.
bastion $ ssh ubuntu@[서비스용 서버 IP]

bastion $ vi /etc/hosts
[서비스용IP]    [별칭]

bastion $ ssh [별칭]
```

#### Session Timeout 설정
```shell
bastion $ sudo vi ~/.profile
HISTTIMEFORMAT="%F %T -- "    ## history 명령 결과에 시간값 추가
export HISTTIMEFORMAT
export TMOUT=600              ## 세션 타임아웃 설정 
    
bastion $ source ~/.profile
bastion $ env
```

#### shell prompt 변경
```shell
bastion $ sudo vi ~/.bashrc
USERNAME=BASTION
PS1='[\e[1;31m$USERNAME\e[0m][\e[1;32m\t\e[0m][\e[1;33m\u\e[0m@\e[1;36m\h\e[0m \w] \n\$ \[\033[00m\]'

bastion $ source ~/.bashrc
```

#### Command 감사로그 설정
```shell
$ sudo vi ~/.bashrc
tty=`tty | awk -F"/dev/" '{print $2}'`
IP=`w | grep "$tty" | awk '{print $3}'`
export PROMPT_COMMAND='logger -p local0.debug "[USER]$(whoami) [IP]$IP [PID]$$ [PWD]`pwd` [COMMAND] $(history 1 | sed "s/^[ ]*[0-9]\+[ ]*//" )"'

$ source  ~/.bashrc
```
```shell
$ sudo vi /etc/rsyslog.d/50-default.conf
local0.*                        /var/log/command.log
# 원격지에 로그를 남길 경우 
local0.*                        @원격지서버IP
    
$ sudo service rsyslog restart
$ tail -f /var/log/command.log
```

### 소스코드 배포, 빌드 및 실행
- [서버를 시작 시간이 너무 오래 걸리는 경우 -Djava.security.egd 옵션을 적용](https://lng1982.tistory.com/261)
```shell
$ java -Djava.security.egd=file:/dev/./urandom -jar [jar파일명] &
```
- 터미널 세션이 끊어질 경우, background로 돌던 프로세스에 hang-up signal이 발생해 죽는 경우 nohup명령어를 활용
```shell
$ nohup java -jar [jar파일명] 1> [로그파일명] 2>&1  &
```
