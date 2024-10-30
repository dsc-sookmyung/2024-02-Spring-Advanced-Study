

## 노션 정리본 링크 : https://merciful-marmot-54e.notion.site/3-1206873728d080fd8810fa1dfb003ad8?pvs=4


# 3주차 - 템플릿

<aside>
💡

**템플릿** : 자유롭게 변경되는 성질을 가진 부분 / **변경이 적어 일정한 패턴으로 유지**되는 특성을 가진 부분을 **독립**시켜 효과적으로 활용할 수 있게 하는 부분

</aside>

## 예외처리 기능을 갖춘 DAO - JDBC 수정기능의 예외처리코드

### 공유 리소스의 반환 - pool

**리소스 공유** 방식 : Connection, PrerparedStatement 같은 리소스는 **풀(pool) 방식**으로 운영된다.

미리 정해진 풀 안에서 제한된 수의 **리소스를 미리 생성**해두고, **필요할 때 할당받아 사용**하고, **반환하면 다시 풀에 넣는 방식**으로 운영한다.
요청이 많은 서버 환경에서는 **매번 새로 리소스를 생성하는 대신, 풀 방식으로 미리 만들어둔 리소스를 할당**받아 사용하는 것이 유리하다.
**주의할 점 : 사용할 리소스를 빠르게 반환**해야 한다 → 리소스 **고갈**의 문제가 발생할 수 있다

- **close()** : Connection, PreparedStatement 등의 close() 메소드는 이 **리소스를 풀에 반환**하는 역할을 한다

### 예외상황과 리소스 반환

- 제한된 리소스를 공유하는 서버에서는 어떤 예외상황에서도 리소스를 반환하도록 try/catch/finally 구문을 사용하여 예외처리를 적용해야 한다
- 예외마다 반환되지 못한 리소스가 쌓이면 리소스가 모자라다는 오류를 내며 서버가 중단 될 수 있다

```java
public void deleteAll() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	
	try {
		c = dataSource.getConneciton();
		ps = c.prepareStatement("delete from users");
		ps.executeUpdate();
	} catch(SQLException e) {
		throw e;
	} finally {
		if(ps!=null) {
			try {
				ps.close();
			} catch (SQLException e) {}
		}
		if(c!=null) {
			try {
				c.close();
			} catch (SQLException e) {}
		}
	}
}
```

- **null 확인**
    
    예외 발생 시점에 따라 close를 수행해야 하는 변수가 달라지기 때문에 **리소스 변수마다 null을 확인하고 close를 호출**한다
    
- close에도 **try**
    
    **close()에서도 SQLException이 발생**할 수 있으므로 try/catch로 처리해야 한다
    
- close **순서**
    
    자원에 대한 **close는 만들어진 순서의 반대로 실행**해야 한다
    생성 : Connection → PreparedStatement → ResultSet
    반환 : ResultSet → PreparedStatement → Connection
    

---

## 변하는 것과 변하지 않는 것

- 현재 코드의 문제점
    1. try/catch/finally 문이 모든 메소드마다 반복된다
    2. 반복되는 부분을 작성하다가 실수해도 컴파일 에러가 나지 않으며, 실수했어도 테스트로 잡기 어렵다
- 해결방법 : **“변하지않으며+많은곳에서중복되는코드”**를 확장되고 변하는 코드로부터 **분리하여 재사용** 할 수 있도록 하자

### 디자인패턴 적용 - 템플릿메소드 패턴

- **변하지 않는 부분은 슈퍼클래스** + **변하는 부분은 추상메소드로 정의** 해두기
    
    → 변하는 부분을 **서브클래스에서 오버라이드** 해서 새롭게 정의해 사용
    

```java
// 추상클래스 UserDao 안의 추상 메소드 makeStatement
abstart protected PreparedStatement makeStatement(Connection c) thrwos SQLException;
```

> 단점
> 
> - DAO **로직마다 새로운 클래스**를 만들어야 한다
>     - UserDao의 makeStatement → 서브클래스 : UserDaoAdd, UserDaoGet, UserDaoDeleteAll
> - 확장구조가 **클래스 레벨에서 컴파일 시점에 이미 고정**된다

## 디자인패턴 적용 - 전략 패턴

- 오브젝트를 둘로 분리 → **클래스 레벨에서는 인터페이스를 통해서만 의존**하도록 하기
    
    → 추상화된 **인터페이스를 통해 확장을 위임**하기
    

