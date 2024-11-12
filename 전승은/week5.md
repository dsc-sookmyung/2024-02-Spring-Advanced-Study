# 5장 서비스 추상화

 
> 학습 목표 : DAO에 트랜잭션을 적용해보면서 스프링이 어떻게 성격이 비슷한 여러 종류의 기술을 추상화하고 이를 일관된 방법으로 사용할 수 있도록 지원하는지를 살펴보자.

## 5.1 사용자 레벨 관리 기능 추가
정기적으로 사용자의 활동내역을 참고해서 레벨을 조정해주는 기능을 추가하자.
### 5.1.1 필드 추가
- Level 이늄
	- User 클래스에 사용자의 레벨을 저장할 필드를 추가
	- 각 레벨을 코드화해서 숫자로 넣기
		1. 자바의 User에 추가할 프로퍼티 타입도 숫자
-> 의미 없는 숫자를 프로퍼티에 사용하면 타입이 안전하지 않아 위험
		2. 상수 값을 정해놓고 int 타입으로 레벨을 사용
`` private static final int BASIC = 1;``
-> level의 타입이 int이기 때문에 다른 종류의 정보를 넣는 실수를 해도 컴파일러가 체크해주지 못함
		3. 자바 5 이상에서 제공하는 이늄(enum) 사용
``` java
public enum Level {
    BASIC(1), // BASIC은 생성자 Level(1)을 호출
    SILVER(2), // SILVER는 생성자 Level(2)을 호출
    GOLD(3); // GOLD는 생성자 Level(3)을 호출

    private final int value;

    Level(int value) { // 생성자
        this.value = value; // 전달된 값을 value 필드에 저장
    }

    public int intValue() {	// 값을 가져오는 메소드
        return value;
    }
	public static Level valueOf(int value) { // 값으로부터 level 타입 오브젝트를 가져오도록 만든 스태틱 메소드
		switch(value) {
		case 1: return BASIC;
		case 2: return SILVER;
		case 3: return GOLD;
		default: throw new AssertionError("Unknown value: " + value);
		}
	}
}
```

- User 필드 추가
```java
public class User {
	String id;
	String name;
	String password;
	Level level;	// level 추가
	int login;	// 로그인 횟수, level에 사용
	int recommend;	// 추천 횟수, level에 사용
	...
	
	// User 클래스 생성자의 파라미터 추가
	public User(String id, String name, String password, Level level,
			int login, int recommend) {
		this.id = id;
		this.name = name;
		this.password = password;
		this.level = level;
		this.login = login;
		this.recommend = recommend;
	}
	
	...
	public Level getLevel() {
		return level;
	}

	public void setLevel(Level level) {
		this.level = level;
	}
}
```
- UserDaoTest 테스트 수정
1. 테스트 픽스처에 새 필드 값 추가
2. User 클래스 생성자의 파라미터 추가 (위에 참조)
3. 두 개의 User 오브젝트 필드 값이 모두 같은지 비교하는 checkSameUser() 메소드 수정
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
4. addAndGet() 메소드에 assertThat() -> checkSameUser() 로 변경
	```java
	@Test 
	public void andAndGet() {		
		...
		
		User userget1 = dao.get(user1.getId());
		checkSameUser(userget1, user1);
		
		User userget2 = dao.get(user2.getId());
		checkSameUser(userget2, user2);
	}
	```
- UserDaoJdbc 수정
INSERT 문장이 들어있는 add() 메소드의 User 오브젝트 매핑용 콜백인 userMapper에 추가된 필드를 넣는다.
```java
public class UserDaoJdbc implements UserDao {
	private RowMapper<User> userMapper = 
		new RowMapper<User>() {
				public User mapRow(ResultSet rs, int rowNum) throws SQLException {
				User user = new User();
				user.setId(rs.getString("id"));
				user.setName(rs.getString("name"));
				user.setPassword(rs.getString("password"));
				user.setLevel(Level.valueOf(rs.getInt("level")));	// int 타입의 값을 level 타입의 이늄 오브젝트로 만들어서 setLevel() 메소드에 넣어줌
				user.setLogin(rs.getInt("login"));
				user.setRecommend(rs.getInt("recommend"));
				return user;
			}
		};

// Level 이늄은 오브젝트이므로 DB에 저장될 수 있는 SQL 타입이 아니기 때문에 저장가능한 정수형 값으로 변환해야함
	public void add(User user) {
		this.jdbcTemplate.update(
				"insert into users(id, name, password, level, login, recommend) " +
				"values(?,?,?,?,?,?)", 
					user.getId(), user.getName(), user.getPassword(), 
					user.getLevel().intValue(), // 각 level 이늄의 db 저장용 값을 얻기 위해 사용
					user.getLogin(), user.getRecommend());
	}
```

++ 빠르게 실행 가능한 포괄적인 테스트를 만들어두면 기능의 추가나 수정이 일어날 때 좋음

### 5.1.2 사용자 수정 기능 추가
수정할 정보가 담긴 User 오브젝트를 전달하면 id를 참고해서 사용자를 찾아 필드 정보를 UPDATE 문을 이용해 모두 변경해주는 메소드 만들기
- 수정 기능 테스트 추가
	```java
	@Test
	public void update() {
		dao.deleteAll();
		// 픽스처 오브젝트 등록
		dao.add(user1);		// 수정할 사용자
		
		// 픽스처에 들어 있는 정보를 변경해서 수정 메소드를 호출
		user1.setName("오민규");
		user1.setPassword("springno6");
		user1.setLevel(Level.GOLD);
		user1.setLogin(1000);
		user1.setRecommend(999);
		// id를 제외한 필드의 내용을 바꾼 뒤 update()를 호출
		dao.update(user1);
		
		User user1update = dao.get(user1.getId());
		checkSameUser(user1, user1update);
		
	}
	```
- UserDao와 UserDaoJdbc 수정
	- UserDao 인터페이스에 update() 메소드를 추가
	- UserDaoJdbc의 update() 메소드 : JdbcTemplate의 update() 기능을 사용해서 UPDATE 문과 바인딩할 파라미터를 전달
	```java
	public void update(User user) {
		this.jdbcTemplate.update(
				"update users set name = ?, password = ?, level = ?, login = ?, " +
				"recommend = ? where id = ? ", user.getName(), user.getPassword(), 
		user.getLevel().intValue(), user.getLogin(), user.getRecommend(),
		user.getId());
		
	}
	```
