# 5장. 서비스 추상화

✨ 지금까지 만든 DAO에 트랜잭션을 적용해보면서 스프링이 어떻게 성격이 비슷한 여러 종류의 기술을 추상화하고 이를 일관된 방법으로 사용할 수 있도록 지원하는지 살펴보자!

<br>

## 5.1 사용자 레벨 관리 기능 추가

🤔 UserDao는 CRUD 작업만 가능하고 비즈니스 로직을 가지고 있지 않다. UserDao에 간단한 비즈니스 로직을 추가해보자!

- 다수의 회원이 가입할 수 있는 인터넷 서비스의 사용자 관리 모듈에 적용해보자.

- 인터넷 서비스의 사용자 관리 기능에서 구현해야 할 비즈니스 로직
    1. 사용자의 레벨은 BASIC, SILVER, GOLD 세 가지 중 하나다.
    2. 사용자가 처음 가입하면 BASIC 레벨이며, 이후 활동에 따라 한 단계씩 업그레이드 될 수 있다.
    3. 가입 후 50회 이상 로그인을 하면 BASIC에서 SILVER 레벨이 된다.
    4. SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨이 된다.
    5. 사용자 레벨의 변경 작업은 일정한 주기를 가지고 일괄적으로 진행된다. 변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않는다.


<br>
<br>

### 5.1.1 필드 추가

#### Level 이늄

- 먼저, User 클래스에 사용자의 레벨을 저장할 필드를 추가하자.
    - 각 레벨을 코드화해서 범위가 작은 숫자로 관리하면 DB 용량도 많이 차지하지 않고 가벼워서 좋다.

- 그럼 자바의 User에 추가할 프로퍼티 타입도 숫자로 하면 될까?
    - 의미 없는 숫자를 프로퍼티에 사용하면 타입이 안전하지 않아서 위험하기 때문에 좋지 않다!

- 상수 값을 정하고 int 타입으로 레벨을 사용한다고 가정해보자
    
    ```java
    public class User {
        ...
        int level;
        private static final int BASIC = 1;
        private static final int SILVER = 2;
        private static final int GOLD = 3;
    
        public void setLevel(int level){
            this.level = level;
        }
    }
    ```
    

- 의미 있는 상수를 정의했기 때문에 아래처럼 깔끔하게 코드를 작성할 수 있다.
    
    ```java
    if (user1.getLevel() == User.BASIC) {
    	user1.setLevel(User.SILVER);
    }
    ```
    

- 다만, 두 가지 문제점이 있다.
    1. level의 타입이 int이기 때문에 다른 종류의 정보를 넣는 실수를 해도 컴파일러가 체크하지 못한다.
        - **`user1.setLevel(other.getSum());`**
    2. 범위를 벗어나는 값을 넣을 위험도 있다.
        - **`user1.setLevel(1000);`**

- 따라서 숫자 타입을 직접 사용하기 보다는 **enum을 이용**하는 게 안전하고 편리하다.
    
    ```java
    public enum Level {
        BASIC(1), SILVER(2), GOLD(3);
    
        private final int value;
    
        Level(int value){
            this.value = value;
        }
    
        public int intValue(){ // 값을 가져오는 메소드
            return value;
        }
    
        public static Level valueOf(int value){ // 값으로부터 Level 타입 오브젝트를 가져오도록 만든 스태틱 메소드
            switch (value){
                case 1: return BASIC;
                case 2: return SILVER;
                case 3: return GOLD;
                default: throw new AssertionError("Unknown value : " + value);
            }
        }
    }
    ```
    
    - 이렇게 만들어진 Level enum은 내부에는 DB에 저장할 int 타입의 값을 갖고 있지만, 겉으로는 Level 타입의 오브젝트이기 때문에 안전하게 사용할 수 있다.

<br>

#### User 필드 추가

- Level 타입의 변수, 로그인 횟수, 추천수를 User 클래스에 추가하자.
    - 로그인 횟수와 추천수는 단순한 int 타입으로 만들어도 된다.
        
        ```java
        public class User {
            ...
            Level level;
            int login;
            int recommend;
        
            public Level getLevel() { // 레벨을 가져오는 메소드
                return level;
            }
        
            public void setLevel(Level level) { // 레벨을 설정하는 메소드
                this.level = level;
            }
                ...
            // login, recommend, getter/setter 생략
        }
        ```
        

- USER 테이블에도 필드를 추가한다
    
    
    | 필드명 | 타입 | 설정 |
    | --- | --- | --- |
    | Level | tinyint | Not Null |
    | Login | int | Not Null |
    | Recommend | int | Not Null |

<br>

#### UserDaoTest 테스트 수정

✨ UserDaoJdbc와 테스트에도 필드를 추가해야 한다.

1. 테스트 픽스처로 만든 user1, user2, user3에 새로 추가된 새 필드의 값을 넣는다.
    
    ```java
    public class UserDaoTest {
    	@Autowired UserDao dao; 
    	@Autowired DataSource dataSource;
    	
    	private User user1;
    	private User user2;
    	private User user3; 
    	
    	@Before
    	public void setUp() {
    		this.user1 = new User("gyumee", "박성철", "springno1", Level.BASIC, 1, 0);
    		this.user2 = new User("leegw700", "이길원", "springno2", Level.SILVER, 55, 10);
    		this.user3 = new User("bumjin", "박범진", "springno3", Level.GOLD, 100, 40);
    	}
    ```
    

1. 이에 맞게 User 클래스 생성자의 파라미터도 추가해준다.
    
    ```java
    public class User {
    	...
    	
    	public User(String id, String name, String password, Level level,
    			int login, int recommend) {
    		this.id = id;
    		this.name = name;
    		this.password = password;
    		this.level = level;
    		this.login = login;
    		this.recommend = recommend;
    	}
    ```
    

1. 두 개의 User 오브젝트 필드 값이 모두 같은지 비교하는 checkSameUser() 메소드를 수정한다.
    
    ```java
    private void checkSameUser(User user1, User user2) {
    		assertThat(user1.getId(), is(user2.getId()));
    		assertThat(user1.getName(), is(user2.getName()));
    		assertThat(user1.getPassword(), is(user2.getPassword()));
    		assertThat(user1.getLevel(), is(user2.getLevel()));
    		assertThat(user1.getLogin(), is(user2.getLogin()));
    		assertThat(user1.getRecommend(), is(user2.getRecommend()));
    	}
    ```
    

1. User 오브젝트를 비교하는 로직을 일정하게 유지할 수 있도록 수정한다.
    
    ```java
    @Test 
    	public void andAndGet() {		
    		dao.deleteAll();
    		assertThat(dao.getCount(), is(0));
    
    		dao.add(user1);
    		dao.add(user2);
    		assertThat(dao.getCount(), is(2));
    		
    		User userget1 = dao.get(user1.getId());
    		**checkSameUser(userget1, user1);**
    		
    		User userget2 = dao.get(user2.getId());
    		**checkSameUser(userget2, user2);**
    	}
    ```
    

<br>

#### UserDaoJdbc 수정

✨ 미리 준비된 테스트가 성공하도록 UserDaoJdbc 클래스를 수정하자!

- add() 메소드의 SQL과 User 오브젝트 매핑용 콜백인 userMapper에 추가된 필드를 넣는다.
    
    ```java
    public class UserDaoJdbc implements UserDao {
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
    				user.setLevel(Level.valueOf(rs.getInt("level")));
    				user.setLogin(rs.getInt("login"));
    				user.setRecommend(rs.getInt("recommend"));
    				return user;
    			}
    		};
    
    	public void add(User user) {
    		this.jdbcTemplate.update(
    				"insert into users(id, name, password, level, login, recommend) " +
    				"values(?,?,?,?,?,?)", 
    					user.getId(), user.getName(), user.getPassword(), 
    					user.getLevel().intValue(), user.getLogin(), user.getRecommend());
    	}
    ```
    
    - 여기서 눈여겨볼 것은 Level 타입의 level 필드를 사용하는 부분이다.
        - **DB에 저장할 때**: **`Level`** ➔ **`int`**
        - **DB에서 조회할 때**: **`int`** ➔ **`Level`**

🤔 그런데… 테스트가 실패한다!

- 에러 메시지를 통해 SQL 문법이 틀렸음을 확인할 수 있다.
- 수정한 내용 중 SQL에 영향을 주는 부분은 필드 이름을 추가한 곳이므로
- 어딘가 필드 이름을 잘못 적었음을 알 수 있다…!

→ **이처럼 미리 DB까지 연동되는 테스트를 만들어두면 오타마저 빠르게 잡아낼 수 있게 된다!**

<br>
<br>

### 5.1.2 사용자 수정 기능 추가

✨ 사용자 관리 비즈니스 로직에 따르면 사용자 정보는 여러 번 수정될 수 있다.

- 성능을 극대화하기 위해 여러 개의 수정용 DAO 메소드를 만들어야 할 때도 있지만…
- 아직은 사용자 정보가 단순하고,
- 필드도 몇 개 되지 않으며,
- 사용자 정보 역시 자주 변경되지 않으므로 **간단히 접근**하자!

→ 수정할 정보가 담긴 User 오브젝트를 전달하면 id로 사용자를 찾아 UPDATE 문을 이용해 필드 정보를 모두 변경해주는 메소드를 하나 만들어보자.

<br>

#### 수정 기능 테스트 추가

- 만들어야 할 코드의 기능을 생각해보기 위해 테스트를 먼저 작성한다.
    
    ```java
    @Test
    public void update() {
    	dao.deleteAll();
    		
    	dao.add(user1);		// 수정할 사용자
    	dao.add(user2);		// 수정하지 않을 사용자
    		
    	// 픽스처에 들어 있는 정보를 변경해서 수정 메소드를 호출
    	user1.setName("오민규");
    	user1.setPassword("springno6");
    	user1.setLevel(Level.GOLD);
    	user1.setLogin(1000);
    	user1.setRecommend(999);
    		
    	dao.update(user1);
    		
    	User user1update = dao.get(user1.getId());
    	checkSameUser(user1, user1update);
    }
    ```
    
    1. 픽스처 오브젝트를 등록
    2. id를 제외한 필드의 내용을 바꾼 뒤 update()를 호출
    3. 해당 id의 사용자 정보가 변경되었는지 확인

<br>

#### UserDao와 UserDaoJdbc 수정

📌 여기까지 만들면 UserDao 인터페이스에 update() 메소드가 없다는 에러가 발생한다.

- IDE의 자동 수정 기능을 이용한다.

```java
public interface UserDao {

	...

	public void update(User user);

}
```

📌 이제 UserDaoJdbc에서 메소드를 구현하지 않았다는 에러가 발생한다

- 자동 추가 기능을 이용한다.

```java
public void update(User user) {
		this.jdbcTemplate.update(
				"update users set name = ?, password = ?, level = ?, login = ?, " +
				"recommend = ? where id = ? ", user.getName(), user.getPassword(), 
				user.getLevel().intValue(), user.getLogin(), user.getRecommend(),
				user.getId());	
}
```

<br>

#### 수정 테스트 보완

🤔 JDBC 개발에서 가장 많은 실수가 일어나는 곳은 바로 SQL 문장이다.

- 필드 이름이나 SQL 키워드를 잘못 입력했다면 테스트에서 확인할 수 있으나
- UPDATE 문장에서 WHERE 절을 빼먹는 경우에는 테스트로 검증하지 못할 수도 있다.

→ 현재의 테스트는 수정할 로우의 내용이 바뀐 것만 확인하고 수정하지 않아야 할 로우의 내용이 남아있는지는 확인하지 못한다는 문제점이 있다.

