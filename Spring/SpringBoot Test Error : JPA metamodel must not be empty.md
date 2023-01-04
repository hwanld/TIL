#  SpringBoot Test Error : JPA metamodel must not be empty<br>

## 1. 문제
* JsonTesting을 하던 도중, 아래 코드의 test를 진행하였더니 **JPA metamodel must not be empty!** 에러가 발생하였다.
```kotlin
class VehicleDetails(val make: String, val model: String) {}

@JsonTest
class MyJsonTest(@Autowired val json: JacksonTester<VehicleDetails>) {

    @Test
    fun serialize() {
        val details = VehicleDetails("Honda", "Civic")
        val expectedJson = "{\"make\":\"Honda\",\"model\":\"Civic\"}"
        assertThat(json.write(details)).isEqualToJson(expectedJson)

        // Assert against a `.json` file in the same package as the test
        // assertThat(json.write(details)).isEqualToJson("package.json")

        // Or use JSON path based assertions
        assertThat(json.write(details)).hasJsonPathStringValue("@.make")
        assertThat(json.write(details)).extractingJsonPathStringValue("@.make").isEqualTo("Honda")
    }

    @Test
    fun deserialize() {
        val content = "{\"make\":\"Ford\",\"model\":\"Focus\"}"
        assertThat(json.parse(content)).isEqualTo(VehicleDetails("Ford", "Focus"))
        assertThat(json.parseObject(content).make).isEqualTo("Ford")
    }

}
```
## 2. 원인
```kotlin
@EnableJpaAuditing
@EnableScheduling
@SpringBootApplication
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args)
}
```
* 프로젝트의 `@SpringBootApplication` 클래스를 보면, `@EnableJpaAuditing` 어노테이션을 확인할 수 있다. Spring Data JPA에서 Audit 기능을 사용할 수 있도록 하는 어노테이션으로, 보통 LocalDateTime의 사용을 위해서 사용하는 경우가 많은데,
* `@EnableJpaAuditing`이 `@SpringBootApplication` 클래스에 등록되어 있어서 모든 테스트들이 항상 JPA 관련 Bean들을 필요로 하고 있다.
* 하지만 `@JsonTest`는 통합 테스트가 아닌 슬라이스 테스트이기 때문에, 모든 Bean들을 주입받지 않기 때문에 에러가 발생한다.

## 3. 해결
### 1. @MockBean을 통한 bean 주입
```kotlin
@MockBean(JpaMetamodelMappingContext::class)
@JsonTest
class MyJsonTest(@Autowired val json: JacksonTester<VehicleDetails>) {
    //생략
}

```
### 2. @EnableJpaAuditing을 Configuration으로 분리
```kotlin
@Configuration
@EnableJpaAuditing
class JpaAuditingConfiguration {
}
```