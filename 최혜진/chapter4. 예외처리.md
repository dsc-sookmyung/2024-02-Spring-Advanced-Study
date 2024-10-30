# Chapter4. 예외처리

> Chapter4. 예외처리의 학습 목표
>> JdbcTemplate 을 대표로 하는 스프링의 데이터 액세스 기능에 담겨있는 예외처리와 관련된 접근 방법을 알아본다.

## 4.1 사라진 SQLException
`JdbcTemplate 적용 전`
```java
public void deleteAll() throws SQLException {
	this.jdbcContext.executeSql("delete from users");
}
```
`JdbcTemplate 적용 후`
```java
public void deleteAll() {
	this.jdbcTemplate.update("delete from users");
}
```
`JdbcTemplate` 을 사용한 코드에서는 `throws SQLException` 가 사라졌다. 어디로 간 것일까?
### 4.1.1 초난감 예외처리
모든 예외는 적절하게 복구되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 한다.

**예외 블랙홀**
```java
try {
  ...
} catch (Exception e) {}
```
예외가 발생했음에도 아무것도 하지 않고 넘어가는 건 정말 위험하다.
- 프로그램 실행 중 어디선가 오류가 있어 예외가 발생했는데 그것을 무시하고 계속 진행하기한다.
- 기능이 오작동하거나, 메모리나 리소스 소진의 문제가 될 수 있다.

**예외 출력만 하기**
```java
} catch (Exception e) {
  System.out.println(e);
}

} catch (Exception e) {
  e.printStackTrace();
}
```
단순히 출력만 하는 것은 예외를 처리한 것이 아니다.
- 로그는 금방 묻힌다.
- 로그 모니터링을 하지 않는 이상 굉장히 위험하다.

**그마나 나은 예외처리 →시스템 종료**
```java
} catch (SQLException e) {
  e.printStackTrace();
  System.exit(1);
}
```
실전에서 이렇게 만들라는 의미는 아니다.
<u>예외를 무시하거나 잡아먹어 버리는 코드는 만들지 말란 의미이다.</u>

**무의미하고 무책임한 throws**
```java
public void method1() throws Exception {
  method2();
  ...
}

public void method2() throws Exception {
  method3();
  ...
}

public void method3() throws Exception{
  ...
}
```
메소드 선언에 `throws Exception` 을 기계적으로 붙였다.
-  EJB 시절에 흔했던 코드
- catch 블록으로 예외를 잡아봐야 해결할 방법도 없고, JDK API나 라이브러리가 던지는 각종 이름이 긴 예외들을 처리하는 코드를 매번 throws로 선언하기도 귀찮아진 개발자의 임시방편
- 메소드 선언에서 의미있는 정보를 얻을 수 없다.(정말 무엇인가 실행 중 예외적인 상황이 발생할 수 있다는 것인지, 습관적으로 붙인 것인지 알 수 없다.)
- 적절한 처리를 통해 복구될 수 있는 예외 상황도 제대로 다룰 수 있는 기회가 박탈된다.\

> 예외처리를 무시하는 것보다 낫다고는 하지만 위의 경우도 예외 처리에 대한 나쁜 습관이므로 어떤 경우에도 용납하지 않아야 한다.

### 4.1.2 예외의 종류와 특징
#### Error
1. `java.lang.Error` 클래스의 서브클래스
2. 시스템에 비정상적인 상황이 발생했을 경우 사용된다.
3. 애플리케이션에서 대응할 필요가 없다.
#### Exception
1. `java.lang.Exception` 클래스와 그 서브 클래스로 정의되는 예외들
2. 애플리케이션 코드 작업 중 예외 상황이 발생했을 때 사용된다.
### Checked Exception
1. `Exception`클래스의 서브 클래스이면서, `RuntimeException`클래스를 상속하지 않은 것을 말한다.
2. IDE에서 예외처리를 강요한다.
3. 예외를 catch 로 처리하지 않거나, throws로 밖으로 예외를 던지지 않을 시, 컴파일 에러가 발생한다.
### Unchecked Exception
1. `RuntimeException`을 상속한 클래스들을 말한다.
2. IDE에서 예외처리를 강요하지 않는다.
3. 주로 프로그램 오류가 있을 때 발생되도록 의도된 것들이다.
4. `NullPointerException`, `IllegalArgumentException`등이 있다.
5. 피할 수 있지만, 개발자가 부주의해서 발생할 수 있는 경우에 발생하도록 만든 것이다.
### 4.1.3 예외처리 방법
#### 1. 예외 복구
 예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓는 것
 `재시도를 통해 예외 복구하는 코드`
 ```java
 int maxRetry = MAX_RETRY;

while(maxRetry --> 0) {
  try {
    ... // 예외가 발생할 수 있는 시도
    return; // 작업 성공
  }
  catch(SomeException e) {
    // 로그 출력, 정해진 시간만큼 대기
  }
  finally {
    // 리소스 반납, 정리 작업
  }
}
throw new RetryFailedException(); // 최대 재시도 횟수를 넘기면 직접 예외 발생
```
사용자가 요청한 파일을 읽으려고 시도했는데, 해당 파일이 없거나 다른 문제가 있어서 읽어지지 않는 경우 (IOException) 다른 파일을 이용해보라고 안내한다.
<u>단, IOException 에러 메시지가 사용자에게 그냥 던져지는 것이 예외 복구라고 볼 수 없다.</u>

