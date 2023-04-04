# Spring REST Docs
## 0. Before
Spring REST Docs 는 테스트 코드 기반의 API 문서 명세서를 자동으로 작성해주는 프레임워크이다.<br>
공식 문서에는 2.x 버전과 3.x 버전 두 개의 레퍼런스가 존재하는데, 각 버전의 요구 사항은 아래와 같다.
* 2.x 버전 : Spring 5 이상 및 Java 8 이상
* 3.x 버전 : Spring 6 이상 (SpringBoot 3.0 이상) 및 Java 17 이상 <br>

Java 17을 사용하기 위해선 CentOS 8 이상의 사용이 필요하지만, 현재 VM의 경우 CentOS 7 버전을 사용하고 있고 SpringBoot 역시 2.7.x
버전을 사용하고 있기 때문에 `Spring RSET Docs 2.x` 버전을 기준으로 작성하였다.
## 1. Setting
가장 먼저 프로젝트의 빌드 설정을 해야한다. 아래와 같이 `build.gradle` 파일을 수정한다.
```gradle
plugins { 
	id "org.asciidoctor.jvm.convert" version "3.3.2"
}

configurations {
	asciidoctorExt 
}

dependencies {
	asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor:{project-version}' 
	testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc:{project-version}' 
}

ext { 
	snippetsDir = file('build/generated-snippets')
}

test { 
	outputs.dir snippetsDir
}

asciidoctor { 
	inputs.dir snippetsDir 
	configurations 'asciidoctorExt' 
	dependsOn test 
}
```
이어서 생성된 문서를 프로젝트의 jar 파일에 패키징 할 수 있도록 해서 Spring Boot 에서 `Static Content` 로 제공될 수 있도록 한다.
아래와 같이 빌드 설정을 통해서 (build.gradle), 다음과 같은 프로젝트의 빌드를 구성한다.<br>
1. JAR 빌드 전에 문서가 생성되도록 한다. 
2. 생성된 문서를 jar `static/docs` 디펙토리에 복사되도록 한다.
```gradle
bootJar {
	dependsOn asciidoctor 
	from ("${asciidoctor.outputDir}/html5") { 
		into 'static/docs'
	}
}
```
이후 사용중인 JUnit 버전에 따라서 약간의 설정을 추가로 해주어야 하는데, 필자가 사용하는 `JUnit5` 버전의 설정은 다음과 같다.<br>
첫 번째로, `RestDocumentationExtension` 를 테스트 클래스에 적용하는 것이다.
```java
@ExtendWith(RestDocumentationExtension.class)
public class JUnit5ExampleTests {
```
`RestDocumentationExtension`은 자동으로 output directory 까지 `configure` 해준다. 빌드 툴 별로 `Output directory`는 다음과 같다.
> Output Directory (Gradle) : **build/generated-snippets**<br>
> Output Directory (Maven) : **target/generated-snippets**<br> 

이어서 `@BeforeEach` 어노테이션을 사용해서 MockMvc, WebTestClient 또는 REST Assured 를 설정해 주어야 한다.
필자는 MockMvc 기반으로 작성하고 있음으로, 설정 메소드는 아래와 같다.
```java
    @BeforeEach
    void setUpForSpringRestDocs(WebApplicationContext webApplicationContext, RestDocumentationContextProvider restDocumentation) {
            this.mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext)
            .apply(documentationConfiguration(restDocumentation))
            .build();
    }
```
이제 제대로 설정이 되었는지 테스트 해보고자 한다. 다음과 같은 코드 기반의 테스트 코드를 작성하고, 빌드한다.
```java
mockMvc.perform(get("/")
        .contentType(MediaType.APPLICATION_JSON)
        .header("Token", accessToken))
        .andExpect(status().isOk())
        .andDo(document("index"))
        .andReturn();
```
빌드가 정상적으로 이루어졌다면, `<output-directory>/index/*.adoc` 파일이 총 6개가 생성된다. `<output-directory>` 는 gradle의 경우, `build/generated-snippets`이다.

