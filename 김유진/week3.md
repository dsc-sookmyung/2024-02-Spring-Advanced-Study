# 3장. 템플릿

### ✨ 스프링에 적용된 템플릿 기법 살펴보기, 이를 적용해 완성도 있는 DAO 코드 만들기

## 3.1 다시 보는 초난감 DAO

🤔 UserDao 코드는 아직 예외상황에 대한 처리와 관련한 문제점이 남아있다!

<br>

### 3.1.1 예외처리 기능을 갖춘 DAO

- JDBC 코드에는 반드시 지켜야할 원칙이 있음
    
    ⇒ **예외처리**!
    
    - 예외가 발생시 사용한 리소스를 반환하지 못하면 시스템에 심각한 문제 발생

#### JDBC 수정 기능의 예외처리 코드

- deleteAll() 메소드
    
    ```java
    public void deleteAll() throws SQLException {
        Connection c = dataSource.getConnection();
    
        PreparedStatement ps = c.preparedStatement("delete from users");
        ps.executeUpdate();
    
        ps.close();
        c.close();
    }
    ```
    
    - PreparedStatement를 처리하는 중에 예외가 발생하면 어떻게 될까?
        - 메소드 실행을 끝마치지 못하고 바로 메소드를 빠져나감.
        - Connection과 PreparedStatement의 close() 메소드가 실행되지 않아서 제대로 리소스가 반환되지 않을 수 있음.
    
    ⇒ 일반적으로 서버는 제한된 개수의 DB 커넥션을 만들어서 재사용 가능한 풀로 관리하기 때문에, 반환되지 못한 Connection이 계속 쌓이면 **어느 순간 서버가 중단**될 수도 있다.
    
    ⇒ **try/catch/finally 구문**을 사용하자!
    

- 예외 발생 시에도 리소스를 반환하도록 수정한 deleteAll()
    
    ```java
    public void deleteAll() throws SQLException {
        Connection c = null;
        PreparedStatement ps = null;
    
        try {
            c = dataSource.getConnection();
            ps = c.preparedStatement("delete from users");
            ps.executeUpdate();
        } catch (SQLException e) {
            throw e;
        } finally {
            if (ps != null) {
                try {
                    ps.close(); // 커넥션 반환
                } catch (SQLException e) {
                }
            }
            if (ps != null) {
                try {
                    c.close(); // 커넥션 반환
                } catch (SQLExeption e) {
                }
            }
        }
    }
    ```
    
    - 어느 시점에 예외가 나는가에 따라 Connection과 PreparedStatement 중 어떤 것의 close() 메소드를 호출해야 할지가 달라짐.
        - finally에서 반드시 c와 ps가 null이 아닌지 먼저 확인한 후 close() 메소드를 호출해야 함.
        

#### JDBC 조회 기능의 예외처리

✨ 등록된 User의 수를 가져오는 getCount() 메소드에 예외처리 블록을 적용해보자!

- JDBC 예외처리를 적용한 getCount() 메소드
    
    ```java
    public int getCount() throws SQLException {
    
        Connection c = null;
        PreparedStatement ps = null;
        ResultSet rs = null;
    
        try {
            c = dataSource.getConnection();
            
            ps = c.preparedStatement("select * from users");
    
            rs = ps.executeQuery();
            rs.next();
            return rs.getInt(1);
        } catch (SQLException e) {
            throw e;
        } finally {
            if (ps != null) {
                try {
                    rs.close();
                } catch (SQLException e) {
                }
            }
            if (ps != null) {
                try {
                    ps.close();
                } catch (SQLException e) {
                }
            }
            if (ps != null) {
                try {
                    c.close();
                } catch (SQLExeption e) {
                }
            }
        }
    }
    ```
    
    ⇒ 이제 이상적인 DAO가 완성됐다.
    
    🤔 하지만! 여전히 뭔가 아쉬움이 남아 있다.


<br>

## 3.2 변하는 것과 변하지 않는 것

### 3.2.1 JDBC try/catch/finally 코드의 문제점

- try/catch/finally 블록이 적용된 UserDao 코드의 문제점
    - 복잡한 try/catch/finally 블록이 중첩됨.
    - 모든 메소드마다 반복됨.

💡 이런 코드를 작성할 때 사용할 수 있는 가장 효과적인 방법

- Copy&Paste
    - 그런데, **한 줄을 빼고 복사**하거나 **몇 줄을 잘못 삭제**하면 어떻게 될까?
    - 커넥션이 하나씩 반환되지 않고 쌓여가면서 추후에는 **서비스가 중단** 될 것이다.

🤔 이런 코드를 **효과적으로 다룰 수 있는 방법**은 없을까?

- 변하지 않는, 그러나 많은 곳에서 중복되는 코드와 로직에 따라 자꾸 확장되고 변하는 코드를 잘 분리해내야 한다!

### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용

✨ UserDao의 메소드에서 변하는 성격이 다른 것을 찾아내보자!

- deleteAll()에서 변하는 부분
`ps = c.preparedStatement("delete from users");`
- 로직에 따라 변하는 부분을 변하지 않는 나머지 코드에서 분리해서 변하지 않는 부분을 재사용할 수 있는 방법이 있을까?

#### 메소드 추출

💡 **변하는 부분을 메소드로 추출**하자!

```java
public void deleteAll() throws SQLException {

    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();
        
        ps = makeStatement(c)

        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
      ...
    }
}

private PreparedStatement makeStatement(Connection c) throws SQLException {
    PreparedStatement ps;
    ps = c.preparedStatement("delete from users");
    return ps;
}
```

- 분리시키고 남은 메소드 → 재사용이 필요한 부분
- 분리된 메소드 → 새롭게 만들어서 확장돼야 하는 부분

‼️**뭔가 반대로 됐다…!**

#### 템플릿 메소드 패턴의 적용

💡 **템플릿 메소드 패턴**을 이용해서 분리해보자!

- 템플릿 메소드 패턴?
    - 상속을 통해 기능을 확장해서 사용하는 부분
        - 변하지 않는 부분: 슈퍼클래스
        - 변하는 부분: 추상메소드로 정의 → 서브클래스에서 오버라이드하여 새롭게 정의해서 쓰도록 하는 것.
- 별도의 메소드로 독립시킨 makeStatement() 메소드를 추상 메소드 선언으로 변경
    
    ```java
    abstract protected PreparedStatement makeStatement(Connection c)
        throws SQLException;
    ```
    
    - 상속을 통해 자유롭게 확장 가능
    - 확장 때문에 기존의 상위 DAO 클래스에 불필요한 변화 X
        - 하지만…
            - DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 하고,
            - 확장구조가 이미 클래스를 설계하는 시점에서 고정되어 유연성이 떨어진다.

#### 전략 패턴의 이용

💡 **전략 패턴**

- 개방 폐쇠 원칙을 잘 지키면서 유연하고 확장성이 뛰어난 것
- 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 것

