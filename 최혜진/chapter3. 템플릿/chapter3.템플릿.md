# Chapter3. 템플릿
> `템플릿이란?`
>> 변경이 덜 하고, 일정한 패턴으로 유지되는 특성을 가진 부분
>> 자유롭게 변경되는 성질을 가진 부분으로부터 독립시킨다.

## 예외처리 기능을 갖춘 DAO
DB 커넥션이라는 제한적인 리소스를 공유해 사용하는 서버에서 동작하는 JDBC 코드에는 반드시 지켜야 하는 원칙이 있다.
<u>바로 예외처리다.</u>

`JDBC API 를 이용한 DAO 코드인 deleteAll()`
```java
//챕터 2 테스트에서 추가된 메소드  
public void deleteAll() throws SQLException, ClassNotFoundException {  
    Connection c = connectionMaker.makeConnection();  
    PreparedStatement ps = c.prepareStatement("delete from users");  
  
    ps.executeUpdate();  
    ps.close();  
    c.close();  
}
```
DB 커넥션은 DB의 소중한 자원이므로, 어떤 이유든 예외가 발생하더라도 리소스를 반드시 반환해야 한다.
그렇지 않으면 시스템에 큰 문제가 발생할 수 있다.

위의 코드는 한 번 실행되고 애플리케이션 전체가 종료되는 간단한 예제에선 괜찮지만, 장시간 운영되는 다중 사용자를 위한 서버에 적용하기는 치명적일 수 있다.

> `PreparedStatement` 를 처리하는 중에 예외가 발생한다면?
>> 메소드 실행을 끝마치지 못 하고 바로 메소드를 빠져나가게 되어 Connection과 PreparedStatement의 close() 메소드가 실행되지 않아 제대로 리소스가 반환되지 않을 수 있다.

`예외 발생 시에도 리소스를 반환하도록 수정한 deleteAll()`
```java
//챕터 2 테스트에서 추가된 메소드  
public void deleteAll() throws SQLException, ClassNotFoundException {  
    Connection c= null;  
    PreparedStatement ps = null;  
    try{  
        //예외가 발생할 가능성이 있는 코드를 모두 try 블록으로 묶어준다.  
        c= connectionMaker.makeConnection();  
        ps = c.prepareStatement("delete from users");  
        ps.executeUpdate();  
    }catch (SQLException e) {  
        //예외가 발생했을 때 부가적인 작업을 해줄 수 있도록 catch 블록을 둔다.  
        //아직은 예외를 다시 메소드 밖으로 던지는 것 밖에 없음  
        throw e;  
    }finally { // finally 이므로 try 블록에서 예외가 발생했을 때나 안 했을 때나 모두 실행된다.  
        if(ps != null){  
        //try/catch 블록 없이 ps.close()를 처리하다가 예외가 발생하면 아래 
        // c.close() 부분이 실행되지 않고 메소드를 빠져나가는 문제가 발생하기 때문이다.
            try{  
                ps.close();  
            }catch (SQLException e){  
                //ps.close() 메소드에서도 SQLException이 발생할 수 있기 때문에  
                //이를 잡아줌  
                //그렇지 않으면 Connection 을 close() 하지 못 하고 메소드를 빠져나갈 수 있다.  
            }  
        }  
        if(c != null){  
            try{  
                c.close(); //Connection 반환  
            }catch (SQLException e){  
            }        }  
    }   
}
```