- 수정 테스트 보완
	- 문제점 
		- UPDATE는 WHERE가 없어도 아무런 경고 없이 정상적으로 동작
		- 현재 update() 테스트는 수정할 로우의 내용이 바뀐 것만 확인할 뿐이지, 수정하지 않아야 할 로우의 내용이 그대로 남아 있는지 확인 X.
	- 해결 방법
		1. JdbcTemplate의 update()는 UPDATE나 DELETE 같이 테이블의 내용에 영향을 주는 SQL 을 실행하면 영향받은 로우의 개수를 돌려준다.
		UserDao의 add(), deleteAll(), update() 메소드의 리턴 타입을 int로 바꾸고 이 정보를 리턴하게 만들고 update()에 이 값이 1인지 확인하는 코드 추가
		2. 사용자를 두 명 등록해놓고, 그중 하나만 수정한 뒤에 수정된 사용자와 수정하지 않은 사용자의 정보를 모두 확인
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
			
			dao.update(user1);	// user2에도 적용됨
			
			User user1update = dao.get(user1.getId());
			checkSameUser(user1, user1update);
			User user2same = dao.get(user2.getId());
			checkSameUser(user2, user2same);	// user2same은 user1과 같음
		}
		```
		update() 메소드의 sql에서 WHERE을 빼먹었다면 테스트 실패

### 5.1.3 UserService.upgradeLevels()
UserDao의 getAll() 메소드로 사용자를 다 가져와서 사용자별로 레벨 업그레이드 작업을 진행하면서 UserDao의 update()를 호출해 DB에 결과를 넣어줌
- 사용자 관리 비즈니스 로직을 담을 UserService 클래스 생성
UserService는 UserDao의 구현 클래스가 바뀌어도 영향받지 않도록 해야함 
-> DAO의 인터페이스를 사용하고 DI를 적용 
-> UserService도 스프링의 빈으로 등록
- UserService 클래스와 빈 등록
```java
public class UserService {

	private UserDao userDao;	// UserDao 오브젝트를 저장해둘 인스턴스 변수 선언

	// UserDao 오브젝트의 DI가 가능하도록 수정자 메소드 추가
	public void setUserDao(UserDao userDao) {
		this.userDao = userDao;
	}
}
```
프로퍼티 추가
```xml
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
		<property name="dataSource" ref="dataSource" />
	</bean> 

	<bean id="userService" class="springbook.user.service.UserService">
		userDao 빈을 DI 받도록 프로퍼티 추가
		<property name="userDao" ref="userDao" /> 
	</bean>
