# 그럴듯한 서비스 만들기

## 2단계 - 서비스 배포하기

### 요구사항
- [x] 운영 환경 구성하기
- [x] 개발 환경 구성하기

### 요구사항 설명

#### 운영 환경 구성하기
- [x] 웹 애플리케이션 앞단에 Reverse Proxy 구성하기
  - [x] 외부망에 Nginx로 Reverse Proxy를 구성
  - [x] Reverse Proxy에 TLS 설정
- [x] 운영 데이터베이스 구성하기

#### 개발 환경 구성하기
- [x] 설정 파일 나누기
  - JUnit : h2, Local : docker(mysql), Prod : 운영 DB를 사용하도록 설정
  
#### 구현
##### AWS 설정 변경
- 80, 443 포트만 오픈하고 nginx 로 reverse proxy & load balancer 적용
- 외부망 8080 포트 오픈, nginx 허용

##### 도커 설치
```shell
sudo apt-get update && \
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - && \
sudo apt-key fingerprint 0EBFCD88 && \
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" && \
sudo apt-get update && \
sudo apt-get install -y docker-ce && \
sudo usermod -aG docker ubuntu && \
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && \
sudo chmod +x /usr/local/bin/docker-compose && \
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
##### 1. Reverse Proxy
- Dockerfile
```shell
FROM nginx

COPY nginx.conf /etc/nginx/nginx.conf
```
- nginx.conf
```shell
events {}

http {
  upstream app {
    server 172.17.0.1:8080;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://app;
    }
  }
}
```
```shell
$ docker build -t give928/reverse-proxy .
$ docker run -d -p 80:80 give928/reverse-proxy
```

##### 2. TLS 설정
서버의 보안과 별개로 서버와 클라이언트간 통신상의 암호화가 필요합니다. 평문으로 통신할 경우, 패킷을 스니핑할 수 있기 때문입니다.

letsencrypt를 활용하여 무료로 TLS 인증서를 사용할 수 있어요.
```shell
$ docker run -it --rm --name certbot \
  -v '/etc/letsencrypt:/etc/letsencrypt' \
  -v '/var/lib/letsencrypt:/var/lib/letsencrypt' \
  certbot/certbot certonly -d 'give928.kro.kr' --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory
```

인증서 생성 후 유효한 URL인지 확인을 위해 DNS TXT 레코드로 추가합니다.
```shell
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name:

_acme-challenge.give928.kro.kr.

with the following value:

AncwM_QdzHIbbpBTXOtIGCj0VubqY6ezMsFwCURlraY

Before continuing, verify the TXT record has been deployed. Depending on the DNS
provider, this may take some time, from a few seconds to multiple minutes. You can
check if it has finished deploying with aid of online tools, such as the Google
Admin Toolbox: https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.give928.kro.kr.
Look for one or more bolded line(s) below the line ';ANSWER'. It should show the
value(s) you've just added.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/give928.kro.kr/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/give928.kro.kr/privkey.pem
This certificate expires on 2022-08-29.
These files will be updated when the certificate renews.

NEXT STEPS:
- This certificate will not be renewed automatically. Autorenewal of --manual certificates requires the use of an authentication hook script (--manual-auth-hook) but one was not provided. To renew this certificate, repeat this same certbot command before the certificate's expiry date.
```

DNS를 설정하는 사이트에서 DNS TXT 레코드를 추가한 후, 제대로 반영되었는지 dig 명령어로 확인한 후에 인증서 설정 진행을 계속합니다.
```shell
$ dig -t txt _acme-challenge.give928.kro.kr +short
```

생성한 인증서를 활용하여 Reverse Proxy에 TLS 설정을 해봅시다. 우선 인증서를 현재 경로로 옮깁니다.
```shell
$ cp /etc/letsencrypt/live/give928.kro.kr/fullchain.pem ./
$ cp /etc/letsencrypt/live/give928.kro.kr/privkey.pem ./
```

Dockerfile 을 아래와 같이 수정합니다.
```shell
FROM nginx

COPY nginx.conf /etc/nginx/nginx.conf 
COPY fullchain.pem /etc/letsencrypt/live/give928.kro.kr/fullchain.pem
COPY privkey.pem /etc/letsencrypt/live/give928.kro.kr/privkey.pem
```

nginx.conf 파일을 아래와 같이 수정합니다.
```shell
events {}

