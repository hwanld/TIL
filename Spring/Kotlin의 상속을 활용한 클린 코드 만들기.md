#  Kotlin의 상속을 활용한 클린 코드 만들기
## 1. Kotlin의 상속
<br>

`Kotlin` 에서의 클래스는 우리가 `final` 키워드를 붙이지 않아도, `default` 로 `final Class` 이다. 즉, 모든 클래스는 기본적으로 `부모 클래스가 될 수 없다` 는 특징을 가지고 있다. 따라서 Kotlin에서 상속을 하기 위해선, `open` 키워드를 통해 해당 클래스를 `open class` 로 바꿔주어야 한다. 

```kotlin
open class UsersLoginResReturnDto(
    val user_id: String,
    val accessToken: String,
    val nickname: String,
    val refreshToken: String,
) {
}

private class DummyUsersLoginResReturnDto(val detailedMessage: String) : UsersLoginResReturnDto("", "", "", "") {
}
```

다음과 같이 상속받고자 하는 class를 `open class` 로 만들어준 뒤, 상속을 받는 class에서는 부모 객체의 생성자를 전부 만족시키면서 상속을 받아야 한다. 위 내용이 kotlin에서의 기본적인 상속 내용이다. 물론 이 외에도 `abstract` 와 같은 다양한 방법이 있으나, 오늘은 해당 부분에 대해서 다뤄 보고자 한다.

<br>

---

## 2. 상속을 활용한 코드 재사용 줄이기
<br>

`MVC 패턴` 을 따르는 스프링 프로젝트의 경우, 대부분 `Service` 계층과 `Controller` 계층이 존재할 것이다. 그리고 `Service` 계층에서는 각자 메소드의 로직에 따라 정보를 `CRUD` 하는 것이 기본일 것이고, `Controller` 계층에서는 `FE` 로부터 필요 자료들을 전달받은 뒤 해당 자료들을 바탕으로 `FE` 로 다시 `REST API` 등의 방법으로 응답을 보내는 것이 기본일 것이다.

```kotlin
@RestController
class ProjectsController(
    private val projectsService: ProjectsService
){

    private fun sendResponseHttpByJson(message: String, data: Any?): ResponseEntity<RestAPIMessages> {
        val restAPIMessages = RestAPIMessages(
            httpStatus = HttpStatus.OK,
            message = message,
            data = data,
            errorCode = 0
        )
        val headers: HttpHeaders = HttpHeaders()
        headers.contentType = MediaType("application", "json", Charset.forName("UTF-8"))
        return ResponseEntity<RestAPIMessages>(restAPIMessages, headers, HttpStatus.OK)
    }

    @PostMapping("api/projects/save")
    fun save(@RequestBody projectsSaveRequestDto: ProjectsSaveRequestDto): ResponseEntity<RestAPIMessages> =
        sendResponseHttpByJson("Project is saved well", projectsService.save(projectsSaveRequestDto))

    ,,,생략

}
```
`Controller` 계층의 예시를 하나 가져온 것이다. 현재는 생략되어 `save` 메소드 밖에 존재하지 않지만, 이외에도 다양한 메소드들이 존재할 것이다. 현재 보고 있는 예시는 `REST API`의 방법으로 `ResponseEntity<>` 를 리턴하는 `Controller` 계층이기 때문에, 모든 메소드는 `ResponseEntity<>` 를 리턴할 것이다. 따라서 지난 번 리펙토링 당시에 모든 메소드에서 공통으로 쓰이고 있던 부분을 함수로 따로 분리해서, 위에 보는 것 처럼 `sendResponseHttpByJson` 이라는 메소드로 따로 분리해서 사용하고 있었다. <br>
<br>

여기까지만 해도 충분히 만족할 수 있다고 생각할 수 있겠지만, 조금 더 코드를 간략하게 만들고 싶었다. 위와 같은 방법은 모든 `Controller` 클래스에 대해서 `sendResponseHttpByJson` 이라는 메소드를 만들어야 했다. 이도 결국 코드의 반복이었고, 줄일 수 없을까 하는 방법을 생각하다가 프로젝트 팀원이 "지금 코드 제작할 때 상속은 따로 사용안해?" 라고 한 말이 떠올라서, 상속을 한 번 활용해 보고자 했다.<br>
<br>

