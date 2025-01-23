# [Week 2] Spring Study

## 목차

1. [Spring Container](#spring-container)
2. [Spring Bean](#spring-bean)
3. [Configuration annotation](#configuration-annotation)

## Spring Container


### Spring Container 란?

Spring FrameWork 에서 Spring Container 또는 Spring IOC Container 란 빈을 인스턴스화, 구성 및 조립하는 역할을 한다. 
또한 생성된 빈들을 싱글톤 패턴으로 관리한다. 즉 메모리상 단 하나만 각 인스턴스들을 생성한다.
이처럼 싱글톤 객체를 생성하고 관리하는 기능을 `Singleton Registry` 라고 한다.


### Spring Container 생성

**ApplicationContext** Interface 를 Spring Container 라고 하며 여러 구현체를 통해 컨테이너를 생성할 수 있다.
일반적으로 어노테이션 기반 설정 클래스를 통해 스프링 컨테이너를 생성하는 `AnnotationConfigApplicationContext` 를 구현체로 사용한다.

- AppConfig.java
    ```java
  
  @Configuration
  public class AppConfig {
        @Bean
        public MemberService memberService() {
            System.out.println("call AppConfig.memberService");
            return new MemberServiceImpl(memberRepository());
        }

        @Bean
        public OrderService orderService() {
            System.out.println("call AppConfig.orderService");
            return new OrderServiceImpl(memberRepository(), discountPolicy());
        }

        @Bean
        public MemberRepository memberRepository() {
            System.out.println("call AppConfig.memberRepository");
            return new MemoryMemberRepository();
        }
  
        @Bean
        public DiscountPolicy discountPolicy() {
            return new RateDiscountPolicy();
        }
  }
  ```

위 Configuration class 를 통해 Spring Container 를 생성하는 방법은 다음과 같다.

- create spring container
  ```java
  ApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
  ```

### Spring Container 생성과정

**1. new AnnotationConfigApplicationContext(AppConfig.class)**

- 어노테이션 기반 설정 클래스를 사용하여 스프링 컨테이너를 생성하는 코드이다.
- ```java
  public AnnotationConfigApplicationContext(Class<?>... componentClasses) { 
    this();
    this.register(componentClasses);
    this.refresh();
  }
  ```
- 설정 Class 기반으로 Spring Container 를 생성할 때 위 생성자가 호출된다. 구성 클래스 정보를 매개변수로 받는 것을 확인할 수 있다.

**2. Spring Bean 등록**

- `@Bean` 어노테이션이 달린 메서드들을 호출하며 반환된 값을 Spring Container 내부에 Bean 으로 등록한다.
- 이때 Bean 의 이름은 메서드 이름이다.
- 빈의 이름은 절대 중복되어서는 안된다.
- `@Bean(name = "foo")` 처럼 이름을 직접 설정할 수 있다.

**3. Spring Bean 의존관계 설정 및 주입**

- 구성 정보 클래스를 기반으로 Dependency Injection 을 진행한다.
- 메서드가 두번 호출되어도 단 하나의 인스턴스만 빈으로 등록된다.


## Spring Bean

### Spring Bean 이란?

Spring Bean 은 Spring Container 가 생성하는 싱글톤 인스턴스이다. 
Container 내부에 인스턴스가 단 하나만 생성되며 여러 요청이 들어올 경우 빈이 계속 생성되는 것이 아닌 같은 인스턴스가 재활용된다.

### Bean 설계시 주의점 - 동시성 문제

Spring 기반 Web Application 은 thread 기반으로 작동한다.
즉 사용자의 요청 하나당 하나의 thread 가 할당되어 동작하게 된다.

만약 빈이 값을 저장하고 조회할 수 있는 필드가 존재할 경우 해당 필드는 모든 thread 가 공유하게 되어 `Race Condition` 이 발생하게된다.

이러한 상황을 피하기 위해 다음과 같은 선택을 할 수 있다.

- local variable 
- parameter
- thread local

하지만 가장 좋은 방법은 Bean 을 `Stateless` 하게 설계하는 것이다. 즉 특정 값을 저장하면 안된다.

### 단일 Bean 조회

```java
    // AbstractApplicationContext's getBean Method
    public Object getBean(String name) throws BeansException {
        this.assertBeanFactoryActive();
        return this.getBeanFactory().getBean(name);
    }

    public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
        this.assertBeanFactoryActive();
        return (T)this.getBeanFactory().getBean(name, requiredType);
    }

    public <T> T getBean(Class<T> requiredType) throws BeansException {
        this.assertBeanFactoryActive(); 
        return (T)this.getBeanFactory().getBean(requiredType);
    }
    ...
```

오버로딩된 메서드들을 보면 이름, 이름과 타입, 타입 등으로 Bean 을 조회할 수 있는 것을 확인할 수 있다.

```java
    @DisplayName("없는 이름으로 빈 조회")
    @Test
    void findBeanByNameX() {
        assertThatThrownBy(() -> ac.getBean("not exist bean name"))
                .isInstanceOf(NoSuchBeanDefinitionException.class);
    }
```

만약 없는 이름을 조회할 경우 `NoSuchBeanDefinitionException` 예외가 발생하는 것을 알 수 있다.

### 특정 타입 빈 조회

```java
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SameBeanConfig.class);
```

```java
    @Configuration
    static class SameBeanConfig {

        @Bean
        public MemberRepository memberRepository1() {
            return new MemoryMemberRepository();
        }

        @Bean
        public MemberRepository memberRepository2() {
            return new MemoryMemberRepository();
        }

    }
```

```java

    @DisplayName("특정 타입 모두 조회하기")
    @Test
    void findAllBeanByType() {

        Map<String, MemberRepository> beansOfType = ac.getBeansOfType(MemberRepository.class);
        
        assertThat(beansOfType).hasSize(2)
                .containsKey("memberRepository1")
                .containsKey("memberRepository2");
    }
```
특정 타입으로 조회할 경우 해당 타입의 모든 빈이 조회 된다.

### 같은 타입 둘 이상 단일 빈 조회 시도

```java
    @DisplayName("타입으로 조회시 같은 타입이 둘 이상있을 경우 중복 오류가 발생한다.")
    @Test
    void findBeanByTypeDuplicate() {

        assertThatThrownBy(() -> ac.getBean(MemberRepository.class))
                .isInstanceOf(NoUniqueBeanDefinitionException.class);

    }
```

동일한 타입의 빈이 두개 이상인 상황에서 타입으로만 단일 빈 조회를 할 경우 `NoUniqueBeanDefinitionException` 예외가 발생한다.

이런 상황일 경우 다음과 같이 이름으로 조회하면 된다.

```java
    @DisplayName("타입으로 조회시 같은 타입이 둘 이상 있을 경우 이름을 지정하면 된다.")
    @Test
    void findBeanByName() {

        MemberRepository bean = ac.getBean("memberRepository1", MemberRepository.class);

        assertThat(bean).isInstanceOf(MemberRepository.class);
        assertThat(bean).isInstanceOf(MemoryMemberRepository.class);

    }
```

### 상속관계 조회

부모 타입으로 조회시 모든 자식 빈들이 조회된다.

```java
    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(TestConfig.class);

    @Configuration
    static class TestConfig {

        @Bean
        public DiscountPolicy rateDiscountPolicy() {
            return new RateDiscountPolicy();
        }

        @Bean
        public DiscountPolicy fixDiscountPolicy() {
            return new FixDiscountPolicy();
        }

    }
```

```java
    @DisplayName("부모 타입으로 모두 조회")
    @Test
    void findAllBeanByParentType() {
        //given
        Map<String, DiscountPolicy> beansOfType = ac.getBeansOfType(DiscountPolicy.class);
        //when

        //then
        assertThat(beansOfType).hasSize(2)
                .containsKey("rateDiscountPolicy")
                .containsKey("fixDiscountPolicy");
    }
```

### Bean Definition

Bean Definition 을 Bean 설정 메타 정보 라고 부른다. `@Bean` 또는 `<Bean>` 빈 설정 메타 정보가 하나씩 생성된다.
Spring Container 는 Bean Definition 들을 보고 Container 에 Bean 을 생성하여 관리한다.

Bean Definition 에는 다음과 같은 정보들이 담겨 있다.

- `BeanClassName` : 생성할 Bean 의 class 명. 단 Factory 역할의 Bean 을 사용할 경우 없음
- `factoryBeanName` : 팩토리 역할의 빈을 사용할 경우 해당 클래스의 이름. (ex AppConfig)
- `factoryMethodName` : 빈을 생성할 펙터리 메서드 
- `Scope` : 싱글톤이 기본값
- `lazyInit` : 빈 생성 지연 여부
- `InitMethodName` : 빈을 생성하고 의존관계를 적용한 뒤 호출되는 초기화 메서드 명
- `DestroyMethodName` : 빈의 생명주기가 끝나 제거하기 직전 호출되는 메서드 명
- `Constructor arguments`, `Properties` : 의존 관계 주입에서 사용. 팩터리 역할의 빈을 사용할 경우 없음

사용자가 직접 등록한 빈 설정 메타정보를 출력할 경우 다음과 같은 결과가 나온다.

```text
beanDefinitionName = appConfig beanDefinition = Generic bean: class=hello.core.AppConfig$$SpringCGLIB$$0; scope=singleton; abstract=false; lazyInit=null; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; fallback=false; factoryBeanName=null; factoryMethodName=null; initMethodNames=null; destroyMethodNames=null
beanDefinitionName = memberService beanDefinition = Root bean: class=null; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; fallback=false; factoryBeanName=appConfig; factoryMethodName=memberService; initMethodNames=null; destroyMethodNames=[(inferred)]; defined in hello.core.AppConfig
beanDefinitionName = orderService beanDefinition = Root bean: class=null; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; fallback=false; factoryBeanName=appConfig; factoryMethodName=orderService; initMethodNames=null; destroyMethodNames=[(inferred)]; defined in hello.core.AppConfig
beanDefinitionName = memberRepository beanDefinition = Root bean: class=null; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; fallback=false; factoryBeanName=appConfig; factoryMethodName=memberRepository; initMethodNames=null; destroyMethodNames=[(inferred)]; defined in hello.core.AppConfig
beanDefinitionName = discountPolicy beanDefinition = Root bean: class=null; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; fallback=false; factoryBeanName=appConfig; factoryMethodName=discountPolicy; initMethodNames=null; destroyMethodNames=[(inferred)]; defined in hello.core.AppConfig
```

## Configuration Annotation

설정 정보를 통해 스프링 빈을 생성할 때 각 빈들이 Singleton 으로 유지되는 이유는 바로 `@Configuration` 때문이다.

빈을 싱글톤으로 생성하는 과정은 다음과 같다.

1. `@Bean` 이 붙은 메서드를 호출
2. 반환된 객체가 컨테이너 내부에 있는지 조회
3. 있을 경우 컨테이너 내부에 있는 해당 객체를 반환
4. 없을 경우 CGLIB 을 통해 해당 객체를 상속받은 객체를 Spring Container 에 등록 후 반환


만약 설정 클래스에 @Configuration 없이 @Bean 만 존재할 경우 CGLIB 을 통해 상속 받은 객체가 컨테이너에 등록되는 것이 아닌 반환하는 객체 그대로 Container 에 등록된다.
따라서 Singleton 으로 관리 되지 않고 호출 횟수만큼 Bean 이 생성 되는 일이 발생한다.

---
