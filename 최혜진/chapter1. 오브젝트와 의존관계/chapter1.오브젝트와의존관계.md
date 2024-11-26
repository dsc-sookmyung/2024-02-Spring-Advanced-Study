# 1. 초난감 DAO

> **<mark>DAO 란?</mark>**
>> DAO(Data Acceess Object) 는 DB를 사용해 데이터를 조회하거나 조작하는 기능을 전담하도록 만든 오브젝트이다.<br><br>

> **<mark>DAO가 필요한 이유</mark>**
>> 1. **분리된 책임**<br>
>>    비지니스 로직과 데이터 접근 로직을 분리한다.
>> 2. **추상화**<br>
>>    데이터베이스의 세부 구현을 추상화해, 비니지스 로직이 데이터베이스의 종류나 구조에 의존하지 않도록 한다.<br>
>>    이는 <u>데이터베이스를 변경할 때, 비지니스 로직에 미치는 영향을 최소화할 수 있다.</u><br>
>> 3. **일관성 있는 데이터 접근** <br><br>

>**<mark>DAO의 이점</mark>**
>> 1. 코드의 재사용성
>> 2. 유연성
>> 3. 예외 처리
>> 4. 트랜잭션 관리

## User

사용자의 정보를 JDBC API를 통해 DB 에 저장하고 조회할 수 있는 간단한 DAO 를 하나 만들 것이다.
```java
package org.example.chapter1.domain;  
  
public class User {  
  
    private String id;  
    private String name;  
    private String password;  
  
    public String getId() {  
        return id;  
    }  
  
    public void setId(String id) {  
        this.id = id;  
    }  
  
    public String getName() {  
        return name;  
    }  
  
    public void setName(String name) {  
        this.name = name;  
    }  
  
    public String getPassword() {  
        return password;  
    }  
  
    public void setPassword(String password) {  
        this.password = password;  
    }  
}
```
위의 User 오브젝트에 담긴 정보가 실제로 보관될 DB 테이블을 만들 것이다.

MySQL 을 사용해서 DB 생성을 해보자.

```
create table users(
    id varchar(10) primary key,
    name varchar(20) not null,
    password varchar(20) not null
)
```

코드 실행을 하다 보면, id 칼럼을 auto increment 로 할 걸이라는 후회가 밀려온다.<br>
하지만 나중에 저 부분에 대한 언급이 있다고 하니, 우선 이 코드를 유지하자.

> **<mark>자바빈</mark>**
>>  자바빈(JavaBean) 은 원래 비주얼 툴에서 조작 가능한 컴포넌트를 의미한다. 자바빈의 몇 가지 코드 관례는 JSP 빈, EJB와 같은 표준 기술과 자바빈 스타일의 오브젝트를 사용하는 오픈소스 기술을 통해 이어져 왔다. <br> 이제 자바빈이라고 하면 **비주얼 컴포넌트라기보다 디폴트 생성자, 프로퍼티 관례를 따라 만들어진 오브젝트**를 가르킨다.
> 
>>  - `디폴트 생성자`  : 자바빈은 파라미터가 없는 디폴트 생성자를 갖고 있어야 한다. 툴이나 프레임워크에서 리플렉션을 이욯해 오브젝트를 생성하기 때문에 필요하다. 
>> -  `프로퍼티`  : 자바빈이 노출하는 이름을 가진 속성을 프로퍼티라고 한다. 프로퍼티는 set 으로 시작하는 수정자 메소드 (setter) 와 get 으로 시작하는 접근자 메소드(getter)를 이용해 수정 또는 조회할 수 있다.

## UserDao

사용자의 정보를 DB에 넣고 관리할 수 있는 DAO 클래스를 만들어보자.
```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.DriverManager;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import org.example.chapter1.domain.User;  
  
public class UserDao {  
	public void add(User user) throws ClassNotFoundException, SQLException {  
	Class.forName("com.mysql.cj.jdbc.Driver");  
	Connection c = DriverManager.getConnection(  
			"jdbc:mysql://localhost/hyejindb", "hyejin66", "u1234");  
	PreparedStatement ps = c.prepareStatement(  
			"insert into users(id, name, password) values(?,?,?)");  
  
	ps.setString(1, user.getId());  
	ps.setString(2, user.getName());  
	ps.setString(3, user.getPassword());  
  
	ps.executeUpdate();  
  
	ps.close();  
	c.close();  
  
	}  
  
	public User get(String id) throws SQLException, ClassNotFoundException {  
	    Class.forName("com.mysql.cj.jdbc.Driver");  
	    Connection c = DriverManager.getConnection(  
	            "jdbc:mysql://localhost/hyejindb", "hyejin66", "u1234");  
	    PreparedStatement ps = c.prepareStatement(  
	            "select * from users where id = ?");  
	  
	    ps.setString(1, id);  
	    ps.execute();  
	  
	    ResultSet rs = ps.executeQuery();  
	    rs.next();  
	  
	    User user = new User();  
	    user.setId(rs.getString("id"));  
	    user.setName(rs.getString("name"));  
	    user.setPassword(rs.getString("password"));  
	  
	    return user;  
	}
}
```

위의 코드는 새로운 사용자를 생성하는 add() 메서드와 아이디를 기준으로 사용자의 정보를 가져오는 get() 메서드로 이루어져 있다.

## UserDao 테스트 코드(+ DB 연결 확인)
window 기준 클래스명 누르고 `alt + enter` 한 후에 테스트 코드 만들기!
```java
package org.example.chapter1.dao;  
  
import static org.junit.jupiter.api.Assertions.*;  
  
import java.sql.SQLException;  
import org.example.chapter1.domain.User;  
import org.junit.jupiter.api.Test;  
  
class UserDaoTest {  
  
    @Test  
    public void addAndGet() throws SQLException, ClassNotFoundException {  
        UserDao dao = new UserDao();  
        User user = new User();  
  
        user.setId("1");  
        user.setName("최따루");  
        user.setPassword("1234");  
  
        dao.add(user);  
  
        User result = dao.get("1");  
        System.out.println("UserId:::: " + result.getId());  
        System.out.println("UserName:::: " + result.getName());  
        assertEquals("1", result.getId());  
  
    }  
}
```

