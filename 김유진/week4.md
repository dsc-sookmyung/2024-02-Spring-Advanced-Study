# 4장. 예외

### ✨ 스프링의 데이터 액세스 기능에 담겨 있는 예외처리 & 접근 방법에 대해 알아보자!

<br>

## 4.1 사라진 SQLException

🤔 JdbcTemplate으로 바꾸기 전과 후의 deleteAll() 메소드를 비교해보자!

- JdbcTemplate 적용 이전에는 있었던 throws SQLException 선언이 적용 후에는 사라졌다!
- 이 SQLException은 어디로 간 것일까?

<br>

### 4.1.1 초난감 예외처리

😱 개발자들의 코드에서 종종 발견되는 초난감 예외처리의 대표 선수들을 살펴보자.

<br>

#### 1️⃣ 예외 블랙홀

- 초난감 예외처리 코드 예시
    - 초난감 예외처리 코드 1
        
        ```java
        try{
        ...
        }
        catch(SQLException e) {
        }
        ```
        
        - 예외가 발생하고 catch 블록을 써서 잡아내는 것까지는 좋음.
        - 다만, **아무것도 하지 않고 넘어가서 다음 코드를 실행하는 것**은 매우 위험하다!!
            - 어떤 기능이 비정상적으로 동작하거나,
            - 메모리나 리소스가 소진되거나,
            - 예상치 못한 다른 문제로 이어질 것이다!
    - 초난감 예외처리 코드 2
        
        ```java
        catch(SQLException e) {
        	System.out.println(e);
        }
        ```
        
        ```java
        catch(SQLException e) {
        	e.printStackTrace();
        }
        ```
        
        - 예외 메시지를 화면에 출력해주기만 하는 것은 예외를 처리한 것이 아니다.
            - 다른 로그 혹은 메시지에 금방 묻혀버리면 놓치기 쉽상임!
<br>

- 예외를 처리할 때 반드시 지켜야 할 핵심 원칙은?
    - 모든 예외는 **적절하게 복구**되거나,
    - **작업을 중단**시키고 **운영자나 개발자에게 통보**되어야 한다.

<br>

- SQLException이 발생하는 이유는 심각한 상황이 벌어졌기 때문이다.
    - 예시
        - SQL 문법 에러
        - DB에서 처리할 수 없을 정도의 데이터 액세스 로직 버그
        - 네트워크 끊김
    - 따라서, 예외를 무시하고 다음 코드로 실행을 이어가는 건 정말 위험하다.

<br>

**✨ 예외를 무시하거나 잡아먹어 버리는 코드는 만들지 말자!**

<br>

#### 2️⃣ 무의미하고 무책임한 throws

- catch 블록으로 예외를 잡아봐야 해결 방법이 없거나
- 이름이 긴 예외들을 처리하는 코드를 매번 선언하기 귀찮아지면

⇒ 메소드 선언에 throws Exception을 기계적으로 붙이는 개발자가 있다…!

<br>

- 초난감 예외처리 코드 4
    
    ```java
    public void method1() throws Exception {
    	method2 ();
    }
    public void method2() throws Exception {
    	method3 ();
    }
    public void method3() throws Exception {
    	method3 ();
    }
    ```
    
<br>

🤔 이렇게 무책임한 throws 선언도 심각한 문제가 있다!

- 사용하려고 하는 메소드에 throws Exception이 선언되어 있다고 가정해보자
    - 정말 실행 중 예외적인 상황이 발생할 수 있다는 것인지
    - 습관적으로 복사해서 붙여넣은 것인지 알 수 없다.

⇒ **결과적으로 적절한 처리를 통해 복구될 수 있는 예외 상황도 제대로 다룰 수 있는 기회를 박탈당한다.**

<br>

### 😤 이 두 가지 나쁜 습관은 어떤 경우에도 용납하지 않아야 한다.

<br>
<br>

### 4.1.2 예외의 종류와 특징

🤔 그렇다면 예외를 어떻게 다뤄야 할까?

- 개발자들 사이에서 가장 큰 이슈
    - 체크 예외(명시적인 처리가 필요한 예외를 사용하고 다루는 방법)

<br>