`JDBC 예외처리를 적용한 getCount() 메소드`
조회를 위한 JDBC 코드는 좀 더 복잡하다.
```java
//챕터 2 테스트에서 추가된 메소드  
public int getCount() throws SQLException, ClassNotFoundException {  
    Connection c = null;  
    PreparedStatement ps = null;  
    ResultSet rs = null;  
    try {  
        c = connectionMaker.makeConnection();  
        ps = c.prepareStatement("select count(*) from users");  
        //ResultSet도 다양한 SQLException이 발생할 수 있는 코드이므로, try 블록으로 묶어준다.  
        rs = ps.executeQuery();  
        rs.next();  
        return rs.getInt(1);  
    } catch (SQLException e) {  
        throw e;  
    } finally {  
        //만들어진 ResultSet 을 닫아주는 기능  
        //close()는 만들어진 순서의 반대로 하는 것이 원칙  
        if (rs != null) {  
            try {  
                rs.close();  
            } catch (SQLException e) {  
            }        }  
        if (ps != null) {  
            try {  
                ps.close();  
            } catch (SQLException e) {  
            }        }  
        if (c != null) {  
            try {  
                c.close();  
            } catch (SQLException e) {  
            }        }  
    }
```

> `리소스 반환과 close()`
>>Conncetion이나 PreparedStatement 에는 close() 메소드가 있다.
>>리소스를 반환한다는 의미로 이해하는 것이 좋다.
>
>>Connection과 PreparedStatement 는 보통 풀(pool) 방식으로 운영된다.
>>미리 정해진 풀 안에 제한된 수의 리소스를 만들어두고 필요할 때 이를 할당하고, 반환하면 다시 풀에 넣는 방식으로 운영된다.
>
>> 요청이 많은 서버환경에서는 매번 새로운 리소스를 생성하는 대신 풀에 미리 만들어둔 리소스를 돌려가며 사용하는 것이 유리하다.
>> 대신, 사용한 리소스는 빠르게 반환해야 한다.
>> 그렇지 않으면, 리소스가 고갈되고 결국 문제가 발생한다.
## 변하는 것과 변하지 않는 것
### JDBC try/catch/finally 코드의 문제점
- 복잡한 try/catch/finally 블록이 2중으로 중첩까지 되어 나오는데다, 모든 메소드마다 반복된다.

### 분리와 재사용을 위한 디자인 패턴 적용
로직에 따라서 변하는 부분을 변하지 않는 나머지 코드에서 분리하는 것은 어떨까?
#### 메소드 추출
`방법: 변하는 부분을 메소드로 빼는 것`
별 이득이 없어 보인다. 왜냐 보통 <u>메소드 추출 리팩토링을 적용하는 경우에는 분리시킨 메소드를 다른 곳에서 재사용할 수 있어야</u> 하는데, 이건 반대로 분리시키고 남은 메소드가 재사용이 필요한 부분이고, 분리된 메소드는 DAO 로직마다 새롭게 만들어서 확장돼야 하는 부분이기 때문에 **뭔가 반대로 됐다!**

#### 템플릿 메소드 패턴 적용
상속을 통해 기능을 확장해서 사용하는 부분이다.
`변하지 않는 부분`
- 슈퍼 클래스에 둔다.
 `변하는 부분`
- 추상 메소드로 정의해둬서  서브클래스에서 오버라이드 해 새롭게 정의해 쓰도록 한다.

```java
public abstract class UserDao {
    PreparedStatement makeStatement(Connection c) throws SQLException {
        return null;
    }
```

```java
public class UserDaoDeleteAll extends UserDao {
    @Override
    protected PreparedStatement makeStatement(Connection c) throws SQLException {
        return c.prepareStatement("delete from users");
    }
}
```

UserDao 클래스의 기능을 확장하고 싶을 때마다 상속을 통해 자유롭게 확장할 수 있고, 확장 때문에 기존 상위 DAO 클래스에 불필요한 변화는 생기지 않도록 할 수 있으므로 `객체지향 설계 핵심 원리인 개방 폐쇄 원칙`을 그럭저럭 지킨 구조이다.

![img.png](img.png)

> 템플릿 메소드 패턴을 이용했을 때의 문제점
>> 1. DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다.
>> 2. 확장구조가 이미 클래스를 설계하는 시검에서 고정된다.

#### 전략 패턴 적용
개방 폐쇄 원칙을 잘 지키는 구조이면서도 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어난 것이, 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하게 만드는 전략 패턴이다.

