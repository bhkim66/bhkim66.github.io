# Pipeline

`Pipeline`이란 말 그대로 파이프를 이어 붙인것과 같은 형태로 Step by Step 형식의 각 단계를 이어 붙여 실행하는 방식

- `Jenkins`에서의 `Pipeline`은 한 곳에서부터 작업을 시작해서 일렬로 혹은 여러 갈래로 뻗어져 나갔다가(`병렬처리`) 다시 한 쪽으로 모이면서 마무리되는 방식의 작업 흐름을 의미한다
- 이를 통해 지속 제공과 지속적인 배포가 가능하다

## Jenkins 파이프라인 Script 문법

### **Pipeline**

```livescript
pipeline {
    /* insert Declarative Pipeline here */
}
```

- 파이프라인을 정의하기 위해서는 반드시 pipeline을 포함해야한다
- 항상 최상위 레벨이 되어야 한다

### **Section**

- `agent`: agent를 추가할 경우 Jenkins는 해당 agent를 설정한다.
- `agent`는 pipeline의 최상위에 포함되어야 하며, agent가 none으로 작성되었을 경우 stage에 포함되어야 한다
    - ex) docker

    ```yaml
    agent {
        docker {
            image 'myregistry.com/node'
            label 'my-defined-label'
        }
    }
    ```

    - ex) agent가 적용 된 JenkinsFile Sample

    ```yaml
    pipeline {
        agent none
        stages {
            stage('Example Build') {
                agent { docker 'maven:3-alpine' }
                steps {
                    echo 'Hello, Maven'
                    sh 'mvn --version'
                }
            }
        }
    }
    ```

    - agent none이 pipeline의 최상위에 정의되어 있을 경우 stage는 각각 `agent`를 포함하여야 한다.

`post` : 특정 stage나 pipeline이 시작되거나 이전 또는 이후에 실행 될 `confition block`을 정의한다

- always, changed, fixed, regression, aborted, failure, success, unstable, unsuccessful,와 cleanup 등의 상태를 정의할 수 있다.

```livescript
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
    post { 
        always { 
            echo 'I will always say Hello again!'
        }
        success {
		        echo 'this is success'
        }
        failure {
		        echo 'this is fail'
        }
    }
}
```

`stages` : 하나 이상의 stage에 대한 모음을 정의한다

- pipeline 블록 안에서 한번만 실행 될 수 있다

```livescript
pipeline {
    agent any
    stages { 
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

### **Directives**

`environment` : key-value 형태로 파이프라인 내부에서 사용할 환경 변수로 선언할 수 있다

```livescript
pipeline {
    agent any
    environment {
        THIS_IS_HELLO = 'HELLO'
    }
    stages { 
        stage('Example') {
            steps {
                echo '${THIS_IS_HELLO}'
            }
        }
    }
}
```

**`parameters` :** 사용자가 제공해야 할 변수에 대해 선언할 수 있다

- string, text, booleanParam, choice, password 등을 정의할 수 있다.
- parameters는 pipeline에서 한번만 정의할 수 있다.

```livescript
pipeline {
    agent any
    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')

        text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')

        booleanParam(name: 'TOGGLE', defaultValue: true, description: 'Toggle this value')

        choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')

        password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    }
    stages {
        stage('Example') {
            steps {
                echo "Hello ${params.PERSON}"

                echo "Biography: ${params.BIOGRAPHY}"

                echo "Toggle: ${params.TOGGLE}"

                echo "Choice: ${params.CHOICE}"

                echo "Password: ${params.PASSWORD}"
            }
        }
    }
}
```

`triggers`: 파이프라인을 다시 트리거해야 하는 자동화된 방법을 정의한다

- cron, pollSCM, upstream 등 여러 방식으로 트리거를 구성할 수 있다

```livescript
pipeline {
    agent any
    triggers {
        cron('H */4 * * 1-5')
    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

`tools`: 설치 도구를 정의한다

- agent none으로 지정된 경우 무시한다
- 지원되는 도구는 maven, jdk, gradle이 있다
- tools 이름은 `Jenkins` 관리의 `Global Tool Configuration`에서 사전 정의되어 있어야 한다

```livescript
pipeline {
    agent any
    tools {
        maven 'apache-maven-3.0.1' 
    }
    stages {
        stage('Example') {
            steps {
                sh 'mvn --version'
            }
        }
    }
}
```

### Pipeline 평행 실행

```livescript
stage('Parallel Stage') {
      when {
          branch 'master'
      }
      failFast true
      parallel {
          stage('Branch A') {
              agent {
                  label "for-branch-a"
              }
              steps {
                  echo "On Branch A"
              }
          }
          stage('Branch B') {
              agent {
                  label "for-branch-b"
              }
              steps {
                  echo "On Branch B"
              }
          }
        }
}        
```

`parallel` : 파이프라인이 평행 실행을 할 수 있게 해준다

`when`: 조건문으로 사용할 때 사용한다