```

- UserServiceTest 테스트 클래스
UserServiceTest 클래스를 추가하고 테스트 대상인 UserService 빈을 제공받을 수 있도록 @Autowired가 붙은 인스턴스 변수로 선언 해줌
	```java
	@RunWith(SpringJUnit4ClassRunner.class)
	@ContextConfiguration(locations="/test-applicationContext.xml")
	public class UserServiceTest {
		@Autowired 	
		UserService userService;	
	}
	```	
- upgradeLevels() 메소드
```java
public void upgradeLevels () {
	List<User> users = userDao.getAll();
	for(User user : users) {
		Boolean changed = null; // 레벨의 변화가 있는지를 확인하는 플래그
		// BASIC 레벨 업그레이드 작업
		if (user.getLevel() == Level.BASIC && user. getLogin() >= 50) {
			user.setLevel (Level. SILVER);	
			changed = true;
		}
		// SILVER 레벨 업그레이드 작업
		else if (user.getLevel() == Level.SILVER && user .getRecommend () >= 30) {
			user.setLevel(Level.GOLD); 
			Changed = true; // 레벨 변경 플래그 설정
		}
		else if (user.getLevel () == Level.GOLD) { changed = false; } // GOLD 레벨은 변경이 일어나지 않는다
		else { changed = false; } // 일치하는 조건이 없으면 변경 없음
		if (changed) { userDao.update(user); } // 레벨의 변경이 있는 경우에만 update() 호출
	}
}
```
		
- upgradeLevels() 테스트
	```java
	// test fixture
	@Before
	public void setUp() {
		users = Arrays.asList( // 배열을 리스트로 만들어주는 편리한 메소드, 배열을 가변인자로 넣어주면 더욱 편리하다.
			new User("bumjin", "박범진", "p1", Level.BASIC, 49, 0),	// 업그레이드 x
			new User("joytouch", "강명성", "p2", Level.BASIC, 50, 0), // 업그레이드 o
			new User("erwins", "신승한", "p3", Level.SILVER, 60, 29),// 업그레이드 	x
			new User("madnite1", "이상호", "p4", Level.SILVER, 60, 30), // 업그레이드 o
			new User("green", "오민규", "p5", Level.GOLD, 100, 100) // GOLD -> 변경 x
				);
	}

	@Test
	public void upgradeLevels() {
		userDao.deleteAll();
		for(User user : users) userDao.add(user);
		
		userService.upgradeLevels();
		// 각 사용자별로 업그레이드 후의 예상 레벨을 검증
		checkLevelUpgraded(users.get(0), false);
		checkLevelUpgraded(users.get(1), true);
		checkLevelUpgraded(users.get(2), false);
		checkLevelUpgraded(users.get(3), true);
		checkLevelUpgraded(users.get(4), false);
	}

	// DB에서 사용자 정보를 가져와 레벨을 확인하는 코드가 중복되므로 메소드로 분리
	private void checkLevelUpgraded(User user, boolean upgraded) {
		User userUpdate = userDao.get(user.getId());
		if (upgraded) {
			assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel()));
		}
		else {
			assertThat(userUpdate.getLevel(), is(user.getLevel()));
		}
	}
	```
### 5.1.4 UserService.add()
처음 가입하는 사용자는 기본적으로 BASIC 레벨이어야 한다.
-> 사용자 관리에 대한 비즈니스 로직을 담고 있는 UserService에 이 로직을 추가
- 방법 : add()를 호출할때 level의 값이 비어 있으면 로직을 따라서 basic을 부여해주고, 미리 설정된 레벨을 가진 User 오브젝트인 경우에는 그대로 둠
- User 오브젝트의 레벨이 변경됐는지 확인할 수 있는 테스트 케이스
1. UserService의 add() 메소드를 호출할 때 파라미터로 넘긴 User 오브젝트에 level 필드를 확인
2. UserDao의 get() 메소드를 이용해서 DB에 저장된 User 정보를 가져와 확인 -> 확실한 방법
	```java
	@Test 
	public void add() {
		userDao.deleteAll();
		
		User userWithLevel = users.get(4);	  // GOLD 레벨  
	
		// 레벨이 비어있는 사용자, 로직에 따라 등록 중에 BASIC 레벨로 설정돼야 한다.
		User userWithoutLevel = users.get(0);  
		userWithoutLevel.setLevel(null);
		
		userService.add(userWithLevel);	  
		userService.add(userWithoutLevel);
		
		// DB에 저장된 결과를 가져와 확인한다.
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

### 5.1.5 코드 개선
1️⃣ 코드에 중복된 부분은 없는가?
2️⃣ 코드가 무엇을 하는 것인지 이해하기 불편하지 않은가?
3️⃣ 코드가 자신이 있어야할 자리에 있는가?
4️⃣ 앞으로의 변경이 일어난다면 어떤 것이 있을 수 있고, 그 변화에 쉽게 대응할 수 있게 작성되어 있는가?

- upgradeLevels() 메소드 코드의 문제점
	for 루프 속에 들어있는 if/elseif/else 블록들이 레벨의 변화 단계와 업그레이드 조건, 조건이 충족됐을 때 해야할 작업이 한데 섞여 있어서 로직을 이해하기가 쉽지 않음 
	-> 성격이 조금씩 다른 것들이 섞여 있거나 분리돼서 나타나는 구조
- upgradeLevels() 리팩토링
	기본 작업 흐름만 남겨둔 upgradeLevels()
	```java
	public void upgradeLevels() {
		List<User> users = userDao.getAll();  // 모든 사용자 정보를 가져옴
		for(User user : users) {  
			if (canUpgradeLevel(user)) {  // 한 명씩 업그레이드가 가능한지 확인
				upgradeLevel(user);  // 업그레이드
			}
		}
	}
	```
	주어진 user에 대해 업그레이드가 가능하면 true, 가능하지 않으면 false 리턴
	```java
	// 업그레이드 가능 확인 메소드
	private boolean canUpgradeLevel(User user) {
		Level currentLevel = user.getLevel(); 
		
	   // 레벨별로 구분해서 조건을 판단
		switch(currentLevel) {                                   
		case BASIC: return (user.getLogin() >= 50); 
		case SILVER: return (user.getRecommend() >= 30);
		case GOLD: return false;
		
		default: throw new IllegalArgumentException("Unknown Level: " +
		 currentLevel); // 현재 로직에서 다룰 수 없는 레벨이 주어지면 예외를 발생시킴.
						// 새로운 레벨이 추가되고 로직을 수정하지 않으면 에러가 나서 확인할 수 있다. 
		}
	}
	```
	레벨의 순서와 다음 단계 레벨이 무엇인지를 결정 -> Level
	```java
	public enum Level {
		GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);  
		ㄴ 이늄 선언에 DB에 저장할 값과 함께 다음 단계의 레벨 정보도 추가
		
		private final int value;
		private final Level next; // 다음 단계의 레벨 정보를 스스로 갖고 있도록 Level 타입의 next 변수를 추가
		
		// 생성자 파라미터를 추가해서 다음 단계 레벨 정보를 지정할 수 있게 해줌
		Level(int value, Level next) {  
			this.value = value;
			this.next = next; 
		}
		...
		
		// 다음 레벨이 무엇인지 알려줌
		public Level nextLevel() { 
			return this.next;
		}	
		 …
	}
	```
	사용자 정보가 바뀌는 부분 -> User
	```java
	public void upgradeLevel() {
		Level nextLevel = this.level.nextLevel();	// 레벨의 다음 단계가 무엇인지 확인
		if (nextLevel == null) { // 예외 상황 검증								
			throw new IllegalStateException(this.level + "은  업그레이드가 불가능합니다");
		}
		else {
			this.level = nextLevel;
		}	
	}
	```
	간결해진 upgradeLevel()
	```java
	private void upgradeLevel(User user) {
		user.upgradeLevel();
		userDao.update(user);
	}
	```
	객체지향적인 코드는 다른 오브젝트의 데이터를 가져와서 작업하는 대신 데이터를 갖고 있는 다른 오브젝트에게 작업을 해달라고 요청한다.

- User 테스트
```java
public class UserTest {
	User user;
	
	스프링의 테스트 컨텍스트를 사용하지 않아도 됨
	(스프링이 IoC로 관리해주는 오브젝트가 아니지 때문)
	-> 컨테이너가 생성한 오브젝트를 @Autowired로 가져오는 대신 
	// 생성자를 호출
	@Before
	public void setUp() {
		user = new User();
	}
	
	@Test()
	public void upgradeLevel() {
		Level[] levels = Level.values(); // 1. 이늄에 정의된 모든 레벨을 가져옴
		for(Level level : levels) {
			if (level.nextLevel() == null) continue;
			user.setLevel(level);	// 1에서 가져온 레벨을 User에 설정
			user.upgradeLevel();
			assertThat(user.getLevel(), is(level.nextLevel()));	// 다음 레벨로 바뀌는지 확인
		}
	}
	
	@Test(expected=IllegalStateException.class)
	public void cannotUpgradeLevel() {
		Level[] levels = Level.values();
		for(Level level : levels) {
		// nextLevel()이 null인 경우에 강제로 upgradeLevel()을 호출
			if (level.nextLevel() != null) continue; 
			user.setLevel(level);
			user.upgradeLevel();
		}
	}

}
```
- UserServiceTest 개선
기존 테스트에서는 checkLevel() 메소드를 호출할 때 일일이 다음 단계의 레벨이 무엇인지 넣어줌 
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
	
	 어떤 레벨로 바뀔 것인가가 아니라, 다음 레벨로 업그레이드 될 것인가 아닌가를 지정한다
													 🔼
	private void checkLevelUpgraded(User user, boolean upgraded) {
		User userUpdate = userDao.get(user.getId());
		if (upgraded) {
			// 업그레이드가 일어났는지 확인
			assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel())); // 다음 레벨이 무엇인지는 Level에게 물어봄
		}
		else {
			// 업그레이드가 안 일어났는지 확인
			assertThat(userUpdate.getLevel(), is(user.getLevel()));
		}
	}
	```
	업그레이드 조건인 로그인 횟수와 추천 횟수가 애플리케이션 코드와 테스트 코드에서 중복돼서 나타남
	```java
	public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
	public static final int MIN_RECCOMEND_FOR_GOLD = 30;

	private boolean canUpgradeLevel(User user) {
		Level currentLevel = user.getLevel(); 
		switch(currentLevel) {                                   
		case BASIC: return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER); 
		case SILVER: return (user.getRecommend() >= MIN_RECCOMEND_FOR_GOLD);
		case GOLD: return false;
		default: throw new IllegalArgumentException("Unknown Level: " + currentLevel); 
		}
	}
	```
	UserServiceTest에도 UserService에 정의해둔 상수를 사용하도록
	```
	import static springbook.user.service.UserService.MIN_LOGCOUNT_FOR_SILVER;
	import static springbook.user.service.UserService.MIN_RECCOMEND_FOR_GOLD;
	```
	숫자로만 되어있는 경우에는 비즈니스 로직을 상세히 코멘트로 달아놓거나 설계문서를 참조하기 전에는 이해하기 힘들었던 부분이 이제는 무슨 의도로 어떤 값을 넣었는지 이해하기 쉬워졌다.
	
💡 사용자 업그레이드 정책을 UserService에서 분리하는 방법 
	
## 5.2 트랜잭션 서비스 추상화
사용자 레벨 조정 작업은 중간에 문제가 발생해서 작업이 중단된다면 그때까지 진행된 변경 작업을 모두 취소시켜야 함
### 5.2.1 모 아니면 도
지금까지 만든 사용자 레벨 업그레이드 코드는 어떻게 동작?
- 테스트용 UserService 대역
	- UserService를 대신해서 테스트의 목적에 맞게 동작하는 클래스를 만들어 사용 (테스트 클래스 내부에 스태틱 클래스로 만듦)
	- UserService를 상속해서 테스트에 필요한 기능을 추가하도록 일부 메소드를 오버라이딩
	- 네 번째 사용자를 처리하는 중에 예외를 발생시키고, 그 전에 처리한 두 번째 사용자의 정보가 취소됐는지, 아니면 그대로 남아있는지를 확인
	- UserService의 upgradeLevel() 메소드 접근권한을 private -> protected로 수정해서 상속을 통해 오버리이딩이 가능하게 함
```java
static class TestUserService extends UserService {
		private String id;
		
