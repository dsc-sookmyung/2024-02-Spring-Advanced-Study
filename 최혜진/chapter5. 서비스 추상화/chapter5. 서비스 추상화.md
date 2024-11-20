> `학습 목표`
> DAO 에 트랜잭션을 적용해보면서 스프링이 어떻게 성격이 비슷한 여러 종류의 기술을 추상화 하고 이를 일관된 방법으로 사용할 수 있도록 지원하는지 살펴볼 것

# 5.1 사용자 레벨 관리 기능 추가
> 사용자 정보를 DB에 넣고 빼는 것 외의 비지니스 로직을 갖고 있지 않다.
> 간단한 비지니스 로직을 구현해보자.

## 5.1.1 필드 추가
### level 이늄(enum)
> 레벨을 저장할 필드를 추가

BASIC, SILVER, GOLD 를 int 타입으로 설명하면 오류가 발생할 가능성이 있다.
- 컴파일러가 다른 종류의 정보를 넣었을 때, 이를 체크해주지 못 한다.
- 우연히 레벨에 대응하는 숫자를 넣거나 해당 값이 범위를 초과하면 버그가 발생한다.
  그래서 숫자 타입을 직접 사용하는 것보다 자바 5 이상에서 제공하는 `enum` 을 이용하는 게 안전하고 편리하다.

```java
package springbook.user.domain;
...
public enum Level {
	BASIC(1), SILVER(2), GOLD(3); // enum 오브젝트 정의
    
    private final int value;
    
    Level(int value) { // 생성자 만들기
    	this.value = value;
    }
    
    public int intValue() { // 값을 가져오는 메소드
    	return value;
    }
    
    //값으로부터 Level 타입 오브젝트를 가져오도록 만든 스태틱 메소드
    public static Level valueOf(int value){ 
    	switch(value) {
        	case 1: return BASIC;
            case 2: return SILVER;
            case 3: return GOLD;
            default: throw new AssertionError("Unknown value: " + value);
        }
    }
}
```

이렇게 만들어진 level enum 은 내부에는 DB에 저장할 int 타입의 값을 갖고 있으나 겉으로는 level 타입의 오브젝트이기 때문에 안전하게 사용할 수 있다.
### User 필드 추가
> level 타입의 변수를 User 클래스에 추가

```java
public class User {
	...
    Level level;
    int login; //로그인 횟수
    int recommend; // 추천 수
    
    public Level getLevel() {
    	return level;
    }
    
    public void setLevel(Level level) {
    	this.level = level;
    }
    //login recommend 의 getter/setter 생략
}
```
DB의 USER 테이블에도 필드를 추가

| 필드명       | 타입      | 설정       |
| --------- | ------- | -------- |
| level     | tinyint | not null |
| login     | int     | not null |
| recommend | int     | not null |
### UserDaoTest 테스트 수정
> 테스트가 성공할 수 있도록 UserDaoJdbc 수정

```java
public class UserDaoJdbc implements UserDao {
	...
    private RowMapper<User> userMapper =
    	new RowMapper<User>() {
        	public User mapRow(ResultSet rs, int rowNum) throws SQLException {
            	User user = new user();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));
                user.setLevel(Level.valueOf(rs.getInt("level")));
                user.setLogin(rs.getInt("loign"));
                user.setRecommend(rs.getInt("recommend"));
                return user;
            }
        };
        
    public void add(User user){
    	this.jdbcTemplate.update(
        "insert into users(id, name, password, level, login, recommend)" +
        "values(?,?,?,?,?,?)", user.getId(), user.getName(),
        user.getPassword(), user.getLevel().intValue(),
        user.getLogin(), user.getRecommend());
    }
}
```

- 테스트를 돌렸을 때, 실패한다.
    - BadSqlGrammerException (sql 문법이 틀렸음) 의 메시지를 확인할 수 있다.
    - login 을 loign 으로 잘못 썼다.
      미리 DB까지 연동되는 테스트를 잘 만들어뒀기 때문에 SQL 문장에 사용될 필드 이름의 오타를 빠르게 잡아낼 수 있었다.

>만약, 테스트가 없었다면?
>> 빌드와 서버 배치, 서버 재시작, 수동 테스트 등에 시간을 낭비하고, 엄청난 에러를 직면하게 될 것

## 5.1.2 사용자 수정 기능 추가
> 수정할 정보가 담긴 User 오브젝트를 전달하면 id 를 참고해서 사용자를 찾아 필드 정보를 UPDATE 문을 이용해 모두 변경해주는 메소드를 하나 만들어보자.

