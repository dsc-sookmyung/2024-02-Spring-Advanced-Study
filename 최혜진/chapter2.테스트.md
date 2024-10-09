# Chapter2. 테스트

 > chapter2의 학습 목표
 >> 1. 테스트란 무엇인가
 >> 2. 테스트의 가치와 장점
 >> 3. 테스트의 활용 전략
 >> 4. 테스트와 스프링의 관계

### 웹을 통한 DAO 테스트 방법의 문제점
```plain text
**웹 프로그램에서 사용하는  DAO 테스트 방법**
1. DAO, 서비스 계층, MVC 프레젠테이션 계층까지 포함한 모든 입출력 기능을 대충이라도 다 만든다.
2. 만들어진 테스트용 웹 어플리케이션을 서버에 배치한다.
3. 웹 화면을 띄워 폼을 연다.
4. 값을 입력하고 버튼을 눌러 등록해본다. (폼의 값을 받아서 파싱한 뒤 User 오브젝트를 만들고 UserDao 호출은 이미 준비가 된 상태여야 함)
5. 테스트를 했을 때, 에러가 없다면 이번엔 검색 폼이나 파라미터를 지정할 수 있는 URL 을 사용해 방금 입력한 데이터를 가져올 수 있는지 테스트한다. (이미 UserDao 결과를 출력할 기능은 만들어져 있어야 함)
```
이런 방법의 테스트는 단점이 너무 많다.
가장 큰 단점은 DAO 뿐만 아니라 다른 서비스 클래스, 컨트롤러, JSP 뷰 등 <u>모든 레이어의 기능을 다 만들고 나서야 테스트가 가능하다는 점</u>이다.

또한 테스트의 범위가 너무 방대해 만약 에러가 뜨더라도, 어디 부분을 어떻게 수정할지 찾기가 힘들다는 점이다.

<mark>그렇다면 우리는 이제 테스트를 어떻게 만들면 이런 문제를 피할 수 있고, 효율적인 테스트를 할 수 있는지 알아야 한다.</mark>

## 효율적인 테스트를 위한 방법
### 1. 작은 단위의 테스트
테스트는 가능하면 작은 단위로 쪼개서 집중할 수 있어야 한다.

> `단위 테스트 (Unit Test)`
>> 작은 단위의 코드에 대해 테스트를 수행한 것
>`단위 테스트의 목적`
>> 개발자가 설계하고 만든 코드가 원래 의도대로 동작하는지를 개발자 스스로 빨리 확인받기 위해서이다.


<mark> Q. 우리가 앞장에서 (Chapter1) 본 UserDaoTest 는 단위 테스트가 맞나? </mark>
DB의 상태가 매번 달라지고, 테스트를 위해 DB를 특정 상태로 만들어줄 수 없다면 그때는 UserDaoTest 가 단위 테스트로서 가치가 없다.
그런 차원에서 **통제할 수 없는 외부의 리소스 테스트는 단위 테스트가 아니라고 본다.**

<mark>Q. 각 단위별로 테스트를 먼저 모두 진행하고 긴 테스트를 시작한다면 장점은 뭘까?</mark>
긴 테스트로 합쳐지는 과정에서 예외나 테스트 실패가 발생할 수 있겠지만, 이미 각 단위별로 검증을 마치고 오류를 잡았으므로 훨씬 더 빠르게 점검해볼 수 있을 것이다.

### 2. 자동수행 테스트 코드
테스트는 자동으로 수행되도록 코드로 만들어지는 것이 중요하다.

<mark>Q. 자동으로 수행되는 테스트의 장점은?</mark>
자주 반복할 수 있다. 번거로운 작업이 없고 테스트를 빠르게 실행할 수 있기 때문에 언제든 코드를 수정하고 나서 테스트를 해볼 수 있다.

### 3. 지속적인 개선과 점진적인 개발을 위한 테스트
테스트를 이용하면 새로운 기능도 기대한 대로 동작하는지 확인할 수 있을 뿐 아니라, 기존에 만들어뒀던 기능들이 새로운 기능을 추가하느라 수정한 코드에 영향을 받지 않고 잘 돌아가는지 확인할 수 있다.


## UserDaoTest의 문제점
`기존의 UserDaoTest 코드`
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
        
    }  
  
}
```

**기존의 테스트 코드의 문제점**
1. 수동 확인 작업의 번거로움
	테스트 수행은 코드에 의해 자동으로 진행되지만 테스트의 결과를 확인하는 일은 사람의 책임이므로 완전히 자동으로 테스트 되는 방법이 아니다.
2. 실행 작업의 번거로움
	

## UserDaoTest 개선
### 테스트 검증의 자동화
---
모든 테스트는 성공과 실패로 나뉜다.
테스트의 실패는 테스트가 진행되는 동안에 <u>에러가 발생해서 실패하는 경우</u>와 테스트 작업 중에 에러가 발생하진 않았지만 <u>기대한 것과 다르게 나오는 경우</u>가 있다.
전자를 테스트 에러, 후자를 테스트 실패로 이해하면 된다.

위에서 언급된 기존의 테스트 코드의 문제점 중 `수동 확인 작업의 번거로움`  에 대해 언급을 했었다. 이걸 개선해보자.

`수정 전 코드`
```java
	User result = dao.get("2");  
	System.out.println("UserId:::: " + result.getId());  
	System.out.println("UserName:::: " + result.getName());  
        
```
`수정 후 코드`
```java
	if (!user.getName().equals(result.getName())) {  
	    System.out.println("테스트 실패 (name)");  
	}else if (!user.getPassword().equals(result.getPassword())) {  
	    System.out.println("테스트 실패 (password)");  
	}else {  
	    System.out.println("테스트 성공");  
	}
