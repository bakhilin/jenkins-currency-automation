# Jenkins Multibranch Pipeline Documentation

<p align="center">
    <img src='docs/img/treasure-island-dr-livesey.gif'>
</p>


Данная автоматизация была создана для проекта по предоставлению информации по курсу валют. `currency-rest-api`

## Оглавление
1. [Introduction](#overview)
2. [Pipeline Stages](#pipeline-stages)
   - [CLONE](#clone)
   - [LINT ](#lint)
   - [BUILD](#build)
   - [SCAN and LOGIN](#scan-and-login)
   - [DEPLOY and DELETE](#deploy-and-delete)
3. [Shared Library](#shared-library)
4. [Results](#screenshots)
5. [How to Use](#how-to-use)

---

## Introduction
Был создан проект Multibranch Pipeline для автоматизации сборки и деплоя проекта на публичный реестр Докера. Для написания `Jenkinsfile` был использован `scripted pipeline` стиль. Добавлено параллельное выполнение некоторых этапов. 

![Overview Stages](docs/img/stages.png)

---

## Pipeline Stages

### CLONE
- **Описание**: Клонирование репозитория.
- **Код**:
  ```groovy
  stage('CLONE repo') {
      buildSuccess = pipeline_utils.errorHandler(STAGE_NAME){
          checkout scm
      }
  }
  ```
- **Результат**:  
  ![CLONE repo](docs/img/clone-stage.png)

---

### LINT 
- **Описание**: Линтер `hadolint` для Dockerfile. 
- **Код**:
  ```groovy
  stage('LINT dockerfile') {
      try {
          sh 'hadolint Dockerfile'
          echo "Dockerfile is valid"
      } catch(err) {
          echo "Linting errors: ${err.getMessage()}"
          currentBuild.result = 'UNSTABLE'
      }
  }
  ```
- **Результат**:  
  ![LINT Dockerfile](docs/img/lint-stage.png)

---

### BUILD 
- **Описание**: Сборка докер образа из Dockerfile.
- **Код**:
  ```groovy
  stage('BUILD Docker Image') {
      if (buildSuccess) {
          buildSuccess = pipeline_utils.errorHandler(STAGE_NAME){
              dockerUtils.buildImage(dockerImageName, dockerImageTag)
          }
      }
  }
  ```
- **Результат**:  
  ![BUILD Docker Image](docs/img/build-stage.png)

---

### SCAN and LOGIN - Parallel Stages
- **Описание**: 
  - **SCAN image by Trivy**: Поиск уязвимостей в собранном образе (сделать информативный этап, без учета ошибок).
  - **Login to Docker Hub**: Логин в DockerHub по установленным секретам (credentials)Jenkins.
- **Код**:
  ```groovy
  stage('SCAN and LOGIN') {
      parallel(
          'SCAN image by Trivy': {
              dockerUtils.scanImageTrivy(dockerImageName, dockerImageTag)
          }, 
          'Login to Docker Hub': {
              if (buildSuccess) {
                  buildSuccess = pipeline_utils.errorHandler(STAGE_NAME) {
                      dockerUtils.loginToDockerHub('docker-hub-credentials')
                  }
              }
          }
      )
  }
  ```
- **Результат**:  
  ![SCAN](docs/img/scan-stage.png)
  ![LOGIN](docs/img/login-stage.png)
---

### DEPLOY and DELETE - Parallel Stages
- **Описание**:
  - **DEPLOY Docker Image**: Публикация образа в реестр.
  - **DELETE bad images**: Удаление не нужных образов на виртуальной машине.
- **Код**:
  ```groovy
  stage('DEPLOY and DELETE') {
      parallel(
          'DEPLOY Docker Image': {
              if (buildSuccess || currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                  buildSuccess = pipeline_utils.errorHandler(STAGE_NAME) {
                      dockerUtils.pushImageDockerHub(dockerImageName, dockerImageTag)
                  }   
              }                        
          },
          'DELETE bad images': {
              pipeline_utils.errorHandler(STAGE_NAME){
                  dockerUtils.deleteImages()
              }
          }
      )
  }
  ```
- **Результат**:  
  ![DEPLOY and DELETE](docs/img/deploy-delete-stage.png)

---

## Shared Library
Исходный код находится по адресу [код](https://gitlab-pub.yadro.com/devops/2025/k.oreshin/shared_libraries_group/shared_libraries/-/tree/n.bakhilin/jenkins-shared-library?ref_type=heads)


- **`DockerUtils`**: Выполнение операций Docker (build, scan, login, push, delete).
- **`pipeline_utils`**: Функции для обработки ошибок и логирования.


### DockerUtils 

```groovy
package org.currency

class DockerUtils implements Serializable {
    def steps

    DockerUtils(steps) {
        this.steps = steps
    }

    // Builds a Docker image
    void buildImage(String imageName, String tag) {
        steps.sh "docker build -t ${imageName}:${tag} ."
    }

    // Scans a Docker image using Trivy
    void scanImageTrivy(String imageName, String tag) {
        steps.sh "trivy image ${imageName}:${tag}"
    }

    // Logs into Docker Hub using credentials
    void loginToDockerHub(String credentialsId) {
        steps.withCredentials([steps.usernamePassword(
            credentialsId: credentialsId,
            passwordVariable: 'DOCKER_HUB_PASSWORD',
            usernameVariable: 'DOCKER_HUB_USERNAME'
        )]) {
            steps.sh '''
                echo $DOCKER_HUB_PASSWORD | docker login -u \
                    $DOCKER_HUB_USERNAME --password-stdin
            '''
        }
    }

    // Pushes a Docker image to Docker Hub
    void pushImageDockerHub(String imageName, String tag) {
        steps.sh "docker push ${imageName}:${tag}"
    }

    // Deletes Docker images
    void deleteImages() {
        def wasteImages = steps.sh(script: 'docker images --filter "dangling=true" -q', returnStdout: true).trim()
        if (wasteImages) {
            steps.sh "docker rmi -f ${wasteImages}"
        } else {
            steps.echo "No dangling images found."
        }
    }
}
```

#### Ключевые методы:
- **`buildImage`**: Собирает докер образ.
- **`scanImageTrivy`**: Сканирует докер образ на уязвимости и предоставляет информацию.
- **`loginToDockerHub`**: Логин в DockerHub.
- **`pushImageDockerHub`**: Публикация образа на публичный реестр.
- **`deleteImages`**: Удаление не нужных образов.


### Служебные функции



#### `errorHandler`
Блок перехвата и обработки ошибки при выполнении stage. Если ошибка произошла, устанавливается статус FAILURE. 

```groovy
def errorHandler(String stageName, Closure body) {
    try {
        body()
    } catch(err) {
        echo "Error in stage ${stageName}: ${err}"
        currentBuild.result = 'FAILURE'
        return false // Indicates failure
    }
    return true // Indicates success
}
```

#### `logBuildInfo`
Информация по сборке.

```groovy
def logBuildInfo() {
    echo """
        Build ID: ${env.BUILD_ID} \n
        Build Tag: ${env.BUILD_TAG} \n
        Workspace: ${env.WORKSPACE} \n
    """
}
```

---

## Статусы сборок
1. **Unstable build**:  
   ![Unstable build](docs/img/unstable-status.png)

2. **Successful Build**:  
   ![Successful Build](docs/img/successfull-status.png)

3. **Failed Build**:  
   ![Failed Build](docs/img/failure-status.png)