### 수정 기능 테스트 추가
`사용자 정보 수정 메소드 테스트`
```java
@Test
public void update() {
	dao.deleteAll();
    
    dao.add(user1);
    
    //픽스쳐에 들어있는 정보를 변경하여 수정 메소드를 호출
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
### UserDao 와 UserDaoJdbc 수정
`update() 메소드 추가`
```java
//update()메소드가 없으니 인터페이스 추가
public interface UserDao {
	...
    public void update(User user1);
}
```
`사용자 정보 수정용 update() 메소드`
```java
// 인터페이스로 만든 update() 구현
public void update(User user){
	this.jdbcTemplate.update(
    		"update users set name = ?, password = ?, login = ?, " +
            "recommend = ? where id = ? ", user.getName(), user.getPassword(), 
            user.getLevel().intValue(), user.getLogin(), user.getRecommend(), 
            user.getId());
}
```

> IDE 의 자동 수정 기능과 테스트 코드 작성
>> 대부분의 자바 IDE 에는 컴파일 에러에 대한 자동 수정 기능이 있다.
>> `Ctrl + 1` (이클립스) , `Alt + Enter` (인텔리제이) 키를 눌러서 IDE 가 제안하는 자동 수정 내용 중 하나를 선택할 수 있다.

### 수정 테스트 보완
> UPDATE 문장에서 WHERE 절이 없어도 아무런 경고 없이 정상 작동하는 것처럼 보인다.
>
> 하지만 update()  테스트는 수정할 로우의 내용이 바꾼 것만 확인할 뿐이지, 수정하지 않아야 할 로우의 내용이 그대로 남아 있는지 확인해주지 못 한다.

`해결책`
1. JdbcTemplate 의 update()가 돌려주는 리턴 값 확인
2. 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 직접 확인
    - 사용자를 두 명 등록해두고 하나만 수정한 뒤 수정된 사용자와 수정되지 않은 사용자의 정보를 모두 확인하는 방법
```java
@Test
public void update() {
	dao.deleteAll();
    
    dao.add(user1); // 수정할 사용자
    dao.add(user2); // 수정하지 않을 사용자
    
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
## 5.1.3 UserService.upgradeLevels()
> 사용자 관리 비지니스 로직을 담을 UserService 클래스를 하나 생성하자.

![[Pasted image 20241105204607.png]]
UserDao 의 구현클래스가 바뀌어도 영향 받지 않도록 하기 위해 DAO의 인터페이스 타입으로 하는 인스턴스 변수를 사용하고 DI를 적용한다.
`UserSevice 클래스`
```java
package springbook.user.service;
...
public class UserService {
	UserDao userDao; //인터페이스를 타입으로 하는 변수
    
    public void setUserDao(UserDao userDao) { // 수정자 메소드로 DI가 가능하도록 함
    	this.userDao = userDao;
    }
}
```

### UserServiceTest 테스트 클래스
> UserService 를 테스트 할 UserServiceTest 클래스를 만들어보자.

```java
package springbook.user.service;
...
@RunWith(SpringJunit4ClassRunner.class)
@ContextConfiguration(location="/test-applicationContext.xml")
public class UserServiceTest {
	@Autowired
    UserService userservice;
}
```

```java
@Test
public void bean(){
	assertThat(this.userService, is(notNullValue()));
}
```
이렇게 테스트 메소드도 추가해서 JUnit으로 테스트를 수행해보자.

### upgradeLevels() 메소드
> 사용자 레벨을 관리하는 기능을 가진 메소드 만들기
>> 로직을 먼저 구현해보고 테스트 작성

`사용자 레벨 업그레이드 메소드`
```java
public void upgradeLevels() {
	List<User> users = userDao.getAll();
    for(User user : users) {
    	Boolean changed = null; //레벨 변화가 있는지 확인하는 플래그
        
        // 로그인 횟수가 50이상이면 SILVER로 승급
        if(user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
        	user.setLevel(Level.SILVER);
            changed = true;
        }
        // SILVER 등급에서 추천수가 30개 이상이면 GOLD로 승급
        else if(user.getLevel() == Level.SILVER && user.getRecommend() >= 30) {
        	user.setLevel(Level.GOLD);
            changed = true; //레벨 변경 플래그 설정
        }
        // GOLD 윗단계는 없음
        else if (user.getLevel() == Level.GOLD) { changed = false; }
        else { changed = false; }
        
        // 레벨의 변경이 있으면 update()호출
        if(changed) { userDao.update(user); }
    }
}
```
### upgradeLevels() 테스트

> 사용자 레벨은 Basic, Silver, Gold 세가지이고 , 변경이 일어나지 않는 Gold 를 제외한 나머지 두 가지는 업그레이드가 되는 경우와 아닌 경우가 있을 수 있으므로 최소한 다섯 가지 경우를 살펴봐야 한다.

```java
class UserServiceTest {
	...
    List<User> users;
    
    @Before
    public void setUp() {
    // 배열을 리스트로 만들어주는 편리한 메소드, 배열 가변인자로 넣어주면 더욱 편리
    	users = Arrays.asList(
        		new User("bumjin", "박범진", "p1", Level.BASIC, 49, 0),
                new User("joytouch", "강명성", "p2", Level.BASIC, 50, 0),
                new User("erwins", "신승한", "p3", Level.SILVER, 60, 29),
                new User("madnite1", "이상호", "p4", Level.SILVER, 60, 30),
                new User("green", "오민규", "p5", Level.GOLD, 100, 100)
        );
    }
    
    
    @Test
    public void upgradeLevels() {
    	userDao.deleteAll();
        for(User user : users) userDao.add(user);
        
        userService.upgradeLevels();
        
        // 사용자의 예상 레벨을 검증 - 모두 맞아야 테스트 통과
        checkLevel(users.get(0), Level.BASIC);
        checkLevel(users.get(1), Level.SILVER);
        checkLevel(users.get(2), Level.SILVER);
        checkLevel(users.get(3), Level.GOLD);
        checkLevel(users.get(4), Level.GOLD);
    }
	// db에서 사용자 정보를 가져와 레벨을 확인하는 코드가 중복되므로 헬퍼 메소드로 분리
    private void checkLevel(User user, Level expectedLevel) {
    	User userUpdate = userDao.get(user.getId());
        assertThat(userUpdate.getLevel(), is(expectedLevel));
    }
}
```

## 5.1.4 UserService.add()
> UserService에서 add() 를 만들어서 처음 가입한 사용자는 생성자 레벨을 Basic으로 설정하도록 한다.

- 테스트를 먼저 만들어보자.
- `두 가지의 테스트 코드`
    - 레벨이 비어있으면, 로직을 따라서 Basic 을 부여해주자.
    - 특별한 이유가 있어서 미리 설정된 레벨을 가진 User 오브젝트인 경우 그대로 두자.
      `User 오브젝트 레벨이 변경됐는지 확인하는 두가지 방법`
1. UserService 의 add() 메소드를 호출할 때 파라미터로 넘긴 User 오브젝트에 level 필드를 확인해보는 것
2. UserDao 의 get() 메소드를 이용해 DB에 저장된 User 정보를 가져와 확인하는 것
   우리가 만들 건 2번째 방법
```java
@Test
public void add() {
	userDao.deleteAll();
	//Gold 레벨이 이미 지정된 User라면 레벨을 초기화하지 않아야 한다.
    User userWithLevel = users.get(4); // GOLD 레벨
    
    // 레벨이 비어있는 사용자
    User userWithoutLevel = users.get(0); 
    userWithoutLevel.setLevel(null);
    
    userService.add(userWithLevel); //레벨이 정해져 있으면 그대로 두기
    userService.add(userWithoutLevel); //비어있으면 BASIC
    
    //DB에 저장된 결과를 가져와 확인
    User userWithLevelRead = userDao.get(userWithLevel.getId());
    User userWithoutLevelRead = userDao.get(userWithoutLevel.getId());
    
    //그대로 두기
    assertThat(userWithLevelRead.getLevel(), is(userWithLevel.getLevel()));
    //BASIC
    assertThat(userWithoutLevelRead.getLevel(), is(Level.BASIC)); 
}
```
테스트 성공하도록 add() 메소드를 작성하자.
`사용자 신규 등록 로직을 담은 add() 메소드`
```java
public void add(User user) {
	if(user.getLevel() == null) user.setLevel(Level.BASIC);
    userDao.add(user);
}
```

## 5.1.5 코드 개선

- 작성된 코드를 다음의 질문을 하며 살펴보자.
    1. 코드에 중복된 부분은 없는가?
    2. 코드가 무엇을 하는 것인지 이해하기 불편하지 않은가?
    3. 코드가 자신이 있어야 할 자리에 있는가?
    4. 앞으로 변경이 일어난다면 어떤 것이 있을 수 있고, 그 변화에 쉽게 대응할 수 있게 작성되었는가?
### upgradeLevels() 메소드 코드 문제점
> if/elseif/else 블록들이 읽기 불편하다.
```java
/ 현재 레벨을 파악하는 로직       // 업그레이드 조건을 담은 로직
if(user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
 			// 다음 단계의 레벨이 무엇이며, 어떤 작업을 하는가
        	user.setLevel(Level.SILVER);
            // 그 자체로는 의미가 없음.
            changed = true;
}
...
// 이 작업을 위해 changed를 사용
if(changed) { userDao.update(user); }
```
### upgradeLevels() 리펙토링
기존의 upgradeLevels() 메소드는 자주 변경될 가능성이 있는 구체적인 내용이 추상적 로직의 흐름에 섞여 있다.
`기본 작업 흐름만 남겨둔 upgradeLevels()`
```java
public void upgradeLevels() {
	List<User> users = userDao.getAll();
    for(User user : users) {
    	if(canUpgradeLevel(user)) {
        	upgradeLevel(user);
        }
    }
}
```
- 모든 사용자 정보를 가져와 한 명씩 업그레이드가 가능한지 살펴보고, 가능하면 업그레이드를 한다.

이제 위의 코드의 메소드들을 하나씩 추가하면 된다.
먼저, 업그레이드 가능 여부를 확인하는 canUpgradeLevel() 을 만들어보자.
```java
private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel();
    // 레벨별로 조건 판단
    switch(currentLevel) {
    	case BASIC: return (user.getLogin() >= 50);
        case SILVER: return (user.getRecommend() >= 30);
        case GOLD: return false;
        // 다룰 수 없는 레벨이 주어지면 예외 발생.
        default: throw new IllegalArgumentException("Unknown Level: " +
        	currentLevel);
    }
}
```
다음으로 업그레이드를 수행하는 upgradeLevel()메소드를 만들어보자.
```java
private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel();
    // 레벨별로 조건 판단
    switch(currentLevel) {
    	case BASIC: return (user.getLogin() >= 50);
        case SILVER: return (user.getRecommend() >= 30);
        case GOLD: return false;
        // 다룰 수 없는 레벨이 주어지면 예외 발생.
        default: throw new IllegalArgumentException("Unknown Level: " +
        	currentLevel);
    }
}
```
> upgradeLevel(User user)의 문제점은?
>> 예외 상황에 대한 처리가 없다.
>> 다음 단계가 무엇인지에 대한 로직과 사용자 오브젝트의 level 필드를 변경해준다는 로직이 함께 있어서 단일 책임 원칙을 위배한다.

위의 문제를 해결하기 위한 레벨의 순서와 다음 단계 레벨이 무엇인지 결정하는 일을 level 에게 맡기자.
```java
package springbook.user.domain;
...
public enum Level {
	BASIC(1, SILVER), SILVER(2, GOLD), GOLD(3, null); // 이늄 선언에 db에 저장할 값과 함께 다음 단계 레벨도 정의
    
    private final int value;
    private final Level next; //다음 단계 레벨정보를 스스로 가지고 있도록 함
    
    Level(int value, Level next) { // 생성자 만들기
    	this.value = value;
        this.next = next;
    }
    
    public int intValue() { // 값을 가져오는 메소드
    	return value;
    }
    
    public Level nextLevel() {
    	return this.next;
    }
    
    //값으로부터 Level 타입 오브젝트를 가져오도록 만든 스태틱 메소드
    public static Level valueOf(int value){ 
    	switch(value) {
        	case 1: return BASIC;
            case 2: return SILVER;
            case 3: return GOLD;
            default: throw new AssertionError("Unknown value: " + value);
        }
    }
}
```
upgradeLevel(User user) 수정을 위해 upgradeLevel() 생성
```java
public void upgradeLevel() {
	Level nextLevel = this.level.nextLevel();
    if(nextLevel == null)
    	throw nex IllegalStateException(this.level + "은 업그레이드가 불가능합니다.");
    else
    	this.level = nextLevel;
}
```
간결해진 최종 upgradeLevel(User user) 작성
```java
private void upgradeLevel(User user) {
	user.upgradeLevel();
    userDao.update(user);
}
```
### User 테스트
> User에 추가한 upgradeLevel() 메소드에 대한 테스트

```java
package springbook.user.service;
...
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
        	if(level.nextLevel() == null) continue;
            user.setLevel(level);
            user.upgradeLevel();
            assertThat(user.getLevel(), is(level.nextLevel()));
        }
    }
    
    @Test(expected=IllegalStateException.class)
    public void cannotUpgradeLevel() {
    	Level[] levels = Level.values();
        for(Level level : levels) {
        	if(level.nextLevel() != null) continue;
            user.setLevel(level);
            user.upgradeLevel();
        }
    }
}
```

### UserServiceTest 개선
> 다음 레벨을 저장해두고 있으므로 다음 레벨을 굳이 전달할 필요가 없다. (중복)
> 업그레이드 여부만 볼 수 있게 수정
```java
@Test
public void upgradeLevels() {
  	userDao.deleteAll();
	for(User user : users) userDao.add(user);
        
	userService.upgradeLevels();
        
    // 사용자의 예상 레벨을 검증 - 모두 맞아야 테스트 통과
	//어떤 레벨로 바뀌냐 -> 다음 레벨로 업그레이드가 되는가? 를 지정
	checkLevel(users.get(0), false);
    checkLevel(users.get(1), true);
    checkLevel(users.get(2), false);
    checkLevel(users.get(3), true);
    checkLevel(users.get(4), false);
}

private void checkLevelUpgraded(User user, boolean upgraded) {
    User userUpdate = userDao.get(user.getId());
    if(upgraded) //업그레이드 한다면
    // 업그레이드 일어났는지 확인
    	assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel())); 
	else //업그레이드 하지 않는다면
	//업그레이드 일어나지 않았는지 확인
    	assertThat(userUpdate.getLevel(), is(user.getLevel()));
}
```

`코드의 중복 제거`
- case BASIC: return (user.getLogin() >= **50**); - UserService
- new User("joytouch", "강명성", "p2", Level.BASIC, **50**, 0) - UserServiceTest  
  => **업그레이드 조건인 50이 중복**되고 있는 상태. 만일 **조건을 바꾼다면 두 곳에서 수정이 필요할 것**

`상수의 도입`
```java
// UserService 클래스
// 정수형 상수로 업그레이드 조건을 선언
public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
public static final MIN_RECCOMEND_FOR_GOLD = 30;

private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel();
    // 레벨별로 조건 판단
    switch(currentLevel) {
    	// 50 -> MIN_LOGCOUNT_FOR_SILVER로 변경
    	case BASIC: return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER);
        // 30 -> MIN_RECCOMEND_FOR_GOLD로 변경
        case SILVER: return (user.getRecommend() >= MIN_RECCOMEND_FOR_GOLD);
        case GOLD: return false;
        // 다룰 수 없는 레벨이 주어지면 예외 발생.
        default: throw new IllegalArgumentException("Unknown Level: " +
        	currentLevel);
    }
}
```
`상수를 사용하도록 만든 테스트`
```java
// UserServiceTest 클래스
// UserService에서 선언한 상수 받아오기
import static soringbook.user.service.UserService.MIN_LOGCOUNT_FOR_SILVER;
import static soringbook.user.service.UserService.MIN_RECCOMEND_FOR_GOLD;
...
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

업그레이드 정책을 담은 인터페이스를 만들어두고  UserService는 DI 로 제공받은 정책 구현 클래스를 이 인터페이스를 통해 사용할 수 있다.
`업그레이드 정책 인터페이스`
```java
public interface UserLevelUpgradePolicy{
	boolean canUpgradeLevel(User user);
	void upgradeLevel(User user);
}
```
# 5.2 트랜잭션 서비스 추상화
> 만약 정기 사용자 레벨 관리 작업을 수행하는 도중 에러가 발생했다면, 그 전까지 변경된 사용자 레벨은 어떻게 해야 할까요?
>> ?: 변경된 모든 작업들을 취소시키도록 한다.

## 5.2.1 모 아니면 도
테스트를 만들어서 확인해봐야 되는데,
에러가 발생되는 상황을 강제로 만들어야 하므로 예외를 억지로 발생시켜야 한다.
### 테스트용 UserService 대역
작업 중간에 예외를 강제로 만들어야 한다.
- 기존의 코드를 수정하는 것보다 테스트를 위한 클래스를 새로 만드는 방법이 좋다.
- 코드의 중복을 피하기 위해 기존의 코드를 상속받아서 오버라이딩 하는 것이 좋다.
```java
// UserService 의 upgradeLevel()메소드를 오버라이딩해야하므로 접근자 변경
protected void upgradeLevel(User user) { ... }
```

```java
// 테스트 대역 작성
static class TestUserService extends UserService {
	private String id;
    
    private TestUserService(String id) { // 예외를 발생시킬 id를 지정
    	this.id = id;
    }
    
    protected void upgradeLevel(User user) { //UserService 메소드를 오버라이딩
    	// 지정된 id의 User 오브젝트가 발견되면 예외던지기
    	if(user.getId().equals(this.id)) throw new TestUserServiceException(); 
        super.upgradeLevel(user);
    }
}

//테스트에 사용할 예외 정의
static class TestUserServiceException extends RuntimeException {
}
```
### 강제 예외를 통한 테스트
> `테스트의 목적`
> 사용자 레벨 업그레이드를 시도하다가 중간에 예외가 발생했을 경우, 그 전에 업그레이드 했던 사용자도 다시 원래 상태로 돌아갔는지 확인하는 것

```java
@Test
public void upgradeAllOrNothing() {
	//예외를 발생시킬 4번째 사용자의 id를 넣기
	UserServicee testUserService = new TestUserService(users.get(3).getId());
    testUserService.setUserDao(this.userDao); //userDao 수동 DI
    
    userDao.deleteAll();
    for(User user : users) userDao.add(user);
    
    try{
    	testUserService.upgradeLevels();
        fail("TestUSerServiceException expected");
    }
    catch(TestUSerServiceException e) { //해당 예외가 맞다면 그대로 진행
    }
    
    checkLevelUpgraded(users.get(1), false); //예외가 발생하기전, 후가 같은지 확인
}
```
테스트는 실패한다.
두번째 사용자의 레벨이 Basic 에서 Silver로 바뀐 것이 네 번째 사용자 처리 중 예외가 발생했지만 그대로 유지되고 있기 때문에
### 테스트 실패 원인
upgradeLevels() 메소드의 작업은 하나의 작업 단위인 트랜잭션이 적용되지 않았기 때문에 새로 추가된 기술 요건을 만족하지 못 하고 , 이를 검증하기 위해 만든 테스트가 실패하는 것이다.

- upgradeLevels() 메소드가 하나의 트랜잭션 안에서 동작하지 않음
- 하나의 트랜잭션 안에 있어야 예외로 인해 롤백할 때 모두 초기 상태로 바뀜

## 5.2.2 트랜잭션 경계설정
### UserService와 UserDao의 트랜잭션 문제
![[Pasted image 20241105215030.png]]
트랜잭션이 3개 생성됐다.
결국 DAO를 사용하면 비지니스 로직을 담고 있는 UserService 내에서 진행되는 여러 작업을 하나의 트랜잭션으로 묶는 것이 불가능하다.
### 비지니스 로직 내의 트랜잭션 경계설정
> 하나의 트랜잭션에 위 과정을 모두 넣으면서 지금까지 만든 UserService와 UserDao 를 그대로 둔 채 트랜잭션을 적용하기 위해서 경계설정 작업을 UserService로 가져와야 한다.
`upgradeLevels 의 트랜잭션 경계설정 구조`
```java
//upgradeLevels의 트랜잭션 경계설정 구조
public void upgradeLevels() throws Exception {
	1) DB Conncetion 생성
    2) 트랜잭션 시작
    try {
    	3) DAO 메소드 호출
        4) 트랜잭션 커밋
    }
    catch(Exception e) {
    	5) 트랜잭션 롤백
        throw e;
    }
    finally {
    	6) DB Connection 종료
    }
}
```
`Connection 오브젝트를 파라미터로 전달받은 UserDao 메소드`
```java
//Connection 오브젝트를 UserDao에서 사용하기 위해 Connection을 파라미터로 받음
public interface UserDao {
	public void add(Connection c, User user);
    public User get(Connection c, String id);
    ...
    public void update(Connection c, User user1);
}
```

```java
//Connection을 공유하도록 수정한 UserService메소드
class UserService {
	public void upgradeLevels() throws Exception {
    	Connection c = ...; //Connection 생성
        ...
        try{
        	...
            upgradeLevel(c,user); //DAO 메소드 호출
            ... // 트랜잭션 커밋
        }
        ... // 트랜잭션 롤백
    } // 트랜잭션 종료
    
