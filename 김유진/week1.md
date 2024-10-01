# 1장. 오브젝트와 의존관계

### ✨ 오브젝트의 설계와 구현, 동작원리에 집중할 것

## 1.1 초난감 DAO

- 사용자 정보를 JDBC API를 통해 DB에 저장하고 조회할 수 있는 간단한 DAO 생성

### 1.1.1 User

```java
pulbic class User {
	String id;
	String name;
	String password;
	
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

### 1.1.2 UserDao

사용자 정보를 DB에 넣고 관리할 수 있는 DAO 클래스를 생성

- UserDao.java
    
    ```java
    public class UserDao {
        public void add(User user) throws ClassNotFoundException, SQLException {
            Class.forName("com.mysql.jdbc.Driver");
            Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8&serverTimezone=Asia/Seoul", "root",
                    "ch122411");
    
            PreparedStatement ps = c.prepareStatement(
                    "insert into users(id, name, password) values(?,?,?)");
            ps.setString(1, user.getId());
            ps.setString(2, user.getName());
            ps.setString(3, user.getPassword());
    
            ps.executeUpdate();
    
            ps.close();
            c.close();
        }
    
        public User get(String id) throws ClassNotFoundException, SQLException {
            Class.forName("com.mysql.jdbc.Driver");
            Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8&serverTimezone=Asia/Seoul", "root",
                    "ch122411");
            PreparedStatement ps = c
                    .prepareStatement("select * from users where id = ?");
            ps.setString(1, id);
    
            ResultSet rs = ps.executeQuery();
            rs.next();
            User user = new User();
            user.setId(rs.getString("id"));
            user.setName(rs.getString("name"));
            user.setPassword(rs.getString("password"));
    
            rs.close();
            ps.close();
            c.close();
    
            return user;
        }
    }
    ```
    

위의 코드가 제대로 동작하는지 어떻게 확인할 수 있을까?

1️⃣ 웹 브라우저를 통해 DAO 기능을 사용해 본다
   → 배보다 배꼽이 더 큰 일이 된다.
2️⃣ 눈으로만 확인한다
   → 꺼림칙하다.

### 1.1.3 main()을 이용한 DAO 테스트 코드

✨ **스태틱 메소드 main()을 이용한다!**

→ 오브젝트 스스로 자신을 검증하게 만들어준다.

> 📌 하지만 현재의 코드는 개발팀에서 바로 쫓겨날 수 있는 초난감 DAO 코드이다. 이를 객체지향 기술의 원리에 충실한 멋진 스프링 스타일의 코드로 개선해볼 것이다.
> 
> - 과연 무엇이 문제일까? <br>


## 1.2 DAO의 분리

### 1.2.1 관심사의 분리

변경이 일어날 때 필요한 작업을 최소화하고, 그 변경에 다른 곳에 문제를 일으키지 않게 하기 위해서는…

→  **`분리와 확장`**을 고려한 설계가 필요하다!

- 관심사의 분리(Seperation of Concerns) 관심이 같은 것끼리는 하나의 객체 안으로 또는 친한 객체로 모이게 하고, 관심이 다른 것은 가능한 따로 떨어져서 서로 영향을 주지 않도록 분리하는 것이라고 생각할 수 있다.
- 관심사가 같은 것끼리 모으고 다른 것은 분리해줌으로써 같은 관심에 효과적으로 집중할 수 있게 만들어주는 것이다.

### 1.2.2 커넥션 만들기의 추출

**#### 🤔 User DAO의 관심사항이 너무 많다 !**

1️⃣ DB와 연결을 위한 커넥션을 어떻게 가져올까.

2️⃣ SQL문장을 담을 Statement를 만들고 실행하는 것.

3️⃣ 작업이 끝나면 공유 리소스를 시스템에 돌려주는 것.

- 가장 큰 문제는 DB 커넥션을 가져오는 코드가 다른 관심사와 섞여있고, 또 get() 메소드에도 중복되어 있다는 점이다.
- 이렇게 하나의 관심사가 중복되고 흩어져 있으면, 변경에 대해 유연하게 대처하기 어렵고, 스파게티 코드가 되는 것이다.

#### 😮 **중복 코드의 메소드 추출**

- 중복된 DB 연결 코드를 getConnection()이라는 이름의 독립적인 메소드로 만들어준다!
    - getConnection() 메소드를 추출해서 중복을 제거한 UserDao
        
        ```java
        private Connection getConnection() throws ClassNotFoundException, SQLException {
                Class.forName("com.mysql.jdbc.Driver");
                Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8&serverTimezone=Asia/Seoul", "root",
                        "ch122411");
                return c;
            }
        ```
        
    
    → DB 연결과 관련된 부분에 변경이 일어났을 경우, getConnection()이라는 한 메소드만 수정하면 된다.
    
    → 관심의 종류에 따라 코드를 구분해놓았기 때문에 한 가지 관심에 대한 변경이 일어날 경우 그 관심이 집중되는 부분의 코드만 수정하면 된다.
    

#### 😤 **변경사항에 대한 검증: 리팩토링과 테스트**

- **`중복 코드의 메소드 추출`** 에서 한 작업은 UserDao의 기능에는 아무런 변화를 주지 않았지만, 이전보다 훨씬 1️⃣ **깔끔**해졌고 2️⃣ **변화에 유연**한 코드가 됐다.
    
    → 이러한 작업을 ❗**리팩토링**❗이라고 한다.
    
- 또한 getConnection()처럼 공통의 기능을 담당하는 메소드로 중복된 코드를 뽑아내는 것을
    - **메소드 추출 기법**이라고 부른다.

### 1.2.3 DB 커넥션 만들기의 독립

> 🤔 납품 과정에서 UserDao를 공개하지 않고, 고객 스스로 원하는 DB 커넥션 생성 방식을 적용하면서 UserDao를 사용하게 할 수 있을까?
> 

#### **상속을 통한 확장**

- 기존 UserDao 코드를 한 단계 더 분리한다.
- UserDao의 메소드 구현 코드를 지우고, getConnection()을 추상메소드로 만든다.
- 해당 추상 클래스의 서브 클래스를 만들어서 고객이 원하는 대로 구현하면 된다.

→ UserDao의 소스코드를 제공하지 않아도 getConnection() 메소드를 원하는 방식으로 확장하여 사용할 수 있다.

![상속을 통한 확장 방법이 제공되는 UserDao](https://github.com/user-attachments/assets/d81a922c-6788-4f58-a61f-c94e70c5bbb0)


- 상속을 통한 확장 방법이 제공되는 UserDao
    
    ```java
    public abstract class UserDao {
    
    	public void add(User user) throws ClassNotFoundException, SQLException {
          Connection c = getConnection();
          ...
       }
       public User get(String id) throws ClassNotFoundException, SQLException {
          Connection c = getConnection();
          ...
       }
       public abstract Connection getConnection() throws ClassNotFoundException, 
    		   SQLException;
    }
    
    public class NUserDao extends UserDao {
    		
        @Override
        public Connection getConnection() throws SQLException {
            // N사 DB Connection 생성 코드
            return null;
        }
    }
    
    public class DUserDao extends UserDao {
    
        @Override
        public Connection getConnection() throws SQLException {
            // D사 DB Connection 생성 코드
            return null;
        }
    }
    ```
    

이처럼 슈퍼클래스에서 기본적인 로직의 흐름을 정의하고, 일부 기능을 추상 메소드나 오버라이딩 가능한 메소드로 구현한 후, 서브클래스에서 이러한 메소드를 필요에 맞게 재구현하여 사용하는 방법을 디자인 패턴에서는 **템플릿 메소드 패턴**이라고 한다.

➡️ 이 설명을 "UserDao에 팩토리 메소드 패턴을 적용해서 getConnection()을 분리합시다."라는 한마디로 담아낼 수 있다.

> 💡디자인패턴
> 
> 
> 소프트웨어 설계 시 특정 상황에서 자주 만나는 문제를 해결하기 위해 사용할 수 있는 재사용 가능한 솔루션을 말한다.
> 

> 💡템플릿 메소드 패턴 template method pattern
> 
> 
> 디자인 패턴에서 이렇게 슈퍼클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메소드나 오버라이딩이 가능한 protected 메소드 등으로 만든 뒤 서브클래스에서 이런 메소드를 필요에 맞게 구현해서 사용하도록 하는 방법.
> 

> 💡팩토리 메소드 패턴 (factory method pattern)
> 
> 
> 서브클래스에서 오브젝트 생성 방법과 클래스를 결정할 수 있도록 미리 정의해둔 메소드를 팩토리 메소드라고 하고, 이 방식을 통해 오브젝트 생성 방법을 나머지 로직, 즉 슈퍼 클래스의 기본 코드에서 독립시키는 방법을 팩토리 메소드 패턴이라고 한다.
> 

팩토리 메서드 패턴이나 템플릿 메서드 패턴을 사용하면 관심사가 다른 코드를 분리하고 확장할 수 있다. 그러나 이러한 방법은 상속을 기반으로 하므로 몇 가지 단점이 있다.

1. **다중 상속의 제한**: 자바는 다중 상속을 허용하지 않는다. 단순히 커넥션을 가져오는 방법을 분리하기 위해 상속 구조를 만들면, `UserDao`는 다른 목적의 상속을 적용하기 어려워진다.
2. **밀접한 상하위 클래스 관계**: 상속을 통해 관심사를 분리하고 확장성을 높였지만, 상속 관계로 인해 두 가지 다른 관심사가 긴밀하게 결합될 수 있다. 슈퍼클래스의 내부 변경이 있을 경우, 모든 서브클래스를 함께 수정하거나 다시 개발해야 할 수도 있다.
3. **기능의 재사용 제한**: 만약 새로운 DAO 클래스가 계속 만들어진다면, 상속을 통해 구현된 `getConnection()` 코드가 각 DAO 클래스마다 중복되어 나타나는 심각한 문제가 발생할 수 있다. <br>

## 1.3 DAO의 확장

추상 클래스를 만들고 이를 상속한 서브클래스에서 변화가 필요한 부분을 바꿔서 쓸 수 있게 만든 이유는

- **변화의 성격이 다른 것을 분리**하고
- **독립적으로 변경**하기

위해서다. 

🤔 그러나 여러가지 단점이 많은, **상속**이라는 방법을 사용했다는 사실이 **불편**하게 느껴진다.

### 1.3.1 클래스의 분리

> 🤔 관심사를 본격적으로 **독립**시키면서 동시에 **손쉽게 확장**할 수 있는 방법을 알아보자.
> 
> 
> ➡️ DB 커넥션과 관련된 부분을 서브클래스가 아닌 별도의 클래스에 담고, UserDao가 이용하게 하자.
> 
- *SimpleConnectionMaker*라는 새로운 클래스를 만들고 DB 생성 기능을 그 안에 넣는다. 그리고 UserDao는 new 키워드를 사용해 *SimpleConnectionMaker* 클래스의 오브젝트를 만들어 두고, 이를 add(), get() 메소드에서 사용하면 된다.

😱 아래의 코드를 보면, 상속을 통해 커넥션 기능의 확장이 **다시 불가능**해졌다.

```java
public class SimpleConnectionMaker {
    public Connection makeNewConnection() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8&serverTimezone=Asia/Seoul", "root",
                "ch122411");
        return c;
    }
}
```

이런 문제의 원인은 UserDao가 바뀔 수 있는 정보, 즉 DB커넥션을 가져오는 클래스에 대해 너무 많이 알고 있기(종속적이기) 때문이다.

### 1.3.2 인터페이스의 도입

🤔 그렇다면 클래스를 분리하면서도 이런 문제를 해결할 수는 없을까?

➡️ **인터페이스!**

![인터페이스를 도입한 결과](https://github.com/user-attachments/assets/39e50cac-fd34-4289-99c0-cc4aa72da5a1)


😱 인터페이스를 이용해서 DB커넥션을 제공하는 클래스에 대한 구체적인 정보는 모두 제거되었지만, 초기에 어떤 클래스의 오브젝트를 사용할 지 결정하는 생성자의 코드가 제거되지 않고 남아 있었다.

결국, 또 다시 **원점**이다.

### 1.3.3 관계설정 책임의 분리

**🤔 원점으로 돌아간 이유가 무엇일까?**

- ⚠️ **문제점**: UserDao는 특정 ConnectionMaker 구현 클래스의 객체를 사용해야 하는 관심사가 존재
- 💡 **결론**: 이러한 관심사를 UserDao에서 분리하지 않으면, UserDao는 독립적이고 확장 가능한 클래스 X

**클라이언트와의 관계**

- UserDao를 사용하는 클라이언트가 적어도 하나 존재
- **역할**: UserDao의 클라이언트 객체는 UserDao와 ConnectionMaker 구현 클래스의 관계를 정의하는 제3의 관심사를 분리하여 적절한 곳에 두는 역할

**ConnectionMaker 결정 방법**

UserDao 사용 전에 어떤 ConnectionMaker 구현 클래스를 사용할지 결정하는 방법:

1. **생성자 호출**: UserDao의 생성자에서 ConnectionMaker 객체를 주입받음.
2. **외부 주입**: UserDao 코드 내에서 관계를 만들 필요 없이, 파라미터를 통해 외부에서 생성된 객체를 가져올 수 있음.

**객체 간의 다이나믹한 관계**

- **클래스 간의 관계**: 코드에서 다른 클래스 이름이 나타남.
- **객체 간의 관계**: 특정 클래스를 몰라도 해당 클래스가 구현한 인터페이스를 사용하여 객체를 다룰 수 이씀.

**객체지향 프로그래밍에서의 다형성**

UserDao 객체가 `DConnectionMaker` 객체를 사용하려면 두 객체 사이에 의존 관계를 설정해야 한다. 이 역할은 UserDao의 클라이언트가 수행한다.

- **클라이언트 역할**: 클라이언트는 ConnectionMaker 구현 클래스를 선택하고 해당 객체를 생성하여 UserDao와 연결.

- **UserDao 코드 예시**
    
    ```java
    public class UserDao {
        private ConnectionMaker connectionMaker;
    
        public UserDao(ConnectionMaker connectionMaker) {
            this.connectionMaker = connectionMaker;
        }
    }
    
    ```
    
    - **변경사항**: 생성자에 ConnectionMaker 타입의 객체를 전달하는 파라미터를 추가하면, UserDao에서 `DConnectionMaker`에 대한 의존성이 사라집니다.
    
- **UserDaoTest 코드 예시**
    
    ```java
    public class UserDaoTest {
        public static void main(String[] args) {
            ConnectionMaker connectionMaker = new DConnectionMaker();
            UserDao dao = new UserDao(connectionMaker); // 구현 클래스를 결정하고 객체를 생성
        }
    }
    
    ```
    
    - **변경사항**: 이제 UserDao를 사용하는 Main에서 ConnectionMaker 객체를 정의하고 이를 UserDao로 전달하게 되었습니다. 이를 통해 UserDao에 불필요한 관심사와 책임을 클라이언트로 떠넘기는 작업이 완료되었습니다.
    

**구조적 정리**

이 상황을 구조도로 나타내면 다음과 같다:

![클래스 사이에 불필요한~](https://github.com/user-attachments/assets/a5653258-5321-48d8-9359-2e3c80e6fc58)


- UserDao의 클라이언트인 **`UserDaoTest`**에서 ConnectionMaker 객체인 **`DConnectionMaker`**를 생성한 후 이를 UserDao에 제공하고, UserDao는 ConnectionMaker 타입의 **`DConnectionMaker`**를 사용
- 이는 관심의 분리를 이루어냈으며, 상속보다 깔끔하고 유연한 방법임!

### 1.3.4 원칙과 패턴

#### **개방-폐쇄 원칙**

- 클래스와 모듈은 확장에는 열려 있어야 하고, 변경에는 닫혀 있어야 한다.
- 이 원칙은 시스템의 유연성과 유지보수성을 높이는 데 중요하다.

#### **높은 응집도와 낮은 결합도**

- 높은 응집도:
    - 하나의 모듈이나 클래스가 하나의 책임 또는 관심사에 집중해야 한다.
    - 공통 관심사는 하나의 클래스에 모아야 한다.
- 낮은 결합도:
    - 서로 다른 책임과 관심사를 가진 객체 또는 모듈 간의 연결은 느슨하게 유지해야 한다.
    - 느슨한 결합: 관계를 유지하는 데 꼭 필요한 최소한의 방법만 제공하고, 나머지는 서로 독립적이어야 한다.

#### **전략 패턴**

- 특정 기능의 맥락(context)에서 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 외부로 분리한다.
- 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 교체하여 사용할 수 있도록 한다.
- 이 패턴은 알고리즘의 유연성을 높이고 코드의 재사용성을 향상시킨다. <br>

## 1.4 제어의 역전(IoC)

🙌 IoC라는 약자로 많이 사용되는 **제어의 역전** *(Inversion of Control)* 이라는 용어에 대해 알아보고 UserDao 코드를 더 개선해보자!

### 1.4.1 오브젝트 팩토리

🤔 **`UserDaoTest`**는 원래 테스트를 위해 만든 클래스이다. 엉겁결에 떠맡은 구현 클래스를 결정하는 기능을 분리해보자.

- UserDao와 ConnectionMaker 구현 클래스의 오브젝트를 만드는 것
- 두 개의 오브젝트가 연결되어 사용될 수 있또록 관계를 맺어주는 것

#### **팩토리**

💡 팩토리란?

- 객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 돌려주는 역할을 담당하는 오브젝트

- UserDao의 생성 책임을 맡은 팩토리 클래스
    
    ```java
    public class DaoFactory{
        public UserDao userDao() {
            ConnectionMaker connectionMaker = new DConnectionMaker();
            UserDao userDao = new UserDao(connectionMaker);
            return userDao;
        }
    }
    ```
    

- 수정된 UserDaoTest
    
    ```java
    public class UserDaoTest {
        public static void main(String[] args) {
           UserDao dao = new DaoFactory().userDao();
            ...
        }
    }
    ```
    

#### **설계도로서의** **팩토리**

- UserDao, ConnectionMaker - 핵심 데이터 로직, 기술 로직
- DaoFactory - 플리케이션 오브젝트 구성 및 관계 정의
    
   ![오브젝트 팩토리를 활용한 구조](https://github.com/user-attachments/assets/9153ad99-6bb8-4743-b5c8-ea8f94e7c751)

    

### 1.4.2 오브젝트 팩토리의 활용

🤔 UserDao가 아닌 다른 DAO의 생성 기능을 DaoFactory에 넣으면 어떻게 될까?

- ⚠️:  ConnectionMaker 구현 클래스를 선정하고 생성하는 코드가 중복되게 됨.
- 💡: 새로 만든 ConnectionMaker 생성용 메소드를 이용하도록 수정한다!

### 1.4.3 제어권의 이전을 통한 제어관계 역전

😮 **제어의 역전?**

- 제어 흐름의 개념을 거꾸로 뒤집는 것
    - 오브젝트가 자신이 사용할 오브젝트를 스스로 선택하지도 않고 생성하지도 않음.
    - 자신이 어떻게 만들어지고, 어떻게 사용되는지도 알 수 없음.
        - 모든 제어 권한을 다른 사람에게 위임하기 때문임. <br>

## 1.5 스프링의 IoC

### 1.5.1 오브젝트 팩토리를 이용한 스프링 IoC

#### **애플리케이션 컨텍스트와 설정정보**

😮 DaoFactory를 스프링에서 사용가능하도록 변신시켜보자!

- 빈(Bean)
    - 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트
    - 자바빈 또는 엔터프라이즈 자바빈에서 말하는 빈과 비슷한 오브젝트 단위의 애플리케이션 컴포넌트
- 스프링 빈(Spring Bean)
    - 스프링 컨테이너가 생성한 관계설정, 사용 등을 제어해주는 제어의 역전이 적용된 오브젝트를 가리키는 말
- 빈 팩토리(Bean Factory)
    - 스프링에서 빈의 생성과 관계설정 같은 제어를 담당하는 IoC 오브젝트
- 애플리케이션 컨텍스트(Application Context)
    - IoC 방식을 따라 만들어진 일종의 빈 팩토리
    - 

📌 **빈 팩토리**와 **애플리케이션 컨텍스트**는 동일하다고 생각하면 됨.

- 빈 팩토리(Bean Factory)
    - 빈을 생성하고 관계를 설정하는 IoC의 기본 기능에 초점
- 애플리케이션 컨텍스트(Application Context)
    - 애플리케이션 전반에 걸쳐 모든 구성요소 제어 작업을 담당하는 IoC 엔진이라는 의미가 좀 더 부각

#### **DaoFactory를 사용하는 애플리케이션 컨텍스트**

🤔 DaoFactory를 스프링의 빈 팩토리가 사용할 수 있는 설정정보로 만들어보자!

1️⃣ @Configuration 어노테이션 추가

2️⃣ @Bean 어노테이션 추가

두 가지 어노테이션으로 스프링 프레임워크의 빈 팩토리 또는 어플리케이션 컨텍스트가 IoC 방식의 기능을 제공할때 사용할 완벽한 설정정보가 된다.

- 스프링 빈 팩토리가 사용할 설정정보를 담은 DaoFactory 클래스
    
    ```java
    @Configuration // 객체 생성 담당 설정 클래스
    public class DaoFactory {
        
        @Bean // 객체 생성 담당
        public UserDao userDao(){
            return new UserDao(connectionMaker());
        }
    
        @Bean // 객체 생성 담당
        public ConnectionMaker connectionMaker(){
            return new DConnectionMaker();
        }
    }
    ```
    

🤔 이제 DaoFactory를 설정정보로 사용하는 어플리케이션 컨텍스트를 만들어보자!

➡️ @Configuration 클래스를 설정정보로 사용하려면 AnnotationConfigApplicationContext를 이용하면 된다.

- 애플리케이션 컨텍스트를 적용한 UserDaoTest
    
    ```java
    public class UserDaoTest {
        public static void main(String[] args) throws ClassNotFoundException, SQLException {
    
            ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
            UserDao dao = context.getBean("userDao",UserDao.class); //빈의 이름은 메서드명
    
           ...
        }
    }
    ```
    

### 1.5.2 애플리케이션 컨텍스트의 동작방식

😮 기존에 오브젝트 팩토리를 이용했던 방식과 스프링 애플리케이션 컨텍스트를 사용한 방식을 비교해보자!

- 오브젝트 팩토리가 스프링의 애플리케이션 컨텍스트에 대응 된다.
    
    ![애플리케이션 컨텍스트가 동작하는 방식](https://github.com/user-attachments/assets/d99f734a-91e3-4f35-b0fe-70104805cc91)


📌 **오브젝트 팩토리와 비교했을 때, 애플리케이션 컨텍스트를 사용하면 얻을 수 있는 장점**

1. 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다.
    - 일관된 방식의 원하는 오브젝트도 얻을 수 있음.
2. 애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다.
    - 오브젝트 사이의 관계 설정 뿐 만 아니라, 오브젝트를 효과적으로 활용할 수 있는 다양한 기능 제공
        - 오브젝트가 만들어지는 방식, 시점과 전략 설정
        - 자동생성, 오브젝트에 대한 후처리, 정보의 조합, 설정 방식의 다변화, 인터셉팅 등
3. 빈을 검색하는 다양한 방법을 제공한다.

### 1.5.3 스프링 IoC의 용어 정리

**빈**

- 스프링이 IoC 방식으로 관리하는 오브젝트.
- 관리되는 오브젝트
    - 스프링이 직접 그 생성과 제어를 담당하는 오브젝트.

**빈 팩토리**

- 스프링의 IoC를 담당하는 핵심 컨테이너.
- 빈을 관리하는 기능을 담당.
    - 빈 등록/생성/조회/반환 등

**애플리케이션 컨텍스트**

- 빈 팩토리를 확장한 IoC 컨테이너.
- 빈 팩토리의 기능에서 스프링이 제공하는 각종 부가 서비스를 추가적으로 제공함.

**설정정보/설정 메타정보**

- 애플리케이션 컨택스트 혹은 빈 팩토리가 IoC를 적용하기 위해 사용하는 메타정보.
    - IoC 컨테이너에 의해 관리되는 애플리케이션 오브젝트를 생성하고 구성할 때 사용함.

**컨테이너 또는 IoC 컨테이너**

- IoC 방식으로 빈을 관리하는 것. 애플리케이션 컨텍스트나 빈 팩토리를 컨테이너 혹은 IoC 컨테이너라고 함.
- 대신 애플리케이션 컨텍스트보다 좀 더 추상적인 표현. <br>


## **1.6 싱글톤 레지스트리와 오브젝트 스코프**

---

> 😮 **DaoFactory를 직접 사용하는 것과 스프링의 애플리케이션 컨텍스트를 통해 사용하는 것에는 차이점이 있다. 과연 무엇일까?**
> 
- **DaoFactory의 userDao()를 여러 번 호출했을 때:**
    - 오브젝트 팩토리 방식: 매번 각각 다른 UserDao 객체가 출력됨.
    - 스프링 애플리케이션 컨텍스트 방식: **매번 같은** UserDao 객체가 출력됨.

➡️ 왜 그럴까?

### 1.6.1 **싱글톤 레지스트리로서의 애플리케이션 컨텍스트**

#### **서버 어플리케이션과 싱글톤**

- **이유**: 스프링은 주로 서버 환경을 다루기 때문에, 클라이언트 요청마다 객체를 새로 만드는 것은 서버에 부하를 준다.
- **해결책**: 서블릿을 이용해 클래스당 하나의 객체를 만들어 여러 요청에 대해 공유하여 사용한다.
    
    > 🤔 **싱글톤 패턴**
    > 
    > 
    > **정의**: 특정 클래스의 객체가 어플리케이션 내에서 하나만 존재하도록 강제하는 디자인 패턴.
    > 
    > **장점**: 자원 관리 효율성 증가.
    > 

#### **싱글톤 패턴의 한계**

- **구현 방법**:
    - 생성자를 `private`으로 설정하여 외부 객체 생성을 방지.
    - static 필드에 싱글톤 객체를 저장.
    - `getInstance()` 메서드를 통해 객체를 반환.
- **문제점**:
    1. **상속 불가**: private 생성자로 인해 상속할 수 없다.
    2. **테스트 어려움**: mock 객체로 대체하기 힘들다.
    3. **서버 환경의 불안정성**: 클래스 로더에 따라 여러 객체가 생성될 수 있다.
    4. **전역 상태 문제**: 자유롭게 접근할 수 있어 바람직하지 않다.

#### **싱글톤 레지스트리**

- **스프링의 해결책**: 싱글톤 형태의 객체 생성을 관리하는 **싱글톤 레지스트리** 제공.
- **장점**:
    - 평범한 자바 클래스를 싱글톤으로 활용 가능.
    - IoC 컨테이너를 통해 관리 용이.
    - public 생성자를 가질 수 있어 테스트 환경에서 유연함.
    - 객체지향 설계와 디자인 패턴 적용에 제약이 없다.
    - 

➡️ 스프링은 IoC 컨테이너이자 싱글톤 레지스트리로써, 효율적인 객체 관리를 지원한다.

### 1.6.2 **싱글톤과 오브젝트의 상태**

> 🥲 **멀티스레드 환경에서 싱글톤은 여러 스레드가 동시에 접근하여 사용할 수 있다. 따라서 상태 관리에 주의를 기울여야 한다.**
> 

- 싱글톤이 멀티스레드 환경에서 사용되는 경우라면 상태 정보를 내부에 갖고 있지 않은 무상태(stateless) 방식으로 만들어져야 한다.
    - 서로 값을 덮어쓰고 자신이 저장하지 않은 값을 읽어올 수 있기 때문!
        - 따라서 인스턴스 변수에 **매번 새로운 값으로 바뀌는 정보를 담는 변수를 지정하면 심각한 문제가 발생**한다.
        - 대신 읽기 전용 정보를 담고 있는 인스턴스 변수는 사용은 괜찮다!

🤔 그렇다면… stateless한 상태에서 각 **요청 정보**나, **DB** or **서버의 리소스로부터 생성한 정보**는 어떻게 다뤄야 할까?

- 파라미터, 로컬 변수, 리턴 값 등을 사용한다!
    - 메소드 안에서 생성되는 로컬 변수는 매번 새로운 값을 저장할 독립적인 공간이 만들어지기 때문임.
- 클래스 인스턴스 변수로 만약 스프링 빈과 같이 싱글톤 빈을 저장하려는 용도라면 사용해도 괜찮음.
    
    : 읽기 전용이고 애초에 오브젝트가 한 개만 만들어지기 때문
    

### 1.6.3 **스프링 빈의 스코프**

> ✨ 빈의 스코프?
> 
> - 빈이 생성되고, 존재하고, 적용되는 범위 <br>


## 1.7 **의존관계 주입(DI)**

### **1.7.1 제어의 역전(IoC)과 의존관계 주입**

> ✨ 스프링의 IoC 컨테이너?
> 
> - 객체를 생성하고 관계를 맺어주는 등의 작업을 담당하는 기능을 일반화한 것.

⚠️ 스프링이 제공하는 IoC 방식의 핵심 = **"의존 관계 주입 (Dependency Injection)"**

- 따라서 초기에는 IoC 컨테이너라고 불리더 스프링이 최근엔 의존관계 주입 컨테이너 또는 DI 컨테이너라고 더 많이 불림.

### **1.7.2 런타임 의존관계 설정**

📌 스프링 IoC 기능의 대표적인 동작원리는 주로 의존관계 주입이라고 불린다.

- 의존관계 주입?
    - 구체적인 오브젝트와 클라이언트를 런타임 시에 연결해주는 작업
- 의존관계 주입의 핵심
    - 설계 시점에는 알지 못했던 두 오브젝트의 관계를 맺도록 도와주는 제 3의 존재가 있다는 것
    - 스프링의 애플리케이션 컨텍스트, 빈 팩토리, IoC 컨테이너 등이 모두 외부에서 오브젝트 사이의 런타임 관계를 맺어주는 책임을 지닌 제 3의 존재라고 볼 수 있다!

### **1.7.3 의존관계 검색과 주입**

😮 **스프링이 제공하는 IoC 방법에는 의존관계 주입만 있는 것이 아님!**

- 의존관계를 맺는 방법이 외부로부터의 주입이 아니라 스스로 검색을 이용하기 때문에 의존관계 검색(dependency Lookup)이라고 불리는 것도 있다.
- 의존관계 검색은 런타임 시 의존관계를 맺을 오브젝트를 결정하는 것과 오브젝트의 생성 작업은 외부 컨테이너에게 IoC로 맡기지만, 이를 가져올 때는 메소드나 생성자를 통한 주입 대신 스스로 컨테이너에게 요청하는 방법을 사용한다.

```java
public UserDao()  {
	AnnotationConfigApplicationContext context =
		new AnnotationConfigApplicationContext(DaoFactory.class);
	this.connectionMaker = context.getBean("connectionMaker",  ConnectionMaker.class);

}
```

### **1.7.4 의존관계 주입의 응용**

- 기능 구현의 교환
    - 개발환경과 운영환경에서 DI 설정정보에 해당하는 DaoFactory만 다르게 만들어두면 나머지 코드에는 전혀 손대지 않고 개발 시와 운영 시에 각각 다른 런타임 오브젝트에 의존관계를 갖게 해준다.
- 부가 기능 추가
    - DI의 장점은 관심사의 분리(SOC)를 통해 얻어지는 높은 응집도에서 나온다.

### **1.7.5 메소드를 이용한 의존관계 주입**

- **수정자 메소드를 이용한 주입:**
    - 수정자(Setter) 메소드는 외부에서 오브젝트 내부의 애트리뷰트 값을 변경하려는 용도로 주로 사용한다.

- **일반 메소드를 이용한 주입:**
    - 수정자 메소드처럼 set으로 시작해야 하고 한 번에 한 개의 파라미터만 가질 수 있다는 제약이 싫다면 여러 개의 파라미터를 갖는 일반 메소드를 DI용으로 사용할 수 있다. <br>

    ## 1.8  **XML을 이용한 설정**

### 1.8.1 XML 설정

- **connectionMaker() 전환**
    
    
    |  | **자바 코드 설정정보** | **XML 설정정보** |
    | --- | --- | --- |
    | 빈 설정파일 | @Configuration | <beans> |
    | 빈의 이름 | @Bean methodName() | <bean id="methodName"> |
    | 빈의 클래스 | return new BeanClass(); | class = "a,b,c ... BeanClass> |
    

### 1.8.2 **XML을 이용하는 애플리케이션 컨텍스트**

- XML 설정파일 이름: applicationContext.xml
    
    ```java
    ApplicationContext context = new GenerixXmlApplicationContext("applicationContext.xml");
    ```
    

### 1.8.3 **DataSource 인터페이스로 변환**

- **DataSource 인터페이스 적용**
    
    ```java
    public interface DataSource extends CommonDataSource, Wrapper {
      Connection getConnection() throws SQLException;
    }
    ```
    

### 1.8.4 **프로퍼티 값의 주입**

#### **값 주입**

- 수정자 메소드를 호출해서 DB 연결을 넣는 경우 코드와 XML로 표현할 수 있다.

#### **value 값의 자동 변환**

- value에 지정한 텍스트 값은 적절한 자바 타입으로 변환할 수 있다.
    - Integer, Double, String, Boolean 뿐 만 아니라 Class, URL, File 등의 오브젝트로도 변환이 가능하다.