## 2. Documenting API
### 1) Request and Response Payloads
기본적으로 Spring REST Docs 는 Request Body 와 Response Body 에 대한 Snippets 를 자동으로 생성한다. 각각의 이름은 `request-body.adoc`, `response-body.adoc` 이다. <br>

다음과 같은 Response Body 가 응답으로 온다고 가정해보자.
```json
{
  "AccessToken" : "accessTokenValue",
  "RefreshToken" : "refreshTokenValue"
}
```
이제 해당 응답이 온다는 것을 테스트 코드에서 작성해 볼 것이다.
```java
mockMvc.perform(post("/api/v2/users/save")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(new ObjectMapper().writeValueAsString(requestDto))
                )
                .andExpect(status().isOk())
                .andDo(document("Users", responseFields(
                        fieldWithPath("accessToken")
                            .description("User's access token value")
                            .type(JsonFieldType.STRING),
                        fieldWithPath("refreshToken")
                            .description("User's refresh token value")
                            .type(JsonFieldType.STRING))
                )))
                .andReturn();
```
먼저 `document`에는 해당 문서의 이름을 명시하고, `responseFields()`라고 문서화 하고 싶은 필드를 명시한 다음 `fieldWithPath` 에서는 해당 필드의 이름을 명시하면 되고, `description` 에는 해당 필드의 설명을 명시하면 된다. `type` 에는 해당 필드의 타입을 명시하면 되는데, 
JSON 에서 사용되면서 동시에 지원되는 타입은 아래와 같다.
> 1. array
> 2. boolean
> 3. object
> 4. number
> 5. null
> 6. string
> 7. varies <br>

이번에는 Response Body가 다음과 같이 온다고 가정해보자.
```json
{
	"contact": {
		"name": "Jane Doe",
		"email": "jane.doe@example.com"
	}
}
```
그렇다면 name과 email은 다음과 같이 표현할 수 있다.
```java
this.mockMvc.perform(get("/user/5")
        .accept(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk())
        .andDo(document("index", responseFields(
                fieldWithPath("contact.email")
                    .description("The user's email address"), 
                fieldWithPath("contact.name")
                    .description("The user's name")))); 
```
Json 에서 계층 구조를 이루고 있는 경우, 다음과 같이 `.` 을 활용해서 나타내면 된다. 위 JSON 을 보면 name, email 필드를 가지고 있는
contact 필드를 확인할 수 있는데, contact 필드의 경우 다음과 `subSectionWithPath` 를 활용해서 나타낼 수 있다. <br>
```java
this.mockMvc.perform(get("/user/5")
        .accept(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk())
        .andDo(document("index", responseFields(
                subsectionWithPath("contact")
                    .description("The user's contact details")))); 
```
`requestFields` 역시 위와 동일하게 작성하면 활용할 수 있다. 최종적으로 작성한 문서는 아래와 같다.
```java
        MvcResult saveResult = mockMvc.perform(post("/api/v2/users/save")
            .contentType(MediaType.APPLICATION_JSON)
            .content(new ObjectMapper().writeValueAsString(requestDto))
            )
            .andExpect(status().isOk())
            .andDo(print())
            .andDo(document("Users",
                requestFields(
                    fieldWithPath("usersId").description("usersID value to save.").type(JsonFieldType.STRING),
                    fieldWithPath("nickname").description("nickname value to save.").type(JsonFieldType.STRING),
                    fieldWithPath("password").description("password value to save.").type(JsonFieldType.STRING)
                ),
                responseFields(
                    fieldWithPath("accessToken").description("User's access token value").type(JsonFieldType.STRING),
                    fieldWithPath("refreshToken").description("User's refresh token value").type(JsonFieldType.STRING)
                )))
            .andReturn();
```

