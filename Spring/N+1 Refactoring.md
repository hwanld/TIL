# N+1 Refactoring
## 1. N+1 문제란?

`N+1` 문제가 무엇인지에 대해서 간단하게 짚고 넘어가고자 한다. N+1 문제란 1개의 쿼리를 요청했으나 N개의 추가 쿼리가 발생하는 문제를 말한다. 아래 두 개의 예시와 함께 살펴보자.

```kotlin
@Entity
class Users(
    @Id @Column(name = "USER_ID")
    val user_id: String? = null,

    @Column
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinColumn(name = "PLANNERS_USERS")
    var planners: MutableSet<Planners> = TreeSet()
){
}
```
```kotlin
@Entity
class Planners(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "PLANNER_ID")
    var planner_id: Long? = null,

    @Column
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinColumn(name = "USERS_PLANNERS")
    var users: MutableSet<Users> = TreeSet()
){
}
```
위와 같이 두 개의 엔티티가 있다고 가정하자. Users-Planners는 지금 `@ManyToMany`, 즉 다대다 조인이 되어 있는 상태이다. 만약 이 상태에서 Users가 1개의 Planners와 조인이 되어 있다고 가정할 때, 유저 조회 쿼리를 날린다면 어떻게 될까? 우리는 분명 `Users만 조회하고자 했지만, Planners를 조회하는 SQL까지` 날아가게 된다. 이러한 문제를 보고 **N+1 쿼리 문제**라고 한다.

문제가 생기는 원리는 다음과 같다. 우선 Users와 Planners는 조인 관계에 있는데, 현지 fetch 전략이 `EAGER`로 설정되어 있다. 이렇게 설정하게 되면 Users를 조회할 때 Users와 조인되어 있는 모든 객체들을 한 번에 불러오게 되는데, 그러면 우리가 필요하지 않는 순간에도 모든 사용자를 한 번에 불러오게 된다. 

그래서 이와 반대되는 fetch 전략이 `LAZY` 전략인데, 해당 전략을 사용하면 planners를 조회하지 않을 때는 Users만 조회하게 된다. 하지만 만약 `LAZY` 전략을 사용하는 도중, users 내부에 있는 planners set을 참조하게 된다면? `EAGER` 때와 마찬가지로 planners를 모두 조회하게 된다.

결국, fetch 전략을 LAZY로 설정하던, EAGER로 설정하던 상관 없이 `N+1문제는 발생`한다. 다만 그 발생하는 시점이 다를 뿐이다. 이를 해결하기 위한 방법은 여러 가지가 있는데, `Fetch Join` 이나 `@EntityGraph`를 제일 많이 사용한다. 

`Fetch Join`은 말 그대로 JPQL의 `Fetch Join` 을 사용하는 것인데, 일반적으로 우리가 Spring Data JPA를 사용한다면 SQL Function을 사용하게 된다. 이때 `@Query` 어노테이션을 활용해 직접 JPQL를 지정할 수 있는데, 이때 해당 메소드의 `JPQL에 join Fetch 키워드를 사용`하는 것이다.
```kotlin
interface UsersRepository : JpaRepository<Users, String> {
    @Query("select u from Users u join fetch u.planners)
    fun findAllJoinFetch() : List<Users>
}
```
이런 식으로 사용하게 되면 만약 우리가 Users를 조회할 때, planners가 N개가 있다고 하면 planners N개를 조회하기 위해서 N개의 쿼리가 아닌, **한 개의 쿼리로 Users와 Join되어 있는 모든 Planners를 조회**할 수 있을 것이다. **연관관계가 있을 경우에도 하나의 쿼리문으로 표현할 수 있기 때문에 매우 유리하다.**

하지만 Fetch Join을 사용하게 되면 단점이 존재하는데, `Fetch Type`을 사용할 수 없다는 점이다. Fetch 전략이 Lazy 전략이라도, 쿼리를 실행할 때 이미 join fetch을 사용해서 fetch를 하기 때문에 원하는 것 처럼 전략을 사용하지 못할 수 있다.

또 다른 단점은, `페이징 쿼리`를 사용할 수 없다는 점이다. 하나의 엔티티 조회를 요청하는 순간 연관되어 있는 모든 조인된 데이터를 하나의 SQL로 다 가져오기 때문에, 페이징 단위로 데이터를 가져오는 것이 불가능하다.