체크 예외들은 예외가 복구될 가능성이 있는 예외이므로, 예외 복구로 처리되기 위해 사용된다.
#### 2. 예외처리 회피
예외처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던져버리는 것
`예외처리 회피`
```java
public void add() throws SQLException {
  try {
    // JDBC API
  }
  catch(SQLException e) {
    // 로그 출력
    throw e;
  }
}
```
콜백과 템플릿의 관계와 같은 명확한 역할 분담이 없는 경우 무작정 예외를 던지면 무책임한 책임 회피가 될 수 있다.
- 만약 DAO 가 SQLException 을 던지면, 이 예외는 처리할 곳이 없어서 서비스 레이어로 갔다가 컨트롤러로 가고 결국은 서버로 갈것이다.
**예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분명해야 된다.**
- 콜백/템플릿처럼 긴밀한 다른 오브젝트에게 예외처리 책임을 분명히 지게 하거나, 자신을 이용하는 쪽에서 예외를 다루는게 최선이라는 확신이 있어야 한다.
#### 3. 예외 전환
 예외를 메소드 밖으로 던지지만, 예외 회피와는 달리 발생한 예외를 그대로 넘기는게 아니라 적절한 예외로 전환해서 던지는 것
 `예외 전환 기능을 가진 DAO 메소드`
 ```java
 public void add(User user) throws DuplicateUserIdException, SQLException {
  try {
    // JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는
    // 그런 기능을 가진 다른 SQLException을 던지는 메소드를 호출하는 코드
  }
  catch(SQLException e) {
    // ErrorCode가 MySQL의 "Duplicate Entry(1062)"이면 예외 전환
    if (e.getERrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
      throw DuplicateUserException();
    else
      throw e; // 그 외의 경우는 SQLException 그대로
  }
}
```
- 예외 자체가 예외 상황에 대해 설명해주지 못 할 때, 의미를 분명히 할 예외로 전환해 던질 수 있다.
- **Q. 예외 전환을 이용해 `SQLException`을 `DuplicateUserIdException`과 같은 예외로 변경해서 던져주면?**
	- A. 예외가 일어난 이유가 한 층 명확해지고 사용자는 다른 아이디를 사용하는 것으로 <u>적절한 복구 작업을 수행</u>할 수 있다.

> `예외 전환 시 예외 전달 방식`
>> 1. 중첩 예외 (nested exception)
>>    예외가 발생한 원인을 다른 예외에 감싸는 형태로, 원래의 예외를 던지면서 다른 예외로 감싸 전달할 때 사용한다.
>>    애플리케이션 계층의 개발자는 발생한 예외의 구체적 원인까지 접근할 수 있어 문제 추적에 용이하다.
>> 2. 포장 예외 (wrap exception)
>>    예외를 감싸서 전달하면서도 예외 자체를 처리할 수 있게 해주는 방식이다.
>>    주로 checked 예외를 unchecked 예외로 변환해 사용자가 반드시 예외를 처리하지 않아도 되게 하거나, 특정 예외 타입을 공통된 형태로 감싸서 처리할 수 있도록 할 때 유용하다.

```java
catch(SQLException e) {
  ...
  throw DuplicateUserIdException(e);
}
```
중첩 예외는 getCause() 메소드를 이용해 처음 발생한 예외가 무엇인지 확인할 수 있게 해준다.

