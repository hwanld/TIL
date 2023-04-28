#  Extension Function을 활용한 클린 코드 만들기
## 1. Kotlin의 Extension Function

`Kotlin` 에서 제공하는 강력한 기능 중 하나는 `Extension Function`이다. 확장 함수 (Extension Function) 은 쉽게 이해하자면 말 그대로 확장 함수를 의미하는 것으로, 객체의 함수를 임의로 확장 정의하여 사용하는 것을 의미한다. 간단하게 예시를 들어서 한 번 보고자 한다.

```kotlin
fun Int.add(a : Int) = this + a

fun main(args: Array<String>) {
    println(3.add(5))
}

// 출력 결과
// 8
```
위의 예시에서, `Int.add` 라는 함수를 선언하였다. 이것이 Kotlin의 확장 함수인데, 내가 원하는 클래스에다가 새로운 메소드를 정의해서 사용할 수 있다. 별 다른 상속이나 인터페이스의 필요 없이 오직 이렇게 간단하게 사용할 수 있다. 마치 `Int`의 함수인 것 처럼 `add()`를 정의할 수 있게 된 것이다.

위의 예제에서 Int.add 는 Int 클래스에 정의된 add라는 확장함수로, 자기 자신에다가 a라는 전달 인자의 값을 더해서 리턴한다. 그리고 3이라는 Int형 객체 (숫자 3이라고 입력하게되면 자연스럽게 해당 숫자는 Int형 객체가 된다)의 add함수를 호출하고 전달 인자로 5라는 값을 전달하면, 해당 함수의 리턴 값은 3+5=8이 된다. 그래서 출력 값은 8이 나오게 되는 것이다.

또 다른 예시로, 공식 문서에 있는 예시를 하나 가져와보자.
```kotlin
fun MutableList<Int>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // 'this' corresponds to the list
    this[index1] = this[index2]
    this[index2] = tmp
}

val list = mutableListOf(1, 2, 3)
list.swap(0, 2) // 'this' inside 'swap()' will hold the value of 'list'
```
이렇게 `MutableList<Int>` 객체의 `swap` 메소드를 만들어 준 것 처럼 만들어 사용하는 것이 가능하다. 또한, Int 형태 뿐만 아니라 Generic을 사용해서도 확장 함수를 만드는 것을 소개하고 있다.

```kotlin
fun <T> MutableList<T>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // 'this' corresponds to the list
    this[index1] = this[index2]
    this[index2] = tmp
}
```
그리고 이렇게 Generic type을 활용하는 경우, 함수의 이름을 정의하기 전에 `<T>`, 즉 generic type parameter를 전언할 필요가 있다고 역시 안내하고 있다. 

Kotlin의 Extension Function과 관련된 자세한 내용은 맨 하단의 참조자료; Kotlin Reference를 참조하면 좋을 것 같다.

## 2. Extension Function을 활용하기

이제는 이 Extension Function을 어떻게 활용했는지 한 번 보여주고자 한다. 많은 상황에서 Extension Function을 활용할 수 있었겠지만, 본인은 _테스트 코드를 작성하는 동안에 있었던 일_ 을 기반으로 작성하였다.

현재 작성중인 테스트 코드의 Users 로직은 다음과 같다.
> * Users 회원 가입
> * Users 로그인
> * 로그인으로 오는 응답의 Token 값 추출
> * JWT Token 값을 활용해서 Security 적용된 메소드 시도

그리고 테스트 코드의 작성을 위해서는, 기본적으로 위의 로직이 전부 실행된 채로 lateinit으로 선언한 accessToken 변수의 값을 초기화해서 해당 값을 계속해서 넣는 식으로 사용해야 했다. 그리고 해당하는 부분은 beforeEach, afterEach를 활용하면 간단하게 할 수 있었다.

문제는 REST API의 Return으로 오는 Content에서 token 값을 추출해야 한다는 것인데, 지금 현재 return으로 오는 content는 다음과 같이 생겼다.
```json
{
    "httpStatus": 200,
    "message": "Success to Login!",
    "data": {
        "user_id": "wkazxf",
        "accessToken": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ3a2F6eGYiLCJ1c2VyUEsiOiJ3a2F6eGYiLCJpYXQiOjE2ODI2NTg2NjIsImV4cCI6MTY4MjY1OTI2Mn0.DKP0Dnc63qsHSPgmoJuGeuHQlDD-he2OboxUHT1LvfU",
        "nickname": "test",
        "refreshToken": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ3a2F6eGYiLCJ1c2VyUEsiOiJ3a2F6eGYiLCJpYXQiOjE2ODI2NTg2NjIsImV4cCI6MTY4Mjg3NDY2Mn0.Q5BEM2CCSxFcR3nbb8r7v1DGmzgrn3yPzsLzaHHdCN0"
    }
}
```
여기서 data필드를 1차적으로 뜯어내고, 그 다음 다시 해당 필드의 accessToken 값을 뜯어내야 하는데 GSON 라이브러리 등 오픈소스의 도움을 받지 않고는 해당 값을 뜯어내는 것이 조금 복잡했다. 뜯어내는 부분의 코드를 작성해서 뜯는 것은 성공했고, 우리 팀원이 해당 부분을 함수화시켜 리펙토링 하여 함수가 아래와 같이 나오게 되었다.
```kotlin
fun getContent(result: MvcResult, target: String): String {
    val objectMapper = ObjectMapper().registerModule(KotlinModule())
    // 1. MockMvcResult의 response를 String (JSON)으로 바꾼다
    val response = result.response.contentAsString

    // 2. String (JSON)을 RestAPIMessages로 변환한다
    val restAPIMessages = objectMapper.readValue<RestAPIMessages>(response, RestAPIMessages::class.java)

    // 3. RestAPIMessages의 data 필드를 String으로 다시 변환한다
    val str = restAPIMessages.data.toString()

    // 4. String으로 바꾼 data 필드를 전처리해서 target 값을 찾아낸다
    val pairs = str.substring(1, str.length - 1).split(", ")
    for (pair in pairs) {
        val keyValue = pair.split("=")
        if (keyValue[0] == target) {
            return keyValue[1]
        }
    }

    return ""
}
```
그런데 곰곰히 생각하니, 이러한 방식을 사용하더라도 IntegrationTest 모든 클래스에 해당 함수를 정의하는 것이 좋은 방식은 아니라고 생각했고, 따라서 해당 함수를 **IntegrationTest 클래스 공통으로 상속받는 BehaviorSpec() 클래스의 Extension Function**으로 만들면 어떨까 하였다.

```kotlin
fun BehaviorSpec.getContent(result: MvcResult, target: String): String {
    //생략
}
```
그래서 클래스 밖에다가 BehaviorSpec의 Extension Function이 될 수 있도록 위와 같이 코드를 변경하였고, 변경한 결과 다른 IntegrationTest 클래스에서도 해당 메소드를 빼서 사용할 수 있었다.
<br><br>

참조자료 : <br>
[Kotlin의 Extension은 어떻게 동작하는가 part1](https://medium.com/til-kotlin-ko/kotlin%EC%9D%98-extension%EC%9D%80-%EC%96%B4%EB%96%BB%EA%B2%8C-%EB%8F%99%EC%9E%91%ED%95%98%EB%8A%94%EA%B0%80-part-1-7badafa7524a) <br>
[Kotlin Extension Reference](https://kotlinlang.org/docs/extensions.html)