`Entity Graph`는 쿼리 수행 시 바로 가져 올 필드명을 지정하면 해당 필드만 Lazy가 아닌 Eager로 가져오는 전략이다. 이 역시 JPQL을 사용해서 한 번에 가져오는, Fetch Join과 비슷한 전략이다.

그 외에도 여러 전략이 존재하지만 정답은 없는 듯 하다. 따라서, 지금 현재 리펙토링 중인 프로젝트의 상황에 가장 알맞게 사용할 수 있도록 한 번 개선을 해보고자 한다.

<br>

## 2. 레거시 분석
우선 엔티티에서의 문제점을 한 번 보고자 한다. <br>

* **엔티티의 연관 관계가 너무 무분별하게 `fetch = FetchType.EAGER`를 사용하고 있다.** 
   
```kotlin
@Entity
class Users(
    @Id @Column(name = "USER_ID")
    val user_id: String? = null,

    @Column
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinColumn(name = "PLANNERS_USERS")
    var planners: MutableSet<Planners> = TreeSet()
){
}
```
지금은 생략했지만, Users Entity는 현재 총 9개의 서로 다른 엔티티와 조인되어 있고, 모든 엔티티는 EAGER 전략을 사용하고 있다. 이렇게 되면 정말 수도 없이 많은 N+1 쿼리를 만들어 낼 것이다.

당시에 LAZY를 사용했더니 에러가 나서 EAGER 타입을 사용했는데, LAZY를 사용해서 에러가 생기는 이유는 영속성 컨텍스트에 대해서 고려하지 않고 트랜잭션 범위를 고려하지 않은 채 무분별하게 엔티티를 조회 후 수정을 시도해서 그런 것 같다. 레거시는 어쩔 수 없지만, 지금부터 하나씩 이를 고쳐 나가야 할 것 같다.

* **`EAGER` 전략을 사용하지 않아도 되는 부분에서도 join 되어있는 엔티티를 조회하고 있다.**
```kotlin
    fun login(usersLoginRequestDto: UsersLoginRequestDto): UsersLoginResReturnDto {
        // Match email first. If failed, throw NotAcceptedException
        if (!isExistUserId(usersLoginRequestDto.user_id)) {
            logger.warn("Fail to Login because email is not valid with email value : '{}'", usersLoginRequestDto.user_id)
            throw NotAcceptedException(DummyUsersLoginResReturnDto("Email is not valid"))
        }
        val users = usersRepository.findById(usersLoginRequestDto.user_id)
        // Match password second with passwordEncoder. If failed, throw NotAcceptedException
        if (!passwordEncoder.matches(
                usersLoginRequestDto.password,
                users.password
            )
        ) {
            logger.warn("Fail to Login because password is not valid")
            throw NotAcceptedException(DummyUsersLoginResReturnDto("Password is not valid"))
        }
        // Create accessToken and refreshToken via jwtTokenProvider
        val accessToken: String = jwtTokenProvider.createAccessToken(usersLoginRequestDto.user_id)
        val refreshToken: String = jwtTokenProvider.createRefreshToken(usersLoginRequestDto.user_id)
        logger.info("Success to Login. Return UsersLoginResReturnDto with accessToken and refreshToken")
        return UsersLoginResReturnDto(users.user_id!!, accessToken, users.nickname, refreshToken)
    }
```
코드가 조금 길지만, usersService에서 로그인을 처리하는 메소드이다. 일부러 비즈니스 로직이 비교적 복잡한 메소드를 가져와 봤는데, 현재 코드의 `usersRepository.findById` 부분은 사용자의 패스워드 하나를 조회하기 위해서 사용되고 있다. (DB에 저장된 암호화된 패스워드와 로그인 DTO의 패스워드 값을 암호화 한 후 두 개를 비교하는 부분) 그런데 패스워드 하나를 조회하기 위해서 usersRepository.findById를 EAGER 전략으로 가져오게 되면 굳이 조인되어 있는 엔티티들은 조회할 필요가 없음에도 불구하고 조회하기 때문에 성능 측면에 있어서 아주 해가 되는 부분이다. 

여기까지 들으면, "그럼 EAGER를 LAZY로 수정하면 되는게 아닐까?" 하는 생각이 들지만, 다음 상황을 보자.

