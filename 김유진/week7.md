# 6장. AOP

## 6.4 스프링의 프록시 팩토리 빈

✨ 이제 스프링이 이러한 문제에 어떤 해결책을 제시하는지 살펴보자!

### 6.4.1 ProxyFactoryBean

✨ 스프링은 서비스 추상화를 프록시 기술에도 동일하게 적용하고 있다.

→ 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어를 제공한다.

- `ProxyFactoryBean`
    - 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈
    - 순수하게 프록시를 생성하는 작업만 담당, 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있음.
        - MethodInterceptor  인터페이스를 구현해서 만듦.
            - `ProxyFactoryBean` 으로부터 타깃 오브젝트에 대한 정보까지도 함께 제공받음.
            - 덕분에 타깃 오브젝트에 상관없이 독립적으로 만들어질 수 있음.
            - 따라서, 타깃이 다른 여러 프록시에서 함께 사용할 수 있고, 싱글톤 빈으로 등록 가능
    - 스프링 `ProxyFactoryBean` 을 이용한 다이내믹 프록시 테스트
        
        ```java
        public class DynamicProxyTest {
            @Test
            public void simpleProxy(){
                // JDK 다이내믹 프록시 생성
                Hello proxiedHello = (Hello) Proxy.newProxyInstance(
                        getClass().getClassLoader(),
                        new Class[] {Hello.class}, 
                        new UppercaseHandler(new HelloTarget()) 
                );
            }
        
            @Test
            public void proxyFactoryBean(){
                ProxyFactoryBean pfBean = new ProxyFactoryBean();
                pfBean.setTarget(new HelloTarget()); // 타깃 설정
                pfBean.addAdvice(new UppercaseAdvice()); // 부가기능. 여러개 가능
        
                //FactoryBean이므로 getObject()로 생성된 프록시를 가져옴
                Hello proxiedHello = (Hello) pfBean.getObject();
        
                assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
                assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
                assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
            }
        
            private static class UppercaseAdvice implements MethodInterceptor {
                @Override
                public Object invoke(MethodInvocation invocation) throws Throwable {
                    //InvocationHandler와 달리 target이 필요 없음
                    String ret = (String)invocation.proceed();
                    return ret.toUpperCase();
                }
            }
        }
        ```
        

#### 어드바이스: 타깃이 필요 없는 순수한 부가기능

🤔 ProxyFactoryBean을 적용한 코드를 기존의 JDK 다이내믹 프록시를 사용했던 코드와 비교해보면 **몇 가지 차이점**이 있다!

1️⃣ 타깃 오브젝트가 필요 없다.

- JDK 다이내믹 프록시에서는 타깃 오브젝트가 반드시 필요하지만, ProxyFactoryBean에서는 **MethodInterceptor**를 통해 타깃과 독립적으로 부가기능을 구현할 수 있음.
- MethodInterceptor는 타깃을 직접 참조하지 않고, **MethodInvocation** 객체를 통해 타깃의 메소드를 호출.

2️⃣ MethodInvocation: 콜백 템플릿 역할

- `proceed()` 메소드를 호출하면 내부적으로 타깃 오브젝트의 메소드가 실행됨.
- 공유 가능한 템플릿 구조로, 동일한 MethodInvocation 객체를 여러 부가기능에서 재사용 가능.

3️⃣ 부가기능을 쉽게 추가

- ProxyFactoryBean은 `addAdvice()` 메소드를 통해 여러 개의 **Advice (MethodInterceptor)**를 추가할 수 있음.
- 이로 인해, 새로운 부가기능을 추가할 때마다 별도의 프록시나 팩토리 빈을 만들 필요가 없음.
    - 📌 `addAdvice()`
        
        MethodInterceptor는 Advice 인터페이스를 상속하고 있는 서브 인터페이스이기 때문
        
        어드바이스?
        
        - 타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트

4️⃣ 인터페이스 자동 검출

- 기존 JDK 다이내믹 프록시는 반드시 타깃 오브젝트가 구현하는 인터페이스를 명시적으로 제공해야 했음.
- ProxyFactoryBean은 타깃 오브젝트가 구현하고 있는 **인터페이스를 자동으로 검출**하여 프록시를 생성.
- 필요 시 `setInterfaces()` 메소드를 통해 특정 인터페이스만 지정할 수도 있음.

#### 포인트컷: 부가기능 적용 대상 메소드 선정 방

📌 기존에 InvocationHandler를 직접 구현했을 때는 부가기능 적용 외에도 한 가지 작업이 더 있었다.

→ 메소드의 이름을 가지고 부가기능을 적용 대상 메소드를 선정하는 것

🤔 그렇다면 스프링의 ProxyFactoryBean과 MethodInterceptor를 사용하는 방식에서도 메소드 선정 기능을 넣을 수 있을까?

- 실제로는 불가능하다.
- 스프링의 싱글톤 빈으로 등록한 MethodInterceptor에 트랜잭션 적용 대상 메소드 이름 패턴을 넣어주는 것은 곤란하다.
    - 트랜잭션 적용 패턴은 프록시마다 다를 수 있기 때문

