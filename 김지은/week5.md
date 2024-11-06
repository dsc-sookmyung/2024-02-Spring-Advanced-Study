# 5주차 - 서비스 추상화

### 노션링크 : [https://merciful-marmot-54e.notion.site/5-1356873728d08021a3e4c15c3d02249e?pvs=4](https://www.notion.so/5-1356873728d08021a3e4c15c3d02249e?pvs=21)

- 기존의 사용자 CRID에서 비즈니스 로직을 추가로 구현해보자
    - 사용자 레벨 BASIC, SILVER, GOLD
    - 가입 후 활동에 다라 한 단계씩 업그레이드 - 로그인, 추천 횟수에 따라
    - 레벨 변경은 일정한 주기로 일괄로 진행

<aside>
💡

**코드 변경점**

1. **LEVEL enum 추가, User 필드 추가**(레벨, 로그인횟수, 추천횟수)+getter, setter
2. **테스트 수정** : 추가된 필드도 포함하도록 테스트 픽스처 생성자, 필드값 검증 메소드 수정
3. **UserDaoJdbc 수정** : 추가된 필드 세팅용 sql 수정
    - 주의할 점 : **LEVEL Enum ↔ SQL 형 변환**
        - java의 enum은 DB에 저장되는 SQL 타입이 아니기 때문에
        - java → DB : Java Object → int
        - DB → java 조회 : int → enum
4. **사용자 정보 수정기능 추가** : 수정할 정보를 담은 User object 전달 → id 참고해서 사용자 조회 → 필드정보를 UPDATE문으로 모두 변경하는 메소드
    - **UPDATE의 테스트 시 주의할 점 :** 수정하지 않아야 할 로우의 내용이 그대로 남아있는지 확인해줘야 한다 : UPDATE 문장에서 WHERE문을 빠트려도 오류는 발생하지 않기 때문
        1. UPDATE()의 리턴값==1을 확인하기
        2. 원하는 사용자 외의 값은 변경되지 않았음을 직접 확인하기 : 
</aside>

### 비즈니스 로직 구현 : UserService

- 레벨 업그레이드 기능을 추가할 비즈니스 로직 서비스를 제공할 UserService 클래스를 추가
    - UserDao 인터페이스 타입으로 userDao 빈을 DI 받아 사용
    - upgradeLevels() 메소드 추가 : 레벨 별 조건 체크 → 플래그 변경 → 플래그에 따라 update수행
    - 처음 가입하는 사용자가 BASIC이 되도록 수정 / 이미 레벨이 설정되어 있는 경우엔 변경x
        - 이 메소드의 위치 : 사용자 정보를 담은 User 오브젝트를 DB에 넣는 것에 충실해야 하는 UserDao에 넣는 것보다는 / UserService에서 사용자가 등록될 때 적용할 비즈니스 로직을 담당하게 하자
    
    ```java
    public class UserService {
    	UserDao userDao;
    	public void setUserDao(UserDao userDao) {
    		this.userDao = userDao;
    	}
    	
    	public void upradeLevels() {
    		List<User> users = userDao.getAll();
    		for(User user : user) {
    			// 변경여부 플래그
    			Boolean changed = null;
    			if(user.getLevel()==Level.BASIC && user.getLogin() >=50) {
    				user.setLevel(Level.SILVER);
    				changed=true;
    			} else if(user.getLevel()==Level.SIVLER && user.getRecommand() >=50) {
    				user.setLevel(Level.GOLD);
    				changed=true;
    			} else {changed=false;}
    			if(changed) userDao.update(user);
    		}
    	}
    	
    	public void add(User user) {
    		if(user.getLevel()==null) user.setLevel(Level.BASIc);
    		userDao.add(user);
    	}
    }
    ```
    

