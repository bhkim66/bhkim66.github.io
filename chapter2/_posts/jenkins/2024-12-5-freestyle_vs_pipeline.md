# freestyle vs pipeline

## Freestyle Project

- `Freestyle project`는 jenkins에서 기본 틀을 정해주고 안에 빈칸을 채우는 형식으로 프로세스 흐름을 정의하는 방식이다

![jenkins_2_1.png](/assets/img/chapter2/jenkins/jenkins_2_1.png)

- `freestyle`의 설정화면
- 각 단계에서 형상 관리 도구와 어떻게 연결을 할 지 또는 빌드, 테스트 등의 구체적인 세부 내용은 어떻게 되는지를 직접 작성해 주면 된다

**장점**

- 진입 장벽이 낮음

단점

- 커스터마이징이 제한적
- 테스트 혹은 빌드 등의 작업에서 병렬처리 미지원
- 다수의 형상관리 Repository와 연계하여 사용할 수 없음

## Pipeline

- `Jenkins`에서의 `Pipeline`은 한 곳에서부터 작업을 시작해서 일렬로 혹은 여러 갈래로 뻗어져 나갔다가(`병렬처리`) 다시 한 쪽으로 모이면서 마무리되는 방식의 작업 흐름을 의미한다

![jenkins_2_2.png](/assets/img/chapter2/jenkins/jenkins_2_2.png)

**장점**

- 커스터마이징의 폭이 넓다
- 테스트 혹은 빌드 등의 작업에서 병렬 처리 지원
- 다수의 형상관리 Repository와 연계하여 사용할 수 있다

단점

- 진입 장벽이 높다

### Groovy

- Pileline 방식에서 모든 절차를 작성하는 언어이다
- **자바 가상 머신**에서 작동하는 **동적 타이핑 프로그래밍 언어**이다
- 자바의 강점 위에 파이썬, 루비, 스몰토크등의 프로그래밍 언어에 영향을 받은 특장점을 더하였다

참고 - 인프런 / Jenkins를 이용한 CI/CD Pipeline 구축