→ 이를 해결할 방법을 생각해보자.

1️⃣ 첫 번째 해결 방법

: JdbcTemplate의 update()가 돌려준 리턴 값을 확인한다.

2️⃣ 두 번째 해결 방법

: 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 확인한다.

: 여기서 사용할 방법이다.

- 두 번째 방법을 적용해서 보완한 update() 테스트
    
    ```java
    @Test
    	public void update() {
    		dao.deleteAll();
    		
    		dao.add(user1);		// 수정할 사용자
    		dao.add(user2);		// 수정하지 않을 사용자
    		
    		user1.setName("오민규");
    		user1.setPassword("springno6");
    		user1.setLevel(Level.GOLD);
    		user1.setLogin(1000);
    		user1.setRecommend(999);
    		
    		dao.update(user1);
    		
    		User user1update = dao.get(user1.getId());
    		checkSameUser(user1, user1update);
    		User user2same = dao.get(user2.getId());
    		checkSameUser(user2, user2same);
    }
    ```
    
<br>
<br>

### 5.1.3 UserService.upgradeLevels()

✨ 레벨 관리 기능을 구현해보자!

- 모든 사용자를 **`getAll()`**로 조회한 후, 레벨을 업그레이드하고 **`update()`** 를 호출해 DB에 반영한다.

- 이때, 사용자 관리 비즈니스 로직을 담을 UserService 클래스를 추가하자.
    - UserService는 UserDao 인터페이스 타입으로 userDao 빈을 DI 받아 사용하게 한다.
- 또, UserService를 위한 테스트 클래스인 UserServiceTest도 하나 추가하자.

![5 1](https://github.com/user-attachments/assets/309b2d0a-8815-4517-b457-90196ad2876e)

<br>

#### UserService 클래스와 빈 등록

📌 UserService 클래스를 만들고 UserDao 오브젝트를 저장해둘 인스턴스 변수를 선언하고

📌 UserDao 오브젝트의 DI가 가능하도록 수정자 메소드도 추가한다.

- UserService 클래스
    
    ```java
    public class UserService {
    
    	private UserDao userDao;
    
    	public void setUserDao(UserDao userDao) {
    		this.userDao = userDao;
    	}
    }
    ```
    

- 스프링 설정파일에 userService 아이디로 빈을 추가
    
    ```java
    <beans
        ...>
        <bean id="userService" class="springbook.user.service.UserService">
            <property name="userDao" ref="userDao"/>
        </bean>
        <bean id="userDao" class="springbook.dao.UserDaoJdbc">
            <property name="dataSource" ref="dataSource"/>
        </bean>
    <beans>
    ```
    
<br>

#### UserServiceTest 테스트 클래스

📌 UserService를 @Autowired가 붙은 인스턴스 변수로 선언

- UserServiceTest 클래스를 추가하고 테스트 대상인 UserService 빈을 제공받기 함
    
    ```java
    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration(locations="/test-applicationContext.xml")
    public class UserServiceTest {
    	@Autowired
    	UserService userService;
    }
    ```
    

📌 userService  빈의 주입을 확인하는 테스트 메소드 추가

```java
@Test 
public void bean() {
		awwertThat(this.userService, is(notNullValue()));
}
```

- 이후에 userService 오브젝트를 추가하면 bean() 테스트는 의미가 없으니 삭제해도 괜찮다.

<br>

#### upgradeLevels() 메소드

✨ 사용자 레벨 관리 기능을 먼저 만들고 테스트를 만들어보자!

- 사용자 레벨 업그레이드 메소드
    
    ```java
    public void upgradeLevels() {
            List<User> users = userDao.getAll();
            for (User user : users) {
                Boolean changed = null; // 레벨의 변화가 있는지 확인하는 플래그
                if (user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
                    user.setLevel(Level.SILVER); // Basic 레벨 업그레이드
                    changed = true;
                } else if (user.getLevel() == Level.SILVER && user.getRecommend() >= 30) {
                    user.setLevel(Level.GOLD); // Silver 레벨 업그레이드
                    changed = true; // 레벨 변경 플래그 설정
                } else if (user.getLevel() == Level.GOLD) {
                    changed = false; // Gold 레벨은 변경X
                } else { // 일치하는 조건이 없으면 변경 X
                    changed = false;
                }
                // 레벨 변경이 있는 경우에만 update() 호출
                if (changed) userDao.update(user);
            }
        }
    }
    ```
    
    - **모든 사용자 조회**: `userDao.getAll()`을 통해 모든 사용자를 가져옴.
    - **레벨 업그레이드 조건 확인**:
        - **BASIC** 레벨은 로그인 수가 50 이상이면 **SILVER**로 업그레이드.
        - **SILVER** 레벨은 추천 수가 30 이상이면 **GOLD**로 업그레이드.
        - **GOLD** 레벨은 변경되지 않음.
        - 조건에 맞지 않으면 레벨 변경 없음.
    - **레벨 변경 확인**: 각 사용자의 레벨이 변경되었으면 `changed` 플래그를 `true`로 설정하고, 변경된 경우에만 `userDao.update(user)`로 DB에 반영.

<br>

#### upgradeLevels() 테스트

🤔 테스트는 최소 다섯 가지 경우를 확인

- 사용자 레벨은 **BASIC**, **SILVER**, **GOLD**가 있고 **GOLD**는 변경되지 않으므로
- **BASIC**, **SILVER**에 대해 업그레이드 되는 경우와
- 그렇지 않은 경우를 포함하여 테스트

➡️ 각 경우에 맞는 사용자 정보를 등록한 후, 업그레이드가 예상대로 진행되는지 확인!

- 테스트 픽스처
    
    ```java
    public class UserServiceTest {
        @Autowired
        UserService userService;
        @Autowired
        UserDao userDao;
        List<User> users;
    
        @Before
        public void setUp(){
            users = Arrays.asList(
                    new User("bumjin", "박범진", "p1", Level.BASIC, 49 ,0),
                    new User("joytouch", "강명성", Level.BASIC, 50 ,0),
                    new User("erwins", "신승한", "p3", Level.SILVER, 60 ,29),
                    new User("madnite1", "이상호", "p4", Level.SILVER, 60 ,30),
                    new User("green", "오민규", "p5", Level.GOLD, 100 ,100)
            );
        }
    ```
    
    - BASIC과 SILVER 레벨의 사용자는 각각 두 개씩 등록
    - 로그인 횟수와 추천 횟수가 기준 값 이상이 되면 레벨이 업그레이드 됨
        - 테스트시에는 데이터를 경계가 되는 값의 전후로 사용하는 것이 좋음.
        - ex) SILVER 업그레이드 경계인 50에서 하나 모자란 49, 업그레이드가 되는 가장 작은 값인 50

- 준비된 픽슽터를 사용해 만든 테스트
    
    ```java
    @Test
    	public void upgradeLevels() {
    		userDao.deleteAll();
    		for(User user : users) userDao.add(user);
    		
    		userService.upgradeLevels();
    		
    		checkLevelUpgraded(users.get(0), false);
    		checkLevelUpgraded(users.get(1), true);
    		checkLevelUpgraded(users.get(2), false);
    		checkLevelUpgraded(users.get(3), true);
    		checkLevelUpgraded(users.get(4), false);
    	}
    
    private void checkLevel(User user, Level expectedLevel) {
    		User userUpdate = userDao.get(user.getId());
    		assertThat(userUpdate.getLevel(), is(expectedLevel));
    }
    ```
    
    - 준비한 다섯 가지 종류의 사용자 정보를 저장한 뒤에 upgradeLevels() 메소드 실행
    - 업그레이드 작업이 끝나면 사용자 정보를 하나씩 가져와 레벨의 변경 여부를 확인

<br>
<br>

### 5.1.4 UserService.add()

🤔 사용자 비즈니스 로직 중 처음 가입하는 사용자의 기본 레벨을 BASIC으로 설정해야 하는 기능이 남아있다!

→ UserService에 이 로직을 넣어보자!

- 테스트 구현
    - 검증할 기능: UserService의 add()를 호출하면 레벨이 BASIC으로 설정되는 것
    - 두 가지의 테스트 케이스 → 두 가지 경우 모두 add() 메소드를 호출하고 결과를 확인
        1. 레벨이 미리 정해진 경우
        2. 레벨이 비어 있는 경우
    - 변경된 레벨을 확인하는 두 가지 방법
        1. 간단한 방법: add() 메소드를 호출할 때 파라미터로 넘긴 User 오브젝트에 level 필드를 확인해 보는 것
        2. 다른 방법: get() 메소드를 이용해서 DB에 저장된 User 정보를 가져와 확인하는 것

- add() 메소드에 대한 테스트
    
    ```java
    @Test 
    	public void add() {
    		userDao.deleteAll();
    		
    		User userWithLevel = users.get(4);	  // GOLD 레벨  
    		User userWithoutLevel = users.get(0);  
    		userWithoutLevel.setLevel(null);
    		
    		userService.add(userWithLevel);	  
    		userService.add(userWithoutLevel);
    		
    		User userWithLevelRead = userDao.get(userWithLevel.getId());
    		User userWithoutLevelRead = userDao.get(userWithoutLevel.getId());
    		
    		assertThat(userWithLevelRead.getLevel(), is(userWithLevel.getLevel())); 
    		assertThat(userWithoutLevelRead.getLevel(), is(Level.BASIC));
    	}
    ```
    

- add() 메소드
    
    ```java
    public void add(User user) {
    		if (user.getLevel() == null) user.setLevel(Level.BASIC);
    		userDao.add(user);
    }
    ```
    
<br>
<br>

### 5.1.5 코드 개선

✨ 비즈니스 로직의 구현을 모두 마쳤지만… **만들어진 코드를 다시 한 번 검토**해보자!

- 코드에 중복된 부분은 없는가?
- 코드가 무엇을 하는 것인지 이해하기 불편하지 않은가?
- 코드가 자신이 있어야 할 자리에 있는가?
- 앞으로 변화가 일어난다면 그 변화에 쉽게 대응할 수 있게 작성되어 있는가?

<br>

#### upgradeLevels() 메소드 코드의 문제점

1. for 루프 속에 들어 있는 if/elseif/else 블록들이 읽기 불편하다.
    - 레벨의 변화 단계, 업그레이드 조건, 그리고 조건이 충족됐을 때 해야할 작업들이 섞여 있어 로직을 이해하기 쉽지 않다.
    - 플래그를 두고 이를 변경하고 마지막에 확인해서 업데이트를 진행하는 방법도 깔끔하지 않다.
        
        ```java
        if (user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
             user.setLevel(Level.SILVER); //basic 업그레이드
             changed = true;
        }
        ...
        if (changed) { userDao.update(user); }
        ```
        
        1. `user.getLevel() == Level.BASIC` : 현재 레벨이 무엇인지 파악
        2. `user.getLogin() >= 50` : 업그레이드 조건을 담은 로직
        3. `user.setLevel(Level.SILVER` : 다음 단계의 레벨이 무엇이며 업그레이드를 위한 작업은 어떤 것인지
        4. `changed = true;` : 5의 작업이 필요한지를 알려주기 위해 임시 플래그로 설정해주는 것
        5. `if (changed) { userDao.update(user); }`
    
    → 성격이 조금씩 다른 것들이 섞여 있거나 분리돼서 나타나는 구조!!
    

