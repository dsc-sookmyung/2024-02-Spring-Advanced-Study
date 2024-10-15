
</br>

객체지향 설계에서는 OCP가 있었다.
개방 폐쇄의 원칙.
어떠한 기능,모듈은 변하지 않는 성질이고 다른 기능,모듈은 확장되는 성질이 있다. 이것을 지키고자 Spring의 Template 개념이 생겨났고 해당 개념은 GoF 디자인 패턴 중 템플릿 메서드에서 유래하였다. 


(GoF 잠시 보고 가시지요..)

![](https://velog.velcdn.com/images/ykky2115/post/3e30ec2b-3558-44c2-9c15-0c7d8f819455/image.png)



</br>

# 템플릿의 정의

스프링의 템플릿도 반복적인 작업이나 공통적인 작업을 추상화하고, 세부적인 구현은 하위 클래스나 메서드를 통해 처리하는 방식이다. 

3장에서는 스프링에 적용된 템플릿 기법을 살펴볼 것이다. 


_여기서 잠깐_

템플릿 들어간 거면 다 템플릿 기법이라는거냐

답은 
</br>
</br>
</br>
</br></br>
</br>
</br>


![](https://velog.velcdn.com/images/ykky2115/post/f307ed4e-2491-4aa7-8d73-db02c800c09b/image.png)


그렇다.

**!**

여기서 잠시 템플릿 기법과 그의 자식들인 뷰 템플릿 및 대표적인 템플릿 클래스를 보살피도록 하겠다. 

**템플릿 기법**

코드의 일부를 미리 정의하고, 특정 부분만 필요에 따라 바꾸는 기법. 
디자인 패턴에서 흔히 활용되는 개념이라 전략 패턴과 팩토리 메서드에도 사용된다.
</br>

**View Template**

구조적인 부분을 미리 정의했다는 것에서 템플릿 기법이라고 본다. 동적 데이터 렌더링을 위해 미리 정의된 HTML, JSON, XML 등의 뼈대를 가지고 있는 파일이다. 필요시 변수를 대체하거나 동적 콘텐츠를 삽입하는 방식이다. 웹 개발에서 데이터와 화면 구조를 분리하여 유연하게 페이지를 생성하게 만든다. ex) JSP, Thymeleaf
</br>

**JdbcTemplate**

스프링에서 JDBC를 통해 데이터베이스와 상호작용할 때 발생하는 반복 코드를 줄인다. 또한 예외 처리, 리소스 관리 등의 공통 작업을 자동 처리한다. 
</br>    

**RestTemplate**

HTTP 기반 RESTful 웹 서비스와 상호작용하기 위해 사용되는 스프링의 템플릿 클래스이다. 외부 서비스와의 통신에서 발생할 수 있는 복잡한 요청/응답 처리 및 예외 처리를 간소화한다.
</br>

**TransactionTemplate**

트랜잭션 처리 로직을 간소화한다. 트랜잭션 경계 설정과 예외 처리를 보다 쉽게 만드는 템플릿 클래스이다. 주로 트랜잭션 매니저와 함께 사용되어 트랜잭션 경계를 명확히 정의할 수 있게 도와준다.
    
    
위 클래스들은 개발자가 복잡한 인프라 작업을 신경 쓰지 않고 비즈니스 로직에 집중할 수 있도록 한다!

참고로, JPA에는 명시적인 템플릿 클래스는 없으나 JPA 자체에 이미 템플릿 기법의 원리를 내포한다. 
과거에는 JpaTemplate가 있었으나 Deprecated 되고 **Spring Data JPA** 라는 모듈을 통해 JPA의 템플릿 기법을 더욱 고도화하여, 인터페이스만으로 데이터 액세스 계층을 구현할 수 있게 함으로써 반복 코드를 아주 굿 줄여준다. 
</br>

![](https://velog.velcdn.com/images/ykky2115/post/31b417fb-a687-4278-a5c4-ce640d79ad6a/image.png)_and never will.._

# JDBC로 보는 템플릿
</br>
우선 try-catch-finally문을 활용하여 예외처리를 하는 DAO 코드를 보겠다.

```
//삭제 DAO 기능
public void deleteAll() throws SQLException {
	Connection conn = null;
    PreparedStatement ps = null;
    
    try {
    	conn = dataSource.getConnection();
        ps = conn.prepareStatement("delete from users");
        ps.executeUpdate();
    } catch(SQLException e) {
    	throw e;
    } finally {
    	if (ps != null) {
        	try {
            	ps.close();
            } catch (SQLException e) {
            }
        }
        if (conn != null) {
        	try {
            	conn.close();
            } catch (SQLException e) {
            }
        }
    }
}

//조회 DAO 기능
public int getCount() throws SQLException {
	Connection conn = null;
    PreparedStatement ps = null;
    ResultSet rs = null;
    
    try {
    	conn = dataSource.getConnection();
        ps = conn.prepareStatement("select count(*) from users");
        
        rs = ps.executeQuery();
        rs.next();
        return rs.getInt(1);
        
    } catch (SQLException e) {
    	throw e;
    } finally {
    	if (rs != null) {
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
        if (conn != null) {
        	try {
            	conn.close();
            } catch (SQLException e) {
            }
        }
    }
}
```

### JDBC try/catch/finally의 문제점

위 두기능은 동작의 문제는 없지만 트/캐/파 블록이 2중으로 <u>중첩</u>까지 나오고 모든 <u>메소드가 반복</u>된다. 
단지 두 가지 기능을 적는 것인데도 트/캐/파를 적으면서 {}(curly braces)가 빠진 부분이 있거나 try를 적고 catch를 빠트린다거나..
코드 서른 줄 적는데도 괄호 빠트린 거 없나 눈이 피로했다.

심지어 DB 관련해서 close()등이 없다면 과부하가 올 것이다. 
메소드가 호출되고 커넥션이 반환되지 않고 쌓여가게 된다면 언젠가 DB 풀에 설정해놓은 최대 DB 커넥션 개수를 넘어설 것이고, 서버에서 리소스가 꽉 찼다는 에러가 나면 서비스가 중단되는 상황이 발생할 수 있다. 

## 분리와 재사용을 위한 디자인 패턴 적용

#### 메소드 추출
deleteAll()을 보면 코드 중 PreparedStatement는 바뀌는 부분(delete -> select,update,insert 등) 으로써 
```
ps = c.prepareStatement("delete from users");
```

이 부분을 독립적인 메소드 추출하는 방식으로 분리하여 코드의 효율을 도모할 수 있겠으나 만약 다음과 같이 바꾼다면,

```
public void deleteAll() {
	ps = makeStatement(c);
}

private PreparedStatement makeStatement(Connection c) throws SQLException {
	PreparedStatement ps;
    ps = c.prepareStatement("delete from users");
    return ps;
}
```

분리한 메소드는 DAO 로직마다 새롭게 만들어서 확장되어야 하는데 delete 자체가 확정되어 있어서 재사용할 수 없다. ->  Fail!

#### 템플릿 메소드 패턴의 적용
템플릿 메소드 패턴은 상속을 통해 기능을 확장한다. 

![](https://velog.velcdn.com/images/ykky2115/post/b36ae94a-fb17-4103-b452-c1db3244e76e/image.png)

abstract한 서브클래스를 템플릿이라고 한다. 그 후 템플릿을 상속한 클래스가 구체적인 기능을 논한다. 
해당 User DAO에서는 다음과 같이 쓸 수 있다. 
![](https://velog.velcdn.com/images/ykky2115/post/6b3b9308-c27f-456a-82b2-57bbfe4b66cb/image.png)

이것을 코드로 구현하면 다음과 같을 것이다. 
```
abstract class UserDao{

	public final String makeStatement() {
    	//공통 로직을 여기서
    	return specificMakeStatement();
   }
   //하위 클래스에서 구현하는 추상 메서드
   protected abstract String specificMakeStatement();
}

class UserDaoAdd extends UserDao {
	@Override
    protected String specificMakeStatement() {
    	return "INSERT INTO users";
    }
}

class UserDaoDeleteAll extends UserDao {
    @Override
    protected String specificMakeStatement() {
        return "DELETE FROM users"; 
    }
}

class UserDaoGet extends UserDao {
    @Override
    protected String specificMakeStatement() {
        return "SELECT * FROM users WHERE id = ?"; 
    }
}

class UserDaoGetCount extends UserDao {
    @Override
    protected String specificMakeStatement() {
        return "SELECT COUNT(*) FROM users"; 
    }
}
```
현재 템플릿 메소드 패턴은 제한이 있다. DAO 로직마다 상속을 통해 새 클래스를 계속 만들어야 한다는 점이다. 또 확장구조가 이미 클래스를 설계하는 시점에서 고정되어 버린다는 점도 문제다. 
이미 클래스 레벨에서 컴파일 시점에 그 관계가 결정되므로 유연성이 떨어진다.
<u>상속을 통해 확장을 꾀하는 템플릿 메소드 패턴의 단점이 고스란히 드러나는 지점이다.</u>

#### 전략 패턴의 적용
OCP을 잘 지키면서 템플릿 메소드 보다 유연하고 확장성이 뛰어난 전략 패턴이다. 확장이 되는 클래스를 추상화된 인터페이스로 위임한다. 
전략 패턴을 따로 정리하였다. 궁금하다면 [해당 링크](https://velog.io/@ykky2115/GoF-%EB%94%94%EC%9E%90%EC%9D%B8-%ED%8C%A8%ED%84%B4-1)


deleteAll()메소드를 예로 변하지 않는 부분은 Context에 해당한다.
deleteAll()의 컨텍스트는 다음으로 정리한다. 

- DB 커넥션 가져오기
- PreparedStatement 만들어줄 외부 기능 호출
- 전달받은 ps 실행하기
- 예외 발생시 메소드 밖으로 던지기
- 모든 경우에 만들어진 PreparedStatement와 Connection 적절히 닫기

여기에서 PreparedStatement 외부 기능 호출이 위(템플릿 메서드 패턴)에서 각 하위클래스 별로 만들었던 부분이다. 
이것이 전략 패턴에서 말하는 전략_Strategy_이다. 
코드로 변환해보겠다!

```
public interface StatementStrategy {
	PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```
이 인터페이스를 상속하는 실제 전략, deleteAll() 메소드의 기능을 위해 만든 전략 클래스이다.
```
public class DeleteAllStatement implements StatementStrategy {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
    	PreparedStatement ps = c.prepareStatement("delete from users");
        return ps;
    }
}
```
이제 contextMethod()에 해당하는 UserDao의 deleteAll() 메소드에서 사용한다.
```
public void deleteAll() throws SQLException {
	try {
    	c = dataSource.getConnection();
        
        StatementStrategy strategy = new DeleteAllStatement();
        ps = strategy.makePreparedStatement(c);
        
        ps.executeUpdate();
    }
}
```
**한계점**
하지만 전략 패턴은 필요에 따라 컨텍스트는 유지하되 전략을 바꿔 써야하는 것인데 위 코드들을 보면 구체적인 전략 클래스DeleteAllStatement로 고정된다. 
정보 은닉에 따라 컨텍스트는 StatementStrategy, DeleteAllStatement 둘다 몰라야 한다..,,.

#### DI 적용을 위한 클라이언트/컨텍스트 분리
아차차 위의 전략 패턴은 컨텍스트와 전략, 구현체만 있을 뿐 클라이언트가 없다. 
클라이언트가 전략을 선택하게 만듦으로 우리는 어떤 구체적인 전략(예를 들면 deleteallstatement())를 쓸 것인지 클라이언트가 정하고 이 때 클라이언트는 여러 구체적인 전략 클래스 중 바꿔가면 쓸 수 있게 된다!

### JDBC 전략 패턴의 최적화

현재 클라이언트를 새로 추가함으로써 다형성이 추가되었지만 DAO 메소드들이 늘어나면 늘어나는대로 전략 구현체 클래스의 수가 증가한다.  
이에 따른 해결책으로** 1. 로컬 클래스화 하기 2. 익명 내부 클래스로 만들기**가 있다. 우선 매번 독립된 파일로 만들지 않고 UserDao 클래스 안에 내부 클래스로 방향을 돌리는 것이다. 
익명 내부 클래스는 new 인터페이스이름() {클래스 본문}; 형태이다. 


## 컨텍스트와 DI
현재 구조는 다음과 같다. 
![](https://velog.velcdn.com/images/ykky2115/post/6f3239f8-76d7-4bbe-92f4-8928a84c59b0/image.png)![](https://velog.velcdn.com/images/ykky2115/post/4fc73c63-394b-41a8-abd1-e6487d137171/image.png)

이번 목차시에는 context인 jdbcContextWithStatementStrategy()를 클래스 밖으로 독립시켜 UserDao 뿐 아닌 모든 DAO가 사용할 수 있도록 해본다.

**1. 클래스 분리**
JdbcContext라는 독립되는 클래스로 만들고 위에 있는 Context Method를 workWithStatementStrategy()라는 메소드로 만든다. 
또한 DataSource에 의존하고 있으므로 Dependency Injection할 수 있게 준비한다.
![](https://velog.velcdn.com/images/ykky2115/post/fbadff1b-6f2d-4fde-8d9e-1ace6bdbdf27/image.png)

UserDao가 분리된 JdbcContext를 DI 받아서 사용하게 한다.
![](https://velog.velcdn.com/images/ykky2115/post/b2fd948b-8afb-4eff-be61-27b62909ef3e/image.png)

**2. 빈 의존관계 변경**
책에는 XML로 되어있었으나 애노테이션으로 변경해보았다. 

UserDao
```
@Repository
public class UserDao {
	private DataSource dataSource;
    private JdbcContext jdbcContext;
    
    @Autowired
    public void setDataSource(DataSource dataSource) {
    	this.dataSource = dataSource;
    }
    
    @Autowired
    public void setJdbcContext(JdbcContext jdbcContext) {
    	this.jdbcContext = jdbcContext;
    }
}
```
JdbcContext
```
@Component
public class JdbcContext {
	private DataSource dataSource;
    
    @Autowired
    public void setDataSource(DataSource dataSource) {
    	this.dataSource = dataSource;
    }
}
```
Configuration
```
@Configuration
@ComponentScan(basePackages = "spring.book.user.dao")
public class AppConfig {
}
```

기존 xml 코드를 참고하였다. 

![](https://velog.velcdn.com/images/ykky2115/post/b9557f4c-9d8e-4697-b8db-e70ec6d18b82/image.png)

## 템플릿과 콜백

위에서 그냥 익명 내부 클래스 쓸 때는 전략 패턴의 템플릿 기법에서 익명 내부 클래스 쓰기로만 알고 있었겠지만 
사실 **템플릿/콜백** 패턴이라고 한다. 

#### 콜백
콜백은 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트를 말한다. 파라미터로 전달되지만 값을 참조하기 위한 것이 아니라 특정 로직을 담은 메소드를 실행시키기 위해 사용한다. 
자바에선 메소드 자체를 파라미터로 전달할 방법은 없기 때문에 메소드가 담긴 오브젝트를 전달해야 한다. 그래서 functional object라고도 한다. 
=> 템플릿 안에서 호출되는 것을 목적으로 만든 오브젝트

### 템플릿/콜백의 특징

- 단일 기능을 처리하기 위해 단일 메소드 인터페이스를 사용한다.
- 콜백 인터페이스의 메소드에는 보통 파라미터가 있다. 

### 템플릿/콜백의 작업 흐름
![](https://velog.velcdn.com/images/ykky2115/post/44104569-1c10-4ced-bba5-02520ad98502/image.png)
콜백은 보통 클라이언트에 의해 제공되거나, 클라이언트가 정의하는 경우가 많다. 
클라이언트는 콜백을 생성하고 템플릿을 자기 일을 하다가 참조 정보를 가지고 콜백 오브젝트의 메소드를 호출한다. 
콜백의 일이 끝나서 템플릿에게 결과를 돌려주면 템플릿은 마저 자기 일을 한다. 

### 콜백의 재활용

### 템플릿/콜백의 응용

### 템플릿/콜백의 재설계

## 스프링의 JdbcTemplate
해당 목차에서 스프링이 제공하는 템플릿/콜백 기술을 살핀다. 

