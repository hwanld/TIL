#  Spring Environment와 getActiveProfiles()
## 1. Environment 인터페이스란?
<br>

Github Action 및 CICD를 활용해서 Blue-Green 배포 전략을 짜던 중, 현재 프로젝트에 적용중인 `profiles` 들을 전부 조회해야 하는 필요가 있었다. 여러 예제 코드를 찾아보던 중, 대부분의 코드가 Environment 인터페이스로부터  `getActiveProfiles` 메소드를 사용해서 현재 작동중인 `profiles`를 조회하는 것을 확인할 수 있었다. Environment 인터페이스가 어떤 역할을 하는 것이고, 어떤 기능을 제공하는 것인지 등에 대해서 더욱 알고 싶어서 Spring Reference를 찾아보게 되었다.


Spring 공식 문서에는 아래와 같이 기술되어 있다. <br>
> `public interface Environment` extends [PropertyResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/PropertyResolver.html)<br><br>
> Interface representing the environment in which the current application is running. Models two key aspects of the application environment: *profiles* and *properties*. Methods related to property access are exposed via the  [PropertyResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/PropertyResolver.html)  superinterface.<br><br>
> [Environment (Spring Framework 5.3.23 API)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/Environment.html)

<br>

---

## 2. Environment Interface의 getActiveProfiles()
<br>

`Environment` 인터페이스에서 제공하는 Method들은 아래와 같다.
``` java
String[] getActiveProfiles()
String[] getDefaultProfiles()
boolean acceptsProfiles(String... profiles)
boolean acceptsProfiles(Profiles profiles)
```
이 중에서 `getActiveProfiles()` 를 활용해서 우리는 어떤 `properties`가 지금 활성화되어 있는지 알아낼 수 있다. 활용 방법은 이따 따로 기술하도록 하고, 현재는 이 메소드가 어떤 역할들을 하는지 한 번 알아보고자 한다. <br><br>

>`String[] getActiveProfiles()`
**Return the set of profiles explicitly made active for this environment.** <br><br>Profiles are used for creating logical groupings of bean definitions to be registered conditionally, for example based on deployment environment. Profiles can be activated by setting "spring.profiles.active" as a system property or by calling ConfigurableEnvironment.setActiveProfiles(String...).<br><br>
If no profiles have explicitly been specified as active, then any default profiles will automatically be activated.

즉, `getActiveProfiles()` 메소드는 지금 running중인 application의 `활성화 되어 있는 properties`가 무엇인지 받아올 수 있는 메소드이다. **Return Type**이 `String[]`인 것을 통해서, 우리는 `getActiveProfiles()` 메소드가 현재 활성화 되어 있는 `properties`들을 전부 찾아와서 list로 찾아낸다는 것을 알 수 있다.

---

## 3. getActiveProfiles()를 활용해서, 현재 활성화 되어 있는 profiles 알아내기
<br>

그러면 이제 해당 메소드를 활용하는 방법을 알아볼 것인데, `properties` 파일의 이름을 `real1 / real2` 와 같이 구분한 다음 각각의 properties 파일에 서로 다른 `server port`를 사용한다고 작성하게 된다면, 우리는 `CI/CD` 단계에서 현재 활성화 되어 있는 `properties`를 보고 어떤 `port`가 지금 사용중이고, 그럼 어떤 `port`를 통해 새로운 배포를 해야지 기존의 서비스가 멈추지 않고 새로운 서비스를 배포할 수 있는지 알 수 있다.
<br><br>
`Blue/Green Deployment` 에 대해서는 다음에 자세히 다루도록 하고, 지금은 현재 메소드를 어떤 식으로 만들어야지, 그리고 어떻게 활용해야지 `Shell Script` 에서 현재 작동중인 서버의 `port`를 알아낼 수 있는지 한 번 코드를 짜 보도록 하겠다.