```java
try {
  ...
} catch (NamingException ne) {
  throw new EJBException(ne);
} catch (SQLException se) {
  throw new EJBException(se);
} catch (RemoteException re) {
  throw new EJBException(re);
}
```
포장 예외는 체크 예외를 언체크 예외로 바꾸는 경우에 사용한다.
중첩 예외를 이용해 새로운 예외를 만들고 원인이 되는 예외를 내부에 담아서 던지는 방식은 같지만 의미를 명확하게 하려고 다른 예외로 전환하는 것은 아니다.

> **Q. 체크드 예외와 언체크드 예외를 언제 쓰나?**
>> 복구가 가능한 상황, 코드가 정상적으로 작동하기 위해 예외 처리가 반드시 필요한 상황엔 `체크드 예외`를 사용한다.
>> 프로그래밍 오류에 가깝고, 복구가 어려운 상황에선 `언체크드 예외`가 쓰인다.

### 4.1.4 예외처리 전략
#### 런타임 예외의 보편화
체크 예외는 복구할 가능성이 조금이라도 있는, 말 그대로 예외적인 상황이기 때문에 자바는 이를 처리하는 catch 블록이나 throws 선언을 강제하고 있다.
- 예외가 발생할 가능성이 있는 API 메소드를 사용하는 개발자의 실수를 방지하기 위한 배려이다.
→ 하지만 실제로는 예외를 제대로 다루고 싶지 않을 만큼 짜증나게 만드는 원인이 되기도 한다.

서버 환경은 일반적인 애플케이션 환경과는 다르다.

**애플리케이션 환경**
-  자바 초기 AWT, 스윙 에서는 파일을 처리한다고 했을 때, 해당 파일이 통제 불가능한 예외라 할지라도 상황을 복구해야 했다.
	워드에서 특정 이름의 파일을 검색할 수 없다고 애플리케이션이 종료돼버리게 할 수 없다.

**서버 환경**
- 한 번에 다수의 사용자가 접근해 해당 서비스를 이용하기 때문에 작업을 중지하고 예외 상황을 복구할 수 없다.
- 애플리케이션 차원에서 예외 상황을 파악하고 예외가 발생하지 않도록 차단하는 것이 좋다.

#### add() 메소드의 예외처리
```java
//아이디 중복 시 사용하는 예외
public class DuplicateUserIdException extends RuntimeException{
    public DuplicateUserIdException(Throwable cause) {
        super(cause);
    }
}
//예외처리 전략을 적용한 add()
public void add() throws DuplicateUserIdException {
  try {
    //
  }
  catch (SQLException e) {
    if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
      throw new DuplicateUserIdException(e); // 예외 전환
    else
      throw new RuntimeException(e); // 예외 포장
  }
}
```
DuplicatedUserIdException도 굳이 체크 예외로 둬야 하는 것은 아니다.
- DuplicatedUserIdException처럼 의미 있는 예외는 add() 메소드를 바로 호출한 오브젝트 대신 더 앞단의 오브젝트에서 다룰 수도 있다.
- 어디에서든 DuplicatedUserIdException을 잡아서 처리할 수 있다면 굳이 체크 예외로 만들지 않고 런타임 예외로 만드는 게 낫다.
- 대신 add() 메소드는 명시적으로 DuplicatedUserIdException 을 던진다고 선언해야 한다. 그래야 add() 메소드를 사용하는 코드를 만드는 개발자게에 의미 있는 정보를 전달해줄 수 있다. 이렇게 되면 런타임 예외도 throws로 선언할 수 있으니 문제 될게 없다.

> 런타임 예외를 일반화해서 사용할 때 단점
> 1. 컴파일러가 예외처리를 강제하지 않으므로 신경 쓰지 않으면 예외상황을 충분히 고려하지 않을 수도 있다.
> 2. 런타임 예외를 사용하는 경우에는 API 문서나 레퍼런스 문서 등을 통해, 메소드를 사용할 때 발생할 수 있는 예외의 종류, 원인, 활용방법을 자세히 설명해야 한다.