- UserService 리팩토링
    
    <aside>
    💡
    
    **문제점 파악**
    
    - **성격이 다른 여러 가지 로직이 한 데에 섞여 있다** 
    : 레벨파악 & 업그레이드 조건 파악 & 다음 레벨이 무엇인지 파악 → 업그레이드 수행 & 플래그 변경
    - 업그레이드 조건이 복잡해지거나, **로직에 변경점이 생기면 대응하기 어렵다**
    - **자주 변경되는 구체적인 내용 & 추상적인 로직의 흐름**이 섞여 있다.
    - 성격이 다른 두 가지 경우를 한 곳에서 처리한다 : 레벨파악 & 업그레이드 조건 파악 → 조건을 두 단계에 걸쳐서 비교해야 한다
    - 추상적인 로직의 흐름 (외부에 노출할 인터페이스) → 구체적인 구현 내용은 메소드로 추출하는 방식
    </aside>
    
    ```java
    public class UserService {
    	UserDao userDao;
    	public void setUserDao(UserDao userDao) {
    		this.userDao = userDao;
    	}
    	
    	public void upradeLevel() {
    		List<User> users = userDao.getAll();
    		for(User user : user) {
    			// 추상적인 로직의 흐름
    			// 업그레이드 가능한지 확인 -> 가능하다면 업그레이드
    			if(canUpgradeLevel(user)) upgradeLevel(user);
    		}
    	}
    	
    	// 구체적인 구현
    	private boolean canUpgradeLevel(User user) {
    		Level curLevel = user.getLevel();
    		switch(curLevel) {
    			case BASIC: return (user.getLogin()>=50);
    			case SILVER: return (user.getRecommand()>=30)'
    			case GOLD: return false;
    			default: throw new IllegalArgumentException("Unknown Level: "+curLevel);
    		}
    	}
    	public void upgradeLevel(User user) {
    		user.upgradeLevel();
    		userDao.update(user);
    	}
    	
    	public void add(User user) {
    		// 조건 판별 요청 & 필드 변경 요청
    		if(user.getLevel()==null) user.setLevel(Level.BASIC);
    		// 변경 사항 DB 업데이트
    		userDao.add(user);
    	}
    }
    ```
    
    ```java
    public class User {
    	//..
    	
    	// 사용자 오브젝트의 level 필드를 변경
    	public void upgradeLevel() {
    		// 레벨 순서 판별
    		Level nextLevel = this.level.getNextLevel();
    		if(nextLevel==null) throw new IllegalStateException(this.level+"은 업그레이드가 불가능합니다");
    		else this.level = nextLevel;
    	}
    }
    ```
    
    - 다음 레벨이 무엇인지 판별하는 로직 (레벨순서) → Level Enum
    - 사용자 오브젝트의 level 필드 변경 → User class
    - 조건 확인 / 변경사항 DB 업데이트 → UserService
    
    → 각 오브젝트, 메소드가 각자 자기 몫의 책임을 맡아 일하는 구조
    
    → **용도 별로 분리된 오브젝트에게 데이터를 요구하지 말고, 작업 처리를 요청한다**
    
    → 로직 변경이 필요할 때 수정할 부분을 쉽게 파악 가능 + 각각 독립적으로 테스트 가능 + 잘못된 요청, 예외사항에 대한 처리 간결함
    

- 더 고려해볼 리팩토링
    - 사용자 업그레이드 정책을 UserService에서 분리하는 방법 : 경우에 따라 레벨 정책이 변경될 수 있는 경우
    - 분리된 업그레이드 정책을 담은 오브젝트를 DI를 통해 UserService에 주입한다

---

## 트랜잭션 서비스 추상화

- 변경 요구사항 : 레벨 관리 기능 수행 중 작업이 **중단되면 모두 취소**시켜야 한다
**= UserService의 updradeLevel()이 하나의 트랜잭션 안에서 동작**해야 한다
- 트랜잭션
    - 원**자성을 띄는 작업의 단위 : 더 이상 나눠질 수 없는** 작업
    - 부분적으로 성공할 수 없고, 작업이 중단되면 초기 상태로 돌려놔야 한다
    - 트랜잭션 **롤백** : 작업이 **중단되면 앞에서 처리한 SQL 작업을 취소**하는 것
    - 트랜잭션 **커밋** : 모든 SQL 수행 작업이 **마무리 되어 작업을 확정**하는 것

