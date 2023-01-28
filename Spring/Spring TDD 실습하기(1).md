# Spring TDD 실습하기(1)
## 0. TDD란?
TDD 란 `Test Driven Development`의 약자로써, 테스트 주도 개발을 의미한다. 즉, 테스트 코드에 의해서 개발이 진행되는 것으로 `테스트 코드 작성` 의 선행이 이루어 지고 난 후에 본 코드를 작성하고, 이후에 테스트 코드를 통해서 본 코드의 개발이 제대로 잘 이루어 졌는지 검증하는 개발 방법론을 의미한다.

Test 코드를 작성하는 장점은 TDD 적용 시 당연하게 따라오는 것이고, TDD 를 적용하게 되면 테스트 코드를 통해서 내가 개발하고자 하는 모든 기능들을 미리 작성한 채로 본 개발을 실행할 수 있기에, 비록 시간이 오래 걸리고 비효율적이라고 생각될 지언정 결과론적으로는 더욱 시간을 적게 사용해서 개발을 할 수 있는 이점이 있다. (물론 아주 간단한 기능의 경우 이가 불 필요 할 수도 있다)
## 1. 요구사항 정리
Test 코드를 작성하기 위해서 우선 개발하고자 하는 기능의 요구 사항을 정리할 필요가 있었다. 요구사항들을 100% 완벽하게 세부 기능까지 나눠서 작성한 뒤에 이를 바탕으로 Test 코드를 작성하는 것이 가장 좋겠지만, 결론적으로 TDD 를 적용하는 이유 역시 효율성 이라고 생각하기에, 적정 선에서 요구사항을 작성하고 작성된 요구사항을 바탕으로 Test 코드를 작성하고자 하였다.

요구사항의 경우, 
1. Entity 측면에서의 요구사항 정리
2. Service 측면에서의 요구사항 정리

정도로 비교적 간단하게 정리하였다. Entity 와 Service 의 로직을 보면 Dto 정도는 굳이 요구사항으로 정리하지 않아도 된다고 생각했고, Repository 의 경우 Service 의 로직을 참고하면 충분히 작성할 수 있었기에 요구사항을 정리하지 않았다. Controller 역시 마찬가지로 요구사항을 정리하지 않았다.

이번에 실습을 위해서 작성한 요구사항은 아래와 같다.
```markdown
1) Entity
    1. Bags
    String {CountryCode}_{RegionName}_{auto_increment}
    boolean isRented
    LocalDateTime whenIsRented
    Users rentingUsers
    
    2. Users
    List<Bags> RentingBagsList
    List<Bags> ReturningBagsList
    List<Bags> ReturnedBagsList

2) Method (Service)

    1. Bags
    + save (only for admin)
    + rentOrReturn
    - rent : Only for Authorized Users
    - return : Only for Authorized Admin
    
    2. QRCodes
    Using Zxing Library,
    + generateQRCode : with source : Bags_rentOrReturn URL
    
    3. Users
    + getRentingBagsList
    + getReturningBagsList
    + getReturnedBagsList
    + delete (with authorizing)
```
이후 위의 요구사항을 기반으로 본 코드를 개발하고자 한다. 위 모든 사례를 실습할 예정이지만 기록으로 남기는 정보는 Bags Entity 에 한정해서 남기고자 한다.

## 2. Entity 개발 및 Repository 검증
Entity 의 경우 사실 검증이라고 하기가 애매한 것이, `ERD` 를 충분히 잘 그려놓거나 요구사항을 충분히 작성하고 이를 바탕으로 개발한다면 크게 어려운 점이 없을 것이라고 생각하였다. 그래서 Entity 의 경우 Test 코드의 선 작성이 아닌 Entity 의 선 작성을 완료한 뒤 `Repository Layer Test` 를 통해서 `Entity` 들의 `Getter, Setter` 등 까지 점검하고자 한다.

