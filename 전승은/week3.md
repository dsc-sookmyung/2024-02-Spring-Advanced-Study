# 3장 템플릿
  * [3.1 다시 보는 초난감 DAO](#31-다시-보는-초난감-dao)
  * [3.2 변하는 것과 변하지 않는 것](#32-변하는-것과-변하지-않는-것)
  * [3.3 JDBC 전략 패턴의 최적화](#33-jdbc-전략-패턴의-최적화)
  * [3.4 컨텍스트와 DI](#34-컨텍스트와-di)
  * [3.5 템플릿과 콜백](#35-템플릿과-콜백)
  * [3.6 스프링의 JdbcTemplate](#36-스프링의-jdbctemplate)
  * [🌿기억에 남는 부분](#🌿기억에-남는-부분)


-----
</br>

- 템플릿 : 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법이다.

## 3.1 다시 보는 초난감 DAO
### 3.1.1 예외처리 기능을 갖춘 DAO
JDBC 코드에서 어떤 이유로든 예외가 발생했을 경우에도 사용한 리소스를 반드시 반환해야함
- JDBC 수정 기능의 예외처리 코드
```java
public void deleteAll() throws SQLException {
		Connection c = dataSource.getConnection();
	
		// 여기서 예외가 발생하면 밑에 close()가 실행되지 않아 리소스가 반환되지 않음
		PreparedStatement ps = c.prepareStatement("delete from users");
		ps.executeUpdate();

		ps.close();
		c.close();
	}	
```
서버에서는 제한된 개수의 DB 커넥션을 만들어서 재사용 -> 위에처럼 반환되지 못한 Connection이 계속 쌓이면 리소스가 모자라서 서버 중단
▶️ try/catch/finally 사용
```java
public void deleteAll() throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;
		// 예외가 발생할 수 있는 코드
		try {
			c = dataSource.getConnection();
			ps = c.preparedStatement("delete from users);
			ps.executeUpdate();
		} catch (SQLException e) {	// 예외처리
			throw e;
		} finally { // 예외가 발생해도 실행
			if (ps != null) { 
				try { 
					ps.close(); // close()도 SQLException 발생할 수 있음
					} catch (SQLException e) {} }
			if (c != null) { try {c.close(); } catch (SQLException e) {} }
		}
	}	
```
일시적인 서버 문제, 네트워크 문제 등 -> c, ps 모두 null
PreparedStatement를 생성하다가 오류 -> c는 null이 아님, ps는 null
-> ps, c 둘다 null 인지 확인

-  JDBC 조회 기능의 예외처리
ResultSet 추가
 try 안에 아래 구문을 넣고, rs.close()도 위에 처럼 try로 예외처리
		 
		ResultSet rs = ps.executeQuery();
		rs.next();
		int count = rs.getInt(1);

## 3.2 변하는 것과 변하지 않는 것
### 3.2.1 JDBC try/catch/finally 코드의 문제점
try/catch/finally 구문이 2중 중첩에 메소드마다 반복됨
복붙하다가 빼먹은 곳이 있다면.. -> 서버 다운
### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용
1. 변하는 부분과 변하지 않는 부분 확인
	변하는 부분 : 			``ps = c.preparedStatement("delete from users);``
	변하지 않는 부분을 재사용
2. 메소드 추출
	변하는 부분을 추출
	``ps = makeStatement(c);``
-> 재사용되지 않는 부분이라 활용도가 낮음
3. 템플릿 메소드 패턴의 적용
	변하지 않는 부분은 슈퍼클래스, 변하는 부분은 추상메소드로 정의해서 서브클래스에서 오버라이드
	문제점 : 메소드가 4개일 경우 서브클래스 4개가 필요, 컴파일 시점에 관계가 이미 결정되어있음
4. 전략 패턴의 적용
	오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 전략 패턴
	개방 폐쇄 원칙을 잘 지킴
	- 전략 부분을 외부 기능(인터페이스)로 만들어두고 메소드를 통해 PreparedStatement 생성 전략을 호출해줌
	- 컨텍스트 내에서 만들어둔 DB 커넥션을 전달 (파라미터로)
	- StatementStrategy 인터페이스 생성 - 이걸 implement한 DeleteAllStatement 클래스 - UserDao에서 오브젝트로 생성
	- 문제점 : DeleteAllStatement를 사용하도록 고정되어있음

- DI 적용을 위한 클라이언트/컨텍스트 분리
클라이언트가 컨텍스트가 어떤 전략을 사용하게 할 것인가를 결정

클라이언트 코드인 StatementStarategy와 컨텍스트에 해당하는 코드를 독립시킴
인터페이스를 컨텍스트 메소드 파라미터로 지정
```java
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;

		try {
			c = dataSource.getConnection();

			ps = stmt.makePreparedStatement(c);
		
			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
			if (c != null) { try {c.close(); } catch (SQLException e) {} }
		}
	}
```
deleteAll()은 클라이언트가 됨 -> 전략 오브젝트를 만들고 컨텍스트를 호출
```java
public void deleteAll() throws SQLException {
	// 전략 오브젝트 생성
	StatementStrategy st = new DeleteAllStatement();
	// 컨텍스트 호출
	jdbcContextWithStatementStrategy(st);				
}
```


## 3.3 JDBC 전략 패턴의 최적화
컨텍스트 : preparedStatement  실행
전략 : PreparedStatement 생성, 클라이언트(DAO메소드)가 함
### 3.3.1 전략 클래스의 추가 정보
add()에 적용. deleteAll()과 같이 StatementStrategy를 implement 하는 AddStatement 클래스 생성해서 전략 부분 옮김
- add()는 deleteAll()과 달리 user 정보가 필요함 -> 클라이언트로부터 User 타입 오브젝트를 받을 수 있도록 AddStatement의 생성자를 만듦.

### 3.3.2 전략과 클라이언트의 동거
문제점 : DAO 메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 함. User같은 전달받아야 할 정보가 있는 경우 이를 위해 인스턴스 변수와 생성자 만들어야 함.
- 로컬 클래스
 StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의 (add() 메소드 안에 정의)
 

> 중첩 클래스 종류
>> 스태틱 클래스 : 독립적으로 오브젝트로 만들어질 수 있음
>> 내부 클래스 : 자신이 정의된 클래스의 오브젝트 안에서만 만들어질 수 있음
>>>멤버 내부 클래스 : 멤버 필드처럼 오브젝트 레벨에 정의됨
>>> 로컬 클래스 : 메소드 레벨에 정의 (메소드 안에 정의), 선언된 메소드 안에서만 사용, 로컬 변수에 직접 접근 가능
>>> 익명 내부 클래스 : 이름을 갖지 않음

```java
public void add(final User user) throws SQLException {
	class AddStatement implements StatementStrategy {			
		public PreparedStatement makePreparedStatement(Connection c)
				throws SQLException {
			PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
			ps.setString(1, user.getId());
			ps.setString(2, user.getName());
			ps.setString(3, user.getPassword());
						
			return ps;
		}
	}
	
	StatementStrategy st = new AddStatement();			
	jdbcContextWithStatementStrategy(st);
}
```

- 익명 내부 클래스
AddStatement를 익명 내부 클래스로 만듦 
	- 선언과 동시에 오브젝트를 생성
	- 클래스 자신의 타입을 가질 수 없음
	- 구현한 인터페이스 타입의 변수에만 저장할 수 있음

단 한번만 사용할테니 jdbcContextWithStatementStrategy() 메소드의 파라미터에서 바로 생성
```java
public void add(final User user) throws SQLException {
// 원래는 StatementStrategy st = new StatementStrategy() {
		jdbcContextWithStatementStrategy( new StatementStrategy() {			
					public PreparedStatement makePreparedStatement(Connection c)
					throws SQLException {
						PreparedStatement ps = 
							c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
						ps.setString(1, user.getId());
						ps.setString(2, user.getName());
						ps.setString(3, user.getPassword());
						
						return ps;
					}
				}
		);
	}
```
## 3.4 컨텍스트와 DI
### 3.4.1 JdbcContext의 분리
jdbc의 일반적인 작접 흐름을 담고 있는 jdbcContextWithStatementStrategy()을 USerDao 클래스 밖으로 독립 시킴
- 클래스 분리
```java
public class JdbcContext {
	private DataSource dataSource;
	
	// DI 받을 수 있게 준비
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
	
	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;

		try {
			c = dataSource.getConnection();

			ps = stmt.makePreparedStatement(c);
		
			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
			if (c != null) { try {c.close(); } catch (SQLException e) {} }
		}
	}
}
```
UserDao에서 JdbcContext를 DI 받아서 사용
- 빈 의존관계 변경
DI는 인터페이스를 사이에 두고 의존 클래스를 바꿔서 사용하도록 하는게 목적, 하지만 JdbcContext는 클래스(구현 방법이 바뀔 가능성이 없어서 인터페이스로 만들지 않음)
기존에 userDao 빈이 dateSource 빈을 직접 의존하는거에서 jdbcContext 빈을 사이에 끼는 걸로 바뀜
-> applicationContext.xml 파일 변경

### 3.4.2 JdbcContext의 특별한 DI
UserDao와 JdbcContext는 클래스 레벨에서 의존관계가 결정됨
- 스프링 빈으로 DI
IoC 관점에서 jdbcContext를 스프링을 이용해 UserDao 객체에서 사용하게 주입했다는 건 DI의 기본을 따르고 있음

 ❓JdbcContext와 UserDao를 DI 구조로 만들어야 하는 이유
 1. JdbcContext가 싱글톤 빈이 되기 때문 (변경되는 상태정보 없음, 싱글톤으로 등록되어 여러 오브젝트에서 공유해 사용하는 것이 이상적)
 2. JdbcContext가 DI를 통해 다른 빈(dataSource)에 의존하고 있기 때문에 스프링 빈으로 등록되어야 함
 
 JdbcContext는 UserDao와 아주 긴밀한 관계를 가지고 강하게 결합되어있음 (항상 같이 사용) -> 위의 두 장점을 위해 인터페이스 없이 강력한 결합을 허용하며 DI
 - 코드를 이용하는 수동 DI
	 - UserDao 내부에서 직접 DI를 적용하는 방법
	 - 싱글톤은 포기, 대신 Dao마다 하나의 JdbcContext 오브젝트를 갖고 있게 함.
	 - UserDao가 JdbcContext의 제어권을 갖고 생성과 초기화를 함.
	 - JdbcContext가 빈이 아니라 dataSource와 DI 불가능 -> UserDao가 임시로 DI 컨테이너처럼 동작하게 해 DI도 맡김
	 
	 1. 설정파일에서 JdbcContext 빈 제거
	 UserDao의 JdbcContext 프로퍼티 제거, DataSource 타입 프로퍼티만 갖도록 함.
	 2. ```java
		private JdbcContext jdbcContext;
			
		public void setDataSource(DataSource dataSource) {	// 수정자 메소드면서 jdbc에 대한 생성, DI 작업 동시에 함 
			this.jdbcContext = new JdbcContext();
			this.jdbcContext.setDataSource(dataSource);

			this.dataSource = dataSource;
		}
		```
장점 : 긴밀한 관계를 갖는 DAO 클래스와 JdbcContext를 따로 빈으로 분리하지 않고 내부에서 직접 만들어 사용하면서도 다른 오브젝트에 대한 DI를 적용할 수 있음
JdbcContext가 UserDao 내부에서 만들어지고 사용되면서 그 관계를 외부에 드러내지 않음

단점 : JdbcContext를 여러 오브젝트가 사용하더라도 싱글톤으로 만들 수 없고, DI 작업을 위한 부가적인 코드가 필요

## 3.5 템플릿과 콜백
템플릿/콜백 패턴 : 전략 패턴의 컨텍스트를 템플릿, 엑명 내부 클래스로 만들어지는 오브젝트를 콜백이라 부름

> 콜백 : 특정 로직을 담은 메소드(콜백)을 다른 오브젝트의 메소드에 전달하기 위해 사용한다.
### 3.5.1 템플릿/콜백의 동작원리
- 콜백은 일반적으로 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다.
- 템플릿/콜백의 특징
	- 콜백 인터페이스의 메소드(makePreparedStatement() )에는 보통 템플릿(workWithStatementStrategy() )의 작업 흐름 중에 만들어지는 컨텍스트 정보(Connection 오브젝트)를 전달 받을 때 사용되는 파라미터가 있다.

	1. 클라이언트가 콜백 오브젝트를 생성하고 참조할 정보 제공
	2. 클라이언트가 템플릿의 메소드를 호출하며 파라미터로 콜백 전달 (메소드 수준의 DI)
	3. 템플릿은 작업을 진해아다 내부에서 생성한 참조 정보를 가지고 콜백 오브젝트의 메소드를 호출
	4. 콜백은 1과 2에서 받은 정보로 작업을 수행하고 결과를 템플릿에게 전달
	5. 템플릿은 콜백이 준 정보를 사용해서 작업을 마저 수행

	기존 DI와의 차이점
	- 매번 메소드 단위로 사용할 오브젝트를 새롭게 전달받음 (기존은 인스턴스 변수를 만들고 수정자 메소드로 받아서 사용)
	- 클라이언트 메소드 내의 정보를 직접 참조
	- 클라이언트와 콜백이 강하게 결합됨

### 3.5.2 편리한 콜백의 재활용
문제점 : 익명 내부 클래스를 사용하기 때문에 상대적으로 코드를 작성하고 읽기가 조금 불편하다
- 콜백의 분리와 재활용
변하는 SQL 문장만 파라미터(final 로 선언)로 받고 메소드 내용 전체를 분리해 별도의 메소드로 만들기
모든 고정된 SQL 문을 실행하는 DAO 메소드는 이걸로 대체 가능
- 콜백과 템플릿의 결합
다른 DAO에서도 사용할 수 있도록 JdbcContext로 옮김 (템플릿은 workWithStatementStrategy() 메소드라 콜백 생성과 템플릿 호출이 담긴 excuteSQL()을 JdbcContext 클래스로 옮겨도 괜찮음)

```java
// JdbcContext.java
public void executeSql(final String query) throws SQLException {
		workWithStatementStrategy(
			new StatementStrategy() {
				public PreparedStatement makePreparedStatement(Connection c)
						throws SQLException {
					return c.prepareStatement(query);
				}
			}
		);
	}	

// UserDao.java
public void deleteAll() throws SQLException {
		this.jdbcContext.executeSql("delete from users");
	}
```

구체적인 구현과 내부의 전략 패턴, 코드에 의한 DI, 익명 내부 클래스 등의 기술은 최대한 감춰두고, 외부에는 꼭 필요한 기능을 제공하는 단순한 메소드만 노출해주는 것

### 3.5.3 템플릿/콜백의 응용
1. 고정된 작업 흐름을 갖고 자주 반복되는 코드 -> 중복된 코드를 메소드로 분리
2. 그 중 일부 작업을 필요에 따라 바꿔 사용해야 한다 -> 인터페이스를 사이에 두고 분리해서 전략 패턴 적용&DI
3. 바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어진다 -> 템플릿/콜백 패턴
- 테스트와 try/catch/finally
파일을 하나 열어서 모든 라인 숫자를 더한 합을 돌려주는 코드
try/catch/finally를 적용하여 예외처리를 해줌
try로 파일을 읽어 합을 해주는 코드를 담고,
catch로 예외처리
finally로 파일이 열려서 BufferedReader 오브젝트가 생성되었다면 닫게 해줌

- 중복의 제거와 템플릿/콜백 설계
기능을 확장해야 한다면 템플릿/콜백 패턴 적용
	- 템플릿이 BufferedReader를 만들어서 콜백에게 전달, 콜백이 각 라인을 읽어서 처리, 최종결과 템플릿에 돌려줌
	- 콜백을 인터페이스로 만든 후 calcSum() 메소드 안에서 내부 익명 클래스로 생성
	- @Before 메소드로 사용할 클래스의 오브젝트와 파일 이름을 정의해놓음

- 템플릿/콜백의 재설계
곱과 합을 각각 계산하는 메소드가 중복되는 코드(각 라인을 읽고 계산하는 while 문)가 있어 그 부분을 추가해 새 템플릿 lineReadTemplate을 만들고 변하는 부분(계산 내용)을 콜백 인터페이스(LineCallback)을 만듦
- 제네릭스를 이용한 콜백 인터페이스
결과의 타입을 다양하게 가져가고 싶다면, 타입 파라미터라는 개념을 도입한 제네릭스 사용
```java
// LineCallback.java
public interface LineCallback<T> {
	T doSomethingWithLine(String line, T value);
}

// Calulator.java
public <T> T lineReadTemplate(String filepath, LineCallback<T> callback, T initVal) throws IOException {
		BufferedReader br = null;
		try {
			br = new BufferedReader(new FileReader(filepath));
			T res = initVal;
			String line = null;
			while((line = br.readLine()) != null) {
				res = callback.doSomethingWithLine(line, res);
			}
			return res;
		}
		catch(IOException e) {
			System.out.println(e.getMessage());
			throw e;
		}
		finally {
			if (br != null) {
				try { br.close(); } 
				catch(IOException e) { System.out.println(e.getMessage()); }
			}
		}
	}
	
public Integer calcSum(String filepath) throws IOException {
		LineCallback<Integer> sumCallback = 
			new LineCallback<Integer>() {
				public Integer doSomethingWithLine(String line, Integer value) {
					return value + Integer.valueOf(line);
				}};
		return lineReadTemplate(filepath, sumCallback, 0);
	}
	
public String concatenate(String filepath) throws IOException {
		LineCallback<String> concatenateCallback = 
			new LineCallback<String>() {
			public String doSomethingWithLine(String line, String value) {
				return value + line;
			}};
			return lineReadTemplate(filepath, concatenateCallback, "");
	}	
```
T로 선언하면 어떤 타입인지 콜백을 정의할때 지정해주면 됨. (concatenate는 String, calcSum은 Integer)

## 3.6 스프링의 JdbcTemplate
jdbcContext를 스프링에서 제공하는 JDBC 전용 템플릿/콜백인 JdbcTemplate으로 바꿈
```java
public class UserDao {
	private DataSource dataSource;
		
	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);
			
		this.dataSource = dataSource;
	}
```
### 3.6.1 update()
첫번째 콜백. StatementStrategy 인터페이스의 makePreparedStatement() 메소드 -> PreparedStatementCreator 인터페이스의 createPreparedStatement() 메소드, 템플릿 메소드 : update()
```java
public void deleteAll() throws SQLException {
		this.jdbcTemplate.update("delete from users");
	}

public void add(final User user) throws SQLException {
	this.jdbcTemplate.update("insert into users(id, name, password) values(?,?,?)",
						user.getId(), user.getName(), user.getPassword());
	}
```

add()도 손쉽게 바꿀 수 있음

### 3.6.2 queryForInt()
getCount()는 SQL 쿼리를 실행하고 ResultSet을 통해 결과 값을 가져옴
-> PreparedStatementCreator 콜백과 ResultSetExtractor 콜백(템플릿이 제공하는 ResultSet을 이용해 원하는 값을 추출해서 전달, 제네릭스 타입 파라미터 사용)을 파라미터로 받는 query() 메소드 사용
이중 콜백을 아래처럼 바꿈
```java
public int getCount() {
		return this.jdbcTemplate.queryForInt("select count(*) from users");
	}
```
### 3.6.3 queryForObject()
get()에 템플릿 적용의 관건 : ResultSet의 결과를 User 오브젝트를 만들어 프로퍼티에 넣어줘야함
RowMapper 콜백 사용 (ResultSet의 로우 하나를 매핑받고 필요한 정보를 추출해서 리턴)
```java
public User get(String id) {
		return this.jdbcTemplate.queryForObject("select * from users where id = ?",
				new Object[] {id}, // 쿼리의 ? 부분, 뒤에 파라미터있어서 가변인자 못함
				new RowMapper<User>() {
					public User mapRow(ResultSet rs, int rowNum)
							throws SQLException {
						User user = new User();
						user.setId(rs.getString("id"));
						user.setName(rs.getString("name"));
						user.setPassword(rs.getString("password"));
						return user;
					}
				});
	} 
```
메소드 자체가 RowMapper 콜백을 호출할때 ResultSet이 한 줄만 있는 걸 기대하고, next()를 이용하여 첫번째 줄을 가리키고 있음
예외처리를 해주지 않아도 메소드 안에 예외가 있음

### 3.6.4 query()
- 기능 정의와 테스트 작성
getAll()은 테이블의 모든 로우를 가져옴 -> User 오브젝트 컬렉션으로 만듦
1. User타입의 오브젝트인 user1,2,3을 DB에 등록하고 getAll() 호출 
2. List< User> 타입으로 결과를 돌려받음
3. 리스트의 크기는 3, user1,2,3과 동등한 오브젝트가 id 순으로 담겨져 있어야함
```java
@Test
	public void getAll()  {
		dao.deleteAll();
		
		List<User> users0 = dao.getAll();
		assertThat(users0.size(), is(0));
		
		dao.add(user1); // Id: gyumee
		List<User> users1 = dao.getAll();
		assertThat(users1.size(), is(1));
		checkSameUser(user1, users1.get(0));
		
		dao.add(user2); // Id: leegw700
		List<User> users2 = dao.getAll();
		assertThat(users2.size(), is(2));
		checkSameUser(user1, users2.get(0));  
		checkSameUser(user2, users2.get(1));
		
		dao.add(user3); // Id: bumjin
		List<User> users3 = dao.getAll();
		assertThat(users3.size(), is(3));
		checkSameUser(user3, users3.get(0));  
		checkSameUser(user1, users3.get(1));  
		checkSameUser(user2, users3.get(2));  
	}

	private void checkSameUser(User user1, User user2) {
		assertThat(user1.getId(), is(user2.getId()));
		assertThat(user1.getName(), is(user2.getName()));
		assertThat(user1.getPassword(), is(user2.getPassword()));
	}
```
- query() 템플릿을 이용하는 getAll() 구현
query() 메소드는 여러 개의 로우가 결과로 나오는 일반적인 경우에 쓸 수 있음
리턴 타입은 List< T>, RowMapper< T> 콜백 ㅇ오브젝트에서 결정됨
```java
public List<User> getAll() {
		return this.jdbcTemplate.query("select * from users order by id",// 두번째 파라미터도 추가 가능
				new RowMapper<User>() {
					public User mapRow(ResultSet rs, int rowNum)
							throws SQLException {
						User user = new User();
						user.setId(rs.getString("id"));
						user.setName(rs.getString("name"));
						user.setPassword(rs.getString("password"));
						return user;
					}
				});
	}
```
ResultSet의 모든 로우를 열람하면서 콜백 호출

- 테스트 보완
네거티브 테스트를 잘 만들어야한다. 결과가 하나도 없는 경우에 어떻게 될지 예외처리를 해줘야함
-> query()는 결과가 없을 경우에 크기가 0인 List< T>를 돌려줌. 이걸 그대로 리턴하도록 만듦

### 3.6.5 재사용 가능한 콜백의 분리
- DI를 위한 코드 정리
UserDao의 모든 메소드가 JdbcTemplate을 이용  -> DataSource 인스턴스 변수 제거, 수정자메소드는 남겨둠

- 중복 제거
User용 RowMapper 콜백을 메소드에서 분리
``` java
private RowMapper<User> userMapper = 
		new RowMapper<User>() {
				public User mapRow(ResultSet rs, int rowNum) throws SQLException {
				User user = new User();
				user.setId(rs.getString("id"));
				user.setName(rs.getString("name"));
				user.setPassword(rs.getString("password"));
				return user;
			}
		};

public User get(String id) {
		return this.jdbcTemplate.queryForObject("select * from users where id = ?",
				new Object[] {id}, this.userMapper);
	} 
```
- 템플릿/콜백 패턴과 UserDao
UserDao와 JdbcTemplate은 관심사의 분리로 각각 높은 응집도를 갖고 서로는 낮은 결합도를 유지하고 있음. 다만, JdbcTemplate를 직접 유지한다는 점에서 결함
▶️ JdbcTemplate를 독립적인 빈으로 등록하고 JdbcOperations 인터페이스를 통해 DI 받아 사용하도록 만듦

이외의 보충하면 좋을 것들 
1. userMapper가 인스턴스로 설정되어 있고 변경되지 않음 -> UserMapper를 독립된 빈으로 만들고 XML 설정에 User 테이블 필드 이름과 User 오브젝트 프로퍼티의 매핑정보를 담음
2. Dao 메소드에서 사용하는 SQL 문장을 UserDao 코드가 아니라 외부 리소스에 담고 이를 읽어와 사용

# 🌿기억에 남는 부분
멤버 내부 클래스와 로컬 클래스가 잘 구분이 안돼서 따로 찾아봤다.
>내부 클래스 : 자신이 정의된 클래스의 오브젝트 안에서만 만들어질 수 있음
- 멤버 내부 클래스 : 멤버 필드처럼 오브젝트 레벨에 정의됨
```java
	class OuterClass {
    private String outerField = "외부 필드";

    // 멤버 내부 클래스
    class InnerClass {
        void display() {
            System.out.println("외부 필드: " + outerField);
        }
    }

    void createInnerInstance() {
        InnerClass inner = new InnerClass();
        inner.display();
    }
}

public class Main {
    public static void main(String[] args) {
        OuterClass outer = new OuterClass();
        outer.createInnerInstance();
    }
}
```
- 로컬 클래스 : 메소드 레벨에 정의 (메소드 안에 정의), 선언된 메소드 안에서만 사용, 로컬 변수에 직접 접근 가능
```java
class OuterClass {
    private String outerField = "외부 필드";

    void outerMethod() {
        // 메서드 안에서 정의된 로컬 클래스
        class LocalClass {
            void display() {
                System.out.println("외부 필드: " + outerField);
            }
        }

        // 로컬 클래스 인스턴스 생성 및 메서드 호출
        LocalClass local = new LocalClass();
        local.display();
    }
}

public class Main {
    public static void main(String[] args) {
        OuterClass outer = new OuterClass();
        outer.outerMethod();
    }
}
```

### **차이점**

1.  **선언 위치**
    
    -   **멤버 내부 클래스**는 외부 클래스의 **멤버로** 선언됩니다. 즉, 외부 클래스 안에서 바로 정의되며, 다른 멤버 변수나 메서드와 같은 수준에서 존재합니다.
    -   **로컬 클래스**는 외부 클래스의 메서드 안에서 **지역 변수처럼** 선언됩니다. 메서드 내에서만 사용할 수 있으며, 메서드가 실행될 때마다 새롭게 생성됩니다.
2.  **접근 범위**
    
    -   **멤버 내부 클래스**는 외부 클래스의 모든 멤버(필드와 메서드)에 자유롭게 접근할 수 있습니다.
    -   **로컬 클래스**도 외부 클래스의 멤버에 접근할 수 있지만, 주로 해당 메서드 내에서만 사용할 목적으로 정의됩니다.
3.  **사용 시기**
    
    -   **멤버 내부 클래스**는 외부 클래스의 인스턴스가 있을 때 언제든지 사용할 수 있습니다.
    -   **로컬 클래스**는 선언된 **메서드가 호출될 때만** 생성되고 사용할 수 있습니다. 메서드가 끝나면 로컬 클래스의 존재도 끝납니다.
4.  **생성 방식**
    
    -   **멤버 내부 클래스**는 외부 클래스의 인스턴스에서 멤버 변수처럼 생성할 수 있습니다.
    -   **로컬 클래스**는 해당 메서드 안에서만 인스턴스를 생성할 수 있습니다.

### 요약:

-   **멤버 내부 클래스**는 외부 클래스의 멤버로서 상시 접근 가능하며, 외부 클래스의 인스턴스와 함께 동작합니다.
-   **로컬 클래스**는 특정 메서드 내에서만 사용 가능하며, 그 메서드가 실행될 때에만 존재합니다.
