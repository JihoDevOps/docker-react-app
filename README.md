#   간단한 어플을 실제로 배포해보기

## Index

A. [개발 환경 부분](#a-개발-환경-부분)

1. [섹션 설명](#1-섹션-설명)
2. [리액트 앱 설치하기](#2-리액트-앱-설치하기)
3. [도커를 이용하여 리액트 앱 실행하기](#3-도커를-이용하여-리액트-앱-실행하기)
4. [생성된 도커 이미지로 리액트 앱 실행하기](#4-생성된-도커-이미지로-리액트-앱-실행하기)
5. [도커 볼륨을 이용한 소스 코드 변경](#5-도커-볼륨을-이용한-소스-코드-변경)
6. [도커 컴포즈로 좀 더 간단하게 앱 실행하기](#6-도커-컴포즈로-좀-더-간단하게-앱-실행하기)
7. [리액트 앱 테스트하기](#7-리액트-앱-테스트하기)
8. [운영환경을 위한 Nginx](#8-운영환경을-위한-nginx)
9. [운영환경 도커 이미지를 위한 Dockerfile 작성하기](#9-운영환경-도커-이미지를-위한-Dockerfile-작성하기)

B. [테스트 배포 부분](#b-테스트-배포-부분)

1. [섹션 설명 및 Github에 소스 코드 올리기](#1-섹션-설명-및-github에-소스-코드-올리기)
2. [Travis CI 설명](#2-travis-ci-설명)
3. [Travis CI 이용 순서](#3-travis-ci-이용-순서)
4. [travis yml 파일 작성하기 (테스트)](#4-travis-yml-파일-작성하기-테스트)
5. [AWS 알아보기](#5-aws-알아보기)
6. [Elastic Beanstalk 환경 구성하기](#6-elastic-beanstalk-환경-구성하기)
7. [travis yml 파일 작성하기 (배포)](#7-travis-yml-파일-작성하기-배포)
8. [Travis CI의 AWS 접근을 위한 API 생성](#8-travis-ci의-aws-접근을-위한-api-생성)

---

## A. 개발 환경 부분

### 1. 섹션 설명

#### Flow

1.  Develop
2.  Test
3.  Production

---

#### 1. Develop

##### A. React App Develop

1.  React.js 앱 개발
2.  Dockerfile 작성
3.  Docker Compose 파일 작성

##### B. Github Push

1.  Github에 소스를 Push
2.  Feature Branch
3.  Merge Request → Master Branch

---

#### 2. Test

1.  Travis CI에서 Master Branch에 Push된 코드를 가져감
2.  개발된 소스가 잘 작동하는지 Test

---

#### 3. Production

1.  Hosting site(AWS, Azure, Google, ...)

---

### 2. 리액트 앱 설치하기

#### React 설치

```bash
$ npx create-react-app ./
```

#### React 실행

```bash
$ npm run start
```

#### React 테스트

```bash
$ npm run test
```

#### React 빌드

```bash
$ npm run build
```

>   큰 어려움은 없는 부분이다.
>   리액트를 사용하기 위해서는 NodeJS가 설치되어 있어야 한다.
>   `npx create-react-app` 명령어 때문이라고 생각된다.
>   하나씩 입력해보면서 진행하면 되고, 리액트 설치 오래걸린다.

---

### 3. 도커를 이용하여 리액트 앱 실행하기

#### Dockerfile의 종류

이전까지 dockerfile 하나로 진행했다.
하지만 실제 개발 환경에서는 `개발 환경`과 `배포 환경`을 구분한다.
따라서 개발 단계에서는 `dockerfile.dev`를 작성한다.
내용은 기존과 동일하게 작성하면 된다.

```dockerfile
# dockerfile.dev
FROM node:alpine
WORKDIR /usr/src/app
COPY package.json ./
RUN npm install
COPY ./ ./
CMD ["npm", "run", "start"]
```

위처럼 작성하고 빌드하면 다음과 같은 메시지가 출력된다.

```bash
$ docker build ./
unable to prepare context: unable to evaluate symlinks in Dockerfile path: CreateFile C:\Dev\Workspace\Inflearn\Docker-Inflearn\single-container-app\Dockerfile: The 
system cannot find the file specified.
```

#### unable to evaluate symlink ...

해당 에러가 발생하는 이유는
이미지를 빌드할 때 해당 디렉터리만 지정하면
dockerfile을 자동으로 찾아서 빌드한다.
하지만 현재 dockerfile은 없고, dockerfile.dev만 존재한다.
즉, 자동으로 dockerfile을 찾지 못해 발생하는 에러다.

해결책은 임의로 빌드 시 어떤 파일을 참조하는지 지정하면 된다.
아래처럼 입력하면 된다.

```bash
$ docker build -f dockerfile.dev .
# 웬만하면 이름을 부여하고 빌드하자...
$ docker build -f dockerfile.dev -t jiho/react-app .
```

#### tip

Local에는 node_modules 디렉토리가 필요 없다.
그냥 과감히 node_modules 폴더를 지워도 도커에서는 잘 돌아간다.

---

### 4. 생성된 도커 이미지로 리액트 앱 실행하기

docker의 port와 local의 port를 매핑하여 실행한다.

```bash
$ docker run -p 3000:3000 [image name]
```

> 또 안 된다고 하는데 나는 지금 잘 된다.
> 일단은 `-it` 옵션을 붙여줘야 한다고 한다.
> docker 또는 react 버전에 따라 다르다.

```bash
# -it 옵션은 환경에 따라 필수는 아니다.
$ docker run -it -p 3000:3000 [image name]
# -d 옵션은 백그라운드에서 실행해준다.
$ docker run -it -d -p 3000:3000 [image name]
```

---

### 5. 도커 볼륨을 이용한 소스 코드 변경

#### Volume 복습

>   Local에서 코드를 수정하면
>   다시 이미지를 빌드하지 않아도 컨테이너에 적용되는 방법이다.

예전 강의에서 이미 `COPY` 대신 `VOLUME`을 이용해서
소스를 변경했을 때 다시 이미지를 빌드하지 않아도
변경한 소스 부분이 어플리케이션에 반영되는 부분을 해봤다.

**Volume**은 Local에 있는 파일을 Conatiner가 참조하도록 한다.

```bash
# MacOS
$ docker run -p 3000:3000 -v [workdir]/node_modules -v $(pwd):[workdir] [image]
# Windows
$ docker run -p 3000:3000 -v [workdir]/node_modules -v %cd%:[workdir] [image]
```

> 현재 제대로 동작하지 않고 있다.
> 실시간으로 변경되는 것이 확인되지 않는다.
> dockerfile.dev에서 `npm run start` 대신 `npm run serve`로 바꿨다.
> 그래도 실시간으로 소스 수정이 적용되지 않았다.

> 다시 시도하고 있는데, `npm run serve`에서 오류가 발생한다.
> 다시 되돌렸고, build 폴더를 지웠다. 그래도 해결되지 않았다.
> 어차피 compose 쓸 거니까 일단 넘어간다.

---

### 6. 도커 컴포즈로 좀 더 간단하게 앱 실행하기

volume의 장점은 알겠으나 command 명령어가 너무 길어진다.
이러한 단점을 해결하기 위해 compose를 사용한다.

```yml
# docker-compose.yml

version: "3"                        # docker compose version
services:                           # 실행하려는 containers 정의
  react:                            # container name
    build:                          # 현재 dir에 있는 dockerfile 사용
      context: .                    # 도커 이미지 구성 파일 경로
      dockerfile: dockerfile.dev    # dockerfile 설정
    ports:                          # 포트 매핑
      - "3000:3000"
    volumes:                        # local 머신에 있는 파일들 매핑
      - /usr/src/app/node_modules
      - ./:/usr/src/app
    stdin_open: true                # 리액트 앱 종료 시 필요(버그 수정)
```

위 파일을 생성 후 다음과 같이 간단하게 실행할 수 있다.

```bash
$ docker-compose up
$ docker-compose up --build # 다시 빌드하여 실행
```

> 아직도 실시간으로 수정이 되지 않는다.
> 심지어 이전에 진행했던 프로젝트도 동일하다.
> 환경설정의 문제라는 가능성이 커졌다.

> 아무리 해도 고쳐지지 않는다...
> 아이패드 배터리도 떨어졌다... 집에 가자...

#### ❗❓ react는 추가 설정이 필요하다...

그렇게 구글링해도 안 나오던 게
[인프런](https://www.inflearn.com/questions/84167)에 있었다.

> 등잔 밑이 어둡다.

Windows OS에 한정되는 현상으로 보인다.
진짜 Mac을 사는 게 맞는 걸까?
별짓 다 했는데, 아래 코드 두 줄만 추가하면 된다.

```dockerfile
  react:
    # ...
    environment:
      - CHOKIDAR_USEPOLLING=true
    # ...
```

---

### 7. 리액트 앱 테스트하기

Image가 build 된 상황에서 다음과 같이 명령어를 입력하면
docker 환경에서 테스트를 진행할 수 있다.

```bash
$ docker run -it [image] npm run test
# -it : 테스트 결과를 보기 편하게 하는 설정
```

테스트 코드 또한 compose되어 Local에서 수정하여
실시간으로 반영되게 설정하는 방법이 있다.

기존 방법과 유사하게 volume을 이용하고,
`docker-compose.yml`에 설정을 추가한다.

```yml
tests:
  build:
    context: .
    dockerfile: dockerfile.dev
  volumes:                   # 디렉터리 맵핑
    - [workdir]/node_modules # Local과 연결하지 않는다.
    - [target-dir]:[workdir] # 개발 dir과 docker dir을 연결한다.
  command: ["npm", "run", "test"]
```

> 이 친구도 실시간 반영이 되지 않는다.
> react app 처럼 chokidar 설정을 했지만 소용없었다.
> 현재는 `jset`라는 종속성을 추가하여 빌드하고 있다.

---

### 8. 운영환경을 위한 Nginx

운영 환경(배포 후)을 다룬다.

#### Nginx가 필요한 이유

##### 개발 환경에서 실행

```text
< Browser >--------+         < React Container>------------------+
| http://localhost | ------- | +------------+   index.html       |
|                  |         | | Dev Server |                    |
+------------------+         | +------------+   *.js, *.css, ... |
                             +-----------------------------------+
```

개발 환경에서는 개발 서버로 앱이 실행된다.
하지만, 실제 운영 중일 경우 개발 서버는 없다.

##### 운영 환경에서 실행

```text
< Browser >--------+         < React Container>-------------+
| http://localhost | ------- | +-------+   index.html       |
|                  |         | | Nginx |                    |
+------------------+         | +-------+   *.js, *.css, ... |
                             +------------------------------+
                                            빌드된 파일들
```

개발 서버 대신 Nginx가 대체하여 실행한다.

###### 왜 개발 환경 서버와 운영 환경 서버를 다르게 하나?

개발에서 사용하는 서버는 소스를 변경하면
자동으로 전체 앱을 다시 빌드해서 변경 소스를 반영한다.
이처럼 개발 환경에 특화된 기능들이 있기 때문에
그런 기능이 없는 Nginx 서버보다 적합하다.

운영환경에서 소스 변경 시 다시 반영할 필요 없고,
개발에 필요한 기능들이 필요하지 않으므로
더 깔끔하고 빠른 Nginx를 웹 서버로 사용한다.

> 가볍고 빠르게 운영 환경에서 사용하기 위함

---

### 9. 운영환경 도커 이미지를 위한 Dockerfile 작성하기

Nginx를 포함하는 React 운영 환경 이미지 생성을 위해
`dockerfile`을 작성한다.

`dockerfile.dev`는 개발 환경을 위한 설정이고,
`dockerfile`은 운영 환경을 위한 설정이다.

둘의 설정 차이는 크게 `CMD` 부분이다.
`dockerfile.dev`는 `npm run start`이지만,
`dockerfile`은 `npm run build`로 설정한다.
빌드된 파일을 Nginx 서버가 읽고 브라우저에 보여준다.

운영 환경을 위한 Dockerfile을 요약하자면 2 단계로 이루어진다.

1.  (Builder Stage) 빌드 파일들을 생성한다.
2.  (Run Stage) Nginx를 가동하고 빌드 파일들을 브라우저로 제공한다.

#### Builder Stage

```dockerfile
# 이 FROM부터 다음 FROM까지는 Builder Stage
FROM node:alpine as builder
WORKDIR /usr/src/app
COPY package.json .
RUN npm install
COPY . .
RUN ["npm", "run", "build"]
# 빌드 결과물은 WORKDIR/build에 생성된다.
```

#### Run Stage

```dockerfile
FROM nginx
COPY --from=builder /app/build /usr/share/nginx/html
# --from=builder: Stage 명시
```

builder stage에서 생성된 파일들은 `/usr/src/app/build`에 들어가게 되며
그곳에 저장된 파일들을 `/usr/share/nginx/html`로 복사한다.
nginx가 웹 브라우저의 http 요청이 올 때마다 알맞은 파일을 전달한다.

`/usr/share/nginx/html`로 파일들을 복사하는 이유는
이 장소로 파일을 위치하면 Nginx가 알아서 Client에서 요청에 대응한다.
이 장소는 설정을 통해 변경 가능하다.

>   애초에 npm build를 저기에 해도 되잖아?

---

## B. 테스트 배포 부분

### 1. 섹션 설명 및 Github에 소스 코드 올리기

1.  GitHub 새로운 Repo에 react-app push
2.  Travis CI 연결

### 2. Travis CI 설명

Travis CI는 GitHub에서 진행하는 오픈소스 프로젝트를 위한
지속적인 통합(Continuous Integration) 서비스이다.
2011년 설립되어 2012년 급성장했다.
기존에는 Ruby 언어만 지원했으나 현재 대부분 개발 언어를 지원한다.
Travis CI를 이용하면 GitHub repository에 있는 프로젝트를
특정 이벤트에 따라 자동으로 테스트, 빌드 및 배포할 수 있다.
Private repository는 유료로 일정 금액을 지불하여 사용한다.

#### Travis CI Flow

>   Local Git → GitHub → Travis CI → AWS

1.  Local Git에 있는 소스를 GitHub 저장소에 Push
2.  GitHub master branch에 push 이벤트 발생 시
    Travis CI에게 소스가 Push 되었다고 알림
3.  Travis CI는 업데이트 된 소스를 GitHub에서 가져옴
4.  GitHub에서 가져온 소스의 테스트 코드 실행
5.  테스트 실행 후 성공 시 AWS와 같은 호스팅 사이트로 보내 배포

### 3. Travis CI 이용 순서

#### Travis CI 이용 순서

GitHub에 Push 시 Travis CI에서 그 소스를 가져가야 한다.
GitHub과 Travis CI 연결이 필요하다.

1.  Travis CI 사이트 방문 및 가입
2.  `.travis.yml` 파일 작성하여 설정 완료

GitHub에서 Travis CI로 소스를 어떻게 전달할 것인지,
전달 받은 소스를 어떻게 테스트 할 것인지,
테스트가 성공하고 어떻게 AWS에 배포할 것인지를 설정해야 한다.

`docker-compose.yml`과 비슷하게 `.travis.yml`로 설정한다.


### 4. travis yml 파일 작성하기 (테스트)

1.  Test를 수행하기 위한 준비
    -   도커 환경에서 리액트 앱을 실행하고 있으니
        Travis CI에서도 도커 환경 구성
    -   구성된 도커 환경에서 Dockerfile.dev로 도커 이미지 생성
2.  Test 수행
    -   어떻게 Test를 수행할 것인지 설정
3.  AWS로 배포하기
    -   어떻게 AWS에 소스코드를 배포할 것인지 설정하기

```yml
# 관리자 권한 설정
sudo: required

# 언어 플랫폼 선택
language: generic

# Docker 환경 구성
services:
  - docker

# Scipt 실행 전 필요한 수행 - 이미지 빌드
before_install:
  - echo "Start creating on image with dockerfile"
  - docker build -t jiho/docker-react-app -f dockerfile.dev .

# Script로 테스트 실행
script:
  - docker run -e CI=true jiho/docker-react-app npm run test -- --coverage

# 테스트 성공 후 필요한 수행 - 로그 찍기
after_success:
  - echo "Test Success"
```

#### Repository가 Organization에 있을 경우

Organization을 등록하고, Free Plan을 등록해야만 알아먹는다.
아직 Travis Documents에 토글이 있다고 하는데 지금은 없다.
그래서 혼동이 왔으나 역시나 스택 오버플로우가 나를 도왔다.

### 5. AWS 알아보기

1.  [AWS 사이트 방문](https://aws.amazon.com)
2.  회원가입 및 신용카드 정보 등록
3.  AWS Dashboard
4.  Elastic BeanStalk 검색

#### What is EC2?

>   Elastic Compute Cloud

Amazon Elastic Computed Cloud는 Amazon Web Services(AWS) 클라우드에서
확장식 컴퓨팅을 제공합니다.
Amazon EC2를 사용하면 하드웨어에 선투자할 필요가 없어
더 빠르게 애플리케이션을 개발하고 배포할 수 있습니다.
Amazon EC2는 요구 사항이나 갑작스러운 인기 증대 등
변동 사항에 따라 신속하게 규모를 확장하거나 축소할 수 있어
서버 트래픽 예측 필요성이 줄어듭니다.

#### What is EB?

>   Elastic BeanStalk

AWS Elasic BeanStalk는 Apache, Nginx같은 친숙한 서버에서
Java, NET, PHP, Node.js, Python, Ruby, Go 및 Docker와 함꼐
개발된 웹 응용 프로그램 및 서비스를 배포하고 확장하기 쉬운 서비입니다.

EB는 EC2 인스턴스나 데이터베이스 같이 많은 것들을 포함한 "환경"을
구성하며 만들고 있는 소프트웨어를 업데이트를 할 때마다 자동으로 관리한다.

### 6. Elastic Beanstalk 환경 구성하기

EB에는 로드밸런서가 있기 때문에 트래픽 양에 따라
자동적으로 EC2 유닛을 관리해준다.

-   실행 버전: Sample Application
-   플랫폼
    -   Docker
    -   64bit Amazon Linux/2.16.11

### 7. travis yml 파일 작성하기 (배포)

#### `.travis.yml` 파일 작성하기 (배포 부분)

현재는 도커 이미지를 생성 후 애플리케이션을 테스트만 진행한다.
테스트에 성공하면 AWS EB에 자동으로 배포하는 부분을 작성한다.

```yml
deploy:
  # 외부 서비스 표시: s3, elasticbeanstalk, firebase 등
  provider: elasticbeanstalk
  # 현재 사용하고 있는 AWS의 서비스가 위치하고 있는 물리적 장소
  region: "ap-northeast-2"
  # 생성된 애플리케이션의 이름
  app: "docker-react-app"
  # Elastic BeanStalk Environment 이름
  env: "Dockerreactapp-env"
  # 해당 elasticbeanstalk을 위한 s3 버켓 이름
  # s3: 파일들을 저장하는 곳
  # Travis가 build 파일을 압축하여 s3로 전송하기 때문에 필요하다.
  # s3는 EB 생성 시 알아서 생성되어 있다.
  # aws console에서 s3 검색하면 이미 존재하는 인스턴스를 확인한다.
  bucket_name: "elasticbeanstalk-ap-northeast-2-869075270387"
  # 애플리케이션 이름과 동일하다.
  # 자세한 설명은 따로 찾아보자.
  bucket_path: "docker-react-app"
  # GitHub 설정
  on:
    # 어떤 branch에서 Push 이벤트 발생 시 AWS에 배포할 것인지
    branch: master
```

>   철자에 주의하자.

### 8. Travis CI의 AWS 접근을 위한 API 생성

현재까지 Travis CI에서 AWS에 어떤 파일을 전해줄 것인지,
AWS에서 어떤 서비스를 이용할 것인지 부수적인 설정을 했다.

Travis와 AWS가 실질적으로 소통을 할 수 있게 인증 부분을 작성한다.

#### 소스 파일을 전달하기 위한 접근 요건

GitHub → Travis CI → AWS

-   Travis CI 아이디 로그인 시 GitHub 연동으로 인증
-   AWS에서 제공하는 Secret Key를 `.travis.yml`에 작성

#### Secret, Access API Key 발급

인증을 위해 API Key를 발급해야 한다.

##### 1. IAM USER 생성

##### IAM(Identity and Access Management)

-   AWS 리소스에 대한 액세스를 안전하게 제어할 수 있는
    웹 서비스
-   IAM을 사용하여 리소스를 사용하도록 인증(로그인) 및
    권한 부여된 대상을 제어
-   Root 사용자
    -   현재 우리가 처음 가입하여 사용하고 있는 계정
    -   AWS 서비스 및 리소스에 대한 완전한 액세스 권한이 있음
-   IAM 사용자
    -   root 사용자가 부여한 권한만 가진다.

>   보안을 위해 새로 계정을 생성하는 것

Dashboard → IAM 검색 → 사용자 클릭 → 사용자 추가 클릭

##### 2. API Key를 `.travis.yml`에 작성하기

-   직접 API Key를 `.travis.yml`에 작성하면 노출되므로
    다른 곳에 작성하고 그것을 가져와야 한다.
-   Travis 웹사이트 해당 저장고 대시보드 방문
-   설정 클릭
-   AWS에서 받은 API Keys를 Name과 Value에 적어서 관리한다.
-   Travis CI 웹사이트에서 보관 중인 Keys를
    로컬 환경에서 가지고 올 수 있게 `.travis.tml` 파일에 설정

```yml
deploy:
  ...
  # Key(Travis의 환경 변수) 등록
  access_key_id: $AWS_ACCESS_KEY
  secret_access_key: $AWS_SECRET_ACCESS_KEY
```