![image](https://github.com/user-attachments/assets/22d2f091-52ad-4eb5-a15e-bc6000034026)


- 좌측의 Context의 `contextMethod()`에서 일정한 구조를 가지고 동작하다가,
- **특정 확장 기능**은 Strategy 인터페이스를 통해 외부의 독립된 전략 클래스에 위임함.
- `deleteAll()` 의 변하지 않는 부분 = contextMethod()

- `deleteAll()`의 컨텍스트
    1. DB 커넥션 가져오기
    2. PreparedStatement를 만들어줄 외부 기능 호출하기
    3. 전달받은 PreparedStatement 실행하기
    4. 예외가 발생하면 이를 다시 메소드 밖으로 던지기
    5. 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기

⇒ 전략 패턴에서 말하는 전략: 
- 두 번째 작업에서 사용하는 “**PreparedStatement를 만들어주는 외부 기능**”

- 이 기능을 인터페이스로 만들고 PreparedStatement 생성 전략을 호출하면 됨.

- StatementStrategy 인터페이스
    - PreparedStatement를 만드는 전략의 인터페이스는 컨텍스트가 만들어둔 Connection을 전달 받아서 PreparedStatement를 만들고, 만들어진 PreparedStatement 오브젝트를 돌려줌.
    
    ```java
    public interface StatementStrategy {
        PreparedStatement makePreparedStatement(Connection c)
            throws SQLException;
    }
    ```
    

- 이 인터페이스를 상속해서 PreparedStatement를 생성하는 클래스
    
    ```java
    public class DeleteAllStatement implements StatementStrategy {
        public PreapredStatement makePreparedStatement(Connection c)
        throws SQLException {
            PreparedStatement ps = c.prepareStatement("delete from users");
    
            return ps;
        }
    }
    ```
    
- 전략 DeleteAllStatement을 사용한 UserDao 메소드
    
    ```java
    public void deleteAll() throws SQLException {
        ...
        try {
            c = dataSource.getConnection();
    
            StatementStrategy strategy = new DeleteAllStatement();
            ps = strategy.makePreparedStatement(c);
    
            ps.executeUpdate();
        } catch (SQLException e) {
          ...
        }
    }
    ```
    

> 🤔 컨텍스트 안에서 전략 클래스를 사용하도록 고정되어 있는 점이 이상하다.
> 
> - 전략 패턴은 필요에 따라 컨텍스트는 그대로 유지되면서 전략을 바꿔 쓸 수 있어야 하기 때문

#### DI 적용을 위한 클라이언트/컨텍스트 분리

💡**이를 해결하기 위해 전략 패턴의 실제적인 사용 방법을 더 살펴보자!**

![image (1)](https://github.com/user-attachments/assets/d46e6e07-f64f-4b61-ae9f-6c049947c5c9)


- Context가 어떤 전략을 사용하게 할 것인가는 Client가 결정하는 게 일반적임.
- Client가 구체적인 전략을 선택하고 오브젝트로 만들어서 Context로 전달는 것.
- Context는 전달받은 그 Strategy 구현 클래스의 오브젝트를 사용함.

- 이 패턴 구조에서 중요한 것
    - 컨텍스트에 해당하는 코드(JDBC try/catch/finally 코드)를 클라이언트 코드(StatementStrategy)를 만드는 부분에서 독립시켜야 함.
    - deleteAll() 메소드에서 클라이언트에 들어가야할 코드
    `StatementStrategy strategy = new DeleteAllStatement();`
        - 컨텍스트에 해당하는 부분 → 별도의 메소드로 독립시킴
        
- 메소드로 분리한 try/catch/finally 컨텍스트 코드
    
    ```java
    public void jdbcContextWithStatementStrategy(StatementStrategy stmt)
        throws SQLException {
    
        Connection c = null;
        PreparedStatement ps = null;
    
        try {
            c = dataSource.getConnection();
    
            ps = stmt.makePreparedStatement(c);
    
            ps.executeUpdate();
        } catch (SQLException e) {
            throw e;
        } finally {
            if (ps != null) {
                try { ps.close(); } catch (SQLException e) {}
            }
            if (ps != null) {
                try { c.close(); } catch (SQLExeption e) {}
            }
        }
    }
    ```
    
    - 클라이언트로부터 StatementStrategy 타입의 전략 오브젝트를 제공받음.
    - JDBC try/catch/finally 구조로 만들어진 컨텍스트 내에서 작업을 수행.
    - 제공받은 전략 오브젝트는 `PreparedStatment` 생성이 필요한 시점에 호출해서 사용.

- 클라이언트 책임을 담당할 deleteAll() 메소드
    
    ```java
    public void deleteAll() throws SQLException {
        StatementStrategy st = new DeleteAllStatement();
        jdbcContextWithStatementStrategy(st);
    }
    ```
    
    - 전략 클래스의 오브젝트를 생성하고 컨텍스트를 호출하면 됨.

➡️ 이 구조를 기반으로 UserDao 코드의 **본격적인 개선**을 시작할 수 있다.

<br>

## 3.3 JDBC 전략 패턴의 최적화

- **문제점**
    - `deleteAll()` 메소드 안에 **항상 같은 부분**과 **상황에 따라 바뀌는 부분**이 섞여 있어서 관리가 어려움.
- **해결 방법**
    - **전략 패턴**을 사용해서, 고정된 부분과 바뀌는 부분을 분리.
    - **고정된 부분(JDBC 작업 흐름)**을 `jdbcContextWithStatementStrategy()` 메소드에 담아 공통으로 사용.
- **동작 방법**
    - DAO 메소드들은 **바뀌는 부분**(PreparedStatement 생성)을 준비해서 `jdbcContextWithStatementStrategy()`에 전달.
    - 고정된 흐름과 바뀌는 로직을 분리해 **코드를 깔끔하게** 만들고 **재사용성**을 높임.

<br>

### 3.3.1 전략 클래스의 추가 정보

- add() 메소드에 적용
    - 변하는 부분(PreparedStatement)를 생성하는 코드를 AddStatement 클래스로 옮김.
        
        ```java
        public class AddStatement implements StatementStrategy {
            public PreparedStatement makePreparedStatement(Connection c)
                throws SQLException {
                PreparedStatement ps = c.prepareStatement(
                    "insert into users(id, name, password) values(?,?,?)");
        
                ps.setString(1, user.getId());
                ps.setString(2, user.getName());
                ps.setString(3, user.getPassword());
        
                return ps;
            }
        }
        ```
        
        - 그런데 user는 어디서 가져올까?
    - User 정보를 생성자로부터 제공받도록 만든 AddStatement
        
        ```java
        public class AddStatement implements StatementStrategy {
            User user;
        
            public AddStatement(User user) {
                this.user = user;
            }
        
            public PreparedStatement makePreparedStatement(Connection c)
                throws SQLException {
        
                PreparedStatement ps = c.prepareStatement(
                    "insert into users(id, name, password) values(?,?,?)");
        
                ps.setString(1, user.getId());
                ps.setString(2, user.getName());
                ps.setString(3, user.getPassword());
        
                return ps;
            }
        }
        ```
        
        → user 정보를 생성자를 통해 전달해주도록 add() 메소드(클라이언트) 수정
        
    - user 정보를 AddStatement에 전달해주는 add() 메소드
        
        ```java
        public void add(User user) throws SQLException {
            StatementStrategy st = new AddStatement(user);
            jdbcContextWithStatementStrategy(st);
        }
        ```
        
<br>

### 3.3.2 전략과 클라이언트의 동거

🤔 **현재 구조에서 두 가지 문제점**

1. DAO 메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 함.
2. DAO 메소드에서 StatementStrategy에 전달할 부가적인 정보가 있는 경우, 오브젝트를 전달받는 생성자와 이를 저장할 인스턴스 변수를 번거롭게 만들어야 함.

💡 **어떻게 해결하면 좋을까?**

- 로컬 클래스
- 익명 내부 클래스

#### 로컬 클래스

‼️ 전략 클래스를 매번 독립된 파일로 만들지 말고 **내부 클래스로 정의**하자!

- 장점
    - 클래스 파일이 하나 줄어든다.
    - 메소드 안에서 PreparedStatement 생성 로직을 함께 볼 수 있어 코드에 대한 이해도가 높아진다.
    - 클래스가 선언된 곳의 정보에 접근할 수 있다.
- add() 메소드의 로컬 변수를 직접 사용하도록 수정한 AddStatement
    
    ```java
    public void add(final User user) throws SQLException {
        class AddStatement implements StatementStrategy {
            public PreparedStatement makePreparedStatement(Connection c)
                throws SQLException {
    
                PreparedStatement ps = c.prepareStatement(
                    "insert into users(id, name, password) values(?,?,?)");
    
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
    

#### 익명 내부 클래스

‼️ 좀 더 간결하게 **클래스 이름을 제거**하자! (**익명 내부 클래스**로 만들어보자!)

- 익명 내부 클래스
    - 선언과 동시에 오브젝트 생성
    - 클래스 자신의 타입을 가질 수 없음
    - 구현한 인터페이스 타입의 변수에만 저장 가능
- 메소드 파라미터로 이전한 익명 내부 클래스
    
    ```java
    public void add(final User user) throws SQLException {
    
        jdbcContextWithStatementStrategy(
            new StatementStrategy() {
                public PreparedStatement makePreparedStatement(Connection c)
                    throws SQLException {
    
                    PreparedStatement ps = c.prepareStatement(
                        "insert into users(id, name, password) values(?,?,?)");
    
                    ps.setString(1, user.getId());
                    ps.setString(2, user.getName());
                    ps.setString(3, user.getPassword());
    
                    return ps;
                }
        });
    }
    ```
    
    - DeleteAllStatement도 deleteAll() 메소드로 가져와서 익명 내부 클래스로 처리할 수 있음.
- 익명 내부 클래스를 적용한 deleteAll() 메소드
    
    ```java
    public void deleteAll() throws SQLException {
        jdbcContextWithStatementStrategy(
            new StatementStrategy() {
                public PreparedStatement makePreparedStatement(Connection c)
                    throws SQLException {
                    return c.prepareStatement("delete from users");
                }
        });
    }
    ```

<br>

## 3.4 컨텍스트와 DI

### 3.4.1 JdbcContext의 분리

- 전략 패턴의 구조
    - 클라이언트: UserDao의 메소드
    - 개별 전략: 익명 내부 클래스로 만들어지는 것
    - 컨텍스트: jdbcContextWithStatementStrategy() 메소드

- `jdbcContextWithStatementStrategy()`는 UserDao외에도 다른 DAO에서도 사용가능해야 함.
    
    ➡️  jdbcContextWithStatementStrategy()를 UserDao 클래스 밖으로 독립시켜서 모든 DAO가 사용할 수 있게 해보자!
    

#### 클래스 분리

- 분리해서 만들 클래스의 이름: **JdbcContext**
    - JDBC 작업 흐름을 분리해서 만든 JdbcContext 클래스
        
        ```java
        public class JdbcContext {
            private DataSource dataSource;
        
            public void setDataSource(DataSource datasource) {
                this.dataSource = dataSource;
            }
        
            public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
                Connection c = null;
                PreparedStatement ps = null;
        
                try {
                    c = this.dataSource.getConnection();
        
                    ps = stmt.makePreparedStatement(c);
                    ps.executeUpdate();
                } catch (SQLException e) {
                    throw e;
                } finally {
                    if (ps != null) { try { ps.close(); } catch (SQLException e){} }
                    if (c != null) { try { c.close(); } catch (SQLException e){} }
                }
            }
        }
        ```
        
    
    - JdbcContext를 DI 받아서 사용하도록 만든 UserDao
        
        ```java
        public class UserDao {
            ...
            private JdbcContext jdbcContext;
        
            public void setJdbcContext(JdbcContext jdbcContext) {
                this.jdbcContext = jdbcContext;
            }
        
            public void add(final User user) throws SQLException {
                this.jdbcContext.workWithStatementStrategy(
                    new StatementStrategy() {...}
                );
            }
        
            public void deleteAll(final User user) throws SQLException {
                this.jdbcContext.workWithStatementStrategy(
                    new StatementStrategy() {...}
                );
            }
        }
        ```
        

#### 빈 의존관계 변경

‼️ **새롭게 작성된 오브젝트 간의 의존관계를 살펴보고 스프링 설정에 적용해보자!**

- 스프링의 DI → 인터페이스를 사이에 두고 의존 클래스를 바꿔서 사용하게끔 하는 것이 목적
    - **UserDao → JdbcContext**에 의존
    - **JdbcContext**는 인터페이스가 아닌 구체 클래스
    
    ⇒ **JdbcContext**는 자체적으로 **JDBC 작업을 제공하는 서비스 오브젝트**로서 의미가 있을 뿐이고 **구현이 바뀔 가능성**이 없기 때문에 **인터페이스 없이** DI가 적용된 특별한 구조로 설정됨.
    
- JdbcContext가 추가된 의존 관계를 나타내주는 클래스 다이어그램
    
    ![image (2)](https://github.com/user-attachments/assets/609321d1-9e29-4cd2-a299-084a47735967)

    
<br>

### 3.4.2 JdbcContext의 특별한 DI

#### 스프링 빈으로 DI

> 🤔 UserDao는 인터페이스를 거치지 않고 코드에서 바로 JdbcContext라는 구체 클래스를 사용하고 있다.
> 
> - 이렇게 인터페이스를 사용하지 않고 DI를 적용하는 것은 문제가 있진 않을까?

- 스프링 DI의 기본 의도에 맞게 JdbcContext의 메소드를 인터페이스로 뽑아내어 정의해두고, 이를 UserDao에서 사용하게 해야 하지 않을까?
    - 꼭 그럴 필요는 없다.
- 객체의 생성과 관계설정에 대한 제어권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC 개념을 포괄하므로 DI의 기본을 따르고 있다고 볼 수 있다.

🤔 JdbcContext를 UserDao와 DI 구조로 만들어야 할 이유를 생각해보자.

1. JdbcContext가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되기 때문
2. JdbcContext가 DI를 통해 다른 빈에 의존하고 있기 때문

실제로 스프링에서는 클래스를 직접 의존하는 DI가 등장하는 경우도 있음.

- 중요한 것은 인터페이스의 사용 여부!
→ 왜 인터페이스를 사용하지 않았을까?
    
    > 인터페이스가 없다는 것 → UserDao가 JdbcContext가 긴밀한 관계로 강하게 결합되어 있다는 것
    > 
    > - JDBC 방식이 아닌 JPA나 하이버네이트 같은 ORM을 사용해야 한다면?→ **JdbcContext도 통째로 바뀌어야 함.**
    > - JdbcContext는 테스트에서도 다른 구현으로 대체해서 사용할 이유가 없음.
    >     - 이런 경우에는 굳이 인터페이스를 두지 말고 강력한 결합을 가진 관계를 허용하고, 스프링 빈으로 등록해서 UserDao에 DI 되도록 해도 됨.

‼️ 단, 이런 클래스를 바로 사용하는 코드 구성을 DI에 적용하느 것은 **가장 마지막 단계에서 고려**해보아야 한다.

#### 코드를 이용하는 수동 DI

✨ JdbcContext를 UserDao에 DI 하는 대신 UserDao 내부에서 직접 DI를 적용하는 방법도 존재

- 대신 JdbcContext를 싱글톤으로 만들려는 것은 포기해야 한다.
    - DAO 하나마다 하나씩의 JdbcContext 오브젝트를 가지고 있어야 한다.
- DAO 개수만큼 JdbcContext 오브젝트가 만들어지겠지만, 그정도는 메모리에 주는 부담이 거의 없음.
    - JdbcContext에는 내부에 두는 상태 정보가 거의 없고
    - 자주 만들어졌다가 제거되는 것도 아니기에 GC에 대한 부담도 없음.

😮 JdbcContext를 빈으로 등록하지 않았기에, JdbcContext의 생성과 초기화의 제어권은 UserDao가 갖는다.

남은 문제는 JdbcContext가 다른 빈(DataSource)을 인터페이스를 통해 간접적으로 의존하고 있다는 점이다.

- 다른 빈을 의존하고 있다면 자신도 빈으로 등록돼어 있어야 한다. 이런 경우에는 어떻게 해야 할까?
    - 이 경우에는 UserDao에 JdbcContext의 DataSoucre DI까지 맡기면 된다.
        - UserDao가 임시로 DI 컨테이너처럼 동작하게 만드는 것!
        
        → JdbcContext에 주입해줄 의존 오브젝트인 DataSource는 UserDao가 대신 DI 받도록 하면 됨.
        
        - UserDao는 직접 DataSource 빈을 필요로 하지 않지만, JdbcContext에 대한 DI 작업에 사용할 용도로 제공받는 것임.
        
        ![image (3)](https://github.com/user-attachments/assets/5429aaea-312c-4434-95c5-be2bfb2d43e2)

        

- 스프링 설정파일에 userDao와 dataSource 두 개만 빈으로 정의.
- userDao 빈에 DataSource 타입 프로퍼티를 지정하여 dataSource 빈을 주입받음.
- UserDao는 JdbcContext 오브젝트를 만들면서 DI 받은 DataSource 오브젝트를 JdbcContext의 수정자 메소드로 주입.
- 만들어진  JdbcContext의 오브젝트는 UserDao의 인스턴스 변수에 저장해두고 사용
    
    ```java
    public class UserDao {
        ...
        private JdbcContext jdbcContext;
    
        public void setDataSource(DataSource dataSource) {
            // JdbcContext 생성(IoC)
            this.jdbcContext = new JdbcContext();
    
            // 의존 오브젝트 주입(DI)
            this.jdbcContext.setDataSource(dataSource);
            this.dataSource = dataSource;
        }
    }
    ```
    
    - `setDataSource()` 메소드 → DI 컨테이너가 DataSource 오브젝트를 주입해줄 때 호출됨.
        - 이때, JdbcContext에 대한 수동 DI 작업을 진행
    - 장점
        - 굳이 인터페이스를 두지 않아도 될 만큼 긴밀한 관계를 갖는 Dao 클래스와 JdbcContext를 어색하게 따로 빈으로 만들지 않고 내부에서 직접 만들어 사용하면서도 다른 오브젝트에 대한 DI를 적용할 수 있다는

✨ 지금까지 인터페이스를 사용하지 않고 DAO와 밀접한 관계를 갖는 클래스를 DI에 적용하는 방법 두 가지를 알아봤다.

- 상황에 따라 적절하다고 판단되는 방법을 선택해서 사용하자!
- 분명한 이유와 근거를 설명할 수 없다면 차라리 인터페이스를 만들어서 평범한 DI 구조로 만드는 게 나을 수도 있다.

<br>

## 3.5 템플릿과 콜백

✨ 템플릿/콜백 패턴

- 전략 패턴의 기본 구조에 익명 내부 클래스를 활용한 방식

📌 템플릿 - 전략 패턴의 컨텍스트

📌 콜백 - 익명 내부 클래스로 만들어지는 오브젝트

<br>

### 3.5.1 템플릿/콜백의 동작원리

- 템플릿
    - 고정된 작업 흐름을 가진 코드를 재사용한다는 의미에서 붙인 이름
- 콜백
    - 템플릿 안에서 호출되는 것을 목적으로 만들어진 오브젝트

#### 템플릿/콜백의 특징

- 전략 패턴의 전략과 달리 템플릿/콜백 패턴의 콜백은 보통 단일 메소드 인터페이스를 사용.
    - 템플릿의 작업 흐름 중 특정 기능을 위해 한 번 호출되는 경우가 일반적이기 때문
- 콜백은 일반적으로 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다!

- 콜백 인터페이스의 메소드에는 보통 파라미터가 존재
    - 템플릿의 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용
    - JdbcContext에서는 템플릿인 workWithStatementStrategy() 메소드 내에서 생성한 Connection 오브젝트를 콜백의 메소드인 makePreparedStatement()를 실행할 때 파라미터로 넘겨준다!

- 템플릿/콜백 패턴의 일반적인 작업 흐름도
    
    ![image (4)](https://github.com/user-attachments/assets/8c61f947-1802-41a0-b592-34f0b0486d17)

    
    - **클라이언트의 역할**: 클라이언트는 콜백 객체를 만들고, 콜백이 참조할 정보를 제공한다. 콜백은 클라이언트가 템플릿 메소드를 호출할 때 파라미터로 전달된다.
    - **템플릿의 역할**: 템플릿은 작업 흐름을 진행하며, 필요한 시점에 콜백 메소드를 호출해 작업을 수행시킨다. 콜백은 템플릿과 클라이언트 정보를 사용해 작업을 처리하고 결과를 돌려준다.
    - **작업 완료**: 템플릿은 콜백에서 받은 결과를 활용해 작업을 마무리하고, 필요에 따라 결과를 클라이언트에게 반환한다.

📌 조금 복잡해 보이지만 DI 방식의 전략 패턴 구조라고 생각하고 보면 간단하다.

- 템플릿/콜백 패턴은 **메소드 레벨의 DI(의존성 주입)** 방식이다.

- 일반 DI와 달리, 템플릿/콜백 방식에서는 **매번 메소드 단위로 새로운 객체를 전달**받아 사용한다.
    - **콜백 객체는 클라이언트 메소드의 정보를 직접 참조**할 수 있고, 클라이언트와 콜백이 강하게 결합된다는 점에서 일반적인 DI와 다르다.

- 템플릿/콜백 패턴은 **전략 패턴과 DI의 장점을 익명 내부 클래스**와 결합한 독특한 패턴이다.
    - 이 패턴을 이해하기 위해서는 **전략 패턴과 수동 DI** 개념을 이해할 수 있어야 함.

#### JdbcContext에 적용된 템플릿/콜백

🤔 UserDao, JdbcContext와 StatementStrategy의 코드에 적용된 템플릿/콜백 패턴을 한 번 살폅자!

- 템플릿과 클라이언트가 메소드 단위인 것이 특징.
- JdbcContext의 workWithStatementStrategy() 템플릿은 리턴 값이 없는 단순한 구조
- 템플릿의 작업 흐름이 더 복잡한 경우 → 한 번 이상 콜백 호출 or 여러 개의 콜백을 클라이언트로부터 받아서 사용

<br>

### 3.5.2 편리한 콜백의 재활용

🤔 DAO 메소드에서 매번 익명 내부 클래스를 사용하기 때문에 코드를 작성하고 읽기가 조금 불편하다!

#### 콜백의 분리와 재활용

‼️ 복잡한 익명 내부 클래스의 사용을 최소화할 수 있는 방법을 찾아보자!

- 분리를 통해 재사용이 가능한 코드를 찾아보자.

- 익명 내부 클래스를 사용한 클라이언트(deleteAll() 메소드)
    
    ```java
    public void deleteAll() throws SQLException {
        this.jdbcContext.workWithStatementStrategy(
            new StatementStrategy() {
                public PreparedStatement makePreparedStatement(Connection c)
                        throws SQLException {
                    // "delete from users" 라는 문자열만 바뀐다!
                    return c.prepareStatement("delete from users");
                }
            }
        );
    }
    ```
    
    - 변할 수 있는 부분
        - "delete from users" 라는 문자열
        
        ➡️ SQL 문장만 파라미터로 받아서 바꿀 수 있게 하고 메소드 내용 전체를 분리해서 별도의 메소드로 만들어보자!
        

- 변하지 않는 부분을 분리시킨 deleteAll() 메소드
    
    ```java
    public void deleteAll() throws SQLException {
    		// 변하는 SQL 문장
        executeSQL("delete from users");
    }
    
    // 분리
    public void executeSQL(final String query) throws SQLException {
        this.jdbcContext.workWithStatementStrategy(
            new StatementStrategy() {
                public PreparedStatement makePreparedStatement(Connection c)
                        throws SQLException {
                    // 바뀌는 문자열을 query 파라미터로 받는다!
                    return c.prepareStatement(query);
                }
            }
        );
    }
    ```
    
    - 바뀌지 않는 모든 부분 → executeSql() 메소드로 만듦.

#### 콜백과 템플릿의 결합

😮 executeSQL() 메소드는 UserDao만 사용하기에는 아깝다.

- 재사용 가능한 콜백을 담고 있는 메소드를 DAO가 공유하는 템플릿 클래스로 옮겨보자!

- JdbcContext로 옮긴 executeSql() 메소드
    
    ```java
    public class JdbcContext {
        ...
    
        public void executeSQL(final String query) throws SQLException {
            workWithStatementStrategy(
                new StatementStrategy() {
                    public PreparedStatement makePreparedStatement(Connection c)
                            throws SQLException {
                        return c.prepareStatement(query);
                    }
                }
            );
        }
    }
    ```
    
    - UserDao 메소드에서도 jdbcContext를 통해 executeSql() 메소드를 호출하도록 수정해야 함.
    - 이렇게 코드를 변경하면 JdbcContext 안에 클라이언트와 템플릿, 콜백이 모두 공존하며 동작하는 구조가 된다.

<br>

### 3.5.3 템플릿/콜백의 응용

✨ 템플릿/콜백 패턴은 스프링에서만 사용할 수 있는 기술은 아니지만 특히 스프링이 이 패턴을 적극적으로 활용한다.

- 템플릿/콜백 클래스 & API
    
    : 다양한 자바 엔터프라이즈 기술에서 사용할 수 있도록 미리 만들어져 제공
    

따라서 스프링을 사용하는 개발자라면 템플릿/콜백 기능을 잘 사용하고 직접 만들수도 있어야 함.

고정된 작업 흐름을 가지고 있으면서 자주 반복되는 코드가 있다면 분리할 방법을 생각해보는 습관을 기르자.

1. 중복된 코드는 먼저 메소드로 분리한다.
2. 일부 작업을 바꾸어 사용해야 한다면 인터페이스를 사이에 두고 분리해서 전략 패턴을 적용하고 DI로 의존 관계를 관리한다.
3. 바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 템플릿/콜백 패턴 적용을 고려한다.’
    - 가장 전형적인 템플릿/콜백 패턴의 후보: try/catch/finally 블록을 사용하는 코드

#### 테스트와 try/catch/finally

🤔 파일을 하나 열어서 모든 라인의 숫자를 더한 합을 돌려주는 코드를 만들어보자!

1. numbers.txt 파일을 준비
2. 파일 경로를 주면 10을 돌려주도록 하는 테스트 코드 작성
    
    ```java
    package springbook.learningtest.template;
    ...
    
    public class CalcSumTest {
    	
    	@Test
    	public void sumOfNumbers() throws IOException {
    		
    		Calculator calculator = new Calculator();
    		int sum = calculator.calcSum(getClass().getResource("numbers.txt").getPath());
    		
    		assertThat(sum, is(10));
    	}
    
    }
    ```
    
3. 클래스 생성
    
    ```java
    package springbook.learningtest.template;
    ...
    
    public class Calculator {
    	
    	public Integer calcSum(String filepath) throws IOException {
    		
    		BufferedReader br = new BufferedReader(new FileReader(filepath));
    		Integer sum = 0;
    		String line = null;
    		
    		while((line = br.readLine()) != null) { //마지막 라인까지 한줄씩 읽어가면서 숫자를 더한다.
    			sum += Integer.valueOf(line);
    			
    		}
    		
    		br.close();
    		return sum;
    	}
    }
    ```
    
4. 예외처리
    - 파일이 열렸으면 반드시 닫아주도록
    - 로그
    
    ```java
    package springbook.learningtest.template;
    ...
    
    public class Calculator {
    	
    	public Integer calcSum(String filepath) throws IOException {
    		BufferedReader br = null;
    		
    		try {
    			br = new BufferedReader(new FileReader(filepath));
    			Integer sum = 0;
    			String line = null;
    			
    			while((line = br.readLine()) != null) {
    				sum += Integer.valueOf(line);	
    			}	
    			return sum;
    		} 
    		catch (Exception e) {
    			System.out.println(e.getMessage());
    			throw e;
    			
    		} 
    		finally {
    			if(br != null) {
    				try {br.close();} 
    				catch (IOException e) {System.out.println(e.getMessage());}
    			}
    		}
    	}
    }
    ```
    

#### 중복의 제거와 템플릿/콜백 설계

😱 파일에 있는 모든 숫자의 곱을 계산하는 기능을 추가해야 하고 여러 기능이 계속 추가될 것이라는 소식이 들려왔다. 어떻게 할 것인가?

⇒ 템플릿/콜백 패턴을 적용해보자!

- 템플릿에 담을 반복되는 작업 흐름은 어떤 것인가?
- 템플릿이 콜백에게 전달해줄 내부의 정보는 무엇인가?
- 콜백이 템플릿에게 돌려줄 내용은 무엇인가?
- 템플릿이 작업을 마친 뒤 클라이언트에게 전달해 줘야할 내용은 무엇인가?

- 가장 쉽게 생각해볼 수 있는 구조
    - BufferedReader를 만들어서 콜백에게 전달해주고, 콜백이 각 라인을 읽어서 처리한 후 결과만 템플릿에게 돌려주는 것
        
        ```java
        public interface BufferedReaderCallBack {
            Integer doSomethingWithReader(BufferedReader br) throws IOException;
        }
        ```
        

- 템플릿 부분을 메소드로 분리
    - BufferedReaderCallback 인터페이스 타입의 콜백 오브젝트를 받아서 적절한 시점에 실행
    - 콜백이 돌려준 결과는 처리 후에 다시 클라이언트에게 전달
        
        ```java
        public Integer fileReadTemplate(String filepath, BufferedReaderCallBack callback)
          throws IOException {
            BufferedReader br = null;
            try {
                br = new BufferedReader(new FileReader(filepath));
                int ret = callback.doSomethingWithReader(br);
                return ret;
            } catch (IOException e) {
                System.out.println(e.getMessage());
                throw e;
            } finally {
                if (br != null) {
                    try { br.close(); }
                    catch (IOException e) {
                        System.out.println(e.getMessage());
                    }
                }
            }
          }
        ```
        

- fileReadTemplate()을 사용하도록 calcSum() 메소드를 수정
    - 분리한 부분 외에 코드를 BufferedReaderCallback 인터페이스로 만든 익명 내부 클래스에 담는다.
    - 처리할 파일의 경로와 함께 익명 내부 클래스의 오브젝트를 템플릿에 전달
    - 템플릿이 리턴하는 값을 최종 결과로 사용
        
        ```java
        public Integer calcSum(String filepath) throws IOException {
            BufferedReaderCallBack sumCallback =
                new BufferedReaderCallBack() {
                    public Integer doSomethingWithReader(br) throws IOException {
                        Integer sum = 0;
                        String line = null;
                        while ((line = br.readLine()) != null) {
                            sum += Integer.valueOf(line);
                        }
                        return sum;
                    }
                };
            return fileReadTemplate(filepath, sumCallback);
        }
        ```
        

😮 곱을 구하는 메소드도 이 템플릿/콜백 메소드를 이용해 만들어보자!

1. 테스트코드 작성
    
    ```java
    public class CalcSumTest {
    	Calculator calculator;
    	String numFilepath;
    		
    	@Before
    	public void setUp() {
    			this.calculator = new Calculator();
    			this.numFilepath = getClass().getResource("numbers.txt").getPath());
    	}
    	
    	@Test
    	public void sumOfNumbers() throws IOException {
    		assertThat(calculator.calcSum(this.numFilepath), is(10));
    	}
    	
    	@Test
    	public void multiplyOfNumbers() throws IOException {
    		assertThat(calculator.calcMultiply(this.numFilepath), is(24));
    	}
    
    }
    ```
    
2. 테스트를 성공시키는 코드 작성
    
    ```java
    public Integer calcMultiply(String filePath) throws IOException {
    	
    	BufferedReaderCallback multiplyCallback = new BufferedReaderCallback() {
    		
    		public Integer doSomethingWithReader(BufferedReader br) throws IOException {
    			Integer multiply = 1;
    			String line = null;
    			while((line = br.readLine()) != null) {
    				multiply *= Integer.valueOf(line);
    			}
    			return multiply;
    		}
    	};
    	return fileReadTemplate(filePath, multiplyCallback);
    }
    ```
    

#### 템플릿/콜백의 재설계

🤔 calcSum()과 calcMultiply()에 나오는 두 개의 콜백을 비교하며 공통적인 패턴이 발견되는지 살펴보자.

- calcSum() 메소드
    
    ```java
    Integer sum = 0;
    String line = null;
    while((line = br.readLine()) != null) {
        sum += Integer.valueOf(line);
    }
    return sum
    ```
    
- calcMultiply() 메소드
    
    ```java
    Integer multifly = 1;
    String line = null;
    while((line = br.readLine()) != null) {
        multifly *= Integer.valueOf(line);
    }
    return sum
    ```
    
    ⇒ 두 개의 코드가 아주 유사하다!
    
- 템플릿과 콜백을 찾아낼 때는, 변하는 코드의 경계를 찾고 경계를 기준으로 주고 받는 정보가 있는지 확인하면 된다.
    - 위 메소드에서 바뀌는 코드: 네 번째 줄
        - **`sum += Integer.valueOf(line);`**
        - **`multifly *= Integer.valueOf(line);`**
    - 네 번째 라인으로 전달되는 정보
        - **multifly** or **sum**임.
    - 해당 라인을 처리하고 다시 외부로 전달되는 것
        - multifly or sum과 각 라인의 숫자 값을 가지고 계산한 결과
- 이를 콜백 인터페이스로 정의해보면…
    
    ```java
    public interface LineCallback {
        Integer doSomethingWithLine(String line, Integer value);
    }
    ```
    
- LineCallback 인터페이스를 경계로 해서 만든 새로운 템플릿
    
    ```java
    public Integer lineReadTemplate(String filepath, LineCallback callback, int initVal) throws IOException {
        BufferedReader br = null;
    
        try {
            br = new BufferedReader(new FileReader(filepath));
            Integer res = initVal;
            String line = null;
            while((line = br.readLine()) != null) {
                res = callback.doSomethingWithLine(line, res);
            }
            return res;
        } catch (IOException e) {
            ...
        } finally {
            ...
        }
    }
    ```
    
- 수정한 템플릿을 사용하는 코드
    
    ```java
    public Integer calcSum(String filepath) throws IOException {
        LineCallback sumCallback =
            new LineCallBack() {
                public Integer doSomethingWithLine(String line, Integer value) {
                    return value + Integer.valueOf(line);
                }
            };
        return lineReadTemplate(filepath, sumCallback, 0);
    }
    
    public Integer calcMultiply(String filepath) throws IOException {
        LineCallback multiplyCallback =
            new LineCallBack() {
                public Integer doSomethingWithLine(String line, Integer value) {
                    return value * Integer.valueOf(line);
                }
            };
        return lineReadTemplate(filepath, sumCallback, 0);
    }
    ```
    

#### 제네릭스를 이용한 콜백 인터페이스

😮 만약 파일을 라인 단위로 처리해서 만드는 결과의 타입을 다양하게 가져가고 싶다면, 제네릭스를 이용하면 된다.

- 제네릭스(Generics)란?
    
    : 자바 언어에 타입 파라미터라는 개념을 도입한 것
    
- 제네릭스를 이용하면…
    - 다양한 오브젝트 타입을 지원하는 인터페이스나 메소드 정의 가능

- 파일의 각 라인에 있는 문자를 모두 연결해서 하나의 스트링으로 반환하는 기능을 만들어보자.
    - 템플릿이 리턴하는 타입 = 스트링
    - 콜백의 작업 결과 = 스트링

⇒ 기존에 만들었던 Integer 타입의 결과만 다루는 콜백과 템플릿을 스트링 타입의 값도 처리할 수 있도록 확장해보자!

1. 콜백 인터페이스 수정
    - 콜백 메소드의 리턴 값 & 파라미터 값의 타입을 제네릭 타입 파라미터 T로 선언
        
        ```java
        public interface LineCallback<T> {
            T doSomethingWithLine(String line, T value);
        }
        ```
        

1. 템플릿 수정
    - 템플릿 메소드도 제네릭 메소드로 변경
    - 콜백의 타입 파라미터 = initVal의 타입 = 템플릿 결과 값의 타입
        
        ```java
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
            } catch (IOException e) {
                ...
            } finally {
                ...
            }
        }
        ```
        

⇒ 콜백과 템플릿은 파일의 라인을 처리해서 T 타입의 결과를 만들어내는 **범용적인 템플릿과 콜백**이 됨

1. 파일의 모든 라인의 내용을 하나의 문자열로 길게 연결하는 기능을 가진 메소드 추가
    
    ```java
    public String concatenate(String filepath) throws IOException {
        LineCallback<String> concatenameCallback =
            new LineCallback<String>() {
                public String doSomethingWithLine(String line, String value) {
                    return value + line;
                }
            };
    
        return lineReadTemplate(filepath, concatenateCallback, "");
    }
    ```
    
    ⇒ lineReadTemplate() 메소드의 결과도 스트링 타입이 되어 concatenate() 메소드의 리턴 타입도 스트링으로 정의할 수 있게 됨.
    

✨ 이렇게 **범용적으로 만들어진 템플릿/콜백을 이용**하면 **파일을 라인 단위로 처리하는 다양한 기능을 편리하게 구현**할 수 있다.

## 3.6 스프링의 JdbcTemplate

✨ 스프링이 제공하는 템플릿/콜백 기술을 살펴보자.

- JdbcTemplate
    - 스프링이 제공하는 JDBC 코드용 기본 템플릿

😞 지금까지 만들었던 JdbcContext 대신에 훨씬 강력하고 편리한 JdbcTemplate으로 코드를 변경해보자!

```java
public class UserDao {
    ...
    **private JdbcTemplate jdbcTemplate;**

