# Docker + Jenkins CI/CD

내 로컬 서버에 내가 만든 war를 배포하려고 한다

이때 지속적인 통합과 배포를 위해서 CI/CD를 지원하는 `Jenkins`를 이용해서 서버에 배포해보도록 하겠다

[Jenkins란](/chapter2/_posts/jenkins/2024-12-4-Jenkins.md)

## 1. Jenkins 이미지를 받아 도커 컨테이너를 생성

```docker
docker image pull jenkins/jenkins:jdk17
```

- 스프링 부트가 3 이상 버전이기 때문에 jdk17 이상부터 지원이 된다

```bash
docker pull jenkins/jenkins
```

- 버젼을 입력하지 않으면 `latest` 버전을 사용하게 된다

```bash
docker run -d -p 8888:8080 -p 50001:50000 --restart=on-failure --name auth-backend jenkins/jenkins:lts-jdk17
```

- **`run` :**기본적으로 도커에서 이미지 생성 시 사용되는 명령어\
- **`-d`** : detach 모드. 현재 우리가 실행하고 있는 콘솔, 터미널과 분리해서 실행하겠다는 의미. 즉 백그라운드(데몬 형태)로 기동하겠다는 의미, 터미널을 빠져 나와도 컨테이너는 실행된다
- `-p`: 컨테이너 내부에 있는 포트를 컨테이너 밖의 환경에서 어떻게 접속해서 사용할 것인지를 나타내는 설정이다.
    - 8888:8080 은 컨테이너 밖에서 `8888`(앞)으로 접속하면 컨테이터 내부의 `8080`(뒤)로 접속하겠다 라는 의미가 된다
- **`lts-jdk17`**: tag 이름 lst-jdk17 을 쓰겠다는 말이다
- **`--name`** : 만들고자 하는 컨테이너에 이름을 부여하는 옵션. 이 옵션이 없다면 도커가 랜덤한 이름을 생성해 부여한다. 즉 해당하는 컨테이너를 알 수 없게 된다. 그러므로 추천되는 옵션

이후 컨테이너를 확인하면

![docker_jenkins_1_1.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_1.png)

- `auth-backend` 컨테이너가 실행중인걸 확인할 수 있다
    - 방금 시작했기 때문에 `STAUTS`가 최근임을 알 수 있다

## 2. Jenkins 구성

내가 설정한 http://localhost:8888로 접속해보면

![docker_jenkins_1_2.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_2.png)

해당 페이지가 나오게 된다

페이지에서 알려주는 경로로 이동해 비밀번호를 입력해주면 된다

![docker_jenkins_1_3.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_3.png)

이후 모든 플러그인을 설치한다

![docker_jenkins_1_4.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_4.png)

이후 계정 생성을 요청하는 페이지에서 계정을 생성한다

![docker_jenkins_1_5.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_5.png)

이후 메뉴에서 새로운 Item 메뉴를 선택해 새로운 프로젝트를 생성한다

![docker_jenkins_1_6.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_6.png)

프로젝트 이름을 선택하고 freestyle project를 생성한다

[freestyle vs pipeline](..%2F..%2F..%2Fchapter2%2F_posts%2Fjenkins%2F2024-12-5-freestyle_vs_pipeline.md)

소스코드 관리에서 Git을 선택 후 리포지토리 주소를 입력해준다

![docker_jenkins_1_7.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_7.png)

- Github 정보 입력후 빌드 유발에서 GitHub hook trigger for GITScm polling 선택
    - Github의 hook trigger을 받으면 빌드를 하겠다는 뜻이다

![docker_jenkins_1_8.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_8.png)

- 목록중에 `Excute shell`을 선택한다
- 빌드 단계에서 실행할 `profile`을 `dev`로 지정했다
- gradle로 빌드 후 war 파일이 생성되면 auth-backend 라는 이름으로 변경하도록 했다

![docker_jenkins_1_9.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_9.png)

- 빌드 후 조치에서 `Deploy war/ear to a container` 선택 빌드 후 배포까지 진행했다
- `“**/”.war”`
    - 빌드 된 war 파일을 대상으로 선택
- Credentails 생성
- Tomcat이 설치되어 있는 대상을 적어준다

저장 후 apply를 하고 지금 빌드를 실행해서 빌드에 성공하면 이런 ui를 확인 할 수 있다

![docker_jenkins_1_10.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_10.png)

빌드 시 발생한 로그는 좌측 메뉴에서 확인 할 수 있다

![docker_jenkins_1_11.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_11.png)

내부 서버에 war 파일이 생성되고 디렉토리가 생성되면서 배포에 성공한다

![docker_jenkins_1_12.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_12.png)

![docker_jenkins_1_13.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_13.png)

![docker_jenkins_1_14.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_14.png)

## 3. Redis 컨테이너 구성

Auth 서버는 `Redis`를 연동해서 사용하고 있기 때문에 별도의 `Redis` 서버를 구축해야 한다

```bash
docker pull redis
```