		private TestUserService(String id) {  
			this.id = id;	// 예외를 발생시킬 User 오브젝트의 id를 지정할 수 있게 만든다.
		}

		protected void upgradeLevel(User user) { // UserService의 메소드를 오버라이드한다.
			if (user.getId().equals(this.id)) throw new TestUserServiceException();  
			super.upgradeLevel(user); ㄴ 지정된 id의 User 오브젝트가 발견되면 예외를 던져서 작업을 강제로 중단시킨다. 
			ㄴ 원래꺼 호출
		}
	}
	// 테스트 클래스 내에 스태틱 멤버 클래스로 테스트 예외 만듦	
	static class TestUserServiceException extends RuntimeException {
	}
```

- 강제 예외 발생을 통한 테스트
	```java
	@Test
	public void upgradeAllOrNothing() throws Exception {  
		// 예외를 발생시킬 네 번째 사용자의 id를 넣어서 테스트용 UserService 대역 오브젝트를 생성
		UserService testUserService = new TestUserService(users.get(3).getId());  
		testUserService.setUserDao(this.userDao); // UserDao를 수동 DI
		testUserService.setDataSource(this.dataSource);
		 
		userDao.deleteAll();			  
		for(User user : users) userDao.add(user);
		
		try {	// TestUserService는 업그레이드 작업 중에 예외가 발생해야 함. 정상 종료라면 문제가 있으니 실패
			testUserService.upgradeLevels();   
			fail("TestUserServiceException expected"); 
		}
		catch(TestUserServiceException e) { // TestUserService가 던져주는 예외를 잡아서 계속 진행되도록 한다. 그 외의 예외라면 테스트 실패
		}
		
		checkLevelUpgraded(users.get(1), false); // 예외가 발생하기 전에 레벨 변경이 있었던 두번째 사용자의 레벨이 처음 상태로 바뀌었나 확인
	}
	```
	결과 :  실패. 두번째 사용자의 변경된 레벨이 그대로 유지됨
- 테스트 실패의 원인
모든 사용자의 레벨을 업그레이드하는 작업인 upgradeLevels() 메소드가 하나의 트랜잭션 안에서 동작하지 않았지 때문이다.
	> 트랜잭션 : 더 이상 나눌 수 없는 단위 작업

### 5.2.2 트랜잭션 경계설정
하나의 SQL 명령을 처리하는 경우는 DB가 트랜잭션을 보장해줌
> **트랜잭션 롤백** : 두 가지 작업이 하나의 트랜잭션이 되려면, 두 번째 SQL이 성공적으로 DB에서 수행되기 전에 문제가 발생할 경우에는 앞에서 처리한 SQL 작업도 취소시켜야 한다. 이런 취소 작업

> **트랜잭션 커밋** :  여러 개의  SQL을 하나의 트랜잭션으로 처리하는 경우에 모든 SQL 수행 작접이 다 성공적으로 마무리 됐다고 DB에 알려줘서 작업을 확장 시키는 것
- JDBC 트랜잭션의 트랜잭션 경계 설정
```java
Connection c = dataSource.getConnection();

c.setAutoCommit(false); // 트랜잭션 시작
try {
    PreparedStatement st1 = c.prepareStatement("update users ...");
    st1.executeUpdate();

    PreparedStatement st2 = c.prepareStatement("delete users ...");
    st2.executeUpdate();

    c.commit(); // 트랜잭션 커밋
} catch (Exception e) {
    c.rollback(); // 트랜잭션 롤백
}