```kotlin
@RestController
open class BaseController {
    fun sendResponseHttpByJson(message: String, data: Any?): ResponseEntity<RestAPIMessages> {
        val restAPIMessages = RestAPIMessages(
            httpStatus = HttpStatus.OK,
            message = message,
            data = data,
            errorCode = 0
        )
        val headers: HttpHeaders = HttpHeaders()
        headers.contentType = MediaType("application", "json", Charset.forName("UTF-8"))
        return ResponseEntity<RestAPIMessages>(restAPIMessages, headers, HttpStatus.OK)
    }
}
```
```kotlin
@RestController
class ProjectsController(
    private val projectsService: ProjectsService
) : BaseController() {

    @PostMapping("api/projects/save")
    fun save(@RequestBody projectsSaveRequestDto: ProjectsSaveRequestDto): ResponseEntity<RestAPIMessages> =
        sendResponseHttpByJson("Project is saved well", projectsService.save(projectsSaveRequestDto))
    
    ,,,생략
}
```

위 코드를 보면 `BaseController` 라는 `open class` 를 하나 만들고, 다른 `Controller` 들에서 해당 객체를 상속받고 있다. `BaseController` 에서는 REST API 통신에 사용되는 `sendResponseHttpByJson` 이라는 메소드를 하나 가지고 있고, `BaseController`를 상속받는 모든 `Controller` 는 해당 메소드를 사용할 수 있다. 따라서 이제는 `Controller`의 개수가 몇 개이든, **전부 BaseController를 상속받아서 더 이상 메소드를 새로 만들 필요가 없어졌다.** 이와 같은 방법을 활용해서 재사용 부분을 조금 더 줄일 수 있었다. 

<br>

---

## 3. 상속을 활용한 DTO 개수 줄이기
<br>

이번에는 API를 고치는 상황에서 발생한 일이다. 로그인 시 user_id 또는 password가 잘못된 경우에, 리턴하는 에러 메세지의 Data 필드에 들어가는 값을 `String에서 {기존 DTO}+String` 조합으로 변경해 달라고 요청이 왔다. 기존 DTO만 리턴한다면 DTO 클래스를 하나 생성해서 넘겨주면 되지만, 문제는 `새로운 String 변수를 하나 더 추가해 달라고 요청`한 것이다. 새로운 DTO를 만들어서 넘겨주어야겠다고 생각한 이후 DTO 디렉토리에 새로운 DTO를 만들다가, 문득 묘수가 떠올랐다.

<br>

> "현재 메소드에서만 사용한다면 DTO를 상속받은 private class를 만들어서 사용하면 안될까?"

<br>

따라서 바로 진행해 보고자 하였다. 우선, 정상적인 경우에 리턴하는 dto 클래스를 `open class` 로 바꾸었다.

```kotlin
open class UsersLoginResReturnDto(
    val user_id: String,
    val accessToken: String,
    val nickname: String,
    val refreshToken: String = "DummyTokenValue"
) {
}
```

이후, 어차피 `하나의 메소드에서 발생하는 Exception` 에 대해서만 사용하면 되는 클래스이기 때문에, `해당 Exception이 발생하는 메소드 아래에 클래스를 private하게 정의`하기로 하였다.

```kotlin
public fun login(usersLoginRequestDto: UsersLoginRequestDto): UsersLoginResReturnDto {
    if (!isExistsUser(usersLoginRequestDto.user_id)) throw NotAcceptedException(DummyUsersLoginResReturnDto("Email is not valid"))
    val users = findUsers(usersLoginRequestDto.user_id)
    if (!passwordEncoder.matches(
            usersLoginRequestDto.password,
            users.password
        )
    ) throw NotAcceptedException(DummyUsersLoginResReturnDto("Password is not valid"))
    val accessToken: String = jwtTokenProvider.createAccessToken(usersLoginRequestDto.user_id)
    val refreshToken: String = jwtTokenProvider.createRefreshToken(usersLoginRequestDto.user_id)
    return UsersLoginResReturnDto(users.user_id!!, accessToken, users.nickname, refreshToken)
}

private class DummyUsersLoginResReturnDto(val detailedMessage: String) : UsersLoginResReturnDto("", "", "", "") {
}
```

다음과 같은 방법으로 `private class` 를 생성한 후, 필요한 자료형을 가지고 있는 `DTO를 상속했다.` 클래스 내부에서 클래스를 정의할 수 있기 때문에, 다음과 같은 방법으로 클래스를 선언한 뒤 해당 클래스를 만들도록 하였다. 추가적으로 필요한 변수들에 대해선 생성자를 통해서 전달하였다.

어떤 방면에서 보면 해당 코드는 가독성이 떨어질 수도 있는 코드이다. 다만, 해당 방법을 통해서 `DTO 디렉토리의 클래스 한 개를 줄일 수` 있다는 점은 확실하다. 한 줄, 두 줄 짜리 클래스를 위해서 새로운 파일을 만드는 것 보단, 다음과 같은 방법이 조금 더 편리하게 작용할 수 있지 않을까 생각한다.
