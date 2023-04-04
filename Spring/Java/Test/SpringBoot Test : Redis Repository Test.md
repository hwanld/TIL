# SpringBoot Test : Redis Repository Test
NoSQL 중에서 key-value 형태의 자료 저장 (dictionary 형태) 을 지원하는 `Redis` 의 경우, `RAM에 저장된다는 특성` 역시 가지고 있어 접근이 
빨라야 하는 상황에서 많이 쓰인다. 또한 `Expiration time` 을 설정할 수 있다는 또다른 장점이 있기 때문에, 보통 캐시 메모리의
구현에서 `Redis` 를 많이 사용하곤 한다.

현재 RBB 프로젝트에서 `RefreshToken` 의 저장을 위해서 `Redis` 를 도입하고자 하는데, 기존에 사용한 적 있던 `RedisTemplate`
가 아닌, `Spring Data Redis` 를 사용하였다. 기본적인 셋팅 후 `Repository Layer` 의 테스트를 진행하려고 기존의 `RDBMS`
와 비슷한 방식으로 작성하였으나 생각보다 많은 어려움을 겪어서, 간단하게나마 정리해 보고자 한다.

## Can't load Application Context 
```java
@DataJpaTest
@AutoConfiguredTestDatabase
public class RefreshTokenRepositoryTest {

    private static final Long RefreshTokenValidTime = 30 * 1000L;
    
    @Autowired
    RefreshTokenRepository refreshTokenRepository;

    @Test
    @DisplayName("[Repository] Refresh Token 생성 테스트")
    void Repository_RefreshToken_생성_테스트() {
        //given
        RefreshToken refreshToken = RefreshToken.builder()
                .usersId("test@gmail.com")
                .refreshToken("testRefreshToken")
                .expiredTime(RefreshTokenValidTime)
                .build();

        String expectedRefreshToken = "testRefreshToken";

        //when
        refreshTokenRepository.save(refreshToken);
        RefreshToken target = refreshTokenRepository.findById("test@gmail.com").orElseThrow(IllegalArgumentException::new);

        //then
        assertThat(target.getRefreshToken()).isEqualTo(expectedRefreshToken);
    }
}
```
다음과 같이 Test Code 를 작성하였으나, 계속해서 `Bean` 과 관련된 에러가 발생하였다. 정확히는 `ApplicationContext` 를 불러올 수 없다는 Error가 계속 발생하였다.

분명 `@SpringBootApplication` 을 사용하게 되면 `@EnableJpaRepository` 는 자동으로 작성되는 것으로 알고 있었다. 따라서 
`JpaRepository` 를 사용하는 `RefreshTokenRepository` 는 당연히 `Bean` 에 등록되어야 하는데, 계속해서 해당 Repository 를 찾지 못한다는 에러가 발생했디.

그렇다면 `@DataJpaTest`를 통해서 import 되는 Bean 들 중에서 Repository 가 없다는 말이 되는데, 조금 더 찾아보니 `@EnableRedisRepository` 라는 어노테이션이 있었다.
해당 어노테이션을 사용해서 `Redis` 와 관련된 추가적인 것들을 import 하였다. (RedisTemplate 등등)

그럼에도 불구하고 여전히 `ApplicationContext` 를 불러오지 못하는 애러가 발생했고, 결국 `RedisConfig.class`, 즉 Redis 를 사용하기 위해서
작성한 `RedisConfig.class` 를 import 함으로써 문제는 해결할 수 있었다.

## @AutoConfiguredTestDatabase
`@AutoConfiguredTestDatabase` 어노테이션은 `Repository Layer` 테스트를 위해서 사용될 Repository 를 자동으로 설정하도록
도와주는 어노테이션이다. 해당 어노테이션을 활용하고 별 다른 설정을 주지 않으면 `인메모리 데이터베이스(h2)` 를 활용해서 테스트용
DB 를 자동으로 구축해준다.

따라서 내가 테스트 하고 픈 데이터베이스가 있더라도, 별 다른 설정을 하지 않으면 별개의 DB 로 테스트가 진행될 것이다.

그래서 전달 인자로 H2 가 아닌 다른 인메모리 데이터베이스를 찾아보았지만, 역시나 Redis 가 존재하지는 않았다.

그래서 지금과 같은 상황에서는 `@AutoConfiguredTestDatabase` 가 아니라, 별개로 Repository 와 관련된 설정을 import 해서 
테스트 DB를 구축하도록 하게 되었다.

```java
@DataJpaTest
@EnableRedisRepositories
@Import(RedisConfig.class)
public class RefreshTokenRepositoryTest {

    private static final Long RefreshTokenValidTime = 30 * 1000L;
    @Autowired
    RefreshTokenRepository refreshTokenRepository;

    @Test
    @DisplayName("[Repository] Refresh Token 생성 테스트")
    void Repository_RefreshToken_생성_테스트() {
        //given
        RefreshToken refreshToken = RefreshToken.builder()
                .usersId("test@gmail.com")
                .refreshToken("testRefreshToken")
                .expiredTime(RefreshTokenValidTime)
                .build();

        String expectedRefreshToken = "testRefreshToken";

        //when
        refreshTokenRepository.save(refreshToken);
        RefreshToken target = refreshTokenRepository.findById("test@gmail.com").orElseThrow(IllegalArgumentException::new);

        //then
        assertThat(target.getRefreshToken()).isEqualTo(expectedRefreshToken);
    }
}
```
추가로, `@AutoConfiguredTestDatabase` 를 사용하더라도 비교적 자동화해서 테스트 DB 를 구축할 수 있다. 단, 해당 경우에는 
Test 패키지에 DB 와 관련된 설정 등을 또 만들어야 하기 때문에 (.properties 등등) 나는 위와 같이 Redis 를 테스트하였다.