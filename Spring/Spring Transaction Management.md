# Spring Transaction Management
_아래 글은 [Spring Reference](https://docs.spring.io/spring-framework/reference/data-access/transaction.html) 을 보고 공부한 내용을 요약해서 남긴 글입니다. 지극히 주관적인 글이니 참고만 해 주시면 감사하겠습니다._

**Comprehensive transaction support**은 Spring Framework를 사용하는 강력한 이유중 하나이다. Spring Framework는 transaction management를 위해 아래 장점을 가지는 **consistent abstraction** (일관성 있는 추상화) 를 제공한다.

* 서로 다른 트랜잭션 API들 사이에서 일관성있는 프로그래밍 모델 제공 (예시 : Java Transaction API, GDBC, Hibernate, JPA)
* 선언 가능 한 트랜잭션 관리 (declarative transaction management)
* Programmatic transaction을 위한 간단한 API (예시 : JTA)
* Spring의 data access abstraction과 뛰어난 통합성

## Advantage of the Spring Framework's Transaction Support Model
전통적으로 EE 어플리케이션 개발자들은 트랜잭션 관리를 위해서 1) 전역 트랜잭션 2) 로컬 트랜잭션 두 가지의 선택지를 가지고 있었으나, 둘 다 각자의 한계를 가지고 있다. 각자에 대해 제한 사항들을 알아보고 Spring Framework의 트랜잭션 관리가 이를 어떻게 해결하는지 알아본다.

### Global Transactions
전역 트랜잭션을 활용하면 관계형 데이터베이스와 메시지-큐 와 같은 여러 트랜잭션 리소스에서 작업할 수 있다. 어플리케이션 서버는 JTA를 활용해서 전역 트랜잭션을 관리한다. 게다가 JTA는 일반적으로 JNDI에서 sourced 되야 하는데, 이때문에 개발자는 JTA를 사용하기 위해서 JNDI를 필요로 했다. JTA는 일반적으로 어플리케이션 서버 환경에서만 사용할 수 있음으로 전역 트랜잭션을 사용하면 어플리케이션 코드의 잠재적인 재사용이 제한된다.

과거 전역 트랜잭션을 위해 권장되는 방법은 EJB CMT (Container Managed Transaction) 였다. CMT는 선언형 트랜잭션 관리의 한 형태였다. (프로그래밍 트랜잭션 관리와 구별됨) EJB CMT는 트랜잭션 관련 JNDI 조회 필요성을 제거하지만 EJB 자체를 사용하려면 JNDI를 사용해야 한다. 이는 트랜잭션 관리를 위해서 전부는 아니지만 대부분의 자바 코드를 제거할 수 있다. 중요한 단점은 CMT가 JTA 및 어플리케이션 서버 환경에 묶여 있다는 것이다. 또한 EJB에서 비즈니스 로직을 구현하기로 선택한 경우에만 사용할 수 있다. 일반적으로 EJB의 단점은 너무 커서 선언형 트랜잭션 관리를 위한 매력적인 대안에 직면한 좋은 방법은 아니다.

### Local Transactions
로컬 트랜잭션은 JDBC 연결과 관련된 트랜잭션과 같이 리소스 별로 다르다. 로컬 트랜잭션은 사용하기는 쉽지만 여러 트랜잭션 리소스에서느 작동할 수 없다는 큰 단점이 있다. 예를 들어서 JDBC 연결을 사용해서 트랜잭션을 관리하는 코드는 전역 JTA 트랜잭션 내에서 실행할 수 없다. 어플리케이션 서버는 트랜잭션 관리에 관여하지 않기 때문에, 서버는 여러 리소스에서 정확성을 보장할 수 없다. 또 다른 단점은 로컬 트랜잭션이 프로그래밍 모델을 침범한다는 것이다.

### Spring Framework's Consistent Programming Model
Spring은 전역 및 로컬 트랜잭션의 단점을 해결한다. 이것은 개발자가 어떤 환경에서는 일관된 개발을 할 수 있도록 도와준다. 코드를 한 번 작성하면 다양한 환경에서 다양한 트랜잭션 관리 전략을 활용할 수 있다. Spring Framework에서는 declarative하고 programmatic한 트랜잭션 관리를 둘 다 제공한다. 대부분의 사용자들은 declarative transaction management를 선호하고, Spring 팀 역시 이를 추천한다.

Programmatic 트랜잭션 관리에선 개발자는 Spring Framework transaction abstraction과 함께 모든 트랜잭션 인프라에서 실행할 수 있게 작업한다. 선호되는 declarative 모델과 함께 개발자는 아주 조금 혹은 코드를 작성하지 않기 때문에 Spring Framework Transaction API 또는 기타 트랜잭션 API에 의존하지 않아도 된다.

참조자료 : <br>
[Spring Reference : Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)