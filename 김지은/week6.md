# 6주차 - AOP 1

- 노션 링크 : [6주차 - AOP 1](https://www.notion.so/6-AOP-1-13d6873728d080d8ba25ec61627dc257?pvs=21)

---

## 트랜잭션 코드의 분리

- 트랜잭션 경계설정 기능을 AOP를 도입해 더 깔끔한 방식으로 변경해보자
- 기존에 추상화를 적용한 UserService의 메소드

```java
 public void upgradeLevels() throw Exception {
	 TransactionStatus status = this.TransactionManager
				 .getTransaction(new DefaultTransactionDefinition());
	 try{
	 
		 // 비즈니스 로직
		 upgradeLevelsInternal();
		 
		 this.transactionmanager.commit();
	 } catch(Exception e) {
		 this.transactionManager.rollback();
		 throw e;
	 }
 }
 
 private void upgradeLevelsInternal() {
	 List<User> users = userDao.getAll();
	 for(User user : users) {
		 if(canUpgradeLevel(user)) upgradeLevel(user);
	 }
 }
```

<aside>
💡

특징

- 비즈니스 로직인 upgradeLevelsInteranl()과 트랜잭션 경계설정 코드(try-catch)는 
**주고 받는 데이터가 없는 완벽히 독립적인 코드**이다
- 비즈니스 로직의 코드가 트랜잭션 시작-종료 작업 **사이에 수행돼야 한다는 사항**만 지켜지면 된다

⇒ **트랜잭션 코드**를 **비즈니스 로직 클래스 밖으로** 뽑아내보자

</aside>

### DI를 이용한 클래스의 분리

- UserService에서 트랜잭션 코드를 분리해내고도 클라이언트는 UserService와 트랜잭션 기능을 모두 사용할 수 있도록 DI를 통해 UserService를 간접적으로 접근할 수 있게 하자
- DI를 이용했을 때 활용할 수 있는 점
    1. 구현 클래스를 바꿔가면서 사용
    2. **한 번에 두 개의 UserService 인터페이스 구현 클래스를 동시에 사용
    USerService 인터페이스를 두고 → 비즈니스로직클래스 / 트랜잭션클래스 동시에 사용**

```java
public interface UserService {
	void add(User user);
	void upgradeLevels();
}
```

```java
public class UserServiceImpl implements UserService {
	UserDao usesrDao;
	MailSender mailSender;
	
	void add(User user) {
		//...
	}
	
	void upgradeLevels() {
		List<User> users = userDao.getAll();
	 for(User user : users) {
		 if(canUpgradeLevel(user)) upgradeLevel(user);
	 }
	}
}
```

```java
public class **UserServiceTx** implements UserService {

	
	UserService userService;
	PlatformTransactionManager transactionManager;
	
	public void setTransaction(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}
	//작업 위임하기 위한 usreServiceImpl 주입받을 것
	public void setuserService(UserService userService) {
		this.userService = userService;
	}
	
	void add(User user) {
	// 비즈니스 로직 위임
		userService.add();
	}
	
	void upgradeLevels() {
		TransactionStatus status = this.TransactionManager
				 .getTransaction(new DefaultTransactionDefinition());
		 try{
	 
			 // 비즈니스 로직 위임
			 userSevice.upgradeLevels();
		 
			 this.transactionmanager.commit();
		 } catch(Exception e) {
			 this.transactionManager.rollback();
			 throw e;
		 }
	 }
}
```

<aside>
💡

**한 번에 두 개의 UserService 인터페이스 구현 클래스를 동시에 사용**

- 인터페이스 : -UserService
    - **실제 비즈니스 로직** 구현 클래스 : **UserServiceImpl**
    - **트랜잭션 처리** 구현 클래스 : **UserServiceTx**
- 오브젝트 의존관계
    
    : **클라이언트** → **UserServiceTx** : 트랜잭션 처리 + **실제 비즈니스는 UserServiceImpl에 위임** → **UserServiceImpl**
    
</aside>

---

## 다이내믹 프록시와 팩토리 빈

### 프록시와 타깃

- **프록시** : 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받는 오브젝트
    - 위에서 트랜잭션을 처리하는 **UserServiceTx가 프록시 → UserServiceImpl를 호출**
    - 특징
        1. 타깃과 같은 인터페이스를 구현했다
        2. 프록시가 타깃을 제어할 수 있다 (타깃에 실제 핵심 기능 수행을 위임한다)
- 타깃 : 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트
    - 위에서 **UserServiceTx에서 호출되는 UserServiceImpl이 타깃**
    - 클라이언트는 인터페이스만 보고 사용 → **프록시 - 타깃은 같은 인터페이스를 구현**하게 함
    - **클라이언트는 실제로 부가기능을 구현한 프록시를 호출**
        
        → 프록시 : 부가기능 호출 후 핵심기능을 구현한 클래스로 위임
        
        → 클**라이언트는 프록시를 거쳐서 핵심기능으로 위임**된다 
        
          클라이언트 → 핵심기능 인터페이스 ( : 프록시, 부가기능 수행 ) → 핵심기능 인터페이스 ( : 타깃, 실제 핵심기능 수행 )  
        

<aside>
💡

프록시의 **사용 목적**

1. 클라이언트 → 타깃 **접근 방법 제어**
2. 타깃에 **부가적인 기능 부여**
</aside>

### 데코레이터 패턴

- 타깃에 **부가적인 기능을 런타임에 다이나믹하게 부여**하기 위해 프록시를 사용하는 패턴
    - 코드 상에는 프록시 - 타깃의 연결이 정해져 있지 않다
    - 여러 개의 프록시를 사용하며, 단계적으로 위임할 수 있다
    - 런타임 시에 이를 적절한 순서로 조합해서 사용할 수 있다
- 각 데코레이터는 **위임하는 대상에도 인터페이스로 접근한**다
    - 다음 위임 대상은 **인터페이스로 선언하고, 위임 대상을 외부에서 DI** 받는다

### 프록시 패턴

- 타깃에 대한 **접근방법을 제어**하는 목적을 가진 경우 : 기능 자체에는 관여하지 않으면서 접근 방법만 제어
    1. **생성 지연** : 타깃 오브젝트를 당장 생성하지 않았을 때, 타깃 오브젝트에 대한 레퍼런스가 미리 필요한 경우
        - 실제 타깃 오브젝트를 생성하는 대신 프록시를 넘겨주고
        - 실제로 타깃을 사용하려고 하는 시점에 타깃 오브젝트를 생성해서 요청을 위임해주는 방식
    2. 특별한 상황에 타깃에 대한 **접근권한을 제어**하는 경우
        - 읽기 전용으로 동작하고, 접근 불가 예외를 발생시키기
- 접근할 타깃 클래스의 정보를 알고 있는 경우도 있음 + 인터페이스로 접근하기도 함

---

### 리플렉션

런타임에 메소드, 클래스, 인터페이스의 동작에 접근하여 검사하거나 수정하는 등,**동적으로 자바의 코드 자체를 추상화 하여 접근**할 수 있도록 하는 API, 기법

- 자바의 clas 타입 오브젝트 : **Java.lang.Class**
    - .java → .class 파일로 클래스 로더(ClassLoader)에 의해서 클래스 파일이 메모리에 올라갈 때, Class 클래스는 이.class 파일의 클래스 정보들을 가져와 힙 영역에 자동으로 객체화된다
    - 자바의 모든 클래스는 **클래스 자체의 구성정보를 담은 Class 타입 오브젝트**를 하나씩 갖는다
        - 클래스이름.class, 오브젝트의 getClass() 메소드로 Class 타입의 오브젝트를 가져올 수 있다
        - 클래스 오브젝트를 이용하여 클래스의 메타정보를 가져오거나, 오브젝트를 조작할 수 있다
- **Method 인터페이스 활용 - java.lang.reflect.Method**
    - 클래스 정보, 이름으로 **메소드 정보**를 가져올 수 있다
        
        : getMethod(methodname)
        
    - 메소드 대상 오브젝트, 파라미터 목록을 인자로 받아 **메소드를 실행**시킬 수 있다
        
        : **`public Object invoke(Object obj, Obejct.. args)`**
        
    - 다음과 같은 방법으로 실행
    
    ```java
    // String 클래스의 length라는 메소드 정보 얻어음
    Method lengthMth = String.class.getMethod("length");
    
    int length = lengthMth.invoke("spring");
    // length == 6
    ```
    

---

## 다이내믹 프록시

### 프록시 작성의 문제점

1. 타깃과 같은 인터페이스를 구현하고 위임한다 → 인**터페이스의 모든 메소드를 구현하고, 위임하는 코드를 일일히 작성**해야 한다
2. 지정된 요청에 대한 부가기능을 수행한다 → **부가기능 코드가 중복**될 가능성이 많다

### 다이내믹 프록시 적용

![image.png](image.png)

- **인터페이스를 구현한 프록시를 작성**해야 하고 + **부가기능 중복 코드**가 발생한다는 **프록시의 단점을 해소**하기 위한 구조

1. **클라이언트 → 프록시 팩토리** : 인터페이스 정보를 제공해 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 생성 (다이내믹 프록시)
2. **다이내믹 프록시** : 프록시 팩토리에 의해 런타임에 동적으로 만들어지는 오브젝트, 타킷의 인터페이스와 동일한 타입
    - **요청정보를 리플렉션으로 변환**해 **InvocationHandler 구현 클래스의 invoke로 넘긴다**
    
    ```java
    //타깃의 인터페이스인 Hello
    Hello proxiedHello = (Hello) Proxy.newProxyInstance(
    	getClass().getClassLoader(),
    	new Class[] {Hello.class},
    	new UppercaseHandler(new HelloTraget())
    	);
    ```
    
    - 다이내믹 프록시 **생성 - 파라미터**
        - 프록시 정의되는 **클래스로더** 지정 : `getClass().getClassLoader()`
        - 다이내믹 프록시가 구현할 인터페이스 (**타깃 인터페이스**) : `new Class[] {Hello.class}`
        - **InvocationHandler 구현 클래스** : `new UppercaseHandler(new HelloTraget())`
3. **InvocationHandler** : 부가기능 제공코드는 InvocationHandler 인터페이스의 구현 클래스에 담음
    - **`public Object invoke(Object target, Method method, Object[] args)`** 를 구현한다
    - 리플렉션 Method 인터페이스를 파라미터로 받음 → **전달받은 타깃의 특정 메소드를 수행**한다
    - Reflection.invoke()을 활용해 **모든 메소드 요청을 invoke() 메소드 하나로 처리**할 수 있다
    - **다이나믹 프록시로부터 요청을 전달받아 → 부가적인 구현내용 + 타깃에 위임 하는 역할**
    
    ```java
    public class UppercaseHandler implements InvocationHandler {
    	Object target;
    	
    	// 타깃 오브젝트에 위임 (타깃의 메소드 수행 : invoke(target))하기 위해 
    	// 타깃 오브젝트 주입 받음
    	private UppcaseHandler(Obejct target) {this.target = target;}
    	
    	public Object invoke(Object proxy, Method method, Object[] args) {
    		// 타깃에 위임 : 타깃 오브젝트의 메소드 호출
    		Obejct ret = method.invoke(proxy, args);
    		// 부가기능 제공 : touppercase
    		if(ret instanceof String) return ((String)ret).toUpperCase());
    		else return retl
    	}
    }
    ```
    
    - 모든 요청을 invoke 하나로 받기 때문에 어떤 메소드로 받을지 선택하거나 적용 여부를 조정하는 처리가 필요하기도 하다

---

## 다이내믹 프록시를 이용한 트랜잭션 부가기능

- 트랜잭션 InvocationHandler 구현체

```java
public class TransactionHandler implements InvocationHandler {
		// 타깃 오브젝트
    private Object target;
    // 트랜잭션 기능 제공
    private PlatformTransactionManager transactionManager;
	  // 트랜잭션을 적용할 메소드 이름 패턴
    private String pattern;

    public void setTarget(Object target) {this.target = target;}
    public void setTransactionManager(PlatformTransactionManager transactionManager) {this.transactionManager = transactionManager;}
		public void setPattern(String pattern) {this.pattern = pattern;}

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		    // 트랜잭션 적용 (부가기능) 대상 메소드 선별해서 적용
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        }
        // 부가기능 없이 타깃에 위임
        return method.invoke(target, args);
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
		    // 부가기능 적용
        TransactionStatus status = this.transactionManager
		        .getTransaction(new DefaultTransactionDefinition());

        try {
		        // 타깃에 위임
            Object returnValue = method.invoke(target, args);
            // 커밋
            this.transactionManager.commit(status);
            return returnValue;
        } catch (InvocationTargetException e) {
            this.transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}
```

- **InvocationTargetException**
    - Method.invoke()에서 발생하는 예외는 InvocationTargetException 로 한 번 감싸서 전달된다
    - getException()으로 중첩되어 있는 예외를 가져와야 한다

- 테스트

```java
public void upgradeAllOrNothing() throws Exception {
    //...

    TransactionHandler transactionHandler = new TransactionHandler();
    // transactionhandler -> 필요한 오브젝트 DI
    transactionHandler.setTarget(testUserService);
    transactionHandler.setTransactionManager(transactionManager);
    transactionHandler.setPattern("upgradeLevels");
		// 다이나믹 프록시 생성
    UserService userServiceTx = (UserService) Proxy.newProxyInstance(getClass().getClassLoader(), new Class[]{UserService.class}, transactionHandler);

    //...
    // 이후 테스트 실행
}
```

---

## 다이내믹 프록시 - 팩토리 빈

### 클래스 정보 + 디폴트 생성자를 이용한 빈 생성

- 내부적으로 **리플렉션 API**를 이용하여 빈 정의에 나오는 클래스 이름을 가지고 빈 오브젝트를 생성 : **클래스 이름, 프로퍼티**
    - `Class.forName(”beanName”).newInstance();`
- 하지만 **다이내믹 프록시**는 오직 스태틱한 팩토리 메소드인 **Proxy.newProxyInstance()로만** 만들 수 있다
    
    → 미리 클래스 정보를 알아내 스프링 빈에 **정의할 방법이 없다**
    
    - **리플렉션으로 생성하려면 기본생성자가 존재해야 하기 때문**에
    - 사실 private 생성자로도 등록이 되긴 하지만, 스태틱 메소드로만 생성하기로 지정한 이유가 존재할 것이다 → 맘대로 변경해서 등록하면 위험함

### 팩토리 빈

- **스프링을 대신해서 오브젝트의 생성로직을 담당**하도록 만들어진 특별한 빈
- 스프링의 FactoryBean 인터페이스 구현해서 생성 : 세 가지 메소드로 구성
    - **getObject**() : 빈 오브젝트 생성해 반환
    - **getObejctType**() : 오브젝트 타입 알려줌
    - **isSingleton**() : 항상 같은 싱글톤 오브젝트인지 여부
    - **FactoryBean 인터페이스를 구현**한 클래스를 스프링의 빈으로 등록해준다
        - getObject() 오브젝트를 생성하는 코드를 넣으면 됨

### 다이내믹 프록시를 만들어주는 팩토리 빈

```java
@Setter
public class TxProxyFactoryBean implements FactoryBean<Object> {
	// TranscationHandler 생성에 필요한 오브젝트들
  private Object target;
  private PlatformTransactionManager transactionManager;
  private String pattern;
  
  // 다양한 타입의 프록시 오브젝트 생성에 재사용 할 수 있다
  // DI 받은 인터페이스의 타입에 따라 팩토리 빈이 생성하는 오브젝트 타입이 달라진다
  private Class<?> serviceInterface;          
  
  // FactoryBean 인터페이스 구현 메소드
  // DI 받은 정보를 이용해 Transactionhandler를 사용하는 다이내믹 프록시를 생성해 반환
  public Object getObject() throws Exception {
    TransactionHandler txHandler = new TransactionHandler();
    txHandler.setTarget(target);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern(pattern);
    return Proxy.newProxyInstance(getClass().getClassLoader(),
            new Class[]{serviceInterface},
            txHandler);
  }
  public Class<?> getObjectType() {
    return serviceInterface;
  }
    public boolean isSingleton () {
      // getObject가 매 번 같은 객체를 리턴하지 않는다 : 매번 새로운 다이나맥 프록시 리턴
      return false;
  }
}
```

### 프록시 팩토리 빈 방식의 장점

- 프록시 팩토리 빈의 **재사용**
    - 다이내믹 프록시를 사용하면 **프록시의 단점을 해결**해준다
        
        : 인터페이스 구현 클래스 작성, 부가기능의 코드 중복 발생 해결
        
    - DI 설정을 더해주면 번거로운 다이내믹 프록시 생성 코드도 제거 가능
    - 다양한 타깃 오브젝트에 적용 가능

### 프록시 팩토리 빈의 한계

- **여러 개의 클래스에 공통적인 부가기능을 제공**하는 것이 불가능하다
    - 여러 클래스에 적용해야 한다면 비슷한 프록시 팩토리 빈의 설정이 중복될 것이다
- **하나의 클래스에 여러 개의 부가기능을 적용**할 때도 비슷한 프록시 팩토리 빈의 설정이 중복된다
    - 즉 **여러 클래스, 여러 부가기능에 걸쳐 제공되려면 비슷한 내용이 반복되는, 복잡한 설정파일을 작성**해야 한다
- InvocationHandler의 구현체 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다
    - 서로 다른 프로퍼티(타깃) 개수만큼 다른 빈으로 등록 → 오브젝트로 생성된다
    - 모든 타깃에 적용 가능한 싱글톤 빈이 아니다