🤔 그렇다면 이 문제는 어떻게 해결할 수 있을까?

→ 코드 개선 전략을 적용해보자.

- MethodInterceptor에는 재사용 가능한 순수한 부가기능 제공 코드만 남겨준다.
- 대신 프록시에 부가기능 적용 메소드를 선택하는 기능을 넣는다.
- 메소드를 선별하는 기능은 프록시로부터 다시 분리하는 편이 낫다.

📌 **기존 JDK 다이내믹 프록시를 이용한 방식과의 차이점**

1️⃣  기존 JDK 다이내믹 프록시를 이용한 방식

![image](https://github.com/user-attachments/assets/ed03574b-eabb-4804-bc85-04c6e590d935)



- InvocationHandler가 부가기능과 메서드 선정 알고리즘을 DI받아 target에 실행을 위임한다.
    
    → 부가기능을 가진 InvocationHandler가 target과 메서드 선정 알고리즘 코드에 의존함.
    
    → target이 다르거나 메서드 선정 방식이 다르면 해당 InvocationHandler는 여러 프록시에서 공유가 불가능함.
    
    → 그래서 부가기능이 target obj마다 새로 만들어지는 문제가 있다
    

2️⃣ 스프링의 ProxyFactoryBean을 이용한 방식

![image (1)](https://github.com/user-attachments/assets/511fc536-0efb-4dee-9f3b-031fe075aa3e)


1. 프록시는 클라이언트로부터 요청을 받으면 포인트컷에 부가기능을 부여할 메서드인지를 확인해달라고 요청한다.
2.  포인트컷으로부터 부가기능을 적용할 대상 메서드인지 확인받으면 MethodInterceptor 타입의 어드바이스를 호출한다.
3. 어드바이스에서 target 메서드의 호출이 필요하면 proceed()메서드를 호출한다.
4. proceed()메서드 수행하는 부분은 target obj에 위임되어 실행된다.

- 부가기능을 제공하는 오브젝트를 **어드바이스**라고 하고, 메서드 선정 알고리즘을 담은 오브젝트는 **포인트컷** 이라 한다.
    
    → 어드바이스와 포인트컷은 프록시에 DI로 주입되어 사용
    
    → 어드바이스는 target에 직접 의존하지 않도록 템플릿 구조로 설계되어 있음
    
    - target 정보 상태값을 가지고 있지 않다
    
    → 어드바이스나 포인트컷을 싱글톤 빈으로 만들어 둘 수 있어 재사용이 가능하다.
    

- 📌 **어드바이저**
    - 어드바이스와 포인트컷을 묶은 오브젝트
    - 어드바이저 = 포인트컷(메소드 선정 알고리즘) + 어드바이스(부가기능)

### 6.4.2 ProxyFactoryBean 적용

#### TransactionAdvice

- 부가기능을 담당하는 어드바이스는 MethodInterceptor를 구현해서 만든다
- TransactionHandler의 코드에서 타깃과 메소드 선정 부분을 제거해주면 된다.
    
    ```java
    // 스프링의 어드바이스 인터페이스 구현
    public class TransactionAdvice implements MethodInterceptor {
        private PlatformTransactionManager transactionManager;
    
        public void setTransactionManager(
                PlatformTransactionManager transactionManager){
            this.transactionManager = transactionManager;
        }
    
        @Override
        // 타깃을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받는다.
        // 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용 가능하다.
        public Object invoke(MethodInvocation invocation) throws Throwable {
            TransactionStatus status =
                    this.transactionManager.getTransaction(
                            new DefaultTransactionDefinition());
            try{
                Object ret = invocation.proceed();
                this.transactionManager.commit(status);
                return ret;
            }catch (RuntimeException e){
                this.transactionManager.rollback(status);
                throw e;
            }
        }
    }
    ```
    

#### 스프링의 XML 설정파일

- 학습 테스트에 직접 DI해서 사용했던 코드를 XML 설정으로 바꿔주기만 하면 된다.
- 트랜잭션 어드바이스 빈 설정
    
    ```java
    <bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
        <property name="transactionManager" ref="transactionManager"/>
    </bean>
    ```
    

- 포인트컷 빈 설정
    
    ```java
    <bean id="transactionPointcut"
              class="org.springframework.aop.support.NameMatchMethodPointcut">
        <property name="mappedName" value="upgrade*"/>
    </bean>
    ```
    

- 어드바이저 빈 설정
    
    ```java
    <bean id="transactionAdvisor"
              class="org.springframework.aop.support.DefaultPointcutAdvisor">
        <property name="advice" ref="transactionAdvice"/>
        <property name="pointcut" ref="transactionPointcut"/>
    </bean>
    ```
    

- ProxyFactoryBean 설정
    
    ```java
    <bean id="userService"
              class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="target" ref="userServiceImpl"/>
        <property name="interceptorNames">
            <list>
                <value>transactionAdvisor</value>
            </list>
        </property>
    </bean>
    ```
    

#### 테스트

💡 테스트 코드도 정리하자.

- ProxyFactoryBean을 이용한 트랜잭션 테스트
    
    ```java
    public class UserServiceTest {
        @Test
        @DirtiesContext // 컨텍스트 설정 변경하기 때문에 여전히 필요
        public void upgradeAllOrNothing() {
            UserServiceImpl.TestUserService testUserService =
                    new UserServiceImpl.TestUserService(users.get(3).getId());
            testUserService.setUserDao(this.userDao);
            testUserService.setMailSender(this.mailSender);
    
            ProxyFactoryBean txProxyFactoryBean =
                    context.getBean("&userService", ProxyFactoryBean.class);
            txProxyFactoryBean.setTarget(testUserService);
            UserService txUserService = (UserService) txProxyFactoryBean.getObject();
    
            userDao.deleteAll();
            for (User user : users) userDao.add(user);
    
            try {
                txUserService.upgradeLevels();
                fail("TestUserServiceException expected");
            } catch (UserServiceImpl.TestUserServiceException e) {
            }
    
            checkLevelUpgraded(users.get(0), false);
        }
    }
    ```
    

#### 어드바이스와 포인트컷의 재사용

- ProxyFactoryBean은 DI, 템플릿/콜백, 서비스 추상화 등의 기법이 적용됨
    
     → 덕분에 여러 프록시가 공유할 수 있는 어드바이스와 포인트컷으로 확장 기능을 분리할 수 있었음.
    
- UserService 이외에 새로운 비즈니스 로직을 담은 서비스 클래스가 만들어져도 TransactionAdvice를 그대로 재사용 할 수 있음
- TransactionAdvice는 하나만 만들어서 싱글톤 빈으로 등록하면, DI 설정을 통해 모든 서비스에 적용 가능
    
    ![image (2)](https://github.com/user-attachments/assets/3780fab5-2dc5-44de-a92e-ad1f6f26aa64)


## 6.5 스프링 AOP

✨ 지금까지 해왔던 작업의 목표는 비즈니스 로직에 반복적으로 등장해야만 했던 트랜잭션 코드를 깔끔하고 효과적으로 분리하는 것이었다.

→ 이렇게 분리해낸 트랜잭션 코드는 부가기능을 적용한 후에도 기존 설계와 코드에는 영향을 주지 않는다.

### 6.5.1 자동 프록시 생성

🤔 투명한 부가기능을 적용하는 과정에서 발견됐던 거의 대부분의 문제는 제거됐지만 아직 한 가지 해결할 과제가 남아있다.

- 부가기능의 적용이 필요한 타깃 오브젝트마다 거의 비슷한 내용의 ProxyFactoryBean 빈 설정 정보를 추가해주는 부분
    - 새로운 타깃이 등장할때마다 설정을 매번 복사해서 붙이고 target 프로퍼티의 내용을 수정해야 한다.
    - 이런 류의 중복은 더 이상 제거할 수 없는 것일까?

#### 중복 문제의 접근 방법

✨ 지금까지 다뤄봤던 반복적이고 기계적인 코드에 대한 해결책을 생각해보자.

1. 전략 패턴과 DI를 적용해서 해결
2. 런타임 코드 자동생성 기법(다이내믹 프록시)을 적용
    - 의미 있는 부가기능 로직은 코드로 만들게 하고
    - 기계적인 코드는 자동 생성하게 한 것\

→ 반복적인 ProxyFactoryBean 설정 문제는 설정 자동등록 기법으로 해결할 수 없을까?

- 지금까지 살펴본 방법에서 한 번에 여러 개의 빈에 프록시를 적용할 만한 방법은 없었다.

#### 빈 후처리기를 이용한 자동 프록시 생성기

✨ 스프링은 **유연한 확장**을 스프링 컨테이너 자신에게도 다양한 방법으로 적용한다.

→ 그중 관심을 가질 만한 포인트: BeanPostProcessor 인터페이스를 구현해서 만드는 빈 후처리기

- 빈 후처리기?
    - 스프링 빈 오브젝트로 만들어지고 난 후에, 빈 오브젝트를 다시 가공할 수 있게 해줌.

- DefaultAdvisorAutoProxyCreator
    - 스프링이 제공하는 빈 후처리기
    - 어드바이저를 이용한 자동 프록시 생성기

- 빈 후처리기를 스프링에 적용하는 방법
    1. 빈 후처리기 자체를 빈으로 등록
    2. 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청
    
    → 빈 후처리기는 스프링 설정을 참고해서 만든 오브젝트가 아닌 다른 오브젝트를 빈으로 등록시키는 것이 가능
    
    → 이를 잘 이용하면, 스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고, 프록시를 빈으로 대신 등록할 수도 있다.
    
    → 이것이 바로, 자동 프록시 생성 빈 후처리기이다!
    
- 빈 후처리기를 이용한 자동 프록시 생성 방법을 설명하는 그림
    
    ![image](https://github.com/user-attachments/assets/8272c2c9-6d47-498d-907b-8b050d19ec7a)

    
    - DefaultAdvisorAutoProxyCreator 빈 후처리기가 등록되어 있으면 스프링은 빈 오브젝트를 만들 때마다 후처리기에게 빈을 보냄.
    - DefaultAdvisorAutoProxyCreator는 빈으로 등록된 모든 어드바이저 내의 포인트컷을 이용해 전달받은 빈이 프록시 적용 대상인지 확인
    - 프록시 적용 대상인 경우 → 내장된 프록시 생성기에 현재 빈에 대한 프록시를 생성하게 함.
    - 프록시가 생성되면 빈 후처리기는 원래 컨테이너가 전달해준 빈 오브젝트 대신 프록시 오브젝트를 컨테이너에 돌려줌.
    - 컨테이너는 빈 후처리기가 돌려준 오브젝트를 빈으로 등록하고 사용

⇒ 마지막 남은 번거로운 ProxyFactoryBean 설정 문제를 말끔하게 해결함.

#### 확장된 포인트컷

🤔 지금까지 포인트컷이란 어떤 메소드에 부가기능을 적용할지를 선정해주는 역할을 한다고 했는데 갑자기 포인트컷이 등록된 빈 중에서 어떤 빈에 프록시를 적용할지를 선택한다는 식으로 설명한다. 어떻게 된 일일까?

- 사실 포인트 컷은 두 가지 기능을 모두 가지고 있다.
- 실제 포인트 컷의 선별 로직은 클래스 필터와 메소드 매처 두 가지 타입의 오브젝트에 담겨 있다.
- 지금까지는 포인트컷이 제공하는 두 가지 기능 중에서 MethodMatcher라는 메소드를 선별하는 기능만 사용한 것이다.
- 모든 빈에 대해 프록시 작용 적용 대상을 선별해야 하는 DefaultAdvisorAutoProxyCreator는 클래스와 메소드 선정 알고리즘을 모두 가지고 있는 포인트 컷이 필요

#### 포인트컷 테스트

✨ 포인트컷의 기능을 간단한 학습 테스트로 확인해보자.

- Hello 인터페이스와 이를 구현한 HelloTarget, 부가기능인 HelloAdvice를 사용했던 DynamicProxyTest에 테스트 메소드를 추가한다.
    
    ```java
    public class DynamicProxyTest {
        ...
        @Test
        public void classNamePointcutAdvisor(){
    		    // 포인트컷 준비
            NameMatchMethodPointcut classMethodPointcut = new NameMatchMethodPointcut(){
                @Override
                public ClassFilter getClassFilter() { // 익명 내부 클래스 방식으로 클래서 정의
                    return new ClassFilter() {
                        @Override
                        public boolean matches(Class<?> clazz) {
                            //class 이름이 HelloT로 시작하는 것만 선정
                            return clazz.getSimpleName().startsWith("HelloT"); // 클래스 이름이 HelloT로 시작하는 것만 선정
                        }
                    };
                }
            };
            classMethodPointcut.setMappedName("sayH*"); // sayH로 시작하는 메소드 이름을 가진 메소드만 선정됨
    
            //테스트
            checkAdviced(new HelloTarget(), classMethodPointcut, true);
    
            class HelloWorld extends HelloTarget{};
            checkAdviced(new HelloWorld(), classMethodPointcut, false);
    
            class HelloToby extends HelloTarget{};
            checkAdviced(new HelloToby(), classMethodPointcut, true);
        }
    
        private void checkAdviced(Object target, Pointcut pointcut,
                                  boolean adviced) {
            ProxyFactoryBean pfBean = new ProxyFactoryBean();
            pfBean.setTarget(target);
            pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
            Hello proxiedHello = (Hello) pfBean.getObject();
    
            if(adviced){
                assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
                assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
                assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
            }else{
                assertThat(proxiedHello.sayHello("Toby"), is("Hello Toby"));
                assertThat(proxiedHello.sayHi("Toby"), is("Hi Toby"));
                assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
            }
        }
    }
    ```
    
    - 세 가지 클래스에 대해 테스트를 진행
    - 두 개의 메소드에는 어드바이스를 적용, 마지막 것은 적용 X

### 6.5.2 DefaultAdvisorAutoProxyCreator의 적용

#### 클래스 필터를 적용한 포인트컷 작성

- 메소드 이름만 비교하던 포인트컷(NameMatchMethodPointcut)을 상속해서 프로퍼티로 주어진 이름 패천을 가지고 클래스 이름을 비교하는 ClassFilter를 추가
    
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
    

#### 어드바이저를 이용하는 자동 프록시 생성기 등록

- 적용할 자동 프록시 생성기(DefaultAdvisorAutoProxyCreator)는 Advisor 인터페이스를 구현할 것을 탐
- 생성되는 모든 빈에 대해 어드바이저의 포인트컷을 적용해보면서 프록시 적용 대상을 선정
- 빈 클래스가 프록시 선정 대상이면 → 프록시를 만들어 원래 빈 오브젝트와 바꿔치기

→ 원래 빈 오브젝트는 프록시 뒤에 연결돼서 프록시를 통해서만 접근 가능하게 바뀌는 것

→ 타깃 빈에 의존한다고 정의한 다른 빈들은 프록시 오브젝트를 대신 DI 받게 됨.

- DefaultAdvisorAutoProxyCreator 등록
    - `<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>`

#### 포인트컷 등록

- 포인트컷 빈
    
    ```java
    <beans
        ...>
        <bean id="transactionPointcut"
              class="springbook.service.NameMatchClassMethodPointcut">
            <property name="mappedClassName" value="*ServiceImpl"/>
            <property name="mappedName" value="upgrade*"/>
        </bean>
    <beans/>
    ```
    

#### 어드바이스와 어드바이저

- 어드바이스와 어드바이저를 수정할 필요 X
- 하지만, **어드바이저로서 사용되는 방법은 바뀌었다!**

#### ProxyFactoryBean 제거와 서비스 빈의 원상복구

📌 명시적인 프록시 팩토리 빈을 등록하지 않기 때문에 userServiceImpl 빈의 아이디를 userService로 되돌려놓을 수 있다.

- 프록시 팩토리 빈을 제거한 후의 빈 설정
    
    ```java
    <beans
        ...>
        <bean id="userService" class="springbook.service.UserServiceImpl">
            <property name="userDao" ref="userDao"/>
            <property name="mailSender" ref="mailSender"/>
        </bean>
    <beans/>
    ```
    

#### 자동 프록시 생성기를 사용하는 테스트

🤔 @Autowired를 통해 컨텍스트에서 가져오는 UserService 타입 오브젝트는 트랜잭션이 적용된 프록시여야 한다.

→ 이를 검증하려면 테스트가 필요한데, 기존의 방법으로는 한계가 있다.

- 타깃을 테스트용 클래스로 바꿔치기하기 위해 가져올 빈이 없기 때문
- 프록시 오브젝트만 남아있다…!

그렇다면, 어떻게 해야 할까?

- 기존에 사용하던 강제 예외 발생용 TestUserSerive 클래스를 직접 빈으로 등록해보자.
    - 두 가지 문제점이 있다.
        1. TestUserService가 클래스 내부의 스태틱 클래스임.
        2. 포인트컷이 트랜잭션 어드바이스를 적용해주는 대상 클래스의 이름 패턴 때문에 TestUserService 클래스는 빈으로 등록해도 포인트컷이 프록시 적용 대상으로 선정하지 않음.
    
    → 두 가지 문제 해결을 위해 TestUserService를 조금 수정하자.
    
    - 수정한 테스트용 UserService 구현 클래스
        
        ```java
        public class UserServiceImpl implements UserService {
            ...
            public static class TestUserServiceImpl extends UserServiceImpl {
                private String id = "k2";
        
                @Override
                protected void upgradeLevel(User user) {
                    if (user.getId().equals(this.id)) throw new TestUserServiceException();
                    super.upgradeLevel(user);
                }
            }
        }
        ```
        

- 이제 TestUserServiceImpl을 빈으로 등록하자.
    
    ```java
    <beans
        ...>
        <bean id="testUserService" class="springbook.user.service.UserServiceImpl$TestUserServiceImpl"
              parent="userService">
        </bean>
    <beans/>
    ```
    
    - 특이사항 두 가지
    1. 클래스 이름에 사용한 $ 기호
        - 스태틱 멤버 클래스를 지정할 때 사용하는 것
    2. parent 애트리뷰트
        - <bean> 태그에 parent 애트리뷰트를 사용하면 다른 빈 설정의 내용을 상속받을 수 있다.

- 마지막으로, upgradeAllOrNothins() 테스트를 새로 추가한 testUserService 빈을 사용하도록 수정
    
    ```java
    public class UserServiceTest {
        @Autowired
        UserService userService;
    
        @Autowired
        UserService testUserService;
        ...
        @Test
        public void upgradeAllOrNothing() {
            userDao.deleteAll();
            for (User user : users) userDao.add(user);
    
            try {
                this.testUserService.upgradeLevels();
                fail("TestUserServiceException expected");
            } catch (UserServiceImpl.TestUserServiceException e) {
            }
    
            checkLevelUpgraded(users.get(0), false);
        }
    }
    ```
    
    - 테스트가 정상적으로 동작하는지 확인하고
    - upgradeAllOrNothing() 테스트를 통해서 자동 프록시 생성기가 평범한 비즈니스 로직만 담고 있는 빈을 자동으로 트랜잭션 부가기능을 제공해주는 프록시로 대체했는지 확인해보자!

#### 자동생성 프록시 확인

1️⃣ 트랜잭션이 필요한 빈에 트랜잭션 부가기능이 적용됐는지

2️⃣ 아무 빈에나 트랜잭션 부가기능이 적용된 것은 아닌지

확인해보자!

1️⃣은 앞서 만든 upgradeAllOrNothing() 테스트를 통해 검증했다.

2️⃣는 클래스 필터가 제대로 동작하는지 확인하면 된다.

→ 포인트컷 빈의 클래스 이름 패턴을 변경해서  testUserService 빈에 트랜잭션이 적용되지 않게 해보자!

```java
<beans
    ...>
    <bean id="transactionPointcut"
          class="springbook.user.service.NameMatchClassMethodPointcut">
        <property name="mappedClassName" value="*NotServiceImpl"/>
        <property name="mappedName" value="upgrade*"/>
    </bean>
<beans/>
```

- 테스트를 실행하면 upgradeAllOrNothing()만 실패해야 한다.
- 확인한 후 다시 테스트를 원상복귀시켜서 테스트가 모두 성공하도록 해야 한다.

- 원상복귀 시켰다면 테스트에 컨테이너가 돌려준 서비스 빈의 타입을 확인하는 코드를 넣자
    
    ```java
    @Test
    public void advisorAutoProxyCreator() {
    	assertThat(testUserService, is(java.lang.reflect.Proxy.class));
    }
    ```
    

### 6.5.3 포인트컷 표현식을 이용한 포인트컷

✨ 지금까지 사용했던 포인트컷 → 메소드의 이름, 클래스 이름 패턴을 클래스 필터와 메소드 매처 오브젝트로 비교해서 선정하는 방식

→ 좀 더 편리한 포인트컷 작성 방법을 알아보자!

- 스프링은 간단하고 효과적인 방법으로 포인트컷의 클래스와 메소드를 선정하는 알고리즘을 작성하는 방법을 제공함
    
    → 포인트컷 표현식
    

#### 포인트컷 표현식

- 포인트컷 표현식을 지원하는 포인트컷을 적용하려면 AspectJExpressionPointcut 클래스를 사용하면 됨.
    
    → 클래스와 메소드의 선정 알고리즘을 포인트컷 표현식을 이용해 한 번에 지정 가능
    
- AspectJ 포인트컷 표현식
    
    스프링이 사용하는 포인트컷 표현식은 AspectJ라는 유명한 프레임워크에서 제공하는 것을 가져와 일부 문법을 확장해서 사용하는 것
    

#### 포인트컷 표현식 문법

📌 AspectJ 포인트컷 표현식은 포인트컷 지시자를 이용해 작성함.\

- 가장 대표적인 것: execution ([]: 생략 가능을 의미, |: OR 조건ㄴ)

- execution([접근제한자 패턴] 타입패턴 [타입패턴.]이름패턴 (타입패턴 | "..", ...) [throws 예외 패턴])
    - 복잡해 보이지만 메소드의 풀 시그니처를 문자열로 비교하는 개념이라고 생각하면 간단함.

- 출력 내용
    - public
        - 접근 제한자
        - 포인트컷 표현식에서 생략 가능
    - int
        - 리턴 값의 타입을 나타내는 패턴
        - 필수항목
        - *: 모든 타입을 다 선택
    - springbook.learning.spring.pointcut.Target
        - 여기까지가 패키지와 타입 이름을 포함한 클래스의 타입 패턴
        - 생략 가능
    - minus
        - 메소드 이름 패턴
        - 필수항목
        - *: 모든 메소드 선택
    - (int. int)
        - 메소드 파라미터의 타입 패턴
        - ‘,’로 파라미터 타입 구분, 순서대로 작성
        - 파라미터가 없는 메소드 지정시 ( ) 작성
        - 파라미터의 타입과 개수에 상관없이 모두 다 허용하는 패턴 → ‘..’ 사용
        - 뒷 부분의 파라미터 조건만 생략시 → ‘…’ tkdyd
        - 필수항목
    - throws java.lang.RuntimeException
        - 예외 패턴으로 생략 가능 함

#### 포인트컷 표현식 테스트

🤔 메소드 시그니처를 그대로 사용한 포인트 표현식의 문법 구조를 참고해서 정리해보자!

- 필수가 아닌 항목을 생략할 수 있다
- execution(int minus(int, int))
    - 생략한 부분은 모든 경우를 다 허용한다.
- 리턴 값의 타입에 대한 제한을 없애려면
    - execution(* minus(int, int))
- 파라미터의 개수와 타입을 무시하려면
    - execution(* minus (..))
- 모든 메소드를 다 허용하려면
    - execution(* *(..))

#### 포인트컷 표현식을 이용하는 포인트컷 적용

📌 AspectJ 포인트컷 표현식은 다양한 문법과 활용 방법이 있다

ex) bean(): 스프링에서 사용될 때 빈의 이름으로 비교하는 것

- 특정 애노테이션이 타입, 메소드, 파라미터에 적용되어 있는 것을 메소드를 선정할 수 있게 하는 포인트컷 작성 가능
    - `@annotation(org.springframework.transaction.annotation.Transactional)`

- 기존 포인트컷과 동일한 기준으로 메소드를 선정하는 알고리즘을 가진 포인트컷 표현식을 만들어보자!
- 포인트컷 표현식: AspectExpressionPointcut 빈을 등록하고 expression 프로퍼티에 넣어주면 된다.
- 설정 파일을 수정했으면 테스트가 잘 돌아가는지 확인하자!

#### 타입 패턴과 클래스 이름 패턴

- 클래스 이름 패턴 사용
    - `ServiceImpl`로 끝나는 클래스 이름을 기준으로 빈을 선정.
    - `UserServiceImpl`과 `TestUserServiceImpl` 두 클래스가 트랜잭션 대상이 됨.
    - 테스트를 위해 `TestUserService`를 `TestUserServiceImpl`로 변경하여 이름 패턴에 맞춤.
    
- 포인트컷 표현식에서 타입 패턴 도입
    - 클래스 이름 대신 타입을 기준으로 빈 선정.
    - 표현식: `execution(* *..*ServiceImpl.upgrade*(..))`
    - 타입 패턴이므로 `ServiceImpl`로 끝나는 타입을 가진 빈이 대상.
        - 상속 관계나 인터페이스 구현까지 포함.
    - 결과적으로 `UserServiceImpl`과 `TestUserServiceImpl` 모두 타깃에 포함.

- 테스트 중 발생한 의문
    - `TestUserServiceImpl`을 다시 `TestUserService`로 이름 변경.
    - 예상: 클래스 이름이 `ServiceImpl`로 끝나지 않으니 타깃에서 제외 → 테스트 실패.
    - **결과**: 테스트 성공.
        - **왜?** 포인트컷 표현식은 **타입 패턴**을 사용했기 때문.

- 타입 패턴의 작동 원리
    - `TestUserService` 클래스는 다음 타입을 모두 가짐:
        1. 본인 타입 (`TestUserService`)
        2. 상위 클래스 타입 (`UserServiceImpl`)
        3. 구현 인터페이스 타입 (`UserService`)
    - 따라서, `ServiceImpl`로 끝나는 상위 클래스 타입을 통해 포인트컷 조건 충족.

### 6.5.4 AOP란 무엇인가?

✨ 비즈니스 로직을 담은 UserService에 트랜잭션을 적용해온 과정을 정리해보자.

#### 1️⃣ 트랜잭션 서비스 추상화

- 첫 번째 문제: 특정 트랜잭션 기술에 종속되는 코드가 되는 것
    
    → 추상적인 작업 내용은 유지하면서 구체적인 구현 방법을 바꿀 수 있는 서비스 추상화 기법 적용
    

#### 2️⃣ 프록시와 데코레이터 패턴

- 서비스 추상화를 하더라도 여전히 비즈니스 로직에 트랜잭션을 적용하고 있다는 사실이 노출됨.
- 단순한 추상화와 메소드 추출 방법으로는 더 이상 제거할 수 없음.
    
    → DI를 이용해 데코레이터 패턴을 적용
    

#### 3️⃣ 다이내믹 프록시와 프록시 팩토리 빈

- 프록시를 이용해서 비즈니스 로직에서 트랜잭션은 모두 제거할 수 있었지만
- 프록시 클래스를 일일이 만드는 작업이 문제가 됨.
    
    → JDK 다이내믹 프록시 기술을 적용
    
    - 프록시 클래스 없이도 프록시 오브젝트를 런타임 시에 생성

#### 4️⃣ 자동 프록시 생성 방법과 포인트컷

- 트랜잭션 적용 대상이 되는 빈마다 프록시 팩토리 빈을 설정해줘야 함
    
    → 스프링 컨테이너의 빈 생성 후처리 기법을 활용
    

#### 5️⃣ 부가기능의 모듈화

- 부가기능은 핵심 기능과 같은 방식으로 모듈화하기 힘들다.
    - 독립적인 방식으로 존재해서 적용되기 어렵기 때문
    
    → 부가기능을 어떻게 독립적인 모듈로 만들 수 있을까?
    
    - DI, 데코레이터 패턴, 다이내믹 프록시, 오브젝트 생성 후처리, 자동 프록시 생성, 포인트컷

⇒ 결국, 지금까지의 작업은 부가기능을 효과적으로 모듈화하는 방법을 찾는 것이었다!

#### AOP: 애스펙트 지향 프로그래밍

- 애스펙트(aspect)
    - 부가기능 모듈을 특별한 이름으로 부르기 시작한 말
    - 핵심기능에 부가되어 의미를 갖는 특별한 모듈
    - 어드바이스, 포인트컷을 함께 갖고 있음.
    - 어드바이저 → 아주 단순한 형태의 애스펙트￼

- 독립 애스펙트를 이용한 부가기능의 분리와 모듈화
    
   ![image (1)](https://github.com/user-attachments/assets/04f47a24-f746-4dce-bb4d-c13dd8a65f83)

    
    - 왼쪽: 애스펙트로 부가기능을 분리하기 전
    - 오른쪽: 핵심기능 코드 사이에 침투한 부가기능을 애스펙트로 구분한 것
        - 핵심기능은 순수하게 그 기능을 담은 코드로만 존재하고
        - 독립적으로 살펴볼 수 있도록 구분된 면에 존재하게 됨.

📌 이처럼 애플리케이션의 핵심기능에서 부가기능을 분리해서 애스펙트라는 독특한 모듈로 만들어서 설계하고 개발하는 방법 → **애스펙트 지향 프로그래밍(AOP)**

- AOP를 통해 애플리케이션을 다양한 측면에서 독립적으로 모델링하고, 설계하고, 개발할 수 있음.

### 6.5.5 AOP 적용기술

#### 프록시를 이용한 AOP

✨ 스프링은 다양한 기술을 조합해 AOP를 지원하지만 그중 핵심은 **프록시**를 이용한다는 것이다.

- 프록시로 만들어서 DI로 연결된 빈 사이에 적용해 타깃의 메소드 호출 과정에 참여해서 부가기능을 제공해주도록 구현

→ 스프링 AOP는 기본 JDK와 스프링 컨테이너 외에는 특별한 기술, 환경을 요구하지 X

→ 부가기능 모듈을 타깃 오브젝트의 메소드에 다이내믹하게 적용하기 위해 중요한 역할을 하는것이 프록시이므로

→ 스프링 AOP는 **프록시 방식의 AOP**라고 한다.

#### 바이트코드 생성과 조작을 통한 AOP

🤔 그렇다면 프록시 방식이 아닌 AOP도 있을까?

- **AspectJ**는 프록시를 사용하지 않는 대표적인 AOP 기술

→ AspectJ는 프록시를 사용하지 않고 어떻게 부가기능을 다이내믹하게 타깃 오브젝트에 적용할까?

- 타깃 오브젝트를 고쳐서 부가 기능을 직접 넣어주는 방법을 사용

🤔 그렇다면, 왜 굳이 프록시를 두고 복잡한 방법을 사용할까?

1. DI 컨테이너의 도움을 받아 자동 프록시 생성 방식을 사용하지 않아도 AOP를 적용할 수 있기 때문
2. 프록시 방식보다 훨씬 강력하고 유연한 AOP가 가능하기 때문

### 6.5.6 AOP의 용어

- 타깃
    - 부가기능을 부여할 대상
- 어드바이스
    - 타깃에게 제공할 부가기능을 담은 모듈
    - 메소드 호출 과정에 전반적으로 참여하는 것도 있고
    - 일부에서만 동작하는 어드바이스도 있음.
- 조인 포인트
    - 어드바이스가 적용될 수 있는 위치
- 포인트컷
    - 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정의한 모듈
- 프록시
    - 클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트
    - DI를 통해 타깃 대신 클라이언트에게 주입됨
    - 클라이언트의 메소드 호출을 대신 받아 타깃에 위임
    - 그 과정에서 부가기능을 부여
- 어드바이저
    - 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트
    - 어떤 부가기능을 어디에 전달할 것인가를 알고 있는 모듈
- 애스펙트
    - AOP의 기본 모듈
    - 한 개 또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어짐
    - 대개 싱글톤 형태의 오브젝트로 존재

### 6.5.7 AOP 네임스페이스

✨ 스프링의 프록시 방식 AOP를 적용하려면 최소한 네 가지 빈을 등록해야 함.

1️⃣ 자동 프록시 생성기

2️⃣ 어드바이스

3️⃣ 포인트컷

4️⃣ 어드바이저

#### AOP 네임스페이스

- 스프링에서는 이렇게 AOP를 위해 빈들을 간편한 방법으로 등록할 수 있음.

→ 이때, AOP와 관련된 태그를 정의해둔 app 스키마를 제공

→ app 스키마에 정의된 태그는 별도의 네임스페이스를 지정해서 디폴트 네임스페이스의 <bean> 태그와 구분해서 사용 가능

- 네임스페이스 선언
    
    ```java
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:aop="http://www.springframework.org/schema/aop"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
                                http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
                                http://www.springframework.org/schema/aop
                                http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">
        ...
    </beans>
    ```
    

- aop 네임스페이스를 적용한 AOP 설정 빈
    
    ```java
    <aop:config>
        <aop:pointcut id="transactionPointcut" expression="execution(* *..*ServiceImpl.upgrade*(..))"/>
        <aop:advisor advice-ref="transactionAdvice" pointcut-ref="transactionPointcut"/>
    </aop:config>
    ```
    
    - 이해하기 쉬워졌고 코드의 양도 대폭 줄어든다.

#### 어드바이저 내장 포인트컷

📌 AspectJ 포인트컷 표현식을 활용하는 포인트컷은 스트링으로 된 표현식을 담은 expression 프로퍼티 하나만 설정해주면 사용할 수 있음.

- 포인트컷을 내장한 어드바이저 태그
    
    ```java
    <aop:config>
        <aop:advisor advice-ref="transactionAdvice" pointcut="execution(* *..*ServiceImpl.upgrade*(..))"/>
    </aop:config>
    ```

