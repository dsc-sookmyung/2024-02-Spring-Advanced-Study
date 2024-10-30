# 4장 예외
## 4.1 사라진 SQLException
jdbcContext -> JdbcTemplate으로 바꾸며 deleteAll() 메서드에 SQLException가 사라졌다. 
### 4.1.1 초난감 예외처리
- 예외 블랙홀 
	- 예외를 잡고 아무 작업도 하지 않음
	- 예외가 발생하면 예외 메세지를 출력함 -> 콘솔 로그를 모니터링하지 않으면 처리되지 않음
	- 모든 예외는 적절하게 복구되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야함
- 무의미하고 무책임한 throws
발생하는 모든 예외를 무조건 던져버리면 코드에서 유의미한 정보를 얻을 수 없음

### 4.1.2 예외의 종류와 특징
> 체크 예외 : 명시저인 처리가 필요한 예외

자바에서 throw를 통해 발생시킬 수 있는 예외
- Error : 주로 자바 VM에서 발생 -> 애플리케이션 코드에서 잡아도 할 수 있는게 없음
- Exception과 체크 예외
	> Exception : java.lang.Exception 클래스와 그 서브클래스
	>> 체크 예외 : Exception의 서브클래스면서 RuntimeException을 상속하지 않음. 일반적인 예외. 초기 자바는 모든 예외에 이걸 적용 -> 비판받음
	>> 언체크 예외 : RuntimeException을 상속함
- RuntimeException과 언체크/런타임 예외
	프로그램의 오류가 있을 때 발생하도록 의도된 것들. 개발자가 부주의해서 발생 -> 굳이 catch나 throws를 사용하지 않아도 됨.

### 4.1.3 예외처리 방법
- 예외 복구 : 예외 상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓는 것 (재시도, 다른 방법 시도 등)
체크 예외들은 이렇게 예외를 어떤 식으로든 복구 가능할 때 사용
- 예외처리 회피 : 예외처리를 자신을 호출한 쪽으로 던져버리는 것
	- throws 문으로 선언해서 예외가 발새하면 알아서 던져지게 하거나 
catch 문으로 일단 예외를 잡은 후에 로그를 남기고 다시 예외를 던짐
	- 콜백 오브젝트는 예외를 회피하고 템플릿으로 던짐
- 예외 전환 : 예외를 적절한 예외로 전환해서 메소드 밖으로 던짐
 목적 1. 의미를 분명하게 해줄 수 있는 예외로 바꿔주기 위해
	- API가 발생하는 기술적인 로우 레벨을 상황에 적합한 의미를 가진 예외로 변경하는 것
	- 보통 전환하는 예외에 원래 발생한 예외를 담아서 중첩예외로 만드는 것이 좋다. 중첩예외는 getCause() 메소드를 이용해서 처음 발생한 예외가 무엇인지 확인할 수 있다.
		- 생성자나 initCause() 메소드로 근본 원인이 되는 예외를 넣어줌
 목적 2. 예외를 처리하기 쉽고 단순하게 만들기 위해 포장하는 것
	 - 중첩 예외를 이용. 체크 예외를 언체크 예외인 런타임 예외로 바꾸는 경우에 사용
	 - EJBException같은 비즈니스 로직으로 볼 때 의미 있는 예외이거나 복구가능한 예외가 아닐때 런타임 예외로 던져 롤백

### 4.1.4 예외처리 전략
- 런타임 예외의 보편화
초기와 다르게 자바 서버 환경으로 바뀌면서 예외 처리를 강제하는 체크 예외의 활용도가 떨어짐 -> API가 발생시키는 예외가 항상 복구할 수 있는 예외가 아니라면 언체크 예외로 바꾸고 런타임 예외를 던지게 함.
- add() 메소드의 예외처리
	-  SQLException은 대부분 복구 불가능한 예외이므로 런타임 예외로 포장해 던져버려서 그 밖의 메소드들이 신경 쓰지 않게 해주는 편이 낫다.
	 - 어디에서든 DupicatedUserIdException을 잡아서 처리할 수 있다면 굳이 체크 예외로 만들지 않고 런타임 예외로 만드는 게 낫다. 대신 add() 메소드는 명시적으로 DupicatedUserIdException를 던진다고 선언해야 한다.
	 - DupicatedUserIdException 클래스 생성
		 필요하면 언제든 잡아서 처리할 수 있도록 별도의 예외로 정의하기는 하지만, 필요없다면 신경쓰지 않아도 되도록 RumtimeException을 상속한 런타임 예외로 만든다.
```java
public class DuplicateUserldException extends RuntimeException { 
    public DuplicateUserIdException(Throwable cause) {
        super(cause); 
    }
}
```
```java
public void add() throws DuplicateUserldException { 
    try {
        // JDBC를 이용해 user 정보를 애에 추가하는 코드 또는
        // 그런 기능이 있는 다른 SQLException을 던지는 메소드를 호출하는 코드
    }
    catch (SQLException e) {
        if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
            throw new DuplicateUserldException(e); // 예외 전환
        else
            throw new RuntimeException(e); // 에외 포장
    }
```