#### 애플리케이션 예외
런타임 예외 중심의 전략은 굳이 이름을 붙이자면 낙관적인 예외처리 기법이다.
-  복구할 수 있는 예외는 없다고 가정한다.
-  예외가 생겨도 어차피 런타임 예외이므로 시스템이 알아서 처리해줄 것이고, 꼭 필요한 경우는 런타임 예외라도 잡아서 복구하거나 대응할 수 있으니 문제가 될 것이 없다는 낙관적 태도를 기반으로 한다.
이런 면에서 혹시 놓치는 예외가 있을까 처리를 강제하는 체크 예외의 비관적 접근 방법과 대비된다.

반면 애플리케이션 예외도 있다.
**애플리케이션 예외**는 시스템 또는 외부의 예외 상황이 원인이 아니라 <u>애플리케이션 자체의 로직에 의해 의도적으로 발생시키고, 반드시 catch 해서 무엇인가 조치를 취하도록 요구하는 예외</u>이다.

`애플리케이션 예외를 사용한 코드`
```java
try {
  BigDecimal balance = account.withdraw(amount);
  ...
  // 정상적인 처리 결과를 출력하도록 진행
}
catch(InsufficientBalanceException e) { // 체크 예외
  // InsufficientBalanceException에 담긴 인출 가능한 잔고 금액 정보를 가져옴
  BigDecimal availFunds = e.getAvailFunds();
  ...
  // 잔고 부족 안내 메세지를 준비하고 이를 출력하도록 진행
}
```

### 4.1.5 SQLException 은 어떻게 됐나?
`SQLException`은 복구 불가능
일반적으로 해당 예외가 발생하는 이유는 SQL 문법이 틀렸거나, 제약조건을 위반했거나, DB 서버가 다운됐거나, 네트워크가 불안정하거나, DB 커넥션 풀이 꽉 차서 DB 커넥션을 가져올 수 없는 경우 등이다.
예외처리 전략을 적용해서 필요없는 기계적인 throws 선언이 등장하도록 방치하지 말고 가능한 빨리 언체크/런타임 예외로 전환해줘야 한다.

**스프링 런타임 예외 보편화 전략**
스프링 API 메소드에서 정의되어 있는 대부분의 예외는 런타임 예외이다. 따라서 발생 가능한 예외가 있더라도 이를 처리하도록 강제하지 않는다.

`SQLException`이 사라진 이유는 스프링의 `JdbcTemplate`은 `런타임 예외의 보편화` 전략을 따르고 있기 때문이다.

`JdbcTemplate` 템플릿과 콜백 안에서 발생하는 모든 `SQLException`을 런타임 예외인 `DataAccessException`으로 포장해서 던져준다.

`JdbcTemplate`의 `update()`, `queryForInt()`, `query()` 메소드 선언을 잘 살펴보면 모두 `throws DataAccessException`이라고 되어 있음을 발견할 수 있다.

```java
public int update(final String sql) throws DataAccessException {
    //...
}
```

`throws`로 선언되어 있긴 하지만 `DataAccessException`이 런타임 예외이므로 `update()`를 사용하는 메소드에서 이를 잡거나 다시 던질 이유는 없다.

## 4.2 예외 전환
### 4.2.1 JDBC의 한계
DBC는 Connection, Statement, ResultSet 등의 표준 인터페이스로 기능을 제공한다. 따라서 개발자들은 DB 종류와 상관없이 일관된 방법으로 프로그래밍이 가능하다.
인터페이스를 사용하는 객체지향 프로그래밍 방법의 장점을 잘 경험할 수 있는 것이 바로  JDBC이다.
하지만, DB 종류에 따라 데이터 액세스 코드가 달라질 수 있다.

#### 비표준  SQL 문제
DB마다 SQL 비표준 문법이 제공된다.
최적화 기법, 쿼리 조건 관련 추가적인 문법이 있을 수 있다.
만약 작성된 비표준 SQL 이 DAO 코드에 들어가게 되면 해당 DAO 는 특정 DB에 대해 종속적인 코드가 된다.

**해결책**
- 호환되는 표준 SQL 만 사용 (페이징 쿼리부터 사용 X)
-  DB 별 DAO 만들기
-  SQL을 외부에서 독립시켜 DB 에 따라 변경 가능하게 만들기

