**#  SpringBoot Test**
[SpringBoot Test](https://docs.spring.io/spring-boot/docs/2.7.7/reference/html/features.html#features.testing) SpringBoot Testing Reference를 참고.
<br>
## 1. build.gradle.kts (Gradle) Setting
* build.gradle.kts 파일에 다음을 추가한다.
```groovy
depencencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```
 `spring-boot-starter-test` 의 import를 통해서 JUnit, Spring Test, AssertJ 등 다양한 아래 테스트 모듈을 사용할 수 있으며, 각 특징은 아래와 같다.

*  [JUnit 5](https://junit.org/junit5/) : The de-facto standard for unit testing Java applications.
*  [Spring Test](https://docs.spring.io/spring-framework/docs/5.3.24/reference/html/testing.html#integration-testing)  & Spring Boot Test: Utilities and integration test support for Spring Boot applications.
*  [AssertJ](https://assertj.github.io/doc/) : A fluent assertion library.
*  [Hamcrest](https://github.com/hamcrest/JavaHamcrest) : A library of matcher objects (also known as constraints or predicates).
*  [Mockito](https://site.mockito.org/) : A Java mocking framework.
*  [JSONassert](https://github.com/skyscreamer/JSONassert) : An assertion library for JSON.
*  [JsonPath](https://github.com/jayway/JsonPath) : XPath for JSON.
  
<br>

---
## 2. Testing SpringBoot Applications
### 1. Testing SpringBoot Applications
`@SpringBootTest`는 서버를 시작하지 않고 테스트를 진행한다. `@SpringBootTest`의 `webEnvironment` attribute를 사용해서 test가 실행되는 환경을 직접 조성할 수 있다.

* MOCK(Default) : Loads a web ApplicationContext and provides a mock web environment. Embedded servers are not started when using this annotation. If a web environment is not available on your classpath, this mode transparently falls back to creating a regular non-web ApplicationContext. It can be used in conjunction with  [@AutoConfigureMockMvc or @AutoConfigureWebTestClient](https://docs.spring.io/spring-boot/docs/2.7.7/reference/html/features.html#features.testing.spring-boot-applications.with-mock-environment)  for mock-based testing of your web application.
* RANDOM_PORT: Loads a WebServerApplicationContext and provides a real web environment. Embedded servers are started and listen on a random port.
* DEFINED_PORT: Loads a WebServerApplicationContext and provides a real web environment. Embedded servers are started and listen on a defined port (from your application.properties) or on the default port of 8080.
* NONE: Loads an ApplicationContext by using SpringApplication but does not provide any web environment (mock or otherwise).

또한 실행하고자 하는 테스트가 `@Transactional` 인 경우, 테스트가 끝나게 되면 자동으로 롤백하는 것을 기본으로 한다. 하지만 `RANDOM_PORT` 나 `DEFINED_PORT` 를 사용하는 경우는 real servlet environment를 암묵적으로 제공하기 때문에, HTTP 클라이언트와 서버는 별도의 스레드에서 작동하고, 별도의 스레드에서 별도의 트랜젝션을 실행한다. 이 경우는 롤백이 일어나지 않는다.

`@SpringBootTest`를 `webEnvironment = WebEnvironment.RANDOM_PORT`로 설정하는 경우, 만약 application이 사용중인 port를 피해서 다른 port를 사용한다.

### 2. Detecting Web Application Type
* Spring MVC로만 개발된 경우, Spring MVC를 바탕으로 한 application context가 configured된다. Spring WebFlux로만 개발된 경우, WebFlux-based application context가 configured된다.
* 만약 둘 다를 사용하고 있는 경우, Spring MVC가 우선된다. 만약 WebFlux를 우선하고 싶다면, 아래와 같이 property를 설정한다.
```kotlin
import org.springframework.boot.test.context.SpringBootTest

@SpringBootTest(properties = ["spring.main.web-application-type=reactive"])
class MyWebFluxTests {
    // ...
}
```

### 3. Using Application Arguments
* `@SpringBootTest` 에다가 `args` attribute를 inject하므로써 애플리케이션에서 인수가 필요한 경우 주입할 수 있다.
```kotlin
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.ApplicationArguments
import org.springframework.boot.test.context.SpringBootTest

@SpringBootTest(args = ["--app.test=one"])
class MyApplicationArgumentTests {

    @Test
    fun applicationArgumentsPopulated(@Autowired args: ApplicationArguments) {
        assertThat(args.optionNames).containsOnly("app.test")
        assertThat(args.getOptionValues("app.test")).containsOnly("one")
    }

}
```

### 4. Testing With a Mock Environment
* `@SpringBootTest`의 경우 default로 서버를 실행하지 않고, mock environment를 만들어서 web endpoint를 test한다.
* Spring MVC를 테스트하기 위해서, `MockMvc` 또는 `WebTestClient`를 사용해서 테스트할 수 있다.
* 만약 완전한 `ApplicationContext`를 실행하지 않고 `web layer`만 테스트 하고 싶다면, `@WebMvcTest`를 사용해서 테스트 하는 것이 더 좋다.
* `MockMvc`를 사용해서 테스트하는 것은 서버 전체를 run해서 테스트 하는 것 보다 훨씬 빠르게 테스트 할 수 있지만, mocking이 Spring MVC 레이어에서 실행되기 때문에, 하위 수준의 servet container에 의존하는 행위들은 MockMvc에서 테스트가 불가능하다. 예를 들어서 예외 처리의 경우, MVC 레이어가 예상대로 예외를 발생시키고 처리하는지는 테스트 할 수 있지만 특정 사용자 지정 오류 페이지가 렌더링 되는지는 직접 테스트 할 수 없다. 이러한 부분은 완전히 실행중인 서버의 시작과 테스트로 해결할 수 있다.
```kotlin
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.test.web.reactive.server.expectBody
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders
import org.springframework.test.web.servlet.result.MockMvcResultMatchers

@SpringBootTest
@AutoConfigureMockMvc
class MyMockMvcTests {

//"{BaseUrl}/chat url로 get 요청을 보내고, status==200이고, result의 string이 index인지 검증
    @Test
    fun testWithMockMvc(@Autowired mvc: MockMvc) {
        mvc.perform(MockMvcRequestBuilders.get("/chat"))
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(MockMvcResultMatchers.content().string("index"))
    }


    // If Spring WebFlux is on the classpath, you can drive MVC tests with a WebTestClient

    @Test
    fun testWithWebTestClient(@Autowired webClient: WebTestClient) {
        webClient
            .get().uri("/")
            .exchange()
            .expectStatus().isOk
            .expectBody<String>().isEqualTo("Hello World")
    }

}
```

### 5. Testing With a Running Server
* 이미 동작하고 있는 server를 테스트 하고 싶다면, random ports를 사용하는 것이 바람직하다. `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)` 
* `@LocalServerPort` 어노테이션은 사용중인 실제 port를 주입 후 테스트 할 수 있도록 해준다. 이때, `WebTestClient`를 `@Autowire`해서 동작중인 서버의 테스트가 가능하다.
* WebTestClient는 WebClient를 테스트 하기 위해서 사용하는 것인데, 이는 WebFlux 기반의 테스트를 하고자 할때 사용한다. 만약 WebFlux를 사용중이라면, MVC와 Flux 모두 WebTestClient를 활용해서 테스트 할 수 있다. (반대로 MVC만 사용중이라면 WebTestClient의 사용은 불가능하다)
* MVC만 사용 할 때는, 아래 코드와 같이 `TestRestTemplate`를 사용해서 테스트를 진행하면 된다.
```kotlin

@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class MyRandomPortTestRestTemplateTests {

    @Test
    fun exampleTest(@Autowired restTemplate: TestRestTemplate) {
        val body = restTemplate.getForObject("/chat", String::class.java)
        assertThat(body).isEqualTo("index")
    }

}
```
 

#TIL