### JDBC 트랜잭션의 트랜잭션 경계설정

- **트랜잭션 경계설정** : **트랜잭션이 시작되고 끝나는(롤백, 커밋) 지점을 설정**하는 것
- **JDBC의 트랜잭션 설정**
    - 하나의 **Connection을 가져와서 사용하다가 닫는 사이에 발생**한다
    - 기본 설정 : DB 작업을 수행한 직후에 **자동으로 커밋** → 여러 개의 DB 작업을 모아서 트랜잭션을 만드는 기능이 꺼져 있음
        - 트랜잭션 **시작 : Connection.setAutoCommit(false)**
        - 트랜잭션 **끝 : Connection.commit(), Connection.rollback()**
    - 하나의 Conneciton이 만들어지고, 닫히는 범위 안에 존재한다 : **로컬 트랜잭션**

### JdbcTemplate - JDBC 트랜잭션

- JdbcTemplate의 템플릿 메소드를 사용하면, 내부에서 Connection 오브젝트를 가져오고 ~ 작업을 마치고 Conneciton을 닫고 메소드를 빠져나오는 과정이 진행된다
    
    → 템플릿 메소드 호출 → 한 개의 DB 커넥션 생성, 닫힘 → **템플릿 메소드마다 하나씩 독립적인 트랜잭션이 생성**된다
    
    → DAO로 데이터 액세스를 분리해놓았을 경우에는 **DAO 메소드를 호출할 때마다 새로운 트랜잭션이 만들어진다 :** 트랜잭션은 Connection 오브젝트 안에서 만들어지고, DAO 메소드에서는 매 번 DB Connection을 생성하기 때문
    
- 해결방법 : UserService와 UserDao의 분리 수준을 유지한 채로 트랜잭션을 적용하려면, **트랜잭션의 경계설정 작업을 UserService로 옮겨야 한다**
→  **upgradeLevel() 메소드 시작 = 트랜잭션 시작 / upgradeLevel() 메소드 끝 = 트랜잭션 종료**
    
    → **트랜잭션 경계설정을  upgradeLevel() 내부에서 수행**해야 한다
    
    ```java
    public void upgradeLevel() throws Exception {
    	// (1) DB Connection 생성
    	// (2) 트랜잭션 시장
    	try {
    		// (3) DAO 메소드 호출
    		// (4) 트랜잭션 커밋
    	} catch(Exception e) {
    		//(5) 트랜잭션 롤백
    	}
    	finally {
    		// (6) DB Connection 종료
    	}
    }
    ```
    

- **해당 방식의 문제점**
1. JdbcTemplate을 사용할 수 없어 **JDBC API를 직접 활용하는 방식의 단점**이 그대로 따라옴
2. DAO, UserService 메소드들에 **Connection 파라미터가 추가**되어야 한다
    
    : 트랜잭션이 필요한 작업에 참여하는 메소드들은 전부 Connection을 주고 받으며 수행되어야 하기 때문
    
3. Connection 파라미터가 추가되면, DAO는 **데이터 액세스 기술에 종속**된다
    
    : 데이터 액세스 기술에 따라 Conneciton이 달라지기 때문
    

---

## 트랜잭션 동기화

- 스프링이 제안하는 트랜잭션 동기화 : 트랜잭션을 시작하기 위해 만든 **Connection 오브젝트를 저장소에 보관 → 이후에 저장된 Connection을 가져다가 사용**하는 방식
- 작업 스레드마다 독립적으로 Connction을 관리함 → 멀티스레드 환경에서도 충돌음
- DAO가 사용하는 **JDBCTemplate이 트랜잭션 동기화 방식을 사용**하게 하자

<aside>
💡

작동 순서