### 전략패턴 1 - DI 적용을 위한 클라이언트/컨텍스트 분리

1. **클라이언트** : **컨텍스트를 사용**하는 클라이언트가 **전략 오브젝트 선택, 생성**해서 **컨텍스트에 전략 제공**
    - 전략 오브젝트를 생성하고 전달하며 컨텍스트(변하지 않는) 메소드를 호출(사용)해야 한다
        - 즉 **전략 오브젝트 생성+컨텍스트로 전달**을 담당하며
        - **컨텍스트를 사용**하기도 한다
    
    ```java
    public void deletAll() throws SQLException {
    	//전략 오브젝트 선택, 생성
    	StatementStrategy stmt = new DeleteAllStatement();
    	//전략 오브젝트를 전달하며 컨텍스트 메소드 사용
    	jdbcContextWithStatementStrategy(st);
    ```
    
2. **컨텍스트 : 변하지 않는 부분**의 **분리, 재사용** : Context - contextMethod()
    - DB 커넥션 가져오기 + 외부기능 **(PreparedStatement 생성 전략) 호출**하기 + 실행하기 + 예외처리 + 리소스 close - **메소드로 분리**
    
    ```java
    //클라이언트가 컨텍스트를 호출할 때 넘겨주는 전략 오브젝트(파라미터 StatementStrategy)
    public void jdbcContextWithStatementStrategy(StatementStrategy stmt) 
    throws SQLException {
    	Connection c = null;
    	PreparedStatement ps = null;
    	try {
    		c = dataSource.getConnection();
    		
    		//클라이언트로부터 전달받은 전략 오브젝트를 이용
    		ps = statement.makePreparedStatement(c);
    		
    		ps.executeUpdate();
    	} catch(SQLException e) {
    		throw e;
    	} finally {
    		if(ps!=null) {
    			try {ps.close();} catch (SQLException e) {}
    		}
    		if(c!=null) {
    			try {c.close();} catch (SQLException e) {}
    		}
    	}
    }
    ```
    
3. **전략 : 변화하는, 확장되는 부분** : Strategy - algorithmMethod() : **인터페이스**
    - PreparedStatement 생성 전략
    
    ```java
    public interface StatementStrategy {
    	// 1. Connection을 인자로 받아서 2. 필요에 따른 ps 오브젝트 돌려받기
    	PreparedStatement makePreparedStatement(Connection c) throws SQLException;
    }
    ```
    
4. **구현 클래스 : 전략 인터페이스를 상속받아 실제 구현**하는 부분
    
    ```java
    public class DeleteAllStatement implements StatementStrategy {
    	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
    		PreparedStatement ps = c.prepareStatement("delete from users");
    		return ps;
    	}
    }
    ```
    

- **마이크로 DI**
    - DI의 일반적인 구성 : 의존관계에 있는 두 개의 오브젝트 / 의존관계 오브젝트의 관계를 동적으로 설정해주는 팩토리(DI 컨테이너) / 이것을 사용하는 클라이언트
    - 위의 코드는 클라이언트+팩토리가 같은 클래스로 결합된 형태
    - DI의 장점을 순화해서 IoC 컨테이너의 도움 없이 작은 단위의 코드와 메소드 사이에서 DI를 적용하는 경우를 마이크로 DI라고 한다

> **단점**
> 
> - DAO **로직마다 새로운 구현 클래스**를 만들어야 한다
>     - 템플릿 메소드 패턴에서의 단점이 그대로 존재한다
> - **전략 오브젝트에 전달할 부가적인 정보**가 필요한 경우 생성자를 통해 이를 전달받는데, 이를 위한 추가적인 **생성자와 인스턴스 변수**가 필요해진다

### 전략패턴 2 - 익명 내부 클래스 사용하기

- 클래스 개수 줄이기 +생성자, 인스턴스 변수 사용 불편함 → 익명 내부 클래스로 해결하자

- 익명내부클래스의 특징
    - **클래스 선언 + 오브젝트 생성**이 결합된 형태
    - 상속할 클래스나 구현할 인터페이스를 생성자 대신 사용한다
    - 클래스 자신의 타입을 갖지 않고, **인터페이스 타입 변수에 저장**할 수 있다
- 클라이언트 코드 - add()