c.close();
```
JDBC의 기본 설정은 DB작업을 수행한 직후에 자동으로 커밋이 되도록 되어있다. 작업마다 커밋해서 트랜잭션을 끝내버리므로 여러 개의 DB 작업을 모아서 트랜잭션을 만드는 기능이 꺼져있음 -> JDBC에서는 이 기능을 false로 설정해주면 새로운 크랜잭션이 시작되게 만들 수 있음
> 트랜잭션의 경계설정 : setAutoCommit(false)로 트랜잭션의 시작을 선언하고 commit또는 rollback으로 트랜잭션을 종료하는 작업

> 로컬 트랜잭션 : 하나의 DB 커넥션 안에서 만들어지는 트랜잭션

- UserService와 UserDao의 트랜잭션 문제
하나의 템플릿 메소드 안에서 DataSource의 getConnection() 메소드를 호출해서 Connection 오브젝트를 가져오고, 작업을 마치면 Connection을 닫아주고 템플릿 메소드를 빠져나옴
-> JdbcTemplate의 메소드를 사용하는 UserDao는 각 메소드마다 하나의 독립적인 트랜잭션으로 실행될 수밖에 없음(커넥션 안에 트랜잭션 생성)

	💡어떤 일련의 작업이 하나의 트랜잭션으로 묶이려면 그 작업이 진행되는 동안 DB 커넥션도 하나만 사용해야함
	
- 비즈니스 로직 내의 트랜잭션 경계설정
UserDao의 update() 메소드는 반드시 upgradeLevels() 메소드에서 만든 Connection을 사용해야한다. 그래야만 같은 트랜잭션 안에서 동작하기 때문이다 
-> Dao 메소드를 호출할때마다 Connection 오브젝트를 파라미터로 전달해줘야함
`` public void add(Connection c, User user);``
UserDao의 update를 사용하는 것은 upgradeLevels()가 아닌 upgradeLevel() 메소드
-> UserService의 메소드 사이에도 같은 Connection 오브젝트를 사용하도록 파라미터로 전달해줘야 함

- UserService 트랜잭션 경계 설정의 문제점
1. DB 커넥션을 비롯한 리소스의 깔끔한 처리를 가능하게 했던 JdbcTemplate을 활용 x
2. DAO의 메소드와 비즈니스 로직을 담고 있는 UserService 의 메소드에 connection 파라미터가 추가돼야 함. + 모든 메소드에 걸쳐서 Connection 오브젝트가 계속 전달돼야함
3. Connection 파라미터가 UserDao 인터페이스 메소드에 추가되면 UserDao는 더이상 데이터 액세스 기술에 독립적일 수 없음
4. DAO 메소드에 Connection 파라미터를 받게 하면 테스트 코드에도 영향을 미침

### 5.2.3 트랜잭션 동기화
- Connection 파라미터 제거
	> 트랜잭션 동기화: UserService에서 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관해두고 이후에 호출되는 DAO의 메소드에서는 저장된 Connection을 가져다가 사용하게 함.


	- JdbcTemplate 메소드에서는 트랜잭션 동기화 저장소에 현재 시작된 트랜잭션을 가진 Connection 오브젝트가 존재하는지 확인하고 이를 가져와 사용 후 connection을 닫지 않음.
	- 모든 작업이 정상적으로 끝났다면 UserService에서 commit()을 호출하고 트랜잭션 저장소에서 제거함
	- 트랜잭션 동기화 저장소는 작업 스레드마다 독립적으로 Connection 오브젝트를 저장하고 관리

- 트랜잭션 동기화 적용
	```java
	// Connection을 생성할 때 사용할 DataSource를 DI 받음
	private DataSource dataSource;  			

	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
	
	public void upgradeLevels() throws Exception {
		TransactionSynchronizationManager.initSynchronization();  // 트랜잭션 동기화 관리자를 이용해 동기화 작업 초기화
		Connection c = DataSourceUtils.getConnection(dataSource); // DB 커넥션 생성과 동기화를 함께해주는 유틸리티 메소드
		c.setAutoCommit(false);
		
		try {									   
			List<User> users = userDao.getAll();
			for (User user : users) {
				if (canUpgradeLevel(user)) {
					upgradeLevel(user);
				}
			}
			c.commit();  // 정상적으로 작업을 마치면
		} catch (Exception e) {    // 예외가 발생하면 롤백
			c.rollback();
			throw e;
		} finally {
			DataSourceUtils.releaseConnection(c, dataSource);	
			// 스프링 유틸리티 메소드를 이용해 DB 커넥션을 안전하게 닫는다.
			TransactionSynchronizationManager.unbindResource(this.dataSource);  
			TransactionSynchronizationManager.clearSynchronization();  
		}
	}
	```
- 트랜잭션 테스트 보완
트랜잭션 동기화에 필요한 DataSource를 DI
```java
@Autowired DataSource dataSource;
...
@Test
	public void upgradeAllOrNothing() throws Exception {
		UserService testUserService = new TestUserService(users.get(3).getId());  
		testUserService.setUserDao(this.userDao); 
		testUserService.setDataSource(this.dataSource);
		...
```
UserService에 dataSource 프로퍼티 추가
- JdbcTemplate과 트랜잭션 동기화
만약 미리 생성돼서 트랜잭션 동기화 저장소에 등록된 DB 커넥션이나 트랜잭션이 없는 경우에는 JdbcTemplate이 직접 DB 커넥션을 만들고 트랜잭션을 시작해서 JDBC 작업을 진행
-> DAO를 사용할 때 트랜잭션이 굳이 필요없다면 바로 호출해서 사용해도 되고 DAO 외부에서 트랜잭션을 만들고 이를 관리할 필요가 있다면 미리 DB 커넥션을 생성한 다음 트랜잭션 동기화를 해주고 사용. Dao에서 사용하는 JdbcTemplate은 자동으로 트랜잭션에서 동작

### 5.2.4 트랜잭션 서비스 추상화

- 기술과 환경에 종속되는 트랜잭션 경계설정 코드
  - 하나의 트랜잭션 안에서 여러 개의 DB에 데이터를 넣는 작업을 해야 할 필요
  - 로컬 트랜잭션은 하나의 DB Connection에 종속되기 때문에 각 DB와 독립적으로 만들어지는 Connection을 통해서가 아니라, 별도의 트랜 잭션 관리자를 통해 트랜잭션을 관리하는 **글로벌 트랜잭션** 방식을 사용해야한다.
  - 자바는 JDBC 외에 이런 글로벌 트랜잭션을 지원하는 트랜잭션 매니저를 지원하기 위한 API인 **JTA**를 제공
  - JTA를 이용해 트랜잭션 매니저를 활용하면 여러 개의 D나 메시징 서버에 대한 작업을 하나의 트랜잭션으로 통합하는 분산 트랜잭션 또는 글로벌 트랜잭션이 가능해짐

- 트랜잭션 API의 의존관계 문제와 해결책
  - Userservice에서 트랜잭션의 경제 설정을 해야 할 필요가 생기면서 다시 특정 데이터 액세스 기술에 종속되는 구조가 됨
  - 문제는 JDBC에 종속적인 Connection을 이용한 트랜잭션 코드가 Userservice에 등장하면서부터 userservice는 UserDaoJdbc에 간접적으로 의존하는 코드가 돼버렸다는 점
  - JDBC, JTA, 하이버네이트, JPA, JDO, JMS 모두 트랜잭션 개념을 갖고 있으니 공통적인 특징을 모아서 추상화된 트랜잭션 관리 계층을 만들 수 있다. 그리고 애플리케이션 코드에서는 트랜잭션 추상 계층이 제공하는 API를 이용해 트랜잭션을 이용하게 만들어준다면 특정 기술에 종속되지 않는 트랜잭션 경계설정 코드를 만들 수 있을 것이다.

- 스프링의 트랜잭션 서비스 추상화
스프링은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공하고 있다. 이를 이용하면 애플리케이션에서 직접 각 기술의 트랜잭션 API를 이용하지 않고도, 일관된 방식으로 트랜잭션을 제어하는 트랜잭션 경계 설정 작업이 가능해진다.
```java
public void upgradeLevels() {
		PlatformTransactionManager = transactionManager = 
			new DataSourceTrsnsactionManager(dataSource); // Jdbc 트랜잭션 추상 오브젝트 생성
		// 트랜잭션 시작
		TransactionStatus status = 
			this.transactionManager.getTransaction(new DefaultTransactionDefinition()); // 트랜잭션 매니저가 DB 커넥션을 가져오는 작업도 같이 수행
		try {	// 트랜잭션 안에서 진행되는 작업
			List<User> users = userDao.getAll();
			for (User user : users) {
				if (canUpgradeLevel(user)) {
					upgradeLevel(user);
				}
			}
			this.transactionManager.commit(status); // 트랜잭션 커밋
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status); // 트랜잭션 롤백
			throw e;
		}
	}
