# 4주차 - 예외처리


### 노션링크 : https://merciful-marmot-54e.notion.site/4-1236873728d080fb99f7df8a16ee6912?pvs=4




- 예외처리를 할 때 반드시 지켜야할 원칙 : 모든 예외는 적절하게 복구되거나, 작업을 중단시키고 운영자에게 분명하게 통보되어야 한다.
그냥 화면에 에러를 출력하는 것은 올바른 예외처리가 아니다

## 예외의 종류

- **Error**
    - java.lang.**Error**의 서브클래스
    - 시스템에서 **비정상적인 예외상황**이 발생하는 경우 : 주로 자바 VM에서 발생
    - 애플리케이션 코드에서 잡지 않아도 됨
- **Exception**
    - java.lang.**Exception**의 서브클래스
    - 개발자들이 만든 **애플리케이션 코드**의 작업 중에 예외상황이 발생하는 경우
    - **언체크예외 : RuntimException**
        - **RuntimeException**(Exception의 서브클래스)을 상속한 클래스
        - 명시적인 예외처리를 **강제하지 않는다**
        - 프로그램에 **오류가 있을 때 발생**하도록 의도된, 개발자가 부주의한 경우 발생할 수 있는 에러들
        - **NullPointerException, IllegalArgumentException**, …
    - **체크예외**
        - Exception 중 언체크예외를 제외한 것들
        - 예외를 처리하는 코드가 **강제된다 : throws, catch**
            - 없으면 **컴파일 에러** 발생
        - **IOException, SQLException**

## 예외처리 방법

### 예외복구

- 예외상황을 파악해 문제를 해결해서 정상 상태로 돌려놓는 것
- 에러메시지를 던지기만 하는 것이 아닌, 애플리케이션의 설계된 흐름에 따라 진행되며 예외를 해결
- 대기 후 최대횟수만큼 재시도, IOException 발생 시 다른 파일로 이용 안내, …

### 예외처리 회피

- 예외처리를 자신이 담당하지 않고, 자신을 호출한 쪽으로 던져버리는 것
    - 예외를 회피하려면, 반드시 다른 곳에서 예외를 대신 처리할 수 있도록 던져줘야 한다
- 예외를 회피하는 것은 의도가 분명해야 한다
    - 자신을 호출한 쪽에서 예외를 다루는 것이 최선이라는 확신이 있어야 한다
    - 콜백/템플릿 : 콜백은 예외를 회피하여 자신을 호출한 템플릿에서 처리하도록 예외를 더닞ㄴ다
        - 긴밀한 연관관계에 있으며, 예외처리의 역할이 분명하게 분담되어 있어야 한다

### 예외 전환

- **적절한 예외로 전환**하여 메소드 밖으로 예외를 던진다
- 서비스 계층(비즈니스 로직을 수행하는)에서 에러를 해석하는 코드를 담는 것은 어색하다
    - 예외가 발생하는 특정 계층(DAO)에서 밖으로 예외를 던질 때는 의미가 분명한 예외로 전환하여 던지는 것이 바람직하다
- **목적**
    1. 로우레벨의 예외를 **더 의미 있고 추상화 된 예외**로 바꾸어 던지기 위해
    2. **복구 불가능한 체크 예외**를 **런타임 예외로 포장**하여 catch/throws를 줄이고, 다른 계층으로 넘어가지 않게 하기 위해
- **방법**
    1. **중첩예외**
    - 전환하는 예외에 원래 발생한 예외를 담아서 중첩예외를 만들기
    - 생성자 / initCause()로 생성 → getCause()로 확인
    
    ```java
    catch(SQLException e) {
    	//throw DuplicateUserIdException(e);
    	//throw DuplicateuserIdException().initCause(e);
    }
    ```
    
    1. **포장**
    - **체크예외 → 언체크예외**로 바꾸는 경우에 사용 : 발생하는 체크예외가 의미 있는 예외이거나 복구 가능한 예외가 아닌 경우
    - 런타임예외로 전환하여 클라이언트에서 일일히 잡거나 다시 던지지 않아도 되도록 변경
    - **어차피 복구가 불가능한 예외**인 경우, 무의미하게 throws로 넘기는 것보다 **런타임 예외로 포장하여 다른 계층으로 넘어가지 않게 하는 목적**

<aside>
💡

**SQLException은 어떻게 됐나?**
**JDBCTemplate의 예외처리 전략 : 예외전환**

- 템플릿, 콜백 안에서 발생하는 모든 **SQLException → DataAccessException(런타임 예외)**으로 포장해서 던진다
- 필요한 경우에만 DataAccessException을 잡아서 처리면 되고 / 아닌 경우에는 꼭 처리해야 할 의무는 없다
</aside>

---

### add() 메소드의 예외처리

JDBC 코드에서 SQLException이 발생하는 경우