    protected void upgradeLevel(Connection c, User user) {
    	user.upgradeLevel();
        userDao.update(c,user);
    }
}
```
### UserService 트랜잭션 경계설정 문제점
1. DB 커넥션을 비롯한 리소스의 깔끔한 처리를 가능하게 했던 JdbcTemplate을 더이상 활용할 수 없다.
    - 따라서 원래 사용했던 try/catch/finally문을 사용해야 한다.
2. DAO의 메소드와 비지니스 로직을 담고 있는 UserService의 메소드에 Connection 파라미터추가돼야 한다.
    - UserService가 스프링 빈이라서 Connection을 다른 메소드에 전달할 수 없다.(스프링 빈은 싱글톤이기 때문)
3. Connection 파라미터가 UserDao 인터페이스 메소드에 추가되면 UserDao는 더 이상 데이터 액세스 기술에 독립적일 수 없다.
    - JPA, Hibernate 로 구현 방식을 변경하려면 Connection 이 아닌 다른 오브젝트를 사용해야 한다.
    - Connection이 들어간 모든 코드를 수정해야 한다.

## 5.2.3 트랜잭션 동기화
스프링은 위의 딜레마를 해결할 수 있는 방법을 제공해준다.
### Connection 파라미터 제거
> 먼저 Connection을 파라미터로 직접 전달하는 문제를 해결하자.

upgradeLevels() 가 경계설정을 해야 되는 사실은 피할 수 없다.
하지만 DAO 를 호출할 때 Conncetion을 전달하지 않아도 된다면 제거가 가능하다. (스프링이 트랜잭션 동기화를 통해서 이 문제를 해결하기 때문)

> `트랜잭션 동기화란?`
>> 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관해두고, 이후에 호출되는 DAO 의 메소드에서는 저장된 Connection을 가져다 사용하게 하는 것이다.

![[Pasted image 20241105222536.png]]

`트랜잭션 동기화 방식을 적용한 UserService`
```java
private DataSource dataSource;