1. 그런 if 조건 블록이 레벨 개수만큼 반복된다.
    - 만약 새로운 레벨이 추가된다면
        - Level 이늄도 수정하고
        - upgradeLevels()의 레벨 업그레이드 로직을 담은 코드에 if 조건식과 블록을 추가
    
    → 업그레이드 조건이 복잡해지거나 단지 level 필드를 변경하는 수준 이상이 되면 upgradeLevels() 메소드는 점점 길어지고 복잡해지며 갈수록 이해하고 관리하기가 어려워진다.
    

1. 레벨과 업그레이드 조건을 동시에 비교하는 부분도 어딘가 이상하다.
    - BASIC이면서 로그인 횟수가 50이하 → 마지막 else 블록으로 이동
    - 새로운 레벨이 추가되는 경우 → 마지막 else 블록으로 이동
    
    → 성격이 다른 두 가지 경우가 모두 한 곳에서 처리되는 것은 무언가 이상하다
    
    - 제대로 만들기 위해서는 두 단계의 조건을 비교해야 한다.
        - 레벨을 확인하고 각 레벨별로 다시 조건을 판단하는 조건식 추가

- 이렇게 만들면 지금보다 깔끔해지겠지만 코드가 훨씬 복잡해지게 된다.
- 곰곰이 따져보면 상당히 변화에 취약하고 다루기 힘든 코드이다!

<br>

#### upgradeLevels() 리팩토링

- **기존 코드의 문제점**
    - 중복된 if/else 조건문
    - 업무 로직과 구체적 조건이 혼재
    - 변화에 취약

🤔 이제 코드를 리팩토링해보자!

- **추상적인 레벨에서 로직을 작성**해보자.
    - 기본 작업 흐름만 남겨둔 `upgradeLevels()`
        
        ```java
        public void upgradeLevels() {
             List<User> users = userDao.getAll();
             for (User user : users) {
                 if(canUpgradeLevel(user)) {
                     upgradeLevel(user);
                 }
            }
        }
        ```
        
        - 모든 사용자 정보를 가져와 한 명씩 업그레이드가 가능한지 확인
        - 가능하다면 업그레이드
    - 구체적인 내용을 담은 메소드 구현
        - `canUpgradeLevel()` 메소드
            - 업그레이드가 가능한지를 알려주는 메소드
            - 주어진 user에 대해 업그레이드가 가능하면 true, 가능하지 않으면 false를 리턴
            
            ```java
            private boolean canUpgradeLevel(User user) {
                Level currentLevel = user.getLevel();
                    
                switch(currentLevel) {
                    case BASIC -> user.getLoginCount() >= 50;
                    case SILVER -> user.getRecommendCount() >= 30;
                    case GOLD -> return false;
                    default -> throw new IllegalArgumentException("Unknown Level: " + currentLevel);
                }
            }
            ```
            
            - User 오브젝트에서 레벨을 가져와서, switch 문으로 구분하고, 업그레이드 조건을 만족하는지 확인
        - `upgradeLevel()` 메소드
            - 업그레이드 조건을 만족했을 경우, 구체적으로 무엇을 할 것인가를 담고 있는 메소드
                
                ```java
                private void upgradeLevel(User user) {
                    if (user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
                    else if (user.getLevel() == Level.SILVER) user.setLevel(Level.GOLD);
                    userDao.update(user);
                }
                ```
                
                - 사용자 오브젝트의 레벨정보를 다음 단계로 변경하고, 변경된 오브젝트를 DB에서 업데이트함.
        - 메소드 수정
            - 다음 단계가 무엇인가 하는 로직 & 사용자 오브젝트의 level 필드를 변경해준다는 로직이 함께 존재
            - 예외 상황에 대한 처리가 없음
            
            → 리팩토링 해보자
            
        - Level 이늄의 역할 확장
            - `Level` 이늄에 `nextLevel()` 메소드를 추가하여 레벨 순서를 정의하고, 다음 레벨 정보를 가지게 함.
                
                ```java
                public enum Level {
                    //  이늄 선언데 DB에 저장할 값과 함께 다음 단계의 레벨 정보도 추가
                    GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);
                
                    private final int value;
                    private final Level next;
                
                    Level(int value, Level next) {
                        this.value = value;
                        this.next = next;
                    }
                
                    public Level nextLevel() {
                        return next;
                    }
                
                    public int intValue() {
                        return value;
                    }
                
                    public static Level valueOf(int value) {
                        return switch (value) {
                            case 1 -> BASIC;
                            case 2 -> SILVER;
                            case 3 -> GOLD;
                            default -> throw new AssertionError("Unknown value: " + value);
                        };
                    }
                }
                ```
                
                - 각 레벨의 다음 단계가 무엇인지 로직에서 반복적으로 조건문을 사용하지 않고 `nextLevel()`을 통해 얻을 수 있게 됨.
        - 사용자 정보가 바뀌는 부분을 `UserService` 메소드에서 `User`로 동
            - `UserService`보다는 `User`가 레벨 업그레이드 작업을 스스로 처리하는 편이 낫다.
                
                ```java
                public void upgradeLevel() {
                    Level nextLevel = this.level.nextLevel();
                
                    if (nextLevel == null) {
                       throw new IllegalStateException(this.level + "은 업그레이드가 불가능합니다.");
                    } else {
                        this.level = nextLevel;
                    }
                }
                ```
                
                - Level의 nextLevel() 기능을 이용해 다음 단계의 레벨 확인 후 레벨 변경
                - 단, 더 이상 업그레이드가 불가능한 경우가 있으므로 예외상황에 대한 검증 기능을 가지고 있는 편이 안전
                    - 다음 레벨이 없는 경우 예외를 던져야 함.
        - `User`에 업그레이드 작업을 담당하는 독립적인 메소드 추가
            - `this.lastUpgraded = new Date();` 를 **`upgradedLevel()`**에 추가
                
                ```java
                 private void upgradeLevel(User user) {
                    user.upgradeLevel();
                    userDao.update(user);
                }
                ```
                

✨ if 문장이 많이 들어 있던 이전 코드보다 간결하고 작업 내용이 명확하게 드러 나는 코드가 됐다!

<br>

#### User 테스트

🤔 방금 `User`에 간단하지만 로직을 담은 메소드를 추가했으니 이에 대한 테스트 코드를 작성해보자!

- `User` 테스트
    
    ```java
    public class UserTest {
    	User user;
    	
    	@Before
    	public void setUp() {
    		user = new User();
    	}
    	
    	@Test()
    	public void upgradeLevel() {
    		Level[] levels = Level.values();
    		for(Level level : levels) {
    			if (level.nextLevel() == null) continue;
    			user.setLevel(level);
    			user.upgradeLevel();
    			assertThat(user.getLevel(), is(level.nextLevel()));
    		}
    	}
    	
    	@Test(expected=IllegalStateException.class)
    	public void cannotUpgradeLevel() {
    		Level[] levels = Level.values();
    		for(Level level : levels) {
    			if (level.nextLevel() != null) continue;
    			user.setLevel(level);
    			user.upgradeLevel();
    		}
    	}
    }
    ```
    
    - `upgradeLevel()` 테스트
        - Level 이늄에 정의된 모든 레벨을 가져와서 User에 설정해두고 User의 upgradeLevel()을 실행해서 다음 레벨로 바뀌는지 확인
        - 이렇게 테스트를 만들어두면 나중에 메소드에 복잡한 기능이 추가되더라도 확장해서 사용할 수 있다.
    - 더 이상 업그레이드할 레벨이 없는 경우에 `upgradeLevel()` 을 호출하면 예외가 발생하는지 확인하는 테스트
        - `nextLevel()`이 null인 경우에 강제로 `upgradeLevel()` 호출
        - 이때, @Test(expected=) 에 설정한 예외가 발생하면 → 테스트 성공

<br>

#### UserServiceTest 개선

🤔 UserService 테스트도 손볼 데가 없을지 살펴보자!

- 기존 테스트
    
    : checkLevel() 메소드를 호출할 때 일일이 다음 단계의 레벨이 무엇인지 넣어줌 → 중복
    
- 개선한 `upgradeLevels()` 테스트
    
    ```java
    @Test()
    public void upgradeLevel() {
    	userDao.deleteAll();
    	for(User user : users) userDao.add(user);
    	
    	userService.upgradeLevels();
    	
    	checkLevelUpgraded(users.get(0), false);
    	checkLevelUpgraded(users.get(1), true);
    	checkLevelUpgraded(users.get(2), false);
    	checkLevelUpgraded(users.get(3), true);
    	checkLevelUpgraded(users.get(4), false);
    }
    
    private void checkLevelUpgraded(User user, boolean upgraded) {
    	User userUpdate = userDao.get(user.getId());
    	if (upgraded ){
    		// 업그레이드가 일어났는지 확인
    		assertTaht(userUpdate.getLevel(), is(user.getLevel().nextLevel()));
    	}
    	else {
    		// 업그레이드가 일어나지 않았는지 확인
    		assertThat(userUpdate.getLevel(), is(user.getLevel()));
    	}
    }
    ```
    
    - 각 사용자에 대해 업그레이드를 확인하려는 것인지 아닌지가 좀 더 이해하기 쉽게 나타나 있어서 좋다.
    - 업그레이드 됐을 때, 어떤 레벨인지는 Level 이늄의 `nextLevel()`을 호출해보면 된다.

- 코드에 나타난 중복을 제거해보자
    - 업그레이드 조건인 로그인 횟수와 추천 횟수가 애플리케이션 코드, 테스트 코드에 중복되어 나타난다.
        - `case BASIC: return (user.getLogin() ≥ 50);`
        - `new User(”joytouch”, “강명성”, “p2”, Level.BASIC, 50, 0)`
    - 테스트와 애플리케이션 코드에 나타난 이런 숫자의 중복도 제거해주어야 한다.
        - 기준이 되는 최소 로그인 횟수가 변경될 때도 한 번만 수정할 수 있도록 만들자.
            
            ```java
            public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
            public static final int MIN_RECOMMEND_FOR_GOLD = 30;
            
            public boolean canUpgradeLevel(User user){
                Level currentLevel = user.getLevel();
            
                switch(currentLevel){
                    case BASIC : return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER);
                    case SILVER : return (user.getRecommend() >= MIN_RECOMMEND_FOR_GOLD);
                    case GOLD : return false;
                    default : throw new IllegalArgumentException("Unknown Level : " + currentLevel);
                }
            }
            ```
            
        - 테스트도 정의해둔 상수를 사용하도록 만들자!
            
            ```java
            import static springbook.user.service.UserService.MIN_LOGCOUNT_FOR_SILVER;
            import static springbook.user.service.UserService.MIN_RECCOMEND_FOR_GOLD;
            
            @Before
            public void setUp() {
            	users = Arrays.asList(
            			new User("bumjin", "박범진", "p1", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER-1, 0),
            			new User("joytouch", "강명성", "p2", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER, 0),
            			new User("erwins", "신승한", "p3", Level.SILVER, 60, MIN_RECCOMEND_FOR_GOLD-1),
            			new User("madnite1", "이상호", "p4", Level.SILVER, 60, MIN_RECCOMEND_FOR_GOLD),
            			new User("green", "오민규", "p5", Level.GOLD, 100, Integer.MAX_VALUE)
            		);
            }
            ```
            
    
    ✨ 숫자로만 되어 있는 경우에는 이해하기 힘들었던 부분이 이제는 무슨 의도로 어떤 값을 넣었는지 이해하기 쉬워졌다.
    

- 레벨을 업그레이드 하는 정책을 유연하게 변경할 수 있도록 개선
    - 사용자 업그레이드 정책을 UserService에서 분리하는 방법을 고려해볼 수 있다.
    
    ```java
    public interface UserLevelUpgradePolicy {
    	boolean canUpgradeLevel(User user);
    	void upgradeLevel(User user);
    }
    ```

