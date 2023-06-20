# Proxy Pattern (프록시 패턴)

## 1. 프록시 패턴이란?
프록시 패턴이란 소프트웨어 디자인 패턴 중 하나로 프록시란 **"다른 무언가와 이어지는 인터페이스의 역할을 하는 클래스"** 이다. 일반적으로 다른 무언가와 이어지는 인터페이스의 역할을 하는 클래스로, 어떠한 것과도 인터페이스의 역할을 할 수 있다.

```java
import java.util.*;

interface Image {
    public void displayImage();
}

//on System A
class RealImage implements Image {
    private String filename;
    public RealImage(String filename) {
        this.filename = filename;
        loadImageFromDisk();
    }

    private void loadImageFromDisk() {
        System.out.println("Loading   " + filename);
    }

    @Override
    public void displayImage() {
        System.out.println("Displaying " + filename);
    }
}

//on System B
class ProxyImage implements Image {
    private String filename;
    private Image image;

    public ProxyImage(String filename) {
        this.filename = filename;
    }

    @Override
    public void displayImage() {
        if (image == null)
           image = new RealImage(filename);

        image.displayImage();
    }
}

class ProxyExample {
    public static void main(String[] args) {
        Image image1 = new ProxyImage("HiRes_10MB_Photo1");
        Image image2 = new ProxyImage("HiRes_10MB_Photo2");

        image1.displayImage(); // loading necessary
        image2.displayImage(); // loading necessary
    }
}
```
위 예시 코드의 main 함수에서는 2개의 ProxyImage 객체를 생성한다. ProxyImage 객체의 displayImage 메소드를 보면 해당 객체의 Image 필드가 널이라면 RealImage를 생성해서 주입하는 것을 볼 수 있는데, 이처럼 **다른 무언가와 이어지는 인터페이스의 역할을 하는 객체를 보고 Proxy 객체**라고 한다.

## 2. Spring에서의 Proxy Pattern
Spring의 가장 큰 장점 중 하나인 AOP (Aspect Oriented Programming)의 시초가 Proxy 라고 해도 과언이 아니다. 바로 아래 예제를 살펴보자.

```java
public interface Calculator {
    public long factorial (long num);
}

public class ImpeCalculator implements Calculator {
    @Override
    public long factorial(long num) {
        long result = 1;
        for (long i = 1; i <= num; i++) {
            result *= i;
        }
        return result;
    }
}

@AllArgsConstructor
public class ExeTimeCalculator implements Calculator{

    private Calculator delegate;

    @Override
    public long factorial(long num) {
        long start = System.nanoTime();
        long result = delegate.factorial(num);
        long end = System.nanoTime();
        System.out.printf("%s.factorial(%d) 실행 시간 = '%d'\n", delegate.getClass().getSimpleName(), num, (end-start));
        return result;
    }
}

public class Sp5Application {
	private static ApplicationContext ctx = null;

	public static void main(String[] args) {
		ImpeCalculator impeCalculator = new ImpeCalculator();
		ExeTimeCalculator calculator = new ExeTimeCalculator(impeCalculator);
		long result = calculator.factorial(4);
	}
}

// [실행 결과]
// ImpeCalculator.factorial(4) 실행 시간 = '1250'
```
차례대로 Calculator 인터페이스. ImpeCalculator 클래스, ExeTimeCalculator 클래스, 그리고 메인 클래스이다. 메인 클래스에서 calculator 라는 객체는 ExeTimeCalculator 클래스인데, 이때 생성자에서 impeCalculator를 전달인자로 사용했다. 그리고 calculator 객체의 factorial 메소드를 사용하였다.

그런데 ExeTimeCalculator 클래스의 Implementation을 보면 factorial 값을 계산하는 로직은 존재하지 않는다. 계산기의 핵심 기능은 팩토리얼 계산은 delegate라는 Calculator 클래스에게 위임하고, 해당 클래스는 메소드의 시작과 종료 시간을 측정한 뒤 출력하는 기능만 제공하고 있다.

그럼 다시 메인 코드에서 생성자에 주입했던 ImpeCalculator 클래스는 어떻게 구성되어 있을까? ImpeCalculator 객체의 factorial 메소드는 이번에는 오로지 팩토리얼 값을 계산만 하고 있는 모습을 볼 수 있다.

요약해서, 우리는 ExeTimeCalculator 객체를 생성했고 객체 생성 시에 생성자로 ImpeCalculator 객체를 전달했다. ExeTimeCalculator 객체는 오로지 메소드 실행 시간 측정만 제공하고, ImpeCalculator 객체는 오로지 팩토리얼 값 계산만 제공한다. 그리고 우리는 한 객체를 선언할 때 다른 객체로 이어지도록 하는 "프록시 패턴 (Proxy Pattern)" 을 활용해서 factorial 값의 계산과 메소드 실행 시간까지 측정할 수 있었다.

여기서 핵심은, **"팩토리얼 값의 계산" 과 "메소드 실행 시간 측정 및 출력" 기능을 서로 분리**해서 서로 다른 클래스에 정의했다는 점이다. 그리고 우리는 두 클래스를 자연스럽게 연결해서 사용하였다. 

**프록시의 특징은 핵심 기능은 구현하지 않는다**는 점이다. 대신 **여러 객체에 공통으로 적용할 수 있는 기능을 구현**한다. 우리는 이렇게 여러 객체에 공통으로 적용할 수 있는 기능을 보고 "공통 관심사" 라고 말한다.

## 3. AOP
사실 위에서의 예제는 "프록시 패턴" 보다는 "데코레이터 패턴" 에 더 가깝다. 프록시 패턴은 "접근 제어 관점" 에 초점이 맞춰저 있다면, 데코레이터 패턴은 "기능 추가와 확장" 에 초점이 맞춰져 있다. 사실 이보다 중요한 것은 왜 이렇게 사용하는 것이냐 하는 건데, 여기서 팩토리얼 값의 계산은 **"핵심 관심사"** 에 해당할 것이고, 메소드 시간 측정은 **"공통 관심사"** 에 해당할 것이다. 이렇게 핵심 관심사와 공통 관심사를 분리해서 개발한다면 비즈니스 코드 개발에 집중할 수 있을 뿐만 아니라 유지보수에 있어서도 큰 장점을 가진다. 그리고 이러한 점에 의거해서 Spring에서 출발한, 그리고 가장 핵심 기능 중 하나가 바로 AOP이다.

참조자료 : <br>
초보 웹 개발자를 위한 스프링5 프로그래밍 입문 (최범균 지음. 가메출판사) <br>
[Proxy Pattern](https://ko.wikipedia.org/wiki/%ED%94%84%EB%A1%9D%EC%8B%9C_%ED%8C%A8%ED%84%B4)