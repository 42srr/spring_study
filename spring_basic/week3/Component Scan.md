

# Bean 자동 등록
---
Spring Container에 Bean을 등록하는 방법은 크게 두가지로 나뉜다.

- 수동 빈 등록
- 자동 빈 등록

수동으로 등록할 때 Config class 의 `@Bean` 또는 XML 파일의 `<bean>` 을 이용했다. 간단한 프로젝드에 등록되는 Bean 은 몇개 안되지만 프로젝트의 규모가 커질 수록 등록 해야할 Bean 의 수는 기하급수적으로 증가할 가능성이 높다.

이렇게 Bean 이 많아질 경우 일일이 등록하기 힘들고 설정 정보도 커질 뿐더러 심지어 누락하는 문제까지 발생할 수 있다.

따라서 Spring 은 설정 정보가 없어도 자동으로 Spring Bean 을 등록하는 **Component Scan** 이라는 기능을 제공하며 이에 맞추어 의존관계를 자동으로 주입하는 **@Autowired** 기능을 제공한다.


# Configuration
---
```java
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.context.annotation.FilterType;

@Configuration  
@ComponentScan(  
        excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Configuration.class)  
)  
public class AutoAppConfig {  

}
    
```

수동 등록할때와는 다르게 `@ComponentScan` 이라는 annotation을 사용해야 한다. 또한 Class 내부에 `@Bean` 으로 등록한 클래스가 하나도 없다.

> ❗️ 참고
> `@ComponentScan` 에 `excludeFilters` 를 설정해 주었는데 기존 수동 빈 등록 설정 클래스가 Bean 으로 등록되지 않게 하기 위함이다.
> 만약 해당 수동 빈 설정 클래스를 Bean 으로 등록시킬 경우 해당 클래스 내부에 있는 `@Bean` 이 붙어있는 메서드 들이 실행되어 빈이 중복등록된다. 


`Component Scan` 은 `@Component` 어노테이션이 붙은 클래스들을 스캔해서 Spring Bean 으로 등록한다. 따라서 Bean 으로 등록 시켜줄 클래스에 해당 어노테이션을 붙여야 한다.

```java
@Component  // Bean 으로 등록 시킬 구현체에 @Component 라고 달아줘야 함
public class MemberServiceImpl implements MemberService{ ... }

@Component  
public class MemoryMemberRepository implements MemberRepository{ ... }

...

```



# Autowired
---
```java
@Component  
public class OrderServiceImpl implements OrderService{  
    private final MemberRepository memberRepository;  
    private final DiscountPolicy discountPolicy;  
  
    @Autowired  
    public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {  
        this.memberRepository = memberRepository;  
        this.discountPolicy = discountPolicy;  
    }  
  
    @Override  
    public Order createOrder(Long memberId, String itemName, int itemPrice) {  
        Member member = memberRepository.findById(memberId);  
  
        int discountPrice = discountPolicy.discount(member, itemPrice);  
        return new Order(memberId, itemName, itemPrice, discountPrice);  
    }  
  
    public MemberRepository getMemberRepository() {  
        return memberRepository;  
    }  
}
```

빈을 수동으로 등록할 때 factory Bean 내부에서 의존 관계 설정을 직접 해주어야 했다. 하지만 component scan 을 할 경우 `@Component` 어노테이션이 달린 클래스내부에서 의존관계 주입을 해결해야 한다.

이때 사용하는 것이 `@Autowired` 어노테이션이다.

```java
	//Order ServiceImpl 생성자
    @Autowired  
public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {  

    this.memberRepository = memberRepository;  
    this.discountPolicy = discountPolicy;  
    
 }  
```

위 코드는 `OrderServiceImpl` 의 생성자이다. 생성자 주입을 받기 때문에 생성자 메서드 바로 위에 `@Autowired` 를 달아주면 의존관계가 자동 주입된다.

### simple test

```java
@Test  
void basicScan() {  
    //given  
    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AutoAppConfig.class);  
  
    //when  
    MemberService memberService = ac.getBean(MemberService.class);  
  
    //then  
    assertThat(memberService).isInstanceOf(MemberService.class);  
}
```

해당 문서 맨 처음 나온 AutoAppConfig 를 통해 Spring Container 를 생성했다. 이후 type 으로 빈을 조회하였을 경우 memberService 가 null 이 아닌 것을 확인할 수 있다.

# Component scan 과 Autowired 동작원리
---

### 1. Component Scan
![img1](./Pasted%20image%2020250122165057.png)