<br>
<br>

## 5.2 트랜잭션 서비스 추상화

🤔 정기 사용자 레벨 관리 작업을 수행하는 도중에 네트워크가 끊기거나 서버에 장애가 생겨서 작업을 완료할 수 없다면, 그때까지 변경된 사용자의 레벨은 그대로 두는 게 나을까? 아니면 모두 초기 상태로 돌려놓는 게 좋을까?

→ 중간에 문제가 발생해서 작업이 중단된다면 진행된 모든 변경 작업을 취소시킨다.

- 일부 사용자는 레벨이 조정되고 일부는 되지 않는다면 반발이 심할 것으로 예상되기 때문

<br>

### 5.2.1 모 아니면 도

✨ 지금까지 만든 사용자 레벨 업그레이드 코드는 어떻게 동작할지 궁금해진다

- 테스트를 만들어 확인해보자!

<br>

#### 테스트용 `UserService` 대역

- 예외를 강제로 발생시키는 방법
    - 중간에 예외를 발생시키려면 애플리케이션 코드를 수정할 수도 있지만, 테스트를 위해 직접 코드를 변경하는 건 바람직하지 않다. 대신 `UserService`의 테스트 대역을 사용하는 방법이 좋다. 테스트용 대역은 `UserService`를 대신하여 테스트의 목적에 맞게 동작하도록 만들어진 클래스이다.
- 테스트용 `UserService` 확장 클래스 생성
    - `UserService`의 코드를 복사해 수정할 수도 있지만, 이는 코드 중복과 유지보수의 어려움을 초래할 수 있다. 더 나은 방법은 `UserService`를 상속하고 필요한 메서드만 오버라이딩하는 것이다.
    - 현재 5명의 테스트 사용자 중 2번째와 4번째가 업그레이드 대상이다. 예외는 4번째 사용자 처리 중 발생시키고, 2번째 사용자의 업그레이드 상태가 유지되는지 확인할 것이다.
- `UserService`를 상속한 테스트용 클래스 작성
    - 테스트에서만 사용할 클래스는 테스트 클래스 내부에 스태틱 클래스로 작성하는 것이 편리하다.
    - `UserService`의 `upgradeLevel()` 메서드를 상속하여 오버라이딩할 예정이다. 그런데 `upgradeLevel()` 메서드는 현재 `private` 접근제한자로 되어 있어 오버라이딩이 불가능하다. 테스트 목적으로 `protected`로 접근제한을 수정해야 한다.
    
    ```java
    protected void upgradeLevel(User user) { ... }
    
    ```
    
- `UserService` 대역 클래스 작성
    - `TestUserService` 클래스에서 `upgradeLevel()`을 오버라이딩하여 특정 사용자 ID를 만났을 때 예외를 던지도록 구현한다.
    
    ```java
    static class TestUserService extends UserService {
        private String id;
    
        private TestUserService(String id) {
            this.id = id;
        }
    
        @Override
        protected void upgradeLevel(User user) {
            if (user.getId().equals(this.id)) {
                throw new TestUserServiceException();
            }
            super.upgradeLevel(user);
        }
    }
    
    ```
    
    - `upgradeLevel()` 메서드는 기본 기능을 수행하되, 미리 설정한 ID를 가진 사용자를 발견하면 예외를 발생시킨다.
- 테스트용 예외 클래스 정의
    - 다른 예외와 구분하기 위해, 테스트 목적의 예외를 정의해 `TestUserService`에서 사용할 수 있도록 한다. `TestUserServiceException` 클래스는 `TestUserService`와 마찬가지로 테스트 클래스 내에 스태틱 멤버 클래스로 작성해도 된다.

<br>

#### 강제 예외 발생을 통한 테스트

📌 이제 테스트를 만들어보자!

- 목적: 사용자 레벨 업그레이드를 시도하다가 예외가 발생한 경우, 그전에 업그레이드했던 사용자도 다시 원래 상태로 돌아오는지 확인하는 것
- 테스트 코드
    
    ```java
    public class UserServiceTest {
        @Test
        public void upgradeAllOrNothing() {
            // 예외를 발생시킬 네 번째 사용자의 id를 넣어서 테스트용 UserService 대역 오브젝트를 생성한다.
            UserService testUserService = new TestUserService(users.get(3).getId());
            // userDao를 수동 DI 해준다.
            testUserService.setUserDao(this.userDao);
    
            userDao.deleteAll();
            for  (User user : users) {
                userDao.add(user);
            }
    
            // TestUserService는 업그레이드 작업 중 예외가 발생해야한다.
            // 정상 종료라면 문제가 있으니 강제로 실패
            try {
                testUserService.upgradeLvls();
                fail("TestUserServiceException expected");
            // TestUserService가 던져주는 예외를 잡아서 계속 진행하도록 한다.
            // 그 외의 예외라면 테스트 실패
            } catch (TestUserServiceException e) {
            }
    
            // 예외가 발생하기 전 레벨 변경이 있었던 사용자의 레벨이 처음 상태로 바뀌었나 확인
            checkLvlUpgraded(users.get(1), false);
        }
    }
    ```
    
    - 테스트 프로세스
        1. TestUserService의 오브젝트 생성
            - 생성자 파라미터 : 예외를 발생시킬 사용자의 id
        2. UserDao를 수동으로 DI
            - 테스트 메소드에서 특별한 목적으로 사용되므로 스프링 빈으로 등록할 필요 없음
        3. 다섯 개의 사용자 정보를 등록
        4. testUserService의 업그레이드 메소드 실행
        5. 4번째 사용자 오브젝트 차례가 되면 TestUserServiceException 발생
        6. 예외가 발생하지 않고 정상 종료되면 테스트 실패
        7. 이후에는, 두 번째 사용자 레벨이 변경됐는지 확인
            - 네 번째 사용자를 처리하다 예외가 발생했으니 두 번째 사용자도 원래 상태로 돌아가야 함.
    
    → 네 번째 사용자 처리 중 예외가 발생했지만 두 번째 사용자의 레벨 변경이 그대로 유지된다: **테스트 실패**!
    

<br>

#### 테스트 실패의 원인

🤔 모든 사용자의 레벨을 업그레이드하는 작업(upgradeLevels() 메소드)이 하나의 트랜잭션 안에서 동작하지 않았기 때문

- 트랜잭션이란?
    - 더 이상 나눌 수 없는 단위 작업

✨ 모든 사용자에 대한 업그레이드 작업은 전체 다 성공하든지 전체 다 실패해야 한다.

<br>
<br>

### 5.2.2 트랜잭션 경계설정

📌 DB는 그 자체로 완벽한 트랜잭션을 지원하지만 여러 개의 SQL이 사용되는 작업을 하나의 트랜잭션으로 취급해야 하는 경우에는 따로 설정을 해주어야 한다.

- 트랜잭션 롤백
- 트랜잭션 커밋

<br>

#### JDBC 트랜잭션의 트랜잭션 경계설정

- 트랜잭션 시작과 종료
    - 모든 트랜잭션은 시작과 종료 시점이 있다. 트랜잭션은 작업을 완료하고 확정하는 **커밋** 또는 작업을 무효화하는 **롤백** 중 하나로 종료된다.
    - 애플리케이션 내에서 트랜잭션이 시작되고 종료되는 위치를 **트랜잭션 경계**라고 하며, 올바르게 설정하는 것이 중요하다.
- JDBC를 이용한 간단한 트랜잭션 예제
    - 다음 예제는 트랜잭션 처리에 초점을 맞춘 간단한 코드로, `Connection`과 `PreparedStatement`를 이용해 두 가지 DB 작업을 트랜잭션으로 묶는다.
    
    ```java
    // DB 커넥션 시작
    Connection c = dataSource.getConnection();
    
    // 트랜잭션 시작
    c.setAutoCommit(false);
    
    try {
        PreparedStatement st1 = c.prepareStatement("update users ...");
        st1.executeUpdate();
    
        PreparedStatement st2 = c.prepareStatement("delete users ...");
        st2.executeUpdate();
    
        // 트랜잭션 커밋
        c.commit();
    } catch (Exception e) {
        // 트랜잭션 롤백
        c.rollback();
    }
    
    // DB 커넥션 종료
    c.close();
    
    ```
    
- 트랜잭션 경계 설정
    - 트랜잭션의 경계 설정은 `Connection`을 사용해 트랜잭션을 시작하고 종료하는 작업을 의미한다.
    - JDBC 트랜잭션은 하나의 `Connection` 객체에서 일어나며, `setAutoCommit(false)` 메서드로 트랜잭션을 시작한다.
    - JDBC의 기본 설정은 자동 커밋 모드인데, 이는 각 DB 작업이 끝날 때마다 자동으로 커밋되므로, 여러 DB 작업을 하나의 트랜잭션으로 묶을 수 없다. 자동 커밋을 `false`로 설정하면 **트랜잭션 시작**을 선언한 것이 된다.
    - 이후 `commit()` 메서드를 호출하면 트랜잭션이 완료되어 DB에 작업 결과가 반영되고, `rollback()` 메서드를 호출하면 작업 결과가 취소된다. 일반적으로 예외 발생 시 트랜잭션은 롤백된다.
- 로컬 트랜잭션
    - **트랜잭션 경계 설정(Transaction Demarcation)**은 `setAutoCommit(false)`로 시작하여 `commit()` 또는 `rollback()`으로 종료되는 작업을 뜻한다.
    - JDBC 트랜잭션은 하나의 `Connection` 내에서 이루어지는 **로컬 트랜잭션(Local Transaction)**으로, `Connection`이 생성되고 닫히는 범위 안에서 트랜잭션 경계가 설정된다.

<br>

#### UserService와 UserDao의 트랜잭션 문제

- `upgradeLevels()`에서 트랜잭션이 적용되지 않는 이유
    - 현재 `upgradeLevels()`에는 트랜잭션을 시작하고, 커밋하거나 롤백하는 **트랜잭션 경계 설정 코드가 없다**.
    - JDBC의 트랜잭션 설정 메서드는 `Connection` 객체가 필요하지만, `JdbcTemplate`을 사용한 이후 `Connection` 객체를 직접적으로 사용할 수 없게 되었다.
- `JdbcTemplate`의 트랜잭션 흐름
    - `JdbcTemplate`은 `JdbcContext`와 유사한 흐름으로 `DataSource`의 `getConnection()`을 호출해 `Connection`을 가져오고, 메서드 실행 후 커넥션을 종료한다.
    - `JdbcTemplate`의 **각 호출마다 새로운 커넥션이 생성되고, 종료되기 때문에** 메서드마다 독립적인 트랜잭션이 만들어진다.
    - 따라서 `UserDao`의 각 메서드는 독립된 트랜잭션으로 실행되며, **하나의 작업을 여러 메서드 호출에 걸쳐 하나의 트랜잭션으로 묶는 것이 불가능**하다.
- 트랜잭션이 독립적으로 실행되는 예제
    - `upgradeAllOrNothing()` 메서드는 5명의 사용자를 순차적으로 업그레이드한다.
    - 두 번째 사용자 레벨이 변경되어 `UserDao`의 `update()` 메서드가 호출되면, `JdbcTemplate`은 이 작업을 하나의 트랜잭션으로 **자동 커밋**한다.
    - 이후 예외가 발생하거나 서버가 다운되더라도, 이미 커밋된 결과는 DB에 영구적으로 남는다.
