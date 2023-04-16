# GitHub Action을 활용한 CICD 적용하기(1)
GitHub Action을 활용해서 CI/CD를 구현하고자 하였는데, 여러 가지 문제점이 있었고 이를 해결하는 과정을 기록해두면 좋을까 싶어 해당 과정을 전부 기록하고자 한다. <br><br>
## 1. CI/CD with GitHub Action
GitHub Action의 경우 Github과 매우 좋은 연동성을 보여주며 사용함에 있어 러닝 커비가 낮기 때문에 CI/CD 툴로써 많이 사용한다고 하여, 이를 적용하고자 하였다. 또한 AWS S3와 AWS CodeDeploy 역시 CD를 위해서 활용하였다.

목표하고자 하는 Flow는 다음과 같다.
> 1. main branch에 push를 감지한 다음 GitHub Action의 스크립트가 작동
> 2. main branch의 코드를 기반으로 Gradle build를 통한 jar 파일 생성
> 3. jar 파일과 CodeDeploy를 위한 Scripts, YML과 함께 zip 파일 생성
> 4. zip 파일을 AWS S3에 업로드
> 5. AWS CodeDeploy에서 AWS S3의 zip 파일을 활용해 AWS EC2 서버에서의 배포

우선, AWS S3의 새로운 버킷을 하나 만들어야 한다. 버킷을 만드는 과정에 대해서는 서술하지 않겠다. 버킷은 굳이 public한 객체로 만들 필요는 없다. 버킷을 만든 다음, 해당 버킷에 Full Access Permission을 가지고 있는 IAM을 만들고, 해당 IAM의 두 개의 Key를 잘 저장해둔다.

또한 AWS CodeDeploy에도 하나의 어플리케이션을 만든다. 그리고 S3를 만들때 만들었던 IAM에 AWS EC2 Full Access Permission을 부여해주고, CodeDeploy와 배포중인 서버의 AWS EC2와도 연결해준다. 여기까지의 과정은 크게 어렵지 않기 때문에 별 다른 기록을 남기진 않는다.

처음에는 build 시에 test가 필요하지 않을 것이라고 생각했다. GitHub Action에서 test를 진행하기 위해서는 아래 두 가지 문제점이 존재했다.

1. GitHub Action의 환경을 AWS EC2와 동일하게 생성하지 않으면 테스트하는 의미가 없을 뿐더러 테스트 자체가 불가능하다.
2. AWS EC2에서 nohup 키워드를 사용해서 배포를 진행할 때 build 하게 되면서 어차피 테스트가 일어난다.

이러한 문제점 때문에 처음에는 build 시에 테스트를 제외했으나, 안일한 생각이었고 결국 테스트를 진행할 수 있도록 공부해서 구현하였다. 이에 대해서는 뒤에 다시 설명하고, 우선은 test를 배제하고 build를 할 수 있도록 하였다.

이어서, AWS CodeDeploy에서 사용할 scripts 파일들과 YML 파일을 작성할 것이다. Blue-Green 배포 전략을 사용하기 위해서, 다음과 같이 구현한다.

> 1. 현재 배포중인 서버의 profile을 조회한다. 
> 2. 현재 구동중이나 배포중이지 않은 포트의 프로세스를 종료한다.
> 3. 사용하지 않는 포트에 사용중이지 않은 profile과 함께 서버를 구동한다.
> 4. Nginx의 port forwarding 엔트포인트를 방금 구동한 서버로 바꾼다.

예를 들어서 real1.properties와 real2.properties가 있다면, 각각의 properties 파일에는 서버를 구동할 포트가 정해져 있다. (나는 8081 / 8082 로 사용하였다.) 이후 현재 active 중인 profile을 조회할 수 있다면, 현재 배포중인 서버의 포트 넘버를 알 수 있고 따라서 사용중이지 않은 포트의 프로세스를 종료한 후 해당 포트에 새롭게 서버를 구동하고 엔드포인트를 바꾸는 식으로 작동할 수 있을 것이다.

