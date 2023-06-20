# AOP (Aspect Oriented Programming)

## 1. AOP (Aspect Oriented Programming)
AOP는 Aspect Oriented Programming의 약자로, 여러 객체에 공통으로 적용될 수 있는 기능을 분리해서 재사용성ㅇ르 높여주는 프로그래밍 기법이다. AOP는 핵심 기능과 공통 기능의 구현을 분리함으로써 핵심 기능을 구현한 코드의 수정 없이 공통 기능을 적용할 수 있게 만들어준다.

스프링의 AOP에서는 핵심기능과 공통기능 분리를 Proxy를 사용해서 구현하고 있다. 기본 개념은 핵심 기능에 공통 기능을 삽입하는 것으로, 핵심 기능의 코드를 수정하지 않으면서 공통 구현 기능을 추가하는 것이 AOP이다. 이렇게 공통 기능을 삽입하는 방법은 총 3가지가 있다.
* 컴파일 시점에 코드에 공통 기능을 삽입하는 방법
* 클래스 로딩 시점에 바이트 코드에 공통 기능을 삽입하는 방법
* 런타임에 프록시 객체를 생성해서 공통 기능을 삽입하는 방법

첫 번째 방법과 두 번째 방법은 스프링 AOP에서는 지원하지 않으며 AspectJ와 같이 AOP 전용 도구를 사용해야 적용할 수 있다. 스프링의 AOP에서는 프록시 객체를 사용하는 세 번째 방법을 지원하고 있다. 중간에 프록시 객체를 생성하고, 실제 객체의 기능을 실행하기 전 후에 공통 기능을 호출하는 방식을 지원하고 있다.

<img src=https://user-images.githubusercontent.com/84431962/247010526-f3b4f5cf-8fde-4baa-9963-24845ac11a24.png>

스프링 AOP는 프록시 객체를 자동으로 만들어 주기 때문에 상위 타입의 인터페이스를 상속받은 프록시 객체를 직접 구현할 필요가 없이 공통 기능을 구현한 클래스만 알맞게 구현하면 된다.

AOP에서 공통 기능을 Aspect 라고 하는데 이 외에도 AOP의 핵심 용어들은 다음과 같다.

|용어|의미|
|---------|------------------|
|Advice|언제 공통 관심 기능을 핵심 로직에 적용할 것인지. 예를 들어서 메소드를 호출하기 전에 트랜잭션 기능을 적용한다는 것을 정의|
|JoinPoint|Advice를 적용 가능한 시점을 의미. 메소드 호출, 필드 값 변경 등이 JoinPoint에 해당한다. 스프링은 프록시를 이용해서 AOP를 구현하기 때문에 메소드 호출에 대한 JoinPoint만 지원한다.|
|PointCut|JoinPoint의 부분 집합으로서 실제 Advice가 적용되는 JoinPoint를 나타낸다. 스프링에서는 정규 표현식이나 AspectJ의 문법을 이용해서 PointCut을 정의할 수 있다.|
|Weaving|Advice를 핵심 로직 코드에 적용하는 것을 weaving 이라고 한다.|
|Aspect|여러 객체에 공통으로 적용되는 기능을 Aspect라고 한다. 트랜잭션이나 보안 등이 Aspect의 좋은 예이다.|

## 2. Advice의 종류
스프링은 언급한 바와 같이 프록시를 사용해서 메소드 호출 시점에 Aspect를 적용하기 때문에 구현 가능한 Advice의 종류는 다음과 같다.
|종류|설명|
|---------|------------------|
|Before Advice|대상 객체의 메소드 호출 전에 공통 기능 실행|
|After Returning Advice|대상 객체의 메소드가 익셉션 없이 실행된 이후에 공통 기능 실행|
|After Throwing Advice|대상 객체의 메소드를 실행하는 도중 익셉션이 발생한 경우에 공통 기능 실행|
|After Advice|익셉션 발생 여부에 상관 없이 대상 객체의 메소드 실행 후 공통 기능을 실행|
|Around Advice|대상 객체의 메소드 실행 전, 후 또는 익셉션 발생 시점에 공통 기능 실행|

