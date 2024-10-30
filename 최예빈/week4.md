# 4장. 예외

> 3장에서 직접 만든 JdbcContext를 스프링의 JdbcTemplate로 바꾸면서 throws SQLException이 사라졌다. 왜 안보일까?

## 예외는 꼭 처리하자 

- 컴파일 에러를 없애기 위해 예외를 catch 하지만 아무것도 하지 않고 넘어가버리는 코드는 비단 초보자만의 것이 아니다.
- 예외 발생 콘솔 출력은 예외 처리가 되지 못한다.
  - 계속 돌아가는 애플리케이션의 콘솔 로그를 계속 읽고 있을 수는 없다.

## 예외처리 할 때 고려할 것

### 예외 처리 핵심 원칙

- 모든 예외가 적절히 복구되거나
- 작업을 중단하고 운영자 또는 개발자에게 분명하게 통보해야 한다

### `SQLException`

- SQL 문법 에러, 데이터 엑세스 로직 버그, DB 서버 다운, DB 네트워크 중단 등 심각한 상황.
- 무시해선 안되는 예외다.

### 나쁜 습관 2가지

- 예외를 무시하거나 잡아먹어서는 안된다.
  - 예외를 잡아서 조취를 취할 수 없다면 잡지도 마라.
  - throws SQLException으로 메소드 밖으로 던져서 호출한 코드가 처리하도록 책임 전가 해라.
- 그럼 일단 다 throws Exception으로 던지면 어떨까?
  - 정확하게 예외 이름을 적지 않았다. 메소드 선언에서 의미 있는 정보를 얻을 수 없는 게 문제다.

### 자바 throw로 발생시킬 수 있는 예외 3가지

#### Error

- java.lang.Error과 그 서브클래스
- 주로 JVM이 발생시키는 것이므로 애플리케이션 코드로 잡지 않음

#### Exception

> 체크 예외
- java.lang.Exception과 그 서브클래스로 정의
- 예외 처리 코드 미작성시 컴파일 에러 발생
  - catch하거나
  - throws로 메소드 밖으로 던져야함
- 최근 체크 예외 때문에 예외 블랙홀(catch하고 처리 안함), 무책임한 throws(일단 던지기, Exception로 두루뭉술하게 던짐)이 남발된다는 의견
  - 최신 자바 표준 스펙 API는 예상 가능한 예외 상황은 체크 예외로 만들지 않는 경향이 생김.

> 언체크 예외 == 런타임 에외
- java.lang.RuntimeException와 이를 상속한 예외
- 예외처리를 강제하지 않음
- 주로 프로그램의 오류가 있을 때 발생하도록 의도된 것
  - NullPointException, IllegalArgumentException, ...
  - 개발자가 예외 조건을 잘 고려하여 프로그램을 짠다면 피할 수 있는 문제
  - 예상 못하는 예외사항이 아니라서 굳이 catch, throws를 강제하지 않은 것

## 예외처리

### 예외처리 방법 3가지

#### 예외 복구

- 문제를 해결해서 정상 상태로 돌려놓는 것
- 예) 사용자가 요청한 파일을 읽으려고 시도, 해당 파일이 없거나 다른 문제가 있어서 IOException 발생했을 때
  - 다른 파일을 이용하도록 작업 흐름 유도

> 네트워크 이상으로 원격 DB 접속에 실패하여 SQLException
- 일정 시간 대기했다가 재시도하는 로직
```java
int maxretry = MAX_RETRY;
while(maxretry -- > 0) {
  try {
    ...        // 예외 발생 가능성이 있는 시도
    return;    // 작업 성공
  } catch (SomeException e) {
    // 로그 출력, 정해진 시간 만큼 대기
  }
  finally {
    // 리소스 반납, 정리 작업
  }
}
```

#### 예외처리 회피

- 자신을 호출한 쪽으로 던짐

> 2가지 방식
1) throws 문으로 메소드가 예외를 알아서 던지게 하거나
2) catch문으로 잡은 다음 로그를 남기고 예외를 다시 throw

