
# 1주차 토비의 스프링 3.1 


## 1장 오브젝트와 의존관계


스프링은 객체지향(_object-oriented_)을 추구한다. 때문에 오브젝트는 스프링의 관심사 1순위가 될 수밖에 없다. 
스프링의 추구미로 인해, 프레임워크의 의존율을 낮추며 테스트의 용이성을 갖고 OOP의 기본 원칙을 따를 수 있는 POJO(Plain Old Java <u>Object</u>)를 지향하기도 한다. 

![](https://velog.velcdn.com/images/ykky2115/post/caa441b4-ee7d-402e-a447-6691bf2303f3/image.png)

---
#### 이번 장에서는 여러 오브젝트 중, DAO(Data Access Object)의 사례를 보면서 오브젝트와 의존관계란 무엇인지 알아볼 것이다. 
</br>

### DAO

종종 DAO와 DTO가 헷갈리고는 한다. 

DAO는 Data(DB)에 Access하는 권리를 갖고 있는 오브젝트이고 
DTO는 Data Transfer(모든 계층에 아우르는) 오브젝트라고 생각한다. 

![](https://velog.velcdn.com/images/ykky2115/post/114d6fe0-0f28-404e-8ff9-a693a5e40b03/image.png)

![](https://velog.velcdn.com/images/ykky2115/post/edf3b690-6611-47b4-94af-694ddd694842/image.png)

두번째이미지에선 Domain -> DAO

    여기서 잠깐! 자바빈이라는 예전에는 비주얼 컴포넌트이었다가 엔터프라이즈가 나오면서 
    현재는 어떠한 관례를 따라 만들어진 오브젝트로 불리는 개념이 있다. 
    관례는 다음과 같다. 
    기본 생성자가 있고 모든 속성은 private이지만 각 속성에 getter,setter이 있다. 
    현재에 들어서 자바빈은 DTO와 흡사한 개념이라고 할 수 있다.
    

 
###  좋은 DAO란

기능하나 영 좋지 않은 DAO 코드를 소개하겠다. 

```
// 권고사직을 노리는 코드
public class UserDao {
	public void add(User user) throws ClassNotFoundException, SQLException {
    	Class.forName("com.mysql.jdbc.Driver");
        Connection c = DriverManager.getConnection(
        	"jdbc:mysql://localhost/springbook", "spring", "book");
            
        PreparedStatement ps = c.prepareStatement(
        	"insert into users(id, name, pwd) values(?,?,?)");
        ps.setString(1, user.getId());
        
        ps.executeUpdate();
        
        ps.close();
        c.close();
    }
    
    public User get(String id) throws ClassNotFoundException, SQLException {
    	Class.forName("com.mysql.jdbc.Driver");
        Connection c = DriverManager.getConnection(
        	"jdbc:mysql://localhost/springbook", "spring", "book");
        
        PreparedStatement ps = c.prepareStatement(
        "selece * from users when id=?");
        
        ps.setString(1,id);
        
        ResultSet rs = ps.executeQuery();
        rs.next();
        User user = new User();
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setPwd(rs.getString("pwd"));
        
        rs.close();
        ps.close();
        c.close();
        
        return user;
}
```

이것은 다중의 기능이 한 가지 메서드에 있는 경우다. 이렇게 된다면 add()의 기능이 바뀌면 바뀌어야 하지 않는 기능도 바뀌어야 하거나 코드가 복잡하여 유지보수성을 떨어트린다. 
**-> 이때 객체지향 등장!**

오브젝트를 분리 및 확장하는 것이다. 
꽁꽁 묶인 UserDao의 기능 순서를 살펴보자.

1. DB 연결을 위한 Connection을 가져온다.
2. SQL을 담은 (Pre)Statement를 만든다.
3. Statement를 실행한다.
4. 조회의 경우 실행된 결과를 ResultSet로 받아서 정보를 저장할 오브젝트(User)에 옮긴다.
5. 작업 중 생성된 Connection, Statement, ResultSet 같은 리소스는 작업을 마친 후 닫는다.
6. JDBC API가 만들어내는 예외를 잡아서 직접 처리 혹은 메소드에 throws를 선언하여 메소드 밖으로 던진다. 

### DAO의 분리

#### 관심사의 분리 _Seperation of Concerns_

각 기능의 관심사를 머리/몸통/다리로 나누어보겠다.

- [머리] DB와 연결을 위한 커넥션을 어떻게 가져올까. 라는 관심. 
세분화한다면, 어떤 DB를 쓰고 어떤 드라이버를 사용하고 어떤 로그인 정보를 쓰는데, 그 커넥션을 생성하는 방법은 또 어떤 것이라다라는 것까지 관심을 꼬옥 갖고 구분할 수 있다. ==> DB 연결과 관련된 필자의 관심과 사랑. 

- [몸통] 사용자 등록을 위한 DB에 보낼 SQL statement를 만들고 실행하는 관심. 파라미터로 넘어온 사용자 정보를 Statement에 바인딩하고, 담긴 SQL을 DB를 통해 실행시키는 방법이다. (물론 파라미터를 바인딩하고 which SQL인지도 다른 관심사로 분리할 수 있다.) ==> SQL Statement를 만들고 실행하는 필자의 관심과 사랑.

- [다리] 리소스인 것들을 닫음으로써 소중한 공유 리소스를 시스템에 돌려주는 것이다. ==> 소중히 메소드를 끝내는 필자의 관심과 사랑.


#### 1. 중복 코드의 메소드 추출
앞서 타이핑하면서도 느꼈지만 DB 연결하는 것이 중복되었다. 이를 독립적인 메소드로 분리한다. 
```
public void add(User user) {}
public void get(String id) {}

//커넥션 분리
private Connection getConnection() throws ClassNotFoundException, SQLException {
	Class.forName("com.mysql.jdbc.Driver");
    Connection c = DriverManager.getConnection(
        	"jdbc:mysql://localhost/springbook", "spring", "book");
    return c; 
}
```

#### 2. 상속을 통한 확장 
UserDao의 니즈가 높아졌다. UserDao라는 기술을 팔아야 하지만 해당 코드를 유출하고 싶지 않다.
=> 그럴 때, UserDao를 한 단계 더 분리하는 것이다. 앞선 getConnection()을 추상 메소드로 바꾼다. getConnection의 코드는 볼 수 없지만 메소드 자체는 존재한다. 그리고 사용자는 UserDao를 상속하여 쓴다.

![](https://velog.velcdn.com/images/ykky2115/post/8a24d6c8-9cb4-413f-8630-6eaa3e71fd85/image.png)
한편 이것을 템플릿 메소드 패턴이라고 한다. 
만약 여기에 더해 getConnection()을 분리한다면 그것은 팩토리 메소드 패턴이 될 것이다. 

![](https://velog.velcdn.com/images/ykky2115/post/ee078088-9232-4e29-855d-4ece7c2c2642/image.png)

### DAO의 확장

#### 1. 클래스의 분리
앞서 메소드를 분리하거나 추상화하여 상속관계로 만들었다. 이번에는 완전히 독립적인 클래스로 분리할 것이다. 
이렇게 하면 더 이상 UserDao는 abstract 클래스일 필요가 없다. 

근데 이렇게 되면 초반 목적이었던 UserDao를 볼 수 없게 한다.가 안된다. 
두 번째 문제는 UserDao라는 클래스가 다른 클래스의 정보를 안다. 
(=UserDao가 SimpleConnectionMaker라는 새로운 클래스에 종속됨)

정보 은닉이 안 되는 것이다.

#### 2. 인터페이스의 도입
인터페이스는 자신을 구현한 클래스에 대한 구체적인 정보는 모두 감춰버린다(정보의 은닉). 
따라서, interface ConnectionMaker으로 만든다. 

이제 UserDao는 구체적인 클래스 정보를 알 필욘 없지만 여전히 connectionMaker라는 오브젝트를 만들면서 클래스 이름이 나온다.
```
public class UserDao {
	private ConnectionMaker connectionMaker;
    
    public UserDao() {
		connectionMaker = new DConnectionMaker();
	}
}
```

#### 3. 관계설정 책임의 분리
관계 설정 책임의 분리란 객체 간 관계를 설정하는 책임을 별도의 객체나 컴포넌트에 위임하는 것이다.
단일 책임 원칙(SRP)을 따르도록 돕는다.
위 코드를 없애야 종속성이 없어진다.
```
public class UserDao {
	private ConnectionMaker connectionMaker;
    
	public UserDao(ConnectionMaker connectionMaker) {
    	this.connectionMaker = connectionMaker);
    }
}
```
실제로 개발하면서 많이 쓰이는 패턴이다. 
더 예시를 들자면..
```
//관계설정 무책임 코드
public class OrderService {
	private PaymentProcessor paymentProcessor;
    private InventoryManager inventoryManager;
    
    public OrderService() {
    	this.paymentProcessor = new PaymentProcessor();
        this.inventoryManager = new InventoryManager();
    }
    
    
 
//관계설정 책임가장 코드
public class OrderService {
	private PaymentProcessor paymentProcessor;
    private InventoryManager inventoryManager;
    
    public OrderService(PaymentProcessor paymentProcessor, InventoryManager inventoryManager) {
        this.paymentProcessor = paymentProcessor;
        this.inventoryManager = inventoryManager;
    }
    
//분리된 관계 설정을 담당하는 팩토리 또는 컨테이너
public class ServiceFactory {
    public static OrderService createOrderService() {
        PaymentProcessor paymentProcessor = new PaymentProcessor();
        InventoryManager inventoryManager = new InventoryManager();
        return new OrderService(paymentProcessor, inventoryManager);
    }
}

//활용 예시
public class Main {
    public static void main(String[] args) {
        OrderService orderService = ServiceFactory.createOrderService();
        Order order = new Order();
        orderService.processOrder(order);
    }
}
```

이것은 스프링 프레임워크에 적용하면 IoC컨테이너를 통해 다음과 같이 구현한다. 
```
@Service
@RequiredArgsConsturctor
public class OrderService {
    private final PaymentProcessor paymentProcessor;
    private final InventoryManager inventoryManager;

    public OrderService(PaymentProcessor paymentProcessor, InventoryManager inventoryManager) {
        this.paymentProcessor = paymentProcessor;
        this.inventoryManager = inventoryManager;
    }
}
```
내가 직접 컨테이너 코드를 작성하던 것과 달리 스프링에서 IoC 컨테이너를 이용하면 이 친구가 관계설정책임을 담당하고 필요한 의존성을 주입(Dependency Injection)한다.

#### 4. 원칙과 패턴 
- OCP(개방 폐쇄 원칙) -높은 응집도와 낮은 결합도를 추구한다.
- 전략 패턴

### 제어의 역전(IoC)
일반적인 제어의 흐름이라는 것이 있다. 프로그램의 시작 시점에서 다음에 쓸 오브젝트를 결정, 생성하고 오브젝트에 있는 메소드를 호출하고, 그 메소드 안에서 다음에 쓸 것을 모색하고 호출하는 작업의 방식이 반복되는 것이 그것이다. 
그리고 제어의 역전은 이 흐름의 개념을 반대로 진행하는 것이다. 
오브젝트는 자신이 쓸 오브젝트를 선택/생성하지 않는다. 자신이 언제 불려지고 어떻게 쓰여지고 만들어지는 지도 알지 못 한다. 

<u>모든 제어 권한은 자신이 아닌 다른 사람에게 위임하기 때문이다.</u>


여태 말했던 오브젝트는 스프링의 빈이다. 
빈은 application context에서 주로 사용한다. 
애플리케이션 컨텍스트의 동작방식은 다음과 같다.

![](https://velog.velcdn.com/images/ykky2115/post/08388402-b032-4622-948e-2d4180ae0a89/image.png)

