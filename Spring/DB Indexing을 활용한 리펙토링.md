# DB Indexing을 활용한 리펙토링
## 1. Index란?

`인덱스`는 책의 맨 끝에 있는 찾아보기 (또는 색인)과 비슷하다. 데이터를 빠르게 찾기 위해서 사용하는 하나의 지표와 같은 것이다. DBMS에서 데이터베이스 테이블의 모든 데이터를 검색해서 원하는 결과를 가져오려면 시간이 오래 걸린다. 그래서 **칼럼의 값과 해당 레코드가 저장된 주소를 키와 값의 쌍으로 삼아 인덱스**를 만들어준다. 이러한 인덱스는 DBMS에서 내가 원하는 값을 아주 빠르게 찾아갈 수 있도록 하는 것을 도와주며, 결과적으로 `READ의 성능을 비약적으로 올려주는 역할`을 한다.

인덱스에는 이러한 장점이 있는 만큼 단점 역시 존재한다. SortedList와 ArrayList를 보면 인덱스의 장점을 바로 알 수 있는데, ArrayList에서의 insert는 O(1) 만큼의 시간이 걸리지만 find에는 O(n)의 시간이 걸린다. **SortedList는 find에서는 ArrayList보다 훨씬 장점을 가지지만 반대로 insert나 update 등에 있어서는 매우 불리하다.** 항상 정렬된 상태를 유지해야 하기 때문이다.

이러한 점을 바탕으로, DBMS에서 **`인덱스는 데이터의 저장 (INSERT,UPDATE,DELETE) 성능을 희생하고 그 대신 데이터의 읽기 속도를 높이는 작업`** 이라고 할 수 있다. 즉, 찾기와 저장 중 어느 것이 더 자주 일어나는지를 비교해서 인덱스를 사용해야 하며, 또한 WHERE 조건절에 사용된다도 무분별하게 인덱스를 사용하게 된다면 인덱스의 크기가 비대해져 오히려 역효과를 부를 수 있다.

DBMS에서 이러한 Index가 어떤 식으로 작동하고 구현하는 지에 대해서는 굳이 글로 정리하진 않겠다. (해당 부분은 나중에 CS 공부를 하며 정리를 남길 생각이다) 여튼, 이제는 이러한 DB Index를 활용해서 Spring 레거시를 개선해 보고자 한다.

<br>

## 2. 레거시 분석

우선 현재 DBMS에서 어떤 쿼리가 사용되는지를 알아보기 위해서, Repository 계층의 SQL Function들을 한 번 보고자 한다.

```kotlin
interface UsersRepository : JpaRepository<Users, String> {

    fun existsByNickname (user_id : String) : Boolean

    @Query("select (count(u) > 0) from Users u where u.user_id = ?1")
    fun existsByUser_id (user_id : String) : Boolean

    @Query("select u from Users u left join fetch u.planners p where u.user_id = :user_id")
    fun findUsersByUser_idFetchPlanners (@Param("user_id")user_id : String) : Optional<Users>

    @Query("select u from Users u where u.user_id = ?1")
    fun findUsersByUser_idWithLazy (user_id : String) : Optional<Users>

}
```
Users Repository에서 user_id 그리고 nickname으로 Users 존재 여부를 찾는, **EXIST SQL Function**이 둘 있고, 그리고 Planners를 **join fetch하는 SQL Function**이 있는 상태이다.

그 다음으로는 엔티티를 한 번 봐보자.

```kotlin
@Entity
@EnableJpaAuditing
class Users(
    @Id @Column(name = "USER_ID")
    val user_id: String? = null,

    @Column
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinColumn(name = "PLANNERS_USERS")
    var planners: MutableSet<Planners> = TreeSet(),

    //생략
)
```

