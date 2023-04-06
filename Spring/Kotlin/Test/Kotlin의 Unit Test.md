# Kotlin의 Unit Test
> [2ntrip-Backend-Refactor](https://github.com/Entrip-Ajou/2ntrip-Backend-Refactor) 리펙토링 프로젝트를 진행하는 첫 번째 목표는 테스트 코드를 활용한 검증 및 TDD 기반 개발이었다. [RRB](https://github.com/ReRollBag/ReRollBag-BE) 프로젝트에서는 Java를 사용하여서 TDD까지 실습을 완료하였고, 이번에도 큰 무리 없이 할 수 있을 것이라고 생각했지만 **Kotlin 진영에서의 Test 코드 작성은 완전히 달랐기에**, 공부하고 그 내용을 남기는 바이다. 

<br>

## 1. Library for Testing in Kotlin
Java에서 가장 많이 사용하는 Test Library는 당연히 `JUnit4/5` 일 것이다. 나 역시 Java로 Test 코드를 작성할 때 JUnit5를 활용해서 Test 코드를 작성했고, 그 외에도 BDDMockito, AssertJ 등 다양한 라이브러리 때문에 매우 쉽게 테스트 코드를 작성할 수 있었다. <br>

하지만 조금 자료를 찾아본 결과, Kotlin에서 지원하는 DSL (Domain Specific Language) 스타일의 중괄호를 활용한 코드 스타일을 JUnit, AssertJ, Mockito 등에서는 활용할 수 없었기 때문에 이러한 점들을 보완하면서 테스트 코드를 작성할 수 있는 Kotlin 라이브러리인 **Kotest**를 사용하고자 한다. <br><br>

## 2. Kotest 
코틀린 진영에서 가장 많이 사용되는 테스트 프레임워크 Kotest는 코틀린 DSL을 활용해 테스트 코드를 작성할 수 있으며 아래와 같은 기능들을 포함하고 있다.
* 다양한 테스트 레이아웃 (String Spec, Word Spec, Behavior Spec, Describe Spec 등) 제공
* Kotlin DSL 스타일의 Assertion 기능 제공

의존성 설정의 경우 다음과 같이 간단하게 진행할 수 있다. (v4.4.3)
```groovy
	testImplementation("io.kotest:kotest-runner-junit5:4.4.3")
	testImplementation("io.kotest:kotest-assertions-core:4.4.3")
```
여기서 매우 중요하게 주의해야 할 점이 나오는데, kotest의 가장 최신 버전은 5.x이지만 필자는 결국 4.4.3 버전을 사용했다. 조금 뒤에 설명하겠지만, kotest를 spring에서 사용하기 위해서 Spring Extension을 사용해야 하는데 (@Transactional 등의 어노테이션 사용 등을 위해서) 이때 **Spring Extension과 Kotest의 버전이 다르면 테스트가 정상적으로 작동하지 않는다.**

그럼 지금부터 본격적으로 Kotest의 사용 방법을 보고자 한다. <br><br>

## 2-1. Kotest : Testing Styles
Kotest에서는 10개의 서로 다른 테스트 레이아웃을 제공한다. 이러한 레이아웃 중 몇몇은 서로 다른 테스트 프레임워크에서 영감을 받아 제작되었다고 한다. 10개의 test Style은 아래와 같다. <br>
> * Fun Spec (Inspired By ScalaTest)
> * Describe Spec (Inspired By Javascript frameworkd and RSpec)
> * Should Spec (A Kotest Original)
> * String Spec (A Kotest Original)
> * Behavior Spec (Inspired By BDD frameworks)
> * Free Spec (Inspired By ScalaTest)
> * Word Spec (Inspired By ScalaTest)
> * Feature Spec (Inspired By Cucumber)
> * Expect Spec (A Kotest Original)
> * Annotation Spec (Inspired By JUnit) 

> Tips : Some teams prefer to mandate usage of a single style, others mix and match. There is no right or wrong - do whatever feels right for your team.

Kotest를 사용하기 위해선 기본적으로 위 10개의 Spec 클래스 중 하나를 상속받는 class file를 우선 만들고 해당 클래서 안의 `init { } `블록 안에 테스트 케이스를 작성하면 된다. 많은 Spec이 있지만 그 중에서 몇가지만 짚어 보고자 한다.

* **Annotation Spec** 
  ```kotlin
  class AnnotationSpecExample : AnnotationSpec() {

    @BeforeEach
    fun beforeTest() {
        println("Before each test")
    }

    @Test
    fun test1() {
        1 shouldBe 1
    }

    @Test
    fun test2() {
        3 shouldBe 3
    }
  }
  ```
  Java와 JUnit의 조합과 가장 비슷한 방식이다. 큰 장점이 존재하지는 않지만, JUnit에서 Kotest로의 마이그레이션에 있어서는 가장 쉬운 방식이라고 할 수 있다.

* **Behavior Spec**
    ```kotlin
    class MyTests : BehaviorSpec({
        given("a broomstick") {
            and("a witch") {
                `when`("The witch sits on it") {
                    and("she laughs hysterically") {
                        then("She should be able to fly") {
                            // test code
                        }
                    }
                }
            }
        }
    })
    ```
    BDD Mockito와 비슷한 방식으로, `given/when/then` 형식의 테스트 코드를 지원한다. 다만 when의 경우 kotlin의 ketword이기 때문에 backticks로 감싸서 사용한다. 또한 하나의 given에 여러 개의 when을 달 수 있고, 하나의 when에 여러 개의 then을 달 수 있는 등 `and` 키워드를 사용한 다양한 nested 구조를 지원한다.
  
* **Describe Spec**
    ```kotlin
    class MyTests : DescribeSpec({
        describe("score") {
            it("start as zero") {
                // test here
            }
            describe("with a strike") {
                it("adds ten") {
                    // test here
                }
                it("carries strike to the next frame") {
                    // test here
                }
            }

            describe("for the opposite team") {
                it("Should negate one score") {
                    // test here
                }
            }
        }
    })
    ```
    DCI 형태 ( `describe/context/it` )를 지원하는 레이아웃이다. 이 역시 nested 구조를 지원하고, 테스트 구동 시 nested 구조로 테스트 설명을 확인할 수 있다.

이외에도 다양한 형태의 Spec이 있는데, 성능에 있어서는 아무런 차이가 없음으로 본인이 사용하고 싶은 것을 상황에 맞게 적절히 사용하면 된다고 공식 문서에 나와있다. 필자는 BDDMockito 형식을 따르면서 Kotlin DSL까지 유지하는 Behavior Spec를 사용했다. <br><br>

## 2-2. Kotest : Assertions
`kotest`는 Assertions을 Kotlin DSL을 지원하는 방식으로 지원하는데, Java의 AssertJ와 비교해서 훨씬 직관적으로 보여주고 있다. 대표적인 예시 코드 몇 개만 확인하고, `kotest-assertions-core` 모듈에서 지원하는 모든 메소드는 [공식 레퍼런스](https://kotest.io/docs/assertions/core-matchers.html) 에서 확인할 수 있다.

```kotlin
targetUsers.user_id shouldBe users.user_id
targetUsers.password shouldBe users.password
```

```kotlin
shouldThrow <Exception> {
    // code in here that expected to throw Exception
}
```
<br><br>

## 2-3. Kotest : Lifecycle hooks (Test Listeners)
JUnit를 사용하면서 다양한 테스트를 작성하다보면 자주 사용하는 어노테이션 중 하나는 `@BeforeEach` 그리고 `@AfterEach` 일 것이다. `setup` 또는 `tearDown` 이 테스트의 로직을 위해서 필요한 경우가 많은데, Kotest에서는 아래와 같이 지원한다.

> * beforeTest : 모든 테스트 케이스 실행 전 호출
> * afterTest : 모든 테스트 케이스 실행 후 호출
> * beforeSpec : 모든 beforeTest 함수 호출 전, Spec이 시작될 때 마다 호출
> * afterSpec : 모든 afterTest 함수 호출 후, Spec이 완료될 때 마다 호출
> * beforeSpecClass : Spec을 준비할 때 호출. Spec의 인스턴스화 횟수와 관계 없이 한 번만 호출
> * afterSpecClass : 모든 테스트가 완료되면 한 번만 호출
> * beforeProject : 테스트 엔진이 시작되는 즉시 호출
> * afterProject : 테스트 엔진이 완료되는 즉시 호출
> * 이하 생략,,

[공식 레퍼런스](https://kotest.io/docs/framework/lifecycle-hooks.html)에서도 다양한 hooks를 제공한다고 직접 서술되어 있다. 더 다양한 hooks 들은 직접 레퍼런스에서 찾아보도록 하자.

이러한 hooks들은 다음과 같이 사용하는데, beforeTest를 예시로 작성해 보겠다.
```kotlin
class hooksTest : BehaviorSpec() {

    override fun beforeTest(testCase: TestCase) {
        println("testCase : ${testCase}")
    }
```
Spec 클래스 내부에는 beforeTest와 같은 메소드들이 전부 implement 되어있고, 우리가 이러한 hooks 들을 사용하기 위해선 해당 메소드를 override 해서 사용하면 된다.


## 2-4. Kotest : Mockk (Mocking)

<br><br>

## 3. Kotest Extension for Spring
`@SpringBootTest` 와 같은 통합 테스트, 그리고 `@DataJpaTest` 와 같은 유닛 테스트에서 모두 Kotest의 Spec을 사용할 수 있다. Kotest에서는 Spring Framework의 DI를 위해서 Spring Extension을 제공한다. 아래와 같이 의존성을 추가할 수 있다. (v4.4.3)
```groovy
    implementation("io.kotest:kotest-extensions-spring:4.4.3")
```
여기서 **아주 조심해야 하는 부분**이 있는데, **`Kotest`의 버전과 `Spring Extension` 의 버전이 다르면 테스트가 제대로 작동하지 않는다.** 위에서 언급한 바와 같이, 무턱대고 최신 버전으로 설치했다간 아예 실행이 되지 않는 낭패를 볼 수 있다. 정리하면 아래와 같이 의존성을 추가해야 Kotest를 Spring 환경에서 사용할 수 있다. (v4.4.3)
```groovy
	testImplementation("io.kotest:kotest-runner-junit5:4.4.3")
	testImplementation("io.kotest:kotest-assertions-core:4.4.3")
	implementation("io.kotest:kotest-extensions-spring:4.4.3")
```
이어서, Spring extension을 사용하기 위해서는 모든 테스트 클래스, 또는 특정 테스트 클래스에 extension을 활성화 해주어야 한다. 다음과 같이 설정할 수 있다.
```kotlin
class MyTestSpec : FunSpec() {
   override fun extensions() = listOf(SpringExtension)
}
```
만약 모든 테스트 클래스에 extension을 활성화 하고 싶다면, 다음과 같이 `config` 클래스를 만들어 주면 된다. 일반적으로 test 패키지 내에 아래와 같이 설정하면 된다.
```kotlin
class ProjectConfig : AbstractProjectConfig() {
   override fun extensions() = listOf(SpringExtension)
}
```
<br>

## 4. Repository Unit Test with Kotest
Repository Unit Test를 Kotest를 활용해서 작성하는 법을 알아볼텐데, 위에서 서술한 바와 같이 Spring Extension을 활성화 하여야한다. 또한 본인은 `BehaviorSpec` 을 사용해서 작성하였는데, 상황에 맞게 조율하며 사용하면 될 것 같다.

우선 필자는 **단위 테스트라면 당연히 서버를 띄우지 않고, 다른 계층과 어떠한 유기적인 관계를 가지고 있어서는 안된다**고 생각한다. 그래서 당연히 `@SpringBootTest`를 사용하지 않았고, `@DataJpaTest`와 `@AutoConfigureTestDatabase` 어노테이션을 활용해서 테스트용 DB만을 활용했다.

위에서 본 예제들과는 다르게, Spec을 상속받는 테스트 클래스는 `class Test : Spec () {}` 와 같이 작성된다. 그리고 내가 테스트 하고 싶은 테스트 코드는 { } 코드블럭 안의 init { } 안에다가 작성하면 된다.

`@Autowired` 같은 경우에는 Kotlin의 특성 상 `lateinit` 생성자를 붙여 객체가 생성된 이후에 DI가 될 수 있도록 한다.

또한, Repository의 `tearDown` 의 경우는 위에서 언급한 바와 같이 `beforeTest` 메소드를 implement 하는 방식으로 구현한다.

필자는 given에다가 주어지는 객체 또는 상황을, when은 사용자 (또는 서버) 에서 행해지는 행위를, then은 그 결과를 나타내었는데 일단은 Save와 findById 두 개의 메소드를 테스트하는 코드를 작성하였다.

아래와 같이 코드를 짜게 되면, nested 구조는 다음과 같이 나온다. nested 구조 관련 별 다른 어노테이션을 사용하지 않아도 위와 같이 작성할 수 있다.
```
given("Users")
    when("Users를 저장하면")
        then("Users가 저장된다")
    when("Users를 저장한 뒤 findById를 하면")
        then("Users가 조회된다")
```


전체 코드는 아래와 같다.
```kotlin
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UsersRepositoryTest : BehaviorSpec() {
    override fun extensions() = listOf(SpringExtension)

    @Autowired
    lateinit var usersRepository: UsersRepository

    override fun beforeTest(testCase: TestCase) {
        usersRepository.deleteAll()
    }

    init {
        given("Users") {
            val users: Users = Users(
                user_id = "test@gmail.com",
                nickname = "test",
                gender = 1,
                photoUrl = "test.com",
                token = "token",
                m_password = "password"
            )
            `when`("Users를 저장하면") {
                usersRepository.save(users)
                val savedUsers = usersRepository.findAll().get(0)
                then("Users가 저장된다") {
                    savedUsers.user_id shouldBe users.user_id
                    savedUsers.nickname shouldBe users.nickname
                    savedUsers.gender shouldBe users.gender
                    savedUsers.photoUrl shouldBe users.photoUrl
                    savedUsers.token shouldBe users.token
                    savedUsers.m_password shouldBe users.m_password
                }
            }
            `when`("Users를 저장한 뒤 findById를 하면") {
                usersRepository.save(users)
                val targetUsers = usersRepository.findById(users.user_id!!)
                    .orElseThrow { IllegalArgumentException("Cannot Find Users with user_id") }
                then("Users가 조회된다") {
                    targetUsers.user_id shouldBe users.user_id
                    targetUsers.nickname shouldBe users.nickname
                    targetUsers.gender shouldBe users.gender
                    targetUsers.photoUrl shouldBe users.photoUrl
                    targetUsers.token shouldBe users.token
                    targetUsers.m_password shouldBe users.m_password
                }
            }
        }
    }
}
```
<br>

## 5. Service Unit Test with Kotest
Service Unit Test를 Kotest를 활용해서 작성하는 법을 알아볼텐데, 다양한 방법이 있겠지만 필자는 개발을 함에 있어서 시간을 최대한으로 줄이는 것이 중요하다고 생각한다. Service Test의 경우에는 `Mocking` 을 사용해서 진행하기 때문에 `SpringExtension` 이 필요하지 않다. `SpringExtension` 을 사용하게 되면 실행 시간이 늘어나기 때문에, Service Test 에서는 Spring Extension을 따로 사용하지 않는다. 이번에도 `BehaviorSpec` 을 사용해서 작성하였는데, 상황에 맞게 조율하며 사용하면 될 것 같다.

`Service Unit Test` 의 핵심은 `Mocking` 에 있다. 스프링에서는 결합도를 낮추기 위해서 `DI(Dependency Injection)` 을 사용하고 있는데, 테스트 하고자 하는 객체를 제외하고 필요로 하는 모든 객체들은 `가짜 객체 (Mocking)` 을 활용해서 DI 해주고 테스트 할 것이다. `Kotest` 에서는 `Mockito` 의 Mocking 이 아닌, `Mockk` 라이브러리를 활용해서 Mocking을 하는데, 기본적인 사용법은 기존의 `Mockito` 와 거의 유사하다. 우선 아래와 같이 의존성을 추가한다.

```gradle
	testImplementation("io.mockk:mockk:1.12.0")
	testImplementation("com.ninja-squad:springmockk:3.0.0")
```
이후 Mockking을 하기 위해선 Mock 형식의 객체를 만들어야 하는데, 객체는 다음과 같이 생성과 초기화 할 수 있다.
```kotlin
    val jwtTokenProvider = mockk<JwtTokenProvider>()
    val passwordEncoder = mockk<PasswordEncoder>()
    val plannersRepository = mockk<PlannersRepository>()
    val usersRepository = mockk<UsersRepository>()
```
그 다음 Mocking을 하기 위해선 Mocking을 필요로 하는 범위에서 다음과 같이 할 수 있다.
```kotlin
    every { usersRepository.findAll() } returns emptyList()
    every { usersRepository.save(any()) } returns users
```
Mocking 역시 every 뿐만 아니라 다양한 방법으로 Mocking 할 수 있는데, 해당 부분은 지금은 언급하지 않겠다. <br>
다음으로는 Mocking한 객체를 DI할 것인데, 여기에도 다양한 방법이 있을 수 있다. 필자는 이전에 말했던 바와 같이 빌드 타임을 최대한으로 줄이기 위해서, 최소한의 방법으로 DI를 할 것이다. 따라서, `테스트 하고자 하는 객체가 필요로 하는 객체를 Mocking 한 다음 생성자를 통해서 직접 DI` 하는 방식으로 진행할 것이다.
```kotlin
    val usersService = UsersService(usersRepository, plannersRepository, jwtTokenProvider, passwordEncoder)
```
이제 이러한 방식으로 Service Test 를 짜 볼 것인데, 코드는 아래와 같다.
```kotlin
class UsersServiceTest : BehaviorSpec() {

    val jwtTokenProvider = mockk<JwtTokenProvider>()
    val passwordEncoder = mockk<PasswordEncoder>()
    val plannersRepository = mockk<PlannersRepository>()
    val usersRepository = mockk<UsersRepository>()
    val usersService = UsersService(usersRepository, plannersRepository, jwtTokenProvider, passwordEncoder)

    val users = Users(
        user_id = "test@gmail.com",
        nickname = "test",
        gender = 1,
        photoUrl = "test.com",
        token = "token",
        m_password = "testPassword"
    )

    init {
        given("UsersSaveRequestDto를 주고") {
            val usersSaveRequestDto = UsersSaveRequestDto(
                user_id = "test@gmail.com",
                nickname = "test",
                gender = 1,
                photoUrl = "test.com",
                password = "testPassword"
            )

            every { usersRepository.findAll() } returns emptyList()
            every { usersRepository.save(any()) } returns users

            `when`("save하면") {
                val result = usersService.save(usersSaveRequestDto)
                then("save된 user_id가 리턴된다") {
                    result shouldBe "test@gmail.com"
                }
            }
        }
        // 중략
    }
}
```
추가로, Mocking한 객체가 하나의 테스트가 종료된 이후 다음 테스트에서 재사용 되는 경우, 기존에 Mocking한 내용이 남아 있어 문제가 될 수 있다. Kotest Reference에서도 이러한 상황에 대해 설명하고 있고, Kotest Reference에서 소개하는 두 가지 옵션을 소개하고자 한다. <br><br>
**Option 1 - setup mocks before tests**
```kotlin
class MyTest : FunSpec({

    lateinit var repository: MyRepository
    lateinit var target: MyService

    beforeTest {
        repository = mockk()
        target = MyService(repository)
    }

    test("Saves to repository") {
        // ...
    }

    test("Saves to repository as well") {
        // ...
    }

})
```
<br>

**Option 2 - reset mocks after tests**
```kotlin
class MyTest : FunSpec({

    val repository = mockk<MyRepository>()
    val target = MyService(repository)

    afterTest {
        clearMocks(repository)
    }

    test("Saves to repository") {
        // ...
    }

    test("Saves to repository as well") {
        // ...
    }

})


```
<br>


### REFERENCE
[스프링에서 코틀린 스타일 테스트 코드 작성하기](https://techblog.woowahan.com/5825/) <Br>
[Kotest Reference](https://kotest.io/docs/framework/framework.html)

