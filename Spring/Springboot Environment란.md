# SpringBoot Environment
## 1. Environment 인터페이스란?
  
Github Action 및 CICD를 활용해서 Blue-Green 배포 전략을 짜던 중, 현재 프로젝트에 적용중인 `profiles` 들을 전부 조회해야 하는 필요가 있었다. 여러 예제 코드를 찾아보던 중, 대부분의 코드가 Environment 인터페이스로부터  `getActiveProfiles` 메소드를 사용해서 현재 작동중인 `profiles`를 조회하는 것을 확인할 수 있었다. Environment 인터페이스가 어떤 역할을 하는 것이고, 어떤 기능을 제공하는 것인지 등에 대해서 더욱 알고 싶어서 Spring Reference를 찾아보게 되었다.


Spring 공식 문서에는 아래와 같이 기술되어 있다.
> public interface **Environment** extends  [PropertyResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/PropertyResolver.html)  
> 
> Interface representing the environment in which the current application is running. 
> 
> Models two key aspects of the application environment: *profiles* and *properties*. 
> 
> Methods related to property access are exposed via the  [PropertyResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/PropertyResolver.html)  superinterface.
> 
> 출처 : [Environment (Spring Framework 5.3.23 API)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/Environment.html)

즉, `Environment`는 현재 작동중인 application (정확히는 WAS)의 정보를 나타내는 `Interface` 라고 해석할 수 있다. 해당 Interface를 그럼 어떻게 활용할 수 있을까? 

## 2. Environment Interface의 Methods