1. UserService가 **Connection 생성**, **트랜잭션 동기화 저장소에 Connection 저장**
2. **트랜잭션 시작 : Connection.setAutoCommit()**
3. DAO 기능 사용 : **JdbcTemplate 메소드 호출**
    1. 현재 시작된 트랜잭션을 가진 **Connection 오브젝트의 존재 확인**
    2. **저장해둔 Connection 받아서 작업 수행**
    3. 트랜잭션을 **종료하기 전까지는, 계속 같은 Connection을 받아와서 사용**하고, Connection을 닫지 않는다
4. 종료
    1. 작업이 정상적으로 끝난 경우, Connection.**commit()** 호출
    2. 예외상황이 발생한 경우, Connection.**rollback()** 호출
    3. **Connection 오브젝트 제거**
</aside>

```java
public void upgradeLevel() throws Exception {
	// 1. 트랜잭션 동기화 저장소 초기화
	TransactionSynchronizationManager.initSynchronization();
	// 2. DB Connection 생성 + 트랜잭션 동기화
	Connection c = DataSourceUtils.getConnection(dataSource);
	// 3. 트랜잭션작
	c.setAutoCommit();
	try {
		List<User> users = dao.getAll();
		for(User user: users) {
			if(canupgrade(user) upgradeLevel(user);
			
			// 4-a. 트랜잭션 밋
			c.commit();
		}
	} catch(Exception e) {
		// 4-b. 트랜잭션 롤백
		c.rollback();
	}
	finally {
		// 4-c. connection 닫기, 동기화 종료
		DataConnectionUtils.releaseConnection(c, dataSource);
		TransactionSynchronizationManager.unbindResource(this.dataSource);
		TransactionSynchronizationManager.clearSynchronization();
	}
}
```

- **JdbcTemplate과 트랜잭션 동기화**
    - 동기화 저장소에 등록된 트랜잭션이 **없는 경우 : 직접 Connection 생성**, 트랜잭션 시작해서 JDBC 작업 진행
    - 트랜잭션 **동기화가 시작되어 있는 경우** : 새로 만드는 대신, **저장소에 있는 Connction 사용**
    
    → **트랜잭션 적용 여부에 맞춰 UserDao 코드를 수정할 필요가 없다**
    

- **JTA : Java Transaction API**
    - 로컬 트랜잭션의 한계 : 한 개의 트랜잭션 안에서 여러 개의 DB 로의 작업을 수행할 수 없다
        - Connection 방식
    - **글로벌 트랜잭션** 방식 : 별도의 **트랜잭션 매니저**를 통해 여러 개의 DB가 참여하는 작업을 하나의 트랜잭션으로 만들고, 트랜잭션 기능을 지원하는 다른 서비스도 트랜잭션에 참여할 수 있다
    - 즉, **하나 이상의 DB가 참여하는 트랜잭션을 만들기 위해서는 JTA를 사용해야 한다**

 **< 새로운 문제점 : 기술과 환경에 또 다시 종속된다 >** 

- UserService에서 **트랜잭션 경계 설정 코드에 의해 기술과 환경에 또 다시 종속**되는 문제가 발생한다
    - c.rollback(), c.commit() : Jdbc에 종속적이다 : UserDAO 인터페이스를 사이에 두고 있지만, **UserDaoJdbc에 의존**하고 있다
- 추상화 필요 : 트랜잭션의 경계설정을 담당하는 코드는 일정한 패턴을 갖는 유사한 구조이므로, **공통점을 찾아 뽑아내는 추상화**를 진행해야 한다

### 스프링의 트랜잭션 서비스 추상화 : PlatformTransactionManager

- 스프링이 트랜잭션 경계설정을 위해 제공하는 추상 인터페이스