정리하자면, 
1) 요구사항 바탕의 Entity 설계
2) Repository Layer 테스트 코드 작성
3) Repository 코드 작성

순으로 진행하고자 한다. Service 계층에 의해서 Repository Layer 의 추가적인 Test 가 필요할 수 있지만, 이는 Service 계층을 개발할 때 작성하고자 한다.

```java
@Getter
@Setter
@Entity
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Bags {

    @Id @Column
    private String bagsId;

    @NotNull
    @Column
    private boolean isRented;

    @NotNull
    @Column
    private LocalDateTime whenIsRented;

    @ManyToOne(fetch = FetchType.EAGER)
    private Users users;

}
```

```java
@DataJpaTest
@AutoConfigureTestDatabase
public class BagsRepositoryTest {

    @Autowired
    UsersRepository usersRepository;

    @Autowired
    BagsRepository bagsRepository;

    @BeforeAll
    @DisplayName("Users 저장 이후 Bags 저장 테스트")
    void 테스트전_users저장_그리고_Bags저장_테스트() {
        Users users = Users.builder()
                .usersId("test@gmail.com")
                .nickname("testNickname")
                .password("testPassword")
                .userRole(UserRole.ROLE_USER)
                .build();

        usersRepository.save(users);

        //given
        String expectedId = "KOR_SUWON_1";
        LocalDateTime expectedTime = LocalDateTime.now();

        Bags target = Bags.builder()
                .bagsId(expectedId)
                .isRented(false)
                .whenIsRented(expectedTime)
                .build();

        //when
        bagsRepository.save(target);

        //then
        assertThat(target.getBagsId()).isEqualTo(expectedId);
        assertThat(target.isRented()).isEqualTo(false);
        assertThat(target.getWhenIsRented()).isEqualTo(expectedTime);
    }

}
```
```java
package com.ReRollBag.repository;

import com.ReRollBag.domain.entity.Bags;
import org.springframework.data.jpa.repository.JpaRepository;

public interface BagsRepository extends JpaRepository <Bags, String> {
}
```

## 3. Service 개발 및 검증
Service Layer 의 경우, 다음과 같이 개발을 진행하였다.
1. 개발하고자 하는 Service Logic (Method) 의 요구 사항을 파악한다.
2. 요구 사항을 바탕으로 Test Code 를 작성한다.
3. Test Code 가 작동될 수 있도록 본 개발을 실시한다.

아래 예시는 "KOR", "SUWON" 이라는 두 개의 필드를 가지고 있는 BagsSaveRequestDto 를 주고 BagsService 에서 Save 를 실행하였을 때 Save 메소드의 리턴 값이 true 인지 증명하기 위한 과정이다.

BagsService, BagsSaveRequestDto 모두 하나도 구현이 되어 있지 않기 때문에 `cannot resolve ~` 와 같이 에러가 나타난다.
```java
@ExtendWith(MockitoExtension.class)
public class BagsServiceTest {

    @Autowired
    private BagsService bagsService;

    @Mock
    private BagsRepository bagsRepository;

    @Test
    @DisplayName("[Service] Bags 생성")
    public void Service_가방생성_테스트() throws Exception {
        //given
        BagsSaveRequestDto bagsSaveRequestDto = new BagsSaveRequestDto(
                "KOR",
                "SUWON"
        );
        
        Bags bags = bagsSaveRequestDto.toEntity();

        given(bagsRepository.save(any())).willReturn(bags);

        //when
        boolean result = bagsService.save(bagsSaveRequestDto);

        //then
        assertThat(result).isEqualTo(true);
    }
}
```
이후 IntelliJ 를 기준으로 아래 탭의 `Problems` 를 클릭하고, `Project Errors` 를 클릭하면 프로젝트 전체에서 나타나는 모든 에러들을 확인할 수 있는데, 이제 하나씩 해당 에러를 삭제하기 위한 코드를 작성하면 TDD 를 적용해서 본 코드를 개발할 수 있다.