```

이렇게 수정을 한 코드를 통해서 UserDao 의 두 가지 기능이 정상적으로 동작하는지를 손쉽게 확인할 수 있게 해준다.


### 테스트의 효율적인 수행과 결과 관리
자바 테스팅 프레임워크로 불리는 **JUnit** 은 자바로 단위 테스트를 만들 때 유용하다.
#### JUnit 테스트로 전환
----
> **JUnit 프레임워크가 요구하는 두 가지 조건**
>> 1. 메소드가 public 으로 선언돼야 한다.
>> 2. 메소드에 @Test 라는 애노테이션을 붙여줘야 한다.

#### 검증 코드로 전환
---
테스트 결과를 검증하는 if/else 문을 JUnit 이 제공하는 방법을 이용해서 전환해보자.
- assertThat() 메서드는 첫 번째  파라미터의 값을 뒤에 나오는 매처 (matcher) 라 불리는 조건으로 비교해서 일치하면 다음으로 넘어가고, 아니면 테스트가 실패하도록 만들어준다.
- is() 는 매처의 일종으로 equals()로 비교해주는 기능이다.

`if/else 문장`
```java
if (!user.getName().equals(result.getName())) {  
	    System.out.println("테스트 실패 (name)");  
	}else if (!user.getPassword().equals(result.getPassword())) {  
	    System.out.println("테스트 실패 (password)");  
	}else {  
	    System.out.println("테스트 성공");  
	}
```
`JUnit 제공하는 방법`
```java
import static org.hamcrest.CoreMatchers.is;  
import static org.hamcrest.MatcherAssert.assertThat;

assertEquals("2", result.getId());  
assertThat(result.getName(), is("최혜진"));
```

## 개발자를 위한 테스팅 프레임워크 JUnit
 ### 테스트 결과의 일관성
 ---
 우리가 앞 챕터에서 users 테이블을 만들 때, auto increment 로 하지 않아서 계속 테스트 할 때마다 직접 지워주는 번거로움이 있었다.

deleteAll() 과 getCount() 를 추가해서 일관성 있는 결과를 보장하도록 해보자.
우선 UserDao 에 새로운 기능을 추가해줘야 한다.

`deleteAll() 메서드`: users 테이블의 모든 행을 행을 지운다.
`getCount() 메서드`:  users 테이블의 레코드 개수를 돌려준다.

```java
package org.example.chapter1.dao;  
  
import java.sql.Connection;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import org.example.chapter1.domain.User;  
  
public class UserDaoConnectionMaker {  
  
    private ConnectionMaker connectionMaker; 
    
    public UserDaoConnectionMaker(ConnectionMaker connectionMaker) {  
        this.connectionMaker = connectionMaker;  
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
    //챕터 2 테스트에서 추가된 메소드  
    public void deleteAll() throws SQLException, ClassNotFoundException {  
        Connection c = connectionMaker.makeConnection();  
        PreparedStatement ps = c.prepareStatement("delete from users");  
  
        ps.executeUpdate();  
        ps.close();  
        c.close();  
    }  
    //챕터 2 테스트에서 추가된 메소드  
    public int getCount() throws SQLException, ClassNotFoundException {  
        Connection c = connectionMaker.makeConnection();  
        PreparedStatement ps = c.prepareStatement("select count(*) from users");  
  
        ResultSet rs = ps.executeQuery();  
        rs.next();  
        int count = rs.getInt(1);  
        rs.close();  
        ps.close();  
        c.close();  
        return count;  
    }  
}
```

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
  
import static org.hamcrest.CoreMatchers.is;  
import static org.hamcrest.MatcherAssert.assertThat;  
  
@ExtendWith(SpringExtension.class)  
@ContextConfiguration(classes = DaoFactoryContext.class)  
class UserDaoConnectionMakerTest {  
    @Autowired  
    private ApplicationContext context;  
  
    @Test  
    public void addAndGet() throws SQLException, ClassNotFoundException {  
        UserDaoConnectionMaker dao = context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  
    
        dao.deleteAll();  
        assertThat(dao.getCount(), is(0));  
  
        User user = new User();  
  
        user.setId("2");  
        user.setName("최혜진");  
        user.setPassword("1234");  
  
        dao.add(user);  
        assertThat(dao.getCount(), is(1));  
  
        User result = dao.get(user.getId());  
  
        assertEquals("2", result.getId());  
        assertThat(result.getName(), is(user.getName()));  
        assertThat(result.getPassword(), is(user.getPassword()));  
  
    }  
  
}
```

![[Pasted image 20241006201054.png]]
테스트가 성공한 걸 확인할 수 있다.

<mark>단위 테스트는 항상 일관성 있는 결과가 보장돼야 한다. </mark>

### getCount() 테스트
---
JUnit 은 하나의 클래스 안에 여러 개의 메소드가 들어가는 것을 허용한다.
*@Test 가 붙어 있고 public 접근자가 있으며 리턴 값이 void 형이고 파라미터가 없다는 조건만 지키면 된다.*

> 시나리오 
>>  먼저 User 테이블의 데이터를 다 지우고 getCount() 로 레코드 개수가 0임을 확인한다.
>>  그 후 3개의 사용자 정보를 하나씩 추가하면서 매번 getCount() 결과가 하나씩 증가하는지 확인한다.

```java
//User.java
//생성자 추가
public User() {  
}  
  
public User(String id, String name, String password) {  
    this.id = id;  
    this.name = name;  
    this.password = password;  
}
```

```java
@Test  
public void count() throws SQLException, ClassNotFoundException {  
    UserDaoConnectionMaker dao = context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  
    User user1= new User("1", "최혜진", "1234");  
    User user2= new User("2", "최따루", "38287");  
    User user3= new User("3", "김성호", "123474");  
  
    dao.deleteAll();  
    assertThat(dao.getCount(), is(0));  
  
    dao.add(user1);  
    assertThat(dao.getCount(), is(1));  
  
    dao.add(user2);  
    assertThat(dao.getCount(), is(2));  
  
    dao.add(user3);  
    assertThat(dao.getCount(), is(3));  
}
```

![[Pasted image 20241006202151.png]]

<mark>※ 주의할 점</mark>
두 개의 테스트 (addAndGet() 메소드와 count() 메서드) 의 실행 순서를 보장해주지 않는다.
모든 테스트는 실행 순서와 상관없이 독릭적으로 항상 동일한 결과를 낼 수 있도록 해야 한다.

### addAndGet() 테스트 보완
---
id를 조건으로 해서 사용자를 검색하는 기능을 가진 get() 에 대한 테스트는 조금 부족하다.
get() 이 파라미터로 주어진 id 에 해당하는 사용자를 가져온 것인지 그냥 아무거나 가져온 것인지 테스트를 통해 검증하지 못 했다.

그렇기 때문에 get() 메서드에 대한 테스트 기능을 보완할 필요가 있다.

> 시나리오
>> User 를 하나 더 추가해서 두 개의 User 를 add() 하고, 각 User 의 id 를 파라미터로 전달해서 get() 을 실행하도록 하자.

`get() 테스트 기능을 보완한 addAndGet() 테스트트`
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
  
import static org.hamcrest.CoreMatchers.is;  
import static org.hamcrest.MatcherAssert.assertThat;  
  
@ExtendWith(SpringExtension.class)  
@ContextConfiguration(classes = DaoFactoryContext.class)  
class UserDaoConnectionMakerTest {  
    @Autowired  
    private ApplicationContext context;  
  