> 빈 catch문으로 잡는 것도 회피인가?
- 특별한 의도를 가지고 예외를 복구했거나 아무 개념이 없는 경우. 회피는 아님.

> 예외 던지기
[3.6.1](https://github.com/y3binchoi/2024-02-Spring-Advanced-Study/blob/y3binchoi/%EC%B5%9C%EC%98%88%EB%B9%88/week3.md#361-update)
```java
// 템플릿 메소드 적용
public void deleteAll() {
    this.jdbcTemplate.update(
        new PreparedStatementCreator() {
            public PreparedStatement createPreparedStatement(Connection con)
                throws SQLException {
                return con.prepareStatement("delete from users");
            }
        }
    );
}
// 메소드 사용 
public void deleteAll() throws SQLException {
		this.jdbcTemplate.update("delete from users");
}
```
- ResultSet, PreparedStatement 등의 작업에서 발생하는 SQLException을 템플릿으로 던진다
- SQLException을 처리하는 일은 콜백 오브젝트의 역할이 아니라고 보는 것

> 무작정 예외를 던져도 될까?
- 콜백/템플릿처럼 긴밀한 관계일 때 다른 오브젝트에 예외처리 책임을 분명히 지게 하거나
- 자신을 사용하는 쪽에서 예외를 다루는 게 최선의 방법이라는 분명한 확신이 필요하다

#### 예외 전환

- 복구가 불가능한 상태라서 예외를 밖으로 던진다
- 하지만 회피와 달리, 적절한 예외로 전환해서 던진다

> 목적
1. 의미를 더 분명하게 보여주기 위해 
2. 예외를 처리하기 쉽고 단순하게 만들기 위해 포장wrap

> 새로운 사용자 등록 시도, 아이디가 같은 사용자가 있다면
```java
public void add(User user) throws DuplicateUserIdException, SQLException {
  try {
    // JDBC를 이용해 user 정보를 DB에 추가
  } catch (SQLException e) {
    // ErrorCode가 MySQL의 "Duplicate Entry(1062)"이면 예외 전환
    if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
      throw DuplicateUserIdException();
    else
      throw e;
  }
}
```
- 중첩 예외
```java
throw DuplicateUserIdException(e);              // 원래의 예외를 담음. getCause()로 처음 발생한 예외를 확인할 수 있다.
throw DuplicateUserIdException().initCause(e);  // 근본 원인이 되는 예외를 넣어준다.
```

> 포장하기
- 주로 예외처리를 강제하는 체크 예외를 런타임 예외로 바꾸는 경우에 사용
```java
// EJB 컴포넌트 체크 예외가 비즈니스 로직의 관점에서 의미있거나 복구가능한 예외가 아님. 따라서 EJBException으로 포장해서 던진다.
try {
    OrderHome orderHome = EJBHomeFactory.getInstance().getOrderHome();
    Order order = orderHome.findByPrimaryKey(Integer id);
} catch (NamingException ne) { 
    throw new EJBException(ne);
} catch (SQLException se) {
    throw new EJBException(se);
} catch (RemoteException re) {
    throw new EJBException(re);
}
```

### 예외처리 전략

- 런타임 예외의 보편화. 낙관적 예외 처리 기법
- 애플리케이션 예외

#### 다 런타임 예외로 던지자

- 최신 자바 표준 스펙 또는 오픈소스 프레임워크에서는 API가 발생시키는 예외를 언체크 예외로 정의하는게 일반화 됨
- 항상 복구 가능한 경우가 거의 없기도 하고
- 언체크 예외도 얼마든지 필요할 때 catch 해서 복구/처리 가능
- 체크 예외도 다 포장해서 던지게 됨

> add() 메소드의 예외 처리
- SQLException이 복구 가능한 예외인가? NO
- JdbcTemplate는 이걸 기계적인 throws로 방치하지 않고 런타임 에외로 전환한다.
```java
public void add() throws DuplicateUserIdException {
    try {
        // JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는
        // 그런 기능이 있는 다른 SQLException을 던지는 메소드를 호출하는 코드
    }
    catch (SQLException e) {
        if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
            throw new DuplicateUserIdException(e); // 예외 전환
        else
            throw new RuntimeException(e); // 예외 포장
    }
}
```

#### 애플리케이션 예외

- 시스템 또는 외부의 예외상황이 원인이 아님
- 애플리케이션 자체의 로직에서 의도적으로 예외 발생시키고, 반드시 catch해서 조치를 취하게 하는 예외
<img width="475" alt="image" src="https://github.com/user-attachments/assets/124125d6-7290-4cff-ba9d-3029f88b509f">

#### JdbcTemplate의 예외처리 흐름

## 예외 전환

### 목적 2가지
1) 런타임 예외로 포장해서 굳이 필요 없는 catch/throws를 줄임
2) 로우레벨의 예외를 더 의미 있고 추상화된 예외로 바꿔 던짐