```
PlatformTransactionManager로 시작한 트랜잭션은 트랜잭션 동기화 저장소에 저장됨. 
DataSourceTransactionManager 오브젝트는 PlatformTransactionManager를 통해 시작한 트랜잭션을 USerDao의 JdbcTemplate 안에서 사용되게 해줌

트랜잭션 작업을 모두 수행한 후에는 트랜잭션을 만들 대 돌려받은 TransactionStatus 오브젝트를 파라미터로 해서 PlatformTransactionManager의 commit() 메소드를 호출하거나 rollback() 메소드를 부른다.
- 트랜잭션 기술 설정의 분리
JTA를 이용하는 글로벌 트랜잭션으로 변경하려면 PlatformTransactionManager 구현 클래스를 DataSourceTransactionManage에서 JTATransactionManager로 바꿔주기만 하면 됨.
-> 이는 DI의 원칙에 위배됨
-> DataSourceTransactionManager는 스프링 빈으로 등록하고 UserService가 DI 방식으로 사용하게 해야함.
UserService에는 PlatfromTransactionManager(싱글톤) 인터페이스 타입의 인스턴스 변수를 선언하고, 수정자 메소드를 추가해서 DI가 가능하게 해줌
```java
public class UserService {
	private PlatformTransactionManager transactionManager;