```java
public class UserService {
	// 추상 인터페이스를 DI 받아 사용
	// 구현 기술이 변경되면, 구현체를 바꿔서 DI 하면 됨
	// 자신이 사용할 구체적인 클래스를 스스로 결정하고, 생성하지 않는다
	private PlatformTransactionManager transactionManager;
	
	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}
	
	public void upgradeLevel() throws Exception {
	// 트랜잭션 시작, 트랜잭션 동기화
	TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition);
	try {
		List<User> users = dao.getAll();
		for(User user: users) {
			if(canupgrade(user) upgradeLevel(user);
			
			// 트랜잭션 커밋
			this.transactionManager.commit();
		}
	} catch(Exception e) {
		// 트랜잭션 롤백
		this.transactionManager.rollback();
	}
	
}
```

- PlatformTransactionManager 의 구현체가 UserService에서 드러나지 않고, DI 받을 수 있게 한다
    - 주입되는 기술이 변경되면 클래스를 변경해주면 됨
- PlatformTransactionManager.getTransaction : Connection 가져오고, 트랜잭션 시작

---

## 서비스 추상화와 단일책임원칙

### 수직, 수평 계층구조와 의존관계

- **수평적** 분리 : **내용, 담당하는 코드의 기능적인 관심**에 따라 분리
    - UserDao / UserService
        - **UserDao** : 데이터 액세스 로직
        - **UserService** : 사용자 관리 업무의 비즈니스 로직
        - 두 관심사의 분리, 인터페이스와 DI를 통해 낮은 결합도로 분리
        → 서로 영향을 주지 않고 독립적으로 확장될 수 있다
- **수직적** 분리 : 아예 **다른 계층**의 특성을 갖는 코드를 분리
    - **애플리케이션의 비즈니스 로직 / 로우레벨의 트랜잭션 기술**
    - UserService / 트랜잭션 기술 : PlatformTransactionManager를 통한 추상화 계층을 사이에 두고 낮은 결합도로 분리
    → **애플리케이션 로직**이 **로우레벨의 기술**에서 **독립**할 수 있게 해준다

### 단일책임원칙

- 책임이 여러 개다 = 코드가 수정되는 이유가 여러 가지이다
- 단일책임원칙 : 하나의 모듈은 하나의 책임만 가져야 한다
- 장점 : 변경이 필요할 때 수정이 필요한 대상이 명확해진다 → 실수가 줄어든다
- 인터페이스와 DI를 도입해 모듈 간 결합도를 낮춰 서로의 변경이 영향을 주지 않도록 하고 / 변경이 단일 책임에 집중되도록 응집도를 높이고 / 적절한 책임과 관심이 다른 코드를 분리하고 / 애플리케이션 로직과 기술, 환경을 분리하는 코드를 짜야 한다
    - 이를 위한 핵심적인 도구가 스프링이 제공하는 DI이다

---

## JavaMail 서비스의 추상화

- 변경 요구사항 : 레벨이 변경될 때 메일을 전송하는 기능을 추가하자
- 자바 메일 발송 표준 기술인 JavaMail을 사용

```java
private void sendUpgradeEMail(User user) {
	Properties props = new Properties();
	props.put("mail.smtp.host", "mail.ksug.org");
	Session s = Session.getInstance(props, null);
	
	MimeMessage message = new MimeMessage(s);
	try{
		// message.setFrom, addRecipient, setSubject, ..
		Transport.send(message);
	} catch(AddressException e) {
		// throw ..
	} catch (MessagingException e) {
		// throw ..
	} catch ( UnsupportedEncodingException e) {
		//throw ..
	}
}

```

### 자바 메일 서비스의 테스트

- **메일 발송은 부하가 매우 큰 작업 → 테스트 마다 진짜 메일을 발송하는 것은 서버에 큰 부담**을 줄 수 있다
- 또한 메일 수신 여부를 직접 확인하는 것도 불가능 하다
    - 테스트 가능한 메일 서버까지만 잘 전송되는지 확인하고
    - **테스트 메일 서버를 이용해 전송 요청까지만 확인**하고, 실제로 발송은 되지 않도록 하는 방식