먼저 `@ComponentScan` 은 `@Component` 가 붙은 모든 클래스를 스프링 빈으로 등록한다. 이때 빈의 기본 이름은 클래스명을 사용하되 맨 앞글자는 소문자로 한다.

만약 빈의 이름을 지정하기 위해서는 다음과 같이 하면 된다.

```java
@Component("testName") 
public class TestComponent {}
```

### 2. @Autowired 의존관계 자동 주입

![img2](./Pasted%20image%2020250122165602.png)

생성자에 `@Autowired` 을 지정할 경우 스프링 컨테이너가 자동으로 해당 빈을 조회하여 주입한다.
이때 기본 조회 전략은 타입을 기준이다. 타입이 같은 빈을 찾아 주입한다.

> ❗️참고
> 생성자에 매개변수가 많아도 다 찾아서 자동으로 주입한다.


# Component Scan 위치
---

모든 자바 클래스를 다 `component scan` 할 경우 스캔 시간이 매우 오래 걸린다. 따라서 꼭 필요한 위치부터 탐색하도록 스캔 시작 위치를 지정할 수 있다.

```java
@ComponentScan(
	basePackages = "hello.core",
)
```

- `basePackages` : 탐색할 패키지의 시작 위치를 지정할 수 있다. 지정한 패키지를 포함한 하위 패키지를 모두 탐색하게 된다.

	`basePackages = {"hello.core", "hello.service"}` 이런식으로 여러 시작 위치를 지정할 수 있다.

- `basePackageClasses` : 지정한 클래스의 패키지를 탐색 시작 위치로 지정한다.
- 만약 지정하지 않을 경우 `@ComponentScan` 이 붙은 설정 정보 클래스의 패키지가 시작 위치가 된다.


> ❗️권장하는 방법
> 권장하는 방법은 패키지 위치를 직접 명시하지 않고 설정 정보 클래스의 위치를 프로젝트 최상단에 두는 것이다. Spring Boot 또한 이 방법을 기본으로 제공한다.


# Component Scan 대상
---

앞선 예제에서는 `@Component` 를 Class 에 붙여 컴포넌트 스캔의 대상이 되도록 하였다. 이 어노테이션 뿐만 아니라 다른 몇몇 어노테이션 또한 컴포넌트 스캔의 대상이 될 수 있다.

- `@Component` : Component Scan 에서 사용
- `@Controller` : Spring MVC Controller 에서 사용. 컨트롤러로 인식
- `@Service` : 스프링 비지니스 로직에서 사용. 핵심 비지니스 로직이 여기에 있다고 표시만 하는 용도.
- `@Repository` : 스프링 데이터 접근 계층에서 사용. 데이터 계층의 예외를 스프링 예외로 바꾸어 줌
- `@Configuration` : 스프링 설정 정보에서 사용. 스프링 빈이 싱글톤을 유지하도록 추가 처리를 함


# Filter
---

컴포넌트 스캔을 할 때 특정 대상을 스캔 대상에서 추가 또는 제외하고 싶을 수 있다. 이때 사용할 수 있는게 필터 옵션들이다.

커스텀 어노테이션을 생성해 보자.

```java
@Target(ElementType.TYPE) // type  -> class level 에 붙은 커스텀 어노테이션  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface MyExcludeComponent {  
}
```

```java
@Target(ElementType.TYPE) // type  -> class level 에 붙은 커스텀 어노테이션  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface MyIncludeComponent {  
}
```

`MyExcludeComponent` 는 컴포넌트 스캔에서 제외할 대상에 붙일 어노테이션이며 `MyIncludeComponent` 는 컴포넌트 스캔에 포함할 대상에 붙일 어노테이션이다.

위 어노테이션을 지정한 클래스들을 만들어 보자.

```java
@MyIncludeComponent  
public class BeanA {  
}
```

```java
@MyExcludeComponent  
public class BeanB {  
}
```

`BeanA` class 는 Component Scan 에 포함시키기 위해 `@MyIncludeComponent` 를 지정했고 `BeanB` class 는 Component Scan 에서 제외하기 위해 `@MyExcludeComponent` 를 지정했다.