```java
public void add(final User user) throws SQLException {
	//컨텍스트 메소드 사용
	//컨텍스트 메소드 호출하며 전략 오브젝트 넘기기 : 익명 내부 클래스를 사용하여 
	//선언과 동시에 전략 오브젝트 생성하며 파라미터로 넘김
	jdbcContextWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
			PreparedStatement ps = c.prepareStatement
			("insert into users(id, name, password, values(?,?,?,?)");
			ps.setString(1, user.getId());
			ps.setString(2, user.getWord());
			ps.setString(3, user.getPassword());
			return ps;
			}
		}
	);
	
	/*
		기존코드와 비교
		1. 전략 오브젝트 선택, 생성: 외부 클래스의 오브젝트를 생성
		StatementStrategy stmt = new addStatement(user);
		2. 전략 오브젝트를 전달하며 컨텍스트 메소드 사용
		jdbcContextWithStatementStrategy(st);
	*/
		
```

### 전략 패턴 3 - 클래스 분리

- 내부의 메소드로 분리했던 컨텍스트를 클래스로 분리해보자

- 컨텍스트 클래스  : JdbcContext - workWithStatement

```java
public class JdbcContext {
	//DataSource 타입 빈을 DI 받아야 함
	private DataSource dataSource;
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
	
	//전략 오브젝트 전달받아 중복되는 작업을 수행하는 부분
	public void workWithStatement(StrategyStatement stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;
		try {
			c = dataSource.getConnection();
		
			//클라이언트로부터 전달받은 전략 오브젝트를 이용
			ps = statement.makePreparedStatement(c);
		
			ps.executeUpdate();
		} catch(SQLException e) {
			throw e;
		} finally {
			if(ps!=null) {try {ps.close();} catch (SQLException e) {}}
			if(c!=null) {try {c.close();} catch (SQLException e) {}}
		}
	}
		
```

- 클라이언트 코드

```java
public class UserDao {
	//..
	// jdbcContext 빈 DI 받도록
	private JdbcContext jdbcContext;
	public void setJdbcContext(JdbcContext jdbcContext) {
		this.jdbcContext = jdbcContext;
	}
	
	public void add(final User user) throws SQLException {
		this.jdbcContxt.worWithStrategy(
			new StatementStrategy() {...}
		);
	}
	public void deleteAll(final User user) throws SQLException {
		this.jdbcContxt.worWithStrategy(
			new StatementStrategy() {...}
		);
	}
```

---

### DI 설정 1 - 인터페이스를 사용하지 않는 DI

- 기존 Spring의 DI
    - **인터페이스**를 사이에 둬서 클래스 레벨에서는 의존관계가 고정되지 않고, **런타임에 동적인 의존관계**가 설정된다 : 느슨한 연결관계
- **JdbcContext - UserDao** : 인터페이스를 적용하지 않고 DI를 적용했다
    - 객체의 생**성과 관계설정에 대한 제어권한을 외부로 위임**했다는 IoC 개념 포함
    - JdbcContext를 UserDao 객체에서 사용하게 주입했다는 관점에서 DI
- 인터페이스를 사용하지 않으면서, Jdbc를 **DI구조로 만드는 경우의 장점**
    1. **싱글톤**
        - 컨텍스트 메소드를 제공하는 서비스 오브젝트 역할이므로, **싱글톤으로 등록되어 여러 오브젝트에서 공유**하며 사용하는 것이 이상적이다
        - 변경되는 상태정보를 갖고 있지 않다
    2. **DI 의존성**
        - DataSource 오브젝트를 DI 받아서 사용한다
        - 오브젝트를 **주입 받는 쪽도 스프링 빈으로 등록**돼야 하기 때문에, 다른 빈을 DI 받기 위해서스프링 빈으로 등록돼야 한다
- 단점
    - DI의 근본적인 원칙에서 어긋나는, 구체적인 클래스와의 관계가 설정에 노출된다

### DI 설정 2 - 코드를 이용한 수동 DI

- 스프링 빈을 사용하지 않고, **UserDao 내부에서 직접 JdbcContext를 DI** 하기
- Dao마다 하나의 JdbcContext를 갖고 있게 하자
- DI : JdbcContext의 생성뿐만 아니라 **의존 오브젝트에 대한 제어권**도 UserDao가 갖는다
    - JdbcContext가 의존하는 **DataSource 오브젝트의 DI도 UserDao가 대신 수행**하고
    - JdbcContext를 만들고 **초기화하는 과정에서만 사용**하고 버리는 방식
    
    > **1. DataSource 빈을 주입**받고
    2. **JdbcContext 오브젝트를 생성**하고
    3. DI 받은 DataSource 오브젝트를 **JdbcContext의 수정자 메소드로 주입**하기
    > 