이중 널리 사용되는 것은 Around Advice로 다양한 시점에 원하는 기능을 삽입할 수 있기 때문이다. 캐시 기능, 성능 모니터링 기능과 같은 Aspect를 구현할 때는 Around Advice를 주로 이용한다.

## 3. Spring AOP 구현
스프링 AOP를 이용해서 공통 기능을 구현하고 적용하는 방법은 단순하다.
> Aspect로 사용할 클래스에 @Aspect 어노테이션을 붙인다.
>
> @PointCut 어노테이션으로 공통 기능을 적용할 PointCut을 정의한다.
>
> 공통 기능을 구현한 메소드에 @Around 어노테이션을 적용한다.

바로 예시를 살펴보자.
```java
@Aspect
public class ExeTimeAspect {
    @Pointcut("execution(public * donghwan..*(..))")
    private void publicTarget() {
    }

    @Around("publicTarget()")
    public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();
        try {
            Object result = joinPoint.proceed();
            return result;
        } finally {
            long finish = System.nanoTime();
            Signature signature = joinPoint.getSignature();
            System.out.printf("%s.%s(%s) 실행 시간 : %d ns\n", joinPoint.getTarget().getClass().getSimpleName(), signature.getName(), Arrays.toString(joinPoint.getArgs()), finish - start);
        }
    }
}
```
먼저 ExeTimeAspect라는 클래스에는 @Aspect 어노테이션을 적용했다. 이렇게 적용한 클래스는 Advice와 Pointcut을 함께 제공한다.

@Pointcur은 공통 기능을 적용할 대상을 설정한다. @Pointcut 어노테이션의 값으로 사용할 수 있는 execution 명시자는 정규표현식 형태이다. 위의 public * donghwan..*(..) 형태의 정규표현식은 donghwan 패키지 하위에 있는 모든 public 메소드에 대해서 적용한다는 것을 의미한다.

@Around 어노테이션은 Around Advice를 설정한다. @Around 어노테이션의 값이 "publicTarget()" 인데 이는 publicTarget() 메소드에 정의한 Pointcut에 공통 기능을 적용한다는 것을 의미한다. 

measure 메소드의 ProceedingJoinPoint 타입 파라미터는 프록시 대상 객체의 메소드를 호출할때 사용한다. joinPoint.proceed()와 같은 형태로 핵심 기능을 가지고 있는 객체의 메소드를 실행하고, AOP에서 만들어주는 프록시 객체가 남은 부분을 실행한다고 이해하면 편하다.

또한 joinPoint.getTarget(), getArgs(), getSignature() 와 같은 메소드로 호출한 메소드의 시그니처(메소드 이름 + 전달인자), 대상, 객체, 인자 목록 등을 구하는 것 역시 가능하다.

```java
@Configuration
@EnableAspectJAutoProxy
public class AppCtx {

    @Bean
    public ExeTimeAspect exeTimeAspect() {
        return new ExeTimeAspect();
    }

    @Bean
    public Calculator calculator() {
        return new RecCalculator();
    }
}
```
방금 작성한 @Aspect 어노테이션이 붙은 객체를 공통 기능으로 적용하기 위해선 @EnableAspectJAutoProxy 어노테이션을 사용해서 스프링이 자동으로 @Aspect 어노테이션이 붙은 빈 객체를 찾아서 @Pointcut 설정과 @Around 설정을 사용할 수 있도록 해준다.

위의 @Aspect 객체를 다시 보면, @Around 어노테이션의 PointCut으로 publicTarget() 메소드를 설정했다. publicTarget() 메소드의 @PointCut은 donghwan 패키지 또는 그 하위 패키지에 속한 Bean 객체의 public 메소드를 설정한다. 따라서 이제 Calculator 빈에는 ExeTimeAspect 클래스에 정의한 공통 기능인 measure()가 적용될 것이다.