    @Test  
    public void addAndGet() throws SQLException, ClassNotFoundException {  
        UserDaoConnectionMaker dao = context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  

        User user1= new User("1", "최혜진", "1234");  
        User user2= new User("2", "최따루", "38287");  
  
        dao.deleteAll();  
        assertThat(dao.getCount(), is(0));  
  
        dao.add(user1);  
        dao.add(user2);  
        assertThat(dao.getCount(), is(2));  
		  // 첫 번째 User 의 id 로 get()을 실행하면 첫 번째 User의 값을 가진 오브젝트를 돌려주는지 확인
        User userget1 = dao.get(user1.getId());  
        assertThat(userget1.getName(), is(user1.getName()));  
        assertThat(userget1.getPassword(), is(user1.getPassword()));  
          
        User userget2 = dao.get(user2.getId());  
        assertThat(userget2.getName(), is(user2.getName()));  
        assertThat(userget2.getPassword(), is(user2.getPassword()));  
  
    }  
    
}
```

### get() 예외조건에 대한 테스트
---
get() 메소드에 전달된 id 값에 해당하는 사용자 정보가 없다면 어떻게 될까?
이럴 땐 어떤 결과가 나오면 좋을까?
1. null 과 같은 특별한 값을 리턴하는 방법
2. id 에 해당하는 정보를 찾을 수 없다고 예외 처리를 하는 방법

UserDao의 get() 메서드에서 쿼리를 실행해 결과를 가져올 때 아무것도 없으면 예외를 던지도록 하면 된다.

문제는 예외 발생 여부는 메소드를 실행해서 리턴값을 비교하는 방법으로 확인할 수 없다는 점이다.
즉, assertThat() 메소드로는 검증이 불가능하다.
하지만!!! JUnit 은 에외조건 테스트를 위한 특별한 방법을 제공해준다.

모든 데이터를 지우고, 존재하지 않는 id 로 get() 을 호출한다.
이때 EmptyResultDataAccessException이 던져지면 성공이고 아니면 실패다.
```java
//JUnit5
@Test()  
public void getUserFailure() throws SQLException, ClassNotFoundException {  
    UserDaoConnectionMaker dao = context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  
    dao.deleteAll();  
    assertThat(dao.getCount(), is(0));  
  
    assertThrows(EmptyResultDataAccessException.class, () -> {  
        dao.get("unknown_id");  
    });  
}
```

@Test 애노테이션의 expected 엘리먼트가 가장 중요하다.
expected 는 테스트 메소드 실행 중 발생하리라 기대하는 예외 클래스를 넣어주면 된다.

**expected 를 추가해놓으면 정상적으로 테스트 메소드를 마치면 실패하고, expected에서 지정한 예외가 던져지면 성공이다.**
예외가 반드시 발생해야 하는 경우를 테스트 하고 싶을 대 쓰면 유용하게 쓸 수 있다.

> 현재 이 테스트를 실행하면 테스트는 당연히 실패한다.
> get() 메소드에서 쿼리 결과의 첫 번째 로우를 가져오게 하는 rs.next() 를 실행할 때 가져올 로우가 없다는 SQLException 이 발생할 것이다.


### 테스트를 성공시키기 위한 코드 수정
```java
public User get(String id) throws SQLException, ClassNotFoundException {  
    Connection c = connectionMaker.makeConnection();  
    PreparedStatement ps = c.prepareStatement(  
            "select * from users where id = ?");  
  
    ps.setString(1, id);  
    ps.execute();  
  
    ResultSet rs = ps.executeQuery();  
    User user=null;  
    if(rs.next()){  
        user = new User();  
        user.setId(rs.getString("id"));  
        user.setName(rs.getString("name"));  
        user.setPassword(rs.getString("password"));  
    }  
  
    rs.close();  
    ps.close();  
    c.close();  
  
    if (user == null) throw new EmptyResultDataAccessException(1); //예외 처리  
  
    return user;  
}
```

세 개의 테스트가 모두 성공했다.
이는 getUserFailure() 테스트뿐만 아니라 기존에 get() 으로 사용자 정보를 가져오는 addAndGet() 테스트도 성공한 것이다. 따라서 get() 으로 정상적인 결과를 가져오는 경우와 예외적인 경우에 대해서도 모두 테스트를 성공한 것이다.

>항상 네거티브 테스트를 먼저 만들어보기
>포괄적인 테스트를 중요하게 생각하기!!!


 ## 테스트가 이끄는 개발
 
 ### 기능설계를 위한 테스트
 ---
 추가하고 싶은 기능을 코드로 표현하려고 하는 것
 getUserFailure() 테스트에는 만들고 싶은 기능에 대한 조건과 행위, 결과에 대한 내용이 잘 표현돼 있다.
 `getUserFailure() 테스트 코드에 나타난 기능`

|     | 단계         | 내용                      | 코드                                                                                                  |
| --- | ---------- | ----------------------- | --------------------------------------------------------------------------------------------------- |
| 조건  | 어떤 조건을 가지고 | 가져올 사용자 정보가 존재하지 않는 경우  | dao.deleteAll();<br>assertThat(dao.getCount(), is(0));                                              |
| 행위  | 무엇을 할 때    | 존재하지 않는 id로 get()을 실행하면 | get("unknown_id");                                                                                  |
| 결과  | 어떤 결과가 나온다 | 특별한 예외를 던진다.            | assertThrows(EmptyResultDataAccessException.class, () -> {  <br>    dao.get("unknown_id");  <br>}); |


### 테스트 주도 개발 (TDD or TFD)
---
만들고자 하는 기능의 내용을 담고 있으면서 만들어진 코드를 검증도 해줄 수 있도록 테스트 코드를 먼저 만들고, 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방법

테스트를 코드보다 먼저 작성한다고 해서 <u>테스트 우선 개발</u>이라고도 한다.

> TDD의 기본 원칙
>> 실패한 테스트를 성공시키기 위한 목적이 아닌 코드는 만들지 않는다.

TDD는 아예 테스트를 먼저 만들고 그 테스트가 성공하도록 하는 코드만 만드는 식으로 진행되기 때문에 테스트를 빼먹지 않고 꼼꼼하게 만들어낼 수 있다.


### 테스트 코드 개선
---
중복된 코드를 개선할 필요가 있다.
리펙토링을 해야 하는데 JUnit 프레임워크는 테스트 메소드를 실행할 때 부가적으로 해주는 작업이 몇가지 있다.
그 중에 ***테스트를 실행할 때마다 반복되는 준비 작업을 별도의 메소드에 넣게 해주고, 이를 매번 테스트 메소드를 실행하기 전에 먼저 실행시켜주는 기능이다.***

#### @BeforeEach
---
중복됐던 코드를 넣을 setUp() 이라는 이름의 메소드를 만들고 테스트 메소드에서 제거한 코드를 넣는다. 여기서 dao 변수가 setUp() 메서드의 로컬 변수로 되어 있다는 점이다. 그래서 이번에 로컬 변수인 dao를 테스트 메소드에서 접근할 수 있도록 인스턴스 변수로 변경한다.
```java
@BeforeEach  
public void setUp() {  
    this.dao = context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  
}
```

`전체 코드`
```java
package org.example.chapter1.dao;  
  
import static org.junit.jupiter.api.Assertions.*;  
import static org.hamcrest.CoreMatchers.is;  
import static org.hamcrest.MatcherAssert.assertThat;  
  
import java.sql.SQLException;  
  
import org.example.chapter1.domain.User;  
import org.junit.jupiter.api.BeforeEach;  
import org.junit.jupiter.api.Test;  
import org.junit.jupiter.api.extension.ExtendWith;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.ApplicationContext;  
import org.springframework.dao.EmptyResultDataAccessException;  
import org.springframework.test.context.ContextConfiguration;  
import org.springframework.test.context.junit.jupiter.SpringExtension;  
  
@ExtendWith(SpringExtension.class)  
@ContextConfiguration(classes = DaoFactoryContext.class)  
class UserDaoConnectionMakerTest {  
  
