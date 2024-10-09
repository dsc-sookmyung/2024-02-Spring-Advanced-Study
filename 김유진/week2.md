# 2장. 테스트
✨ 테스트란 무엇이며, 그 가치와 장점, 활용 전략, 스프링과의 관계를 살펴보자!
<br\>


## 2.1 USERDAOTEST 다시 보기

### 2.1.1 테스트의 유용성

> 테스트는 단순히 확인을 넘어서, **코드의 안정성과 완성도를 보장**해주는 핵심 과정
> 
- **테스트의 역할**: 코드가 예상대로 동작하는지 확인하고, 개발자가 자신 있게 코드를 작성할 수 있도록 돕는 과정.
- **테스트 방법**: `main()` 메소드를 이용해 직접 메소드를 호출하고 결과를 눈으로 확인하며 기능을 검증.
- **테스트의 필요성**: 설계나 코드가 변경되더라도 기능이 유지되는지 확인하기 위해 필수적.
- **테스트 결과**: 코드가 의도대로 동작하지 않으면 결함을 발견하고 수정(디버깅) 가능. 성공하면 모든 결함이 제거되었다는 확신을 가질 수 있음.

### 2.1.2 UserDaoTest의 특징

- main() 메소드로 작성된 테스트
    
    ```java
    public class UserDaoTest {
        public static void main(String[] args) throws SQLException{
            Application context =
                    new GenericXmlApplicationContext("applicationContext.xml");
    
            UserDao dao = context.getBean("userDao", UserDao.class);
    
            User user = new User();
            user.setId("user");
            user.setName("백기선");
            user.setPassword("married");
    
            dao.add(user);
    
            System.out.println(user.getId() + " 등록 성공");
    
            User user2 = dao.get(user.getId());
            System.out.println(user2.getName());
            System.out.println(user2.getPassword());
    
            System.out.println(user2.getId() + " 조회 성공");
        }
    
    }
    ```
    
    → main() 메소드를 이용해 쉽게 테스트 수행을 가능하게 함.
    
    → 테스트할 대상인 UserDao를 직접 호출해서 사용함.
    

#### 웹을 통한 DAO 테스트 방법의 문제점

🤔 웹 프로그램에서 사용하는 DAO를 테스트하는 방법?

- 웹 화면을 통해 값을 입력하고 기능을 수행하고 결과를 확인하는 방법.

😱 테스트하고 싶은건 UserDao지만 다른 계층의 코드, 컴포넌트, 서버 설정 상태까지 모두 테스트에 영향을 줄 수 있음.

→ 테스트가 번거로움 + 오류에 대한 빠르고 정확한 대처가 힘들어짐.

→ 어떻게 해야 이런 문제를 피할 수 있을까?

#### 작은 단위의 테스트

📌 한꺼번에 많이 테스트하면 **테스트 수행 과정이 복잡**해짐 + **오류 발생시 정확한 원인을 찾기 힘들어짐**

➡️ 테스트는 가능하면 **작은 단위**로 쪼개서 해야 한다!

> **단위테스트?**
> 
> - 작은 단위의 코드에 대해 테스트를 수행하는 것

여기서 말하는 단위란…?

- **충분히 하나의 관심에만 집중해서 효율적으로 테스트할 만한 범위**

🤔 DB가 사용되어도 단위테스트라고 할 수 있을까?

- DB를 사용하더라도, 상태를 테스트 코드가 제어할 수 있다면 단위 테스트로 볼 수 있음.
    - 다만, DB 상태를 제어할 수 없는 경우, 단위 테스트로서의 가치가 줄어듦!!

📌 때로는 웹 인터페이스부터 DB까지 모든 계층을 포함해 테스트할 필요가 있음.

- 단위별로 검증한 후 통합 테스트를 진행하면, 문제 원인을 쉽게 파악할 수 있음.

<aside>
💡

작은 단위의 테스트는 코드의 안정성을 높이고, 개발 속도를 빠르게 유지하는 중요한 방법임.

</aside>

#### 자동수행 테스트 코드

📌 **수동 테스트**는 번거롭고 오류를 유발할 수 있다. 자주 반복하기 어렵고 비효율적이다.

하지만 **UserDaoTest는…!**

- **간편함**: IDE에서 단축키로 테스트 실행 가능.
- **속도**: 테스트가 1초 이내에 완료됨.
- **자주 테스트 가능**: 코드 수정 후 즉시 테스트로 문제 확인.

→ 테스트를 자주 수행해도 부담이 없다.

😮 **테스트**는 이렇게 **자동**으로 **수행**되도록 코드로 만들어지는 것이 중요함!

➡️ 별도로 테스트용 클래스를 만드는 편이 낫다.

- `UserDao` 클래스 안에 테스트 코드를 넣는 대신, 별도의 `UserDaoTest` 클래스를 만들어 관리함.
    - 코드 구조를 유지하며 테스트 관리 용이.