전략 패턴은 OCP 관점에서 보면 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식이다.
![img_1.png](img_1.png)
좌측의 Context의 contextMethod()에서 일정한 구조를 가지고 동작하다가 특정 확장 기능은 strategy 인터페이스를 통해 외부의 독립된 전략 클래스에 위임하는 것이다.

`deleteAll() 메소드에서 변하지 않는 부분 :`  contextMethod()

> deleteAll() 컨텍스트
> 1. DB 커넥션 가져오기
> 2. preparedStatement 를 만들어줄 외부 기능 호출 → 전략패턴에서 말하는 전략
> 3. 전달받은 preparedStatement 실행
> 4. 예외가 발생하면 이를 다시 메소드 밖으로 던지기
> 5. 모든 경우에 만들어진 preparedStatement 와 Connection 을 적절히 닫아주기

`StatementStrategy라는 인터페이스로 전략을 정의`
```java
  package springbook.user.dao;
  ...
  public interface StatementStrategy {
      PreparedStatement makePrearedStatement(Connection c) throws SQLException;
  }
```

`이 인터페이스를 상속해 실제 바뀌는 코드 부분을 클래스로 만듦`
```java
  package springbook.user.dao;
  ...
  public class DeleteAllStatement implements StatementStrategy{
    @Override
    public PreparedStatement makePrearedStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.prepareStatement("delete from users");
        return ps;
    }
  }
```

`마지막 바뀌지 않는 부분 (맥락) 적용`
```java
  public void deleteAll() throws SQLException {
    ...
    try {
        c = dataSource.getConnection();

        StatementStrategy strategy = new DeleteAllStatement();
        ps = strategy.makePrearedStatement(c);

        ps.executeUpdate();
    } catch (SQLException e){
    ...
```
##### 문제점
전략패턴은 필요에 따라 컨텍스트는 그대로 유지되면서 전략을 바꿔 쓸 수 있다는 것인데,
이렇게 컨텍스트 안에서 이미 구체적인 전략 클래스인 DeleteAllStatement 를 사용하도록 고정되어 있다.

전략 패턴에도 OCP 에도 잘 들어맞지 않는다.

#### DI 적용을 위한 클라이언트/컨텍스트 분리

![img_2.png](img_2.png)
Context가 어떤 전략을 사용할지 정하는 것은 Context를 사용하는 Client 가 결정하는게 더 유연한 디자인이 된다.

<mark>위와같은 방식으로 되려면? StatementStrategy를 컨텍스트 메소드의 파라메터로 지정하면 해결된다. </mark>

`메소드로 분리한 try/catch/finally 컨텍스트 코드`
```java
  public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();

        ps = stmt.makePrearedStatement(c);

        ps.executeUpdate();
    ...
```

`클라이언트 책임을 담당할 deleteAll() 메소드`
```java
  public void deleteAll() throws SQLException {
      StatementStrategy st = new DeleteAllStatement();
      jdbcContextWithStatementStrategy(st);
  }
```

## JDBC 전략 패턴의 최적화
### 전략 클래스의 추가 정보
UserDao 의 add() 메서드 적용
- add() 도 동일한 과정을 거친다.
- 차이점이라고 할 수 있는 부분은 add 할 대상 User 오브젝트를 StatementStrategy의 생성자에서 받는 것이다.
### 전략과 클라이언트와의 동거
- `개선할 부분`
    - DAO  메소드마다 새로운 StatementStrategy 구현 클래스가 만들어져야 함
    -  add()메소드의 User 처럼 부가정보가 필요할 경우에 오브젝트를 전달 받는 생성자와 이를 저장해둘 인스턴스 변수를 번거롭게 만들어야 함

- `해결 방안`
    -  로컬 클래스
        - StatementStrategy 구현 클래스를 UserDao 클래스 안의 내부 클래스로 구현함(특정 StatementStrategy 구현 클래스는 UserDao의 메소드 로직과 강하게 결합되어 있기 때문)
        - 로컬 클래스는 선언된 클래스 안에서만 사용할 수 있음
        - 로컬클래스는 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있음
            - User 직접 전달 필요없음(User final 선언 후 접근가능)