    @Autowired  
    private ApplicationContext context;  
  
    private UserDaoConnectionMaker dao;  
  
    @BeforeEach  
    public void setUp() {  
        this.dao = context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  
    }  
  
    @Test  
    public void addAndGet() throws SQLException, ClassNotFoundException {  
        User user1 = new User("1", "최혜진", "1234");  
        User user2 = new User("2", "최따루", "38287");  
  
        dao.deleteAll();  
        assertThat(dao.getCount(), is(0));  
  
        dao.add(user1);  
        dao.add(user2);  
        assertThat(dao.getCount(), is(2));  
  
        User userget1 = dao.get(user1.getId());  
        assertThat(userget1.getName(), is(user1.getName()));  
        assertThat(userget1.getPassword(), is(user1.getPassword()));  
  
        User userget2 = dao.get(user2.getId());  
        assertThat(userget2.getName(), is(user2.getName()));  
        assertThat(userget2.getPassword(), is(user2.getPassword()));  
    }  
  
    @Test  
    public void count() throws SQLException, ClassNotFoundException {  
        User user1 = new User("1", "최혜진", "1234");  
        User user2 = new User("2", "최따루", "38287");  
        User user3 = new User("3", "김성호", "123474");  
  
        dao.deleteAll();  
        assertThat(dao.getCount(), is(0));  
  
        dao.add(user1);  
        assertThat(dao.getCount(), is(1));  
  
        dao.add(user2);  
        assertThat(dao.getCount(), is(2));  
  
        dao.add(user3);  
        assertThat(dao.getCount(), is(3));  
    }  
  