우선 현재 배포중인 서버에서 사용중인 profile에 대한 정보를 알아와야 하는데, 해당 코드는 아래와 같다.

```kotlin
import org.springframework.core.env.Environment
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class ProfileController(
    private val env: Environment
) {
    @RequestMapping("api/v2/profile")
    fun profile(): String {
        val profiles: List<String> = env.activeProfiles.asList()
        val realProfiles: List<String> = listOf("real", "real1", "real2")
        val defaultProfile: String = profiles.get(0)
        return profiles.stream()
            .filter(realProfiles::contains)
            .findAny()
            .orElse(defaultProfile)
    }
}
```
env 객체에서 현재 활성화 되어 있는 profile을 얻어올 수 있다. 또한 nohup 키워드를 활용해서 활성화 하고자 하는 profile을 지정할 수 있기 때문에, 위와 같이 profileController에서 현재 구동중인 서버의 profile을 얻어올 수 있게 되는 것이다.

이어서 AWS CodeDeploy에서 사용되는 스크립트들은 [여기](https://github.com/Entrip-Ajou/2ntrip-Backend-Refactor/tree/main/scripts)에서 확인할 수 있다. 총 5개의 스크립트를 사용할 것이고, 역할은 아래와 같다.

> * profile.sh : 현재 배포중인 포트 (또는 프로파일) 을 조회
> * stop.sh : 현재 배포중이지 않은 포트의 프로세스 종료
> * start.sh : 현재 사용중이지 않은 포트에 새로운 서버 구동
> * switch.sh : 방금 구동한 새로운 서버로 Nginx의 엔드포인트 수정
> * health.sh : 새롭게 구동한 서버의 profile을 조회해서 제대로 작동하는지 확인

해당 scripts들은 projectDir/scripts 에다가 저장하고, GitHub Action에서 build한 후 압축 시에 같이 넣어줄 수 있도록 한다.

여기까지 완료했다면, 아래와 같이 GitHub Action에 실행될 스크립트를 작성한다. 스크립트는 아래와 같다.
```shell
# This workflow will build a Java project with Gradle
# For more information see: https://help.github.com/actions/language-and-framework-guides/building-and-testing-java-with-gradle

# Repo Action 페이지에 나타날 이름
name: Spring Boot & Gradle CI/CD

# Event Trigger
# master branch에 push가 발생할 경우 동작
# branch 단위 외에도, tag나 cron 식 등을 사용할 수 있음

on:
  push:
    branches: [ main ]

# 환경 변수 설정
env:
  S3_BUCKET_NAME: 2ntrip-backend-refactor-deploy
  CODEDEPLOY_APPLICATION_NAME: 2ntrip-backend-refactor
  DEPLOYMENT_GROUP_NAME: 2ntrip-backend-refactor

jobs:
  build:
    # 실행 환경 지정
    runs-on: ubuntu-latest

    # Task의 sequence를 명시한다.
    steps:
      - uses: actions/checkout@v2

      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8

      # gradlew 파일에 실행 권한 부여
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      # Build
      - name: Build with Gradle
        run: ./gradlew clean build -i

      # 전송할 파일을 담을 디렉토리 생성
      - name: Make Directory for deliver
        run: mkdir deploy

      # Jar 파일 Copy
      - name: Copy Jar
        run: cp ./build/libs/*.jar ./deploy/

      # appspec.yml Copy
      - name: Copy appspec
        run: cp ./appspec.yml ./deploy/

      # script file Copy
      - name: Copy shell
        run: cp ./scripts/* ./deploy/

      # 압축파일 형태로 전달
      - name: Make zip file
        run: zip -r -qq -j ./2ntrip-api-refactor-deploy.zip ./deploy/

      # S3 Bucket으로 copy
      - name: Deliver to AWS S3
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          aws s3 cp \
          --region ap-northeast-2 \
          --acl private \
          ./2ntrip-api-refactor-deploy.zip s3://2ntrip-backend-refactor-deploy/

      # Deploy
      - name: Deploy
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          aws deploy create-deployment \
          --application-name $CODEDEPLOY_APPLICATION_NAME \
          --deployment-group-name $DEPLOYMENT_GROUP_NAME \
          --file-exists-behavior OVERWRITE \
          --s3-location bucket=$S3_BUCKET_NAME,bundleType=zip,key=2ntrip-api-refactor-deploy.zip \
          --region ap-northeast-2
```

Shell 스크립트이기에 크게 어려운 점은 없다. 다만, 디렉토리들을 제대로 설정하지 않으면 에러가 발생할 수 있기에 반드시 제대로 디렉토리를 생성했는지 (AWS S3 및 AWS EC2) 반드시 확인해야 한다.

이어서 AWS CodeDeploy에서 사용하기 위한 스크립트를 작성해야 하는데, 이 역시 크게 어려운 점이 없음으로 스크립트 첨부로 설명을 대체한다. 마찬가지로 디렉토리 작성에 신경을 쓸 필요가 있다.

```Shell
version: 0.0
os: linux
files:
  - source: /
    destination: /home/ec2-user/deploy/zip/
    overwrite: yes

permissions:
  - object: /
    pattern: "**"
    owner: ec2-user
    group: ec2-user

hooks:
  AfterInstall:
    - location: stop.sh # nginx와 연결되지 않은 스프링 부트 종료.
      timeout: 60
      runas: ec2-user
  ApplicationStart:
    - location: start.sh # nginx와 연결되어 있지 않은 포트로 스프링 부트 시작.
      timeout: 60
      runas: ec2-user
  ValidateService:
    - location: health.sh # 새 서비스 health check.
      timeout: 60
      runas: ec2-user
```

여기까지 완료했다면 CI/CD는 끝이다. 라고 생각했지만 큰 문제가 있었고, 어떤 문제였는지, 어떻게 해결했는지에 대해서 기록하고자 한다.

## 2. Test Spring Code with GitHub Action
위에서 서술한 바와 같이, 처음에는 test를 할 필요가 없다고 생각했다. 하지만 다음과 같은 문제가 있어 결국 test를 진행할 수 밖에 없었다.
> 1. test를 진행하기 위해서는 모든 환경 설정 및 properties 정보를 주어야 한다.
> 2. test를 생략하면 asciidoctor가 작동하지 못해서 build가 실패하게 된다.
> 3. 로컬에서 asciidoctor를 실행하고 그 결과인 html을 배포하는 것은 로컬 환경 기준에서의 테스트 결과이기 때문에 신뢰성이 떨어진다.
> 4. 결국 test를 진행해야 하고, 이를 위해선 **모든 환경 설정 및 properties 정보가 필요하다.**

따라서, GitHub Action에서 환경 설정 및 properties 정보를 주어야만 하는 상황이었다.

<br>


### 2-1. Properties Settings
기본적으로 .properties 파일들을 대게 민감 정보를 지니고 있는 경우가 많다. (DB 사용자 아이디 및 패스워드 등) 그래서 비전관리시 .properties 파일들은 대부분 gitIgnore에 등록해서 업로드 하지 않는 경우가 대부분인데, test를 위해서는 GitHub Action이 실행되는 환경에 properties 파일을 만들어야 했다. 하지만 AWS EC2가 아닌, 우리가 제어할 수 없는 환경이기 때문에 많은 어려움이 존재했다. 다양한 방법을 찾아본 끝에, 아래와 같은 Flow를 통해서 해결할 수 있었다.
> 1. .properties 파일들의 값을 GitHub Secrets에 등록한다.
> 2. GitHub Action 스크립트에서 touch 명령어를 사용해 .properties 파일을 생성한다.
> 3. GitHub Secrets의 .properties 파일의 값들을 echo로 출력한 다음, redirection을 통해서 touch로 생성한 properties 파일들에 출력한다.

해당 부분들을 스크립트에서 나타내면 다음과 같다.
```shell
      - name: Create Properties Files
        run: touch ./src/main/resources/application.properties ; touch ./src/main/resources/application-aws-s3.properties ; touch ./src/main/resources/application-mongodb.properties ; touch ./src/main/resources/application-redis.properties ; touch ./src/main/resources/application-security.properties ; touch ./src/test/resources/application-test.properties

      - name: Load Application Properties
        run: echo "${{ secrets.APPLICATION_PROPERTIES }}" > ./src/main/resources/application.properties

      - name: Load Application AWS S3 Properties
        run: echo "${{ secrets.APPLICATION_AWS_S3_PROPERTIES }}" > ./src/main/resources/application-aws-s3.properties

      - name: Load Application MongoDB Properties
        run: echo "${{ secrets.APPLICATION_MONGO_PROPERTIES }}" > ./src/main/resources/application-mongodb.properties

      - name: Load Application Redis Properties
        run: echo "${{ secrets.APPLICATION_REDIS_PROPERTIES }}" > ./src/main/resources/application-redis.properties

      - name: Load Application Security Properties
        run: echo "${{ secrets.APPLICATION_SECURITY_PROPERTIES }}" > ./src/main/resources/application-security.properties
```
<br>

### 2-2. Environment Settings
또한 test 코드에는 ApplicationTest_ContextLoad() 테스트가 있는데, 해당 테스트가 성공하기 위해서는 WAS를 구동하기 위한 모든 Environment (거의 DB)가 구성되어 있어야 한다. 따라서 DB를 셋팅하고자 하는데, 우리는 MYSQL, Redis, MongoDB, H2 이렇게 총 4가지의 DB를 사용하고 있었고, 이 중에서 H2를 제외한 나머지 DB는 모두 직접 실행해야 한다.

Shell Scripts를 활용해서 구동할 수 있겠지만, 이미 이렇게 구동할 수 있도록 Scripts를 만들고 난 다음 오픈 소스로 marketplace에 출시되어 있어서 해당 소스를 사용했다. DB 실행 뿐만 아니라 user, password, DB 등을 지정할 수 있었기에 편리하게 사용할 수 있었다. 또한 password 역시 민감 정보이기 때문에 GitHub Secrets에 등록한 다음 해당 값을 사용하였다. 해당 오픈소스들은 모두 Docker를 기반으로 작동하는 것 처럼 보였다. (로그에서 Docker를 사용하는 것 같은 모습을 보였다.)

```Shell
      - name: Redis Server in GitHub Actions
        uses: supercharge/redis-github-action@1.5.0      
      
      - name: MongoDB in GitHub Actions
        uses: supercharge/mongodb-github-action@1.9.0
        with:
          mongodb-username: wkazxf
          mongodb-password: ${{ secrets.MONGODB_PASSWORD }}
          mongodb-db: 2ntrip

      - name: Start MySQL
        uses: samin/mysql-action@v1.3
        with:
          mysql database: 2ntrip
          mysql root password: ${{ secrets.MYSQL_PASSWORD }}
          mysql user: 'root'
          mysql password: ${{ secrets.MYSQL_PASSWORD }}
```

막상 정리하고 보니 크게 어렵지 않은 것 같지만, 실제로 해당 문제를 해결하는데 꽤 오랜 시간이 걸렸다. 이렇게 test와 asciidoctor를 포함해서 CI/CD를 구축할 수 있었다.

이제는 CI/CD 상황에서 에러가 발생한다면 알림을 보내거나, CI가 실패하면 롤백하는 등의 과정을 차차 알아가면서 추가적으로 스펙업 할 예정이다.