```java
  public void add(final User user) throws SQLException {
      class AddStatement implements StatementStrategy{

          @Override
          public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
              PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?, ?, ?)");
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

	- 익명 클래스
		- 이름을 붙이지 않은 클래스
		- 클래스 선언과 오브젝트 생성이 결합된 형태로 만들어짐
		- 클래스를 재사용할 필요가 없고 구현한 인터페이스 타입으로만 사용할 경우에 유용 
		- 좀 더 간략하게 익명 내부클래스로 add() 와 deleteAll() 을 변경함


## 컨텍스트와 DI
### JdbcContext 의 분리
- 클라이언트 - UserDao 의 메소드
- 개별 전략 - 익명 내부 클래스
- 컨텍스트 - jdbcContextWithStatementStrategy() 메소드
- jdbcContextWithStatementStrategy() 는 다른 DAO 에서도 사용 가능하다.
- jdbcContextWithStatementStrategy() 를 UserDao에서 분리하면 다른 DAO도 쓸 수 있다.

![img_3.png](img_3.png)

-  아직 모든 UserDao의 모든 메소드가 JdbcContext를 사용하는 것은 아니다!
### JdbcContext 의 특별한 DI
- 스프링 bean으로 DI

    - 인터페이스가 아닌 JdbcContext를 DAO 에 DI 해도 되는걸까? 인터페이스로 DI 해야하지 않을까?
    - JdbcContext같이 클래스를 직접 DI 구조로 가야하는 이유
        - JdbcContext를 싱글톤 빈으로 만들어야 하기 때문 (일종의 Service 오브젝트 성격)
        - JdbcContext가 DI 를 통해 다른 bean(DataSource)에 의존하기 때문
            - 스프링에서 DI를 하기 위해서 주입되는 오브젝트와 주입받는 오브젝트 둘다 bean으로 등록되어 있어야 함
        - 인터페이스가 없는 이유는? JdbcContext와 Dao 사이에 강한 응집도를 가지기 때문
    - But! 인터페이스 없이 클래스를 DI하는 상황은 늘 차선책으로 생각하고 개발할 것을 권유
- 코드를 이용하는 수동 DI

    - JdbcContext를 스프링 bean으로 등록하지 않고, UserDao 내부에서 직접 생성후 DI하는 방식
    - 수동 DI 시 문제 2가지
        - 문제점1. 싱글톤으로 만들 수 없다.
            - Dao에서 생성하는 방식이기 때문에 Dao갯수만큼 JdbcContext객체를 생성함
            - 왠만한 대형 프로젝트라도 수백개 이상은 만들어지지 않을 것
            - 따라서 용인될 만한 수준
            - 빈번히 오브젝트가 만들어지고 제거되는 것도 아니니 GC에 무리 없음
        - 문제점2. DataSource는 어떻게 DI 받나?
            - JdbcContext 자신이 Bean이 아니니 DataSource를 DI 받을 수 없음
            - 해결책 JdbcContext에 대한 제어권을 가지고 생성과 관리를 담당하는 UserDao에게 DI를 맞기는 것
    - 수동 DI를 하는 이유
        - 굳이 인터페이스를 두지 않아도 될 만큼 긴밀한 관계인 DAO클래스와 JdbcContext를 어색하게 빈으로 분리하지 않고 내부에서 직접 만들어 사용하면서 다른 오브젝트에 대한 DI를 적용할 수 있기 때문
        - JdbcContext와 UserDao의 관계를 외부에 드러내지 않고 사용 가능하게 하는 것이 목적
        - 그러나 싱글톤 문제와 DI 작업의 부차적 코드가 필요함은 단점
## 템플릿과 콜백
전략 패턴의 기본 구조에 익명 내부 클래스를 확용한 방식 - 템플릿/콜백 패턴
- 전략 패턴의 컨텍스트 - 템플릿
- 익명 내부 클래스로 만들어지는 오브젝트 - 콜백
#### 템플릿 콜백의 동작원리
- 템플릿 - 고정된 작업 흐름을 가진 코드를 재사용

- 콜백 - 템플릿 안에서 호출되는 것을 목적으로 만든 오브젝트

- 특징

    - 콜백 - 보통 단일 메소드 인터페이스 사용
        - 템플릿의 작업흐름 중 특정기능을 위해 한 번 호출되는 경우기 일반적이기 때문
    - 콜백 인터페이스의 파라메터 - 템플릿의 작업흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용
        - 예) makePreparedStatement() 의 Connection 객체, connection은 workWithStatementStrategy() 메소드 내에서 생성되어 전달된다.
    - DI 방식의 전략 패턴 구조
    - 클라이언트가 템플릿 메소드를 호출하면서 콜백 오브젝트를 전달하는 것은 메소드 레벨에서 일어나는 DI이다
    - 전략패턴과 DI의 장점을 익명 내부 클래스 사용 전략과 결합한 독특한 활용법
![img_4.png](img_4.png)

#### 편리한 콜백의 재활용
- 콜백의 분리와 재활용

    - 콜백조차도 반복되는 패턴이 있다 - 단순히 prepareStatement메소드 내 sql파라메터만 바뀌는 경우가 많음
    - 중복되는 부분과 변화되는 부분을 분리

    ```java
      public void deleteAll() throws SQLException {
          executeSql("delete from users");
      }
    ```

    ```java
      private void executeSql(final String query) throws SQLException {
          jdbcContext.workWithStatementStrategy(new StatementStrategy() {
              @Override
              public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                  PreparedStatement ps = c.prepareStatement(query);
                  return ps;
              }
          });
      }
    ```

- 콜백과 템플릿의 결합

    - 한 단계 더 나아가 executeSql메소드는 다른 DAO 에서도 사용 될 수 있다.
    - 그렇다면 JdbcContext로 옮겨서 사용하게 하는게 어떨까?

    ```java
      public void deleteAll() throws SQLException {
          this.jdbcContext.executeSql("delete from users");
      }
    ```

    ```java
    public class JdbcContext {
      ...
      public void executeSql(final String query) throws SQLException {
          workWithStatementStrategy(new StatementStrategy() {
              @Override
              public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                  PreparedStatement ps = c.prepareStatement(query);
                  return ps;
              }
          });
      }
      ...
    }
    ```

- 일반적으로 성격이 다른 코드들은 가능한 분리하지만, 반대로 하나의 목적을 위해 서로 긴밀하게 연결되어 동작하는 응집력이 강한 코드들은 한 군데 모여 있는 것이 유리
#### 템플릿/콜백의 응용
- 스프링은 이 템플릿/콜백 패턴을 적극적으로 응용함

- 스프링에는 다양한 자바 엔터프라이즈 기술에서 사용할 수 있도록 미리 만들어져 제공되는 수십 가지 템플릿/콜백 클래스와 API 가 존재

- 스프링이 내장된 것을 원리도 알지 못한 채 기계적으로 사용하는 경우와 적용된 패턴을 이해하고 사용하는 경우는 큰 차이가 있음

- 고정된 작업흐름을 가지고 여기저기 자주 반복되는 코드가 있다면, 중복되는 코드를 분리할 방법을 생각해보는 습관을 기르자

- 중복된 코드를 먼저 메ㅔ소드로 분리하는 간단한 시도를 해본다

- 일부 작업을 필요에 따라 바꾸어 사용해야 한다면? 인터페이스를 사이에 두고 분리해서 전략 패턴을 적용한다

- 바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 템플릿/콜백 패턴을 적용한다

- 대표적 템플릿/콜백 패턴 후보 - try/catch/finally 블록을 사용하는 코드

- calcSum() 예제를 통해 템플릿/콜백 패턴 변환과정 복습(소스코드 참조)

- calcSum에 try/catch/finally 적용

- 중복의 제거와 템플릿/콜백 설계

    - 추가 요구사항 - 모든 숫자의 곱을 계산하는 기능, 그리고 파일에 담긴 숫자를 다양한 방법으로 처리하는 기능이 대거 추가될 예정..
    - 어떻게 대처할 것인가? 복붙(복사해서 붙여넣기)? NO...
    - 여기도 템플릿/콜백 패턴 적용
        - 템플릿-콜백 둘 사이의 경계정하기
        - 템플릿 -> 콜백 전달해줄 내부의 정보는?
        - 콜백 -> 템플릿 돌려줄 내용은?
    - BufferedReader를 전달 받아 결과값을 돌려주는 콜백 적용
    - 숫자의 합 / 숫자의 곱 기능 같이 사용
- 템플릿/콜백의 재설계

    - calcSum()과 calcMultiply() 에 나오는 두 개의 콜백을 비교해 보면 유사한 부분(중복)이 또 보임
    - 실제로는 각 라인을 읽어들일 때 최종결과에 어떻게 반영하느냐 하는 코드만 다름
    - 또 한번 라인별 콜백을 추가하는 리팩토링을 거쳐 중복을 최소화 해보자!(코드 참고)
- 제네릭스를 이용한 콜백 인터페이스

    - 좀 더 강력한 템플릿/콜백 구조를 만들려면?
    - 현재까지는 결과를 Integer 타입만 가능하게 고정되어 있음
    - 결과 타입을 다양하게 하고 싶다면? Generics(제네릭스)를 이용해 보자(코드 참고)
### 스프링의 JdbcTemplate
#### Update()
- executeSql() 과 비슷한 메소드가 JdbcTemplate의 update()라는 메소드로 존재한다.
#### queryForInt()
- getCount()는 SQL 쿼리를 실행하고 ResultSet을 통해 결과 값을 가져오는 코드
- 콜백이 2개 등장하는 조금 복잡해 보이는 구조
- 첫번째 PreparedStatementCreator 콜백은 템플릿으로부터 Connection을 받고 PreparedStatement를 돌려줌
- 두 번째 ResultSetExtractor 콜백은 템플릿으로부터 ResultSet을 받고 거기서 추출한 결과를 돌려줌
- 결국 2번째 콜백에서 리턴하는 값은 결국 템플릿 메소드의 결과로 다시 리턴됨
- ResultSetExtractor는 제너릭스 타입 파라미터를 갖는다
- 즉 유연하고 재사용하기 쉬운 구조로 잘 되어있다
- 이 제법 복잡해 보이는 구문도 한 줄로 바꿀 수 있다.(queryForInt로)
- **그러나 queryForInt() 메소드는 에석하게 스프링 3.2.2. 이후로는 Deprecated 되어 버렸다**
- 대신 queryForObject()로 대신할 수 있다.

```java
  public int getCount() throws SQLException {
    return this.jdbcTemplate.queryForObject("select count(*) from users", Integer.class);
  }
