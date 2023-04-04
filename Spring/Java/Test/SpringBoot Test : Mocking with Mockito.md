# SpringBoot Test : Mocking with Mockito

## 통합 테스트와 단위 테스트
SpringBoot 에서 Test 시, 크게 `Integration Test (통합 테스트)` 와 `Unit Test (단위 테스트)` 로 나누어서 테스트를 진행한다.

**통합 테스트** 는 직접 SpringBoot Tomcat 을 실행시킨 다음 진행하는 테스트로, Spring Bean 에 등록된 모든 Context 를 로드해오는 등의 특징이 있다. 서버를 띄워 테스트 하는 것과 같은 효과를 볼 수 있지만, Test의 시간에 오래 걸린다는 단점이 있다.
  * 보통은 `@SpringBootTest` 를 활용한다.
  * `@SpringBootTest` 어노테이션이 테스트 클래스에 포함되어 있으면, WAS를 직접 실행시키며 `main`의 Spring `Beans` 들을 전부 import 한다.

**유닛 테스트** 는 작은 단위로 쪼개어 진행하는 테스트로, 보통 `Service Layer`, `Controller Layer (API)`, `Repository Layer` 를 테스트한다.

* 통합 테스트와 다르게 Spring WAS를 실행시키지 않고 테스트를 진행하며 Spring Bean 에 등록된 모든 `@Configuration` 을 로드하진 않는다.
* 통합 테스트와 비교해서 매우 빠르게 테스트가 가능하다는 장점이 있다.
* 테스트하고픈 Layer 와 다른 Layer 와의 연관관계가 끊어져야 한다. 예를 들어서 Repository 를 테스트 하기 위해서 Service 의 로직이 간섭해서는 안된다.

## Mocking이란?
Unit Test 에서는 테스트하고픈 Layer 와 다른 Layer 와의 연관관계가 끊어져야 한다.

하지만 DI 등의 문제로 다른 Layer와의 연관관계를 끊기 어려운 경우가 있다.
  * 예를 들어서 Service 계층에서는 반드시 Repository 계층을 통해서 CRUD 등이 일어나야 하는데, Repository 를 사용하지 않고 Service 를 실행하는 것 자체가 모순이 될 수 있다.
  * 또한 실제 WAS 에서 활용되는 모든 객체를 생성해서 주입하는 것이 생각보다 많은 비용을 필요로 할 수도 있다.
> 이러한 상황에서 유용한 기술이 `Mocking` 인데, 가짜 객체를 만들고 가짜 객체가 하는 역할을 만들어 주는 역할을 한다.

Controller 계층을 테스트 하기 위해서는 다음 두 가지를 필요로 한다.
1. Http 통신을 진행하기 위한 객체
2. Controller 에서 제공하는 API를 실제로 수행하기 위한 객체 (Service 계층)

아까 언급한 바와 같이 위와 같은 역할을 하는 객체를 직접 생성하기에는 많은 비용이 든다. 따라서 **가짜 객체를 만들어서 클래스의 필드에 둔다.** 또한 가짜 객체가 DI를 필요로 하는 경우가 있다. 이런 경우에도 마찬가지로 **가짜 객체를 Bean 에 등록시켜서 가짜 객체를 DI** 할 수 있다.

Mocking 의 경우 다양한 라이브러리가 그 기능을 제공하겠지만, 일반적으로 `Mockito` 를 많이 사용하며 여기에서도 해당 라이브러리를 활용할 예정이다. 