```java
public class UserDao  {
	private JdbcContext jdbcContext;
	public void setJdbcContext(DataSource dataSource) {
		this.jdbcContext= new JdbcContext();
		this.jdbcContext.setDataSource(dataSource);
		this.dataSource = dataSource;
}
```

- 장점
    - JdbcContext - UserDao는 긴밀한 관계를 갖는 오브젝트들이다 : 두 클래스를 어색하게 따로 빈으로 분리하지 않고 내부에서 직접 만들어 사용할 수 있다
    - 관계를 외부에 드러내지 않고, DI를 수행하며 전략을 외부에 감출 수 있다
- 단점
    - 싱글톤 관리 불가능, DI작업 위한 부가적인 코드 필요

---

## 템플릿과 콜백

<aside>
💡

**< 템플릿 >** : **반복되는 일정한 패턴**이 있는 부분
**고정된 틀 안에 바꿀 수 있는 부분을 넣어서 사용**하는 경우, 전략 패턴의 컨텍스트
고정된 작업 흐름을 가진 코드를 **재사용**한다

**< 콜백 >** : 실행되는 것을 목적으로 **다른 오브젝트의 메소드에 전달**되는 오브젝트
**특정 로직을 담은** **메소드를 실행시키기 위해** **템플릿 안에서 호출**되는 것을 목적으로 함
**메소드가 담긴 오브젝트를 파라미터로 전달**한다

</aside>

- 작업흐름
    - 클라이언트
        - 콜백 오브젝트 생성
        - **템플릿 메소드 호출, 콜백 오브젝트 전달**
        - 콜백이 참조할 정보 제공
    - 템플릿
        - **정해진 작업 흐름** 수행
        - **콜백 오브젝트의 메소드 호출**, 콜백 반환결과 받아 다시 작업 수행
    - 콜백
        - 클라이언트의 정보, 템플릿의 참조정보 이용해 작업 수행
        - 수행 결과를 템플릿에 반환
    - 순서 - with JdbcContext
        - 클라이언트 : 템플릿 호출 + 콜백 생성, 전달 → 템플릿 
          UserDao.add() → 익명 콜백 오브젝트 생성, JdbcContext.workWithStatementStrategy() 호출하며 전달
            - 템플릿 : 워크플로우시작, 참조정보 생성
              try/catch/finally workflow, Connection 생성
            - 템플릿 : 콜백 호출 + 참조정보 전달 → 콜백
                - 콜백 : 클라이언트, 템플릿의 변수 참조 해 작업 수행
                    Connection, user 정보 사용해 PreparedStatement 생성
                - 콜백 : 작업 결과 반환 → 템플릿
                  PreparedStatement 반환
            - 템플릿 : 워크플로우 마무리, 작업 결과 반환 → 클라이언트
              PreparedStatement 받아서 ps 수행  
              finally : 리소스 close 작업 수행

- 특징
    - 단일 메소드의 인터페이스를 사용한다
        - 콜백 : **단일메소드인터페이스**를 구현한 **익명내부클래스** 형태
        - 템플릿의 작업 흐름 안에서 **특정 기능을 위해 한 번 호출**되는 용도이기 때문
    - 콜백 인터페이스의 메소드 - **파라미터** : 템플릿의 작업 흐름 안에서 만들어지는 **컨텍스트 정보를 콜백에 넘겨주기 위해 사용**
        - JdbcContext - workWithStatementStrategy (템플릿) → makePreparedStatement (콜백) : Connection을 오브젝트로 전달
    - **“클라이언트의 템플릿 메소드 호출 + 템플릿이 사용할 콜백 오브젝트의 메소드 주입”**이 **동시에** 발생
    - 템플릿은 **매번 메소드 단위로 사용할 오브젝트를 새로 전달**받는다
    - 콜백 오브젝트는 자신을 생성한 클라이언트 내의 정보를 직접 참조한다

---

### 콜백의 재활용 : 1. 콜백의 분리 → 2. 콜백+템플릿 결합

