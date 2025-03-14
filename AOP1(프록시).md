## 초기 코드
- 아래 코드는 기본적인 인터페이스 기반 ControllerImpl을 작성한 것이다. Service, Repository도 동일한 형식으로 되어있다.
- 수동 Bean 등록을 사용한다.
```java
@RequestMapping
@ResponseBody
public interface ControllerInterface {

    @GetMapping("/test/func")
    String func();
}
```
```java
public class ControllerImpl implements ControllerInterface {

    private ServiceInterface service;

    public ControllerImpl(ServiceInterface serviceInterface) {
        this.service = serviceInterface;
    }

    @Override
    public String func() {
        return service.func();
    }
}
```
```java
@Configuration
public class TestConfig {

    @Bean
    public ControllerInterface testController1() {
        return new ControllerImpl(testService1());
    }

    @Bean
    public ServiceInterface testService1() {
        return new ServiceImpl(testRepository1());
    }

    @Bean
    public RepositoryInterface testRepository1() {
        return new RepositoryImpl();
    }
}
```

## 요구사항
- 위 코드에서 LogPrinter 라는 객체를 가지고 계층의 func 메소드에 로그를 남기고 싶다(@Sl4j 같은 어노테이션이 없다고 가정한다)

## 변경 1
```java
public class ControllerImpl implements ControllerInterface {

    private ServiceInterface service;
    private LogPrinter logPrinter;

    public ControllerImpl(ServiceInterface service, LogPrinter logPrinter) {
        this.logPrinter = logPrinter;
        this.service = service;
    }

    @Override
    public String func() {
        logPrinter.print("controller");
        return service.func();
    }
}


public class ServiceImpl implements ServiceInterface {

    private RepositoryInterface repository;
    private LogPrinter logPrinter;

    public ServiceImpl(RepositoryInterface repository, LogPrinter logPrinter) {
        this.logPrinter = logPrinter;
        this.repository = repository;
    }

    @Override
    public String func() {
        logPrinter.print("service");
        return repository.func();
    }
}
```

- 위와 같이 LogPrinter 객체를 선언해서 func 메소드에 직접 사용할 수 있다. 하지만 이는 클라이언트 코드 즉, 원본코드에 변경이 일어난다. 
- 원본코드에 변경이 일어나지 않고 logPrinter.print() 메서드를 계층마다 호출하고 싶다. 어떻게 해야할까?

