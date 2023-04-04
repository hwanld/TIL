# SpringBoot Test : TestMethodOrder
## TestMethodOrder란?
`JUnit5` 를 사용해서 통합 테스트를 진행하던 도중, 아래와 같은 총 4개의 로직을 통합 테스트를 진행하려고 하였다.
1. 회원 가입
2. 로그인
3. 아이디 중복 검사_성공 
4. 아이디 중복 검사_실패

여기서 2, 3, 4번에 해당하는 로직들은 전부 `DB 내부에 회원이 저장되어 있다는 가정`이 필요한 로직이었다.
따라서 기존에는 각 로직에 항상 테스트 하고자 하는 Users 를 저장하는 과정을 선행한 다음 본격적인 테스트를 진행하였다.

하지만 코드를 작성하던 도중 테스트 코드가 너무 길어진다는 생각이 들었고, `1번 테스트가 선행된 이후 2,3,4번 테스트를 진행하면 되지 않을까?`
라는 생각을 가지게 되었고, 이를 해결하기 위한 전략을 다음과 같이 세웠다.
>1. 통합 테스트 클래스의 `@Transactional` 어노테이션은 차후 여러 가지 상황을 위해서 유지한다.
>2. 1번 테스트, 회원가입 테스트가 완료되고 나면 롤백이 실행되지 않도록 `@Rollback (value=false)` 을 통해 롤백을 막는다.
>3. 1번 테스트가 선행된 이후 2,3,4번 테스트가 진행될 수 있도록 테스트 순서를 바꾼다.

`JUnit5` 에서는 `@TestMethodOrder` 라는 어노테이션을 활용해서 테스트의 순서를 매길 수 있게끔 도와준다. 이때 전달 인자로 몇 개의 클래스를 지정할 수 있는데,
이제 이에 대해서 하나씩 소개하고자 한다.

## TestMethodOrder's Parameter
`@TestMethodOrder` 어노테이션은 기본적으로 `MethodOrderer` 인터페이스의 몇 가지 클래스를 `parameter` 로 받는다. 기본적으로 사용하는 방법은 `@TestMethodOrder (MethodOrderer.XXX.class)` 와 같다.
### 1. MethodOrderer.MethodName
이름에서 유추할 수 있겠지만, 해당 클래스는 `Test Method의 이름` 순으로 테스트를 진행한다. `DisplayName` 과는 상관 없이 오직 `TestMethod` 의 이름만 사용한다.
### 2. MethodOrderer.DisplayName
이름에서 유추할 수 있겠지만, 해당 클래스는 `DisplayName` 순으로 테스트를 진행한다. `TestMethod의 이름` 과는 상관 없이 오직 `DisplayName` 만 사용한다.
### 3. MethodOrderer.Random
이 역시 이름에서 유추할 수 있드시 랜덤하게 테스트의 순서를 진행하는 것을 의미한다.
### 4. MethodOrderer.OrderAnnotation
`@Order` 어노테이션을 활용해서 테스트 메소드의 순서를 지정할 수 있는 방법을 의미한다. 우리는 여기서 이 방법을 활용해서 테스트 코드를 더욱 단순화 할 것이다.
각각의 Test 메소드에 `@Order(N)` 과 같이 어노테이션을 작성하면, N 번째로 실행될 수 있도록 설계할 수 있다.

## OrderAnnotation 을 사용해서 테스트 코드 단순화
위에서 설명한 로직을 그대로 코드로 적용한 사례는 다음과 같다.
```java
@SpringBootTest
@Transactional
@AutoConfigureMockMvc
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class UsersIntegrationTest {

    @Autowired
    private UsersService usersService;

    @Autowired
    private UsersRepository usersRepository;

    @Autowired
    private MockMvc mockMvc;

    @Test
    @Order(1)
    @DisplayName("[Integration] 회원 가입")
    @Rollback(value = false)
    void Integration_회원가입_테스트() throws Exception {
        //given
        UsersSaveRequestDto requestDto = new UsersSaveRequestDto(
                "test@gmail.com",
                "testNickname",
                "testPassword"
        );

        UsersResponseDto responseDto = new UsersResponseDto("test@gmail.com", "testNickname");

        //when
        mockMvc.perform(post("/api/users/save")
                .contentType(MediaType.APPLICATION_JSON)
                .content(new ObjectMapper().writeValueAsString(requestDto))
        )

        //then
                .andExpect(status().isOk())
                .andExpect(content().json(new ObjectMapper().writeValueAsString(responseDto)))
                .andDo(print());

    }

    ,,,중략
    
    @Test
    @DisplayName("[Integration] 아이디 중복 검사 실패 case")
    void Integration_아이디_중복검사_실패() throws Exception {
        //given
        String usersId = "test@gmail.com";

        //when
        try {
            mockMvc.perform(get("/api/users/checkUserExist/" + usersId))
        //then
                    .andExpect(status().isAccepted());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

}
```
`@Order(1)` 어노테이션의 사용으로 `Integration_회원가입_테스트` 가 제일 먼저 선행되도록 설정하였다. 또한 `@Rollback(value = false)` 어노테이션으로 해당 메소드가
실행된 이후 트랜잭션 (롤백) 이 일어나지 않도록 설정하였다.

이와 같이 설정하면, 기존의 `@Transactional` 어노테이션은 유지하면서 내가 필요한 부분만 롤백이 되지 않도록 설정할 수 있다. 또한 회원 가입 테스트를 선행함으로써 `@BeforeEach` 를 사용해서 
중복되어야 하는 회원의 정보를 굳이 DB에 넣지 않고도 테스트를 진행할 수 있다. 

결과론적으로 테스트 로직을 조금만 활용하고 테스트 순서의 변경만으로 코드를 더욱 클린하게 만들 수 있었다.