- ID 중복이 원인이라면 좀 더 의미있는 DuplicatedUserIdException으로 전환하자
- 런타임 예외도 thorws로 선언하여 명시적으로 오류 가능성을 알릴 수 있으며 + 필요하다면 어디에서든 잡아서 처리할 수 있고 + 필요 없다면 신경 쓰지 않아도 되도록 → DuplicatedUserIdException도 런타임 예외로 전환하자

```java
// 런타임 예외로 전환 : RuntimeException 상속
public class DuplicatedUserIdException extends RuntimeException {
	// 중첩예외 생성용 생성자
	public DuplicatedUserIdException(Throwable casue) {
		super(cause);
	}
}
```

```java
public void add() throws DuplicatedUserIdException {
	try {
		//codes..
	} catch(SQLException e) {
		// 특별한 의미가 있는 예외인 경우 : 예외 전환
		if(e.getErrorCode()==MySqlErrorNumbers.ER_DUP_ENTRY)
			throw new DuplicatedUserIDException(e);
		else 
			// 그 외의 경우에도 복구 불가능한 경우에 언체크 예외로 전환 : 예외 포장
			throw new RuntimeException(e);
	}
}
```

---

### 애플리케이션 예외

특정한 상황에서 **애플리케이션 자체의 로직에 의해 의도적으로 발생**시켜고 → **반드시 catch 해서 조치**를 하는 경우

<aside>
💡

1. 정상 처리 / 예외 상황 : 서로 다른 종류의 리턴 값을 넘겨주기
    - 리턴 값을 결과 상태정보로 활용하여 예외 상황을 체크
    - 단점 : 전달값의 표준이 없음 → 의사소통 문제 발생 가능 / 잦은 조건문으로 복잡함
2. 정상 흐름은 그대로 두고, 예외 상황에서 비즈니스적인 의미를 띤 에외를 던지기
    - 예외가 발생할 수 있는 코드는 try 안에 / 예외에 대한 처리는 catch에 모아서
    - 체크 예외를 사용하자 : 잊지 않고 예외 처리에 대한 로직을 구현하도록 강제
</aside>

---

## 예외 전환

**JDBCTemplate : SQLException → DataAccessException : 예외전환**

> - 런타임 예외로 포장하고
- 로우레벨의 예외를 더 의미 있고 추상화 된 예외로 바꾸기 위해
> 

### JDBC의 한계

- JDBC의 목적
    - 자바로 DB에 접근하는 방법을 추상화 된 API 형태로 정의하고
    - 개발자는 API를 통해 DB 종류에 상관없이 일관된 방법으로 데이터 액세스 코드를 작성하고자 한다
    
    → 하지만 다음과 같은 한계에 의해 현실적인 어려움이 존재한다
    

1. **비표준 SQL**
    - 대부분의 DB는 표준을 따르지 않는 비표준 문법, 기능을 제공한다
    - 대용량 데이터 처리를 위한 최적화, 페이징 처리, 조건문, 특별한 기능, ..
    - 비표준 SQL에 의해 특정 DB에 종속적인 코드를 작성하게 된다
    - 해결방법
        - DAO를 DB 별로 만들어서 사용 /  SQL을 외부에서 독립시켜 바꿔 쓸 수 있게 하는 것
2. **호환성 없는 SQLException의 DB 에러정보**
    - DB마다 에러의 종류, 원인, 정의한 에러코드가 각각 다르다
    - JDBC는 데이터 처리 중에 발생한 에외를 전부 SQLException 하나에 담아버린다
    → 상세한 에러코드는 getErrorCode()로 받는다
    → 이런 에러코드도 벤더 별로 다르다
    - 예외 발생 시의 일관된 SQL 상태코드도 존재한다
    → getErrorState()로 받는다
    → 상태코드 또한 일관되고 정확한 값이 아니다
    - 따라서 DB 업체별로 유지하는 DB 전용 에러코드를 사용하는 것이 정확하다

---

### DB 에러코드 매핑을 통한 전환

> **DB 전용 에러코드**를 참고하여 **예외의 원인을 해석**하고
→ SQLException을 **의미가 분명히 드러나는 예외로 전환**하자
> 

<aside>
💡

**DataAccessException**
**org.springframework.dao.DataAccessException**

- **런타임 에러**이며
- 서브클래스로 **세분화된 예외 클래스**들을 정의하고 있다
- **특정 기술에 의존적이지 않은, 일관된 예외처리**를 위해 사용

따라서 스프링은 
예외클래스 - DB 별 에러코드 매핑정보 테이블을 두고

**DataAccessException의 계층구조 클래스 (더 세부적인) 중 하나로 예외를 포장해서 반환**해준다
: **의미가 같은 예외라면 데이터 액세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어 준다.**

</aside>