* **`LAZY` 전략을 사용하게 되면 프록시 객체를 조회하는 순간 N+1 문제가 생긴다.**
```kotlin
    @Transactional
    fun addPlanners(planner_id: Long, user_id: String): Long? {
        val planners: Planners = plannersRepostiory.findById(planner_id)
        val users: Users = usersRepository.findById(user_id)
        users.addPlanners(planners)
        planners.addUsers(users)
        logger.info("Planner with planner_id : '{}' is added with User with user_id : '{}'", planner_id, user_id)
        return planners.planner_id
    }
```
Users 엔티티에 Planners 엔티티를 양방향으로 서로 조인해주는 메소드이다. 해당 메소드에서는 users와 조인되어있는 planners 자료구조 (list가 될 수도 있고, set이 될 수도 있는데 본인은 중복을 방지하기 위해서 set을 사용했다) 를 불러와서 해당 자료구조에 조인하고자 하는 planners를 add함으로써 조인 관계를 완성시킨다. (양방향 매핑이기에 그 역 마찬가지로 수행한다.) 그런데 이때, 만약 Users가 LAZY 전략을 사용하고 있었다면 Users.Planners를 조회하는 순간 프록시 객체에서 Planners를 조회하고자 할 것이고, 결국 N+1 쿼리가 발생하게 될 것이다.

요약하면, 현재 문제는 다음과 같다.
> * FetchType.EAGER를 무분별하게 사용해서 성능의 저하가 생긴다.
> * 굳이 EAGER 전략을 사용하지 않아도 되는 부분에서 READ의 저하가 생기기 때문에, LAZY 전략을 도입한다.
> * 하지만 LAZY 전략을 도입해도 프록시 객체를 조회하는 메소드가 있기 때문에 결국 N+1 쿼리가 발생한다.

그럼 이러한 문제를 어떻게 해결할 수 있을까? 이어서 설명할거지만, 나는 fetch join과 lazy한 SQL function을 분리해서 각각의 상황에 맞게 repository에서의 함수를 호출했다.

<br>

## 3. N+1 해결
따라서 지금 발생하는 N+1 문제를 `Fetch Join`을 활용해서 해결하고자 한다. 물론 Fetch Join을 사용하게 되면 `Pagenation`을 활용할 수 없다는 단점이 있지만, 지금 현재는 Pagenation을 활용하고 있지 않을 뿐더러 Pagenation이 필요한 엔티티 관계는 아니기 때문에 Fetch Join을 활용하고자 한다.

우선 Fetch join을 활용하는 SQL Function과 그렇지 않은 SQL Function을 만든다.

```kotlin
    @Query("select u from Users u join fetch u.planners p where u.user_id = :user_id")
    fun findUsersByUser_idFetchPlanners (@Param("user_id")user_id : String) : Optional<Users>

    @Query("select u from Users u where u.user_id = ?1")
    fun findUsersByUser_idWithLazy (user_id : String) : Optional<Users>
```

사실 이미 Lazy 전략을 사용하고 있기 때문에 굳이 `findUsersByUser_idWithLazy` 메소드를 만들 필요는 없었지만, 서비스 Layer에서 메소드 사용 시 헷갈릴 여지가 많기 때문에 두 SQL Function을 분리해서 사용하였다.

이후 Users를 조회할 때 Planners를 같이 조회해야 하는 메소드와 그렇지 않은 메소드로 분류해서, 전자에 해당하면 FetchJoin을 사용한 SQL Function을 사용하고, 후자에 해당하면 Lazy 전략을 활용한 SQL Function을 사용할 수 있도록 코드를 수정하였다.

```kotlin
    // Lazy 전략을 활용한 예시 : UsersService.login

    fun login(usersLoginRequestDto: UsersLoginRequestDto): UsersLoginResReturnDto {
        // Match email first. If failed, throw NotAcceptedException
        if (!isExistUserId(usersLoginRequestDto.user_id)) {
            logger.warn("Fail to Login because email is not valid with email value : '{}'", usersLoginRequestDto.user_id)
            throw NotAcceptedException(DummyUsersLoginResReturnDto("Email is not valid"))
        }
        val users = findUsersWithLazy(usersLoginRequestDto.user_id)
        // Match password second with passwordEncoder. If failed, throw NotAcceptedException
        if (!passwordEncoder.matches(
                usersLoginRequestDto.password,
                users.password
            )
        ) {
            logger.warn("Fail to Login because password is not valid")
            throw NotAcceptedException(DummyUsersLoginResReturnDto("Password is not valid"))
        }
        // Create accessToken and refreshToken via jwtTokenProvider
        val accessToken: String = jwtTokenProvider.createAccessToken(usersLoginRequestDto.user_id)
        val refreshToken: String = jwtTokenProvider.createRefreshToken(usersLoginRequestDto.user_id)
        logger.info("Success to Login. Return UsersLoginResReturnDto with accessToken and refreshToken")
        return UsersLoginResReturnDto(users.user_id!!, accessToken, users.nickname, refreshToken)
    }

    // Fetch Join을 활용한 예시 : UsersService.addPlanners

    @Transactional
    fun addPlanners(planner_id: Long, user_id: String): Long? {
        val planners: Planners = findPlanners(planner_id)
        val users: Users = findUsersWithFetchPlanner(user_id)
        users.addPlanners(planners)
        planners.addUsers(users)
        logger.info("Planner with planner_id : '{}' is added with User with user_id : '{}'", planner_id, user_id)
        return planners.planner_id
    }
```