- **콜백의 분리 : 콜백 내부에서도 중복되는 부분을 분리하고 재사용 하여** 복잡한 익명내부클래스의 사용을 최소화 해보자
    - 현재 콜백 메소드 내부에서 유일하게 **변하는 부분은 String으로 지정하는 SQL문** 뿐이다
    - 변화 O : 변하는 SQL문만 파라미터로 넘기기
    - **변화 X : 템플릿 호출 + 콜백 클래스 정의, 오브젝트 생성 - executeSql() 메소드로 분리**

```java
public class UserDao {
	//...
	public void deleteAll() {
		//변하는 부분 : 쿼리문
		executeUpdate("delete from user");
	}
	
	//변하지 않는 부분 : 템플릿 호출, 콜백 정의+생성
	//변하는 부분인 SQL 문장만 파라미터로 받아서 사용하게 했다
	private void executeUpdate(final String query) throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() {
				public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
					return c.prepareStatement(query);
				}
			}
		);
	}
}
```

- **콜백+템플릿 결합** : **재사용 가능한 콜백을 담은 메소드는 DAO가 공유**하여 사용할 수 있도록 **템플릿 클래스로 옮기자**
    - 재사용 가능한 콜백 메소드를 포함하는 executeSql()을 공유하여 재사용하기 위해 UserDao 밖인 템플릿 클래스 JdbcContext 클래스로 옮기자

```java
public class JdbcContext {
	//...
	
	//클라이언트 : 템플릿 호출 + 콜백 전달
	public void executeSql(final String query) throws SQLException {
		workWithStatementStrategy(
				new StatementStrategy() {
					public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
						return c.prepareStatement(query);
					}
				}
			);
	}
	
	//템플릿 메소드
	public void workWithStatement(StrategyStatement stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;
		try {
			c = dataSource.getConnection();
		
			//클라이언트로부터 전달받은 전략 오브젝트를 이용
			ps = statement.makePreparedStatement(c);
		
			ps.executeUpdate();
		} catch(SQLException e) {
			throw e;
		} finally {
			if(ps!=null) {try {ps.close();} catch (SQLException e) {}}
			if(c!=null) {try {c.close();} catch (SQLException e) {}}
		}
	}
}
```

```java
public class UserDao {
	//...
	public void deleteAll() throws SQLException {
		this.jdbcContext.executeSql("delete from users");
	}
```

- 결과
    - JdbcContext 안에 클라이언트, 템플릿, 콜백이 공존하며 동작
    - 하나의 목적을 위해 연관되어 동작하는 응집력 강한 코드들 → 한 군데에 모여있는 것이 유리하다
    - 구체적 구현, 내부의 전략패턴, DI, 익명내부클래스 등은 JdbcContext 내부에 감춰두고 / 외부에는 필요한 기능을 제공하는 단순한 메소드만 노출 (deleteAll())

<aside>
💡

고정된 작업흐름의 중복코드를 처리하는 방법

1. 메소드로 분리
2. 인터페이스를 두고 분리하여 전략패턴 적용 + DI로 의존관계 관리 : 일부 작업을 필요에 따라 바꿔 사용해야 한다면
3. 템플릿/콜백 패턴 : 바뀌는 부분이 동시에 여러 종류가 만들어질 수 있다면
- 전형적인 후보 : try/catch/finally
</aside>

---

### 템플릿/콜백의 설계

<aside>
💡

템플릿/콜백을 설계할 때 생각할 점

- 템플릿
    - 템플릿에 분리해서 담을 반복되는 작업 흐름
    - 콜백에게 전달해줄 내부 정보
    - 템플릿이 작업을 마친 뒤 클라이언트에게 전달할 내용
- 콜백
    - 반복되는 흐름을 제외한, 핵심 기능에 충실한 코드 정제
    - 콜백이 템플릿에게서 받을 정보 / 템플릿에게 반환할 정보
    - 이에 따른 콜백 인터페이스 정의하기
</aside>

- 흐름
    1. 위의 생각할 점을 고려하여 
        - 콜백 인터페이스 작성
        - 반복되는 부분을 템플릿 메소드로 분리하기
    2. 템플릿 메소드 내부에서 콜백 오브젝트를 받아 메소드 활용하기
    3. 클라이언트 부분에서 익명내부클래스를 통해 콜백 오브젝트 생성, 템플릿 호출하며 오브젝트 전달
    4. 콜백에서 중복되는 부분 더 분리하여 메소드로 넣는 작업 : 재설계
        - 한 번의 템플릿 메소드 호출에서 반복문으로 콜백을 여러 번 호출하기도 O