- 직접 정의한 예외를 발생시키고 싶은 경우엔, 반환받은 예외를 catch 안에서 직접 전환해주는 코드를 DAO 안에 넣으면 된다

```java
// 애플리케이션 레벨에서 의도적으로 직접 정의한 체크예외 발생 시킴
public void add() throws DuplicateUserIdException {
	try {
		// codes..
	} catch (DuplicateKeyException e) {
		// 원하는 체크예외로 전환
		// 원인예외를 중헙하자
		throw new DuplicatedUserIdException(e);
	}
}
```

## 기술에 독립적인 DAO 만들기

- DAO를 비즈니스 로직과 분리하여 사용하는 이유
    - 데이터 액세스 로직을 담은 코드를 성격이 다른 코드와 분리하기 위해
    - DAO를 사용하는 쪽에서 DAO가 내부에서 어떤 데이터 액세스 기술을 사용하는지 신경 쓰지 않도록 함
    - 따라서 인터페이스를 사용해 구체적인 클래스 정보, 구현 방법을 감추고 DI를 통해 제공되도록 하자
1. **인터페이스 적용**
    
    ```java
    public interface UserDao {
    	void add(User user);
    	User get(String id);
    	List<User> getAll();
    	void deleteAll();
    	int getCount();
    }
    
    //사용하는 데이터 액세스 기술에 따라 인터페이스를 구현
    // setDataSource는 여기서 구현 기술에 따라 알맞게 사용
    public class UserDaoJdbc implements UserDao {
    	//..
    }
    ```
    
2. **런타임 예외 전환**
    
    ```java
    public void add(User user) throws PersistentException;
    public void add(User user) throws HibernateException;
    public void add(User user) throws JdoException;
    
    public void add(User user) throws Exception;
    ```
    
    - 다음과 같이 인터페이스를 선언하면서도, 구현 기술에 의존적이게 되는 문제가 발생하거나
    - 무책임한 Exception으로 해결하지 않도록 해야 한다.
        - Hiberante, JPA, JdoException은 런타임 예외를 사용한다
        - 따라서 JDBC를 직접 사용하는 DAO에서 모든 SQLException을 런타임 예외로 포장한다는 전제 하에 1번의 코드와 같이 인터페이스를 작성할 수 있다.
    - 하지만 여전히 액세스 기술에 따라 같은 상황에서도 다른 종류의 예외가 발생할 수 있다는 문제가 존재한다
    → 모든 에러를 전환하는 방식으로만 처리할 수는 없다
    → 결국 발생할 예외들을 추상화 한 DataAccessException의 계층 구조를 활용하는 것이 필요하다
3.    **DataAccessException 예외 추상화**  
    - DB별로 매핑하여 SQLException → 그에 해당하는 DataAccessException의 서브 클래스들 중 하나로 전환하여 던진다
    - 즉 같은 원인의 예외라면, 데이터 액세스 기술에 상관없이 같은 예외를 던지도록 한다
        - 예시
        - 부정확한 데이터 액세스 기술 사용 : InvalidDataAccessResourceUsageException
            - JDBC : BadSqlGrammarException
            - Hibernate : HibernateQueryException, TypeMismatchDataAccessException
        - 낙관적 락킹 : ObjectOptimisticLockingFailureException
            - HibernateOptimisticLockingFailureException / JdbcOptimisticLockingFailureException / JdoOptimisticLockingFailureException

---

### DataAccessException 활용 시 주의사항

- 정말로 존재하는 모든 에러들을 일관되게 매핑해주는 것은 아님
- 세분화 되어 있지 않은 예외들에 대해선 차이가 발생한다
- 예시
    - 중복 키가 발생하는 경우
        - JDBC : DuplicateKeyExceptinon
        - Hibernate, JPA : ConstraintViolationException → 더 포괄적인 무결성 관련 예외인 DataIntegrityViolationException으로 변환
    - 교착상태
        - JDBC : DeadlockLoserDataAccessException
        - JPA/Hibernate : OptimisticLockException, PessimisticLockException

### 원하는 DataAccessException으로 전환

- 위와 같이 다르게 매핑되는 에러들을 SQLException이나 RuntimeException으로 뭉뚱그려 던지는 것보다, DataAccessException 계층의 예외로 전환하여 의미를 파악할 수 있도록 해보자

```java
catch(DuplicateKeyException ex) {
	SQLException sqlEx = (SQLException) ex.getRootCause();
	SQLExceptionTranslator set = new SQLErrorCodeSQLExceptionTranslator(this.dataSource);
	
	set.translate(null, null, sqlEx);
}
```

- 전환된 예외들은 내부에 중첩예외로 처음 발생한 SQLException을 갖고 있다
→ getRootCause()로 SQLException을 가져올 수 있다
- SQLErrorCodeSQLExceptionTranslator.translate → SQLException을 DataAccessException 타입의 예외로 변환해준다