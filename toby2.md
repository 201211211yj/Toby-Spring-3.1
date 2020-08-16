# 2. 테스트

## 2.1 UserDaoTest 다시보기

### UserDaoTest의 특징
아래 테스트 코드는 main() 메소드를 이용해 UserDao 오브젝트의 add(), get() 메소드를 호출하고 그 결과를 출력한다.<br>
```java
public class UserDaoTest{
	public static void main(String[] args) throws SQLException{
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
		
		UserDao dao = context.getBean("userDao", UserDao.class);
		
		User user = new User();
		user.setId("user");
		user.setName("영준");
		user.setPassword("hello");
		
		dao.add(user);
		
		System.out.println(user.getId() + "등록 성공");
		
		User user2 = dao.get(user.getId());
		System.out.println(user2.getName());
		
		System.out.println(user2.getId() + "조회 성공");
	}
}
```

<br>

#### 웹을 통한 DAO 테스트 방법의 문제점
웹 화면을 통해 값을 입력하고, 기능을 수행하고, 결과를 확인하는 방법은 가장 흔히 씌는 방법이지만 DAO자체에 대한 테스트로는 단점이 많다. DAO뿐만 아니라 서비스 클래스, 컨트롤러, JSP 뷰 등 모든 레이어의 기능을 다 만들고 나서야 테스트가 가능하다는 점이 가장 큰 문제다. 테스트를 하는 중에 에러가 나거나 테스트가 실패했따면, 과연 어디에서 문제가 발생했는지를 찾아내야 하는 수고가 필요하다. 이런 방식으로 테스트하는 것은 오류가 있을 때 빠르고 정확하게 대응하기가 힘들다는 문제가 있다.

<br>

#### 작은 단위의 테스트
테스트하고자 하는 대상이 명확하다면 그 대상에만 집중해서 테스트하는 것이 바람직하다. UserDaoTest는 한 가지 관심에 집중할 수 있게 작은 단위로 만들어진 테스트다. 웹 인터페이스, MVC 클래스, 서비스 오브젝트 등이 필요가 없다. 서버에 배포할 필요도 없다. <br>
이렇게 작은 단위의 코드에 대해 테스트를 수행한 것을 단위 테스트(unit test)라고 한다. 여기서 말하는 단위란 무엇인지, 크기와 범위가 어느 정도인지 딱 정해진 건 아니다. 충분히 하나의 관심에 집중해서 효율적으로 테스트할 만한 범위의 단위라고 보면 된다. <br>

<br>

### UserDaoTest의 문제점
* **수동 확인 작업의 번거로움** : UserDaoTest는 모두 자동으로 진행하도록 만들어졌지만, 여전히 사람의 눈으로 확인하는 과정이 필요하다. add()에서 User 정보를 DB에 등록하고, 이를 다시 get()을 이용해 가져왔을 떄 입력한 값과 가져온 값이 일치하는지를 테스트코드는 확인해주지 않는다.
* **실행 작업의 번거로움** : 아무리 간단히 실행 가능한 main()메소드라 해도 매번 그것을 실행하는 것은 제일 번거롭다. 전체 기능을 테스트해보기 위해 main()메소드를 수백 번 실행하는 수고가 필요할 수도 있다.

<br>

## 2.2 UserDaoTest 개선
### 테스트 검증의 자동화
첫 번째 문제점인 테스트 결과의 검증 부분을 코드로 만들어보자.
```java
public class UserDaoTest{
	public static void main(String[] args) throws SQLException{
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
		
		UserDao dao = context.getBean("userDao", UserDao.class);
		
		User user = new User();
		user.setId("user");
		user.setName("영준");
		
		dao.add(user);
				
		User user2 = dao.get(user.getId());
		
		if(!user.getName().equals(user2.getName())){
			System.out.println("테스트 실패 (name)");
		}else if(!user.getPassword().equals(user2.getPassword())){
			System.out.println("테스트 실패 (password)");
		}else{
			System.out.println("테스트 성공");
		}
	}
}
```

### 테스트의 효율적인 수행과 결과 관리
좀 더 편리하게 결과를 확인하려면 main()메소드 만으로는 한계가 있다. 자바에는 단순하면서도 실용적인 테스트를 위한 도구가 여러가지 존재한다.

#### JUnit 테스트로 전환
테스트를 프레임워크에 의해 동작하게 함으로써 IoC를 적용한다.

<br>

#### 테스트 메소드의 전환
가장 먼저 할 일은 main()메소드에 있던 테스트 코드를 public 메소드로 변경하고 @Test 어노테이션을 붙혀준다.
```java
import org.junit.Test;
...
public class UserDaoTest{
	@Test
	public void addAndGet() thorws Exception{
		ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
		
		UserDao dao = context.getBean("userDao", UserDao.class);
		...
	}
}
```

<br>

#### 검증 코드 전환
if/else 문장을 JUnit이 제공하는 방법을 이용해 전환해보자.
```java
if(!user.getName().equals(user2.getName())){...}
```


```java
assertThat(user2.getName(), is(user.getName()));
```
assertThat()메소드는 첫 번쨰 파라미터의 값을 뒤에 나오는 매처(matcher)라고 불리는 조건으로 비교해서 일치하면 다음으로 넘어가고, 아니면 테스트가 실패하도록 만들어준다.
<br>

```java
import static org.hamcrest.CoreMatchers.is;
import static org.junit.Assert.assertThat;
...
public class UserDaoTest{
	public static void main(String[] args) throws SQLException{
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
		
		UserDao dao = context.getBean("userDao", UserDao.class);
		
		User user = new User();
		user.setId("user");
		user.setName("영준");
		
		dao.add(user);
				
		User user2 = dao.get(user.getId());
		
		assertThat(user2.getName(), is(user.getName()));
		assertThat(user2.getPassword(), is(user.getPassword()));
	}
}
```
■ 추가할 라이브러리
* com.springsource.org.junit

#### JUnit 테스트 실행
JUnit 프레임워크를 이용해 앞에서 만든 테스트 메소드를 실행하도록 해보자.
```java
import org.junit.runner.JUnitCore;
...
public static void main(String[] args){
	JUnitCore.main("spring...UserDaoTest");
}
```
<br>
이 코드를 실행하면 다음과 같은 메세지가 출력된다.

```
JUnit version 4.7
Time: 0.578
OK (1 test)
```

만약 코드에 이상이 있어 검증에 실패할 경우 다음과 같은 메세지가 출력된다.

```
Time: 1.094
There was 1 failure:
1) addAndGet(spring...UserDaoTest)
java.lang.AssertionError:
Expected: is "영준"
     got: null
     ...
        at spring...UserDaoTest.main(UserDaoTest.java:36)
FAILURES!!!
Tests run : 1, Failures: 1
```

<br>

## 2.3 개발자를 위한 테스팅 프레임워크 JUnit
### JUnit 테스트 실행 방법

#### IDE
이클립스에서 @Test가 들어 있는 테스트 클래스를 선택한 뒤에 Run As - JUnit Test를 선택하면 테스트가 자동으로 실행된다.
<br>
테스트가 시작되면 JUnit 테스트 정보를 표시해주는 뷰(View)가 나타나서 테스트 진행 상황을 보여준다.
<br>
소스 트리를 선택하고 Run As - JUnit Test를 실행하면 해당 패키지 아래에 있는 모든 JUnit 테스트를 한번에 실행해준다.

#### 빌드 툴