---

### 제네릭스를 이용한 콜백 인터페이스

- 콜백메소드를 **타입 파라미터**를 사용해 제네릭메소드로 만들기
    - **더 범용적인 타입**의 결과를 만들어내는 템플릿/콜백으로 사용할 수 있다

> **제네릭** : 클래스 내부에서 사용할 **타입을 외부에서 지정**하는, **타입을 변수화**한 것
- 타입 파라미터 : <>안에 식별자 기호를 지정해 파라미터화 하여 사용
> 

```java
// 콜백 메소드
public interface LineCallback<T> {
	T doSomethingWithLine(String line, T value);
}
```

- 타입 파라미터를 받는 제네릭 인터페이스 LineCallback
- 제네릭을 파라미터로 받고, 반환하는 제네릭 메소드 doSomethingWithline

```java
// 제네릭 인터페이스(콜백)을 사용하는 템플릿
// 전달받는 콜백 오브젝트를 타입 파라미터로정
public <T> lineReadTemplate(String filePath, LineCallback<T> callback, T initVal)
	throws SQLException {
		BufferedReader br = null;
		try {
			br = new BufferedReader(new FileReader(filePath));
			T res = initVal;
//..
```

---

## 스프링의 JdbcTemplate

- **템플릿/콜백을 활용**하여 **JDBC의 반복적이고 복잡한 코드(커넥션 열기, 자원 닫기, 예외 처리 등)**를 간소화하여 **간단한 메소드 호출만으로 사용 가능**하도록 만들어진 **JdbcTemplate**이 이미 존재한다

### < JdbcTemplate>

- **생성자 파라미터로 DataSource 주입 필요**
    
    ```java
    public class UserDao {
    	//...
    	private JdbcTemplate jdbcTemplate;
    	
    	public void setDataSource(DataSource dataSource) {
    		this.jdbcTemplate = new JdbcTemplate(dataSource);
    	}
    	//...
    }
    ```
    

- **update()**
    - SQL 연산을 통해 **데이터베이스를 갱신할 때(INSERT, DELETE, UPDATE) 사용**하는 메소드
    - **치환자(?)**를 가진 SQL로 PreparedStatement를 만들고, 제공하는 **파라미터를 순서대로 바인딩** 가능
    - 콜백
        - **PreparedStatementCreateor** - **createdPreparedStatement() :** Connection을 받아서 PreparedStatement를 생성해 반환
            - 파라미터 : Connection
            - 반환 : PreparedStatement
- **query()**
    - SQL을 실행하고 **정수 결과값을 받아올 때 사용**하는 메소드
    - 콜백
        - **PreparedStatementCreateor** - **createdPreparedStatement()**
        - **ResultSetExtractor - query()** : ResultSet으로 부터 값 추출
            - 파라미터 : ResultSet<T> : 제네릭스 : 다양한 타입의 값을 추출할 수 있기 때문에
            - 반환 : ResultSet
    - **반환 : List<T> : 여러 개의 로우**가 나오는 경우
- **queryForInt()**  → ResultSet<T> 활용 : **Integer 결과**를 가져오는 경우 사용
- **queryForObject()  → RowMapper** 콜백 활용 : ResultSet의 결과를 **오브젝트에 매핑**하는 경우
    - 콜백
        - ResultSetExtractor 대신 **RowMapper 사용** : **한 번에 한 개의 로우**를 매핑함
    - **빈 결과에 대한 예외처리** 이미 존재 : EmptyResultDataAccessException

- 예사 : query()를 사용해서 모든 User 조회 결과값을 받아 매핑하는 getAll()

```java
public List<User> getAll() {
	return this.jdbTemplate.query("select * from users order by id",
					new RowMapper<User>() {
							public User mapRow(ResultSet rs, int rowNum) throws SQLException {
								User user = new User();
								user.setId(rs.getString("id"));
								user.setName(rs.getString("name"));
								user.setPassword(rs.getPassword("password"));
								return user;
							}
					});
}
```

- ResultSet의 모든 로우를 열람하며 모든 로우마다 RowMapper 콜백을 호출함 → List<User> 반환
- RowMapper는 현재 로우의 내용을 User 오브젝트에 매핑해서 돌려준다