#### 호환성 없는 SQLException의 DB 에러 정보 문제
DB 마다 에러의 종류와 원인은 제각각이다.
- JDBC 는 데이터 처리 중 발생하는 다양한 예외를 그냥 `SQLException`하나에 모두 담아버린다.
-  이마저도 SQLException.getErrorCode()로 에러 코드를 가져왔을 때, DB 벤더마다 에러코드가 달라서 각각 처리해주어야 한다.
```java
// MySQL에서 중복된 키를 가진 데이터를 입력하려고 시도했을 때
if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY) { ...
```
-  getSQLState()와 같은 메소드로 예외 상황에 대한 상태 정보를 가져올 수 있지만, 해당 값을 신뢰하기 힘들다. 어떤 DB는 표준 코드와 상관없는 엉뚱한 값이 들어있기도 하다.

> 결과적으로 SQL 상태 코드를 믿고 결과를 파악하도록 코드를 작성하는 것은 위험하다.
> 결국 호환성 없는 에러 코드와 표준을 잘 따르지 않는 상태 코드를 가진 `SQLException`만으로 DB에 독립적인 유연한 코드를 작성하는 것은 불가능에 가깝다.

### 4.2.2 DB 에러 코드 매핑을 통한 전환
스프링은 `DataAccessException`의 서브 클래스로 세분화된 예외 클래스들을 정의하고 있다.

- SQL 문법 : `BadSqlGrammerException`
- DB 커넥션 : `DataAcessResourceFailureException`
- 데이터의 제약조건을 위배했거나 일관성을 지키지 못함 : `DataIntegrityViolationException`
- 그 중에서도 중복 키 때문에 발생한 경우 : `DuplicatedKeyException`

문제가 있다. DB마다 에러 코드가 제각각이라는 점이다.

이를 해결하기 위해, DB별 에러 코드를 참고해 발생한 예외 원인을 해석해줄 해석기가 필요하다.

**스프링 코드 매핑 테이블**
`오라클 에러 코드 매핑 파일`
```xml
<bean id="Oracle" class="org.springframework.jdbc.support.SQLErrorCodes">
        <property name="badSqlGrammarCodes">
            <value>900,903,904,917,936,942,17006,6550</value>
        </property>
        <property name="invalidResultSetAccessCodes">
            <value>17003</value>
        </property>
        <property name="duplicateKeyCodes">
            <value>1</value>
        </property>
        <property name="dataIntegrityViolationCodes">
            <value>1400,1722,2291,2292</value>
        </property>
        <property name="dataAccessResourceFailureCodes">
            <value>17002,17447</value>
        </property>
        <property name="cannotAcquireLockCodes">
            <value>54,30006</value>
        </property>
        <property name="cannotSerializeTransactionCodes">
            <value>8177</value>
        </property>
        <property name="deadlockLoserCodes">
            <value>60</value>
        </property>
</bean>
```
이를 통해 JdbcTemplate은 DB 에러 코드를 적절한 DataAccessException 서브클래스로 매핑한다.

`JdbcTemplate이 제공하는 예외 전환 기능을 이용한 add() 메소드`
```java
public void add(User user) throws DuplicateKeyException {
    this.jdbcTemplate.update("insert into users(id, name, password) values (?, ?, ?)"
            , user.getId()
            , user.getName()
            , user.getPassword()
    );
}
```

`중복 키 예외의 전환`
```java
public void add(User user) throws DuplicateUserIdException {
    try {
        this.jdbcTemplate.update("insert into users(id, name, password) values (?, ?, ?)"
                , user.getId()
                , user.getName()
                , user.getPassword()
        );
    } catch (DuplicateKeyException e) {
        throw new DuplicateUserIdException(e);
    }
}
```
위와 같이 더 명확한 예외 클래스로 예외 전환도 가능하다.

### 4.2.3 DAO 인터페이스와  DataAccessException 계층구조
`DataAccessException` 은 의미가 같은 예외라면 데이터 액세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어준다.
데이터 액세스 기술에 독립적인 추상화된 예외를 제공하는 것이다.

#### DAO 인터페이스와 구현의 분리
**Q. DAO 를 굳이 따로 만들어서 사용하는 이유는?**
	A. 데이터 액세스 로직을 담은 코드를 성격이 다른 코드에서 분리해놓기 위해서이다. 또한 분리된 DAO 는 전략패턴을 적용해 구현 방법을 변경해서 사용할 수 있게 만들기 위해서 이다.

DAO를 인터페이스로 분리하면, 내부의 데이터 액세스 기술에 의존하지 않게 된다. POJO를 주고받으며 데이터 액세스 기능을 사용하기만 하면 된다.