- Redis 이미지를 받아온다

```bash
$ docker run -p 6379:6379 --name redis-server -d redis:latest 
```

- Redis를 컨테이너로 실행시긴다

이후 Jenkins 컨테이너와 Redis 컨테이너가 서로 통신하기 위해서 동일한 Docker network을 구성해야한다

Docker 네트워크는 기본적으로 생성되는 `bridge` 네트워크 외에도, 사용자가 직접 생성할 수도 있다

![docker_jenkins_1_15.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_15.png)

```bash
docker network connect auth-network redis-server
docker network connect auth-network auth-backend
```

**docker network connect {네트워크명} {컨테이너명}**을 입력해 컨테이너와 네트워크를 연결한다.

- [Docker 네트워크](/chapter2/_posts/docker/2024-12-17-Docker_네트워크.md)

```bash
docker network inspect auth-network
```

- 두 컨테이너를 연결한 `auth-network`의 세부 정보를 출력하면

![docker_jenkins_1_16.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_16.png)

- 위 그림과 같이 `auth-backend`와 `redis-server` 컨테이너가 속해있는 것을 확인할 수 있다

## 3-1. Compose 파일 구성

**[Docker Compose란?](/chapter2/_posts/docker/2024-12-17-Docker_Compose.md)**

**redis Dockerfile 구성**

```docker
FROM redis:7.4.1-alpine

EXPOSE 6379

ENV REDIS_DEFAULT_CONFIG_FILE=./volumes/redis/redis.conf
ENV REDIS_DATA_PATH=./volumes/redis/_data

COPY $REDIS_DEFAULT_CONFIG_FILE /usr/local/etc/redis/redis.conf

RUN echo $REDIS_DEFAULT_CONFIG_FILE
RUN echo $REDIS_DATA_PATH

CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]
```

- 컨테이너로 실행시킬 `Redis`를 Dockerfile로 정의했다
- `EXPOSE 6379` : 6379 포트를 열어준다는 의미이다
- `ENV REDIS_DEFAULT_CONFIG_FILE`, `ENV REDIS_DATA_PATH` : volume 파일과 redis.conf 파일을 마운트 할 위치를 환경 변수로 선언했다
- `COPY $REDIS_DEFAULT_CONFIG_FILE /usr/local/etc/redis/redis.conf` : 호스트 서버에 있는 redis.conf 파일을 컨테이너 서버에 복사한다
- `CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]` : 컨테이너가 생성되면서 `redis.conf` 설정을 기반으로 서버가 실행된다

**docker-compse**

```yaml
version: "3.8"

services:
  redis:
    image: my-redis:1.0.2
    container_name: redis-server
    ports: 
      - "6379:6379"
    volumes: 
      - ./redis-data:/data
    networks:
      - jenkins-redis-networks

  jenkins:
    image: jenkins/jenkins
    container_name: jenkins-server
    ports:
      - "8888:8080"
    networks:
      - jenkins-redis-networks
    extra_hosts:
      - host.docker.internal:host-gateway

networks:
  jenkins-redis-networks:
    driver: bridge
```

- 실행시킬 Redis 이미지와 Jenkins 이미지를 서비스 별로 작성했다
- Redis를 연동하기 위해서는 `volume` 필요하다
- redis 컨테이너를 종료 후 재시작하면, 기존 `redis`에 저장해둔 데이터가 보존되지 않는다
    - 만약 빌드 및 배포가 진행되면 `docker compose down` 후 다시 `docker compose up`을 통해 같이 재시작 될 수 있다
    - 호스트 서버에 디렉토리에 `Redis` 서버 data를 마운트하여 데이터를 유실하지 않고 저장할 수 있게 했다
- `extra_hosts`: 컨테이너의 `/etc/hosts`에 외부 호스트 정보를 추가할 수 있다
    - 외부 호스트 서버에 접근할 수 있게 추가했다
- `networks: jenkins-redis-networks:` : Jenkins 컨테이너redis 컨테이너로 요청이 있기 때문에 서로 같은 네트워크로 구성했다

  ![docker_jenkins_1_17.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_17.png)

    - 컨테이너는 원래 `docker0` 이라는 기본 브릿지를 통해 외부와 통신하게 된다
    - `eth0` : 호스트의 eth0은 실제 외부와 연결할 때 사용하는 ip가 할당된 호스트 네트워크 인터페이스이다
    - `veth`:컨테이너의가상 네트워크 인터페이스이다
        - 이 veth를 호스트의 eth0과 연결시키므로써 외부와 통신이 가능하다
    - `docker0` : 도커가 설치될 때, 기본적으로 구성되는 브릿지이다. 이 브릿지의 역할은 호스트의 네트워크인 `eth0`과 컨테이너의 네트워크와 연결을 해주는 역할을 한다

  ![docker_jenkins_1_18.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_18.png)

    - 이렇게 새로운 브릿지로 컨테이너를 연결해주면 컨테이너 간의 서로 통신이 가능하다

