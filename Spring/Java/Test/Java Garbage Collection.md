# Java Garbage Collection
**Garbage Collection**이란 자바의 메모리 관리 방법 중 하나로 JVM의 **Heap 영역에서 동적으로 할당했던 메모리 영역 중 필요 없게 된 메모리 영역을 주기적으로 삭제하는 프로세스**를 의미한다. C와 같은 개발 환경에서는 동적 할당했던 메모리들을 개발자가 일일히 해제하지 않으면 메모리 누수(Leak) 현상이 일어나게 되는데, Java 환경에서는 이를 방지하기 위해 Garbage Collertor가 메모리 관리를 대행하게 된다.

## Describing Garbage Collection

### Step 1 : Marking
첫 번째 프로세스는 marking이라고 한다. 이것은 garbage collector가 어느 위치의 메모리가 현재 사용중이고 어느 위치가 사용중이 아닌지를 식별하는 것이다.

Marking Phase에 있는 객체들은 이러한 결정을 하기 위해서 Garbage collector에 의해 스캔된다. 만약 시스템의 모든 객체들이 스캔되어야 한다면, 해당 프로세스는 시간이 꽤 많이 걸리게 된다.

### Step 2 : Normal Deletion
Normal deletion은 unreferenced 객체들을 삭제하고 이로 인해 생긴 빈 공간에 포인터를 남긴다. 이러한 빈 공간에 새로운 객체가 다시 생성될 수 있도록 빈 공간의 reference를 가지고 있는다.

### Step 2a: Deletion with Compacting
**성능의 향상을 위해서, unreferenced 객체를 지우는 것에 더하여 남아있는 referenced 객체들을 압축**할 수 있다. **referenced 객체들 사이에 빈 공간이 없게 전부 움직여서 빈 공간이 연속되게** 만들고, 이를 통해서 새로운 객체가 할당될 때 더욱 쉽고 빠르게 할 수 있도록 한다.

### Why Generational Garbage Collection?
위에서 설명한 바와 같이 모든 객체들에 대해서 스캔하고 표시하고 압축하는 것은 꽤 오랜 시간이 걸리고, 이는 비효율적이다. 점점 더 많은 객체가 할당됨에 따라 garbage collection time이 길어지게 된다. 하지만 경험적 분석에 따르면 애플리케이션의 대부분의 객체 수명은 짧다.

### JVM Generations
개체 할당에서 학습한 정보는 JVM의 성능을 향상시키는 데 사용할 수 있다. 힙 메모리 공간은 다시 더 작은 세대들로 나눌 수 있는데, Young Generation, Old Generation 그리고 Permanent Generation 으로 나눌 수 있다.

**Young Generation은 모든 새 객체가 할당**되는 곳이다. 만약 Young Generation 공간이 모두 가득 찬다면 Minor Garbage Collection가 일어나게 된다. 대부분의 객체가 reference되는 시간이 그렇게 길지 않기 때문에, unreferenced 객체들은 빠르게 수집되고, reference 되는 시간이 비교적 긴 객체들은 Old Generation으로 이동되게 된다.

Stop the World Event - **모든 Minor Garbage Collection은 "Stop the World" 이벤트**이다. 모든 애플리케이션 쓰레드는 Minor Garbage Collection 가 완료될 때 까지 멈추게 된다. Minor Garbage Collection 는 항상 Stop the World Event이다.

**Old Generation은 오래동안 referenced 되는 객체들을 저장**하는데 사용된다. 종종 Young Generation 의 임계값 (threshold)이 설정되고 해당 임계값을 넘는 객체들은 Old Generation으로 이동된다. 결국 Old Generation 역시 collected 되야 하는데, 이를 Major Garbage Collection이라고 한다.

**Major Garbage Collection 역시 Stop the World Event**이다. 종종 Major Garbage Collection이 더 느린데, 이것이 reference 되는 객체들을 포함하기 때문이다. 그래서 반응형 애플리케이션의 경우 Major Garbage Collection은 최소화 되어야 한다. 또한 Major Garbage Collection 의 Stop the World Event 시간은 Old Generation에 사용되는 Major Garbage Collector에 영향을 받는다.

**Permanent Generation은 JVM이 애플리케이션에서 사용되는 클래스와 메소드를 describe하기 위해 필요한 메타데이터까지 포함**하고 있다. Permanent Generation은 어플리케이션이 사용하는 클래스들에 기초해서 런**타임 시 JVM에 의해 채워진다**. 또한 Java SE 라이브러리 클래스 및 메소드를 여기에 저장할 수 있다.

JVM에서는 객체가 더 이상 필요하지 않고 다른 객체를 위해 공간이 필요할 수도 있는 경우에 해당 공간을 collect 한다. Permanent Generation은 Full Garbage Collection에 포함된다.

## The General Garbage Collection Process
1. 모든 새 객체가 eden 공간에 할당된다. 두 survivor 공간 모두 비어있다.
2. eden 공간이 가득 차면 Minor Garbage Collection이 trigger 된다.
3. referenced 객체는 first survivor space로 이동된다. unreferenced 객체들은 eden 공간이 지워지면 삭제된다.
4. 다음 Minor Garbage Collection에서 동일한 일이 eden 공간에서 실행된다. 하지만 이번에 referenced 객체들은 second survivor space로 이동된다. 또한 두 번째 Minor GC에서 살아남은 first survivor space에 있던 객체들은 second survivor space로 이동되며 age 값이 1 증가한다. 이렇게 되면 first survivor space와 eden space 모두 지워지게 되는데, second survivor space에 있는 객체들의 age는 전부 다르다.
5. 그 다음 Minor Garbage Collection에서는 마찬가지로 동일한 일이 일어나는데, 이번에는 first survivor space를 destination으로 해서 객체들이 옮겨가게 된다. 그러면 이번에는 second survivor space와 eden space가 지워지게 된다.
6. 이렇게 과정을 계속해서 반복하다가 threshold(임계값)을 넘는 나이를 가진 객체가 생긴다면, old generation으로 이동시킨다. 이러한 과정을 반복하면서 old generation이 점점 쌓이게 된다.
7. old generation에 객체가 일정 수준을 넘어서면, major garbage collection이 실행되면서 다시 공간을 압축하게 된다.


참조자료 : <br>
[Java Garbage Collection Basics_Oracle](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)