![스크린샷 2024-09-30 221739](https://github.com/user-attachments/assets/0a106e9e-41bc-4276-9e3b-b9a378b29260)<br>
db 에 잘 저장된 걸 확인할 수 있다.
![스크린샷 2024-09-30 222030](https://github.com/user-attachments/assets/ebd4a9b2-5188-4000-862c-1c555467ce79)<br>
get() 메서드를 통해 id 칼럽값이 1인 User 의 정보도 잘 출력되는 걸 확인할 수 있다.

## <mark>**요약**</mark><br>
> 지금 만든 UserDao 클래스 코드는 **여러 문제점이** 있다.<br>
> 문제점이라 하면 보통 코드가 정상적으로 작동되지 않아야 할텐데,
> 우리가 기대한 동작들이 충실히 동작된다는 점을 확인했다.
> 
><u> 그렇다면? </u>
> <u>뭐가 문제일까? 왜 개선해야 되는 걸까?</u>
> <u>잘 동작되는 코드를 굳이 수정하고 개선해야 되는 이유는 뭘까?</u>
> 
> 이제부터 알아보도록 하자.

# 2. DAO 의 분리
## 관심사의 분리
프로그래밍의 기초 개념 중 하나이다.<br>
객체지향에 적용해보면, 관심이 같은 것끼리는 하나의 객체 안으로 또는 친한 객체로 모이게 하고, 관심이 다른 것은 가능한 한 따로 떨어져서 서로 영향을 주지 않도록 분리하는 것이다.

## 커넥션 만들기의 추출 (중복된 코드 최적화)
**기존에 있던 UserDao 클래스에서의 문제점**
-  **DB연결을 위한 Connection 을 가져오는 부분**<br>
   사용자 등록을 위해 DB 에 보낼 SQL 문장을 담은 Statement  부분과 마지막에 오브젝트를 닫아주고 공유 리소스를 돌려주는 부분의 관심사와 섞여있다.<br>
   <mark>더 큰 문제는 코드가 get() 메서드 안의 코드와 중복된 코드이다.</mark>

DB 연결 코드는 getConnction() 이라는 메소드를 만들어서 중복된 코드를 분리한다.
```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.DriverManager;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import org.example.chapter1.domain.User;  
  
public class UserDao {  
    //중복 코드 최적화  
    private Connection getConnection() throws ClassNotFoundException, SQLException {  
        Class.forName("com.mysql.cj.jdbc.Driver");  
        Connection c = DriverManager.getConnection(  
                "jdbc:mysql://localhost/hyejindb", "hyejin66", "u1234");  
        return c;  
    }  
  
    public void add(User user) throws ClassNotFoundException, SQLException {  
        Connection c = getConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "insert into users(id, name, password) values(?,?,?)");  
  
        ps.setString(1, user.getId());  
        ps.setString(2, user.getName());  
        ps.setString(3, user.getPassword());  
  
        ps.executeUpdate();  
  
        ps.close();  
        c.close();  
  
    }  
  
    public User get(String id) throws SQLException, ClassNotFoundException {  
        Connection c = getConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "select * from users where id = ?");  
  
        ps.setString(1, id);  
        ps.execute();  
  
        ResultSet rs = ps.executeQuery();  
        rs.next();  
  
        User user = new User();  
        user.setId(rs.getString("id"));  
        user.setName(rs.getString("name"));  
        user.setPassword(rs.getString("password"));  
  
        return user;  
    }  
}
```
## 변경사항에 대한 검증 : 리펙토링과 테스트
위의 과정을 통해서 **리팩토링의 메소드 추출 기법**을 사용했다.<br><br>

> **<mark>리팩토링이란?</mark>**<br>
>> 기존의 코드를 외부의 동작방식에는 변화 없이 내부 구조를 변경해서 재구성하는 작업 또는 기술을 말한다.<br><br>
>>
> **<mark>리팩토링의 이점**</mark><br>
>> 1. 코드 내부의 설계가 개선 → 코드 이해 ↑
>> 2. 생산성 높아진다.
>> 3. 코드의 품질 높아진다.
>> 4. 유지보수에 용이하다.
>> 5. 코드가 유연해진다.


## DB 커넥션 만들기의 독립

> **UserDao 가 너무 잘 짜여있어서  N사와 D사에서 사용자 관리를 위해 이 UserDao를 구매하겠다는 주문이 들어왔다.**
>> 문제는 N사와 D사가 각기 다른 종류의 DB 를 사용하고 있고, DB 커넥션을 가져오는데 있어 독자적으로 만든 방법을 적용하고 싶다는 점이다.
>>
>> **UserDao 의 소스코드를 제공해주지 않고도 고객 스스로가 원하는  DB 커넥션 생성 방식을 적용하면서, UserDao 를 사용할 수 있을까?**

### 방법 1. 상속을 통한 확장 (템플릿 메서드 패턴)
---
UserDao 에서 메소드의 구현 코드를 제거하고, getConnection() 을 추상 메소드로 만들어 놓는다. 
추상 메소드라서 메소드 코드는 없지만 메소드 자체는 존재한다.
따라서, add()와 get() 메소드에서 getConncetion() 을 호출하는 코드는 그대로 유지할 수 있다.

#### 슈퍼클래스 
```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import org.example.chapter1.domain.User;  
  
  
public abstract class UserDaoAbstract {  
    //중복 코드 최적화  
    //추상메소드
    public abstract Connection getConnection() throws ClassNotFoundException, SQLException;  
  
    public void add(User user) throws ClassNotFoundException, SQLException {  
        Connection c = getConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "insert into users(id, name, password) values(?,?,?)");  
  
        ps.setString(1, user.getId());  
        ps.setString(2, user.getName());  
        ps.setString(3, user.getPassword());  
  
        ps.executeUpdate();  
  
        ps.close();  
        c.close();  
  
    }  
  
    public User get(String id) throws SQLException, ClassNotFoundException {  
        Connection c = getConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "select * from users where id = ?");  
  
        ps.setString(1, id);  
        ps.execute();  
  
        ResultSet rs = ps.executeQuery();  
        rs.next();  
  
        User user = new User();  
        user.setId(rs.getString("id"));  
        user.setName(rs.getString("name"));  
        user.setPassword(rs.getString("password"));  
  
        return user;  
    }  
}
```
#### 서브클래스
```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.DriverManager;  
import java.sql.SQLException;  
  
//chapter1의 db 커넥션 만들기의 독립  
public class NUserDao extends UserDaoAbstract {  
    @Override  
    public Connection getConnection() throws ClassNotFoundException, SQLException {  
        //템플릿 메서드 패턴 적용  
        //서로 다른 db 에 접속하는 방법이 다르기 때문에 서브클래스에서 구현하도록 함  
        //url을 바꾸거나, 사용자 이름과 비밀번호를 바꾸는 등의 작업이 필요할 때 서브클래스에서 getConnection()을 오버라이드해서 구현하면 된다  
        //관심사를 분리 db에 관련된 코드는 UserDaoAbstract에, db에 접속하는 방법은 서브클래스에 위임  
        Class.forName("com.mysql.cj.jdbc.Driver");  
        Connection c = DriverManager.getConnection(  
                "jdbc:mysql://localhost/hyejindb", "hyejin66", "u1234");  
        return c;  
    }  
  
}
```

UserDao 를 구입한 포탈사들은 UserDao 클래스를 상속해서 각 서브클래스를 만든다.
서브클래스에서는 UserDao 에서 추상 메소드로 선언했던 getConncetion() 메소드를 원하는 방식대로 구현할 수 있다.

현재 위의 코드는 DAO의 핵심 기능인 어떻게 데이터를 등록하고 가져올지 에 대한 관심을 담당하는  UserDaoAbstract와 DB 연결 방법을 어떻게 할 것인지에 대한 관심을 담고 있는 NUserDao 클래스 레벨로 구분돼 있다.

![image](https://github.com/user-attachments/assets/f57862c5-6974-4e3c-8ae2-2b9d9869a2b8)

<mark>**위 방식은 기존의 같은 클래스에 다른 메소드로 분리했던 DB 커넥션 연결이라는 관심을 상속을 통해 서브클래스로 분리한 것이다.**</mark>

> <mark>**템플릿 메소드 패턴**</mark>
>> 디자인 패턴 중 하나이다.
>> 상속을 통해 슈퍼클래스의 기능을 확장할 때 사용하는 가장 대표적인 방법이다.
>> 
>> 슈퍼클래스에는 변하지 않는 기능을 만들고, 서브클래스에는 자주 변경되며 확장할 기능을 만든다.
>> 슈퍼클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메소드나 오버라이딩 가능한 protected 메소드 등으로 만든 뒤 서브클래스에서 이런 메소드를 필요에 맞게 구현해서 사용하도록 한다.
>> 
> <mark>**예시**</mark>
>>  알고리즘의 공통적인 흐름을 정의하고, 세부 구현을 서브클래스에서 다르게 할 때 사용된다.
>위의 코드가 템플릿 메소드 패턴의 예시이다.

### 방법 2. 상속을 통한 확장 (팩토리 메서드 패턴)
---

UserDao의 getConnction() 메소드는 Connction 타입 오브젝트를 생성한다는 기능을 정의해놓은 추상 메소드이다. 
UserDao의 서브클래스의  getConnction() 메서드는 어떤 Conncetion 클래스의 오브젝트를 어떻게 생성할 것인지를 결정하는 방법이라고 볼 수 있다.

이렇게 **<u>서브클래스에서 구체적인 오브젝트를 생성하는 방법을 결정하는 것을 팩토리 메서드 패턴이라고 한다.</u>**

![image](https://github.com/user-attachments/assets/c5d632b4-6a87-4997-929a-63d971ddaf70)

##### 코드를 통한 이해
```java
// UserDao.java
public abstract class UserDao {
	//추상메서드
	//팩터리 메서드: 하위 클래스에서 DB 연결 방법을 정의
    public abstract Connection getConnection();
    
    public void add(User user) {
        Connection conn = getConnection();
        // DB 작업 수행
    }

    public User get(int id) {
        Connection conn = getConnection();
        // DB 작업 수행
        return user;
    }
}

// NUserDao.java
public class NUserDao extends UserDao {
    @Override
    public Connection getConnection() {
        // N 데이터베이스에 대한 연결 리턴
        return new AConnection();
    }
}

// DUserDao.java
public class DUserDao extends UserDao {
    @Override
    public Connection getConnection() {
        // D 데이터베이스에 대한 연결 리턴
        return new BConnection();
    }
}
```

> <mark>**팩토리 메소드 패턴이란?**</mark>
>> 템플릿 메소드 패턴과 마찬가지로 상속을 통해 기능을 확장하는 패턴이다.
>> 템플릿 메소드 패턴과 구조도 비슷하다.
>> 
>> 슈퍼 클래스 코드에서는 서브클래스에서 구현할 메소드를 호출해서 필요한 타입의 오브젝트를 가져와 사용한다.
>> 이 메소드는 주로 인터페이스 타입으로 오브젝트를 리턴하므로 서브클래스에서 정확히 어떤 클래스의 오브젝트를 만들어 리턴할지는 슈퍼클래스에서 알지 못 한다. 
>> 
>> 서브클래스는 다양한 방법으로 오브젝트를 생성하는 메소드를 재정의할 수 있다.
>
>
> <mark>**팩토리 메소드란?**</mark>
>> 서브클래스에서 오브젝트 생성 방법과 클래스를 결정할 수 있도록 미리 정의해둔 메소드<br>
>
> **<mark>팩토리 메소드 패턴이란?</mark>**
>> 팩토리 메소드 방식을 통해 오브젝트 생성 방법을 독립시키는 방법<br>
>> 
><mark>**예시**</mark><br>
>> 다양한 객체를 생성해야 할 때 사용된다. <br>
>위의 코드는 팩토리 메소드 패턴의 예시이다.

### 문제점
---
<u>상속을 사용했다는 단점이 있다.</u>
자바는 클래스의 다중상속을 허용하지 않는다. 
커넥션 객체를 가져오는 방법을 분리하기 위해 상속구조로 만들면, 후에 다른 목적으로 UserDao 에 상속을 적용하기 어렵다.

*서브클래스는 슈퍼클래스의 기능을 직접 사용할 수 있어서 슈퍼클래스의 내부 변경이 있을 때 모든 서브클래스도 함께 수정하거나 개발해야 될 수 있다.*

# 3. DAO 의 확장
## 방법 1. 클래스의 분리
첫 시도: 독립된 메소드를 만들어서 분리
두 번째 시도: 상하위 클래스로 분리
<u>세 번째 시도(현재!!) : 상속관계도 아닌 완전히 독립적인 클래스로 분리</u>

![image](https://github.com/user-attachments/assets/0a306daf-3ec6-44a4-af7a-5b621156d387)

SimpleConnectionMaker 이라는 새로운 클래스를 만들고 DB 생성 기능을 안에 넣는다.<br>
UserDao 는 new 키워드를 사용해 SimpleConncetionMaker 클래스의 오브젝트를 만들어주고, 
<br>이를 UserDao 클래스에 있는 add(), get() 메서드에서 사용하면 된다.

코드를 봐보자.
```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import org.example.chapter1.domain.User;  
  
public class UserDaoSimpleConnectionMaker {  
	private SimpleConnectionMaker simpleConnectionMaker;  
      
    public UserDaoSimpleConnectionMaker() {  
        simpleConnectionMaker = new SimpleConnectionMaker();  
    }  
  
  
    public void add(User user) throws ClassNotFoundException, SQLException {  
        Connection c = simpleConnectionMaker.makeConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "insert into users(id, name, password) values(?,?,?)");  
  
        ps.setString(1, user.getId());  
        ps.setString(2, user.getName());  
        ps.setString(3, user.getPassword());  
  
        ps.executeUpdate();  
  
        ps.close();  
        c.close();  
  
    }  
  
    public User get(String id) throws SQLException, ClassNotFoundException {  
        Connection c = simpleConnectionMaker.makeConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "select * from users where id = ?");  
  
        ps.setString(1, id);  
        ps.execute();  
  
        ResultSet rs = ps.executeQuery();  
        rs.next();  
  
        User user = new User();  
        user.setId(rs.getString("id"));  
        user.setName(rs.getString("name"));  
        user.setPassword(rs.getString("password"));  
  
        return user;  
    }  
}
```

```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.DriverManager;  
import java.sql.SQLException;  
//추상클래스로 만들 필요성이 없이  
public class SimpleConnectionMaker {  
    public Connection makeConnection() throws ClassNotFoundException, SQLException {  
        Class.forName("com.mysql.cj.jdbc.Driver");  
        Connection c = DriverManager.getConnection(  
                "jdbc:mysql://localhost/hyejindb", "hyejin66", "u1234");  
        return c;  
    }  
    //1. 상속을 사용하는게 마음에 안 들어서 바꾼건데, db 커넥션을 제공하는 클래스가 어떤 것인지 userdaoConnectionMaker에서 구체적으로 알고 있어야 해서 수정을 또 해줘야 된다는 점  
    //종속적  
    //그럼 인터페이스 도입!!  
}
```

### 문제점
---
UserDao의 코드가 SimpleConnectionMaker 이라는 특정 클래스에 종속되어 있어서 UserDao 의 코드 수정 없이 DB 커넥션 생성 기능을 변경할 방법이 없다.<br>
결국 다시 원점으로 돌아온 상황에서 ,**우리가 해결해야 될 문제는 2가지 이다.**

1. SimpleConnectionMaker의 메소드 문제 
2. DB 커넥션을 제공하는 클래스가 어떤 건지 UserDao 가 구체적으로 알고 있어야 된다는 문제
→ UserDao 는 DB 커넥션을 가져오는 구체적인 방법에 종속된다.

## 방법 2. 인터페이스의 도입
두 개의 클래스가 서로 긴밀하게 연결되지 않도록 중간에 추상적인 느슨한 연결고리를 만들어주는 방법<br>
**추상화를 하기 위해 제공되는 가장 유용한 도구는 인터페이스이다.** <br>
인터페이스는 자신이 구현한 클래스에 대한 구체적인 정보는 모두 감춰버린다.

![image](https://github.com/user-attachments/assets/72fe5362-6673-4cb0-9576-d471c2b7fc27)

인터페이스는 어떤 일을 하겠다는 기능만 정의해 놓고, 구현 방법은 나타내지 않는다.

```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.SQLException;  
  
//인터페이스 상속을 받은 nconnectionMaker, dconnectionMaker 구현할 예정  
public interface ConnectionMaker {  
    public Connection makeConnection() throws SQLException, ClassNotFoundException;  
}
```

```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.DriverManager;  
import java.sql.SQLException;  
  
public class NConnectionMaker implements ConnectionMaker{  
    @Override  
    public Connection makeConnection() throws SQLException, ClassNotFoundException {  
        Class.forName("com.mysql.cj.jdbc.Driver");  
        Connection c = DriverManager.getConnection(  
                "jdbc:mysql://localhost/hyejindb", "hyejin66", "u1234");  
        return c;  
    }  
}
```

```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import org.example.chapter1.domain.User;  
  
public class UserDaoConnectionMaker {  
  
    private ConnectionMaker connectionMaker;  
  
    //관계설정 책임 분리가 안 된 상태  
    public UserDaoConnectionMaker() {  
        //관계설정 책임의 분리을 해야 되는데,, 이렇게 하면 UserDaoConnectionMaker가 ConnectionMaker의 구현 클래스에 종속적이다.  
        connectionMaker= new NConnectionMaker();  // 여기에 클래스 명이 나옴
    }  
  
    public void add(User user) throws ClassNotFoundException, SQLException {  
        Connection c = connectionMaker.makeConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "insert into users(id, name, password) values(?,?,?)");  
  
        ps.setString(1, user.getId());  
        ps.setString(2, user.getName());  
        ps.setString(3, user.getPassword());  
  
        ps.executeUpdate();  
  
        ps.close();  
        c.close();  
  
    }  
  
    public User get(String id) throws SQLException, ClassNotFoundException {  
        Connection c = connectionMaker.makeConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "select * from users where id = ?");  
  
        ps.setString(1, id);  
        ps.execute();  
  
        ResultSet rs = ps.executeQuery();  
        rs.next();  
  
        User user = new User();  
        user.setId(rs.getString("id"));  
        user.setName(rs.getString("name"));  
        user.setPassword(rs.getString("password"));  
  
        return user;  
    }  
}
```

### 문제점
---
UserDao 의 다른 모든 곳에서는 인터페이스를 이용해 DB 커넥션을 제공하는 클래스에 대한 구체적인 정보를 모두 제거했으나 초기에 한 번 어떤 클래스의 오브젝트를 사용할지를 결정하는 생성자 코드는 제거되지 않고 남아있다.

## 방법 3. 관계설정 책임의 분리
UserDao 가 어떤 ConnectionMaker 구현 클래스의 오브젝트를 이용할지 결정하는 것이다.<br>
UserDao 오브젝트와 특정 클래스로부터 만들어진 ConnectionMaker 오브젝트 사이에 관계를 설정해주는 것이다.<br>
따라서 클래스가 아니라 오브젝트와 오브젝트 사이의 관계를 설정해주는 것이다. 
```java
connectionMaker= new NConncetionMaker();
```
이런 식으로 직접 생성자를 호출해서 직접 오브젝트를 만드는 방법도 있지만, 이는 관심설정 책임 분리가 안 된 상태이다.<br>

그렇기 때문에 외부에서 만들어준 것을 가져오는 방법을 사용하려고 한다.

![image](https://github.com/user-attachments/assets/54336c7e-92cd-4f80-869d-3152bd8ed617)


**UserDao 의 모든 코드는 ConnectionMaker 인터페이스 외에는 어떤 클래스와도 관계를 가져서는 안된다.** <br>

객체지향의 다형성이라는 특징 덕분에 특정 클래스를 전혀 알지 못하더라도 해당 클래스가 구현한 인터페이스를 사용했다면, 그 클래스의 오브젝트를 인터페이스 타입으로 받아서 사용할 수 있다.

```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import org.example.chapter1.domain.User;  
  
public class UserDaoConnectionMaker {  
  
    private ConnectionMaker connectionMaker;  
  
    //관계설정 책임 분리가 된 상태  
    //현재 인터페이스랑만 관계가 맺어져 있는 것을 확인할 수 있다.  
    public UserDaoConnectionMaker(ConnectionMaker connectionMaker) {  
        this.connectionMaker = connectionMaker;   //특정 클래스명이 사라졌다!!
    }  
  
    public void add(User user) throws ClassNotFoundException, SQLException {  
        Connection c = connectionMaker.makeConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "insert into users(id, name, password) values(?,?,?)");  
  
        ps.setString(1, user.getId());  
        ps.setString(2, user.getName());  
        ps.setString(3, user.getPassword());  
  
        ps.executeUpdate();  
  
        ps.close();  
        c.close();  
  
    }  
  
    public User get(String id) throws SQLException, ClassNotFoundException {  
        Connection c = connectionMaker.makeConnection();  
        PreparedStatement ps = c.prepareStatement(  
                "select * from users where id = ?");  
  
        ps.setString(1, id);  
        ps.execute();  
  
        ResultSet rs = ps.executeQuery();  
        rs.next();  
  
        User user = new User();  
        user.setId(rs.getString("id"));  
        user.setName(rs.getString("name"));  
        user.setPassword(rs.getString("password"));  
  
        return user;  
    }  
}
```

<mark>NConnectionMaker 는 왜 사라졌을까?</mark><br>
NConnectionMaker는 UserDao 와 특정 ConnectionMaker 구현 클래스의 관계를 맺는 책임을 담당하는 코드였는데, 이를 <u>클라이언트에게 넘겨버렸기 때문이다.</u><br>

그 대신 테스트코드에서 UserDao와 ConnectionMaker 구현 클래스와의 런타임 오브젝트 의존관계를 설정하는 책임을 담당해야 된다!<br>

이렇게 해서 UserDao 에는 전혀 손대지 않고 DB 연결 기능을 확장해서 사용할 수 있게 된다.

![image](https://github.com/user-attachments/assets/8697f340-b13c-47ae-82d3-bb0d61fc78d6)

관계설정 책임을 담당하는 클라이언트 UserDaoTest 가 추가된 구조

### 결론
---
상속을 통한 확장 방법보다 **더 깔끔하고 유연한 방법**으로 UserDao 와 ConncetionMaker 클래스들을 분리하고, 서로 영향을 주지 않으면서도 필요에 따라 자유롭게 확장할 수 있는 구조가 됐다.

## 원칙과 패턴

![image](https://github.com/user-attachments/assets/2306fd43-9db5-4f2a-802b-fd53d2292d7d)


> **<mark>개방 폐쇄 원칙이란?</mark>**
>> 깔끔한 설계를 위해 적용 가능한 객체지향 설계 원칙 중 하나이다.<br>
>> 클래스나 모듈은 확장에는 열려 있어야 하고, 변경에는 닫혀 있어야 한다.<br>
> 위의 그림은 개방 폐쇄 원칙을 잘 보여주는 그림이다.<br>
> 
>> **인터페이스를 사용해 확장 기능을 정의한 대부분의 API 는 개방 폐쇄 원칙을 따른다고 볼 수 있다.** <br>
>> 또한 개방 폐쇄 원칙은 높은 응집도와 낮은 결합도라는 원리로도 설명 가능하다.<br>
>> <mark>`높은 응집도`</mark>: 하나의 모듈, 클래스가 하나의 책임 또는 관심사에만 집중되어 있다는 뜻이다.<br>
>> <mark>`낮은 결합도`</mark> : 하나의 오브젝트가 변경이 일어날 때 관계를 맺고 있는 다른 오브젝트에게 변화를 요구하는 정도가 낮다는 것 , 즉 느슨한 연결 상태를 유지하고 있는 것을 의미한다.<br>
>> 결합도가 낮으면 변화에 대응하는 속도가 높고 구성이 깔끔해지며, 확장에도 용이하다. <br>
>
<br>

> **<mark>전략 패턴이란?</mark>**
>> 자신의 기능 맥락에서 필요에 따라 변경이 필요한 알고리즘을 **인터페이스를 통해 통째로 외부로 분리** 시키고, 이를 구현한 구체적인 (독립적 책임으로 분리하는) 알고리즘 클래스를 필요에 따라 바꿔 사용할 수 있게 하는 디자인 패턴이다.
>
>> UserDao 라는 컨텍스트를 사용하는 UserDaoTest 라는 클라이언트는 컨텍스트가 사용할 전략인 ConnectionMaker 를 구현한 클래스를 컨텍스트 생성자를 통해 제공해주는 것이 일반적이다.
>

```java
UserDaoConnectionMaker dao = new UserDaoConnectionMaker(new NConnectionMaker());
//이런 식으로 UserDaoTest 에서 해줘야 된다.
```
<br>

> <mark>**객체지향 설계 원칙(SOLID)**</mark>
>> 객체지향 설계 원칙은 객체지향의 특징을 잘 살릴 수 있는 설계의 특징을 말한다.
>> SRP (The Single Responsibility Principal): 단일 책임의 원칙
>> OCP (The Open Closed Principal): 개방 폐쇄 원칙
>> LSP (The Liskov Substitution Principal): 리스코프 치환 원칙
>> ISP (The Interface Segregation Principal): 인터페이스 분리 원칙
>> DIP (The Dependency Inversion Principal): 의존관계 역전 원칙

### 결론
---
위의 그림은 **개방 폐쇄 원칙**을 잘 따르고 있으며, **응집력이  높고 결합도는 낮으며**, **전략 패턴**을 잘 적용했음을 알 수 있다.

# 4. 제어의 역전 (IOC)
## 오브젝트 팩토리
원래 UserDaoTest 는  UserDao 의 기능이 잘 동작하는지를 테스트하려고 만든 것 아닌가?
근데!! 지금 다른 책임까지 떠맡고 있으니 뭔가 문제가 있다.

또 분리를 해줘야 되는데, 이때 분리될 기능은 UserDao 와 ConnectionMaker 구현 클래스의 오브젝트를 만드는 것과, 그렇게 만들어진 두 개의 오브젝트가 연결돼서 사용될 수 있도록 관계를 맺어주는 것이다.

```java
package org.example.chapter1.dao;  
//팩토리 메소드 패턴  
public class DaoFactory {  
    public UserDaoConnectionMaker userDaoConnectionMaker() {  
        return new UserDaoConnectionMaker(new NConnectionMaker());  
    }  
}
```
> **<mark>팩토리란?</mark>**
>> 객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 돌려주는 것

위의 코드를 보면 DaoFactory의 userDaoConnectionMaker() 메서드를 호출하면 NConnectionMaker를 사용해 DB 커넥션을 가져오도록 이미 설정된 userDaoConnectionMaker 오브젝트를 돌려준다.

### 테스트코드
---

```java
package org.example.chapter1.dao;  
  
import static org.junit.jupiter.api.Assertions.*;  
  
import java.sql.SQLException;  
  
import org.example.chapter1.domain.User;  
import org.junit.jupiter.api.Test;  
import org.junit.jupiter.api.extension.ExtendWith;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.ApplicationContext;  
import org.springframework.test.context.ContextConfiguration;  
import org.springframework.test.context.junit.jupiter.SpringExtension;  
  
@ExtendWith(SpringExtension.class)  
@ContextConfiguration(classes = DaoFactoryContext.class)  
class UserDaoConnectionMakerTest {  
    @Autowired  
    private ApplicationContext context;  
  
    @Test  
    public void addAndGet() throws SQLException, ClassNotFoundException {  

	    //한줄 처리 (팩토리 메소드 패턴 적용)  
        UserDaoConnectionMaker dao = new DaoFactory().userDaoConnectionMaker();  
        User user = new User();  
  
        user.setId("2");  
        user.setName("최혜진");  
        user.setPassword("1234");  
  
        dao.add(user);  
  
        User result = dao.get("2");  
        System.out.println("UserId:::: " + result.getId());  
        System.out.println("UserName:::: " + result.getName());  
        assertEquals("2", result.getId());  
  
    }  
  
}
```

### 설계도로서의 팩토리
---

![image](https://github.com/user-attachments/assets/e6ca8eb3-4325-41b6-83e5-0bdd63fcff27)

`UserDao`: 사용자 데이터 관련 로직에 대한 책임<br>
`ConnectionMaker`: DB 연결 기술에 관한 책임<br>
`UserDaoTest`: 동작을 테스트하는 책임<br>
`DaoFactory`: 오브젝트를 구성하고 관계를 정의하는 책임<br>

## 오브젝트 팩토리의 활용
DAO 생성 메소드의 추가로 인해 발생하는 중복을 제거한 ConnectionMaker 타입
```java
public class DaoFactory {
    public UserDao userDao() {
        return new UserDao(getConnectionMaker());
    }

    public MessageDao messageDao() {
        return new MessageDao(getConnectionMaker());
    }

    public AccountDao accountDao() {
        return new AccountDao(getConnectionMaker());
    }

    private ConnectionMaker getConnectionMaker() {
        return new DSimpleConnectionMaker();
    }
}
```
## 팩토리로 배워보는 제어관계 역전
이전까지는 클라이언트가 직접 어떤 클래스를 사용할지에 대해 구체적인 클래스를 결정했다.<br>
이제 팩토리가 어떤 클래스를 사용할지 구체적인 클래스를 대신 결정한다.<br>
**즉, 클라이언트는 모든 제어 권한을 팩토리에게 위임한 것과 같은 의미이다.** <br>

## 템플릿 메소드 패턴 관점에서의 제어의 역전
서브 클래스에서 작성한 메서드는 상위 클래스가 지정한 흐름에서 호출된다.<br>
제어권을 상위 템플릿 메서드에 넘기고 자신은 필요할 때 호출되어 사용되도록 한다는, 제어의 역전 개념을 이용해 문제를 해결한다.

## 프레임워크 관점에서의 제어의 역전
프레임워크는 애플리케이션 코드가 프레임워크에 의해 사용된다.<br>
보통 프레임워크 위에 개발한 클래스를 등록해두고, 프레임워크가 흐름을 주도하는 중 개발자가 만든 애플리케이션 코드를 사용하는 방식이다.<br>

# 5. 스프링의 IoC
## 오브젝트 팩토리를 이용한 스프링 IoC

> **<mark>bean이란?</mark>**<br>
>> 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트<br>
>> 오브젝트 단위의 애플리케이션 컴포넌트를 의미<br>
>> 스프링 컨테이너가 생성과 관계설정, 사용 등을 제어해주는 제어의 역전이 적용된 오브젝트를 의미<br>

> <mark>bean factory린?</mark><br>
>> 빈의 생성과 관계설정 같은 제어를 담당하는 IoC 오브젝트<br>
>> 빈을 생성하고 관계를 설정하는 IoC 의 기본 기능에 초점을 맞춤<br>

><mark>application context란?</mark><br>
>> 빈 팩토리보다는 좀 더 확장된 것<br>
>> 애플리케이션 전반에 걸쳐 모든 구성요소의 제어 작업을 담당하는 IoC 엔진이다.<br>
>> 제어 작업을 총괄한다!!<br>

DaoFactory를 BeanFactory 의 설정 정보로 만들어보자.
```java
package org.example.chapter1.dao;  
  
  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  

@Configuration // `애플리케이션 컨텍스트` 혹은 `빈 팩토리`가 사용할 설정 정보라는 표시이다.
public class DaoFactory {

    @Bean // 오브젝트 생성을 담당하는 IoC용 메소드라는 표시이다.
    public UserDao userDao() {
        return new UserDao(getConnectionMaker());
    }

    @Bean // 오브젝트 생성을 담당하는 IoC용 메소드라는 표시이다.
    public DSimpleConnectionMaker getConnectionMaker() {
        return new DSimpleConnectionMaker();
    }
}
```

DaoFactory 빈 활용하기
```java
package org.example.chapter1.dao;  
  
import static org.junit.jupiter.api.Assertions.*;  
  
import java.sql.SQLException;  
  
import org.example.chapter1.domain.User;  
import org.junit.jupiter.api.Test;  
import org.junit.jupiter.api.extension.ExtendWith;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.ApplicationContext;  
import org.springframework.test.context.ContextConfiguration;  
import org.springframework.test.context.junit.jupiter.SpringExtension;  
  
@ExtendWith(SpringExtension.class)  
@ContextConfiguration(classes = DaoFactoryContext.class)  
class UserDaoConnectionMakerTest {  
    @Autowired  
    private ApplicationContext context;  
  
    @Test  
    public void addAndGet() throws SQLException, ClassNotFoundException {  
       UserDaoConnectionMaker dao=  context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  
        User user = new User();  
  
        user.setId("2");  
        user.setName("최혜진");  
        user.setPassword("1234");  
  
        dao.add(user);  
  
        User result = dao.get("2");  
        System.out.println("UserId:::: " + result.getId());  
        System.out.println("UserName:::: " + result.getName());  
        assertEquals("2", result.getId());  
  
    }  
  
  
}
```

## 애플리케이션 컨텍스트의 동작방식

![image](https://github.com/user-attachments/assets/65554b9b-c754-4e36-9a2e-494c2cafadfb)

애플리케이션 컨텍스트는 애플리케이션에서 IoC를 적용해서 관리할 모든 오브젝트에 대한 생성과 관계설정을 담당한다.<br>

>애플리케이션 컨텍스트는 @Configuration 이 붙은 클래스를 설정정보로 등록해두고, @Bean 이 붙은 메서드의 이름을 가져와 빈 목록을 만들어둔다. 클라이언트가 이 애플리케이션 컨텍스트의 `getBean()` 메서드를 호출하면 자신의 빈 목록에서 요청한 이름이 있는지 찾고, 있다면 빈을 생성하는 메서드를 호출해서 오브젝트를 생성시킨 후 클라이언트에 돌려준다.

><mark>**애플리케이션 컨텍스트를 사용했을 때 장점**</mark><br>
>> 1. 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다.<br>
>> 2. 애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다.<br>
>> 3. 애플리케이션 컨텍스트는 빈을 검색하는 다양한 방법을 제공한다.<br>

# 6. 싱글톤 레지스트리와 오브젝트 스코프

> <mark>오브젝트의 동일성과 동등성</mark><br>
>
> `동일성`<br>
>> 두 개의 오브젝트가 완전히 같은 오브젝트라는 뜻으로, == 연산자를 이용해 비교한다.<br>
>> 사실 하나의 오브젝트만 존재하는 것이고 두 개의 오브젝트 래퍼천스 변수를 갖고 있는 것이다.<br>
>
>`동등성`<br>
>> 두 개의 오브젝트가 동일한 정보를 담고 있다는 뜻으로, equals() 메서드를 이용해 비교한다.<br>
>> 두 개의 각기 다른 오브젝트가 메모리상에 존재하는 것이다.<br>

## 스프링 컨텍스트로부터 가져온 오브젝트 출력 코드

```java
ApplicationContext ac = new AnnotationConfigApplicationContext(DaoFactory.class);
UserDao dao3 = ac.getBean("userDao", UserDao.class);
UserDao dao4 = ac.getBean("userDao", UserDao.class);

System.out.println(dao3);
System.out.println(dao4);
```

`출력결과`
```plain text
com.springbook.user.dao.UserDao@5b38c1ec
com.springbook.user.dao.UserDao@5b38c1ec
```
 getBean()을 두 번 호출해서 가져온 오브젝트가 동일하다는 사실을 알 수 있다.<br>
 
우리가 만들었던 오브젝트 팩토리와 스프링의 애플리케이션 컨텍스트의 동작방식에 무엇인가 차이점이 있다. <br>
<u>스프링은 여러 번에 걸쳐 빈을 요청하더라도 매번 동일한 오브젝트를 돌려준다는 것인데, 왜 그럴까?</u><br>


> <mark>싱글턴 패턴이란?</mark><br>
>> 어떤 클래스를 애플리케이션 내에서 제한된 인스턴스 개수, 이름처럼 주로 하나만 존재하도록 강제하는 패턴이다.<br>
>> 이렇게 하나만 만들어지는 클래스의 오브젝트는 애플리케이션 내에서 전역적으로 접근이 가능하다.<br>
>> 단일 오브젝트만 존재해야 하고 , 애플리케이션의 여러 곳에서 공유하는 경우에 주로 사용한다.<br>

## 싱글톤 레지스트리로서의 애플리케이션 컨텍스트
애플리케이션 컨텍스트는 우리가 만들었던 오브젝트 팩토리와 비슷한 방식으로 동작하는 IoC 컨테이너면서 동시에 싱글톤을 저장하고 관리하는 싱글톤 레지스트리(singleton registry)이기도 하다.<br>
**스프링은 기본적으로 별다른 설정을 하지 않으면 내부에서 생성하는 빈 오브젝트를 모두 싱글톤으로 만든다.**

### 서버의 애플리케이션과 싱글톤 
---
<u>Q. 왜 스프링은 싱글톤으로 빈을 만드는 것일까? </u><br>
**이는 스프링이 주로 적용되는 대상이 자바 엔터프라이즈 기술을 사용하는 서버환경이기 때문이다.** <br>

**매번 클라이언트에서 요청이 올 때마다 각 로직을 담당하는 오브젝트를 새로 만들어서 사용한다고 생각해보자. 아무리 자바의 오브젝트 생성과 가비지 컬렉션의 성능이 좋아졌다고 한들 이렇게 부하가 걸리면 서버가 감당하기 힘들다.**
### 싱글톤 패턴의 한계
---
- 클래스 밖에서는 오브젝트를 생성하지 못하도록 생성자를 private으로 만든다.<br>
- 생성된 싱글톤 오브젝트를 저장할 수 있는 자신과 같은 타입의 스태틱 필드를 정의한다.<br>
- 스태틱 팩토리 메서드인 getInstance()를 만들고 이 메소드가 최초로 호출되는 시점에서 한번만 오브젝트가 만들어지게 한다. 생성된 오브젝트는 스태틱 필드에 저장된다. 또는 스태틱 필드의 초기값으로 오브젝트를 미리 만들어둘 수도 있다.<br>
- 한번 오브젝트(싱글톤)가 만들어지고 난 후에는 getInstance() 메서드를 통해 이미 만들어져 스태틱 필드에 저장해둔 오브젝트를 넘겨준다.<br>

`싱글톤 패턴을 적용한 UserDao`
```java
public class UserDao {
    private static UserDao INSTANCE;
    private static ConnectionMaker connectionMaker;

    private UserDao(ConnectionMaker connectionMaker) {
        this.connectionMaker = connectionMaker;
    }
    
    public static synchronized UserDao getInstance() {
        if (INSTANCE == null) INSTANCE = new UserDao(???);
        return INSTANCE;
    }
    ...
}
```
UserDao에 싱글톤 패턴을 도입함으로서 생기는 문제<br>
- private로 바뀐 생성자는 외부에서 호출할 수가 없기 때문에 DaoFactory에서 UserDao를 생성하며 ConnectionMaker 오브젝트를 넣어주는 게 불가능해졌다.<br>

싱글톤 패턴 구현 방식의 문제점들<br>
1. private 생성자를 갖고 있기 때문에 상속 불가<br>
2. 싱글톤은 테스트가 힘들다.<br>
3. 서버환경에서는 싱글톤이 하나만 만들어지는 것을 보장하지 못 한다.<br>
4. 싱글톤의 사용은 전역 상태를 만들 수 있기 때문에 바람직하지 못 하다.<br>

### 싱글톤 레지스트리
---
스프링은 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공한다. 이것이 바로 싱글톤 레지스트리이다.<br>

스프링 컨테이너는 싱글톤을 생성하고, 관리하고, 공급하는 싱글톤 관리 컨테이너이기도 하다.

#### 싱글톤 레지스트리의 장점
---
- **평범한 자바 클래스를 싱글톤으로 활용할 수 있다.**
    - `static`, `private 생성자`등을 이용할 필요가 없다.
- **테스트에서 싱글톤 방식으로 사용될 클래스를 활용할 수 있다.**
    - 테스트 환경에 따라 자유롭게 오브젝트를 만들 수 있다.
    - 생성자 파라미터에 구현체를 주입하는데도 아무런 문제가 없다.

## 싱글톤과 오브젝트의 상태
싱근톤이 멀티스레드 환경에서 서비스 형태의 오브젝트로 사용되는 경우에는 상태정보를 내부에 갖고 있지 않은 <mark>**무상태 `stateless` 방식으로 만들어져야 한다.**</mark><br>
싱글톤은 기본적으로 인스턴스 필드의 값을 변경하고 <u>유지하는 상태 유지 방식으로 만들지 않는다.</u><br>

`인스턴스 변수를 사용하도록 수정한 UserDao`
```java
public class UserDao {
    private static ConnectionMaker connectionMaker;
    private Connection c;
    private User user;

    public User get(String id) throws ClassNotFoundException, SQLException {
        this.c = connectionMaker.makeConnection();
        ...
        this.user = new User();
        this.user.setId(rs.getString("id"));
        this.user.setName(rs.getString("name"));
        this.user.setPassword(rs.getString("password"));
        ...
        return this.user;
    }
}
```
스프링의 싱글톤 빈으로 사용되는 클래스를 만들 때 인스턴스 필드로 선언해도 되는 것<br>
- `개별적으로 바뀌는 정보 - Connection, User`<br>
	- 싱글톤으로 만들어져서 멀티스레드 환경에서 사용하면 심각한 문제가 발생한다.<br>
	- 이런 정보는 로컬 변수로 정의하거나, 파라미터로 주고 받는 방법으로 사용해야 한다.<br>
- `읽기 전용 정보 - ConnectionMaker`<br>
	- 초기에 설정하면 사용 중에는 수정되지 않기 때문에 멀티스레드 환경에서 사용해도 문제가 없다.<br><br>

>동일하게 읽기전용의 속성을 가진 정보라면 싱글톤에서 인스턴스 변수로 사용해도 좋다. <br>
>물론 단순한 읽기전용 값이라면 static final이나 final로 선언하는 편이 낫다.

## 스프링 빈의 스코프 

>**<mark>빈의 스코프란?</mark>**
>빈이 생성되고, 존재하고, 적용되는 범위를 의미한다.<br>
>
>> `기본 스코프 : 싱글톤 스코프`<br>
>> 싱글톤 스코프는 컨테이너 내에 한 개의 오브젝트만 만들어져서, 강제로 제거하지 않는 한 스프링 컨테이너가 존재하는 동안 계속 유지된다.<br>
>> `그 외의 스코프 : 프로토타입 스코프, 요청 스코프, 세션 스코프 `<br>
>> 	1 . 프로토타입(prototype) 스코프 : 컨테이너에 빈을 요청할 때마다 매번 새로운 오브젝트가 생성된다.<br>
>> 	2. 요청(request) 스코프 : 웹을 통해 새로운 HTTP 요청이 생길 때마다 생성된다.<br>
>> 	3. 세션(session) 스코프 : 웹의 세션과 스코프가 유사하다.<br>


# 7. 의존관계 주입 (DI)
## 제어의 역전과 의존관계 주입
스프링 IoC 기능의 대표적인 동작원리는 주로 **의존관계 주입(Dependency Injection)** 이라고 불린다.<br> 물론 스프링이 컨테이너이고 프레임워크이니 기본적인 동작원리가 모두 IoC 방식이라고 할 수 있지만, 스프링이 여타 프레임워크와 차별화돼서 제공해주는 기능은 의존관계 주입이라는 용어를 사용할 때 분명하게 드러난다.<br> 그래서 <u>지금은 스프링이 DI 컨테이너라고 더 많이 불리고 있다.</u>

## 런타임 의존관계 설정
### 의존관계에 대해
---
A가 B를 의존한다고 가정해보자.<u> **의존한다는 건 의존대상, 즉 B가 변하면 그것이 A에 영향을 미친다는 뜻이다.**</u> B의 기능이 추가되거나 변경되거나, 형식이 바뀌거나 하면 그 영향이 A로 전달된다는 뜻이다.

### UserDao의 의존관계
---
![image](https://github.com/user-attachments/assets/a0e39ec4-d602-4946-ad06-457cb931e3ed)

**UserDao는 ConnectionMaker 인터페이스에만 의존하고 있다. 따라서 ConnectionMaker 인터페이스가 변한다면 그 영향을 UserDao가 직접적으로 받게 된다.** 하지만 ConnectionMaker 인터페이스를 구현한 클래스는 변화가 생겨도 UserDao에 영향을 주지 않는다. 이렇게 인터페이스에 대해서만 의존관계를 만들어두면 인터페이스 구현 클래스와의 관계는 느슨해지면서 변화에 영향을 덜 받는 상태인 결합도가 낮은 상태가 된다.<br>

> **<mark>의존관계 주입을 충족하기 위한  3가지 조건</mark>**<br>
>> 1.  클래스 모델이나 코드에는 런타임 시점의 의존관계가 드러나지 않는다. 그러기 위해서는 인터페이스에만 의존하고 있어야 한다.<br>
>> 2. -런타임 시점의 의존관계는 컨테이너나 팩토리 같은 제3의 존재가 결정한다.<br>
>> 3. 의존관계는 사용할 오브젝트에 대한 레퍼런스를 외부에서 제공(주입)해줌으로써 만들어진다.<br><br>

<mark>**의존관계 주입의 핵심은 설계 시점에는 알지 못했던 두 오브젝트의 관계를 맺도록 도와주는 제3의 존재가 있다는 것이다. DI에서 말하는 제3의 존재는 바로 관계설정 책임을 가진 코드를 분리해서 만들어진 오브젝트라고 볼 수 있다.**</mark><br><br>

`UserDao의 의존관계 주입_ 관계설정 책임을 분리하기 전 생성자`
```java
public UserDao() {
	this.connectionMaker = new DConnectionMaker();
}
```
- `문제`<br>
   인터페이스를 사이에 두고 의존관계를 느슨하게 만들긴 했지만, UserDao가 설계 시점에서 사용할 구체적인 클래스를 알고 있다.<br>
	- 모델링 때의 의존관계, 즉 ConnectionMaker 인터페이스의 관계뿐 아니라 런타임 의존관계인 DConnectioMaker 오브젝트를 사용하겠다는 것까지 UserDao가 결정하고 관리하고 있는 셈이다.<br>
- `해결`<br>
   IoC 방식을 써서 UserDao로부터 제3의 존재에 런타임 의존관계 결정 권한을 위임한다. 그래서 만들어진 것이 DaoFactory였다.<br>
- `DaoFactory`<br>
	 두 오브젝트 사이의 런타임 의존관계를 설정해주는 의존관계 주입 작업을 도와주는 존재이자 IoC 방식으로 오브젝트의 생성과 초기화, 제공 등의 작업을 수행하는 컨테이너이다. (즉, 의존관계 주입을 담당하는 컨테이너로 DI 컨테이너)<br>

`의존관계 주입을 위한 코드`
```java
private static ConnectionMaker connectionMaker;

  public UserDao(ConnectionMaker connectionMaker) {
      this.connectionMaker = connectionMaker;
  }
```

두 개의 오브젝트 간에 런타임 의존관계가 만들어졌다. <br>
**DI 컨테이너에 의해 런타임 시에 의존 오브젝트를 사용할 수 있도록 그 레페런스를 전달받는 과정이 마치 메서드(생성자)를 통해 DI 컨테이너가 UserDao에게 주입해주는 것과 같다**고 해서 이를 의존관계 주입이라고 부른다.<br>

## 의존관계 검색과 주입

>**<mark>의존관계 검색</mark>**<br>
>> 외부로부터의 주입이 아니라, 스스로 검색을 이용한다.<br>
>> 자신이 필요로 하는 의존 오브젝트를 능동적으로 찾는다.<br>
>> 런타임 시 의존관계를 맺을 오브젝트를 결정하는 것과 오브젝트의 생성 작업은 일부 컨테이너에게 IoC 로 맡기지만, 이를 가져올 때 **스스로 컨테이너에게 요청하는 방법을 사용한다.** <br>

`의존관계 검색을 이용하는 UserDao 생성자`
```java
public class UserDao {
    ConnectionMaker connectionMaker;

    // DL (Dependency Lookup) 를 이용한 방법
    public UserDao() {
        ApplicationContext applicationContext
                = new AnnotationConfigApplicationContext(DaoFactory.class);

        this.connectionMaker = applicationContext.getBean(ConnectionMaker.class);
    }
}
```

스프링 빈이 아닌 오브젝트에서 스프링 빈을 가져와야 할 때 의존관계 검색이 유용하다.<br>

## 의존관계 주입의 주요 방식
- 생성자 주입 : 클래스의 생성자를 통해 의존 객체를 전달하는 방식<br>
- setter 주입 : Setter 메소드를 이용하여 의존 객체를 전달하는 방식<br>
- 메소드 주입 : 특정 메소드를 통해 의존 객체를 전달하는 방식<br>

# 8. XML 을 이용한 설정
스프링은 DaoFactory와 같은 자바 클래스를 이용하는 것 외에도, 다양한 방법을 통해 DI 의존관계 설정정보를 만들 수 있다. 가장 **대표적인 것이 바로 XML**이다.

>**<mark> XML 의 장점</mark>**
>> 1. 단순한 텍스트 파일이라 다루기 쉽다.
>> 2. 쉽게 이해할 수 있으면 컴파일과 같은 별도의 빌드 작업이 없다.
>> 3. 환경이 달라져서 오브젝트의 관계가 바뀌는 경우에도 빠르게 변경사항을 반영할 수 있다.
>> 4. 스키마나 DTD를 이용해 정해진 포맷을 따라 작성됐는지 손쉽게 확인할 수 있다.

## XML 설정
> **`@Configuration`을 `<beans>`, `@Bean`을 `<bean>`에 대응해서 생각하면 이해하기 쉬울 것이다.**


|        | 자바 코드 설정정보              | XML 설정정보                   |
| ------ | ----------------------- | -------------------------- |
| 빈 설정파일 | @Configuration          | `<beans>`                   |
| 빈의 이름  | @Bean methodName()      | <bean id= "methodName"     |
| 빈의 클래스 | return new BeanClass(); | class="a.b.c...BeanClass"> |


DaoFactory  의 @Bean 메소드에 담긴 정보를 1:1 로 XML의 태그와 애트리뷰트로 전환해주기만 하면 된다.<br>
클래스 애트리뷰트에 넣을 클래스 이름은 패키지까지 모두 포함해야 한다.<br>
<mark> ※ 단, `<bean>` 태그의 class 애트리뷰트에 지정하는 것은 자바 메소드에서 오브젝트를 만들 때 사용하는 클래스 이름이다! </mark><br><br>
> **하나의 @Bean 메서드를 통해 얻을 수 있는 빈의 DI 정보**<br>
>> 빈의 이름, 빈의 클래스, 빈의 의존 오브젝트<br>

### UserDao() 전환
---
userDao 메서드를 XML 로 변환해보자.<br>
XML 에서는 `<property>` 태그를 사용해 의존 오브젝트와의 관계를 정의한다.<br>
`<property>` 태그는 name 과 ref 라는 두 개의 애트리뷰트를 갖는다.<br>
name은 DI 에 사용할 수정자 메서드의 프로퍼티 이름이다.<br>
ref 는 수정자 메소드를 통해 주입해줄 오브젝트의 빈 이름이다.<br>
![image](https://github.com/user-attachments/assets/443f7189-fef8-4834-a145-1df3e56656a8)

`UserDao 빈 설정`
```XML
<bean id="userDao" class="springbook.dao.UserDao">
	<property name="connetionMaker" ref="connectionMaker" />
</bean>
```
## XML의 의존관계 주입 정보
`<beans>`로 전환한 두 개의 `<bean>`태그를 감싸주면 DaoFactory로부터 XML로의 전환 작업이 끝난다.<br>
`완성된 XML 설정 정보`
```XML
<beans>
    <bean id="connectionMaker" class="springbook.user.dao.DConnectionMaker" />
    <bean id="userDao" class="springbook.user.dao.UserDao">
        <property name="connetionMaker" ref="connectionMaker" />
    </bean>
</beans>
```

`자바 설정 코드`
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springbook.user.dao.DConnectionMaker;
import springbook.user.dao.UserDao;

@Configuration
public class DaoFactory {

    @Bean
    public DConnectionMaker connectionMaker() {
        return new DConnectionMaker(); // DConnectionMaker 객체를 생성하여 반환
    }

    @Bean
    public UserDao userDao() {
        UserDao userDao = new UserDao();
        userDao.setConnectionMaker(connectionMaker()); // connectionMaker 주입
        return userDao;
    }
}
```

>**<mark>같은 인터페이스를 구현한 의존 오브젝트를 여러 개 정의할 경우</mark>**  
>> 같은 인터페이스를 구현한 의존 오브젝트를 여러 개 정의해두고 그 중에서 원하는 걸 골라서 DI 하는 경우도 있다. <br>
>> 이 때는 각 빈의 이름을 독립적으로 만들어두고 ref 애트리뷰트를 이용해 DI 받을 빈을 지정해주면 된다.

```XML
<beans>
    <bean id="localDBConnectionMaker" class="..localDBConnectionMaker" />
    <bean id="testDBConnectionMaker" class="..testDBConnectionMaker" />
    <bean id="productionDBConnectionMaker" class="..productionDBConnectionMaker" />
 
    <bean id="userDao" class="springbook.user.dao.UserDao">
        <property name="connetionMaker" ref="localDBConnectionMaker" />
    </bean>
</beans>
```

## 프로퍼티 값의 주입
### 값 주입
---
스프링의 빈으로 등록될 클래스에 수정자 메서드가 정의되어 있다면 `<property>`를 사용해 정보를 주입한다.<br> 하지만 다른 빈 오브젝트의 레퍼런스(`ref`)가 아니라 단순 값(`value`)을 주입해주는 것이기 때문에 **`value` 애트리뷰트를 사용한다.** <br>

`코드를 통한 DB 연결정보 주입`
```java
dataSource.setDriverClass(com.mysql.jdbc.Driver.class);
dataSource.setUrl("jdbc://mysql://localhost/springbook");
dataSource.setUsername("spring");
dataSource.setPassword("book");
```

`XML을 통한 DB 연결정보 설정`
```XML
<property name="driverClass" value="com.mysql.cj.jdbc.Driver"/>
<property name="url" value="jdbc:mysql://localhost/springbook"/>
<property name="username" value="spring"/>
<property name="password" value="book"/>
```

### Value 값의 자동 변환
 **스프링이 프로퍼티의 값을, 수정자 메서드의 파라미터 타입을 참고로 해서 적절한 형태로 변환해준다.** <br>
 
`DataSource를 적용 완료한 applicationContext.xml`
```XML
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
    
    <bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
        <property name="driverClass" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost/springbook"/>
        <property name="username" value="spring"/>
        <property name="password" value="book"/>
    </bean>

    <bean id="userDao" class="springbook.user.dao.UserDao">
        <property name="dataSource" ref="dataSource" />
    </bean>
</beans>
```

# 9. 느낀 점
양이 방대해서 다시 한 번 복습해야 될거 같다는 생각이 든다.<br>
초난감 UserDao를 어떻게 리펙토링 하는지부터 시작해 여러 디자인 패턴과 객체지향 원칙을 적용하면서 코드를 개선하는 과정에서 더 깔끔하고 유연한 코드로 변하는 걸 보면서 흥미로웠다.<br>
조금은 객체지향 설계에 대해 이해한 거 같기도 하다.<br>
근데 양이 좀 많다 보니 다시 한 번 복습을 통해서 이해를 해봐야 될거 같다.<br>

무엇보다 제어의 역전과 의존성 주입의 개념은 언제 봐도 어려운거 같다.<br>
내가 이해한 바로는
제어의 역전 → 개발자가 흐름을 제어하는 것이 아닌 프로그램 자체에서 흐름 제어를 하는 것<br>
의존성 주입 → 객체가 자신의 역할을 모두 다 해내는 것이 아니라 외부에 의존한다는 느낌 (주입 받아야 쓰니까!)<br>

마지막에 XML 을 이용한 설정은 거의 흐린 눈 하고 봤는데,,<br>
처음에 XML 의 장점 어쩌고 하면서 좋은 말 나오길래 의심을 했어야 했다.<br>
개인적으로 이해가 잘 안 됐다...(하하..)<br>
다시 공부해야 될거 같았다.<br>

# 10. 면접 질문
애플리케이션 컨텍스트의 동작 방식에 대해 설명해주세요.<br>
![image](https://github.com/user-attachments/assets/758f8e0e-dc38-403b-bedd-9e4476bff78d)

참고해서 말해주세요.
<details>
	<summary>답안</summary>
	애플리케이션 컨텍스트는 @Configuration 이 붙은 클래스를 설정정보로 등록해두고, @Bean 이 붙은 메서드의 이름을 가져와 빈 목록을 만들어둔다. 클라이언트가 이 애플리케이션 컨텍스트의 `getBean()` 메서드를 호출하면 자신의 빈 목록에서 요청한 이름이 있는지 찾고, 있다면 빈을 생성하는 메서드를 호출해서 오브젝트를 생성시킨 후 클라이언트에 돌려준다.
</details>