## 4. Pipeline 구축

* [pipeline](/chapter2/_posts/jenkins/2024-12-6-pipeline.md) 

new Item에서 Pipeline을 선택한다

![docker_jenkins_1_18.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_18.png)

```groovy
pipeline {
    agent any
    environment {
        TOMCAT_USER = 'deployer'
        
        REMOTE_TOMCAT_URL = ''
        REMOTE_INFO = ''
        REMOTE_PORT = 22
       
        REMOTE_TOMCAT_DIR = '/home/bhkim/tomcat'
        
        AUTH_BACKEND = 'AUTH-BACKEND'
    }
    stages {
        stage('Git clone') {
            steps {
                git branch: 'main', url: 'https://github.com/bhkim66/Auth.git'
            }
        }
        stage('build') {
            steps {
                echo 'Build Start'
                sh '''chmod +x gradlew
                    export SPRING_PROFILES_ACTIVE=dev
                    ./gradlew clean build
                    mv build/libs/*.war build/libs/${AUTH_BACKEND}.war'''
            }
        }
        stage('deploy') {
            steps {
                 echo 'Deploy Start ${AUTH_BACKEND}'
                 sh 'ls -l build/libs/' // 디버깅용
                 sh 'scp -r build/libs/${AUTH_BACKEND}.war ${REMOTE_INFO}:${REMOTE_TOMCAT_DIR}/webapps'
                 // sh 'ssh -p ${REMOTE_PORT} ${REMOTE_INFO} \'sh ${REMOTE_TOMCAT_DIR}/bin/deploy.sh > /dev/null 2>&1 '
            }
        }
    }
}
```

- environment에 공통으로 사용될 변수들을 선언해줬다
- 각각 스테이지의 구성은 이렇다
    - `Git clone`
        - 내 깃 저장소에서 소스코드를 가져온다
    - `build`
        - 가져온 코드를 바탕으로 `gradle을` 통해 빌드를 실행한다
    - `deploy`
        - 빌드가 완료되어 생성된 war는 Jenkins 도커에 저장되어있다
        - 호스트 서버의 tomcat으로 war을 이동시켜야 하기 때문에 scp 명령어를 통해서 전달한다

호스트 서버에서 war파일을 전달 받아야 하기 때문에 ssh로 접속하여 전달 받아야한다

```bash
sudo apt install openssh-server
```

- SSH Server을 설치한다

이후 22번 포트의 방화벽을 허용한다

```bash
sudo ufw allow 22
```

![docker_jenkins_1_19.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_19.png)

이후 Jenkins 서버에 접속해 내 호스트 서버와의 통신을 위한 설정을 한다

```bash
ssh-keygen -t rsa
```

- Jenkins 서버에 접속 후 key를 발급 받는다

```bash
ssh-copy-id bhkim@172.23.158.28
```

- 로컬의 공용 키를 원격 호스트의 .`ssh/authorized_keys` 파일에 복사한다.
- 이를 통해 호스트 서버에 접속이 가능해진다

모든 설정을 마치고 Jenkins에 빌드를 실행하면 빌드 및 배포에 성공한다

![docker_jenkins_1_20.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_20.png)

- Stage View 또한 Plugin에서 다운로드 받아 사용해야 한다

### ※ 추가 push시와 동시에 pipeline 실행하기

![docker_jenkins_1_21.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_21.png)

- 빌드가 완료된 이후 Gtihub webhook 기능을 통해 코드가 push 이벤트가 발생하면 이를 감지해 자동으로 빌드 및 배포를 실행한다

![docker_jenkins_1_22.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_22.png)

- Github Setting에서 Webhooks 메뉴를 선택한다
- Jenkins가 설치된 URL을 입력 후 뒤에 `/github-webhook/` path를 추가한다
- 나는 내 로컬 pc에 Jenkins를 설치했기 때문에 가상의 도메인 주소가 필요하다
    - 이를 위해 `ngrok` 이라는 서비스를 사용했다

**Ngrok**

- 로컬 개발환경에서 인터넷을 통해 웹 어플리케이션에 안전하게 접근할 수 있도록 해주는 도구이다
- 보안 연결을 통해 인터넷에서 서버를 실행할 수 있으며, 웹 어플리케이션을 외부에 노출시키지 않고 테스트를 할 수 있다

Ngrox를 설치 후 회원가입 후 실행을 시키면 커맨드 창이 뜬다

```bash
ngrok http 8888
```

- Jenkins가 설치된 포트를 임시 도메인을 연결하여 외부에서 접근할 수 있도록 한다

![docker_jenkins_1_23.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_23.png)

- 임시 도메인이 localhost:8888로 향하는 것을 확인할 수 있다
- 이후 push 이벤트가 발생하면 Jenkins가 동작하는 것을 확인할 수 있다

![docker_jenkins_1_24.png](/assets/img/chapter1/docker_jenkins/docker_jenkins_1_24.png)

- Webhook 성공