📌 **자동화 테스트의 효과**

- **안정성 확보**
    - 작은 코드 변경이 전체 기능에 미치는 영향을 쉽게 확인.
- **신속한 문제 확인**
    - 운영 중인 애플리케이션 수정 후, 전체 테스트로 문제 발생 여부 빠르게 확인 가능.

#### 지속적인 개선과 점진적인 개발을 위한 테스트

- 점진적인 개발
    - 단순한 기능 개발 후 테스트로 검증하여 확신을 가짐 → 기능을 조금씩 추가
- 지속적인 개선
    - 기존의 기능들이 수정된 코드에 영향을 받지 않는지 검증

### 2.1.3  UserDaoTest의 문제점

> UserDaoTest가 수동 테스트에 비해 장점이 많은 건 사실이지만,
> 
> 
> 🤔 만족스럽지 못한 부분도 있다…!
> 

- 수동 확인 작업의 번거로움
    - 테스트 수행은 자동으로 진행되지만 테스트 결과를 확인하는 일은 수동으로 진행된다
    
- 실행 작업의 번거로움
    - 매번 main() 메소드를 실행하는 것이 번거롭다.
 
## 2.2 USERDAOTEST 개선

🤔 **UserDaoTest**의 **두 가지 문제점**을 **개선**해보자!

### 2.2.1 테스트 검증의 자동화

#### 첫 번째 문제점인 테스트 결과의 검증 부분을 코드로 만들어보자!

- 먼저, **테스트 결과**에 대해 생각해보자
    1. 테스트 에러
        - 테스트가 진행되는 동안 에러가 발생하는 경우
    2. 테스트 실패
        - 테스트 결과가 기대한 것과 다르게 나오는 경우
        
- 위의 결과를 출력하기 위한 테스트 코드
    - 기대값과 결과값 비교해주는 로직을 추가
    
    ```java
    if (!user.getName().equals(user2.getName())) {
        System.out.println("테스트 실패 (name)")
    }
    else if (!user.getPassword().equals(user2.getPassword())) {
        System.out.println("테스트 실패 (password)")
    }
    else {
        System.out.println("조회 테스트 성공")
    }
    ```
    
    - 마지막으로 성공 메시지를 확인하면 된다!

### 2.2.2 테스트의 효율적인 수행과 결과 관리

🤔 main() 메소드를 이용한 테스트 작성 방법만으로는 수백 개의 테스트를 수행하는 일이 점점 부담이 될 것이다

→ JUnit을 사용하자!

- 자바로 단위테스트를 만들 때 유용하게 사용

#### JUnit 테스트로 전환

지금까지 만들었던 main() 메소드 테스트를 JUnit을 이용해 다시 작성한다.

- JUnit은 프레임워크
    - 프레임워크는 개발자가 만든 클래스에 대한 제어 권한을 넘겨받아서 주도적으로 애플리케이션의 흐름을 제어
    - 개발자가 만든 클래스의 오브젝트 생성, 실행하는 일은 프레임워크에 의해 진행됨.
        - 따라서 프레임워크에서는 main() 메소드도 필요없다.
    

#### 테스트 메소드 전환

테스트가 main() 메소드로 만들어진 것: 제어권을 직접 갖는다는 의미

→ 적합 X

→ main()에 있는 테스트 코드를 일반 메소드로 이동

- JUnit 프레임워크가 요구하는 조건
    1. 메소드가 public으로 선언
    2. 메소드에 @Test라는 애노테이션을 붙여주는 것.
        - 위 조건에 맞게 재구성한 코드
        
        ```java
        public class UserDaoTest{
        
            @Test
            public void addAndGet() throws SQLException {
                Application context =
                        new GenericXmlApplicationContext("applicationContext.xml");
        
                UserDao dao = context.getBean("userDao", UserDao.class);
        
                ...
            }
        }
        ```
        

#### 검증 코드 전환

- add()에 사용했던 user 오브젝트의 name 값 = get()에서 가져온 user2의 오브젝트의 name 값 → 테스트 성공
    
    ```java
    if (!user.getName().equals(user2.getName())) {...}
    ```
    

- 위 if문을 JUnit 에서 제공해주는 assertThat이라는 메소드를 이용해 변경한 코드
    
    ```java
    assertThat(user2.getName(), is(user.getName()));
    ```
    
    - assertThat()
        - 첫번째 파라미터의 값을 뒤에 나오는 매처(조건)로 비교해서 일치하는지 확인함.
        - 여기서는 is()라는 매처가 사용되었고, 이는 equals()와 동일한 기능을 가짐.
        - JUnit에서는 이 assertThat()이 실패하지 않으면 테스트가 성공했다고 인식함.