```

#### queryForObject()
- getCount()는 SQL 쿼리를 실행하고 ResultSet을 통해 결과 값을 가져오는 코드
- 콜백이 2개 등장하는 조금 복잡해 보이는 구조
- 첫번째 PreparedStatementCreator 콜백은 템플릿으로부터 Connection을 받고 PreparedStatement를 돌려줌
- 두 번째 ResultSetExtractor 콜백은 템플릿으로부터 ResultSet을 받고 거기서 추출한 결과를 돌려줌
- 결국 2번째 콜백에서 리턴하는 값은 결국 템플릿 메소드의 결과로 다시 리턴됨
- ResultSetExtractor는 제너릭스 타입 파라미터를 갖는다
- 즉 유연하고 재사용하기 쉬운 구조로 잘 되어있다
- 이 제법 복잡해 보이는 구문도 한 줄로 바꿀 수 있다.(queryForInt로)
- **그러나 queryForInt() 메소드는 에석하게 스프링 3.2.2. 이후로는 Deprecated 되어 버렸다**
- 대신 queryForObject()로 대신할 수 있다.

```java
  public int getCount() throws SQLException {
    return this.jdbcTemplate.queryForObject("select count(*) from users", Integer.class);
  }
```
#### query()
- getAll() 메소드 추가 - 테이블의 모든 User 로우를 가져온다

- List 타입으로 돌려줌

- 먼저 getAll()을 검증하는 Test code 부터 작성(코드 참조)

- getAll은 RowMapper 콜백 오브젝트에서 ResultSet에서 User로 변환하는 로직 작성

- 기능 완성후 테스트 수행해보면 깔끔하게 성공한다.

- 테스트 보완

    - 만약 getAll의 결과가 없다면?
        - null ?? Exception ?? 정하기 나름
    - JdbcTempate의 query()는 결과가 없을 경우 크기가 0 인 List 오브젝트 반환
    - 이미 스프링에서 동작이 정해진 코드도 굳이 검증코드를 추가해야 하나?
        - 테스트에서 관심있는 것은 getAll() 메소드의 실행 결과
        - 중간에 getAll()의 구현을 바꿀 수도 있음. 그래서 test 코드는 필요
        - 내부적으로 query()를 사용했다고 해도 결과를 getAll() 에서 바꿔서 구현했을 수도 있음
#### 재사용 가능한 콜백의 분리
- 이제 UserDao 코드는 처음 try/catch/finally가 덕지덕지 붙여있을 때의 메소드 1개 분량밖에 되지 않음

- 각 메소드의 기능 파악도 쉬움

- 핵심적인 SQL 문장, 파라메터, 생성 결과 타입정보만 남기에 코드를 파악하기 쉬움

- DI 를 위한 코드 정리 - 필요없는 DataSource 인스턴스 변수 제거

- 중복 제거

    - 코드를 보면 get()과 getAll()의 RowMapper의 내용이 똑같음
    - 하나의 User테이블 row를 User오브젝트로 변환하는 로직은 자주 사용될 것으로 예상된다
    - 그리고 향후 User 테이블의 필드 수정,추가가 발생하면 같은 역할의 중복 RowMapper가 있다면 빼먹기 쉽다
    - 현재 콜백을 메소드에 분리해 중복을 제거후 재사용해 보자(코드 참조)
    - 이제 군더더기 없는 UserDao 코드가 완성되었다!
- 탬플릿/콜백 패턴과 UserDao

    - UserDao - User 정보를 DB에 넣거나 가져오거나 조작하는 방법에 대한 핵심적인 로직이 담김
    - JdbcTemplate - JDBC API를 사용하는 방식, 예외처리, 리소스의 반납, DB연결을 어떻게 가져올지에 관한 책임과 관심
    - UserDao와 JdbcTemplate 사이에는 강한 결합을 가지고 있음
    - 여기서 더 개선 가능한가?
        - userMapper가 인스턴스 변수로 설정되어 있고, 한 번 만들어지면 변경 불가능
            - 중도 변경 가능하게 UserMapper를 독립된 빈으로 만들고 DI 하게 만들면?
        - DAO 내 사용중인 SQL문장을 코드가 아니라 외부 리소스에 담고 이를 읽어와 사용하게 하는 것(이후 장에서 다루게 됨)

## 요약
> `이 장에서의 목표`
>> 1. 반복적인 코드 패턴을 추출하고 재사용 가능한 형태로 구조화 하는 것
>> 2. 코드의 중복을 줄이고 유연한 구조를 갖추도록 하는 것
### 템플릿과 콜백 패턴
- **템플릿 메서드 패턴**의 변형으로, 스프링에서는 이를 사용해 특정 작업의 공통적인 부분을 템플릿으로 만들고, 변화하는 부분을 콜백(callback) 메서드로 분리한다.
- 템플릿 메서드는 알고리즘의 구조를 정의하고, 일부 구현은 서브클래스나 콜백을 통해 제공한다.
- 스프링에서는 이러한 템플릿 패턴을 **JdbcTemplate**, **RestTemplate** 등에서 활용하여 데이터베이스 접근, HTTP 통신 등의 반복적인 작업을 단순화한다.
### JdbcTemplate의 활용
- **JdbcTemplate**은 JDBC 작업에서 반복적으로 사용되는 **커넥션 획득**, **SQL 실행**, **리소스 해제** 등의 작업을 템플릿화하여, 개발자는 SQL 처리 로직만 작성할 수 있도록 한다.
- 이를 통해 JDBC 코드를 간결하게 작성할 수 있으며, 예외 처리나 리소스 관리에 신경 쓰지 않아도 된다.
- 핵심 메서드:
    - `queryForObject()`: SQL을 실행하고 결과를 객체로 매핑
    - `update()`: 데이터베이스에 데이터 삽입, 업데이트, 삭제 작업
    - `query()`: 여러 개의 결과를 리스트로 받아오는 작업
### 템플릿 패턴의 장점
- **중복 코드 제거**
  반복되는 작업을 템플릿에 위임하여 코드의 중복을 줄인다.
- **변경에 유연**
  템플릿을 사용하면 코드 변경 시에도 공통된 템플릿 구조는 유지하고, 변화가 필요한 부분만 콜백으로 변경할 수 있다.
- **높은 응집도와 낮은 결합도**
  템플릿은 전체적인 작업 흐름을 관리하고, 세부적인 구현은 콜백으로 분리하여 코드의 응집도를 높이고, 의존성을 줄인다.
### 전략 패턴을 이용한 템플릿 적용
- 스프링에서 템플릿 패턴은 **전략 패턴**과 결합하여 사용된다.
- 전략 패턴은 변할 수 있는 알고리즘을 인터페이스를 통해 추상화하고, 템플릿은 이러한 알고리즘을 실행하는 흐름을 담당한다.
- 예시로, `StatementStrategy` 인터페이스를 이용해 SQL 실행 전략을 정의하고, 템플릿 메서드가 이를 호출하도록 구성한다.
- 이 패턴을 통해 SQL 구문이나 비즈니스 로직에 따라 커스터마이즈된 전략을 템플릿에서 재사용할 수 있다.
### 템플릿의 적용 예시
- **UserDao의 리팩토링 과정**:
    - 초기에 JDBC 코드가 반복되는 문제를 해결하기 위해, 스프링의 템플릿 기법을 적용해 **JdbcContext** 클래스를 만들고, SQL 실행의 변하는 부분을 콜백으로 분리한.
    - 이후 **JdbcTemplate**을 직접 사용하여 더 간결하고 유지보수하기 좋은 형태로 발전시킨다.
- **익명 내부 클래스와 람다 사용**:
    - 콜백을 익명 내부 클래스로 구현하거나, 자바 8 이상에서는 람다 표현식을 사용하여 간결하게 표현할 수 있다.
### 콜백의 재사용과 중첩 템플릿
- 단순한 작업 외에도, 복잡한 로직이 필요할 때 **중첩된 템플릿 구조**를 사용할 수 있다.
- 예를 들어, 트랜잭션 처리와 관련된 템플릿을 작성하여, 특정 로직이 트랜잭션 내에서 실행되도록 제어할 수 있다.
- 이를 통해 트랜잭션 관리, 예외 처리 등을 일관되게 적용할 수 있게 된다.

## 질문
Q. 템플릿 메서드 패턴과 템플릿/콜백 패턴의 차이점은 무엇인가요?
<details><summary>답안</summary>템플릿 메서드 패턴은 상속을 통해 공통된 알고리즘의 구조를 정의하고, 세부적인 처리는 서브클래스가 담당하게 한다. 메서드의 일부를 서브클래스에서 재정의하도록 하는 방식이다.
반면, 스프링에서 말하는 템플릿/콜백 패턴은 상속 대신 인터페이스와 익명 클래스 또는 람다를 통해 공통 작업과 변하는 부분을 분리한다. 이를 통해 코드의 재사용성을 높이고, 유연하게 변화를 주기 쉽다.</details>