public void setDataSource(DataSource dataSource) {
	this.dataSource = dataSource;
}

public void upgradeLevels() throws Exception {
	//트랜잭션 동기화 관리자를 이용해 동기화 작업 초기화
	TransactionSynchronizationManager.initSynchronization(); 
    //DB 커넥션을 생성하고 트랜잭션 시작. 이후 작업들은 모두 이 트랜잭션에서 진행
    Connection c = DataSourceUtils.getConnection(dataSource);
    c.setAutoCommit(false);
    
    try{
    	List<User> users = userDao.getAll();
        for(User user : users) {
        	if(canUpgradeLevel(user)) {
            	upgradeLevel(user);
            }
        }
        c.commit(): // 정상적으로 마치면 트랜잭션 커밋
    } catch (Exception e) { // 예외가 발생하면
    	c.rollback(); // 롤백
        throw e;
    } finally {
    	// 스프링 유틸리티 메소드를 이용해 DB 커넥션을 안전하게 닫는다
    	DataSourceUtils.releaseConnection(c, dataSource);
 		
        //동기화 작업 종료 및 정리      
        TransactionSynchronizationManager.unbindResource(this.dataSource);
        TransactionSynchronizationManager.clearSynchronization(); 
    }
}
```
### 트랜잭션 테스트 보완
> 동기화 된 UserService 테스트를 위해 UserServiceTest를 수정해보자

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

### JdbcTemplate 와 트랜잭션 동기화
> JdbcTemplate는 영리하게 동작하도록 설계돼 있다.
> DB 커넥션이나 트랜잭션이 없으면 직접 생성한다.
> 트랜잭션 동기화를 시작해놓은 상태라면 트랜잭션 동기화 저장소에 저장된 DB커넥션을 가져와 사용한다.

## 5.2.4 트랜잭션 서비스 추상화
### 기술과 환경에 종속되는 트랜잭션 경계설정 코드
DB의 연결방법이 바뀌어도 UserDao, UserService 코드를 수정하지 않아도 된다.
DataSource 인터페이스와 DI 를 적용한 덕분에

하나의 트랜잭션에서 여러 개의 DB를 사용하는 것은 **불가능하다**. <u>왜냐 로컬 트랜잭션은 하나의 DB Connection에 종속되기 때문이다.</u>

- 글로벌 트랜잭션 방식 사용
    - 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리한다.
    - 글로벌 트랜잭션을 지원하는 트랜잭션 매니저를 사용한다.
- JTA(Java Transaction API)
    - 자바는 트랜잭션 매니저를 지원하는 API 인 JTA 를 제공
      ![[Pasted image 20241105225110.png]]
    - JTA를 이용한 트랜잭션 코드 구조
  ```java