    public void setDataSource(DataSource dataSource) {
        **this.jdbcTemplate = new JdbcTemplate(dataSource);**

        this.dataSource = dataSource;
    }
}
```

<br>

### 3.6.1 update()

😮 deleteAll()에 먼저 적용해보자

- 처음 적용했던 콜백
    
    : StatementStrategy 인터페이스의 makePreparedStatment() 메소드
    
- JdbcTemplate의 콜백
    
    : PreparedStatementCreator 인터페이스의 createPreparedStatment() 메소드
    
- JdbcTemplate을 적용한 deleteAll() 메소드
    
    ```java
    public void deleteAll() {
        this.jdbcTemplate.update(
            new PreparedStatementCreator() {
                public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
                    return con.prepareStatement("delete from users");
                }
            }
        )
    }
    ```
    

- 앞에서 만들었던 execureSql()
    - SQL 문장만 전달하면 미리 준비된 콜백을 만들어서 템플릿을 호출하는 것까지 한 번에 해주는 편리한 메소드
    - JdbcTemplate 에서도 기능이 비슷한 메소드가 존재
- 내장 콜백을 사용하는 update()로 변경한 deleteAll() 메소드
    
    ```java
    public void deleteAll() {
        this.jdbcTemplate.update("delete from users");
    }
    ```
    

- JdbcTemplate은 앞에서 구상만 해보고 만들지는 못했던 add() 메소드에 대한 편리한 메소드도 제공.
    - 치환자를 가진 SQL로 PreparedStatement를 만들고 함께 제공하는 파라미터를 순서대로 바인딩해주는 기능을 가진 update() 메소드를 사용 가능
    - SQL과 함께 가변인자로 선언된 파라미터를 제공해주면 됨.
- add() 메소드의 콜백 내부
    
    ```java
    PreparedStatement ps =
        c.prepareStatement("insert into users(id, name, password) values (?,?,?)");
    
    ps.setString(1, user.getId());
    ps.setString(2, user.getName());
    ps.setString(3, user.getPassword());
    ```
    
- 이를 JdbcTemplate에서 제공하는 메소드로 변환
    
    ```java
    this.jdbcTemplate.update("insert into users(id, name, password) values (?,?,?)"),
    	user.getId(), user.getName(), user.getPassword());
    ```
    

⇒ JdbcContext를 이용하던 UserDao 메소드를 모두 JdbcTemplate으로 변경했다.

<br>

### 3.6.2 queryForInt()

😮 아직 템플릿/콜백 방식을 적용하지 않았던 메소드에 JdbcTemplate을 적용해보자!

- getCount()
    - SQL 쿼리를 실행하고 ResultSet을 통해 결과 값을 가져오는 코드
    - 사용할 수 있는 템플릿
        - PreparedStatementCreator 콜백
        - ResultSetExtractor 콜백을 파라미터로 받는 query() 메소드
    - ResultSetExtractor?
        - PreparedStatement의 쿼리를 실행해서 얻은 ResultSet을 전달받는 콜백
    - ResultSetExtractor 콜백
        - 템플릿이 제공하는 ResultSet을 이용해 원하는 값을 추출해서 템플릿에 전달하면, 템플릿은 나머지 작업을 수행한 뒤에 그 값을 query() 메소드의 리턴 값으로 돌려줌.
- JdbcTemplate을 사용하도록 수정한 getCount() 메소드
    
    ```java
    public int getCount() {
        return this.jdbcTemplate.query(
            // 첫 번째 콜백: Statement 생성
            new PreparedStatementCreator(){
                public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
                    return con.prepareStatement("select count (*) from users");
                }
            },
            // 두 번째 콜백, ResultSet으로부터 값 추출
            new ResultSetExtractor<Integer>() {
                public Integer extractData(ResultSet rs) throws     SQLException, DataAccessException {
                    rs.next();
                    return rs.getInt(1);
                }
            }
        );
    }
    ```
    
    - 두 번째 콜백에서 리턴하는 값은 결국 템플릿 메소드의 결과로 다시 리턴된다.
    - ResultSetExtractor는 제네릭스 타입 파라미터를 갖는다.

- 두 번째 콜백을 재사용하는 방법
    - 해당 기능을 가진 콜백을 내장하고 있는 queryForInt()라는 메소드를 사용
    
    ```java
    public int getCount() {
        return this.jdbcTemplate.queryForInt("select count(*) from users");
    }
    ```
    

<br>

### 3.6.3 queryForObject()

🤔 get() 메소드에 JdbcTemplate을 적용해보자!

1️⃣ SQL은 바인딩이 필요한 치환자를 갖고 있음.

- add()에서 사용했던 방법을 적용

2️⃣ ResultSet에서 복잡한 User 오브젝트로 만들어야 함.

- ResultSet의 결과를 User 오브젝트로 만들어 프로퍼티에 넣어줘야 함.
- 이를 위해, RowMapper 콜백 사용
    - ResultSet의 로우 하나를 매핑하기 위해 사용되므로 여러 번 호출 가능함.

- queryForObject()와 RowMapper를 적용한 get() 메소드
    
    ```java
    public User get(String id) {
        return this.jdbcTemplate.queryForObject("select * from users where id = ?",
    
            // SQL에 바인딩할 파라미터 값. (가변인자 대신 배열 사용)
            new Object[] {id},
    
            // ResultSet한 row의 결과를 Object에 매핑해주는 RowMapper 콜백
            new RowMapper<User> {
                public User mapRow(ResultSet rs, int rowNUum) throws SQLExcpetion {
                    User user = new User();
                    user.setId(rs.getString("id"));
                    user.setName(rs.getString("name"));
                    user.setPassword(rs.getString("password"));
                    return user;
                }
            }
        )
    }
    ```
    
    - 예외상황을 처리하기 위해 특별히 해줄 것은 없다.

<br>

### 3.6.4 query()

#### 기능 정의와 테스트 작성

🤔 RowMapper 에 현재 등록되어 있는 모든 사용자 정보를 가져오는 getAll() 메소드를 추가해보자.

- List<User> 타입으로 반환하고, id순으로 정렬해서 가져오도록 하자.
- 먼저 테스트 메소드부터 작성해보자.

- getAll()에 대한 테스트
    
    ```java
    @Test
    public void getAll()  {
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
    
    // 검증 코드는 테스트에서 반복적으로 사용되기 때문에 분리해놓음.
    private void checkSameUser(User user1, User user2) {
        assertThat(user1.getId(), is(user2.getId()));
        assertThat(user1.getName(), is(user2.getName()));
        assertThat(user1.getPassword(), is(user2.getPassword()));
        assertThat(user1.getEmail(), is(user2.getEmail()));
        assertThat(user1.getLevel(), is(user2.getLevel()));
        assertThat(user1.getLogin(), is(user2.getLogin()));
        assertThat(user1.getRecommend(), is(user2.getRecommend()));
    }
    ```
    

#### query() 템플릿을 이용하는 getAll() 구현

😮 이제 이 테스트를 성공시키는 getAll() 메소드를 만들어보자.

- JdbcTemplate의 query() 메소드를 사용한다.
- query()는 여러 개의 row가 결과로 나오는 일반적인 경우에서 사용 가능하다.
- query()의 리턴 타입은 List<T>

- query()를 이용해 만든 getAll() 메소드
    
    ```java
    public List<User> getAll() {
        return this.jdbcTemplate.query("select * from users order by id",
            new RowMapper<User>() {
                public User mapRow(ResultSet rs, int rowNum) throws SQLException {
                    User user = new User();
                    user.setId(rs.getString("id"));
                    user.setName(rs.getString("name"));
                    user.setPassword(rs.getString("password"));
                    return user;
                }
            });
    }
    ```
    
    - **파라미터 구조**
        - 첫 번째 파라미터: 실행할 SQL 쿼리
        - 두 번째 파라미터: 바인딩할 파라미터(생략 가능)
        - 마지막 파라미터: RowMapper 콜백
    1. **SQL 쿼리 실행**
        - ResultSet의 모든 로우를 열람하면서 로우마다 RowMapper 콜백을 호출
        - DB에서 가져오는 로우의 개수만큼 호출
    2. **RowMapper의 역할**
        - RowMapper는 현재 로우의 내용을 User  타입 오브젝트에 매핑해서 돌려줌.
    3. **템플릿 메소드 패턴**:
        - User 오브젝트는 템플릿이 미리 준비한 List<User> 컬렉션에 추가됨.
    4. **반환**
        - 작업을 마치면 모든 로우에 대한 User 오브젝트를 담고 있는 List<USer> 오브젝트가 리턴됨.
    

#### 테스트 보완

🤔 getAll() 역시도 네거티브 테스트를 진행해보자!

- 만약 결과가 하나도 없는 경우에 getAll()을 실행했을 때 어떻게 되는지를 검증해야 한다.
- query() 템플릿과 동일하게 크기가 0인 List<T> 오브젝트를 리턴하도록 만들어보자.

- 데이터가 없는 경우에 대한 검증 코드가 추가된 getAll() 테스트
    
    ```java
    @Test
    public void getAll() {
        dao.deleteAll();
    
        List<User> users0 = dao.getAll();
        assertThat(users0.size(), is(0));
    }
    ```
    

<br>

### 3.6.5 재사용 가능한 콜백의 분리

#### DI를 위한 코드 정리

🤔 필요 없어진 DataSource인스턴스 변수를 제거하자. 

- UserDao의 모든 메소드가 JdbcTemplate을 이용하도록 만들었으니 DataSource를 직접 사용할 일은 없다.
- 정리하면, JdbcTemplate 인스턴스 변수와 DataSource 타입 수정자 메소드만 깔끔하게 남는다.

#### 중복 제거

🤔 중복된 코드가 없나 다시 한 번 살펴보자!

- get()과 getAll()에서 RowMapper의 내용이 똑같다는 사실을 알 수 있다.
- 앞으로도 다양한 조건으로 사용자를 조회하는 검색 기능이 추가될 것 이므로 중복되는 부분은 제거해주는 것이 좋다.
- RowMapper 콜백 오브젝트에는 상태정보가 없으므로 하나의 콜백 오브젝트를 만들어서 공유해도 된다.

- 재사용 가능하도록 독립시킨 RowMapper
    
    ```java
    public class UserDao {
        private RowMapper<User> userMapper = new RowMapper<User>() {
            public User mapRow(ResultSet rs, int rowNum) throws SQLException {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));
                return user;
            }
        }
    }
    ```
    

- 공유 userMapper를 사용하도록 수정한 get(), getAll()
    
    ```java
    public User get(String id){
        return this.jdbcTemplate.queryForObject(
            "select * from users where id = ?",
            new Object[] {id}, this.userMapper);
    }
    
    public List<User> getAll(){
        return this.jdbcTemplate.queryForObject(
            "select * from users order by id",
            this.userMapper);
    }
    ```
    

#### 템플릿/콜백 패턴과 UserDao

- 최종적으로 완성된 UserDao 클래스
    
    ```java
    public class UserDao {
    	public void setDataSource(DataSource dataSource) {
    		this.jdbcTemplate = new JdbcTemplate(dataSource);
    	}
    	
    	private JdbcTemplate jdbcTemplate;
    	
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
    
    	
    	public void add(final User user) {
    		this.jdbcTemplate.update("insert into users(id, name, password) values(?,?,?)",
    						user.getId(), user.getName(), user.getPassword());
    	}
    
    	public User get(String id) {
    		return this.jdbcTemplate.queryForObject("select * from users where id = ?",
    				new Object[] {id}, this.userMapper);
    	} 
    
    	public void deleteAll() {
    		this.jdbcTemplate.update("delete from users");
    	}
    
    	public int getCount() {
    		return this.jdbcTemplate.queryForInt("select count(*) from users");
    	}
    
    	public List<User> getAll() {
    		return this.jdbcTemplate.query("select * from users order by id",this.userMapper);
    	}
    
    }
    ```
    
    - UserDao에는 User정보를 DB에 넣거나 가져오거나 조작하는 방법에 대한 핵심적인 로직만 담게 되었다.
    - User라는 자바 오브젝트와 USER 테이블 사이에 어떻게 정보를 주고 받을지, DB와 커뮤니케이션하기 위한 SQL 문장이 어떤 것인지에 대한 최적화된 코드를 갖고 있다. (=응집도가 높다)
    - 반면, JDBC API를 사용하는 방식, 예외처리, 리소스 반납, DB 연결에 관한 책임과 관심은 모두 JdbcTemplate에 있다.

🤔 UserDao를 여기서 더 개선할 수 있을까?

1. userMapper가 인스턴스 변수로 설정되어 있고, 한 번 만들어지면 변경되지 않는 프로퍼티와 같은 성격을 띠고 있으니 아예 UserDao 빈의 DI용 프로퍼티로 만드는 방법
    - User의 프로퍼티와 User 테이블의 필드 이름이 바뀌거나 매핑 방식이 바뀌는 경우에 UserDao 코드를 수정하지 않고도 매핑정보 변경이 가능해진다.
2. DAO 메소드에서 사용하는 SQL 문장을 외부 리소스에 담고 이를 읽어와 사용하게 하는 방법
    - DB 테이블의 이름이나 필드 이름을 변경하거나 SQL 쿼리를 최적화해야 할 때도 UserDao 코드에는 손을 댈 필요가 없다.