- 자바에서 throw를 통해 발생시킬 수 있는 예외는 크게 세 가지가 있다.
    1. Error
        - 시스템 레벨에서 작업을 하는 게 아니라면, 애플리케이션에서는 이런 에러에 대한 처리는 신경쓰지 않아도 된다.
            - catch 블록으로 잡아봤자 대응 방법이 없다!
    2. Exception과 체크 예외
        - 개발자들이 만든 코드의 작업 중에 예외상황이 발생했을 경우에 사용
        - 체크 예외/언체크 예외
            - 체크 예외
                - Exception 클래스의 서브클래스
                - RuntimeException 클래스를 상속 X
            - 언체크 예외
                - RuntimeException 클래스를 상속
            
            ![image (5)](https://github.com/user-attachments/assets/c0e1ec47-3ed0-4acb-afd2-f8fa3f32fa41)

            
        - 일반적으로 말하는 예외?
            - 체크 예외
                - catch 문으로 잡든지
                - throws를 정의해서 메소드 밖으로 던져야 함.
                    - 컴파일 에러가 발생하기 때문!
    3. RuntimeException과 언체크/런타임 예외
        - 언체크 예외/런타임 예외
            - NullPointerException
            - IllegalArgumentException
        - catch 문으로 잡거나 throws로 선언하지 않아도 된다.
            - 예상하지 못했던 예외상황에서 발생하는 게 아니기 때문

<br>
<br>

### 4.1.3 예외처리 방법

✨ 예외를 처리하는 일반적인 방법을 살펴보고 효과적인 전략을 생각해보자!

<br>

#### 1️⃣ 예외 복구

- 예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓는 것
    - 예시
        1. IOException 발생
            - 사용자가 요청한 파일을 읽으려고 시도 → 파일이 존재하지 않거나 문제가 있어서 읽히지 않는 에러 발생
            - 사용자에게 다른 파일을 이용하도록 안내
        2. SQLException
            - 열악한 환경의 시스템에서 원격 DB 접속에 실패하는 경우
            - 재시도
- 이처럼, 체크 예외들은 복구할 가능성이 있는 경우에 사용한다.

<br>

#### 2️⃣ 예외처리 회피

- 예외처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던져버리는 것
    - thorws 문으로 선언해서 예외 발생시 알아서 던져지게 하거나
    - catch문으로 잡아서 다시 예외를 던지는 것

<br>

- 예시
    - JdbcContext나 JdbcTemplate이 사용하는 콜백 오브젝트
        - SQLException을 처리하지 않고 템플릿으로 던진다.
            - 콜백 오브젝트의 역할이 아니라고 보기 때문

<br>

‼️ 하지만, 콜백과 템플릿처럼 긴밀하게 역할을 분담하고 있는 관계가 아니라면 자신의 코드에서 발생하는 예외를 던져버리는 건 무책임한 책임회피일 수 있다.

<br>

**따라서, 예외를 회피하는 것에는 의도가 분명해야 한다.**

- 긴말한 관계의 다른 오브젝트에게 예외처리 책임을 분명히 지게 하거나
- 자신을 사용하는 쪽에서 예외를 다루는 게 최선의 방법이라는 분명한 확신이 있어야 한다.

<br>

#### 3️⃣ 예외 전환

- 예외를 적절한 예외로 전환한 후 메소드 밖으로 던지는 것
<br>

- 목적
    1. 내부에서 발생한 예외를 그대로 던지는 것이 적절한 의미를 부여해주지 못하는 경우, 의미를 분명하게 해줄 수 있는 예외로 바꿔주기 위해
        - 예시
            - 새로운 사용자를 등록할 때 아이디의 중복으로 인해 DB 에러가 발생하는 경우
                - SQLException 발생
                - 정보를 해석해서 DuplicateUserIdException 같은 예외로 바꾸기
                - 보통 전환하는 예외에 원래 발생한 예외를 담아 **중첩 예외**로 만드는 것이 좋다.
    2. 예외를 처리하기 쉽고 단순하게 만들기 위해 포장하는 것
        - 주로 체크 예외를 런타임 예외로 바꾸는 경우에 사용
        - 예시
            - EJBException
                - EJB 컴포넌트 코드에서 발생하는 예외는 의미 있는 예외이거나 복구 가능한 예외가 아니다.
                - 이런 경우 런타임 예외로 포장해서 던지는 편이 낫다.
        - 어차피 복구하지 못할 예외라면, 런타임 예외로 포장해서 던져버리고
        - 자세한 로그를 남기고, 관리자에게는 통보하고, 사용자에게는 안내 메시지를 보여주는 식으로 처리하는 게 바람직하다.

<br>
<br>

### 4.1.4 예외처리 전략

✨ 자바의 예외를 효과적으로 사용하고, 예외가 발생하는 코드를 깔끔하게 정리하는 데는 여러 가지 신경 써야 할 사항이 많다.

지금까지 살펴본 바를 기준으로 일관된 예외처리 전략을 정리해보자!

<br>

#### 런타임 예외의 보편화

- 체크 예외
    - 복구할 가능성이 조금이라도 있으면 catch 블록이나 throws 선언을 강제한다.
    - 예외를 제대로 다루고 싶지 않을 만큼 짜증나게 만드는 원인이 된다.

<br>

- 하지만 자바 엔터프라이즈 서버 환경은…
    - 서버의 특정 계층에서 예외가 발생했을 때 작업을 중단하고 사용자와 커뮤니케이션하면서 예외 상황을 복구할 수 있는 방법이 없다.
    - 차라리, 예외 상황을 미리 파악하고 차단하는 것이 낫다.

<br>

⇒ 자바의 환경에 서버로 이동하면서 **체크 예외의 활용도와 가치가 점점 떨어지고 있다.**

대응이 불가능한 체크 예외라면 **빨리 런타임 예외로 전환해서 던지는 게 낫다.**

<br>

#### add() 메소드와 예외처리

😮 add() 메소드에서…

- SQLException
    - 처리할 것도 없고, 결국 애플리케이션 밖으로 던져짐
    
    ⇒ 런타임 예외로 포장해서 던져버리자!
    
- DuplicatedUserIdException
    - 꼭 체크 예외로 둬야 하는가…?
    - 어디엣든 처리할 수 있다면 런타임 예외로 만드는 게 낫다.
        - 대신, add() 메소드는 명시적으로 DuplicatedUserIdException을 던진다고 선언해야 한다.

<br>

✨ 이 방법을 이용해 add() 메소드를 수정해보자!

- 아이디 중복 시 사용하는 예외
    
    ```java
    public class DuplicateUserIdException extends RuntimeException{
        //중첩 예외를 위한 생성자를 추가
        public DuplicateUserIdException(Throwable cause){
            super(cause);
        }
    }
    ```
    

- 예외처리 전략을 적용한 add() 메소드
    
    ```java
    public void add() throws DuplicateUserIdException{
        try{
                ...
        }catch (SQLException e){
            if(e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
                throw new DuplicateUserIdException(e); //예외 전환
            else
                throw new RuntimeException(e); //예외 포장
        }
    }
    ```
    
    - add() 메소드를 사용하는 쪽에서 활용할 수 있음을 알려주도록 메소드의  throws 선언에 포함
    - 이제 SQLException을 처리하기 위해 불필요한 throws 선언을 할 필요가 없고,
    - 필요한 경우 아이디 중복 상황을 처리하기 위해 DuplicateUserIdException을 이용할 수 있다.

<br>

#### 애플리케이션 예외

- 시스템 또는 외부의 예외상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고, 반드시 catch해서 무엇인가 조치를 취하도록 요구하는 예외

- 예시
    - 사용자가 요청한 금액을 출금하는 기능을 가진 메소드가 있다고 생각해보자.
        - 현재 잔고를 확인하고, 허용 범위를 넘어서 출금을 요청하면 작업을 중단시키고 사용자에게 경고를 보내야 한다.

- 이런 기능을 담은 메소드를 설계하는 방법 두 가지
    1. 정상적인 출금처리를 했을 경우와 잔고 부족이 발생했을 경우에 각각 다른 종류의 리턴 값을 돌려주는 것
        - 리턴 값을 일종의 결과 상태를 나타내는 정보로 활용한다!
        - 정상적인 출금 처리
            - 요청금액을 리턴
        - 잔고가 부족한 경우
            - 0 또는 -1 같은 특별한 값을 리턴
        - 이런 경우에 불편한 점?
            - 정상적인 처리가 안 됐을 때 전달하는 값의 표준이 존재하지 않는다.
                - 혼란이 발생할 수 있다.
            - 결과 값을 확인하는 조건문이 자주 등장한다.
                - 코드가 지저분해지고 흐름을 파악하고 이해하는 것이 어려울 수 있다.
    2. 정상적인 흐름을 따르는 코드는 그대로 두고, 예외 상황에서는 비즈니스적인 의미를 띤 예외를 던지도록 만드는 것
        - 잔고 부족인 경우 → InsufficientBalanceException을 던진다.
        - 코드를 이해하기 편하고 if 문을 남발하지 않는다.
        
        ```java
        try{
            BigDecimal balance = account.withdraw(amount);
            ...
        }catch(InsufficientBalanceException e){ //체크 예외
            BigDecimal availFunds = e.getAvailFunds();
            ...
        }
        ```
        
<br>
<br>

### 4.1.5 SQLException은 어떻게 됐나?

> 🤔 지금까지의 내용은 JdbcContext를 JdbcTemplate로 변경하니 SQLException이 왜 사라졌는가를 설명하는 데 필요한 것이었다. 이제부터는 DAO에 존재하는 SQLException에 대해 생각해보자!
> 

- 먼저 생각해볼 사항…
    - SQLException은 복구가 가능한가?
        
        ⇒ 99%는 아니기 때문에 예외처리 전략을 적용해야 한다.
        
    - 가능한한 빨리 언체크/런타임 예외로 전환해야 한다.

- 스프링의 JdbcTemplate은 이처럼 **체크 예외**인 SQLException을 **런타임 예외**인 DataAccessException으로 변경하여 예외를 던진다!
    - 따라서, DAO 메소드에서 SQLException이 모두 사라진 것이다.
 

<br>

## 4.2 예외 전환

### 4.2.1 JDBC의 한계외 전환

🤔 DB별로 다른 API를 제공하고 사용해야 한다면 어떻게 될까?

➡️ DB가 바뀔 때마다 DAO 코드도 모두 바뀌고 제각각 다른 API 사용법을 익혀야 한다!

<br>

- JDBC는 자바를 이용해 DB에 접근하는 방법을 추상화된 API 형태로 정의해놓고, 각 DB 업체가 jdbc 표준을 따라 만들어진 드라이버를 제공하게 해준다
    - 하지만, db 종류에 상관없이 사용할 수 있는 데이터 액세스 코드를 작성하는 일은 쉽지 않다.
    - DB를 자유롭게 바꾸어 사용할 수 있는 DB 프로그램을 작성하는 데 두 가지 걸림돌
        1. 비표문 SQL
        2. 호환성이 없는 SQLException의 DB 에러 정보

<br>

#### 1️⃣ 비표준 SQL

대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능도 제공한다.

작성된 비표준 SQL이 DAO 코드에 들어간다면, 해당 DAO는 특정 DB에 종속적인 코드가 된다

이를 해결하기 위해서는…

1. 호환되는 표준 SQL만 사용 → 페이징 쿼리에서부터 문제가 된다.
2. DB별로 별도의 DAO 만들기
3. SQL을 외부에 독립시켜 DB에 따라 변경해서 사용하기

해당 방법은 7장에서 다뤄볼 예정이다.

<br>

#### 2️⃣ 호환성 없는 SQLException의 DB 에러 정보

😮 DB마다 발생하는 에러의 종류, 원인이 제각각이다.

- 그런데 JDBC는 데이터 처리 중에 발생하는 다양한 예외를 그냥 SQLException 하나에 모두 담아버린다.
- 따라서, 원인을 확인하기 위해서는 **에러코드**와 **SQL 상태정보**를 참조해야 한다.

- add() 메소드에서 새로운 사용자를 등록하다가 키가 중복돼서 예외가 발생하는 경우
    
    ```java
    if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY) { ...
    ```
    
    - SQLException의 에러코드를 이용해 중복된 값의 등록이 원인인지 확인하는 것
    - 하지만 MySQL 전용 코드이므로 DB가 바뀐다면 동작 X
        
        → 따라서, **SQL의 상태 정보**도 부가적으로 제공한다!
        

- 하지만…
    - `getSQLState()`와 같은 메소드로 예외 상황에 대한 상태 정보를 가져올 수 있지만, 해당 값을 신뢰하기 힘들다. 어떤 DB는 표준 코드와 상관없는 엉뚱한 값이 들어있기도 하다.
    

> **결국 호환성 없는 에러 코드와 표준을 잘 따르지 않는 상태 코드를 가진 SQLException만으로 DB에 독립적인 유연한 코드를 작성하는 것은 불가능에 가깝다.**
> 

<br>
<br>

### 4.2.2 DB 에러 코드 매핑을 통한 전환

✨ DB 종류가 바뀌더라도 DAO를 수정하지 않으려면 앞의 두 가지 문제를 해결해야 한다.

- SQLException의 비표준 에러 코드와 SQL 상태정보에  대한 해결책을 알아보자

- 해결 방법
    - DB별 에러 코드를 참고해서 발생한 예외의 원인이 무엇인지 해석해주는 기능을 만들기
    - 문제는...
        - DB별로 에러 코드가 제각각이다!

- DB별로 다른 에러코드를
    - 스프링에서는 각 데이터베이스(DB)별 에러 코드를 분류하고, 이를 스프링이 정의한 예외 클래스와 매핑하여 관리
    - 이를 위해 에러 코드 매핑 정보 테이블을 만들어 활용

- 오라클 에러 코드 매핑 파일
    
    ```java
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
    
    - 이를 통해, **`JdbcTemplate`**은 DB 에러 코드를 적절한 **`DataAccessException`** 서브클래스로 매핑한다.

- JDBCTemplate이 제공하는 예외 전환 기능을 이용하는 add() 메소드
    
    ```java
    public void add(User user) throws DuplicateKeyException {
        this.jdbcTemplate.update("insert into users(id, name, password) values (?, ?, ?)"
                , user.getId()
                , user.getName()
                , user.getPassword()
        );
    }
    ```
    
- 중복 키 예외의 전환
    
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
    
    - 위와 같이 더 명확한 예외 클래스로 예외전환도 가능하다.

> 하지만 여전히 체크 예외라는 점과, SQL 상태정보를 이용한다는 점에서 문제점이 있다. 따라서 아직은 스프링의 에러 코드 매핑을 통한 DataAccessException 방식을 사용하는 것이 이상적이다.
> 

<br>
<br>

### 4.2.3 DAO 인터페이스와 DataAccessException 계층구조

💡 **DataAccessException**

- 스프링의 **`DataAccessException`** 계층 구조는 **데이터 액세스 기술에 상관없이 일관된 예외 발생**을 보장함.
- 다양한 데이터 액세스 기술에서 발생하는 예외를 공통의 예외 구조로 처리할 수 있게 해줌.

<br>

#### DAO 인터페이스와 구현의 분리

- **DAO를 인터페이스로 분리**하면 데이터 액세스 기술에 종속되지 않는 코드 작성이 가능해짐.
- **기술 독립적인 인터페이스 사용** → POJO를 통해 데이터 액세스 기능 호출.
- 특정 기술에서 발생하는 예외를 throw하지 않음 → 기술 변경 시 인터페이스에 영향을 미치지 않음.

- 만약 throw를 하게 되면…
    
    ```java
    public interface UserDao {
      public void add(User user) throws SQLException; // SQL 관련 예외
    }
    
    ```
    
    - 위 코드는 **`throws SQLException`**으로 인해 **데이터 접근 기술이 변경되면** 인터페이스 수정이 필요함.

- 기술에 독립적인 이상적인 DAO 인터페이스
    
    ```java
    public interface UserDao {
      public void add(User user); // throws 생략, 이렇게 선언하는 것이 가능할까?
    }
    
    ```
    
    - 결론은 사용할 수 없다!
    - DAO에서 사용하는 데이터 액세스 기술의 API가 예외를 던지기 때문…

- 따라서 아래와 같이 선언되어야 한다.
    
    ```java
    public interface UserDao {
      public void add(User user) throws PersistentException; // JPA
      public void add(User user) throws HibernateException; // Hibernate
      public void add(User user) throws JdoException; // JDO
    }
    ```
    
    - 인터페이스로 메소드의 구현은 추상화했지만 구현 기술마다 던지는 예외가 달라 메소드의 선언이 달라진다는 문제가 있다.
    - 가장 단순한 해결 방법
        - throws Exception으로 선언하는 것
        - 하지만 무책임하다!
    - 나은 해결 방법
        - 런타임 예외로 포장하자.
            
            **`public void add(User user);`**
            

**이제 DAO에서 사용하는 기술에 독립적인 인터페이스 선언이 가능해졌다!**

<br>

#### 데이터 액세스 예외 추상화와 DataAccessException 계층구조

- 스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해서 DataAccessException 계층구조 안에 정리해놓았다.
- **`JdbcTemplate`** 등 **스프링의 데이터 액세스 지원 기술을 통해 DAO**를 작성하면, 기술에 독립적인 일관된 예외를 던질 수 있음.

<br>
<br>

### 4.2.4 기술에 독립적인 UserDao 만들기

#### 인터페이스 적용

✨ UserDao 클래스를 인터페이스와 구현으로 분리해보자!

- 인터페이스로 만든 UserDao
    
    ```java
    public interface UserDao {
    	void add(User user);
    	User get(String id);
    	List<User> getAll();
    	void deleteAll();
    	int getCount();
    }
    ```
    
    - `setDataSource()`는 UserDao의 구현 방법에 따라 변경될 수 있어서 인터페잇에 추가하면 안 된다.

- 이제 기존의 UserDao 클래스는 이름을 변경하고 UserDao 인터페이스를 구현하도록 변경해주어야 한다.
    
    ```java
    public class **UserDaoJdbc implements UserDao** { 
    ```
    

- 스프링 설정파일의 userDao 빈 클래스 이름을 변경해주자.
    
    ```java
    <bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
    		<property name="dataSource" ref="dataSource" />
    </bean> 
    ```
    

<br>

#### 테스트 보완

😮 이제 남은 것은 기존 UserDao 클래스의 테스트 코드이다.

- UserDao 인스턴스 변수 선언도 UserDaoJdbc로 변경해야 할까?

- 굳이 그럴 필요는 없다.
    - UserDaoTest는 DAO의 기능을 검증하는 것이 목적이지 JDBC를 이용한 구현에 관심이 있는 게 아니다.
    - 따라서 UserDao라는 변수 타입을 그대로 두고 스프링 빈을 인터페이스로 가져오도록 만드는 편이 낫다.

- UserDaoTest에 중복된 키를 가진 정보를 등록했을 때 어떤 예외가 발생하는지를 확인하기 위해 테스트를 추가해보자.
    
    ```java
    @Test(expected=DataAccessException.class)
    public void duplicateKey() {
    	dao.deleteAll();
    	
    	dao.add(user1);
    	dao.add(user1);
    }
    ```
    
<br>

#### DataAccessException 활용 시 주의사항

이렇게 스프링을 활용하면

- DB 종류나 데이터 액세스 기술에 상관없이
- 키 값이 중복이 되는 상황에서는 동일한 예외가 발생하리라고 기대할 것이다.

하지만 안타깝게도 `DuplicateKeyException`은 JDBC를 이용하는 경우에만 발생한다!

- 데이터 액세스 기술을 하이버네이트나 JPA를 사용했을 때도 동일한 예외가 발생할 것으로 기대하지만 **실제로 다른 예외가 던져진다.**

따라서, `DataAccessException`을 잡아서 처리하는 코드를 만들려고 한다면 미리 학습 테스트를 만들어서 실제로 전환되는 예외의 종류를 확인해 둘 필요가 있다.

- 만약 DAO에서 사용하는 기술의 종류와 상관없이 **동일한 예외**를 얻고 싶다면?
    - `DuplicatedUserIdException`처럼 직접 예외를 정의해두고, 각 DAO의 `add()` 메소드에서 좀 더 상세한 예외 전환을 해주면 된다.

- 학습 테스트를 하나 더 만들어서 SQLException을 직접 해석해 `DataAccessException`으로 변환하는 코드의 사용법을 살펴보자
    - SQLException으로 전환 기능의 학습 테스트
        
        ```java
        @Test
        	public void sqlExceptionTranslate() {
        		dao.deleteAll();
        		
        		try {
        			dao.add(user1);
        			dao.add(user1);
        		}
        		catch(DuplicateKeyException ex) {
        			SQLException sqlEx = (SQLException)ex.getCause();
        			SQLExceptionTranslator set = new SQLErrorCodeSQLExceptionTranslator(this.dataSource);	// 코드를 이용한 SQLException의 전환		
        			
        			DataAccessException transEx = set.translate(null, null, sqlEx); // 에러 메세지 만들 때 사용하는 정보이므로 null로 넣어도 됨
        			assertThat(transEx, is(DuplicateKeyException.class));
        	}
        }
        ```