	// 프로퍼티 이름은 관례를 따라 transactionManager라고 만드는 것이 편리
	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void upgradeLevels() {
		TransactionStatus status =  // DI 받은 트랜잭션 매니저를 공유해서 사용한다. 멀티스레드 환경에서도 안전하다.
			this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			List<User> users = userDao.getAll();
			for (User user : users) {
				if (canUpgradeLevel(user)) {
					upgradeLevel(user);
				}
			}
			this.transactionManager.commit(status);
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
```
DataSourceTransactionManager는 dataSource 빈으로부터 Connection을 가져와 트랜잭션 처리를 해야 하기 때문에 dataSource 프로퍼티를 갖는다.
```xml
<bean id="userService" class="springbook.user.service.UserService">
	<property name="userDao" ref="userDao" />
	<property name="transactionManager" ref="transactionManager" />
</bean>

트랜잭션을 JTA를 이용하는 것으로 고치고 싶다면 밑의 빈의 class만 고치면 된다.
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource" />  
</bean>
```
테스트에서 스프링 컨테이너로부터 transactionManager 빈을 @AutoWired로 주입받게 하고 이를 직접 DI해준다.
```java
public class UserServiceTest {
	@Autowired PlatformTransactionManager transactionManager; 
	
	@Test
	public void upgradeAllOrNothing() {
		UserService testUserService = new TestUserService(users.get(3).getId());  
		testUserService.setUserDao(this.userDao);
		testUserService.setTransactionManager(this.transactionManager);
```

## 5.3 서비스 추상화와 단일 책임 원칙
- 수직, 수평 계층구조와 의존관계
	- UserDao와 UserService는 같은 애플리케이션 로직을 담은 코드지만 내용에 따라 분리했으므로 같은 계층에서 수평적인 분리다.
	- 트랜잭션 추상화는 애플리케이션의 비즈니스 로직과 그 하위에서 동작하는 로우레벨의 트랜잭션 기술이라는 아예 다른 계층의 특성을 갖는 코드를 분리한, 수직적인 분리이다.
	- UserDao와 DB 연결 기술, userService와 트랜잭션 기술의 결합도가 낮은 분리는 애플 리케이션 코드를 로우레벨의 기술 서비스와 환경에서 독립시켜준다
	- 애플리케이션 로직의 종류에 따른 수평적인 구분이든, 로직과 기술이라는 수직적인 구분이든 모두 결합도가 낮으며, 서로 영향을 주지 않고 자유롭게 확장될 수 있는 구조 를 만들 수 있는 데는 스프링의 DI가 중요한 역할을 하고 있다. 
- 단일 책임 원칙
	>단일 책임 원칙: 하나의 모듈은 한 가지 책임을 가져야 한다.
	
- 단일 책임 원칙의 장점
어떤 변경이 필요할 때 수정 대상이 명확해짐
	 - 스프링의 DI가 없었다면 인터페이스를 도입해서 추상화를 했더라도 적지 않은 코드 사이의 결합이 남아 있게 됨
	 - 객체지향 설계와 프로그래밍의 원칙은 서로 긴밀하게 관련이 있다. 단일 책임 원칙을 잘 지키는 코드를 만들려면 인터페이스를 도입하고 이를 DI로 연결해야 하며, 그 결과로 단일 책임 원칙뿐 아니라 개방 폐쇄 원칙도 잘 지키고, 모듈 간에 결합도가 낮아서 서로의 변경이 영향을 주지 않고, 같은 이유로 변경이 단일 책임에 집중되는 응집도 높은 코드가 나옴 + 테스트 하기도 편함

## 5.4 메일 서비스 추상화
### 5.4.1 JavaMail을 이용한 메일 발송 기능
User 테이블에 email 필드 추가하고 다른 메소드들과 테스트에도 추가
- JavaMail 메일 발송
자바의 이메일 발송 클래스를 이용하여 메소드 호출
```java
protected void upgradeLevel(User user) {
		user.upgradeLevel();
		userDao.update(user);
		sendUpgradeEMail(user); // 직접 JavaMail 코드를 넣지 않고 메소드 호출
	}

// 메일 보내는 메소드
private void sendUpgradeEMail(User user) {
    Properties props = new Properties();
    props.put("mail.smtp.host", "mail.ksug.org");
    Session s = Session.getInstance(props, null);

    MimeMessage message = new MimeMessage(s);
    try {
        message.setFrom(new InternetAddress("useradmin@ksug.org"));
        message.addRecipient(Message.RecipientType.TO,
            new InternetAddress(user.getEmail()));

        message.setSubject("Upgrade 안내");
        message.setText("사용자님의 등급이 " + user.getLevel().name() +
            "로 업그레이드되었습니다.");

        Transport.send(message);
    } catch (AddressException e) {
        throw new RuntimeException(e);
    } catch (MessagingException e) {
        throw new RuntimeException(e);
    } catch (UnsupportedEncodingException e) {
        throw new RuntimeException(e);
    }
}
```

### 5.4.2 JavaMail이 포함된 코드의 테스트
테스트에서 upgradeLevel() 메소드를 호출해서 sendUpgradeMail()이 호출되면 메일이 실제로 발송된다
-> 이는 부하가 매우 큰 작업이므로 테스트에서 메일을 보낸다면 메일 서버에 상당한 부담을 줌
-> SMTP는 충분히 테스트된 기술. 테스트용으로 따로 메일 서버를 만들어 외부로 메일을 발송하지 않고, JavaMail과 연동해서 메일 전송을 요청을 받는 것까지만 확인
-> JavaMail도 충분히 테스트된 기술. JavaMail API를 통해 요청이 들어간 것만 확인해서 부하를 줄임

### 5.4.3 테스트를 위한 서비스 추상화
- JavaMail을 이용한 테스트의 문제점
JavaMail의 핵심 API에는 DataSource처럼 인터페이스로 만들어져 구현을 바꿀 수 있는게 없음
	- JavaMail에서는 Session 오브젝트를 만들어야만 메일 메시지를 생성할 수 있음
	- Session은 인터페이스가 아닌  상속이 불가능한 final 클래스
	- 생성자가 private라 직접 생성이 불가능
- 메일 발송 기능 추상화
	- JavaMail의 추상화 인터페이스
```java
public interface MailSender {
	void send(SimpleMailMessage simpleMessage) throws MailException;
	void send(SimpleMailMessage[] simpleMessage) throws MailException;
}
```
JavaMail을 시용해 메일 발송 기능을 제공하는 JavaMailSenderImpl 클래스를 이용하면 됨.
```java
private void sendUpgradeEMail(User user) {
		JavaMailSenderImpl mailSender= new JavaMailSenderImpl ();
		mailSender.setHost("mail.server.com");
		
		// MailMessage 인터페이스의 구현 클래스 오브젝트를 만들어 내용을 작성한다.
		SimpleMailMessage mailMessage = new SimpleMailMessage();
		mailMessage.setTo(user.getEmail());
		mailMessage.setFrom("useradmin@ksug.org");
		mailMessage.setSubject("Upgrade 안내");
		mailMessage.setText("사용자님의 등급이" + user.getLevel().name());
		
		mailSender.send(mailMessage);
	}
DI 적용 - MailSender 인터페이스만 남기고, 구체적인 메일 전송 구현을 담은 클래스의 정보는 코드에서 모두 제거
```java
private MailSender mailSender;

public void setMailSender(MailSender mailSender) {
		this.mailSender = mailSender;	// DI
	}
	
private void sendUpgradeEMail(User user) {
		SimpleMailMessage mailMessage = new SimpleMailMessage();
		mailMessage.setTo(user.getEmail());
		mailMessage.setFrom("useradmin@ksug.org");
		mailMessage.setSubject("Upgrade 안내");
		mailMessage.setText("사용자님의 등급이" + user.getLevel().name());
		
		this.mailSender.send(mailMessage);
	}
```
스프링의 빈으로 등록되는 MailSender의 구현 틀래스들은 싱글톤으로 사용 가능
```xml
<bean id="userService" class="springbook.user.service.UserService">
		<property name="userDao" ref="userDao" />
		<property name="transactionManager" ref="transactionManager" />
		<property name="mailSender" ref="mailSender" />
	</bean>

	<bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">
		<property name="host" value="mail.server.com" />
	</bean>
```
- 테스트용 메일 발송 오브젝트
테스트용 메일 전송 클래스 만들기
```java
public class DummyMailSender implements MailSender {
	public void send(SimpleMailMessage mailMessage) throws MailException {
	}

	public void send(SimpleMailMessage[] mailMessage) throws MailException {
	}
}
```
테스트 설정파일의 mailSender 빈 클래스 변경
``<bean id="mailSender" class="springbook.user.service.DummyMailSender" />``
수동 DI
```java
public class UserServiceTest {
	...
	@Autowired MailSender mailSender; 
	
	@Test
	public void upgradeAllOrNothing() {
		...
		testUserService.setMailSender(this.mailSender);
```

- 테스트와 서비스 추상화
테스트를 어렵게 만드는 건전하지 않은 방식으로 설계된 API를 사용할때도 유용하게 쓰임
JavaMail이 아닌 다른 메세징 서버의 API를 이용해 메일을 전송해야 하는 경우가 생겨도, 해당 기술의 API를 이용하는 MainSender 구현 클래스를 만들어서 DI해주면 됨
	- 메일 발송 작업에도 트랜잭션을 적용해야함
	1. 메일 발송 대상을 별도의 목록에 저장해둔 뒤 업그레이드 작업이 모두 성공적으로 끝났을 때 한 번에 메일을 전송
	2. MailSender를 확장해서 트랜잭션 개념 적용


### 5.4.4 테스트 대역
테스트 환경에서 유용하게 사용하는 기법 (대부분 테스트할 대상이 의존하고 있는 오브젝트를 DI를 통해 바꿔치기)
- 의존 오브젝트의 변경을 통한 테스트 방법
관심사에 따른 테스트를 하고 싶은거지, DB나 메일서버 같은게 잘 동작하는지는 관심이 없기 때문에 테스트 용 DB와 메일 클래스를 따로 만듦
- 테스트 대역의 종류와 특징
	> 테스트 대역 : 테스트 환경을 만들어주기 위해, 테스트 대상이 되는 오브젝트의 기능에만 충실하게 수행하면서 빠르게, 자주 테스트를 실행할 수 있도록 사용하는 오브젝트
	
	 - 테스트 스텁 : 테스트 대상 오브젝트의 의존객체로서 존재하면서 테스트 동안에 코드가 정상적으로 수행할 수 있도록 돕는 것 (ex. DummyMailSender) 
		 - 리턴 값이나 예외를 발생시키게 할 수도 있음
		 - 간접적인 입력 값을 지정하거나 출력 값을 받게할 수 있음
	 -  테스트 오브젝트가 간접적으로 의존 오브젝트에 넘기는 값과 그 행위 자체에 대해서도 검증하고 싶다면 목 오브젝트를 사용해야 한다. 
	 > 목 오브젝트는 스텁처럼 테스트 오브젝트가 정상적으로 실행되도록 도와주면서, 테스트 오브젝트와 자신의 사이에서 일어나는 커뮤니케이션 내용을 저장해뒀다가 테스트 결과를 검증하는 데 활용할 수 있게 해준다.
	 
	 - 테스트 대상은 의존 오브젝트에게 값을 입력받음 
	 -> 별도로 준비해둔 스텝 오브젝트가 메소드 호 출 시 특정 값을 리턴하도록 만들어두면 된다.
     - 테스트 대상 오브젝트가 의존 오브젝트에게 출력한 값에 관심이 있을 경우가 있다. 또는 의존 오브젝트를 얼마나 사용했는가 하는 커뮤니케이션 행위 자체에 관심이 있을 수가 있다. 이때는 테스트 대상과 의존 오브젝트 사이에 주고받는 정보를 보존해두는 기능을 가진 테스트용 의존 오브젝트인 목 오브젝트를 만들어서 사용해야 한다.


- 목 오브젝트를 이용한 테스트
목 오브젝트를 만들어서 메일 발송 여부를 확인
```java 
static class MockMailSender implements MailSender {
		// UserService로부터 전송 요청을 받은 메일 주소를 저장해두고 이를 읽을 수 있게 함
		private List<String> requests = new ArrayList<String>();	
		
		public List<String> getRequests() {
			return requests;
		}

		public void send(SimpleMailMessage mailMessage) throws MailException {
			requests.add(mailMessage.getTo()[0]); // 전송 요청을 받은 이메일 주소를 저장해둔다.(첫번째 수신자만)
		}

		public void send(SimpleMailMessage[] mailMessage) throws MailException {
		}
	}
```
테스트 대상 오브젝트가 목 오브젝트에게 전달하는 출력정보를 저장해두는 것
```java
@Test 
@DirtiesContext // 컨텍스트의 DI 설정을 변경하는 테스트라는 것을 알려준다.
	public void upgradeLevels() {
		userDao.deleteAll();
		for(User user : users) userDao.add(user);
		// 메일 발송 결과를 테스트할 수 있도록 목 오브젝트를 만들어 
		// userService의 의존 오브젝트로 주입해준다.
		MockMailSender mockMailSender = new MockMailSender();  
		userService.setMailSender(mockMailSender);  
		
		// 업그레이드 테스트, 메일 발송이 일어나면 
		// MockMailSender 오브젝트의 리스트에 그 결과가 저장된다
		userService.upgradeLevels();
		
		checkLevelUpgraded(users.get(0), false);
		checkLevelUpgraded(users.get(1), true);
		checkLevelUpgraded(users.get(2), false);
		checkLevelUpgraded(users.get(3), true);
		checkLevelUpgraded(users.get(4), false);
		
		// 목 오브젝트에 저장된 메일 수신자 목록을 가져와 업그레이드 대상과 일치하는지 확인
		List<String> request = mockMailSender.getRequests();  
		assertThat(request.size(), is(2));  
		assertThat(request.get(0), is(users.get(1).getEmail()));  
		assertThat(request.get(1), is(users.get(3).getEmail()));  
	}
```

목 오브젝트는 보통의 테스트 방법으로는 검증하기가 매우 까다로운 테스트 대상 오브젝트의 내부에서 일어나는 일이나 다른 오브젝트 사이에서 주고받는 정보까지 검증하는 일이 손쉽다.

</br></br>
🌿기억에 남는 내용 :  트랜잭션을 @Transactional 어노테이션만을 이용해 사용해보았는데, 직접 코드를 작성해 트랜잭션을 관리하는 방식으로 트랜잭션의 흐름(시작, 커밋, 롤백)이 어떻게 이루어지는지 확인해서 트랜잭션의 역할과 필요성을 더 잘 이해할 수 있었다. 왜쓰는지 정확히 모르는 채로 당연히 사용하고 있던 기능인데 원리를 자세히 알게되어 좋았다.