```kotlin
package com.entrip.controller

import org.springframework.core.env.Environment
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.util.*

@RestController
class ProfileController(private final val environment: Environment) {

    @GetMapping("api/v2/profile")
    public fun profile(): String {
        return Arrays.stream(environment.activeProfiles)
            .findFirst()
            .orElse("")
    }
}
```
해당 코드에서 첫 번째로는 `Environment`를 `DI` 받는다. 이후, `DI`받은 `Environment`의 `getActiveProfiles()` 메소드를 호출한다. (Kotlin에서는 Getter, Setter를 모든 클래스에 대해서 자동 지원하기 때문에, getActiveProfiles()와 같은 메소드 대신 activeProfiles와 같이 변수에 직접 접근하는 것 처럼 사용하면 그게 곧 Getter가 된다.) 위에서 확인한 레퍼런트에 따르면, `getActiveProfiles()` 메소드는 `String[]`를 리턴하기 때문에, 이를 `Array`로 받아온다. 이후, `Stream`을 사용해서 해당 메소드의 리턴 값 중 내가 원하는 값을 가져올 수 있다. 그럼 이를 어떻게 활용한다는 것일까?
<br><br>
아까 이야기한 것 처럼, `CI/CD` 단계에서는 `Shell Script`를 사용할 것이다. 그럼, 저기 만들어진 `REST API`를 `curl` 명령어를 사용해서 결과를 받아올 수 있을 것이다. 따라서, 현재 활성화 되어 있는 `properties` 파일을 `curl` 명령어를 통해서 알아낼 수 있을 것이고, 그럼 `properties` 파일은 아래와 같이 포트 별로 파일의 이름을 달리 해서 알아낼 수 있을 것이다.

```properties
#application-real1.properties
server.port=8080
```
```properties
#application-real2.properties
server.port=8081
```
위와 같이 두 개의 properties가 추가로 존재한다고 하자. 그리고 `CI/CD` 단계에서 `JAR` 파일을 통해 배포한다면, 아래와 같은 방법으로 우리는 배포하는 `port`를 결정할 수 있다.
```bash
nohup java -jar \
       -Dspring.config.location=classpath:/application.properties\
       -Dspring.profiles.active=real \
       $REPOSITORY/$JAR_NAME 2>&1 &
```
배포하는 스크립트의 제일 하단 부분을 가져온 것 인데, 여기서 우리가 주목해야 할 부분은 `-Dspring.profiles.active=real`이다. 배포 스크립트에서 우리가 현재 어떤 `profiles`를 활성화 할 것인지를 정할 수 있을텐데, 여기에 `real`에 해당하는 부분을 `real1`로 설정하느냐, `real2`로 설정하느냐에 따라서 현재 배포하는 server의 `port`를 달리할 수 있을 것이다. 또는 `Nginx` 웹서버를 활용해서, 최종적으로 `PortForwarding` 하는 포트를 변수화해서 사용하는 방법도 있을 것이다.

마지막으로, 그럼 `curl` 명령어는 어떻게 사용할 것인지에 대해서도 관련된 스크립트를 작성해보겠다.
```bash
#!/usr/bin/env bash

# profile.sh
# 미사용 중인 profile을 잡는다.

function find_idle_profile()
{
    # curl 결과로 연결할 서비스 결정
    RESPONSE_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://2ntrip.com:8080/api/v2/profile)

    if [ ${RESPONSE_CODE} -ge 400 ] # 400 보다 크면 (즉, 40x/50x 에러 모두 포함)
    then
        CURRENT_PROFILE=real2
    else
        CURRENT_PROFILE=$(curl -s https://2ntrip.com/api/v2/profile)
    fi

    # IDLE_PROFILE : nginx와 연결되지 않은 profile
    if [ ${CURRENT_PROFILE} == real1 ]
    then
      IDLE_PROFILE=real2
    else
      IDLE_PROFILE=real1
    fi

    # bash script는 값의 반환이 안된다.
    # echo로 결과 출력 후, 그 값을 잡아서 사용한다.
    echo "${IDLE_PROFILE}"

```
위 스크립트를 보면, curl 명령어를 사용해서 8081포트에 대해서 검증을 실시한다. 이때 에러가 발생한다면, 8080 포트는 사용중이지 않기 때문에, 현재 사용중인 포트는 8081이고, 현재 profile은 `real2` 임을 알 수 있는 것이다. 반대의 경우도 마찬가지로 확인할 수 있을 것이다.