http {       
  upstream app {
    server 172.17.0.1:8080;
  }
  
  # Redirect all traffic to HTTPS
  server {
    listen 80;
    return 301 https://$host$request_uri;
  }

  server {
    listen 443 ssl;  
    ssl_certificate /etc/letsencrypt/live/give928.kro.kr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/give928.kro.kr/privkey.pem;

    # Disable SSL
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

    # 통신과정에서 사용할 암호화 알고리즘
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;

    # Enable HSTS
    # client의 browser에게 http로 어떠한 것도 load 하지 말라고 규제합니다.
    # 이를 통해 http에서 https로 redirect 되는 request를 minimize 할 수 있습니다.
    add_header Strict-Transport-Security "max-age=31536000" always;

    # SSL sessions
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;      

    location / {
      proxy_pass http://app;    
    }
  }
}
```

방금전에 띄웠던 도커 컨테이너를 중지 & 삭제하고 새로운 설정을 반영하여 다시 띄워봅시다.
```shell
$ docker stop 2a414cb0ca96 && docker rm 2a414cb0ca96
$ docker build -t give928/reverse-proxy:0.0.2 .
$ docker run -d -p 80:80 -p 443:443 --name proxy give928/reverse-proxy:0.0.2
```

##### 3. 컨테이너로 운영 DB 사용하기
일반적으로, 실제 운영환경에서 컨테이너로 데이터베이스의 영속성 데이터를 다루지 않습니다. 컨테이너의 철학과 데이터베이스의 영속성은 다소 배치되는 부분이 있다고 생각합니다. 여기서는 원활한 실습을 위해 제가 미리 push해둔 컨테이너를 활용합니다.
```shell
$ docker run -d -p 3306:3306 brainbackdoor/data-subway:0.0.1
```

##### 4. 설정 파일 나누기
```shell
$ java -jar -Dspring.profiles.active=prod [jar파일명] 
```

##### 데이터베이스 테이블 스키마 버전관리
운영중인 서비스의 경우 JPA 등 ORM을 사용하여 기존의 테이블을 변경하는 것은 데이터 유실 우려, 참조 무결성 제약 등으로 인해 어려움이 있습니다. 그리고 데이터베이스 테이블 스키마도 버전관리를 할 필요가 있습니다. 그럴 때 로컬에서 개발 중일 때는 h2 등 in-memory 형태의 데이터베이스를 사용하여 빠르게 개발하고, 운영 DB는 점진적으로 migration 해가는 전략이 유용합니다.

- docker/db/mysql/init에 dump 파일을 넣은 상태로 실행하면 자동으로 초기 데이터를 INSERT할 수 있어요.
- flyway는 V__[변경이력].sql의 형태로 resources/db/migration/ 경로에서 관리합니다. 그리고 flyway_schema_history 테이블에 버전별로 checksum 값을 관리하므로 기존 sql 문을 수정해서는 안됩니다.

기존 Dtabase 존재시 flyway 적용 방법
```properties
# application.properties
spring.flyway.baseline-on-migrate=true
spring.flyway.baseline-version=2
```
이전에 database가 존재할 경우 baseline 옵션을 활용하면 특정 버전(V2__xx.sql 파일) 내용부터 적용이 가능해요.

##### 설정 별도로 관리하기
키, 계정 정보, 접속 URL 등의 설정 정보를 소스코드와 함께 형상관리할 경우 보안 이슈가 발생할 수 있어 따로 관리할 것이 권장됩니다. 보통 Jenkins / Travis CI 등의 배포 서버에 파라미터를 지정하거나, Spring Cloud Config / AWS Service Manager 등의 외부 서비스를 활용하는 방안 등이 활용됩니다. 여기서는 저장소를 분리하여 private repository에서 설정을 관리하도록 합니다.

a. 우선, github private 저장소를 생성한 후 application.properties 등의 설정 파일을 올립니다.

b. git의 서브모듈 기능을 활용하여 특정 경로에 private repository를 참조하도록 설정합니다
```shell
$ git submodule add [자신의 private 저장소] ./src/main/resources/config
```

이후에 소스코드를 받을 떄는 서브모듈까지 clone해야 합니다.
```shell
$ git clone --recurse-submodules [자신의 프로젝트 저장소]
```

c. 설정 파일의 내용이 변경된 경우
```shell
git submodule foreach git pull origin main

git submodule foreach git add .

git submodule foreach git commit -m "commit message"

git submodule foreach git push origin main
```