    @Test  
    public void getUserFailure() throws SQLException, ClassNotFoundException {  
        dao.deleteAll();  
        assertThat(dao.getCount(), is(0));  
  
        assertThrows(EmptyResultDataAccessException.class, () -> {  
            dao.get("unknown_id");  
        });  
    }  
}
```

> **JUnit 이 하나의 테스트 클래스를 가져와 테스트를 수행하는 방식**
>> 1. 테스트 클래스에서 @Test가 붙은 public 이고 void 형이며 파라미터가 없는 테스트 메소드를 모두 찾는다.
>> 2. 테스트 클래스의 오브젝트를 하나 만든다.
>> 3. @BeforeEach가 붙은 메소드가 있으면 실행한다.
>> 4. @Test가 붙은 메소드를 하나 호출하고 테스트 결과를 저장해둔다.
>> 5. @AfterEach가 붙은 메소드가 있으면 실행한다.
>> 6. 나머지 테스트 메소드에 대해 2~5번 과정을 반복한다.
>> 7. 모든 테스트 결과를 종합해 돌려준다.

실제로 더 복잡하지만, 간단히 정리하면 이런 식으로 수행된다.
각 테스트 메소드를 실행할 때마다 테스트 클래스의 오브젝트를 새로 만든다.
한 번 만들어진 테스트 클래스의 오브젝트는 하나의 테스트 메소르르 사용하고 나면 버려진다.

<mark>Q. 테스트 클래스가 @Test 테스트 메서드를 두 개 갖고 있다면, 테스트가 실행되는 중에 JUnit 은 몇 개의 클래스 오브젝트를 만들까?</mark>
2개가 만들어진다.

`JUnit 의 테스트 메소드 실행 방법`
![[Pasted image 20241006230410.png]]

<mark>Q. 왜 테스트 메소드를 실행할 때마다 새로운 오브젝트를 만드는 것일까?</mark>
각 테스트가 서로 영향을 주지 않고 독립적으로 실행됨을 확실히 보장해주기 위해 매번 새로운 오브젝트를 만드는 것이다.

<mark>Q. 테스트 메서드의 일부에서만 공통적으로 사용되는 코드가 있다면 어떻게 해야 될까?</mark>
@BeforeEach 을 사용하기 보다, 일반적인 메소드 추출 방법을 써서 메소드로 분리하고 테스트 메소드에서 호출해서 사용하도록 하는 편이 낫다. 아니면 아예 공통적인 특징을 지닌 테스트 메소드를 모아서 별도의 테스트 클래스로 만드는 방법도 있다.

#### 픽스처
---
테스트를 수행하는데 필요한 정보나 오브젝트

일반적으로 픽스처는 여러 테스트에서 반복적으로 사용돼서 @BeforeEach 메서드를 이용해 생성하면 편하다.

UserDaoTest  에서는 dao 가 대표적인 픽스처이다.
테스트 중 add() 메서드에 전달하는 User 오브젝트들도 픽스처로 볼 수 있다.

```java
private UserDaoConnectionMaker dao;  
private User user1;  
private User user2;  
private User user3;

@BeforeEach  
public void setUp() {  
    this.dao = context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  
    this.user1 = new User("1", "최혜진", "1234");  
    this.user2 = new User("2", "최따루", "38287");  
    this.user3 = new User("3", "김성호", "123474");  
}
```

이렇게 @BeforeEach에 픽스처로 만들어봤다.
## 스프링 테스트 적용
위에서 테스트 코드를 어느 정도 정리를 마쳤지만 조금 찜찜한 부분이 남아있다. 
바로 애플리케이션 컨텍스트 생성 방식이다.
@BeforeEach 메소드가 테스트 메소드 개수만큼 반복되기 때문에 애플리케이션 컨텍스트도 세 번 만들어진다.

<u>애플리케이션 컨텍스트는 한 번만 만들고 여러 테스트가 공유해서 사용해도 된다.</u>

문제는 JUnit 이 매번 테스트 클래스의 오브젝트를 새로 만든다는 점이다.
JUnit은 테스트 클래스 전체에 걸쳐 딱 한 번만 실행되는 @BeforeClass 스태틱 메소드를 지원한다.
→ @BeforeAll 로 현재 쓰임

이 메소드에서 애플리케이션 컨텍스트를 만들어서 스태틱 변수에 저장해두고 테스트 메소드에서 사용할 수 있다. 하지만 이보다 스프링에서 제공하는 애플리케이션 컨텍스트 테스트 지원 기능이 더 편하다.

`편리성`
`스프링 애플리케이션 컨텍스트 > @BeforeAll 스태틱 메서드`

### 테스트를 위한 애플리케이션 컨텍스트 관리
---
#### 스프링 테스트 컨텍스트 프레임워크 적용
---
`@ExtendWith` 는 JUnit 프레임워크의 테스트 실행 방법을 확장할 때 사용하는 어노테이션이다.
`@ContextConfiguration` 는 자동으로 만들어줄 애플리케이션 컨텍스트의 설정파일 위치를 지정한 것이다.

```java
@BeforeEach  
public void setUp() {  
    this.dao = context.getBean("userDaoConnectionMaker", UserDaoConnectionMaker.class);  
    this.user1 = new User("1", "최혜진", "1234");  
    this.user2 = new User("2", "최따루", "38287");  
    this.user3 = new User("3", "김성호", "123474");  
    System.out.println(this.context);   //여기 주목
    System.out.println(this);   //여기 주목
}
```

![[Pasted image 20241007221839.png]]

`출력결과`를 확인해보면, 
하나의 애플리케이션 컨텍스트가 만들어져 모든 테스트 메소드에서 사용되고 있음을 알 수 있다.
반면 UserDaoConnectionMakerTest의 오브젝트는 매번 주소 값이 다르다.
JUnit 은 테스트 메소드를 실행할 때마다 새로운 테스트 오브젝트를 만들기 때문이다.

<mark>Q. context 변수에 어떻게 애플리케이션 컨텍스트가 들어있나?</mark>
스프링의 JUnit 확장 기능은 테스트가 실행되기 전에 딱 한 번만 애플리케이션 컨텍스트를 만들어두고, 테스트 오브젝트가 만들어질 때마다 특별한 방법을 이용해 애플리케이션 컨텍스트 자신을 오브젝트의 특정 필드에 주입해주는 것이다. (일종의 DI)

#### 결론
---
하나의 테스트 클래스 내의 테스트 메소드는 같은 애플리케이션 컨텍스트를 공유해서 사용할 수 있다.


#### @Autowired
----
스프링의 DI에 사용되는 특별한 애노테이션이다.
@Autowired 가 붙은 인스턴스 변수가 있으면, 테스트 컨텍스트 프레임워크는 변수 타입과 일치하는 컨텍스트 내의 빈을 찾는다. **타입이 일치하는 빈이 있으면 인스턴스 변수에 주입**해준다.
<u>→ 타입에 의한 자동 와이어링이라고 함</u>

### DI와 테스트
---
테스트에서도 가능한 한 인터페이스를 사용해서 애플리케이션 코드와 느슨하게 연결하는 편이 좋다.
> `인터페이스를 두고 DI 를 적용해야 하는 이유`
>> 1. 소프트웨어 개발에서 절대로 바뀌지 않는 것은 없기 때문이다.
>> 2. 클래스의 구현 방식은 바뀌지 않는다고 하더라도 인터페이스를 두고 DI 를 적용하게 해두면 다른 차원의 서비스 기능을 도입할 수 있기 때문이다.
>> 3. 테스트 때문이다.

### @DirtiesContext
----
Spring 텍스트에서 컨텍스트가 변경될 수 있는 상황에서 컨텍스트를 리셋하거나 초기화하는 데 사용한다.
1. 클래스 레벨에 적용
   ```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = DaoFactoryContext.class)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)  // 모든 테스트가 끝난 후 컨텍스트를 리셋