현재 Users 엔티티 클래스에서는 아무런 Index를 사용하지 않고 있다. 그렇게 생각할 수 있지만, 사실은 JPA에서 이미 PK를 Unique Key이자 Index로 사용하고 있다. @Id 어노테이션에 의해서 이미 우리는 하나의 인덱스를 사용하고 있는 것과 동일하다. [출처](https://www.baeldung.com/jpa-indexes)

그럼 이제 문제가 되는 서비스 코드와 개선 방안을 생각해 보려고 하는데, 우선 개선하고자 하는 서비스의 메소드는 아래와 같다.
```kotlin
fun findUserWithNicknameOrUserId(nicknameOrUserId: String): String? {
    val usersList: List<Users> = usersRepository.findAll()
    for (users: Users in usersList) {
        if (users.user_id == nicknameOrUserId || users.nickname == nicknameOrUserId)
            return users.user_id
    }
    logger.warn("Fail to find User with Nickname or UserId value : '{}'", nicknameOrUserId)
    throw FailToFindNicknameOrIdException("Fail To Find Nickname Or Id matched Users!")
}
```

위 메소드는 사용자를 **"닉네임 또는 아이디로 검색했을 때 사용자 DTO를 리턴하는"** API 메소드이다. 우선 닉네임과 아이디는 전부 unique 값이며 닉네임과 아이디는 중복될 일이 없다. (아이디는 이메일 형식이기 때문에 중복일 수 없다.) 그런데 지금 해당 메소드가 처리되는 방식을 보면 users를 전부 받아온 다음 list에서 찾기를 수행하는데, 이렇게 수행하게 되면 **"사용자가 많을 경우, 시간이 진짜 오래 걸린다"** 라는 아주 큰 문제가 있다. 

그래서 해당 문제를 어떻게 풀어낼 수 있을까 고민하던 도중, `인덱스`와 `SQL Function`을 적절히 활용하는 방향으로 해결해 보고자 한다.

## 3. DB Index 및 SQL Function을 활용한 해결
우선 ArrayList에서 완전 탐색은 `O(n) 만큼의 시간 복잡도`를 필요로 한다. 반면 SortedList에서 완전 탐색은 `O(logN) 만큼의 시간 복잡도` 를 필요로 하기 때문에, 비슷한 원리로 Index를 사용하는 것이 훨씬 좋을 것이다.

하지만 그 전에, 다음과 같은 체크 리스트를 먼저 점검했다.
> 1. Index를 활용하고자 한다면, READ가 INSERT, DELETE, UPDATE 의 성능 저하를 희생하더라도 우선시되야 하는가?
> 2. 어떤 Index를 활용해서 해당 작업을 수행할 것인가? 복합으로, 아니면 단일으로 등 ?

우선 Users 엔티티는 회원 가입을 제외하고는 INSERT는 따로 일어나지 않는다. 마찬가지로 회원 탈퇴 역시 DELETE를 제외하고는 일어나지 않는다. UPDATE의 경우 자주 일어나지만 현재 PK로 사용되고 있는 INDEX의 경우 UPDATE가 일어나지 않는다.

이어서 어떤 INDEX를 사용할 것인가를 고민해 볼 필요가 있는데, 현재 수정하고자 하는 메소드는 **"닉네임 또는 아이디로 검색했을 때 사용자 DTO를 리턴하는"** API 메소드이기 때문에, Users를 Nickname으로 검색할 수 있도록 하는 SQL Function을 활용할 것이다. 그렇다면, Nickname을 Index로 사용하는 것이 좋을 것 같은데, 우선 메소드를 어떤 식으로 리펙토링 할 것이지 먼저 살펴보자.

> 1. Input의 String이 user_id인지, nickname인지 확인한다.
> 2. user_id라면 user_id로 SQL Function_findById를 사용한다.
> 3. nickname이라면 nickname으로 SQL Function_findByNickname을 사용한다.
> 4. 1의 결과가 둘 다 false라면, 바로 Exception을 Throw한다.

이런 식으로 리펙토링 하고자 하고자 한다. 총 4가지의 SQL Function을 활용해야 하는데, `ExistByUser_id` `ExistByNickname` `findByUser_id` `findByNickname` 총 4가지를 활용할 것이다. 이미 User_id는 PK로 등록이 되어 있고, 비록 아이디 또는 닉네임을 인풋과 동일한 유저를 검색해야 하지만 `select Users u from Users where user_id=$1 or nickname=$1` 와 같은 형식의 쿼리를 날리는 것이 아니기 때문에, Nickname을 단일 인덱스로 사용하는 것이 좋을 것 같다. 또한 **여러 개의 인덱스가 있을 경우 가장 적합한 인덱스를 찾아주기 때문에 PK와 더불어 nickname을 단일 index로** 사용하고자 한다. 마지막으로, **nickname을 제외하고는 where 절에서 사용하는 컬럼이 없기 때문에 nickname을 단일 인덱스로 추가**하고자 한다.

```kotlin
@Entity
@EnableJpaAuditing
@Table(indexes = [Index(name = "idx_nickname", columnList = "nickname")])
class Users(
    @Id @Column(name = "USER_ID")
    val user_id: String,

    @Column
    val nickname: String,
    //생략
){}
```
우선 위와 같이 nickname을 사용하는 index를 추가로 사용해준다. 다음으로 SQL Function을 추가로 정의해준다.
```kotlin
interface UsersRepository : JpaRepository<Users, String> {
    fun existsByNickname (user_id : String) : Boolean
    @Query("select (count(u) > 0) from Users u where u.user_id = ?1")
    fun existsByUser_id (user_id : String) : Boolean

    @Query("select u from Users u left join fetch u.planners p where u.user_id = :user_id")
    fun findUsersByUser_idFetchPlanners (@Param("user_id")user_id : String) : Optional<Users>

    @Query("select u from Users u where u.user_id = ?1")
    fun findUsersByUser_idWithLazy (user_id : String) : Optional<Users>

    fun findUsersByNickname (nickname : String) : Optional<Users>

}
```
마지막으로 위에서 활용한 점을 바탕으로 서비스 API 코드를 개선하였다.
```kotlin
    fun findUserWithNicknameOrUserId(nicknameOrUserId: String): String? {
        if (usersRepository.existsByUser_id(nicknameOrUserId)) {
            val users = usersRepository.findUsersByUser_idWithLazy(nicknameOrUserId).get()
            return users.user_id
        }
        if (usersRepository.existsByNickname(nicknameOrUserId)) {
            val users = usersRepository.findUsersByNickname(nicknameOrUserId).get()
            return users.user_id
        }
        logger.warn("Fail to find User with Nickname or UserId value : '{}'", nicknameOrUserId)
        throw FailToFindNicknameOrIdException("Fail To Find Nickname Or Id matched Users!")
    }
```
이렇게 코드를 개선함으로써, 다음과 같은 플로우들에 대해서 이점을 볼 수 있었다.
> 1. input이 user_id인 경우 첫 번째 조건문에 의해서 바로 검색하게 된다. PK는 이미 Index이기 때문에 검색 속도가 개선된다.
> 2. input이 nickname인 경우 첫 번째 조건문에 의해서 user_id를 활용한 검색은 하지 않고 바로 두 번째 조건문에 의해서 nickname으로 검색을 시작하게 된다. 이때 Index를 사용하기 때문에 검색 속도가 빠르다.
> 3. input이 존재하지 않는 user_id 또는 nickname이라면 굳이 검색을 하지 않고 exist Function만 사용해서 바로 예외처리가 가능하다.

그럼 실제로 얼마나 빨라 졌는지에 대해서 테스트를 해볼 필요가 있다. 다음과 같이 usersRepository에 랜덤한 사용자 50만명을 넣고 테스트를 진행하였다.
```kotlin
        given("오십 만명의 랜덤 사용자가 저장되어 있을 때") {

            fun getRandomString () : String {
                val charset = ('a'..'z') + ('A'..'Z') + ('0'..'9')
                val length = (8..12).random()
                return (1..length)
                    .map { charset.random() }
                    .joinToString("")
            }

            fun getRandomEmail() : String
                = listOf<String>("gmail.com", "naver.com", "ajou.ac.kr").random()

            for (i in 1..500000) {
                val usersSaveRequestDto = UsersSaveRequestDto(
                    user_id = getRandomString() + "@" + getRandomEmail(),
                    nickname = getRandomString(),
                    gender= 1
                )
                val users = usersSaveRequestDto.toEntity()
                usersRepository.save(users)
            }

            `when`("특정 사용자 조회를 하면") {
                then("UsersReturnDto가 리턴된다") {
                    stopWatch.start()

                    mockMvc.get("/api/v1/users/findUserWithNicknameOrUserId/{nicknameOrUserId}", user_id) {
                        contentType = MediaType.APPLICATION_JSON
                        header("AccessToken", accessToken)
                    }.andExpect {
                        status { isOk() }
                    }

                    stopWatch.stop()
                    logger.info("오십만 개의 사용자가 있을 때 아이디를 사용해서 findUsersWithNicknameOrUserId")
                    logger.info(stopWatch.prettyPrint())


                    stopWatch.start()

                    mockMvc.get("/api/v1/users/findUserWithNicknameOrUserId/{nicknameOrUserId}", nickname) {
                        contentType = MediaType.APPLICATION_JSON
                        header("AccessToken", accessToken)
                    }.andExpect {
                        status { isOk() }
                    }

                    stopWatch.stop()
                    logger.info("오십만 개의 사용자가 있을 때 닉네임을 사용해서 findUsersWithNicknameOrUserId")
                    logger.info(stopWatch.prettyPrint())

                }
            }
        }
```

그리고 그 결과는 다음과 같았다.
> 인덱스 사용하고, <br>
> 아이디 : running time = 67042750 ns <br>
> 닉네임 : running time = 109517833 ns <br> <br>
> 인덱스 사용하지 않고, <br>
> 아이디 : running time = 101723208 ns <br>
> 닉네임 : running time = 268639291 ns 

결과를 보면 닉네임을 사용해서 검색을 요구했을 때 `최대 약 60% 가까이 실행 시간을 단축`할 수 있었다. Index와 API 메소드를 개선해서 이렇게 변했지만, N+1 등 기존에 레거시와 현재의 메소드를 비교하면 (모든 리펙토링이 끝난 후 진행할 예정) 그 이상의 성능 개선이 되었음을 확인할 수 있을 것 같다.