- 또 다른 문제점
    - **JavaMail API에는 인터페이스로 만들어져 구현을 바꿀 수 있는 것이 없다**
        - Session,  Transport : 메일 전송을 위해 필요한 오브젝트
        인터페이스가 아닌 클래스이며 +  생성자가 private + 스태틱 팩토리 메소드를 통해 오브젝트를 만드는 방법밖에 없고 + 상속이 불가능한 final 클래스이다
        - 따라서 테스트용으로 바꿔치기 하는 것은 불가능하다
        - 이처럼 **테스트 하기 힘든 구조인 API를 테스트하기 좋게 스프링이 JavaMail에 대한 추상화 기능을 제공**한다

**< MailSender > : 메일 발송 서비스 추상화의 핵심 인터페이스**

- `void send(SimpleMailMessage simpleMessage) thows MailException;` : 메일 전송 메소드
- JavaMailSenderImple 클래스 오브젝트를 주입받아 사용한다

```java
public class UserService {
	private MailSender mailSender;
	public void setMailSender(MailSender mailSender) {this.mailSender = mailSender;}
	
	public void sendUpgradeMail(User user) {
		SimpleMailMessage mailMessage = new SimpleMailMessge();
		// mailMessage.setTo, setFrom, setText, ...
		
		this.mailSender.send(mailMessage);
	}
}
```

**실제 메일 발송을 하지 않는 테스트 메일 발송 오브젝트**

```java
public class DummyMailSender implements MailSender {
	public void send(SimpleMailMessage mailMessage) throws MailException {}
	public void send(SimpleMailMessage[] mailMessage) throws MailException {}
}
```

- 하는 일이 없는 인터페이스
- JavaMailSenderImple 대신 DummyMailSender로 의존성을 주입 해주기

> 서비스 추상화
> 
> - 유사한 기능이지만, 사용 방법이 다른 **로우 레벨의 다양한 기술에 대해 추상 인터페이스와 일관성 있는 접근 방법을 제공**해주는 방식 : 트랜잭션
> - **테스트를 어렵게 하는 API를 사용하기 쉽도록** 지원 : JavaMail

---

### 목 오브젝트

**테스트 대역** : 테스트용으로 사용되는 특별한 오브젝트, 테스트 대상인 오브젝트의 의존 오브젝트

- UserDao - DataSource / UserService - MailSender
- 테스트 대상 오브젝트에게 테스트 환경을 만들어 주어 빠르게, 자주 테스트를 실행할 수 있도록 사용하는 오브젝트
- **테스트 스텁 :** 대상 오브젝트의 의존 객체로 존재하며 **테스트 동안 코드가 정상적으로 수행**될 수 있도록 간접적으로 돕는 것
    - 파라미터로 전달되기보다, 테스트 코드 내부에서 간접적으로 사용된다 → 의존 오브젝트를 DI를 통해 테스트 스텁으로 변경해둬야 다
    - DummyMailSender
- **목 오브젝트 :** 테스트 과정에 적극적으로 참여하여 테스트 대상의 **간접적인 출력결과를 검증하여** 테스트 대상 오브젝트 - 의존 오브젝트 사이에서 일어나는 일을 검증하는데 활용할 수 있는 테스트용 오브젝트
    - 간접 테스트에 사용 : 테스트 오브젝트와 간접적으로 주고 받는 정보를 보존 → 검증
    - 예시
    
    ```java
    static class MockMailSender implements MailSender {
    	private List<String> requests = new ArrayList<String>();
    	public List<String> getRequests {return requests;}
    	
    	public void send(SimpleMailMessage mailMessage) throws MailException {
    		requests.add(mailMessage.getTo()[0]);
    	}
    	public void send(SimpleMailMessage[] mailMessage) throws MailException {}
    	
    }
    ```
    
    - 메일 발송 기능에 더불어 발송대상(테스트 대상 오브젝트 - 의존 오브젝트에 간접적인 출력값 ) 또한 확인할 수 있게 됐다