class UserDaoConnectionMakerTest {
    // ...
}
```
클래스 내 모든 테스트가 실행된 후에 애플리케이션 컨텍스트를 리셋하고 싶을 경우, 클래스 레벨에 `@DirtiesContext`를 추가할 수 있다.

2. 메서드 레벨에 적용
   ```java
@Test  
@DirtiesContext //이 테스트가 끝난 후 애플리케이션 컨텍스트를 리셋한다.  
public void addAndGet() throws SQLException, ClassNotFoundException {  
  
    dao.deleteAll();  
    assertThat(dao.getCount(), is(0));  
  
    dao.add(user1);  
    dao.add(user2);  
    assertThat(dao.getCount(), is(2));  
  
    User userget1 = dao.get(user1.getId());  
    assertThat(userget1.getName(), is(user1.getName()));  
    assertThat(userget1.getPassword(), is(user1.getPassword()));  
  
    User userget2 = dao.get(user2.getId());  
    assertThat(userget2.getName(), is(user2.getName()));  
    assertThat(userget2.getPassword(), is(user2.getPassword()));  
}
```
특정 테스트 메서드에서 컨텍스트가 변경되거나, 이후의 다른 테스트에 영향을 주지 않도록 하고 싶을 경우, 메서드 레벨에 `@DirtiesContext`를 적용할 수 있다.
특정 테스트 메서드가 끝난 후에만 컨텍스트를 리셋하게 된다.

>`언제 사용할까?`
>> 1. 데이터베이스 상태 변경
>> 2. 빈 상태 변경
>> 3. 테스트 간 독립성 보장

`@DirtiesContext`를 적절히 적용하면 테스트 간의 격리성을 보장하고, 예상치 못한 부작용을 방지할 수 있다.

### 컨테이너 없는 DI 테스트
----
아예 스프링 컨테이너를 사용하지 않고 테스트를 만드는 것
```java
UserDao dao;  
@BeforeEach  
public void setUp() {  
    dao = new UserDao();  
    DataSource dataSource = new SingleConnectionDataSource(  
            "jdbc:mysql://localhost/hyejindb", "hyejin66", "u1234", true);  
    dao.setDataSource(dataSource);  
}
```
테스트를 위한 DataSource 를 직접 만드는 번거로움은 있지만 애플리케이션 컨텍스트를 아예 사용하지 않으니 코드가 더 단순해지고 이해하기 쉽다.

### 침투적 기술과 비침투적 기술
----
#### 침투적 기술이란?
----
기술을 적용했을 때 애플리케이션 코드에 기술 관련 API 가 등장하거나, 특정 인터페이스나 클래스를 사용하도록 강제하는 기술
애플리케이션 코드가 해당 기술에 종속되는 결과를 가져온다.

#### 비침투적 기술이란?
----
애플리케이션 로직을 담은 코드에 아무런 영향을 주지 않고 적용이 가능하다. 그렇기 때문에 종속적이지 않은 순수한 코드를 유지할 수 있다.
<u>스프링은 비침투적 기술의 대표적 예시이다. 그렇기 때문에 스프링 컨테이너 없는 DI 테스트도 가능한 것이다.</u>

### DI 를 이용한 테스트 방법 선택
---
1. 테스트를 위해 필요한 오브젝트의 생성과 초기화가 단순하다면, 스프링 컨테이너 없이 테스트 할 수 있는 방법을 우선적으로 생각하자.
2. 여러 오브젝트와 복잡한 의존관계를 갖고 있는 오브젝트를 테스트할 경우, 스프링의 설정을 이용한  DI 방식의 테스트가 편리하다.
3. 테스트 설정을 따로 만들었다고 하더라도 때로 예외적인 의존관계를 강제로 구성해서 테스트하는 경우가 있다. 이런 경우,  컨테이너에서 DI 받은 오브젝트에 다시 테스트 코드로 수동 DI해서 테스트 하는 방법이 있다. (※ 테스트 메소드나 클래스에 @DirtiesContext 애노테이션을 붙여야 함)

## 학습 테스트로 배우는 스프링

> `학습테스트란?`
>> 자신이 만들지 않은 프레임워크나 다른 개발팀에서 만들어서 제공한 라이브러리 등에 대해서도 테스트를 작성해야 한다. 
>`목적`
>> 자신이 사용할 API나 프레임워크의 기능을 테스트로 보면서 사용 방법을 익히기 위함이다.

### 학습 테스트의 장점
---
1. 다양한 조건에 따른 기능을 손쉽게 확인해볼 수 있다.
2. 학습 테스트 코드를 개발 중에 참고할 수 있다.
   아직 익숙하지 않은 기술을 사용해야 하는 개발자에게는 이렇게 미리 만들어진 다양한 기능에 대한 테스트 코드가 좋은 참고 자료가 된다.
3. 프레임워크나 제품을 업그레이드할 때 호한성 검증을 도와준다.
4. 테스트 작성에 대한 좋은 훈련이 된다.
5. 새로운 기술을 공부하는 과정이 즐거워진다.

### JUnit 테스트 오브젝트 테스트
----
JUnit 으로 만드는 JUnit 자신에 대한 테스트를 만들어보자.
`JUnit 테스트 오브젝트 생성에 대한 학습 테스트`
```java
package org.example.chapter1.dao;  
  