//JNDI를 이용해 서버의 UserTransaction 오브젝트를 가져옴
InitialContext ctx = new InitialContext();
UserTransaction tx = (UserTransaction)ctx.lookup(USER_TX_JNDI_NAME);

tx.begin();
Connection c = dataSource.getConnection(); - JNDI로 가져온 dataSource를 사용
try {
// 데이터 액세스 코드
tx.commit();
} catch (Exception e) {
tx.rollback();
throw e;
} finally{
c.close();
}
```
- 트랜잭션 경계설정 코드의 문제점
	- JDBC 로컬 트랜잭션을 JTA를 이용하는 글로벌 트랜잭션으로 바꾸려면 UserService를 수정해야 한다.
	- DB연결 방식이 달라지면 트랜잭션 경계설정 코드도 변경해야 한다.

### 트랜잭션 API 의존관계 문제와 해결책
`문제점`
트랜잭션 경계설정 코드를 추가하면서 특정 액세스 기술에 종속되는 구조가 되었다.
![[Pasted image 20241105225429.png]]
`해결책`
- 독립적인 구조로 바꾸기 위해 트랜잭션 경계설정 코드를 추상화한다.
	트랜잭션 경계설정 코드는 DB마다 코드는 다르지만 일정한 패턴을 갖는 유사한 구조이므로 사용방법을 추상화해서 사용할 수 있다.

### 스프링의 트랜잭션 서비스 추상화
스프링은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공하고 있다.
이를 이용하면 일관된 방식으로 트랜잭션을 제어하는 트랜잭션 경계설정 작업이 가능해진다.
![[Pasted image 20241105225747.png]]
`스프링의 트랜잭션 추상화 API를 적용한 upgradeLevels()`
```java
public void upgradeLevels() {
	//JDBC 트랜잭션 추상 오브젝트 생성
	PlatformTransactionManager transactionManager =
    				new DataSourceTransactionManager(dataSource);
    //트랜잭션 시작
    TransactionStatus status =
    	transactionManager.getTransaction(new DefaultTransactionDefinition());
    
    //트랜잭션 진행중//    
    try { 
    	List<User> users = userDao.getAll();
        for(User user : users) {
        	if (canUpgradeLevel(user)) {
            	upgradeLevel(user);
            }
        }
    ////////////////
    
        transactionManager.commit(status); //트랜잭션 커밋
    } catch (RuntimeException e) {
    	transactionManager.rollback(status); //트랜잭션 커밋
        throw e;
    }
}
```
### 트랜잭션 기술 설정의 분리
UserService 클래스가 구체적인 트랜잭션 매니저를 알고 있으면 DI 원칙이 위배된다.
- 컨테이너를 통해 외부에서 제공받게 하는 스프링 DI 방식으로 변경
- 스프링에서 제공하는 **PlatformTransactionManager**는 싱글톤으로 사용 가능
  `트랜잭션 매니저를 빈으로 분리시킨 UserService`
```java
pubilc class UserService {
	...
    private PlatformTransactionManager transactionManager; ///DI받을 변수 생성
    
    public void setTransactionManager(PlatformTransactionManager 
    		transactionManager) {
    	this.transactionManager = transactionManager;        
    }
    
    public void upgradeLevels() {
        //트랜잭션 시작
        TransactionStatus status =
            this.transactionManager.getTransaction(new 
            	DefaultTransactionDefinition()); //DI받은 매니저 공유해서 사용

        //트랜잭션 진행중//    
        try { 
            List<User> users = userDao.getAll();
            for(User user : users) {
                if (canUpgradeLevel(user)) {
                    upgradeLevel(user);
                }
            }
        ////////////////

            this.transactionManager.commit(status); //트랜잭션 커밋
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status); //트랜잭션 커밋
            throw e;
        }
    }
    ...