- `UserService`와 `UserDao` 간 트랜잭션의 한계
    - `upgradeLevels()`에서 여러 번 `UserDao`의 `update()` 메서드를 호출할 경우, 각 호출은 새로운 `Connection`과 트랜잭션을 생성한다. 첫 번째 `update()` 호출이 성공해 커밋된다면, 이후 `update()`에서 예외가 발생하더라도 첫 번째 작업의 결과는 유지된다.
    - 이렇게 **DAO를 분리한 구조에서는 각 DAO 메서드 호출이 독립적인 트랜잭션**을 생성한다. 이는 JDBC API든 `JdbcTemplate`이든 동일한 구조적 한계다.
    - 결과적으로 `UserService`에서 하나의 트랜잭션으로 묶어야 하는 작업을 구현하기 어렵다.
- 트랜잭션을 하나로 묶는 방법
    - `upgradeLevels()`과 같이 **여러 DB 업데이트를 하나의 트랜잭션으로 묶으려면 단일 DB 커넥션을 사용해야** 한다. 트랜잭션은 `Connection` 객체에서 생성되기 때문이다.
    - 현재 `UserService`는 DB 커넥션을 직접 다룰 수 없어 이러한 방식으로 트랜잭션을 관리할 수 없다.

<br>

#### 비즈니스 로직 내의 트랜잭션 경계설정

- **DAO 메서드 안으로 로직 옮기기?**
    - `upgradeLevels()`의 내용을 `DAO` 안으로 이동해 하나의 `DB` 커넥션과 트랜잭션을 활용하여 여러 사용자의 정보를 업데이트할 수 있다.
    - 그러나 **비즈니스 로직과 데이터 로직이 결합**되는 결과를 낳아, 코드의 확장성과 유지보수성을 크게 떨어뜨린다.
- **해결 방향**
    - 트랜잭션의 경계설정을 `UserService`로 옮기고, `upgradeLevels()` 메서드의 시작과 끝에 **트랜잭션을 설정**하여 `DAO`의 SQL 코드나 JDBC API 활용 방식은 그대로 유지한다.
    - `UserService`에서 최소한의 트랜잭션 코드만 가져와, 데이터 액세스와 비즈니스 로직의 책임을 유지한 채 트랜잭션 문제를 해결할 수 있다.
- **`upgradeLevels()` 메서드의 트랜잭션 경계 설정**
    - `upgradeLevels()` 메서드에서 `DB Connection`을 시작하고 트랜잭션을 관리하는 구조를 도입한다:
        
        ```java
        public void upgradeLvls() throws Exception {
            // (1) DB Connection 시작
            // (2) 트랜잭션 시작
            try {
                // (3) DAO 메서드 호출
                // (4) 트랜잭션 커밋
            } catch (Exception e) {
                // (5) 트랜잭션 롤백
                throw e;
            } finally {
                // (6) DB Connection 종료
            }
        }
        
        ```
        
    - 트랜잭션을 시작하고 커밋 또는 롤백하는 전형적인 `JDBC` 코드 구조다.
    - `Connection` 객체는 `upgradeLevels()` 메서드 안에서 만들어지고 종료된다. 이로써 `UserDao` 메서드들은 `upgradeLevels()`에서 생성된 **같은 트랜잭션을 공유**할 수 있다.
- **`Connection` 객체의 공유 방법**
    - `upgradeLevels()`에서 시작한 `Connection`을 `UserDao`의 메서드가 동일하게 사용하도록, `Connection`을 파라미터로 전달한다:
        
        ```java
        public interface UserDao {
            public void add(Connection c, User user);
            public User get(Connection c, String id);
            public void update(Connection c, User user);
        }
        
        ```
        
    - 이 방식으로 `UserService`에서 **생성한 `Connection`이 `UserDao`의 트랜잭션에서도 동일하게 사용**된다.
- **UserService 내의 메서드 간 `Connection` 공유**
    - `UserDao`의 `update()`를 호출하는 `upgradeLevel()` 메서드에도 같은 `Connection` 객체를 전달한다:
        
        ```java
        class UserService {
            public void upgradeLvls() throws Exception {
                Connection c = ...;
                try {
                    upgradeLvl(c, user);
                }
            }
        
            protected void upgradeLvl(Connection c, User user) {
                user.upgradeLvl();
                userDao.update(c, user);
            }
        }
        
        ```
        
    - `upgradeLevels()`에서 **시작한 트랜잭션에 `upgradeLevel()` 메서드와 `UserDao` 메서드들이 모두 참여**하게 된다. 이를 통해 비즈니스 로직과 데이터 액세스 로직 간의 트랜잭션 일관성을 유지할 수 있다.
- **결국…**
    - `upgradeLevels()`에서 트랜잭션 경계를 설정하고 `Connection`을 공유하는 방식으로, **DAO 코드에서도 트랜잭션을 유지**하면서도 비즈니스 로직과 데이터 로직의 분리를 유지할 수 있다.
    - 이는 `UserService`가 `DB Connection`과 트랜잭션의 경계를 설정하는 중심이 되고, `UserDao`는 데이터 액세스를 전담하는 역할을 유지하도록 하는 방법이다.

<br>

#### UserService 트랜잭션 경계설정의 문제점

🤔 UserService와 UserDao를 이런 식으로 수정하면 트랜잭션 문제는 해결할 수 있겠지만, 그 대신 여러 가지 새로운 문제가 발생한다!

1. JdbcTemplate활용 불가
    - 결국 JDBC API를 직접 사용하는 초기 방식으로 돌아가야 한다.
    - try/catch/finally 블록은 이제 UserService 내에 존재하고, UserService의 코드는 JDBC 작업 코드의 전형적인 문제점을 그대로 가질 수밖에 없다.
2. DAO의 메소드와 비즈니스 로직을 담고 있는 UserService의 메소드에 Connection 파라미터가 추가됨.
    - upgradeLevels()에서 사용하는 메소드의 어딘가에서 DAO를 필요로 한다면, 그 사이의 모든 메소드에 걸쳐서 Connection 오브젝트가 계속 전달돼야 한다.
    - UserService는 싱글톤이므로 인스턴스 변수에 Connection을 저장해 다른 메서드에서 사용할 수 없음.
        - 멀티스레드 환경에서는 값이 덮어써질 위험이 있기 때문
    - 결국, UserService의 메서드들은 Connection 파라미터를 받아야 해 코드가 지저분해짐.
3. Connection 파라미터가 UserDao 인터페이스 메소드에 추가되면 UserDao는 더 이상 데이터 액세스 기술에 독립적일 수가 없음.
    - UserDao 인터페이스가 바뀔 것이고, 그에 따라 userService 코드도 함께 수정돼야 한다.
    - 기껏 인터페이스를 사용해 DAO를 분리하고 DI를 적용했던 수고가 물거품이 될 것
4. DAO 메소드에 Connection 파라미터를 받게 하면 테스트 코드에도 영향을 미친다. 
    - 지금까지 DB 커넥션은 전혀 신경 쓰지 않고 테스트에서 UserDao를 사용 할 수 있었는데, 이제는 테스트 코드에서 직접 오브젝트를 일일이 만들어서 DAO 메소드를 호출하도록 모두 변경해야 함.

<br>
<br>

### 5.2.3 트랜잭션 동기화

🤔 비즈니스 로직을 담고 있는 UserService 메소드 안에서 트랜잭션의 경계를 설정해 관리하려면 지금까지 만들었던 깔끔하게 정리된 코드를 포기해야 할까?

아니면 트랜잭션 기능을 포기해야 할까?

→ 스프링은 이 딜레마를 해결해준다!

<br>

#### Connection 파라미터 제거

📌 먼저, Connection을 파라미터로 직접 전달하는 문제를 해결해보자!

- `upgradeLevels()` 메소드에서 생성된 Connection을 계속 파라미터로 전달해 DAO를 호출하는 방식은 피하고 싶음.
    - 스프링의 **트랜잭션 동기화** 방식을 사용하자!
        - `UserService`에서 트랜잭션을 시작하고 생성된 Connection을 **특별한 저장소에 보관**.
        - DAO 메소드에서는 이 저장소에서 **동기화된 Connection을 사용**.
        - **JdbcTemplate**은 동기화된 Connection을 자동으로 사용.
        - 트랜잭션 종료 시, 동기화된 Connection도 종료됨.