import static org.hamcrest.CoreMatchers.is;  
import static org.hamcrest.CoreMatchers.not;  
import static org.hamcrest.CoreMatchers.sameInstance;  
import static org.hamcrest.MatcherAssert.assertThat;  
import org.junit.jupiter.api.Test;  
  
public class JUnitTest {  
    static JUnitTest jUnitTest;  
  
    @Test  
    public void test1(){  
        assertThat(this, is(not(sameInstance(jUnitTest))));  
        jUnitTest = this;  
    }  
    @Test  
    public void test2(){  
        assertThat(this, is(not(sameInstance(jUnitTest))));  
        jUnitTest = this;  
    }  
    @Test  
    public void test3(){  
        assertThat(this, is(not(sameInstance(jUnitTest))));  
        jUnitTest = this;  
    }  
}
```
`assertThat()` 에서 몇 가지 매처를 사용했다.
`not()` 은 뒤에 나오는 결과를 부정하는 매처다.
`is()` 는  equals() 비교를 해서 같으면 성공이지만 `is(not())`  은 반대로 같지 않아야 성공이다.
`sameInstance()` 는 실제로 같은 오브젝트인지 비교한다.

이 테스트에서 조금 찜찜한 사실은 <u>직전 테스트에서 만들어진 테스트 오브젝트와만 비교</u>를 한다.
그렇기 때문에 만약 test1 과 test3의 테스트 오브젝트가 같은 경유가 있다면 그것은 검증이 안 된다.

`개선 → 세 개의 테스트 오브젝트 중 어떤 것도 중복되지 않는다는 것을 확인하도록 하는 검증 방법`

스태틱 변수로 <u>테스트 오브젝트를 저장할 수 있는 컬렉션</u>을 만들어둔다.
```java
package org.example.chapter1.dao;  
  
import static org.hamcrest.MatcherAssert.assertThat;  
import static org.hamcrest.Matchers.not;  
import static org.hamcrest.Matchers.hasItem;  
  
import java.util.HashSet;  
import java.util.Set;  
import org.junit.jupiter.api.Test;  
  
public class AdvancedJUnitTest {  
    static Set<AdvancedJUnitTest> testObjects = new HashSet<AdvancedJUnitTest>();  
  
    @Test  
    public void test1(){  
        assertThat(testObjects, not(hasItem(this)));  
        testObjects.add(this);  
    }  
    @Test  
    public void test2(){  
        assertThat(testObjects, not(hasItem(this)));  
        testObjects.add(this);  
    }  
    @Test  
    public void test3(){  
        assertThat(testObjects, not(hasItem(this)));  
        testObjects.add(this);  
    }  
  
}
```
현재 테스트 오브젝트가 컬렉션에 이미 등록돼 있는지 확인하고, 없다면 자기 자신을 추가한다.
이 방법을 사용하면 테스트가 어떤 순서로 실행되든 오브젝트 중복 여부를 확인할 수 있다.

`hasItem()`은 컬렉션의 원소인지 검사하는 매처이다.

####  위 코드를 통해 얻은 결론
---
JUnit은 매번 새로운 테스트 오브젝트를 만든다.

### 스프링 테스트 컨텍스트 테스트
----
스프링 테스트용 애플리케이션 컨텍스트는 테스트의 개수와 상관없이 한 개만 만들어진다.
이렇게 만들어진 컨텍스트는 모든 테스트에서 공유된다.
정말인지 검증하기 위해 학습 테스트를 만들어보자.
```java
package org.example.chapter1.dao;  
  