## 프록시
- 보통의 경우엔 클라이언트가 서버에게 바로 요청하고 응답을 받는다.
![proxy1](https://user-images.githubusercontent.com/22884224/230013153-dc65020b-7512-4bb1-9ba0-297021df90f8.png)
- 하지만 클라이언트가 직접적으로 서버에 요청하는 것이 아니라 "프록시"라는 것을 통해 서버에 대신 요청을 보낼 수 있다. 프록시는 번역하면 대리자 이다.   
![proxy2](https://user-images.githubusercontent.com/22884224/230013501-dbd1a3c7-7bb3-4bf9-bb02-a24069239898.png)
- "JPA 프록시 객체"나 아니면 "프록시 서버" 같은 단어도 모두 이 "프록시"개념을 사용한것이다.

## 프록시 역할과 프록시 사용패턴
- 프록시는 여러가지 역할을 수행한다. 간단히 나열하자면 다음과 같다.
    1. 접근제어 (= 권한에 따른 접근 차단, 캐싱 등)
    2. 부가기능 추가 (= 로그, 요청 및 응답 변형 등) 
    3. 프록시체인
- 프록시를 사용하는 디자인패턴들은 다음과 같다. (프록시패턴만 프록시를 사용하는 것이 아니다. 데코레이터 패턴은 의도만 다른것이다)
    1. 프록시 패턴 
    2. 데코레이터 패턴

## 주의점
- 클라이언트는 서버에 요청하는지 프록시에게 요청하는지 몰라야한다. 또는 알 필요도 없다. 이렇게 하기 위해선 서버객체와 프록시객체는 같은 인터페이스를 사용해야한다. 즉,프록시는 아무 객체나 프록시가 될 수가 없다.    
 ![proxy3](https://user-images.githubusercontent.com/22884224/230015686-e56dcd91-c4ba-4405-bb9f-de74277a3bec.png)

## 변경2
- 변경1은 LogPrinter를 사용했지만 클라이언트코드의 직접 수정이 있었다. 이를 지양하기위해 "프록시"를 사용해서 해결할 수 있다. 다음은 Proxy객체 코드다.
- LogPrinter로 로그를 찍고, target은 ControllerImpl 객체가 주입되어서 기존 func()메소드를 호출할 것이다. 
```java
public class ControllerProxy implements ControllerInterface {

    private ControllerInterface target;
    private LogPrinter logPrinter;

    public ControllerProxy(ControllerInterface controllerInterface, LogPrinter logPrinter) {
        this.target = controllerInterface;
        this.logPrinter = logPrinter;
    }

    @Override
    public String func() {
        logPrinter.print("controller");
        return target.func();
    }
}
```
- 기존 Impl 코드는 초기코드와 같다.
```java

public class ControllerImpl implements ControllerInterface {

    private ServiceInterface service;

    public ControllerImpl(ServiceInterface service) {
        this.service = service;
    }

    @Override
    public String func() {
        return service.func();
    }
}
```
- Bean을 등록할떄 이제부터는 Impl객체가 아닌 Proxy객체를 생성한다. 결과적으로 Proxy가 로그를 찍고 Impl을 대신 호출해준다.
```java
@Configuration
public class TestConfig {

    @Bean
    public ControllerInterface testController1(LogPrinter logPrinter) {
        ControllerImpl testController = new ControllerImpl(testService1(logPrinter));
        ControllerProxy controllerProxy = new ControllerProxy(testController, logPrinter);
        return controllerProxy;
    }

    @Bean
    public ServiceInterface testService1(LogPrinter logPrinter) {
        ServiceImpl testService = new ServiceImpl(testRepository1(logPrinter));
        ServiceProxy serviceProxy = new ServiceProxy(testService, logPrinter);
        return serviceProxy;
    }

    @Bean
    public RepositoryInterface testRepository1(LogPrinter logPrinter) {
        RepositoryImpl testRepository = new RepositoryImpl();
        RepositoryProxy repositoryProxy = new RepositoryProxy(testRepository, logPrinter);
        return repositoryProxy;
    }

    @Bean
    public LogPrinter logPrinter() {
        return new LogPrinter();
    }
}
```
## 인터페이스 기반 프록시의 의존관계
- 밑에 사진은 위의 그림과 클래스 이름만 다르다.     
![proxy4](https://user-images.githubusercontent.com/22884224/230032451-dae80a0f-2b6a-4899-90a7-a62bbab67aec.png)
![proxy5](https://user-images.githubusercontent.com/22884224/230032651-ed2c0c6c-cd86-4ed5-a6ee-17790a9fd302.png)

## 구체클래스 기반은?
- 지금까지의 예는 인터페이스 기반의 프록시 생성이었다. 하지만 모든 코드가 인터페이스 기반으로만 동작하지는 않는다. 구체클래스만 있는 경우도 많다. 구체 클래스만 존재하는 경우엔 프록시 적용을 어떻게 할까?
- 간단하다. 구체클래스를 ***상속*** 받아서 프록시를 생성한다. 다른점은 상속이기때문에 부모클래스 생성자를 호출해야한다는 것이다.
``` java
public class ControllerProxyV2 extends ControllerImplV2 {

    private ControllerImplV2 target;
    private LogPrinter logPrinter;

    public ControllerProxyV2(ControllerImplV2 controllerImplV2, LogPrinter logPrinter) {
        super(null);
        this.target = controllerImplV2;
        this.logPrinter = logPrinter;
    }

    @Override
    public String func() {
        logPrinter.print("controller");
        return target.func();
    }
}
```
``` java
public class ServiceProxyV2 extends ServiceImplV2 {

    private ServiceImplV2 target;
    private LogPrinter logPrinter;

    public ServiceProxyV2(ServiceImplV2 target, LogPrinter logPrinter) {
        super(null);
        this.target = target;
        this.logPrinter = logPrinter;
    }

    @Override
    public String func() {
        logPrinter.print("service");
        return target.func();
    }
}
```
- Bean 설정도 크게 다르지 않다. 프록시에 이제는 인터페이스가 아닌 구체클래스를 바인딩하면 된다.
``` java
@Configuration
public class ConcreteConfig {

    @Bean
    public ControllerImplV2 controllerImplV2(LogPrinter logPrinter) {
        ControllerImplV2 controller = new ControllerImplV2(serviceImplV2(logPrinter));
        ControllerProxyV2 controllerProxyV2 = new ControllerProxyV2(controller, logPrinter);
        return controllerProxyV2;
    }


    @Bean
    public ServiceImplV2 serviceImplV2(LogPrinter logPrinter) {
        ServiceImplV2 service = new ServiceImplV2(repositoryImplV2(logPrinter));
        ServiceProxyV2 serviceProxyV2 = new ServiceProxyV2(service, logPrinter);
        return serviceProxyV2;
    }

    @Bean
    public RepositoryImplV2 repositoryImplV2(LogPrinter logPrinter) {
        RepositoryImplV2 repository = new RepositoryImplV2();
        RepositoryProxyV2 repositoryProxyV2 = new RepositoryProxyV2(repository, logPrinter);
        return repositoryProxyV2;
    }
}
```

## 구체클래스 기반 프록시의 의존관계
- 밑에 사진은 위의 그림과 클래스 이름만 다르다.     
![proxy4](https://user-images.githubusercontent.com/22884224/233787185-98ad746d-aba7-4851-98be-ce85cb83709e.png)

## 동적 프록시
- 기존코드를 변경하지 않고 프록시를 사용해서 LogPrinter로 로그를 남길 수 있게 됐다. 
- 하지만 LogPrinter를 사용하려는 클래스들이 100개면 Proxy 클래스를 100개를 만들어줘야한다. 심지어 위의 Proxy 클래스들은 모두 LogPrinter를 출력하는 로직도 같다. 이 작업을 줄일순  없을까? 
- ***동적 프록시***를 사용하면 프록시 클래스를 하나만 만들어서 모든곳에 적용할 수 있다.   
    - 인터페이스 기반 프록시 생성 -> ***JDK Dynamic Proxy***
    - 인터페이스 없이 구현체만 있는 경우 프록시 생성 -> ***CGLIB***

[출처] https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8/dashboard