```
`테스트도 수정- 트랜잭션 매니저를 수동 DI 하도록 수정한 테스트`
```java
@Autowired PlatformTransactionManager transactionManager;
...
@Test
public void upgradeAllOrNothing() throws Exception {
	UserService testUserService = new TestUserService(users.get(3).getId());
    testUserService.setUserDao(this.userDao);
    testUserService.setTransactionManager(transactionManager); // 수동 DI
	...
```
# 5.3 서비스 추상화와 단일 책임 원칙
### 수직 , 수평 계층구조와 의존관계
- 같은 애플리케이션 로직을 담은 코드지만 내용에 따라 분리했다. - 수평적 분리
  UserDao 와 UserService 는 애플리케이션 코드이지만 내용에 따라 분리된 코드
  두 클래스 간의 결합도가 낮아 독립적인 확장 가능
- 트랜잭션의 추상화 코드 - 수직적 분리
  애플리케이션의 비지니스 로직과 로우레벨의 트랜잭션 기술이 서로 분리
  두 계층간 결합도가 낮아서 기술이 변경돼도 애플리케이션 코드는 영향 받지 않는다.

### 단일 책임 원칙(SRP)
하나의 모듈은 한가지 책임을 져야 한다.
- 모듈이 수정되는 이유가 한 가지여야 한다.

> UserService 가 Connection을 직접 사용하던 경우
>> 두가지 책임이 존재했다.
>> 1. 레벨 업그레이드와 관련된 로직이 변경되는 경우
>> 2. UserTransaction을 사용하는 JTA 로 변경하는 경우 (단일 책임 원칙 위배)

### 단일 책임 원칙 장점
수정 대상이 명확하다.
- 기술이 변경되는 경우 기술 코드만 변경하면 되고, 비지니스 로직이 변경되면 애플리케이션 코드만 수정하면 된다.
# 5.4 메일 서비스 추상화
> 레벨이 업그레이드 되는 사용자에게는 안내 메일을 발송해달라는 요청 사항이 들어왔다.

## 5.4.1 JavaMail을 이용한 메일 발송 기능
자바에서 메일을 발송할 때는 표준 기술인 JavaMail 을 사용하면 된다.
```java
//레벨 업그레이드 작업 메소드 수정

protected void upgradeLevel(User user) {
	user.upgradeLevel();
    userDao.update(user);
    sendUpgradeEMail(user);
}
```

```java
//JavaMail을 이용한 메일 발송 메소드 작성
private void sendUpgradeEMail(User user) {
	Properties props = new Properties();
    props.put("mail.smtp.host", "mail.ksug.org");
    Session s = Session.getInstance(props, null);
    
    MimeMessage message = new MimeMessage(s);
    try{
    	message.setFrom(new InternetAddress("useradmin@ksug.org"));
        message.addRecipient(Message.RecipientType.TO,
        						new InternetAddress(user.getEmail()));
        message.setSubject("Upgrade 안내");
        message.setText("사용자님의 등급이 " + user.getLevel().name() +
        	"로 업그레이드되었습니다.");
            
        Transport.send(message); 
    } catch(AddressException e) {
    	throw new RuntimeException(e);
    } catch (MessagingException e) {
    	throw new RuntimeException(e);
    } catch (UnsupportedEncodingException e) {
    	throw new RuntimeException(e);
    }
}
```
### 테스트 대역의 종류와 특징
**테스트 대상**이 되는 **오브젝트의 기능에만 충실하게 수행**하면서 **빠르게, 자주 실행가능**할 수 있도록 사용하는 오브젝트

- **테스트 스텁(test stub)** : **테스트 대상 오브젝트의 의존객체**로서 존재하면서 **테스트가 정상적으로 수행되도록 돕는 것**
    - 위에서 만든 DummyMailSender
- **목 오브젝트(mock object)** : 테스트 대상 오브젝트와 의존 오브젝트 사이에 일어나는 일을 **검증할 수 있도록 특별히 설계된 오브젝트**
### 목 오브젝트를 이용한 테스트
> 목 오브젝트를 이용해 테스트를 수행해보자

목 오브젝트로 만든 메일 전송 확인용 클래스

```java
static class MockMailSender implements MailSender {
	private List<String> requests = new ArrayList<String>();
    
    public List<String> getRequests() {
    	return requests;
    }
    
    //목 오브젝트도 실제로 전송하지 않으므로 거의 빈껍데기다!
    public void send(SimpleMailMessage mailMessage) throws MailException {
    	requests.add(mailMessage.getTo()[0]); // 전송 요청을 받은 이메일 저장
    }
    
    public void send(SimpleMailMessage[] mailMessage) throws MailException {
    }
}
```

메일 발송 대상을 확인하는 테스트

```java
@Test
@DirtiesContext

public void upgradeLevels() throws Exception {
	userDao.deleteAll();
    for(User user : users) userDao.add(user);
    
    MockMailSender mockMailSender = new MockMailSender();
    userService.setMailSender(mockMailSender);
    
    userService.upgradeLevels();
    
    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), true);
    checkLevelUpgraded(users.get(2), false);
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false);
    
    //목 오브젝트에 저장된 메일 수신자 목록을 가져와 업그레이드 대상과 일치하는지 확인
    List<String> request = mockMailSender.getRequests();
    assertThat(request.size(), is(2));
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));
}
```