- 애플리케이션 예외
 시스템 또는 외부의 예외상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고, 반드시 catch 해서 무엇인가 조치를 취하도록 요구하는 예외
ex) 사용자가 요청한 금액을 은행계좌에서 출금하는 기능을 가진 메소드가 잔고 부족 상황이 발생했을 때 체크 예외를 던짐

### 4.1.5 SQLException은 어떻게 됐나?
Dao에 존재하는 SQLException
- SQLException이 복구가 가능한 예외인가? 거의 다 복구 불가능함
▶ 언체크/런타임 예외로 가능한 빨리 전환
- JdbcTemplate 템플릿과 콜백 안에서 발생하는 모든 SQLException을 런타임 예외인 DataAccexxException으로 포장해서 던져줌 -> JdbcTemplaate을 사용하는 UserDao 메소드에선 꼭 필요한 경우에만 런타임 예외인 DataAccessException을 잡아서 처리하고 그 외의 경우엔 무시해도 됨
▶DAO 메소드에서 SQLException이 모두 사라짐

## 4.2 예외 전환 
### 4.2.1 	JDBC의 한계
JDBC는 자바를 이용해 DB에 접근하는 방법을 추상화된 API 형태로 정의해놓고, 각 DB 업체가 jdbc 표준을 따라 만들어진 드라이버를 제공하게 해줌
하지만, db 종류에 상관없이 사용할 수 있는 데이터 액세스 코드를 작성하는 일은 쉽지 않다. 표준화된 jdbc api가 db 프로그램 개발 방법을 학습하는 부담은 확실히 줄여주지만 db를 자유롭게 변경해서 사용할 수 있는 유연한 코드를 보장해주지는 못한다. (예를 들어, 특정 데이터베이스에만 있는 기능이나 SQL 문법이 있을 경우, 이 기능을 사용한 코드는 다른 데이터베이스로 쉽게 이전할 수 없습니다.)

db 프로그램을 작성하는 데 두 가지 걸림돌
1. 비표준 SQL
대부분의 db는 표준을 따르지 않는 비표준 문법과 기능도 제공 -> 비표준 sql 문이 dao 코드에 들어감 -> dao가 특정 db에 종속적이게 됨
▶️ dao를 db 별로 만들어 사용하거나 sql을 외부에서 독립시켜서 바꿔 쓸 수 있게 함
2. 호환성이 없는 SQLException의 DB 에러 정보
DB마다 에러의 종류와 원인도 제각각 -> jdbc는 데이터 처리 중에 발생하는 다양한 예외를 그냥 SQLException 하나에 담음 / SQLException의 getErrorCode()로 가져올 수 있는 DB 에러 코드는 DB 별로 모두 다름
▶️ SQLException은 예외가 발생했을 때의 db 상태를 담은 sql 상태정보를 부가적으로 제공 (getSQLState()로) 
그런데, db의 jdbc 드라이버에서 SQLException을 담을 상태 코드를 정확하게 만들어주지 않아 무용지물

### 4.2.2 DB 에러 코드 매핑을 통한 전환
db별 에러 코드를 참고해서 발생한 예외의 원인이 무엇인지 해석해주는 기능을 만들어야 함
-> 스프링은 db별 에러 코드를 분류해서 스프링이 정의한 예외 클래스와 매핑해놓은 에러 코드 매핑정보 테이블을 만들어두고 이를 이용함
-> SQLException 대신 DataAccessException이라는 런타임 예외와 그의 서브클래스를 이용
중복 키 에러를 따로 분류해서(코드가 같은 지 if 문으로 처리) 예외처리를 해줬던 메소드를 스프링의 jdbcTemplate을 이용하도록 변경
jdbcTemplate은 체크 예외인 SQLException을 런타임 예외인 DataAccessException 계층 구조의 예외로 포장해줌

체크 예외로 하고 싶다면
throws DuplicateKeyException 대신 
```java
public void add() throws [체크예외클래스] {	
	try {
	
	} catch(DuplicateKeyException e) {
		throw new [체크예외클래스] (e);
	}
}
```

### 4.2.3 DAO 인터페이스와 DataAccessException 계층구조
DataAccessException은 의미가 같은 예외라면 데이터 액세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어준다.
- DAO 인터페이스와 구현의 분리
DAO의 사용 기술과 구현 코드는 전략 패턴과 DI를 통해서 DAO를 사용하는 클라이언트에게 감출 수 있지만, 매소드 선언에 나타나는 예외정보가 문제가 될 수 있다.
```java
public interface UserDao {
	public void add(User user);
}
```
🔼 이상적인 DAO 인터페이스
문제점 : 인터페이스의 메소드 선언에는 없는 예외를 구현 클래스 메소드의 throws에 넣을 수 없음
-> 	``public void add(User user) throws SQLException; ``
-> jdbc가 아닌 데이터 액세스 기술로 dao 구현을 전환하면 사용할 수 없음