- 따라서 수정한 UserDaoTest 코드
    
    ```java
    public class userDaoTest {
    
        @Test
        public void addAndGet() throws SQLException {
            Application context =
                    new GenericXmlApplicationContext("applicationContext.xml");
    
            UserDao dao = context.getBean("userDao", UserDao.class);
    
            User user = new User();
            user.setId("user");
            user.setName("백기선");
            user.setPassword("married");
    
            dao.add(user);
    
            User user2 = dao.get(user.getId());
    
            assertThat(user2.getName(), is(user.getName()))
            assertThat(user2.getPassword(), is(user.getPassword()))
        }
    }
    ```
    

#### JUnit 테스트 실행

- 어디에든 main() 메소드를 하나 추가하고, 그 안에 JUnitCore 클래스의 main 메소드를 호출해주는 간단한 코드를 넣어주면 됨.
    
    ```java
    public static void main(String[] args) {
        JUnitCore.main("springbook.user.dao.UserDaoTest");
    }
    ```
    
    - 테스트 실행 시간, 결과, 실행된 테스트 메소드의 개수를 반환함.
 

## 2.3 개발자를 위한 테스팅 프레임워크 JUNIT

✨ 스프링의 핵심 기능 중 하나인 스프링 테스트 모듈도 JUnit을 이용하므로, 스프링의 기능을 익히기 위해서라도 JUnit은 꼭 사용할 줄 알아야 함.

### 2.3.1 JUnit 테스트 실행 방법

🤔 JUnitCore로 콘솔 메시지를 통해 테스트 결과를 확인하는 방법이 가장 간단하지만, 테스트 수가 많아지면 관리가 어려워진다.

→ **자바 IDE에 내장된 JUnit 테스트 지원 도구**를 사용하는 것이 더 효율적이다.

#### IDE

JUnitCore를 사용해 테스트를 실행하는 것보다 훨씬 편리

- JUnit 테스트 정보를 표시해주는 뷰가 나타나서 테스트 진행 상황을 보여줌
    - 테스트 총 수행 시간, 실행한 테스트의 수, 테스트 에러의 수, 테스트 실패의 수
- 한 번에 여러 테스트 클래스를 동시에 실행 가능

#### 빌드 툴

빌드 툴에서 제공하는 JUnit 플러그인이나 태스크를 이용해 JUnit 테스트 실행 가능

- 테스트 실행 결과: HTML, 텍스트 파일의 형태로 생성
- 여러 개발자가 만든 코드를 통합해서 테스트를 수행할 때 유리

### 2.3.2 테스트 결과의 일관성

🤔 테스트결과가 외부 상태에 따라 달라지는 문제점이 있다. 어떻게 해결해야 할까?

- 이전 테스트 때문에 DB에 중복된 데이터가 존재해서 테스트에 실패하는 경우가 생긴다.

➡️ 테스트를 마치고 나면 등록된 사용자 정보를 삭제하게 만들어주자!

#### deleteAll()의 getCount() 추가

📌 일관성 있는 테스트 결과를 위해 UserDao에 새로운 기능을 추가해야 한다.

- deleteAll
    - USER 테이블의 모든 레코드를 삭제
        
        ```jsx
        public void deleteAll() throws SQLException {
          Connection c = dataSource.getConnection();
        
          PreparedStatement ps = c.prepareStatement("delete from users");
          ps.executeUpdate();
        
          ps.close();
          c.close();
        }
        ```
        
- getCount()
    - USER 테이블의 레코드 개수를 반환
        
        ```jsx
        public int getCount() throws SQLException {
          Connection c = dataSource.getConnection();
        
          PreparedStatement ps = c.prepareStatement("select count(*) from users");
          ResultSet rs = ps.executeQuery();
          rs.next();
        
          int count = rs.getInt(1);
        
          rs.close();
          ps.close();
          c.close();
        
          return count;
        }
        ```
        

#### deleteAll()과 getCount()의 테스트

📌 추가된 기능에 대한 테스트를 만들어보자!

- 독립적으로 자동 실행되는 테스트를 만들기가 애매 → addAndGet() 테스트를 확장하자!

- addAndGet() 테스트
    - **단점**: 실행 전에 수동으로 USER 테이블의 내용을 모두 삭제해야 함
    - ➡️ deleteAll() 메소드를 테스트가 시작될 때 실행해주는 것은 어떨까??
        - deleteAll() 메소드를 검증하기 위해 getCount()를 함께 사용해보자!
            - getCount()에 대한 검증 작업은 어떻게 해야 할까…?
                - add() 메소드를 실행한 후 레코드 개수가 0에서 1로 바뀌는지 확인한다!
                
- deleteAll()과 getCount()가 추가된 addAndGet() 테스트
    
    ```jsx
    public class userDaoTest {
    
        @Test
        public void addAndGet() throws SQLException {
            ...
    
            dao.deleteAll()
            assertThat(dao.getCount(), is(0))
    
            User user = new User();
            user.setId("gyumee");
            user.setName("박성철");
            user.setPassword("springno1");
    
            dao.add(user);
            assertThat(dao.getCount(), is(1));
    
            User user2 = dao.get(user.getId());
    
            assertThat(user2.getName(), is(user.getName()))
            assertThat(user2.getPassword(), is(user.getPassword()))
      }
    }
    ```
    