```java
public class ComponentFilterAppConfigTest {  
  
    @Test  
    void filterScan() {  
        //given  
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(ComponentFilterAppConfig.class);  
  
        //when  
        BeanA beanA = ac.getBean(BeanA.class);  
  
        //then  
        assertThat(beanA).isNotNull();  
    }  
  
    @Test  
    void filterExcludeScan() {  
        //given  
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(ComponentFilterAppConfig.class);  
  
       assertThatThrownBy(() -> ac.getBean(BeanB.class))  
               .isInstanceOf(NoSuchBeanDefinitionException.class);  
    }  
  
  
  
    @Configuration  
    @ComponentScan(includeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = MyIncludeComponent.class),  
                    excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = MyExcludeComponent.class)  
    )  
    static class ComponentFilterAppConfig {  
  
    }}
```

예제 테스트 코드이다.

### ComponentScan Config class

```java
    @Configuration  
    @ComponentScan(includeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = MyIncludeComponent.class),  
                    excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = MyExcludeComponent.class)  
    )  
    static class ComponentFilterAppConfig {  
  
    }}
```

이번에도 자동 빈 등록을 위해 `@ComponentScan` 을 사용한다. 이때 Filter 를 통해 컴포넌트 스캔을 할 때 포함 시킬 항목과 포함시키지 않을 항목을 지정할 수 있다.

- `includeFilters` : 타입과 어노테이션의 정보를 바탕으로 해당 어노테이션이 달린 클래스를 빈으로 추가 지정한다.
- `excludeFilters` : 컴포넌트 스캔에서 제외할 대상을 지정할 수 있다.

### filterScan

```java
    @Test  
    void filterScan() {  
        //given  
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(ComponentFilterAppConfig.class);  
  
        //when  
        BeanA beanA = ac.getBean(BeanA.class);  
  
        //then  
        assertThat(beanA).isNotNull();  
    }  
```

when 절을 보면 BeanA 라는 타입으로 Bean 을 조회한다. BeanA 는 `@MyIncludeComponent` 를 가지고 있기 때문에 조회가 가능하다.


### filterExcludeScan

```java
 @Test  
    void filterExcludeScan() {  
        //given  
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(ComponentFilterAppConfig.class);  
  
       assertThatThrownBy(() -> ac.getBean(BeanB.class))  
               .isInstanceOf(NoSuchBeanDefinitionException.class);  
    }  
```

BeanB 는 `@MyExcludeComponent` 를 가지고 있기 때문에 컴포넌트 스캔 대상에서 제외 되었다. 따라서 해당 타입으로 Bean 을 조회할 경우 `NoSuchBeanDefinitionException` 이 발생하는 것을 확인할 수 있다.


# 중복 등록과 충돌
---

Component Scan 에서 같은 빈 이름을 등록하는 경우는 아래 두가지가 있다.

- 자동 - 자동
- 자동 - 수동

## 자동 - 자동

```java
@Component("service")  
public class OrderServiceImpl implements OrderService{...}

@Component("service")  
public class MemberServiceImpl implements MemberService{...}

```

두 빈을 동일한 이름으로 설정하고 실행할 경우 `ConflictingBeanDefinitionException` 이 발생한다. 


## 자동 - 수동

```java
@Configuration  
@ComponentScan(  
        excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Configuration.class)  
)  
public class AutoAppConfig {  
  
    @Bean  
    public MemberRepository memoryMemberRepository() {  
        return new MemoryMemberRepository();  
    }  
}
```

```java
@Component  
public class MemoryMemberRepository implements MemberRepository{  
  
    private static Map<Long, Member> store = new HashMap<>();  
  
    @Override  
    public void save(Member member) {  
        store.put(member.getId(), member);  
    }  
  
    @Override  
    public Member findById(Long memberId) {  
        return store.get(memberId);  
    }  
}
```

만약 `MemoryMemberRepository` 를 Component Scan 대상을 지정한 뒤 설정 클래스에서 수동으로 Bean 등록을 하면 어떻게 될까?

```text
***************************
APPLICATION FAILED TO START
***************************

Description:

The bean 'memoryMemberRepository', defined in class path resource [hello/core/AutoAppConfig.class], could not be registered. A bean with that name has already been defined in file [/Users/joojeon/Downloads/core/out/production/classes/hello/core/member/MemoryMemberRepository.class] and overriding is disabled.

Action:

Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true
```

애플리케이션 실행 자체가 되지 않는다. 

기존에는 수동으로 등록된 빈이 자동으로 등록된 빈을 오버라이딩 해버렸다. 하지만 이럴 경우 여러 설정들이 꼬여 예상치 못한 버그가 만들어진다. 

따라서 최근에는 Spring Boot 가 기본 옵션으로 자동 등록된 빈과 수동 등록된 빈이 중복될 경우 오류가 발생하도록 설정했다.