jdbc 외의 데이터 액세스 기술은 런타임 예외를 사용
jdbc -> dao 메소드 내에서 런타임 예외로 포장해서 던져줄 수 있음 -> jdbc를 이용한 dao에서 모든 SQLException을 런타임 예외로 포장해주기만 하면 위의 이상적인 인터페이스로 할 수 있음

문제점 : 데이터 액세스 기술이 달라지면 같은 상황에서도 다른 종류의 예외가 던져진다는 점
▶️ dao를 사용하는 클라이언트 입장에서는 dao의 사용 기술에 따라서 예외 처리 방법이 달라져야 함 -> 의존적이게 됨

- 데이터 액세스 예외 추상화와 DataAccessException 계층구조
	- 스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해서 DataAccessException 계층구조 안에 정리해놓았다.
	- 스프링의 DataAccessException은 일부 기술에서만 공통적으로 나타나는 예외를 포함해서 데이터 액세스 기술에서 발생 가능한 대부분의 예외를 계층구조로 분류해놓았다.
	ex) 낙관적인 락킹 기능 -> DataAccessException의 서브클래스로 통일 시킬 수 있음. ORM 기술이 아니지만 JDBC 등을 이용해 직접 낙관적인 락킹 기능을 구현했다고 해도 서브클래스의 슈퍼클래스를 상속해서 새로운 예외클래스를 정의해 사용하면 됨.
	- DataAccessException 계층구조에는 템플릿 메소드나 DAO 메소드에서 직접 활용할 수 있는 예외도 정의되어 있음 -> JDBC에서는 예외가 발생하지 않지만 JdbcTemplate에서 볼 때는 예외상황일 때 사용할 수 있도록 예외가 정의되어있음

JDBCTemplate과 같이 스프링의 데이터 액세스 지원 기술을 이용해 DAO를 만들면 사용 기술에 독립적인 일관성있는 예외를 던질 수 있음

### 4.2.4 기술에 독립적인 UserDao 만들기
- 인터페이스 적용
```java
public interface UserDao {
	void add(User user);
	User get(String id);
	List<User> getAll();
	void deleteAll();
	int getCount();
	// setDataSource()는 UserDao의 구현 방법에 따라 변경될 수 있어서 인터페잇에 추가하면 안됨
}
```
UserDao 클래스 이름 변경 : ``public class UserDaoJdbc implements UserDao { ``

빈 클래스 변경
```xml
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
		<property name="dataSource" ref="dataSource" />
</bean> 
```

 - 테스트 보완
UserDaoTest는 DAO의 기능을 검증하는 것이 목적이지 JDBC를 이용한 구현에 관심이 있는 게 아니다. 그러니 UserDao라는 변수 타입을 그대로 두고 스프링 빈을 인터페이스로 가져오도록 만드는 편이 낫다.
-> DI가 적용됨

UserDaoTest에 중복된 키를 가진 정보를 등록했을 때 어떤 예외가 발생하는지를 확인하기 위해 테스트 추가
@Test(expected=DataAccessException.class)
// 예외 발생하면 성공, expected 부분 빼고 돌려보면 실패하고, 어느 예외 클래스가 던져졌는지 확인할 수 있음. DuplicateKeyException으로 바꾸고 해도 성공)

- DataAccessException 활용 시 주의사항
DuplicateKeyException은 jdbc를 이용하는 경우에만 발생
❓SQLException에 담긴 db의 에러 코드를 바로 해석하는 jdbc와 달리 jpa나 하버네이트, jdo 등에서는 각 기술이 재정의한 예외를 가져와 스프링이 최종적으로 DataAccessException으로 반환하는데, db의 에러 코드와 달리 이런 예외들은 세분화되어 있지 않기 때문이다.

	- DataAccessException이 기술에 상관없이 어느 정도 추상화된 공통 예외로 변환해주긴하지만 근본적인 한계 때문에 완벽하지 않음
	- dao에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면 DuplicatedUserIdException처럼 직접 예외를 정해두고, 각 DAO의 add() 메소드에서 좀 더 상세한 예외 전환을 해줄 필요가 있음

학습 테스트를 하나 더 만들어서 SQLException을 직접 해석해 DataAccessException으로 변환하는 코드의 사용법
-> db 에러 코드 이용
SQLException을 코드에서 직접 전환하고 싶다면 SQLExceptionTransLator 인터페이스를 구현한 클래스 중에서 SQLErrorCodeSQLExceptionTranslator 를사용하면 됨

```java
	@Test
	public void sqlExceptionTranslate() {
		dao.deleteAll();
		
		try {
		// 강제로 DuplicateKeyException 발생
			dao.add(user1);
			dao.add(user1);
		}
		catch(DuplicateKeyException ex) {
								// 중첩되어 있는 SQLException을 가져옴
			SQLException sqlEx = (SQLException)ex.getCause();
			SQLExceptionTranslator set = new SQLErrorCodeSQLExceptionTranslator(this.dataSource);	// 코드를 이용한 SQLException의 전환		
			
			DataAccessException transEx = set.translate(null, null, sqlEx); // 에러 메세지 만들 때 사용하는 정보이므로 null로 넣어도 됨
			assertThat(transEx, is(DuplicateKeyException.class));
		}
	}
}
```