이렇게 적재적소에 맞게 SQL Function을 활용해서 N+1 문제를 해결할 수 있었고, 차후에 성능 테스트도 해봐야겠지만 지금 당장의 Integration Test를 show.sql=true 옵션으로 테스트를 돌려보면 쿼리의 갯수가 현저히 줄어든 것을 확인할 수 있었다.

물론 해당 방법이 정답은 아니다. 첫째로 PageNation을 활용할 수 없다는 큰 단점이 있고, 둘째로는 엔티티에 따라서 Repository SQL Function이 너무 많아져서 Service Layer에 의해서 Repository Layer의 기능이 침범될 수 있다는 점이다. (서로 다른 Layer는 간섭이 일어나지 않는 것이 좋지만 Repository Layer가 Service Layer에게 간섭받는 형태가 된다.) 따라서 필요에 따라 그 양이 많지 않다면 여러 개의 엔티티를 한 번에 fetch join 하는 식으로 사용하는 것도 좋은 방법 중 하나라고 생각된다. 예를 들어서 Users가 Planners, Articles, Photos와 Join되어 있다고 가정할 때 Articles와 Photos가 논리적으로 많이 연관되어 있다면, 대부분의 Users 메소드는 Articles와 Photos를 한 번에 활용하는 경우가 많을 것이고, 이때 `1)Users Fetch Join Articles`, `2)Users Fetch Join Photos`, `3)Users Fetch Join Articles Fetch Join Photos`와 같이 3개로 나누기보다 가장 마지막의 `3)Users Fetch Join Articles Fetch Join Photos` 하나의 메소드로만 세 메소드를 대체하는 것이다. 이러한 균형을 잘 맞춰서 작성한다면 Layer 간의 침범도 그만큼 줄일 수 있을 것이다.

또 해당 기능을 리펙토링 하면서 한 가지 추가적인 문제를 만날 수 있었는데, 우선 기존에 작성된 코드를 보자.
```kotlin
    @Query("select u from Users u join fetch u.planners p where u.user_id = :user_id")
    fun findUsersByUser_idFetchPlanners (@Param("user_id")user_id : String) : Optional<Users>
```
다음과 같이 join fetch를 활용해서 Users를 조회하였으나, 아래와 같은 문제가 발생했다.
> * Users.Planners 에 한 개 이상의 객체가 존재한다면 READ가 정상적으로 작동, **하지만**
> * Users.Planners 가 비어있다면 SQL Function이 NULL을 리턴
> 
그리고 이 문제는 [다음 글](https://stackoverflow.com/questions/43879474/jpa-join-fetch-results-to-null-on-empty-many-side)을 참조해서 해결할 수 있었다. `FETCH JOIN` 은 사실 SQL Function이 아니라 JPQL에서 제공해주는 Function이다. 즉, JPQL로 작성한 해당 쿼리는 SQL에서 `inner join`으로 변환되어 실행된다. inner join은 교집합을 결과로 줄텐데, **만약 USERS가 PLANNERS를 하나도 가지고 있지 않다면 USERS와 PLANNERS의 inner join은 존재하지 않게 되고, 이렇게 되면 USERS 테이블의 PLANNERS를 가지고 있지 않은 엔티티들은 Fetch Join의 결과값에서 제외된다는 것이다.** 따라서 이를 해결하기 위해서 교집합 뿐만 아니라 Users 전체를 포함하기 위해서 `LEFT` 키워드를 추가로 사용해서 PLANNERS를 가지고 있지 않은 USERS 역시 리턴할 수 있도록 하였다.
```kotlin
    @Query("select u from Users u left join fetch u.planners p where u.user_id = :user_id")
    fun findUsersByUser_idFetchPlanners (@Param("user_id")user
