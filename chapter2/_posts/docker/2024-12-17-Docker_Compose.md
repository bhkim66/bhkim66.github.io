# Docker Compose

Docker Compose는 다중 컨테이너 애플리케이션을 정의 ****공유할 수 있도록 개발된 도구로 단일 명령을 사용하여 모두 실행 또는 종료 할 수 있도록 개발된 도구이다

```bash
services:
  nhn_app:
    image: nhn_app_v2

  nhn_mysq:
    image: mysql
```
- 이런식으로 하나의 문서에 여러 개의 컨테이너를 정의할 수 있다
### 특징
### 단일 호스트의 여러 격리된 환경

- Compose는 프로젝트 이름을 사용하여 환경을 서로 격리하고 여러 다른 콘텍스트에서 이 프로젝트 이름을 사용하여 접근을 한다
- 예를 들어 `application`이라는 서비스를 실행시킬 때 `my_application`, `your_application` 형식으로 여러 개를 서로 격리하여 서비스가 가능하다

## Volume

컨테이너 **삭제의 경우에만 자체 파일 시스템이 사라지게 되는 특성** 때문에, 코드 수정 이후 코드를 실행을 위한 이미지 리빌딩, 컨테이너 리빌딩의 경우 데이터를 저장할 수 없는 상황이 발생하게 된다

이를 위해 `Volume`을 사용한다

- `volume`이란 호스트 머신의 폴더이다
    - 도커가 인식하는 호스트 머신
    - `volume`은 도커 컨테이너 내부의 폴더에 매핑된다
- 볼륨은 컨테이너 내부 폴더와 컨테이너 외부 폴더를 연결하는 것이다

**종류**

- **익명 볼륨**
    - 컨테이너가 존재하는 동안에만 실제로 존재하는 볼륨이다
- **명명 볼륨**
    - 컨테이너가 컨테이너가 종료된 후에도 볼륨이 유지가 되는 것을 의미한다

### 마운트

물리적 장치를 특정 디렉터리에 연결시켜주는 것을 의미한다

**도커의 3가지 마운트 방식**

- Voluem 마운트
- 바인드 마운트
- tmfs 마운트

### Volume 마운트

- 도커가 생성하고 관리하는 방식
- 도커에 의해 볼륨이 생성되고 도커에 의해 관리되는 방식으로 볼륨이 로컬 디렉토리에 마운트 될 경우 바인드 마운트와 유사하게 작동한다
- 볼륨을 생성하면 자동으로 아래의 경로에 볼륨이 마운트된다
    - `/var/lib/docker/volumes/`
- `Voluem` 마운트는 아래의 경로에 `Volume`이 생성되고 해당 `Volume`을 도커와 연결시켜 관리하는 것이다

![docker_compose_1.png](/assets/img/chapter2/docker/docker_compose_1.png)

### 바인드 마운트

- 호스트의 로컬경로를 직접 지정하여 볼륨을 마운트하는 방식이다
- 해당 방식은 *도커가 아니기 때문에*, 도커프로세스와 non-도커프로세스 간의 차이가 발생한다

### 도커 포트

만약 `Dockerfile`로 이미지를 만들어서 포트를 지정했을 경우 해당 포트를 compose에도 일치하게 작성해줘야 한다

```bash
FROM redis:7.4.1-alpine

EXPOSE 6379

ENV REDIS_DEFAULT_CONFIG_FILE=./volumes/redis/redis.conf
ENV REDIS_DATA_PATH=./volumes/redis/_data

COPY $REDIS_DEFAULT_CONFIG_FILE /usr/local/etc/redis/redis.conf

RUN echo $REDIS_DEFAULT_CONFIG_FILE
RUN echo $REDIS_DATA_PATH

VOLUME ["/data"]

CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]

```

- `Dockerfile`6379로 지정된 포트

```bash
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
```

- compose에도 똑같은 포트로 연결 시켜줘야 한다
