# Mock vs Stub

## Regular Tests
```java
public class OrderStateTester extends TestCase {
  private static String TALISKER = "Talisker";
  private static String HIGHLAND_PARK = "Highland Park";
  private Warehouse warehouse = new WarehouseImpl();

  protected void setUp() throws Exception {
    warehouse.add(TALISKER, 50);
    warehouse.add(HIGHLAND_PARK, 25);
  }

  public void testOrderIsFilledIfEnoughInWarehouse() {
    Order order = new Order(TALISKER, 50);
    order.fill(warehouse);
    assertTrue(order.isFilled());
    assertEquals(0, warehouse.getInventory(TALISKER));
  }

  public void testOrderDoesNotRemoveIfNotEnough() {
    Order order = new Order(TALISKER, 51);
    order.fill(warehouse);
    assertFalse(order.isFilled());
    assertEquals(50, warehouse.getInventory(TALISKER));
  }
}
```
일반적으로 xUnit 테스트는 4단계 순서 (설정, 실행, 확인, 분해)를 따른다. 위의 예제에서 설정 단계는 `setUp` 메소드에서, 수행된다. 실행 단계는 두 테스트 메소드의 `order.fill` 에 해당할 것이다. 그 다음 2개의 `assert` 구문은 실행된 메소드가 올바르게 동작했는지에 대해 확인하는 검증 단계이다. 해당 코드에서 분해는 암묵적으로 생략되었지만, Garbage Collection에 의해서 사용한 객체들이 알아서 분해될 것이다.

위 테스트의 설정 단계에서 우리는 두 종류의 객체를 사용했다. 우선 우리가 테스트하고자 하는 `Order` 클래스를 사용했고, 다른 하나는 `Order.fill`을 작동시키기 위한 `Warehouse` 클래스를 사용했다. 여기서 `Order` 클래스는 우리가 테스트에 집중하는 대상이고 이를 `SUT(System Under Test)` 라는 약어로 부르곤 한다.

이 테스트를 위해서는 `SUT(order)` 와 `collaborator(warehouse)`가 필요하다. 여기서 collaborator가 필요한 이유는 2가지인데, 하나는 테스트된 동작이 작동하도록 하는 것이고 (Order.fill 메소드가 warehouse를 호출하기 때문에) 나머지 하나는 테스트의 검증을 위해 필요하다. (Order.fill 메소드의 결과는 warehouse 객체 내부의 하나의 멤버 값의 감소를 야기하기 때문이다.)

위의 예시와 같은 스타일의 테스트를 보고 `상태 확인 (state veritication)` 이라고 한다. 즉, 메소드가 실행된 후 SUT 및 collaborator의 상태를 검사해서 실행된 메소드가 올바르게 작동했는지 여부를 결정하는 것이다. 이와 다르게 Mock Object는 검증에 대한 다른 접근 방식을 가능케 한다.

## Test with Mock Objects
위의 테스트 코드와 같은 상황을 `Mock Object`를 사용해서 표현해보자. 
```java
public class OrderInteractionTester extends MockObjectTestCase {
  private static String TALISKER = "Talisker";

  public void testFillingRemovesInventoryIfInStock() {
    //setup - data
    Order order = new Order(TALISKER, 50);
    Mock warehouseMock = new Mock(Warehouse.class);
    
    //setup - expectations
    warehouseMock.expects(once()).method("hasInventory")
      .with(eq(TALISKER),eq(50))
      .will(returnValue(true));
    warehouseMock.expects(once()).method("remove")
      .with(eq(TALISKER), eq(50))
      .after("hasInventory");

    //exercise
    order.fill((Warehouse) warehouseMock.proxy());
    
    //verify
    warehouseMock.verify();
    assertTrue(order.isFilled());
  }

  public void testFillingDoesNotRemoveIfNotEnoughInStock() {
    Order order = new Order(TALISKER, 51);    
    Mock warehouse = mock(Warehouse.class);
      
    warehouse.expects(once()).method("hasInventory")
      .withAnyArguments()
      .will(returnValue(false));

    order.fill((Warehouse) warehouse.proxy());

    assertFalse(order.isFilled());
  }
}
```
우선 설정 단계부터 차이점이 존재한다. 설정 단계는 `data` 그리고 `expectation`으로 나뉜다. data 단계에서는 우리가 어떻게 작동하는지 관심있어 하는 객체를 설정한다. (Regular Test와 비슷하게 작동한다.) Regular Test와의 가장 큰 차이점은 Collaborator에 있다. SUT는 동일하게 Order이지만, 위 테스트 코드에서의 Collaborator는 처음 예시처럼 warehouse 객체가 아니라 Mock warehouse 객체이다. (기술적으로는 Mock 글래스의 인스턴스이다.)

설정 단계의 두번째 단계에서는 `mock object`에 대한 `expectation`을 생성하는 것이다. expectation은 SUT가 실행될 때 mock object에서 호출되어야 하는 메소드를 의미한다.

모든 expectation의 셋팅을 완료하면 SUT를 실행한다. 실행 이후 검증 단계는 2가지 관점을 가진다. 우선 첫 번째 관점으로 SUT에 대해서 `assert`를 진행하는데 이는 기존과 동일한다. 나머지 관점으로는 mock을 `verify` 하는 것인데, mock들이 각자의 expectation에 따라서 호출되었는지를 검증하는 것이다.

