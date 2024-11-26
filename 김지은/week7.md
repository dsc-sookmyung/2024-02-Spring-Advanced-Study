# 7주차 - AOP2

- 노션 링크 : [7주차 - AOP2](https://www.notion.so/7-AOP2-1436873728d080f49579e9bed44361c3?pvs=21)

<aside>
💡

1. **부가기능이 타깃 오브젝트마다 새로 만들어지는 문제 : ProxyFactoryBean의 어드바이스로 해결**
2. 타깃 오브젝트마다 거의 비슷한 내용의 ProxyFactoryBean 빈 설정정보를 추가하는 부분 - 중복해결 필요
</aside>

### ProxyFactoryBean - Spring

프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈

- ProxyFactoryBean : 프록시를 생성하는 작업을 담당
- Advice : 프록시를 통해 제공해줄 부가기능은 별도의 빈에 담는다 : MethodInterceptor 인터페이스 구현체
- Pointcut : 부가기능 적용 대상 메소드 선정 방법

### 프록시 팩토리 빈

- 프록시를 생성하는 작업을 담당
- 인터페이스 자동 검출
    - 프록시가 구현해야 할 인터페이스 정보를 제공하지 않아도 됨
    - 타겟이 구현하고 있는 모든 인터페이스를 동일하게 구현하는 프록시를 자동으로 만들어준다
    - 일부만 적용하길 원한다면 직접 인터페이스 정보를 제공할 수도 있다

### 어드바이스 : 타깃이 필요없는 순수한 부가기능

- **타깃 오브젝트에 적용하는 부가기능**을 담은 오브젝트
    - 타깃 오브젝트에 상관없이 **독립적 : 타깃 정보를 갖고 있지 않다**
    - 따라서 스프링 **싱글톤 빈**으로 등록 가능
- **MethodInterceptor** **인터페이스** 구현체 - **invoke() 메소드에서 부가기능 수행, 콜백 호출**
- 부가기능을 구현하고 타깃을 호출하는 **invoke**의 파라미터로 타깃 오브젝트를 직접 받지 않는다
    - **파라미터 : MethodInvocation** : 메소드 정보와 함께 타깃 오브젝트를 알고 있다
    - **메소드 정보 + 타깃** 오브젝트가 담긴 일종의 **콜백** 오브젝트
    - 타깃 오브젝트의 메소드를 내부적으로 실행한다
- 팩토리 빈에 MethodInterceptor 설정 시, 수정자 메소드가 아닌 addAdvice() 메소드를 사용
    - **1개의 ProxyFactoryBean - 여러 개의 MethodInterceptor** 추가 가능
- 코드
    
    ```java
    //어드바이스 : MethodInterceptor 구현
        private static class UppercaseAdvice implements MethodInterceptor {
            @Override
            public Object invoke(MethodInvocation invocation) throws Throwable {
                // target을 전달할 필요가 없다
                // 대신 MethodInvocation으로 타깃의 메소드를 콜백
                String ret = (String)invocation.proceed();
                return ret.toUpperCase();
            }
        }
    ```
    

### 포인트컷 : 부가기능 적용 대상 메소드 선정 방법

- 어드바이스(MethodInterceptor)는 타깃과 독립적이며 싱글톤 빈으로 등록 가능한 개체
→ 재사용 가능한 순수 부가기능 코드만 남긴다
    - 따라서 부가기능 적용 메소드를 선택하는 부분은 어드바이스 내부에 넣지 않는다
- 부가기능 적용 메소드를 선별하는 교환 가능한 알고리즘이며, 이를 여러 프록시가 공유하여 사용할 수 있도록 프록시로부터도 분리해낸다

### 어드바이저 : 포인트컷 + 어드바이스

- 어드바이스와 포인트것을 함께 등록할 때 두 개의 오브젝트를 묶어서 등록하는 형식
- 어떤 어드바이스에 어떤 포인트컷을 적용할지 조합을 만들어서 지정한다
    - Advisor 타입으로 묶어서   addAdvisor() 메소드를 호출
- 코드
    
    ```java
    @Test
        public void pointcutAdvisor(){
            ProxyFactoryBean pfBean = new ProxyFactoryBean();
            pfBean.setTarget(new HelloTarget());
    
            NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
            pointcut.setMappedName("sayH*");
    
            pfBean.addAdvisor(new DefaultPointcutAdvisor(
                    pointcut, new UppercaseAdvice()));
    
            Hello proxiedHello = (Hello) pfBean.getObject();
    
            assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
            assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
            // sayThankYou는 포함 안됨
            assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
        }
    ```
    

### 다이내믹 프록시 - 어드바이스+포인트컷

![image.png](image.png)

<aside>
💡

**최종적인 순서**

1. 다이내믹 프록시 : **포인트 컷**에게 부가기능을 부여할 메소드인지 **확인하는 요청**
2. 대상 메소드 확인되면, **MethodInterceptor 타입의 어드바이스 호출** 
    1. **부가기능 수행** : MethodInterceptor에 구현한 부분
    2. **타깃의 메소드 호출** : 프록시로부터 전달받은 **Invocation 콜백** (proceed 호출)
</aside>

- 프록시로부터 **어드바이스, 포인트컷을 독립**시키고 **DI**를 사용한다
    - 여러 프록시에서 **공유**하여 사용 가능
    - 구현클래스만 변경하면 되기 때문에 자유로운 변경, 확장 가능
    
    → 특정 부가기능을 담은 어드바이스는 하나만 만들어서 싱글톤 빈으로 등록해두면 DI 설정을 통해 모든 서비스에 적용이 가능하다
    

- 다이내믹 프록시, 어드바이스, 포인트컷 적용한 최종적인 테스트 코드

```java
public class DynamicProxyTest {
    @Test
    public void simpleProxy(){
        // 다이내믹 프록시 생성
        Hello proxiedHello = (Hello) Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[] {Hello.class}, //타깃과 프록시가 구현할 인터페이스
                new UppercaseHandler(new HelloTarget()) //부가기능 : 어드바이스
        );
    }
    
    
    @Test
    public void pointcutAdvisor(){
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(new HelloTarget());

        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.setMappedName("sayH*");

        pfBean.addAdvisor(new DefaultPointcutAdvisor(
                pointcut, new UppercaseAdvice()));

        Hello proxiedHello = (Hello) pfBean.getObject();

        assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
        assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
        // sayThankYou는 포함 안됨
        assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
    }
    
    //어드바이스 : MethodInterceptor 구현
    private static class UppercaseAdvice implements MethodInterceptor {
        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            // target을 전달할 필요가 없다
            // 대신 MethodInvocation으로 타깃의 메소드를 콜백
            String ret = (String)invocation.proceed();
            return ret.toUpperCase();
        }
    }
    
    //타깃, 프록시가 공통으로 구현할 인터페이스
    static interface Hello {
	    String sayHello(String n);
	    String sayHi(String n);
	    String sayThankYou(String n);
    }
    
    //타깃 클래스
    static class HelloTarget implements Hello {
	    public String sayHello(String n){return "Hello"+n;}
	    public String sayHi(String n) {return "Hi"+n;}
	    public String sayThankYou(String n) {return "tx"+n;}
    }
}
```

---

<aside>
💡

1. 부가기능이 타깃 오브젝트마다 새로 만들어지는 문제 : ProxyFactoryBean의 어드바이스로 해결
2. **타깃 오브젝트마다 거의 비슷한 내용의 ProxyFactoryBean 빈 설정정보를 추가하는 부분 - 중복해결 필요 : 빈 후처리기로 해결**
</aside>

## 스프링 AOP

- 지금까지 다뤄온 반복적이고 기계적인 코드의 해결방법
    1. 바뀌는 / 바뀌지 않는 부분을 분리하여 템플릿, 콜백 / 클라이언트로 나누는 방법으로 해결
    2. **런타임 코드 자동생성 기법 : 다이내믹 프록시** 이용
        - **특정 인터페이스를 구현한 오브젝트(타겟)**에 대해서 **프록시 역할을 해주는 클래스를 런타임으로 생성**
            - **변하지 않는** 타깃으로의위임, 부가기능 적용 판단 : 다이내믹 프록시 기술로 자동 생성
            - **변화하는** 부가기능 코드 : 별도로 만들어서 다이내믹 프록시 생성 팩토리에 **DI로 주입**
    
    ⇒ 다이내믹 프록시가 구현클래스를 동적으로 생성하듯이
    
    **타깃 빈의 목록을 제공하면 → 자동으로 각 타깃 빈에 대한 프록시를 만들어주는 방법**으로 해결할 수 있다
    

### 빈 후처리기를 이용한 자동 프록시 생성기 : DefaultAdvisorAutoProxyCreator

- 빈 후처리기 : 스프링 빈 오브젝트가 생성된 후에 빈 오브젝트를 다시 가공할 수 있게 하는 확장 포인트
- DefaultAdvisorAutoProxyCreator : 어드바이저를 이용한 자동 프록시 생성기
    - 이용방법 : 빈 후처리기 자체를 빈으로 등록한다
    - 순서
        
        ![image.png](image%201.png)
        
        1. 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청한다
        2. 프록시 적용대상 여부 확인 : 빈으로 등록된 모든 어드바이저 내의 포인트컷(클래스 필터)을 이용해 전달받은 빈이 프록시 적용대상인지 확인한다
        3. 프록시 생성기 : 현재 빈에 대한 프록시를 생성한다
            1. 그리고 생성된 프록시를 어드바이저에 연결해준다 : 포인트컷(메소드매처) + 어드바이스
        4. 빈 후처리기 : 기존의 빈 오브젝트 대신, 생성된 프록시를 컨테이너에 반환한다
        5. 컨테이너 : 돌려받은 프록시를 빈으로 등록한다
            - 기존의 빈 오브젝트는 프록시로 대체된다 → 빈 오브젝트에는 프록시를 통해서만 접근이 가능하다
        

### 확장된 포인트 컷

<aside>
💡

포인트 컷의 두 가지 기능

1. 클래스 필터 : 프록시 적용할 클래스 여부 확인
2. 메소드 매처 : 어드바이스 적용할 메소드인지 확인

→ 두 가지 모두 충족되는 경우에 타깃의 메소드에 어드바이스가 적용

</aside>

### 클래스 필터를 적용한 포인트컷 작성

```java
public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
    public void setMappedClassName(String mappedClassName){
        this.setClassFilter(new SimpleClassFilter(mappedClassName));
    }

    private class SimpleClassFilter implements ClassFilter {
        String mappedName;

        public SimpleClassFilter(String mappedClassName) {
            this.mappedName = mappedClassName;
        }

        @Override
        public boolean matches(Class<?> clazz) {
            return PatternMatchUtils.simpleMatch(mappedName,
                    clazz.getSimpleName());
        }
    }
}
```

- NameMatchMethodPointcut를 상속받아 이름 패턴으로 클래스 이름을 비교하는 ClassFilter를 추가로 구현한다
- 기존의 NameMatchMethodPointcut의 디폴트 클래스 필터 : 모든 클래스를 다 허용
    
    ⇒ ClassFilter를 상속해 필터링 하는 클래스를 만들어서 덮어씌운다
    

---

### AOP란 무엇인가

- 트랜잭션 코드의 분리를 위해 AOP를 구현해나간 과정

 **< 트랜잭션 추상화 >** 

- 문제상황 : **트랜잭션 경계설정 코드를 비즈니스 코드 안에** 넣으면서 문제 발생
- 구체적인 구현을 담은 의존오브젝트는 **런타임 시에 동적으로 연결해주는 DI**를 활용한 방법으로 추상화

→ 비즈니스 로직 코드가 구체적인 트랜잭션 처리 방법, 서버 환경에 종속적이지 않게 독립됨

 **< 프록시와 데코레이터 패턴 >** 

- 문제상황 : 여전히 비즈니스 로직 안에 **트랜잭션을 적용하고 있다는 코드가 노출**된다
- 데코레이터 패턴 사용 : 트랜잭션을 처리하는 코드는 데코레이터에 담겨서 
**클라이언트 - 타깃클래스(비즈니스로직) 사이에 존재**하게 된다
- 클라이언트는 **프록시인 트랜잭션 데코레이터를 거쳐서 타깃에 접근**할 수 있게 된다
    
    → 비즈니스가 트랜잭션 코드로 부터 완전히 독립됐다
    

 **< 다이내믹 프록시 >** 

- 문제상황 : 비즈니스 로직 코드에서 트랜잭션을 제거하는 것은 성공했지만, 프록시를 사용하기 위해 타깃의 인터페이스의 모든 메소드마다 트랜잭션 코드를 넣어 **프록시 클래스를 작성하는 작업의 중복**이 발생함
- **다이내믹 프록시** 기술 적용 : 프록시 클래스를 직접 작성하지 않아도 **프록시 오브젝트를 런타임에 생성**해줌

 **< 프록시 팩토리 빈 >** 

- **다이내믹 프록시 생성 + DI** 도입, 내부적으로 템플릿/콜백패턴 활용
- **어드바이스, 포인트컷이 프록시에서 분리**됨
→ 어드바이스, 포인트컷을 **여러 프록시에서 공유**하여 사용할 수 있게 됐다

 **< 자동 프록시 생성방법, 포인트컷 >** 

- 문제상황 : 트랜잭션 적용 빈마다 **일일이 프록시 팩토리 빈을 설정**해줘야 한다
- **빈 후처리 기법**을 활용 → **컨테이너 초기화 시점**에 자동으로 패턴 검사로 **프록시를 생성**해주기
- **확장된 포인트컷** : 메소드뿐만 아니라, 빈후처리기가 사용할 수 있도록 **클래스 선정 기능을 담은 확장된 포인트컷**을 활용했다
    
    → **설정정보를 포인트컷이라는 독립적인 정보로 분리**하여 활용할 수 있게 됐다
    

 **< 부가기능의 모듈화 >** 

- **“부가기능”**의 특징
    - 애플리케이션 **전반에 흩어져 있다**
    - **중복**되어 나타난다
    - **핵심 기능(비즈니스로직)과 같은 레벨에 독립적으로 존재하는 것이 불가능**하다
- 부가기능을 모듈화 하기 위한 방법들
    - **다이내믹프록시**(클래스를 작성하지 않아도 구현기능을 가진 오브젝트를 동적으로 생성) / **빈 후처리 기술** (빈 초기화 시점에 생성작업을 가로채 빈 오브젝트를 프록시로 대체하는 작업) / **포인트컷**(적용될 클래스, 메소드 선정 기능)

## AOP : 애스펙트 지향 프로그래밍

- **애스팩트** : 핵심 기능(비즈니스 로직0을 담고 있지는 않지만, **핵심기능에 부가되어 의미를 갖는 특별한 모듈**
    - **어드바이스** : **부가될 기능**을 정의한 코드
    - **포인트컷** : **어디에 적용**할 지를 결정하는 로직
- 부가기능은 전반적인 코드에 **흩어지고, 중복되어 나타나**므로 기존의 DI만으로는 분리해내기 어렵다
    - 그러한 부가기능을 **독립적인 모듈인 애스팩트로 구분**해내면
    - 런타임 시에 부가기능 애스펙트는 **필요한 위치에 동적으로 참여**할 수 있다
- **애스펙트지향프로그래밍** : 핵심기능에서 부가기능을 분리하여 애스펙트라는 모듈로 만들고 설계하는 방법
    - AOP는 OOP를 대체하는 기술이 아닌, OOP를 돕는 보조적인 기술이다
    - 애스펙트를 분리함으로써 핵심기능 설계, 구현에서 객체지향적인 가치를 지킬 수 있도록 돕는다

---

- 포인트컷 표현식
    - 기존의 포인트컷 방식 : 클래스필터 / 메소드매처 두 가지 조건의 패턴을 프로퍼티로 넣어줘서 비교해서 선정하는 방식
    - 표현식을 사용해 포인트컷의 클래스, 메소드 선정 알고리즘을 한 번에 지정할 수 있는 정규표현식이 존재한다 : **AspectJExpressionPoincut**