import static org.hamcrest.CoreMatchers.not;  
import static org.hamcrest.MatcherAssert.assertThat;  
import static org.hamcrest.Matchers.either;  
import static org.hamcrest.Matchers.hasItem;  
  
import java.util.HashSet;  
import java.util.Set;  
import org.junit.jupiter.api.Test;  
import org.junit.jupiter.api.extension.ExtendWith;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.ApplicationContext;  
import org.springframework.test.context.ContextConfiguration;  
import org.springframework.test.context.junit.jupiter.SpringExtension;  
import static org.hamcrest.CoreMatchers.is;  
import static org.hamcrest.Matchers.nullValue;  
import static org.junit.jupiter.api.Assertions.assertTrue;  
  
@ExtendWith(SpringExtension.class)  
@ContextConfiguration  
public class SpringJUnit {  
    //테스트 컨텍스트가 매번 주입해주는 애플리케이션 컨텍스트는 항상 같은 오브젝트인지 테스트로 확인하기  
    @Autowired  
    ApplicationContext context;  
  
    static Set<SpringJUnit> testObjects = new HashSet<SpringJUnit>();  
    static ApplicationContext contextObject = null;  
  
    @Test  
    public void test1() {  
        assertThat(testObjects, not(hasItem(this)));  
        testObjects.add(this);  
  
        assertThat(contextObject == null || contextObject == this.context, is(true));  
        contextObject = this.context;  
    }  
  
    @Test  
    public void test2(){  
        assertThat(testObjects, not(hasItem(this)));  
        testObjects.add(this);  
  
        assertTrue(contextObject == null || contextObject == this.context);  
        contextObject = this.context;  
    }  
  
    @Test  
    public void test3(){  
        assertThat(testObjects, not(hasItem(this)));  
        testObjects.add(this);  
  
        assertThat(contextObject, either(is(nullValue())).or(is(this.context)));  
        contextObject = this.context;  
    }  
}
```
테스트 메소드에서 매번 동일한 애플리케이션 컨텍스트가 context 변수에 주입됐는지 확인해야 한다.

`확인하는 과정`
1. context를 저장해둘 스태틱 변수인 contextOnject가 null인지 확인한다.
   null이라면 첫 번째 텍스트일테니 일단 통과 그리고 contextObject에 현재 context를 저장
2. 다음부터는 저장된 contextObject가 null 이 아닕니 현재의 context랑 같은지 비교
3. 한 번이라도 다른 오브젝트가 나오면 테스트 실패!


| 테스트 메소드 | 설명                                                                                                                                                                                                     |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| test1() | assertThat() 을 이용<br>매처와 비교할 대상인 첫 번째 파라미터에 boolean 결과가 나오는 조건문을 넣는다.<br>그 결과를 is() 매처를 써서 true 와 비교                                                                                                   |
| test2() | 조건문을 받아서 그 결과가 true 인지 false인지 확인하도록 만들어진 assertTrue()라는 검증용 메소드를 assertThat() 대신 사용<br>(assertThat()보다 조금 간결해짐)                                                                                       |
| test3() | assertThat()을 이용<br>조건문을 넣어 그 결과를 true 와 비교하는 대신 매처의 조합을 이용<br>either() 는 뒤에 이어서 나오는 or() 와 함께 두 개의 매처 결과를 OR 조건으로 비교해준다. 두 가치 매처 중에 하나만 true 로 나와도 성공이다.<br>`nullValue()` 는 오브젝트가 null 인지 확인해주는 매처이다. |
사실 모든 테스트 메서드가 같은 기능을 하지만 조금씩 사용되는 매처가 다르다.
편한 방법이라고 생각되는 걸로 사용하면 될거 같다!

### 버그 테스트
---
코드에 오류가 있을 때 그 오류를 가장 잘 드러내줄 수 있는 테스트

버그 테스트는 일단 실패하도록 만들어야 한다. 버그가 원인이 돼서 테스트가 실패하는 코드를 만드는 것이다.

> `버그 테스트의 필요성과 장점`
>> 1. 테스트의 완성도를 높여준다.
>> 2. 버그의 내용을 명확하게 분석하게 해준다.
>> 3. 기술적인 문제를 해결하는데 도움이 된다.


`동등분할`
- 같은 결과를 내는 값의 범위를 구분해서 각 대표값으로 테스트 하는 방법
- 어떤 작업의 결과의 종류가 true, false 또는 예외발생 세 가지라면 각 결과를 내는 입력값이나 상황의 조합을 만들어 모든 경우에 대한 테스트를 해보는 것이 좋다.

`경계값 분석`
- 에러는 동등분할 범위의 경계에서 주로 많이 발생한다는 특징을 이용해 경계의 근처에 있는 값을 이용해 테스트하는 방법
- 보통 숫자의 입력 값인 경우 0이나 그 주변 값 또는 정수의 최대값, 최소값 등으로 테스트해보면 도움이 될 때가 많다.

## 면접 질문
Q. @DirtiesContext의 어노테이션에 대해 설명해주세요.
<details><summary>답안</summary>Spring 텍스트에서 컨텍스트가 변경될 수 있는 상황에서 컨텍스트를 리셋하거나 초기화하는 데 사용한다.</details>