여기서의 주요 차이점은 order이 warehouse와의 상호 작용에서 올바른 작업을 수행했는지 확인하는 방법이다. state verification에선 우리는 이것을 warehouse 객체의 상태를 검증하는 것으로 실행했다. 하지만 mock은 `행위 검증(behavior verification)`을 사용하는데, 이는 order 객체가 warehouse를 올바르게 호출했는지에 대해 검증하는 것이다. 우리는 이를 설정 단계에서 mock 객체에게 무엇을 기대하는지 알려주고 검증 단계에서 mock 스스로에게 검증을 물어보는 방식으로 검증한다. assertion을 사용해서 주문만 확인한다.

## Different between Mock and Stub
전체 소포트웨어의 하나의 요소에 초점을 두고 테스트를 진행하는 것을 `Unit Test` 라고 한다. 단위 테스트는 하나의 요소에 대해서만 초점을 두고 진행하지만 하나의 요소가 작동하도록 만들기 위해서는 다른 요소들도 필요로 한다는 문제가 있다. 그래서 위의 테스트 예제의 warehouse와 같은 것들이 필요하다.

위의 두 가지 테스트 방법에서 첫 번째 방법은 진짜 warehouse 객체를 사용해서 테스트를 진행하였고 두 번째 방법은 진짜 객체가 아닌 mock warehouse 객체를 사용해서 테스트를 진행했다. 태스트애서 진짜 warehouse를 사용하지 않는 방법중에 하나는 위의 예제처럼 mock를 사용하는 것인데, 테스트를 위해서 mock 뿐만 아니라 실제 객체가 아닌 다른 가짜 객체들을 활용하는 방법들이 더 있다.

`Test Double`이란 진짜 객체가 와야 할 자리에 테스트의 목적을 위해서 사용하는 의도된 다양한 종류들을 의미한다. 이러한 test double에는 5가지 종류가 있다.
* **Dummy** 객체는 전달은 되지만 실제로 사용되지 않는 객체이다. 보통 전달인자 리스트를 채우기 위해서 사용된다.
* **Fake** 객체는 동작하는 implementations를 가지고 있지만 실제 production을 위해서 사용할 정도로 상세하지는 않는, 테스트만을 위해서 implement되는 객체이다.
* **Stub**은 테스트 중에 만들어진 호출에 미리 준비된 답변을 제공하며 일반적으로 테스트를 위해 프로그래밍 된 것이외의 항목에는 전혀 응답하지 않는다.
* **Spy**는 가짜 객체가 아닌 실제 객체를 생성하지만 특정 항목에 대해서만 Stub 할 수 있다는 점을 가진 객체이다.
* **Mock**은 받을 것으로 예상되는 특정된 호출 (파라미터 등이 정해진 호출)에 대해서 특정 응답을 리턴하는 미리 프로그래밍 된 가짜 객체를 의미한다.

위 종류의 test doubles 중에서 mock만 behavior verification을 요구한다. 다른 test doubles들은 state verification을 사용할 수 있으며 일반적으로 사용한다. Mock은 SUT가 실제 collaborator와 상호작용 중이라고 믿게 해야 하기 때문에 준비와 검증 단계에서 나머지와 차이를 보인다.

아래 예시는 Stub을 사용해서 테스트 하는 예시이다.
```java
public interface MailService {
  public void send (Message msg);
}

public class MailServiceStub implements MailService {
  private List<Message> messages = new ArrayList<Message>();
  public void send (Message msg) {
    messages.add(msg);
  }
  public int numberSent() {
    return messages.size();
  }
}

class OrderStateTester {
  public void testOrderSendsMailIfUnfilled() {
    Order order = new Order(TALISKER, 51);
    Warehouse warehouse = new Warehouse();
    MailServiceStub mailer = new MailServiceStub();
    order.setMailer(mailer);
    order.fill(warehouse);
    assertEquals(1, mailer.numberSent());
  }
}
```
MailService를 테스트하기 위해서, 실제 MailService 인터페이스를 상속받는 Stub 객체를 정의하고, 해당 객체를 collaborator로 사용한다.

그럼 이번에는 Mock을 사용해서 테스트 하는 예시를 살펴보자.
```java
class OrderInteractionTester {
  public void testOrderSendsMailIfUnfilled() {
    Order order = new Order(TALISKER, 51);
    Mock warehouse = mock(Warehouse.class);
    Mock mailer = mock(MailService.class);
    order.setMailer((MailService) mailer.proxy());

    mailer.expects(once()).method("send");
    warehouse.expects(once()).method("hasInventory")
      .withAnyArguments()
      .will(returnValue(false));

    order.fill((Warehouse) warehouse.proxy());
  }
}
```
두 테스트에서 둘 다 test double을 각각 Stub, Mock으로 사용했다. 차이점으론 stub을 사용할 때는 상태 검증을 시행한 반면 mock을 사용할 때는 행위 검증을 시행했다. stub의 상태 검증을 위해서 MailServiceStub 객체에 검증을 위한 numberSent 메소드를 추가로 정의해서 사용하였다. mock을 사용할 때는 항상 행위 검증을 시행하는데, stub 역시도 행위 검증이 가능하다. 


참조자료 : <br>
[Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)