아래 코드는 예시 코드이다.
```java
@WebMvcTest
@AutoConfigureMockMvc
public class UsersControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UsersService usersService;

    @MockBean
    private JwtTokenProvider jwtTokenProvider;

    @MockBean
    private AuthenticationManager authenticationManager;

    @Test
    @DisplayName("[Controller] 회원 가입")
    void Controller_회원가입_테스트() throws Exception {
        //given
        ,,생략
                
        when(usersService.save(any())).thenReturn(responseDto);
        
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

}
```
1. `MockMvc` : Http 통신을 실행하기 위한 가짜 객체로, `@WebMvcTest` 어노테이션을 통해서 Controller 계층을 테스트 하기 위해 필요한 `Configuration` 들만 주입받고, `@AutoConfigureMockMvc` 를 통해서 Http 통신을 실행하기 위한 `Configure` 를 진행한다.
2. `@MockBean` : 스프링 테스트에서 지원하는 어노테이션, Bean 에 등록되는 가짜 객체로, Spring Bean 에 가짜 객체를 등록할 수 있도록 하는 어노테이션이다.
3. `@Mock` : Mockito 라이브러리에서 지원하는 어노테이션, 가짜 객체를 만들고 이를 말해주는 어노테이션
4. `BDD (Behavior-Driven Development)` : Mock 객체가 하는 일을 지정해줄 수 있는 방법 중 하나이다. Mockito 와 BDD Mockito 둘 중 하나를 활용해서 작성할 수 있다.
> when(usersService.save(any())).thenReturn(responseDto); 
> 
> usersService mock 객체의 save 메소드가 아무 객체를 전달인자로 받더라도 그 응답은 무조건 responseDto 객체를 하도록 설정한다.
## Mock / MockBean / InjectMocks / Spy
* 아래 코드를 우선 참고하자.
```java
@ExtendWith(MockitoExtension.class)
public class UsersServiceTest {
    @InjectMocks
    private UsersService usersService;

    @Spy
    private BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();

    @Mock
    private UsersRepository usersRepository;


    @Test
    @DisplayName("[Service] 회원 가입")
    public void Service_회원가입_테스트 () {
        //given
        UsersSaveRequestDto requestDto = new UsersSaveRequestDto(
                "test@gmail.com",
                "testNickname",
                "testPassword"
        );
        Users users = requestDto.toEntity();
        UsersResponseDto responseDto = new UsersResponseDto(users);

        String expectedUsersId = "test@gmail.com";
        String expectedNickname = "testNickname";

        //mocking
        given(usersRepository.save(any()))
                .willReturn(users);

        //when
        UsersResponseDto targetResponseDto = usersService.save(requestDto);

        //then
        assertThat(targetResponseDto.nickname).isEqualTo(expectedNickname);
        assertThat(targetResponseDto.usersId).isEqualTo(expectedUsersId);
    }
}
```
생소한 어노테이션이 몇 개 더 나왔음을 알 수 있다. 큰 범위에서부터 하나씩 다시 정리해 보고자 한다.
1. `@Mock` : 가짜 객체를 만드는 것을 의미
2. `@MockBean` : 가짜 객체 (Mock) 를 Spring Bean (Application Context) 에 등록하는 것을 의미. 단, 기존의 Spring Bean 에 등록되어 있는 객체를 바꿔치기해서 Bean 에 등록
3. `@Spy` : `@Mock`와 마찬가지로 mocking 용 객체를 만드는 것이나, `@Mock` 과는 다르게 실제 인스턴스를 사용해서 mocking 할 수 있게 하는 어노테이션.
4. `@SpyBean` : 가짜 객체 (Spy) 를 Spring Bean (Application Context) 에 등록하는 것을 의미. 단, 기존의 Spring Bean 에 등록되어 있는 객체를 바꿔치기해서 Bean 에 등록
5. `@InjectMocks` : `@Mock` 어노테이션과 `@Spy` 어노테이션이 붙은 Mocking 용 객체들을 DI로 주입하도록 하게 해주는 어노테이션  

따라서 Stub (가짜 행위를 지정하는 것) 이 가능한 가짜 객체는 `@Mock` 또는 `@Spy` 를 통해서 만들 수 있는데, 둘의 차이는 실제 인스턴스를 이용해서 mocking 을 진행하는가 아니면 정말 껍데기만 존재하는 가짜 객체를 이용해서 mocking을 진행하는 가의 차이이다.

`@MockBean` 또는 `@SpyBean` 어노테이션을 활용하면 가짜 객체를 `Application Context` 에 직접 추가로 로드할 수 있다. 필요로 하는 `Configuration` 들만 로드하기 때문에 단위 테스트의 장점을 극대화해서 활용할 수 있다.

`@InjectMocks` 어노테이션을 활용하면 DI를 필요로 하는 객체에 가짜 객체를 주입할 수 있다. 위 코드에서 `UsersService` 객체는 `passwordEncoder`와 `usersRepository` 를 필요로 하기 때문에, 두 개의 가짜 객체를 주입한 것을 확인할 수 있다.

## 어떤 상황에서 @Spy, 어떤 상황에서 @Mock?
다시 위 코드를 참고해보자.
```java
@ExtendWith(MockitoExtension.class)
public class UsersServiceTest {
    @InjectMocks
    private UsersService usersService;

    @Spy
    private BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();

    @Mock
    private UsersRepository usersRepository;


    @Test
    @DisplayName("[Service] 회원 가입")
    public void Service_회원가입_테스트 () {
        //given
        UsersSaveRequestDto requestDto = new UsersSaveRequestDto(
                "test@gmail.com",
                "testNickname",
                "testPassword"
        );
        Users users = requestDto.toEntity();
        UsersResponseDto responseDto = new UsersResponseDto(users);

        String expectedUsersId = "test@gmail.com";
        String expectedNickname = "testNickname";

        //mocking
        given(usersRepository.save(any()))
                .willReturn(users);

        //when
        UsersResponseDto targetResponseDto = usersService.save(requestDto);

        //then
        assertThat(targetResponseDto.nickname).isEqualTo(expectedNickname);
        assertThat(targetResponseDto.usersId).isEqualTo(expectedUsersId);
    }
}
```
위 코드에서 `passwordEncoder`는 `@Spy`를 사용하였고, `usersRepository`는 `@Mock`를 사용한 것을 볼 수 있다.

`usersRepository` 의 경우, 오로지 회원 가입 테스트를 진행하기 위해선 save 메소드가 호출되었을 때 내가 원하는 값만 리턴하면 되기 때문에 given().willReturn 을 활용해서 stub 을 진행하였다.

반면 `passwordEncoder` 의 경우, usersService 의 save 메소드를 호출하였을 때 usersSaveRequestDto 의 password 필드의 값을 받아와서 암호화 하는 역할을 맡고 있다. 만약 이러한 역할을 하는 `passwordEncoder` 객체를 
`@Mock` 으로 만든다면, encode 하는 메소드 역시 stub 해야 할 것이다. 

하지만 차후 로그인 등의 테스트를 위해서, encode 와 decode 는 살제 인스턴스의 메소드를 활용하는 것이 더욱 이득일 것이라 판단했다. 따라서 이러한 경우에는 `@Spy` 어노테이션을 활용하였다. 
