사실은 
예외에 대해서 생각하기란 귀찮은 법이다. 
</br>
사용자가 중간고사를 끝내고 너무 목이 고파 이태원 펍에 갔다.

```
//Itaewon Pub programming
if 맥주 != null,
	do 맥주 주문하기
    
맥주 주문하기(taste) {
	if taste = '라거', return '카스'
    if taste = 'IPA', return 'India'
    else throw new IllegalArgumentException("당신의 취향은 우리 가게와 맞지 않습니다.");
}
```

</br>
</br>
화장실이 있을거라 기대한 사용자는 펍에 들어가 화장실이 어디냐고 물어봤다. </br>
펍은 불타올랐고 먼지와 재만이 남았다. 


![](https://velog.velcdn.com/images/ykky2115/post/c8168d78-92df-48a5-8211-8305aea5e78b/image.png)

기능 요구 사항을 따르는 정상적 기능 구현도 충분히 버겁지만 예외처리를 하지 않는다면 우리의 프로젝트는 신기방통한 유저에 의해 불 타오를 수 있다. 
</br>

이번 장에서는 JdbcTemplate을 대표로 하여 스프링의 데이터 액세스 기능에 담긴 예외처리와 관련된 접근 방법에 대해 알아본다. 이를 통해 예외 처리 베스트 프랙티스도 살펴보면 좋을 것이다. 

## JdbcTemplate 전에...

자바에는 세 가지의 예외가 있다. 

![](https://velog.velcdn.com/images/ykky2115/post/f9b580c5-fd18-47a5-9abb-f6871159828b/image.png)

1. checked exception
Error, RuntimeExceptoin과 하위 클래스를 제외한 모든 예외이다. 
catch or specify requirement을 필요로 하는 대상이다.

2. error(unchecked)
애플리케이션의 외적인 요소로 catch or specify requirement 대상이 아니다.

3. runtime exception(unchecked)
로직 에러나 API를 잘못된 사용을 하려했을때 일어난다. catch or specify requirement 대상이 아니다.

JdbcTemplate에서의 SQLException 혹은 IOException는 checked exception이고
NullPointerException, IllegalArgumentException은 runtime exception에 속한다.

**Spring transactional API는 runtimeException에서만 rollback을 하고 checked의 경우는 따로 처리하지 않는다.**

**이유는 RuntimeException 클래스가 일반적으로 Spring에서 복구 불가능한 오류 조건을 나타내기 때문이다.** 
(EJB 때부터의 관습이기도 하다)

때문에 checkedException의 경우에는 수동 롤백 처리를 해야 한다.

여러 논쟁이 오갔고 현재는 unchecked exception을 권장하는 편이다.
클린 코드 (7장 오류 처리)에서도 이를 권장한다.

한 편 체크와 비체크 기준은 다음과 같다.

- 체크: 외부 환경 혹은 예측 불가 조건에 의해 발생
- 비체크: 프로그래밍 오류, 개발자가 사전에 방지 가능
---
## 4. 예외 처리 방법론
예외 처리 방법론에는 세 가지가 있다.
- 예외 **복구** 
- 예외 **회피** 
- 예외 **전환**

### 예외 복구

네트워크 불안정 등의 이유로 원격 DB 서버 접속이 실패하여 SQLException이 터진다고 가정한다. 
예외가 발생했다면 개발자는 해당 예외의 <u>존재 가능성</u>을 알고 이 예외에 대한 적절한 처리를 시도하도록 하는 것이다.  

```
static final int MAX_RETRY = 10;

public Object AMethod() {
	private ine maxRetry = MAX_RETRY;
    
    while(maxRetry --> 0) {
    	try {
        	//예외가 발생할 가능성이 있는 시도
            return; //작업 성공 시 바로 반환
        }
        catch (SomeException e) {
        	//로그 출력. 정해진 시간만큼 대기
        }
        finally {
        	//리소스 반납. (정리 작업)
        }
        --maxRetry;
    }
    throw new RetryFailedException();
}

```

### 예외 회피
회피란 자신이 담당하지 않고 자신을 호출한 쪽으로 예외를 던져버리는 것이다. 그러나 회피라고 해서 무책임한 행위는 옳지 않다. 콜백/템플릿 처럼 긴밀한 관계에 있는 오브젝트에게 책임을 분명히 지게 하거나, 
자신을 호출한 쪽에서 예외를 다루는 게 최선이라고 확신이 들 때 써야 한다. 그리 추천되는 방식은 아닌 듯 하다.
대표적인 회피 방법은 다음 코드와 같다. 

```
public void add() throws SQLException {
    try {
        // ... 생략
    } catch(SQLException e) {
    	e.printStackTrace(); // 로그만 출력하고
        throw e; // 다시 날린다
    }
}

```

### 예외 전환
예외 회피와 같이 내 기능 안에서는 정상적인 상태로 예외 복구 불가능 하다 판단할 때 쓰는 것인데, 다른 점이라면 발생한 예외를 그대로 넘기는 게 아니고 <u>무언의 작업(다른 예외로 바꾸기)</u>을 한다는 것이다.

#### 예외 전환의 두 가지 목적과 그에 따른 방식
1. 내부에서 발생환 예외를 그대로 전달하는 것보다 더 적절한 의미를 부여하고 싶을 때

 동일 아이디로 인해 JDBC API에서 SQLException이 발생한다. 그런데 비즈니스 로직을 짜는 서비스 계층에서 SQLException을 딱 받고 나서는 어떤 생각을 할 수 있을까?
 
>  아 SQL이 이상햇구나


   이것보다는 중복 아이디라 실패했다는 예외 전환(ex. DuplicatedUserIdException)이 더 낫지 않을까?
   이렇게 한다면 충분히 예상 가능하고 복구 가능한 시츄에이션 처럼 느껴진다. 다음은 이를 위한 DAO 메서드이다.
   
   ```
public class DuplicatedUserIdException extends RuntimeException {
	public DuplicatedUserIdException(Throwable cause) {
    		super(cause);
       }
}

public void add() throws DuplicatedUserIdException {
	try {
    //서비스 로직
    }
    catch(SQLException e) {
    	if (e.getError() == MysqlErrorNumbers.ER_DUP_ENTRY) throw new DuplicatedUserIdException(e);
        else 
        	throw new RuntimeException(e);
    }
}
```

물론, add() throws DuplicatedUserIdException, SQLException을 해도 됐겠지만 체크 예외인 SQLEx은 대개 나타나도 복구 불가능한 예외(에러가 났는데요.. 뭐 어쩌죠?)이므로 차라리 런타임 예외 처리를 하여 복구 가능할 수 있는 선에서 나타내는 것이 좋다.

### SQLException -> DataAccessException
 
 앞서 언체크 예외를 권장한다고 하였다. 그런데 SQLException는 체크 예외로서 복구가 불가능한 것으로 위장한다. 그래서 JdbcTemplate는 제공하는 기능에 기본적으로 throws DataAccessException를 하여 SQLException을 wrapping 한다. 

throws Exception으로 모든 예외를 통틀수도있지만 이 방법은 다소 무책임하다.!!!

</br>

 
## 애플리케이션 예외
 
애플리케이션 자체의 로직에 의해 의도적으로 발생시키고 반드시 catch문을 통하여 어떻게든 무엇이든 예외 처리를 요구하는 방식.

**좋은 애플리케이션 예외 설계방법?**

- 정상적인 흐름을 따르는 코드는 그대로 두고, 잔고 부족과 같은 예외 상황에서는 비즈니스적인 의미를 띤 예외를 던지도록 설계
- 이때 사용하는 예외는 의도적으로 체크 예외로 만듬. 그래서 개발자가 잊지 않고 잔고 부족처럼 자주 발생 가능한 예외 상항에 대한 로직을 구현하도록 강제.


## 정리
- 현재는 언체크 예외를 권장하고 있고 JdbcTemplate의 경우에도 기존 체크예외인 SQLException을 DataAccessException으로 바꾸어 예외 처리 전략을 취하고 있다. 
- 아무 행위 없는 throws Exception e, printstactktree(e); 등은 지양한다.
- 모든 예외를 잡는 Exception이 아닌 각 목적에 맞는 특정 예외를 처리하도록 한다.