> 라고 생각하고 그대로 실행하였지만, 실제로 위와 같이 실행하게 되면 패키지 구조에 따라 다를 수도 있겠지만 BeanCurrentlyInCreationException이 발생하게 된다. @EnableAspectAutoProxy 어노테이션에 의해서 스프링은 @Aspect 어노테이션이 붙은 Bean 객체를 찾아서 @Pointcut 설정과 @Around 설정을 사용한다고 하였다. 그런데 다시 @Aspect 어노테이션이 붙은 ExeTimeAspect 클래스를 보면 @PointCut이 최상위 패키지로 적용되어 있으며 따라서 모든 Bean 객체의 public 메소드에 대해서 proceed 메소드를 실행하게 된다. 그런데, 위의 AppCtx 객체를 보면 ExeTimeAspect를 @Bean으로 등록하는 메소드를 볼 수 있다. 그럼 ExeTimeAspect 객체를 등록하는데 AOP를 다시 적용하는 꼴이 되버린다. 즉, 프록시를 위해 만들어진 객체에 프록시를 적용하는 꼴이 되버리는 것이다. 

이러한 상황을 해결할 수 있는 방법이 2가지가 있는데, 그 중 첫 번째는 @PointCut 에서 AppCtx Configuration 객체의 @Bean 설정을 위한 public 클래스를 제외하는 것이다. 그럼 ExeTimeAspect 클래스의 코드는 다음과 같이 변화되어야 한다.
```java
@Aspect
public class ExeTimeAspect {
    @Pointcut("execution(public * donghwan..*(..)) && !target(donghwan.sp5.chap07.config.AppCtx)")
    private void publicTarget() {
    }

    @Around("publicTarget()")
    public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();
        try {
            Object result = joinPoint.proceed();
            return result;
        } finally {
            long finish = System.nanoTime();
            Signature signature = joinPoint.getSignature();
            System.out.printf("%s.%s(%s) 실행 시간 : %d ns\n", joinPoint.getTarget().getClass().getSimpleName(), signature.getName(), Arrays.toString(joinPoint.getArgs()), finish - start);
        }
    }
}
```

두 번째 방법으로는 @Aspect 객체를 Bean 객체로 등록할 때 @Bean 어노테이션과 public 메소드를 사용하는게 아니라 @Component 어노테이션을 활용하는 것이다. 그리고 AppCtx Configuration 객체에서 @ComponentScan 어노테이션을 활용해서 (이때 탐색 범위는 적절히 잘 설정해야 한다. Configuration 객체를 기준으로 하위 패키지만 탐색하기 때문에, 이러한 부분에 있어서 다시금 문제가 생길 수 있다.) @Bean 객체로 등록하는 것이다. 이러한 방법을 사용하게 되면 코드는 다음과 같이 변화할 수 있다.
```java
@Aspect
@Component
public class ExeTimeAspect {
    @Pointcut("execution(public * donghwan..*(..)) && !target(donghwan.sp5.chap07.config.AppCtx)")
    private void publicTarget() {
    }

    @Around("publicTarget()")
    public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();
        try {
            Object result = joinPoint.proceed();
            return result;
        } finally {
            long finish = System.nanoTime();
            Signature signature = joinPoint.getSignature();
            System.out.printf("%s.%s(%s) 실행 시간 : %d ns\n", joinPoint.getTarget().getClass().getSimpleName(), signature.getName(), Arrays.toString(joinPoint.getArgs()), finish - start);
        }
    }
}

@Configuration
@EnableAspectJAutoProxy
@ComponentScan(basePackages = {"donghwan.sp5"})
public class AppCtx {

    @Bean
    public Calculator calculator() {
        return new RecCalculator();
    }
}
```

참조자료 : <br>
초보 웹 개발자를 위한 스프링5 프로그래밍 입문 (최범균 지음. 가메출판사) <br>