### JDBC의 한계

- 비표준 SQL, 특정 DB 전용 문법. 
- 호환성 없는 SQLException의 DB 에러 정보.

### DB 호환성을 위해 해결할 문제 2가지

- SQLException의 비표준 에러
  - DB 전용 에러 코드는 제각각이지만 예외의 원인을 파악하고 일관적으로 해석하여 일관된 예외로 전달받게 만든다
    - DB별 에러 코드를 분류해서 스프링이 정의한 예외 클래스와 매핑해놓은 에러 코드 매핑 정보 테이블을 만들어두고 이를 이용
    - 오라클 에러 코드 매핑 파일
      - 단지 런타임 예외로 포장하는 것이 아니라 에러 코드를 계층구조 안의 클래스 중 하나로 매핑함
      <img width="481" alt="image" src="https://github.com/user-attachments/assets/8c30001b-57d9-42bb-bee2-06ae66bd90ca">

- SQL 상태 정보

## 스프링이 제공하는 DB 독립적 런타임 예외 계층 

### DataAccessException의 계층 구조

- 애플리케이션에서 직접 정의한 예외를 발생시키고 싶다면
  - 잡아서 전환한다
  - 이때 원인이 되는 예외를 중첩한다

### 스프링이 DataAccessException 계층구조를 이용해 기술 독립적인 예외를 정의하고 사용하는 이유

- DAO를 인터페이스를 사용해 클래스 정보와 구현 방법을 감추고, DI로 제공되도록 만들어도 메소드 선언에 나타나는 예외 정보가 문제가 될 수 있다.
  - 인터페이스를 throws SQLException으로 선언해야 하는데 이렇게 되면 JDBC 이외의 데이터 엑세스 기술로 갈아끼울 수 없다
  - 그럼 모두 받아주는 Exception을 던지면? 무책임하다.
  - 이런 문제 때문에 이후에 등장한 JDO, Hibernate, JPA 등 기술은 런타임 예외를 던진다
  - JDBC도 DataAccessException으로 포장해서 던져줘야 일관적인 메소드 선언이 가능해진다
  - <img width="525" alt="image" src="https://github.com/user-attachments/assets/a903bce7-e256-408c-bc53-bab4c772d41c">

- 그럼에도 의미 있게 처리해야 하는 예외가 있다
  - DataAccessException은 SQLException 뿐만 아니라 자바의 주요 데이터 엑세스 기술의 대부분의 예외를 추상화 한다

### 기술 독립적인 UserDao 만들기

- 인터페이스 도입
- 구현 클래스 정의, 런타임 예외 전환

<img width="483" alt="image" src="https://github.com/user-attachments/assets/5f3c0558-486f-4a7b-a1d6-44ed635043bc">

### DataAccessException의 한계

- 학습 테스트로 실제 전환되는 예외 종류 확인 필요

#### 직접 예외를 정의하고 각 add()에서 더 상세한 예외로 전환

- 하이버네이트 예외도 결국 중첩된 예외로 SQLException이 전달되므로 다시 JDBC 예외 전환 클래스로 전환할 수 있다
<img width="378" alt="image" src="https://github.com/user-attachments/assets/f153941a-d5a3-4777-a9a4-f8630f51f8b8">