- 트랜잭션 동기화를 사용한 경우의 작업 흐름
    
    ![5 2-1](https://github.com/user-attachments/assets/17bedbc1-c54f-4477-bb68-5a7c21ba57f9)


    
    1. UserService는 Connection을 생성
    2. 이를 동기화 저장소에 저장해두고 트랜잭션을 시작시킨 후 DAO의 기능을 이용
    3. 첫 번째 update() 호출
    4. 현재 시작된 트랜잭션을 가진 Connection 오브젝트가 존재하는지 확인하고 
        
        *(2) upgradeLevels()* 메소드 시작 부분에서 저장해 둔 Connection을 발견하고 이를 가져옴
        
    5. Connection을 이용하여 PreparedStatement를 만들어 수정 SQL을 실행
        - 트랜잭션 동기화 저장소에서 DB 커넥션을 가져온 경우 Connection을 닫지 않은 채로 작업을 마침
    6. 두 번째 update()가 호출되면
    7. 트랜잭션 동기화 저장소에서 Connection을 가져와
    8. 사용한다.
    9. 마지막 update()도
    10. 같은 트랜잭션을 가진 Connection을 가져와
    11. 사용한다.
    12. 트랜잭션 내 모든 작업이 정상적으로 끝나면 commit()을 호출해서 완료
    13. 트랜잭션 저장소가 더 이상 Connection 오브젝트를 저장해두지 않도록 제거

<br>

#### 트랜잭션 동기화 적용

🤔 문제는 멀티스레드 환경에서도 안전한 트랜잭션 동기화 방법을 구현하는 일이 기술적으로 간단하지 않다는 점인데…

→ 다행히도 스프링은 JdbcTemplate과 더불어 이런 트랜잭션 동기화 기능을 지원하는 간단한 유틸리티 메소드를 제공하고 있다.

- 트랜잭션 동기화 방식을 적용한 UserService
    
    ```java
    public class UserService {
        private DataSource dataSource;
    
        // Connection을 생성할 때 사용할 dataSource를 DI 받도록 한다.
        public void setDataSource(DataSource dataSource) {
            this.dataSource = dataSource;
        }
    
        public void upgradeLvls() throws Exception {
            // 트랜잭션 동기화 관리자를 이용해 동기화 작업을 초기화한다.
            TransactionSynchronizationManager.initSynchronization();
    
            // DB 커넥션을 생성하고 트랜잭션을 시작한다.
            // 이후의 DAO 작업은 모두 여기서 시작한 트랜잭션 안에서 진행된다.
    
            // DB 커넥션 생성과 동기화를 함께 해주는 유틸리티 메소드
            Connection c = DataSourceUtils.getConnection(dataSource);
            // 트랜잭션 시작
            c.setAutoCommit(false);
    
            try {
                List<User> users = userDao.getAll();
                for (User user : users) {
                    if (canUpgradeLvl(user)) {
                        upgradeLvl(user);
                    }
                }
    
                // 정상적으로 작업을 마치면 트랜잭션 커밋
                c.commit();
            } catch (Exception e) {
                // 예외 발생 시 롤백
                c.rollback();
                throw e;
            } finally {
                // 스프링 유틸리티 메소드를 이용해 DB 커넥션을 안전하게 닫음
                DataSourceUtils.releaseConnection(c, dataSource);
                // 동기화 작업 종료 및 정리
                TransactionSynchronizationManager.unbindResource(this.dataSource);
                TransactionSynchronizationManager.clearSynchronization();
            }
        }
    }
    ```
    
    - UserService에서 DB 커넥션을 직접 다룰 때 DataSource가 필요하므로 해당 빈에 대한 DI 설정을 해둬야 한다.
    - 스프링이 제공하는 트랜잭션 동기화 관리 클래스 → TransactionSycronizationManager
    - 이 TransactionSycronizationManager 클래스를 이용해 먼저 트랜잭션 동기화 작업을 초기화하도록 요청
    - DB 커넥션 생성
        - DataSourceUtils에서 제공하는 `getConnection()` 메소드를 통해 생성
        - 일반 커넥션이 아니라 DataSourceUtils의 메소드를 쓰는 이유 : Connection 오브젝트 생성과 Connection 객체를 저장소에 바인딩 해 줌
    - 트랜잭션 시작
    - DAO 메소드를 사용하는 트랜잭션 내의 작업 진행
    - 작업을 정상적으로 마치면 commit()
    - 예외 발생 시 rollback()
    - DB 커넥션 닫기와 동기화 중단

<br>

#### 트랜잭션 테스트 보완

🤔 트랜잭션이 적용됐는지 테스트 해보자!

- 앞에서 만든 UserServiceTest의 upgradeAllOtNothing() 테스트 메소드에 dataSource 빈을 가져와 주입해주는 코드 추가
    
    ```java
    public class UserDaoTest {
        @Test
        public void upgradeAllOrNothing() throws Exception {
            UserService testUserService = new TestUserService(users.get(3).getId());
            testUserService.setUserDao(this.userDao);
            testUserService.setDataSource(this.dataSource);
    
            userDao.deleteAll();
            for  (User user : users) {
                userDao.add(user);
            }
    
            try {
                testUserService.upgradeLvls();
                fail("TestUserServiceException expected");
            } catch (TestUserServiceException e) {
            }
    
            checkLvlUpgraded(users.get(1), false);
        }
    }
    ```
    

- 나머지 테스트도 문제 없이 동작시키기 위해 UserService의 dataSource 프로퍼티 설정을 설정파일에 추가해야 한다.

<br>

#### JdbcTemplate과 트랜잭션 동기화

- JdbcTemplate의 동작 방식
    - 동기화 저장소에 등록된 DB 커넥션이나 트랜잭션이 없는 경우, 직접 DB 커넥션을 만들고 트랜잭션을 시작해서 JDBC 작업 진행
    - 트랜잭션 동기화가 시작되었다면 동기화 저장소에 있는 DB Connection을 가져와 사용

<br>
<br>

### 5.2.4 트랜잭션 서비스 추상화

#### 기술과 환경에 종속되는 트랜잭션 경계설정 코드

🤔 여러 개의 DB를 사용하는 G사에서 하나의 트랜잭션 안에서 여러 개의 DB에 데이터를 넣은 작업을 요청했다!

- 하지만, 한 개 이상의 DB로의 작업을 하나의 트랜잭션으로 만드는 건 로컬 트랜잭션으로는 불가능하다
    - 로컬 트랜잭션은 하나의 DB 커넥션에 종속되기 때문
    
    → 따라서, 글로벌 트랜잭션 방식을 사용해야 한다.
    

- 자바는 글로벌 트랜잭션을 지원하는 트랜잭션 매니저를 지원하기 위한 API인 JTA(Java Transaction API)를 제공한다
    - JTA를 통한 글로벌/분산 트랜잭션 관리
        
       ![5 2-2](https://github.com/user-attachments/assets/9984b0ea-09e3-4f82-b95d-91e40338ca88)

        

📌 결국 G사의 요청은 서버가 제공하는 트랜잭션 매니저와 트랜잭션 서비스를 사용할테니 JDBC API가 아닌 JTA를 사용해 트랜잭션을 관리하게 해달라는 것이다!

- JTA를 이용한 트랜잭션 코드 구조
    
    ```java
    // JNDI를 이용해 서버의 Transaction 오브젝트를 가져온다.
    InitialContext ctx = new InitialContext();
    UserTransaction tx = (UserTransaction)ctx.lookup(USER_TX_JNDI_NAME);
    
    tx.begin();
    // JNDI로 가져온 dataSource를 사용해야 한다.
    Connection c = dataSource.getConnection();
    try {
      // 데이터 액세스 코드
      tx.commit();
    } catch (Exception e) {
      tx.rollback();
      throw e;
    } finally {
      c.close();
    }
    ```
    
    - JTA를 이용한 방법으로 바뀌긴 했지만 트랜잭션 경계설정을 위한 구조는 JDBC를 사용했을 때와 비슷함.
    - 문제점: JDBC 로컬 트랜잭션을 JTA를 이용하는 글로벌 트랜잭션으로 바꾸려면 UserService 코드를 수정해야 한다는 점이다.
        - 로컬 트랜잭션을 사용하면 충분한 고객을 위해서는 JDBC를 이용한 트랜잭션 관리 코드를,
        - 다중 DB를 위한 글로벌 트랜잭션을 필요로 하는 곳을 위해서는 JTA를 이용한 트랜잭션 관리 코드를 적용해야 한다는 문제가 발생한다.

- 그러던 중, Y사에서 자신들이 하이버네이트를 이용해 UserDao를 직접 구현했다고 알려왔다.
    - 문제점: 하이버네이트를 이용한 트랜잭션 관리 코드는 JDBC나 JTA 코드와는 또 다르다…
    - 그렇게 되면 UserService를 하이버네이트의 Session과 Transaction 오브젝트를 사용하는 트랜잭션 경계설정 코드로 변경할 수 밖에 없다…!

<br>

#### 트랜잭션 API의 의존관계 문제와 해결책

🤔 이러한 문제를 어떻게 해결할 것인가?

- 트랜잭션 도입으로 인한 새로운 의존 관계
    
    ![5 2-3](https://github.com/user-attachments/assets/57f49304-36f2-4f78-a2cd-7cac1822a640)

    

- **문제 발생**:
    
    `UserService`는 원래 `UserDao` 인터페이스에만 의존하도록 설계되었기 때문에 데이터 액세스 기술이 변경되어도 `UserService` 코드에는 영향이 없었음.
    
    그러나, 트랜잭션 경계설정에 `Connection`을 사용하면서 JDBC에 종속성이 생기고, `UserService`가 `UserDaoJdbc`에 간접적으로 의존하게 되어 기존 구조의 장점이 퇴색됨.
    
- **해결 목표**:
    
    `UserService`의 트랜잭션 경계설정이 특정 기술에 종속되지 않도록 만들기.
    
    - 특정 API에 의존하지 않도록 `UserService` 코드에서 트랜잭션 코드 추상화.
- **추상화 개념 적용**:
    
    트랜잭션 경계설정은 JDBC, JTA, Hibernate 등 다양한 기술 간에 일정한 패턴이 존재.
    
    이를 활용해 **공통적인 특징을 추상화**할 수 있음.
    
- **예시와 추상화 기술**:
    - JDBC는 데이터베이스 간의 공통적인 특징(SQL 사용)을 추상화한 것처럼,**트랜잭션 처리**에도 **JDBC, JPA, Hibernate 등에서 공통적인 트랜잭션 개념**이 존재.
    - 이를 기반으로 **추상화된 트랜잭션 관리 계층**을 만들고, 이 계층의 API를 통해 트랜잭션을 설정하도록 함.
- **결과**:
    
    애플리케이션 코드가 특정 트랜잭션 기술에 종속되지 않고 일관된 방식으로 트랜잭션을 사용할 수 있어, 트랜잭션 관리 코드가 기술 독립적으로 유지됨.
    

<br>

#### 스프링의 트랜잭션 서비스 추상화

- 스프링의 트랜잭션 추상화 기술을 이용하면 애플리케이션에서 직접 각 기술의 트랜잭션 API를 이용하지 않고도, 일관된 방식으로 트랜잭션을 제어하는 트랜잭션 경계설정 작업이 가능해진다.
    
    ![5 2-4](https://github.com/user-attachments/assets/cc73d360-1bad-44fd-b117-80fae72d7d8b)

    

- 스프링이 제공하는 트랜잭션 추상화 방법을 UserService에 적용해보면 아래와 같은 코드로 만들 수 있다.
    
    ```java
    public void upgradeLevels() {
    		PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
        
        // 트랜잭션 시작
        TransactionStatus status =
                    transactionManager.getTransaction(new DefaultTransactionDefinition());
    
        try {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (canUpgradeLevel(user)) {
                    upgradeLevel(user);
                }
            }
    
            transactionManager.commit(status);
        }catch(Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
    ```
    
    - `PlatformTransactionManager` : 스프링이 제공하는 트랜잭션 경계설정을 위한 추상 인터페이스
    - JDBC의 로컬 트랜잭션을 이용한다면 → DataSourceTransactionManager 사용
    - 이렇게 시작된 트랜잭션은 `TransactionStatus` 타입의 변수에 저장
    - `Transactionstatus`는 트랜잭션에 대한 조작이 필요할 때 `PlatformTransactionManager` 메소드의 파라미터로 전달
    - 스프링의 트랜잭션 추상화 기술은 앞서 살펴봤던 트랜잭션 동기화를 사용
    - 트랜잭션 동기화 저장소에 트랜잭션을 저장해두고 해당 트랜잭션을 이용해 데이터 액세스 작업을 수행 후 마지막에 `commit`과 `rollback`을 결정

<br>

#### 트랜잭션 기술 설정의 분리

🤔 트랜잭션 추상화 API를 적용한 UserService 코드를 JTA를 이용하는 글로벌 트랜잭션으로 변경하려면 어떻게 해야 할까?

- `PlatformTransactionManager` 구현 클래스를 `DataSourceTransactionManager`에서 `JTATransactionManager`로 바꿔주기만 하면 됨.

- JTA로 바꾸기 위해 `upgradeLevels()` 메소드의 첫 줄을 다음과 같이 수정
    - `PlatformTransactionManager txManager = new JTATransactionManager();`
    - 만약 하이버네이트로 UserDao를 구현했다면 → `HibernateTransactionManager`를, JPA를 적용했다면 `JTATransactionManager`를 사용하면 됨.

→ 하지만 어떤 트랜잭션 매니저 구현 클래스를 사용할지 UserService 코드가 알고 있는 것은 DI 원칙에 위배됨.

→ 자신이 사용할 구체적은 클래스를 스스로 결정하고 생성하지 말고 컨테이너를 통해 외부에서 제공받게 하는 스프링 DI의 방식으로 바꾸자.

- `DataSourceTransactionManager` 는 스프링 빈으로 등록하고 `UserService` 가 DI 방식으로 사용하게 하자!
    
    ```java
    public class UserService {
        private PlatformTransactionManager transactionManager;
    
        public void setTransactionManager(PlatformTransactionManager transactionManager) {
            this.transactionManager = transactionManager;
        }
        
        ....
    
        public void upgradeLevels() {
            TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
            try{
                List<User> users = userDao.getAll();
                for(User user : users) {
                    if(canChangedLevel(user)) {
                        upgradeLevel(user);
                    }
                }
                
                this.transactionManager.commit(status);
            } catch (Exception e) {
                this.transactionManager.rollback(status);
                throw e;
            }
        }
        ....
    }
    ```
    

- `UserService`에 DI될 `transactionManager` 빈을 설정파일에 등록하자
    
    ```java
    <bean id="userService" class="springbook.user.service.UserService">
        <property name="userDao" ref="userDao"/>
        <property name="transactionManager" ref="transactionManager"/> //추가
    </bean>
        
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
    ```
    

- 테스트에서 트랜잭션 예외상황을 테스트하기 위해 수동 DI를 하는 메소드를 수정하자
    
    ```java
    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration(locations = "/test-applicationContext.xml")
    public class UserServiceTest {
        ...
    
        @Autowired
        PlatformTransactionManager transactionManager;
    
        ...
    
        @Test
        public void upgradeAllOrNothing() {
            UserService testUserService = new TestUserService(users.get(3).getId());
            testUserService.setUserDao(this.userDao);
            testUserService.setTransactionManager(this.transactionManager);//수동 DI 추가
    
            ...
        }
    }
    ```
    

✨ 이제 UserService는 트랜잭션 기술에서 완전히 독립적인 코드가 됐다!

<br>
<br>

## 5.3 서비스 추상화와 단일 책임 원칙

✨ 이제 스프링의 트랜잭션 서비스 추상화 기법을 이용해 다양한 트랜잭션 기술을 일관된 방식으로 제어할 수 있게 됐다

#### 수직, 수평 계층구조와 의존관계

✨ 이렇게 기술과 서비스에 대한 추상화 기법을 이용하면 특정 기술환경에 종속되지 않는 포터블한 코드를 만들 수 있다.

- UserDao와 UserService
    - 담당하는 코드의 기능적인 관심에 따라 분리되고, 독자적으로 확장이 가능하도록 만든 것
    - 같은 애플리케이션 로직을 담은 코드지만 내용에 따라 분리
    - 같은 계층에서 수평적인 분리라고 볼 수 있다.

- 트랜잭션 추상화
    - 아예 계층의 특성이 다른 코드를 분리한 것
    - 트랜잭션 추상화는 애플리케이션의 비즈니스 로직과 하위의 트랜잭션 기술을 분리

✨ 수평적인 구분이든, 수직적인 구분이든 모두 결합도가 낮으며, 서로 영향을 주지 않고 자유롭게 확장될 수 있는 구조를 만들 수 있는 데는 스프링의 DI가 중요한 역할을 한다.

✨ DI의 가치는 이렇게 관심, 책임, 성격이 다른 코드를 깔끔하게 분리하는데 있다!

<br>

#### 단일 책임 원칙

🤔 UserService에 JDBC 커넥션 메소드를 직접 사용하는 트랜잭션 코드가 들어 있었을 때롤 생각해보자.

- UserService는 사용자 레벨을 관리하는 것과 트랜잭션을 관리하는 것, 총 두 가지 책임을 가지고 있었다.
    - 두 가지 책임을 갖는다는 것? = 수정되는 이유가 두 가지라는 뜻

- 하지만, 트랜잭션 서비스의 추상화 방식을 도입하고, DI를 통해 외부에서 제어하도록 만들고 나서는 어떻게 됐을까?
    - 바뀔 이유가 한 가지 뿐이다.
    
    → 단일 책임 원칙을 충실하게 지키고 있다고 말할 수 있다.
    
<br>

#### 단일 책임 원칙의 장점

1. **변경 시 수정 대상이 명확함**
    - 단일 책임 원칙을 지키면 변경이 필요한 부분을 쉽게 찾아 수정할 수 있음.
    - 예를 들어, 기술이 바뀌면 기술 추상화 계층의 설정만 변경하면 되고, 데이터베이스 테이블 이름이 바뀌면 `UserDao`만 수정하면 됨.
2. **복잡한 구조에서도 유지보수 용이**
    - 모듈이 많아질 경우 단일 책임 원칙을 지키지 않으면 의존 관계가 복잡해지고, 특정 DAO가 변경될 때 그에 의존하는 서비스 클래스들도 수정해야 하는 문제가 발생.
    - 수백 개의 클래스를 수정해야 하는 상황을 방지할 수 있음.
3. **기술 변경 시 유연성 제공**
    - 애플리케이션 코드가 특정 기술에 종속되지 않도록 설계하면, 기술이 바뀌어도 XML 설정만으로 트랜잭션 기술을 한 번에 전환할 수 있음.
    - 코드 수정을 최소화해 실수를 줄이고, 치명적인 버그의 가능성을 낮춤.

- **스프링 DI의 역할**
    - 스프링의 DI(Dependency Injection)는 계층 간 결합도를 낮추고, 로우레벨 기술의 변화에 비즈니스 로직이 영향을 받지 않도록 함.
    - `PlatformTransactionManager`와 같은 인터페이스를 통해 트랜잭션 처리를 추상화함으로써, 트랜잭션 기술로부터 비즈니스 로직을 분리할 수 있음.

- **객체지향 설계 원칙의 적용**
    - 단일 책임 원칙 외에도 개방-폐쇄 원칙(OCP)과 낮은 결합도, 높은 응집도를 만족하는 설계가 가능해짐.
    - 이를 통해 전략 패턴, 어댑터 패턴, 브리지 패턴, 미디에이터 패턴 등 다양한 디자인 패턴을 자연스럽게 적용할 수 있음.
- **테스트 편의성 향상**
    - 스프링의 DI와 싱글톤 레지스트리를 통해 자동화된 테스트를 쉽게 만들 수 있음.
- **좋은 코드 설계와 지속적인 개선의 필요성**
    - 설계 원칙과 디자인 패턴을 단순히 외우는 것이 아닌, 꾸준히 좋은 코드와 유연한 설계를 고민하는 것이 중요함.
    - 스프링의 DI는 자바 엔터프라이즈 기술에서 발생하는 문제를 해결하고, 유연한 코드와 설계를 지원하는 핵심 도구로 자리 잡음.
- **스프링과 객체지향 기술의 학습 방법**
    - 스프링의 DI 원리를 따라가며 객체지향 설계와 패턴의 장점을 이해하고, 좋은 코드의 가치를 배울 수 있음.
    - 변경이 필요할 때 어디를 수정해야 하는지 이해함으로써 DI의 원리와 객체지향 설계를 체득할 수 있음.


<br>
<br>

## 5.4 메일 서비스 추상화

✨ 레벨이 업그레이드 되는 사용자에게 안내 메일을 발송하는 기능을 구현해보자!

- 사용자의 이메일 정보를 관리
- 메일 발송 기능 추가

<br>

### 5.4.1 JavaMail을 이용한 메일 발송 기능

- 사용자 정보에 이메일을 추가하는 일은 레벨을 추가했을 때와 동일하게 진행하면 된다.
    - DB의 User 테이블에 email 필드 추가
    - `User` 클래스에 email 프로퍼티 추가
    - UserDao에 email 필드 처리 코드를 추가
    - 테스트 코드 수정

<br>

#### JavaMail 메일 발송

- 자바에서 메일을 발송할 때 → JavaMail 사용
    - javax.mail 패키지에서 제공하는 자바의 이메일 클래스를 사용

→ JavaMail을 이용해 업그레이드 시 메일 발송 기능을 추가해보자!

- 레벨 업그레이드 작업 메소드 수정
    
    ```java
    protected void upgradeLevel(User user) {
        user.upgradeLevel();
        userDao.update(user);
        sendUpgradeEmail(user);
    }
    ```
    

- JavaMail API를 사용하는 메소드 추가
    
    ```java
    protected void upgradeLevel(User user) {
            user.upgradeLevel();
            userDao.update(user);
            sendUpgradeEmail(user);
        }
        
        private void sendUpgradeEmail(User user) {
            Properties props = new Properties();
            props.put("mail.smtp.host", "mail.test.org");
            Session s = Session.getInstance(props, null);
    
            MimeMessage message = new MimeMessage(s);
            
            try {
                message.setFrom(new InternetAddress("useradmin@test.org"));
                message.addRecipient(Message.RecipientType.TO, new InternatAddress(user.getEmail()));
                message.setSubject("Upgrade 안내");
                message.setText("사용자님의 등급이 "+user.getLevel().name() + "로 업그레이드되었습니다.");
    
                Transport.send(message);
            } catch(AddressException e) {
                throw new RuntimeException();
            } catch(MessagingException e) {
                throw new RuntimeException(e);
            } catch (UnsupportedEncodingException e) {
                throw new RuntimeException(e);
            }
        }
    ```
    
<br>
<br>

### 5.4.2 JavaMail이 포함된 코드의 테스트

🤔 만약 메일 서버가 준비되어 있지 않다면?

→ 예외가 발생하면서 테스트가 실패한다

- 테스트 실패의 원인
    - 메일을 발송하려는데 메일 서버가 현재 연결 가능하도록 준비되어 있지 않기 때문
    - 서버가 잘 준비되어 있다면 아무런 문제가 없을 것이다

- 그런데…
    
    테스트를 하면서 매번 메일이 발송되는 것이 바람직한가?
    
    - 메일 발송은 부하가 매우 큰 작업이므로 테스트를 실행할때마다 메일을 보내면 서버에 상당한 부담을 줄 수 있다.
    - 게다가, 메일이 실제로 발송된다는 문제도 있다

→ 테스트 때는 메일 서버 설정을 다르게 해서 테스트용으로 따로 준비된 메일 서버를 이용하는 건 어떨까?

- 실제 메일 서버를 사용하지 않고 메일 서버를 이용해 테스트하는 방법
    - 테스트 메일 서버는 외부로 메일을 발송하지 않고, 단지 JavaMail과 연동해서 메일 전송 요청을 받는 것까지만 담당한다.

- 똑같은 원리를 UserService와 JavaMail 사이에도 적용할 수 있지 않을까?
    - JaVaMail은 안정적인 모듈이므로 JavaMail API를 통해 요청이 들어갔다는 보장만 있다면 굳이 테스트를 할 때마다 JavaMAil을 직접 구동시킬 필요가 없다!
    - 테스트 중에는 JavaMail을 대신할 수 있는 동일한 인터페이스를 갖는 코드가 동작하도록 만들어도 될 것이다.

→ 불필요한 메일 전송 요청을 보내지 않아도 되고, 테스트도 빠르고 안전하게 수행된다!

<br>
<br>

### 5.4.3 테스트를 위한 서비스 추상화

#### JavaMail을 이용한 테스트의 문제점

🤔 한 가지 문제점…

- JavaMail의 핵심 API에는 인터페이스로 만들어져서 구현을 바꿀 수 있는게 없다!

→ 결국 JavaMail의 구현을 테스트용으로 바꾸는 것은 불가능하다.

→ JavaMail 대신 테스트용 JavaMail로 대체해서 사용하는 것은 포기해야 할까?

- 서비스 추상화를 적용하자!

<br>

#### 메일 발송 기능 추상화

- JavaMail의 서비스 추상화 인터페이스
    
    ```java
    public interface MailSender {
    	void send(SimpleMailMessage simpleMessage) throws MailException;
    	void send(SimpleMailMessage[] simpleMessages) throws MailException;
    }
    ```
    

- 스프링의 MailSender를 이용한 메일 발송 메소드
    
    ```java
    private void sendUpgradeEmail(User user) {
        JavaMailSenderImpl mailSender = new JavaMailSenderImpl(); // MailSender 구현 클래서의 오브젝트 생성
        mailSender.setHost("mail.server.com");
    
    		// MailMessage 인터페이스의 구현 클래스 오브젝ㅌ를 만들어 메일 내용 작성
        SimpleMailMessage mailMessage = new SimpleMailMessage();
        mailMessage.setTo(user.getEmail());
        mailMessage.setFrom("useradmin@test.org");
        mailMessage.setSubject("Upgrade 안내");
        mailMessage.setText("사용자님의 등급이 "+user.getLevel().name() + "로 업그레이드되었습니다.");
            
        mailSender.send(mailMessage);
    }
    ```
    
    - 발생하는 각종 예외를 MailException이라는 런타임 예외로 포장해서 던져주기 때문에 try/catch 블록이 없어진 것이 눈에 띈다.

- 이제, 스프링의 DI를 적용해보자
    - sendUpgradeEMail() 메소드에는 JavaMailSenderImpl 클래스가 구현한 MailSender 인터페이스만 남기고, 구체적인 메일 전송 구현을 담은 클래스의 정보는 코드에서 모두 제거
    
    ```java
    public class UserService {
        ...
        private MailSender mailSender;
    
        public void setMailSender(MailSender mailSender) {
            this.mailSender = mailSender;
        }
    	...
        
        private void sendUpgradeEmail(User user) {
            SimpleMailMessage mailMessage = new SimpleMailMessage();
            mailMessage.setTo(user.getEmail());
            mailMessage.setFrom("useradmin@test.org");
            mailMessage.setSubject("Upgrade 안내");
            mailMessage.setText("사용자님의 등급이 "+user.getLevel().name() + "로 업그레이드되었습니다.");
    
            this.mailSender.send(mailMessage);
        }
    }
    ```
    
    - 설정파일 안에 JavaMailSenderImpl 클래스로 빈을 만들고 UserService에 DI 해준다.

- 메일 발송 오브젝트의 빈 등록
    
    ```java
    <bean id="userService" class="springbook.user.service.UserService">
            <property name="userDao" ref="userDao"/>
            <property name="transactionManager" ref="transactionManager"/>
            <property name="mailSender" ref="mailSender"/>
    </bean>
        
    <bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">
        <property name="host" value="mail.server.com"/>
    </bean>
    ```
    
<br>

#### 테스트용 메일 발송 오브젝트

🤔 스프링이 제공한 메일 전송 기능에 대한 인터페이스가 있으니 이를 구현해서 테스트용 메일 전송 클래스를 만들어보자!

- 구현해야 할 인터페이스: MailSender

- 아무런 기능이 없는 MailSender 구현 클래스
    
    ```java
    public class DummyMailSender implements MailSender {
    
        @Override
        public void send(SimpleMailMessage simpleMailMessage) throws MailException {
            
        }
    
        @Override
        public void send(SimpleMailMessage... simpleMailMessages) throws MailException {
    
        }
    }
    ```
    

- 테스트용 UserService를 위한 메일 전송 오브젝트의 수동 DI
    
    ```java
    @Test
    public void upgradeAllOrNothing() {
        UserService testUserService = new TestUserService(users.get(3).getId());
            ...
        testUserService.setMailSender(mailSender);
            ...
    }
    ```
    
    - 테스트는 모두 성공으로 끝난다!

<br>

#### 테스트와 서비스 추상화

![5 4](https://github.com/user-attachments/assets/c0f7f48d-4ed7-431a-86c7-136f54d06a07)


- 스프링이 제공하는 MailSender 인터페이스를 핵심으로 하는 메일 전송 서비스 추상화의 구조

- 스프링이 직접 제공하는 MailSender를 구현한 추상화 클래스는 JavaMailServiceImpl 하나 뿐이지만, 이를 사용해 애플리케이션을 작성할 때 얻을 수 있는 장점이 크다.

🤔 현재의 구현에서는 메일 발송 작업에 트랜잭션 개념이 빠져 있음. 

- 예를 들어, 사용자 레벨 업그레이드 작업 중 예외가 발생하여 DB 작업은 롤백됐다고 하자.
- 메일은 사용자별로 업그레이드 처리할때 이미 발송해 버렸다면 어떻게 취소할 것인가?
    
    → 따라서, 메일 발송 기능에도 트랜잭션 개념을 적용해야 한다.
    
- 해결 방법 1
    - 메일을 업그레이드할 사용자를 발견했을 때마다 발송하지 않고 발송 대상을 별도의 목록에 저장해두는 것
    - 업그레이드 작업이 모두 성공적으로 끝났을 때 한 번에 전송
    - 단점
        - 메일 저장용 리스트를 파라미터로 계속 가지고 다녀야 함.

- 해결 방법2
    - MailSender를 확장해서 메일 전송에 트랜잭션 개념을 적용하는 것
    - MailSender를 구현한 트랜잭션 기능이 있는 메일 전송용 클래스를 만든다.

✨ 특히, 외부 리소스와 연동하는 작업 대부분은 추상화 계층을 통해 관리하는 것이 좋음!

<br>
<br>

### 5.4.4 테스트 대역

✨ `DummyMailSender`에 대해 좀 더 생각해보자.

- `DummyMailSender` 클래스는 메소드가 비어 있지만 가치는 매우 크다
- 이 클래스를 이용해 메일을 직접 발송하는 클래스를 대치하지 않았다면 테스트는 매우 불편해지고 자주 실행하기 힘들었을 것이다.

→ 이처럼 테스트 환경에서 유용하게 사용하는 기법이 있다.

- 대부분 테스트할 대상이 의존하고 있는 오브젝트를 DI를 통해 바꿔치기하는 것

<br>

#### 의존 오브젝트의 변경을 통한 테스트 방법

- `UserDaoTest`를 통해 테스트가 진행될 때의 상황
    - 원래 `UserDao`
        - 운영 시스템에서 사용하는 DB와 연결돼서 동작
        - 대용량의 DB 연결 기능에 최적화된 WAS에서 동작하는 DB 풀링 서비스를 사용
        - 최적화된 복잡한 DataSource의 구현 클래스를 이용
    - 하지만 테스트에서는…
        - 번거로운 짐이 될 뿐이다!
    
    → 대신할 수 있도록 테스트 환경에서도 잘 동작하고 준비 과정도 간단한 DataSource를 사용
    
    & DB도 가벼운 버전을 이용
    
- UserService의 테스트
    - 실제 UserService가 운영 시스템에서 사용될 때…
        - JavaMailSenderImpl과 JavaMail을 통해 메일 서버로 이어지는 구성이 필요
    - 하지만 테스트 때는…
        - 일일이 구조를 유지하고 있다면 오히래 손해!
    - `UserServiceTest`의 관심사는 사용자 정보를 가공하는 비즈니스 로직이기 때문!

- 하지만… 메일 전송 기능을 아예 제외하는 건 불가함
    - 테스트 때는 제거했다가 테스트가 끝나면 다시 추가하는 것이 현실적으로 불가능함
    
    → DummyMailSender 도입
    
- 이처럼 테스트 대상인 오브젝트가 의존 오브젝트를 갖고 있기 때문에 발생하는 테스트 상의 문제는 → 간단한 환경으로 만들어주거나 아무런 일을 하지 않는 빈 오브젝트로 대치하자
    - 스프링 DI는 이럴 때 큰 위력을 발휘한다.

<br>

#### 테스트 대역의 종류와 특징

✨ 테스트 대역(test double)?

- 테스트 환경을 만들어주기 위해, 테스트 대상이 되는 오브젝트의 기능에만 충실하게 수행하면서 빠르게, 자주 테스트를 실행할 수 있도록 사용

- 테스트 스텁(test stub)
    - 대표적인 테스트 대역
    - 테스트 대상 오브젝트의 의존객체
    - 존재하면서 테스트 동안에 코드가 정상적으로 수행할 수 있도록 돕는 것
    - 테스트 코드 내부에서 간접적으로 사용
        - 따라서, DI 등을 통해 미리 의존 오브젝트를 테스트 스텁으로 변경해야 함.
    - DummyMailSender

- 테스트 대상 오브젝트의 메소드가 돌려주는 결과뿐만 아니라 테스트 오브젝트가 간접적으로 의존 오브젝트에 넘기는 값과 그 행위 자체에 대해서도 검증하고 싶다면?
    - 목 오브젝트를 이용하자!

- 목 오브젝트(mock object)?
    - 테스트 대상의 간접적인 출력 결과 검증
    - 테스트 대상 오브젝트와 의존 오브젝트 사이에서 일어나는 일을 검증할 수 있도록 특별히 설계됨.

<br>

#### 목 오브젝트를 이용한 테스트

📌 UserServiceTest에 목 오브젝트 개념을 적용해보자.

- 정상적인 사용자 레벨 업그레이드 결과를 확인하는 upgradeLevels() 테스트에서는 메일 전송 자체에 대해서도 검증해야 한다.
    - 목 오브젝트를 만들어서 발송 여부를 확인해보자!

- DummyMailSender 대신에 새로운 MailSender를 대체할 클래스를 하나 만들자.
    
    ```java
    public class MockMailSender implements MailSender {
        private List<String> requests = new ArrayList<String>();
    
        public List<String> getRequests() {
            return requests;
        }
    
        @Override
        public void send(SimpleMailMessage simpleMailMessage) throws MailException {
            requests.add(simpleMailMessage.getTo()[0]);
        }
    
        @Override
        public void send(SimpleMailMessage... simpleMailMessages) throws MailException {
    
        }
    }
    ```
    
    - DummyMailSender와 비슷하지만 테스트 대상이 send() 메소드를 통해 메일 전송 요청을 보냈을 때 관련 정보를 저장해두는 기능이 있다.
    - 전달받은 SimpleMailMessage 오브젝트에서 첫 번째 수신자 메일 주소를 꺼내온다.
    - 수신자 메일 주소를 미리 준비해둔 리스트에 저장한다.
    - 리스트이므로 순서대로 중복 호출까지 포함해서 다 저장한다.

- 테스트 코드를 수정해서 목 오브젝트를 통해 메일 발송 여부를 검증하자.
    
    ```java
    @Test
    @DirtiesContext//컨텍스트 DI 설정을 변경하는 테스트임을 알림
    public void upgradeLevels() {
        userDao.deleteAll();
        for (User user : users){
            userDao.add(user);
        }
    
    		//메일을 수신받는 사용자 정보를 확인 할 수 있도록 userService의 의존 오브젝트로 주입
        MockMailSender mockMailSender = new MockMailSender();
        userService.setMailSender(mockMailSender);
    
        userService.upgradeLevels();
    
        checkLevel(users.get(0), false);
        checkLevel(users.get(1), true);
        checkLevel(users.get(2), false);
        checkLevel(users.get(3), true);
        checkLevel(users.get(4), false);
    
        List<String> requests = mockMailSender.getRequests();
        assertThat(requests.size(), is(2));
        assertThat(requests.get(0), is(users.get(1).getEmail()));
        assertThat(requests.get(1), is(users.get(3).getEmail()));
    }
    ```
    
    - 메소드를 호출하기에 앞서 스프링 설정을 통해 DI된 DummyMailSender를 대신해서 사용할 메일 전송 검증용 목 오브젝트를 준비
    - MockMailSender의 오브젝트를 만들고 UserService 오브젝트의 수정자를 이용해 수동 DI 한다.
    - 테스트 대상의 메소드를 호출하고 업그레이드 결과에 대한 검증 과정을 거친다
    - 테스트를 수행하면 모두 성공일 것이다.

- 목 오브젝트를 이용한 테스트
    - 작성하기는 간단하고 기능은 막강하다
    - 검증하기가 까다로운 테스트 대상 오브젝트의 내부에서 일어나는 일이나 다른 오브젝트 사이에서 주고받는 정보까지 검증하는 일이 손쉽기 때문!

✨ 스텁 오브젝트, 목 오브젝트 두 가지 모두 테스트 대역의 가장 대표적인 방법이며 효과적인 테스트 코드를 작성하는 데 빠질 수 없는 중요한 도구이다.
<br>