#### 동일한 결과를 보장하는 테스트

📌 이제 반복적으로 테스트를 실행해도 동일한 결과가 나오게 된다!

- 동일한 테스트 결과를 얻을 수 있는 **다른 방법**도 있다
    - addAndGet() 테스트를 마치기 직전에 **테스트가 변경하거나 추가한 데이터를 모두 원래 상태로** 만들어 주는 것.
        - addAndGet() 테스트 실행 이전에 다른 이유로 데이터가 들어가 있다면 테스트가 실패할 수도 있다. → 그냥 테스트 전에 데이터를 지우는 게 낫다!

### 2.3.3 포괄적인 테스트

🫠 미처 생각하지 못한 문제가 있을 수도 있으니 더 꼼꼼하게 테스트를 실행해보자!

#### getCount() 테스트

> 🤧 getCount()를 더 꼼꼼히 테스트하기 위해 여러 User를 등록하며 결과를 확인해보자!
> 

➡️ 이때, 한 테스트 메소드에 한 가지 검증 목적에 집중해야 하므로,  **getCount()를 위한 새로운 테스트 메소드**를 만들어볼 것이다.

📌 테스트 시나리오

1. USER 테이블의 데이터를 모두 지우기
2. getCount() 로 레코드 개수가 0임을 확인
3. 3개의 사용자 정보를 하나씩 추가하며 매번 getCount() 결과가 하나씩 증가하는지 확인

- User 클래스에 한 번에 모든 정보를 넣을 수 있도록 초기화가 가능한 생성자를 추가하여 작한 코드
    
    ```jsx
    @Test
    public void count() throws SQLException {
        Application context =
                    new GenericXmlApplicationContext("applicationContext.xml");
    
        UserDao dao = context.getBean("userDao", UserDao.class);
    
        // User 생성자 추가함
        User user1 = new User("gyumee", "박성철", "springno1");
        User user2 = new User("leegw700", "이길원", "springno2");
        User user3 = new User("bumjin", "박범진", "springno3");
    
        dao.deleteAll()
        assertThat(dao.getCount(), is(0))
    
        dao.add(user1);
        assertThat(dao.getCount(), is(1))
    
        dao.add(user2);
        assertThat(dao.getCount(), is(2))
    
        dao.add(user3);
        assertThat(dao.getCount(), is(3))
    }
    ```
    
    📌 주의해야 할 점
    
    - JUnit은 특정한 테스트 메소드의 실행 순서를 보장해주지 않으므로
    - 모든 테스트는 **실행 순서에 상관없이** **독립적으로 항상 동일한 결과**를 낼 수 있도록 해야 한다.
    

#### addAndGet() 테스트 보완

🤧 이번엔 addAndGet() 테스트를 좀 더 보완해보자!

- add()의 기능은 충분히 검증됐지만… **get()**에 대한 테스트는 조금 부족한 감이 있다!
- get()이 주어진 id의 사용자를 정확히 가져왔는지 검증해야 한다.
    - 두 개의 User를 add()
    - 각 User의 id를 파라미터로 전달해서 get()을 실행

