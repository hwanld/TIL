# Transaction Isolation and ReadOnly
**Transaction은 데이터베이스의 상태를 변화시키는 작업의 단위**를 말하며, 일반적으로 INSERT, UPDATE, DELETE가 이에 해당한다. Transaction은 **ACID 특성**을 만족해야 하며, 그중 대표적인 것은 "모두 커밋되거나 아님 모두 롤백대거나" 이다. JPA 환경에서는 Transaction을 위해서 Transaction 객체를 생성하고 start, end와 같이 트랜잭션을 관리했는데, Spring Data JPA 환경에서는 @Transactional 어노테이션 하나로 관리한다. 그런데 만약 **CUD가 아닌 메소드에 대해서는 트랜잭션을 만들어야 할까**? 해당 부분은 **격리성**과 관련되는 부분인데, 이에 알아보기 전에 우선 데이터베이스의 격리수준에 대해 알아보고자 한다.

## 데이터베이스의 격리수준
**트랜잭션의 격리 수준 (isolation level)이란 여러 트랜잭션이 동시에 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할 지 말지를 결정하는 것**이다. 격리 수준은 ANSI 표준에 의해 크게 **"READ UNCOMMITED", "READ COMMITED", "REPEATABLE READ", "SERIALIZABLE"** 4가지로 나뉜다. 순서대로 격리 수준이 낮음>높음 순이며, 격리 수준이 낮을 수록 동시성은 증가하지만 격리 수준에 따른 다양한 문제가 발생한다. 아래 표는 그 문제를 나타낸 것이다.

|격리 수준|DIRTY READ|NON-REPETABLE READ|PHANTOM READ|
|----------|----------|----------------|---------|
|READ UNCOMMITED|O|O|O|
|READ COMMITED| |O|O|
|REPEATABLE READ| | | O(InnoDB는 없음)|
|SERIALIZABLE| | | |

### READ UNCOMMITED
**READ UNCOMMITED 격리 수준에서는 트랜잭션에서의 변경 내용이 COMMIT, ROLLBACK 여부와 상관 없이 다른 트랜잭션에서 보인다**. 즉, 커밋하지 않는 데이터를 읽을 수 있다는 것이다. 

예를 들어서 트랜잭션1이 데이터를 수정하고 있는데 이를 커밋하지 않더라도 트랜잭션2에서 수정 중인 데이터를 조회할 수 있다. 이를 DIRTY READ라고 한다. 만약 트랜잭션1이 데이터를 수정했으나 트랜잭션에 문제가 생겨 커밋하지 않고 롤백하더라도, 그 사이에 트랜잭션2가 데이터를 조회했다면 롤백된 데이터가 트랜잭션2에겐 정상적인 데이터로 인식한다는 점이다. 이러한 DIRTY READ를 유발하는 READ UNCOMMITED는 RDBMS 표준에서는 트랜잭션의 격리 수준으로 인정하지 않을 정도로 정합성에 문제가 많은 격리 수준이다.

### READ COMMITED
오라클 DBMS에서 기본으로 사용되는 격리 수준이며 온라인 서비스에서 가장 많이 선택되는 격리 수준이다. DIRTY READ 현상은 발생하지 않는다. 즉, 하나의 트랜잭션이 데이터를 커밋하기 전에는 다른 트랜잭션에서 해당 데이터를 조회할 수 없다.

해당 격리 수준에서는 **NON-REPETABLE READ 라는 문제**를 가진다. 트랜잭션 1에서 어떤 데이터의 count를 조회했을 때 그 결과가 0이었다고 가정해보자. 이후 트랜잭션 2에서 어떤 데이터를 커밋했고, 다시 트랜잭션 1에서 어떤 데이터의 count를 조회했을때 결과가 이제는 0이 아닌 1이 나오는 상황이 생길 수 있다. 큰 문제가 없어 보일 수 있지만, 하나의 트랜잭션 내에서 똑같은 SELECT 쿼리를 실행했을 때는 항상 같은 결과를 가져와야 한다는 "REPETABLE READ" 정합성에 어긋난다.

### REPETABLE READ
MySQL의 InnoDB가 기본적으로 사용하는 격리 수준이다. 바이너리 로그를 가진 MySQL 서버에서는 최소 REPETABLE READ 이상의 격리 수준을 사용해야 한다. REPETABLE READ는 Undo Log를 사용해서, 해당 커밋이 롤백된다면 테이블이 어떤 식으로 존재해야 하는지에 대한 정보를 저장하게 된다. 만약 트랜잭션1에서 데이터를 요청하고, 이후 해당 데이터를 트랜잭션2에서 수정하고 다시 트랜잭션1에서 해당 데이터를 요청한다면, DB에서는 Undo Log에 있는 정보를 꺼내서 주기 때문에 수정되었지만 이전에 요청한 것과 동일한 데이터를 제공하게 된다. 

해당 격리 수준에서는 **PHANTOM READ 라는 문제**를 가진다. 만약 한 트랜잭션에서 조회한 이후 다른 트랜잭션에서 새로운 데이터를 저장했다고 하자. 이후 원 트랜잭션에서 새로운 데이터가 포함되는 조회 쿼리를 날린다면, 이전의 조회 쿼리와 다른 결과가 리턴되게 된다. 이렇게 하나의 컬럼이 없었다가 생기는 현상을 PHANTOM READ라고 한다.

### SERIALIZABLE
트랜잭션 격리 수준 중에서 가장 엄격한 격리 수준으로, 위에서 언급한 **격리성과 관련된 3가지 문제를 모두 해결할 수 있으나 동시 처리 성능이 다른 트랜잭션 격리 수준에 비해 훨씬 떨어진다**. 읽기 작업 역시 공유 잠금(읽기 잠금)을 획득해야 하며, 동시에 다른 트랜잭션은 그러한 레코드를 변경하지 못하게 된다. 즉, 한 트랜잭션이 한 레코드를 읽기만 하더라도 다른 트랜잭션은 해당 레코드를 건들지 못한다는 말이다.

## @Transactional(readOnly = true)
이제 Spring에서 지원하는 Transactionality에 대해 한 번 알아 보고자 한다. 기본적으로 `CrudRepository`로부터 상속받는 메소드들은 `SimpleJpaRepository`로부터 transactional configuration을 상속받는다. 조회 동작을 위해서는 transaction configuration의 `readOnly` 플래그를 true로 설정한다. 그 외 동작은 그냥 `@Transactional` 하나만 사용해서 default transaction configuration이 적용되도록 사용한다. 

Read-only Transcations를 위해서는 readOnly flag의 값만 변화시키면 된다. `@Transactional(readOnly=true)`와 같이 사용할 수 있다. 하지만 이렇게 설정하는 것은 일부 데이터베이스에선 manipulating 쿼리를 거부하지만 이렇게 하면 mainpulating query를 트리거하지 않는 검사로 작동하지 않는다. 대신 readOnly 플래거는 성능 최적화를 위해 JDBC 드라이버에 힌트로 전파된다. 또한 Spring은 기본 JPA optimizer에 대해 몇 가지 최적화를 수행한다. 예를 들어서 `readOnly` 옵션을 사용한다면 Hibernate를 사용할 때, flush mode는 NEVER로 설정되어 Hibernate가 dirty check를 건너뛰게 된다. (큰 객체 트리에 대해서 성능의 현저한 상향)

참조자료 : <br>
자바 ORM 표준 JPA 프로그래밍 (김영한 지음. 에이콘) <br>
Real MySQL 8.0 (백은빈, 이성욱 지음. 위키북스) <br>
[Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#transactions)<br>