`기술에 독립적인 이상적인 DAO 클래스`
```java
public interface UserDao {
  public void add(User user);
}
```
이때 DAO 안에서 throws 하지 않는다. 만약 throws 하게 되면, 자바 DATA 접근 API 가 바뀔 시 인터페이스도 바뀐다.
인터페이스에서는 체크드 예외를 던지지 않도록 하고, 구현체에서는 데이터 접근 기술별 예외를 처리한 후 런타임 예외로 변환해 던지는 방식으로 처리해야 된다.

> **결론**
>> DAO 인터페이스를 특정 데이터 접근 기술에 종속시키지 않는 것이 중요하다.
>> 위에서 하는 말은 인터페이스에서 특정 기술의 예외 (`SQLException`, `PersistentException`, `HibernateException`, `JdoException` 등)를 선언하면, 그 예외는 기술에 따라 다르기 때문에 기술을 변경할 때마다 인터페이스가 바뀌어야 하는 문제가 발생한다. 그러므로 특정 데이터 접근 기술에 종속시키지 않아야 한다.

#### 데이터 액세스 예외 추상화와 DataAccessException 계층 구조
스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해서 DataAccessException 계층 구조 안에 정리해놓았다.

![Jdbc 를 이용한 낙관적 락킹 예외 클래스의 적용](img.png)
`Jdbc 를 이용한 낙관적인 락킹 예외 클래스의 적용`
`JdbcTemplate`과 같이 스프링의 데이터 액세스 지원 기술을 이용해 `DAO` 를 만들면, 사용 기술에 독립적인 일관성 있는 예외를 던질 수 있다.

결국 인터페이스 사용, 런타임 예외 전환과 함께 `DataAccessException` 예외 추상화를 적용하면 데이터 액세스 기술과 구현 방법에 독립적인 이상적인 DAO를 만들 수 있다.

### 4.2.4 기술에 독립적인 UserDao 만들기
`UserDao 인터페이스`
```java
public interface UserDao {
    void add(User user);
    User get(String id);
    User getByName(String name);
    List<User> getAll();
    void deleteAll();
    int getCount();
}
```
setDataSource() 메소드는 인터페이스에 추가하면 안 된다.
- setDataSource() 메소드는 UserDao 의 구현 방법에 따라 변경될 수 있는 메소드이다.
- UserDao 를 사용하는 클라이언트가 알고 있을 필요도 없다.

이제 기존의 UserDao 클래스는 다음과 같이 이름을 UserDaoJdbc로 변경하고 UserDao 인터페이스를 구현하도록 implements 로 선언해줘야 한다.

```java
public class UserDaoJdbc implements UserDao { ...}
```

또 한 가지 변경사항은 스프링 설정파일의 userDao 빈  클래스 이름이다.
`빈 클래스 변경`
```java
<bean id="userDao" class="springbook.dao.UserDaoJdbc"> <property name="dataSource" ref="dataSource" /> </bean>
```
테스트는 굳이 보완할 필요가 없다. @Autowired 를 통해 자동으로 빈을 주입 받기 때문이다.

#### DataAccessException 활용 시 주의사항
이렇게 스프링을 활용하면 DB 종류나 데이터 액세스 기술에 상관없이 키 값이 중복되는 상황에서는 동일한 예외가 발생하리라 기대할 것이다.

하지만 DuplicateKeyException은 아직까지 JDBC를 이용하는 경우에만 발생한다!
데이터 액세스 기술을 하이버네이트나 JPA를 사용했을 때도 동일한 예외가 발생할 것으로 기대하지만 실제로 다른 예외가 던져진다.

그 이유는 SQLException에 담긴 DB 의 에러 코드를 바로 해석하는 JDBC 의 경우와 달리 JPA나 하이버네이트, JDO 등에서는 각 기술이 재정의한 예외를 가져와 스프링이 최종적으로 DataAccessException으로 변환하는데, DB의 에러코드와 달리 이런 예외들은 세분화되어 있지 않기 때문이다.

따라서, DataAccessException 을 잡아서 처리하는 코드를 만들려고 한다면 미리 학습 테스트를 만들어서 실제로 전환되는 예외의 종류를 확인해 둘 필요가 있다.

Q. 만약 DAO에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면?
	A. DuplicatedUserIdException처럼 직접 예외를 정의해두고, 각 DAO의 add() 메소드에서 좀 더 상세한 예외 전환을 해주면 된다.