- 테스트 기능이 보완된 addAndGet() 테스트 코드
    
    ```jsx
    @Test
    public void addAndGet() throws SQLException {
        Application context =
                    new GenericXmlApplicationContext("applicationContext.xml");
    
        UserDao dao = context.getBean("userDao", UserDao.class);
    
        // User 생성자 추가함
        User user1 = new User("gyumee", "박성철", "springno1");
        User user2 = new User("leegw700", "이길원", "springno2");
    
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
    

#### get() 예외조건에 대한 테스트

🤔 그런데 get() 메소드에 전달된 id 값에 해당하는 사용자 정보가 없다면 어떻게 될까?

1. 특별한 값(null) 리턴
2. 예외 던지기!
    - 여기서는 두 번째 방법을 사용해 볼 것이다!

- 스프링이 정의한 데이터 액세스 예외 클래스(EmptyResultDataAccessException 예외)를 사용한다.
    - 이 때는 예외가 **발생**해야 ****테스트가 성공인데, 어떻게 작성해야 테스트를 성공시킬 수 있을까?
        - `@Test(expected=EmptyResultDataAccessException.class)`를 넣어서 기대하면 예외 클래스를 지정한다.
            
            > @Test 애노테이션에 expected를 추가해놓으면 보통의 테스트와는 반대로,
            > 
            > - 정상적으로 테스트를 마치면 테스트가 실패하고,
            > - expected에서 지정한 예외가 던져지면 테스트가 성공함.
            
            ```jsx
            @Test(expected=EmptyResultDataAccessException.class)
            public void getUserFailure() throws SQLException {
              Application context =
                            new GenericXmlApplicationContext("applicationContext.xml");
            
              UserDao dao = context.getBean("userDao", UserDao.class);
              dao.deleteAll();
            
              assetThat(dao.getCount(), is(0));
            
              dao.get("unknwon_id") // 예외를 발생시키는 아이디
            }
            ```
            
            → 하지만 이 테스트를 실행시키면 테스트는 실패한다!
            

#### 테스트를 성공시키기 위한 코드의 수정

📌 이제부터는 이 테스트가 성공하도록 get() 메소드 코드를 수정해보자!

- 데이터를 찾지 못하면 예외를 발새시키도록 수정한 get() 메소드
    
    ```java
    public User get(String id) throws SQLException{
      ResultSet rs = ps.executeQuery();
    
      User user = null;
      if (rs.next()) {
        user = new User();
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setPassword(rs.getString("password"));
      }
      rs.close();
      ps.close();
      c.clse();
    
      if (user == null) throw new EmptyResultDataAccessException(1);
    
      return user;
    }
    ```
    

#### 포괄적인 테스트

📌 이렇게 DAO의 메소드에 대한 포괄적인 테스트를 만들어두는 것이 **안전**하고 **유용**하다.

- 개발자가 테스트를 만들 때 자주 하는 실수?
    - **성공하는 테스트만 골라서 만드는 것**
    - 발생할 수 있는 다양한 상황과 입력 값을 고려하는 **포괄적인 테스트**를 만들어야 한다.

### 2.3.4 테스트가 이끄는 개발

> 🤔 get() 메소드의 예외 테스트를 만드는 과정을 다시 돌아보자… **한 가지 흥미로운 점**이 있다!
> 
> - 테스트를 먼저 만들고 UserDao 코드를 수정했다는 점!
> - 그런데 **실제로 이러한 개발 전략이 존재**한다…!

#### 기능설계를 위한 테스트

- getUserFailure() 처럼 테스트할 코드도 없는데 어떻게 테스트를 만들 수 있을까?
    - **추가하고 싶은 기능**을 **코드로 표현**하려고 했기 때문에 가능하다!
    - 테스트가 성공하는 순간 코드 구현과 테스트라는 **두 가지 작업이 동시에 끝**나게 된다!

#### 테스트 주도 개발

> 테스트 주도 개발(TDD), 테스트 우선 개발(TFD)
> 
> - 만들고자 하는 기능의 내용을 담고 있으면서 만들어진 코드를 검증도 해줄 수 있도록 테스트 코드를 먼저 만들고 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방법

### 2.3.5 테스트 코드 개선

📌 지금까지 세 개의 테스트 메소드를 만들었다. 이제 테스트 코드를 **리팩토링**해보자!

- UserDaoTest
    - 기계적으로 반복되는 부분이 존재
        - 스프링의 애플리케이션 컨텍스트를 만드는 부분
        - 컨텍스트에서 UserDao를 가져오는 부분
    - JUnit이 제공하는 기능을 활용해서 중복된 코드를 별도의 메소드로 뽑아내자!

#### @Before

- 중복 코드를 제거한 UserDaoTest
    
    ```java
    public class UserDaoTest {
      private UserDao dao;
    
      @Before // @Test가 실행되기 전에 먼저 실행되어야 하는 메소드를 정의한다.
      public void setUp() {
        ApplicationContext context =
            new GenericXmlApplicationContext("applicationContext.xml");
        this.dao = context.getBean("userDao", UserDao.class);
      }
    }
    ...
    ```
    
- JUnit이 하나의 테스트 클래스를 가져와 테스트를 수행하는 방식
    1. 테스트 클래스에서 @Test가 붙은 public이고 void형이며 파라미터가 없는 테스트 메소드를 모두 찾는다.
    2. 테스트 클래스의 오브젝트를 하나 만든다.
    3. `@Before`가 붙은 메소드가 있으면 실행한다.
    4. `@Test`가 붙은 메소드를 하나 호출하고 테스트 결과를 저장해둔다.
    5. `@After`가 붙은 메소드가 있으면 실행한다.
    6. 나머지 테스트 메소드에 대해 2~5번을 반복한다.
    7. 모든 테스트의 결과를 종합해서 돌려준다.

#### 픽스처

📌 픽스처?

- 테스트를 수행하는 데 필요한 정보나 오브젝트
- 일반적으로 여러 테스트에서 반복적으로 사용 → @Before 메소드를 이용해 생성해두면 편리

- User 픽스처를 적용한 UserDaoTest
    
    ```java
    public class UserDaoTest {
      private UserDao dao;
      private User user1;
      private User user2;
      private User user3;
    
      @Before
      public void setUp() {
        ...
    
        this.user1 = new User("gyumee", "박성철", "springno1");
        this.user2 = new User("leegw700", "이길원", "springno2");
        this.user3 = new User("bumjin", "박범진", "springno3");
      }
    }
    ```


## 2.4 스프링 테스트 적용

✨ 애플리케이션 컨텍스트는 매번 생성되면 시간이 많이 걸리기 때문에 테스트 전체에서 공유하는 것이 효율적이다. 

✨ JUnit의 스태틱 메소드나 스프링의 애플리케이션 컨텍스트 테스트 지원 기능을 사용해 컨텍스트를 한 번만 생성하고, 여러 테스트에서 공유하도록 할 수 있다.

### 2.4.1 테스트를 위한 애플리케이션 컨텍스트 관리

🫠 UserDaoTest에 스프링의 테스트 컨텍스트 프레임워크를 적용해보자!

#### 스프링 테스트 컨텍스트 프레임워크 적용

- @Before 메소드에서 애플리케이션 컨텍스를 생성하는 코드를 제거
- ApplicationContext 타입의 인스턴스 변수를 선언하고 스프링이 제공하는 @Autowired 애노테이션 추가
    
    ```java
    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration(locations="/applicationContext.xml")
    public class UserDaoTest {
      @Autowired
      private ApplicationContext context;
    
      @Before
      public void setUp() {
        this.dao = context.getBean("userDao", UserDao.class);
    
        this.user1 = new User("gyumee", "박성철", "springno1");
        this.user2 = new User("leegw700", "이길원", "springno2");
        this.user3 = new User("bumjin", "박범진", "springno3");
      }
    }
    ```
    
    🤔 테스트는 성공했지만 인스턴스 변수인 context를 초기화해주는 코드가 없다.
    
    - 따라서, setUp() 메소드에서 context를 사용하려고 하면 NullPointerException이 발생해야 하지만 테스트는 문제 없이 끝이 난다!

#### 테스트 메소드의 컨텍스트 공유

📌 setUp() 메소드에 확인용 코드를 추가하고 다시 실행해보자!

- 출력된 결과를 확인하면 context는 하나의 애플리케이션 컨텍스트가 만들어져 모든 테스트 메소드에서 사용되고 있다.
- 반면, UserDaoTest의 오브젝트는 매번 주소값이 다르다.

- 그렇다면, context 변수에 어떻게 애플리케이션 컨텍스트가 들어있는 것일까?
    - 스프링의 JUnit 확장 기능은 테스트 전에 한 번만 애플리케이션 컨텍스트를 생성하고, 이를 테스트 오브젝트의 특정 필드에 자동으로 주입해준다.
    

#### 테스트 클래스의 컨텍스트 공유

📌 여러 개의 테스트 클래스가 있는데 **같은 설정 파일**을 가진 애플리케이션 컨텍스트를 사용 → 스프링은 **테스트 클래스 간에도 애플리케이션 컨텍스트를 공유**

- 즉, 설정 파일이 같으면 테스트 전체에서 **하나의 애플리케이션 컨텍스트**만 생성되어 사용

#### @Autowired

✨ @Autowired?

- 스프링 DI에 사용되는 특별한 애노테이션

- @Autowired가 붙은 인스턴스 변수가 있으면, 테스트 컨텍스트 프레임워크는 변수 타입과 일치하는 컨텍스트 내의 빈을 찾고, 타입과 일치하는 빈이 있으면 인스턴스 변수에 주입해준다.
    - 메소드가 없이도 주입이 가능하다!

⇒ **타입에 의한 자동 와이어링**

🤔 테스트 코드에서는 ApplicationContext라는 타입의 변수에 @Autowired를 붙였는데 애플리케이션 컨텍스트가 DI 됐다. 어찌 된 일일까?

- 스프링 애플리케이션 컨텍스트는 초기화할 때, **자기 자신도 빈으로 등록**한다.
- 따라서 애플리케이션 컨텍스트에는 ApplicationContext 타입의 빈이 존재 → DI도 가능.

- UserDao를 직접 DI 받도록 만듣 테스트
    
    ```java
    publc class UserDaoTest {
      @Autowired
      UserDao dao;
    
      ...
    }
    ```
    
    - `@Autowired`는 **타입에 맞는 빈을 자동으로 주입**.
    - **같은 타입의 빈이 여러 개**면 주입할 빈을 결정할 수 없음.
    - **테스트는 애플리케이션 클래스와 밀접해도 문제 없음**, 필요에 따라 타입 구체화 가능.

### 2.4.2 DI와 테스트

> 우리는 절대로 DataSource의 구현 클래스를 바꾸지 않을 것이다. 우리가 개발하는 시스템은 운영 중에 항상 SimpleDriverDataSource를 통해서만 DB 커넥션을 가져올 것이다. 그런데도 굳이 DataSource 인터페이스를 사용하고 DI를 통해 주입하는 방식을 이용해야 하는가?
> 
- 그래도 인터페이스를 두고 DI를 적용해야 한다.
    1. 소프트웨어 개발에서 절대로 바뀌지 않는 것은 없기 때문
        - 언젠가 변경이 필요할때, 수정에 들어가는 시간과 비용의 부담을 줄여줄 수 있다.
    2. 다른 차원의 서비스 기능을 도입할 수 있기 때문
        - 새로운 부가 기능을 추가하는데 용이하다.
    3. 테스트 때문
        - 효율적인 테스트를 손쉽게 만들 수 있다.
        - DI → **테스트가 작은 단위의 대상에 대해 독립적으로 만들어지고 실행**되게 함.

#### 테스트 코드에 의한 DI

📌 **DI는 스프링 컨테이너 외에도 가능**하며, 직접 **`DaoFactory`** 등을 통해 DI 적용 가능하다.

- 테스트 코드 내에서 **테스트용 DataSource**를 사용해 DAO의 의존성을 수동으로 변경할 수 있음.
    - 예를 들어, `SingleConnectionDataSource`를 사용해 빠르게 테스트를 실행하고, `setDatasource()`로 직접 주입 가능.
        
        ```java
        @RunWith(SpringJUnit4ClassRunner.class)
        @ContextConfiguration(locations="/applicationContext.xml")
        @DirtiesContext
        public class UserDaoTest {
          @Autowired
          private ApplicationContext context;
        
          @Before
          public void setUp() {
            ...
            DataSource dataSource = new SingleConnectionDataSource(
              "jdbc:mysql://localhost/testdb", "spring", "book", true
            );
        
          }
        }
        ```
        
        → 테스트 내에서 애플리케이션 컨텍스트의 상태를 변경하면 **모든 테스트에 영향을 미칠 수 있음**.
        
        → 이를 방지하기 위해 `@DirtiesContext` 애노테이션을 사용해 **매 테스트마다 새로운 컨텍스트 생성**.
        
- **장점**: XML 설정을 수정하지 않고도 테스트에서 오브젝트 관계를 재구성 가능.
- **단점**: 매번 새로운 컨텍스트를 생성해 조금 찜짐하다.

#### 테스트를 위한 별도의 DI 설정

> 🤔 DAO가 테스트에서만 다른 DataSource를 사용하게 할 수 있는 방법이 또 있을까?
> 

→ 테스트에서 사용될 **테스트 전용 설정파일**을 만들어두자!

1. 서버에서 운영용으로 사용할 DataSource를 빈으로 등록
2. 테스트에 적합하게 준비된 DB를 사용하는 가벼운 DataSource를 빈으로 등록

- 테스트용 설정 파일 적용
    
    ```java
    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration(locations="/test-applicationContext.xml")
    public class UserDaoTest {
      ...
    }
    ```
    

#### 컨테이너 없는 DI 테스트

> 🙂 아예 스프링 컨테이너를 사용하지 않고 테스트를 만들자!
> 
- 스프링 컨테이너를 이용해서 IoC 방식으로 생성되고 DI되도록 하는 대신, **테스트 코드에서 직접 오브젝트를 만들고 DI해서 사용**해도 된다.

- 애플리케이션 컨텍스트 없는 DI 테스트
    
    ```java
    public class UserDaoTest {
      // @Autowired가 없음.
      UserDao dao;
    
      ...
    
      @Before
      public void setUp() {
        ...
        dao = new UserDao(); // 직접 오브젝트 생성
        DataSource dataSource = new SingleConnectionDataSource(
          "jdbc:mysql://localhost/testdb", "spring", "book", true
        );
        dao.setDataSource(dataSource); // 직접 DI를 해줌.
      }
    }
    ```
    
    - DataSource를 직접 만드는 번거로움은 있지만 코드가 더 **단순**해지고 **이해**하기 편해졌다.

#### DI를 이용한 테스트 방법 선택

🤔 그렇다면 세 가지 방법 중 어떤 것을 선택해야 할까?

- 모두 장단점이 있지만…
    - 항상 스프링 컨테이너 없이 테스트할 수 있는 방법을 우선적으로 고려!
        - 테스트 수행 속도가 가장 빠르고 간결함.
    - 여러 오브젝트와 복잡한 의존관계가 있는 오브젝트
        - 스프링의 설정을 이용한 DI의 테스트 방식을 이용
    - 테스트 설정을 따로 만들었다고 하더라도 예외적인 의존관계를 강제로 구성해서 태스트해야 하는 경우
        - DI 받은 오브젝트에 다시 테스트 코드로 수동 DI 하는 방식 이용
     

## 2.5 학습 테스트로 배우는 스프링

✨ 학습 테스트?

- 자신이 만들지 않은 프레임워크나 다른 개발팀에서 만들어서 제공한 라이브러리에 대해서 테스트를 작성하는 것

✨ 학습 테스트의 목적?

1. 테스트를 만들려고 하는 기술이나 기능에 대한 이해도, 사용 방법의 이해도에 대한 검증
2. 테스트 코드를 작성해보면서 빠르고 정확하게 사용법을 익히기 위함.

### 2.5.1 학습 테스트의 장점

1. **다양한 조건에 따른 기능을 손쉽게 확인해볼 수 있다.**
    - 학습 테스트는 자동화된 테스트 코드로 만들어지기 때문에 다양한 조건에 따라 기능이 어떻게 동작하는지 빠르게 확인할 수 있다.
2. **학습 테스트 코드를 개발 중에 참고할 수 있다.**
    - 다양한 기능과 조건에 대한 테스트 코드를 남겨둘 수 있다
        
        → 실제 개발에서 샘플 코드로 참고할 수 있다.
        
3. **프레임워크나 제품을 업그레이드할 때 호환성 검증을 도와준다.**
    - 기존에 사용했던 API나 기능에 변화가 있거나 업데이트된 제품에 버그가 있는지 확인이 가능하다.
4. **테스트 작성에 대한 좋은 훈련이 된다.**
    - 프레임워크의 학습 테스트는 대체로 단순하다.
        
        → 한결 작성하기 수월하고 부담도 적음.
        
    - 따라서 테스트 작성의 훈련 기회로 삼기에 좋다.
5. **새로운 기술을 공부하는 과정이 즐거워진다.**
    - 테스트 코드를 만들면서 하는 학습은 흥미롭고 재밌다.

### 2.5.2 학습 테스트 예제

📌 지금까지 소개했더 기능에 대해 학습 테스트를 만들어보자.

#### JUnit 테스트 오브젝트 테스트

> JUnit은 테스트 메소드를 수행할 때마다 새로운 오브젝트를 만든다고 했는데, 정말 그러한지 학습 테스트 작성을 통해 알아보자!
> 

- 테스트 방법
    - 새로운 테스트 클래스 생성 → 세 개의 테스트 메소드 추가
    - 테스트 클래스 자신의 타입으로 스태틱 변수를 선언
    - 매 테스트 메소드에서 현재 스태틱 변수에 담긴 오브젝트와 자신을 비교해서 동일하지 않다는 사실을 확인
    - 현재 오브젝트를 스태틱 변수에 저장
    - 중복 X 확인

- JUnit 테스트 오브젝트 생성에 대한 학습 테스트
    
    ```java
    public class JUnitTest {
      static Set<JUnitTest> testObjects = new HashSet<JUnitTest>;
    
      @Test
      public void test1() {
        assertThat(testObjects, not(hasItem(this)))
        testObjects.add(this);
      }
    
      @Test
      public void test2() {
        assertThat(testObjects, not(hasItem(this)))
        testObjects.add(this);
      }
    
      @Test
      public void test3() {
        assertThat(testObjects, not(hasItem(this)))
        testObjects.add(this);
      }
    
    }
    ```
    
    → JUnit의 특성을 이해 + 테스트를 만드는 방법에 대해 공부
    

#### 스프링 테스트 컨텍스트 테스트

> 스프링 테스트용 애플리케이션 컨텍스트는 테스트 개수에 상관없이 한 개만 만들어지고, 모든 테스트에서 공유된다고 했는데 정말 그러한지 학습 테스트 작성을 통해 검증해보자!
> 
- 테스트 방법
    - 새로운 설정파일 생성
    - JUnitTest에 @RunWith와 @ContextConfiguration 애노테이션 추가
    - 생성한 설정파일을 사용하는 테스트 컨텍스트를 적용
    - @Autowired로 주입된 context 변수가 같은 오브젝트인지 확인하는 코드 추가

- 스프링 테스트 컨텍스트에 대한 학습 테스트
    
    ```java
    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration
    public class JUnitTest {
      @Autowired
      ApplicationContext context;
    
      static Set<JUnitTest> testObjects = new HashSet<JUnitTest>();
      static ApplicationContext contextObject = null;
    
      @Test
      public void test1() {
        assertThat(testObjects, not(hasItem(this)));
        testObjects.add(this);
    
        assertThat(
          contextObject == null ||
          contextObject == this.context, is(true)
        );
        contextObject = this.context;
      }
    
      @Test
      public void test2() {
        assertThat(testObjects, not(hasItem(this)));
        testObjects.add(this);
    
        assertTrue(
          contextObject == null ||
          contextObject == this.context);
        contextObject = this.context;
      }
    
      @Test
      public void test3() {
        assertThat(testObjects, not(hasItem(this)));
        testObjects.add(this);
    
        assertThat(contextObject,
          either(is(nullValue())).or(is(this.context))
        );
        contextObject = this.context;
      }
    }
    ```
    

### 2.5.3 버그 테스트

📌 버그 테스트는 **실패하도록** 만들어야 한다.

→ 이후, 버그 테스트가 성공할 수 있도록 애플리케이션 코드를 수정

- 버그 테스트의 필요성과 장점
    - 테스트의 완성도를 높여준다
    - 버그의 내용을 명확하게 분석하게 해준다
    - 기술적인 문제를 해결하는 데 도움이 된다.