### 2) Request Parameters
Request 의 Parameter 의 경우 `requestParameters` 를 활용해서 다음과 같이 사용할 수 있다. `GET` 요청의 `query string` 을 다음과 같이 나타낼 수 있다.
```java
this.mockMvc.perform(get("/users?page=2&per_page=100"))
        .andExpect(status().isOk())
        .andDo(document("users", 
            requestParameters(
                parameterWithName("page")
                    .description("The page to retrieve"), 
                parameterWithName("per_page")
                    .description("Entries per page"))));
```
`POST` 요청의 Parameter의 경우 마찬가지로 다음과 같이 나타낼 수 있다.
```java
this.mockMvc.perform(post("/users").param("username", "Tester"))
        .andExpect(status().isCreated())
        .andDo(document("create-user",
            requestParameters(parameterWithName("username")
                .description("The user's username"))));
```
위 모든 결과들은 `request-parameters.adoc` 파일에 저장된다. `Request Parameters` 를 문서화 할 때, 만약 request 에서 문서화되지 않은 request 를 사용한 경우
테스트가 실패하게 된다. 반대로 문서화 되어 있는 parameter 가 사용되지 않은 경우에도 테스트는 실패하게 된다.
### 3) Path Parameters
Request 의 Path parameters 의 경우에도 `pathParameters` 를 활용해서 다음과 같이 사용할 수 있다.
```java
this.mockMvc.perform(get("/locations/{latitude}/{longitude}", 51.5072, 0.1275)) 
		.andExpect(status().isOk()).andDo(document("locations", pathParameters(
                    parameterWithName("latitude")
                        .description("The location's latitude"), 
                    parameterWithName("longitude")
                        .description("The location's longitude") 
		)));
```
마찬가지로 해당 결과들은 `path-parameters.adoc` 파일에 저장된다. 그리고 마찬가지로 문서화 되어 있는 path 를 사용하지 않았거나,
문서화 되어 있지 않은 path 를 사용한 경우 모두 테스트에 실패하게 된다.
### 4) Request Parts
Request 의 part 또는 multipart 의 경우에도 `requestParts` 를 사용해서 나타낼 수 있다. part 의 경우 파일 등을 전송해야 하는 상황일 때 일반적으로 많이 사용한다.
```java
this.mockMvc.perform(multipart("/upload").file("file", "example".getBytes())) 
		.andExpect(status().isOk()).andDo(document("upload",
                    requestParts(
                            partWithName("file").description("The file to upload")) 
		));
```
마찬가지로 해당 결과들은 `request-parts.adoc` 파일에 저장된다. 그리고 마찬가지로 문서화 되어 있는 part 를 사용하지 않았거나,
문서화 되어 있지 않은 part 를 사용한 경우 모두 테스트에 실패하게 된다.
### 5) Http Headers
Request 의 header 의 경우에도 `requestHeaders` 그리고 `responseHeaders` 로 나타낼 수 있다.
```java
this.mockMvc.perform(get("/people").header("Authorization", "Basic dXNlcjpzZWNyZXQ="))
        .andExpect(status().isOk()).andDo(document("headers", 
                requestHeaders(
                        headerWithName("Authorization").description("Basic auth credentials")), 
				responseHeaders(
						headerWithName("X-RateLimit-Limit").description("The total number of requests permitted per period"),
						headerWithName("X-RateLimit-Remaining").description("Remaining requests permitted in current period"),
						headerWithName("X-RateLimit-Reset").description("Time at which the rate limit period will reset")
                )
        ));
```
마찬가지로 해당 결과들은 `request-headers.adoc` 파일에 저장된다. 그리고 마찬가지로 문서화 되어 있는 header 를 사용하지 않았거나,
문서화 되어 있지 않은 header 를 사용한 경우 모두 테스트에 실패하게 된다.


이 외에도 `constraints`, `reusing API`, `default Snippets` 와 같이 다양한 기능을 제공하고 있다. 우선 위의 5가지 기능으로도 현재 작성하고자 하는 API 문서를 작성할 수 있기에, `Documenting` 은 이 정도 학습하고자 한다.
