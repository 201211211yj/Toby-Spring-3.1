# 3. 템플릿

## 3.1 UserDao 재정비
UserDao에는 아직 문제점이 남아있다. 예외상황에 대한 처리가 미흡한 것이 문제점이다.

### 예외처리 기능을 갖춘 DAO
JDBC 코드에는 예외처리를 반드시 지켜야한다. 정상적인 JDBC 흐름을 따르지 않고 중간에 어떤 이유로든 예외가 발생했을 경우에도 사용한 리소스를 반환해야한다.

#### JDBC 수정 기능의 예외처리 코드

```java
public void delteAll throws Exception{
	Connection c = dataSource.getConnection();
	
	PreparedStatement ps = c.prepareStatement("delete from users");
	ps.executeUpdate();
	
	ps.close();
	c.close();
}
```

이 메소드에서는 Connection과 PreparedStatement라는 두 개의 공유 리소스를 가져와서 사용한다. 물론 정상적으로 처리되면 메소드를 마치기 전에 각각 close()를 호출해 리소스를 반환한다. 하지만 PreparedStatement 처리 도중 예외가 발생한다면 메소드 실행을 끝마치지 못하고 빠져나간다. 이 경우 close() 메소드가 실행되지 않아 제대로 리소스가 반환되지 않을 수 있다. <br>
일반적으로 서버에서는 제한된 개수의 DB 커넥션을 만들어서 재사용 가능한 풀로 관리한다. DB풀은 매번 getConnection()으로 가져간 커넥션을 명시적으로 close()해서 돌려줘야 다시 풀에 넣었다가 다음 커넥션 요청이 있을 때 재사용할 수 있다. 하지만 이런식으로 오류가 발생되어 Connection이 계속 쌓이게 되면 어느 순간에 커넥션 풀에 여유가 없어지고 리소스가 모자란다는 심각한 오류를 내며 서버가 중단될 수 있다. <br>
<br>
**리소스 반환과 close()** 
<br>
Connection이나 PreparedStatement에는 close()메소드가 있다. 보통 리소스를 반환한다는 의미로 이해하는 것이 좋다. 이 둘은 보통 pool 방식으로 운영된다. 미리 정해진 풀 안에 제한된 수의 리소스(Connection, Statement)를 만들어두고 필요할 때 이를 할당하고, 반환하면 다시 풀에 넣는 방식으로 운영된다. 요청이 매우 많은 서버환경에서는 매번 새로운 리소스를 생성하는 대신 풀에 미리 만들어둔 리소스를 돌려가며 사용하는 편이 유리하다. 대신 사용한 리소스는 빠르게 반환해야한다. 그렇지 않으면 풀에 리소스가 고갈될 수 있다.
<br>
<br>

이런 JDBC코드에서는 어떤 상황에서도 가져온 리소스를 반환하도록 try/catch/finally 구문 사용을 권장한다. <br>

```java
public void deleteAll() throws SQLException{
	Connection c = null;
	PrepareStatement ps = null;
	
	try{
		c = dataSource.getConnection();
		ps = c.prepareStatement("delete from users");
		ps.executeUpdate();
	} catch (SQLException e) {
		throw e;
	} finally {
		if (ps != null){
			try{
				ps.close();
			} catch (SQLException e){
				//ps.close()메소드에서도 SQLException이 발생할 수 있다.
			}
		}
		
		if (c != null){
			try{
				c.close();
			} catch (SQLException e){
			}
		}
	}
}
```

#### JDBC 조회 기능의 예외처리
조회를 위한 JDBC 코드는 ResultSet이 추가되고 이 또한 반환되어야 한다.
```java
public int getCount() throws SQLException{
	Connection c = null;
	PreparedStatement ps = null;
	ResultSet rs = null;
	try{
		c = dataSource.getConnection();
		ps = c.prepareStatement("select count(*) from users");
		
		rs = ps.executeQuery();
		rs.next();
		return rs.getInt(1);
	} catch (SQLException e) {
		throw e;
	} finally {
		if(rs != null){
			try{
				rs.close();
			} catch (SQLException e){
			}
		}
		
		if (ps != null){
			try{
				ps.close();
			} catch (SQLException e){
			}
		}
		
		if (c != null){
			try{
				c.close();
			} catch (SQLException e){
			}
		}
	}
}	
```

## 3.2 변하는 것과 변하지 않는 것
### JDBC try/catch/finally 코드의 문제점
위 코드의 문제점은 try/catch/finally가 2중으로 중첩하며, 모든 메소드마다 반복된다는 점이다. 

<br>

### 분리와 재사용을 위한 디자인 패턴 적용 
```java
public void deleteAll() throws SQLException{
	Connection c = null;
	PrepareStatement ps = null;
	
	try{
		c = dataSource.getConnection();
		
		ps = c.prepareStatement("delete from users");
		/*
		이 코드만 제외하고 나머지는 전부 변하지 않는 코드이다.
		만일 add() 메소드라면
		ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());
		*/
		
		ps.executeUpdate();
	} catch (SQLException e) {
		throw e;
	} finally {
		if (ps != null){
			try{
				ps.close();
			} catch (SQLException e){
			}
		}
		
		if (c != null){
			try{
				c.close();
			} catch (SQLException e){
			}
		}
	}
}
```

#### 메소드 추출
변하지 않는 부분이 변하는 부분을 감싸고 있어서 변하는 부분만 추출해본다.

```java
public void deleteAll() throws SQLException{
	Connection c = null;
	PrepareStatement ps = null;
	
	try{
		c = dataSource.getConnection();
		
		ps = c.prepareStatement("delete from users");
		
		ps.executeUpdate();
	} catch (SQLException e) {
		...
	}
	
	private PreparedStatement makeStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```

<br>

#### 템플릿 메소드 패턴의 적용
다음은 템플릿 메소드 패턴을 이용해서 분리해보자. 템플릿 메소드 패턴은 상속을 통해 기능을 확장해서 사용하는 부분이다. 변하지 않는 부분은 슈퍼클래스에 두고 변하는 부분은 추상 메소드로 정의해둬서 서브클래스에서 오버라이드하여 새롭게 정의해 쓰도록 한다.
<br>
```java
abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
```
<br>
```java
public class UserDaoDeleteAll extends UserDao{
	protected PreparedStatement makeStatement(Connection c) throws SQLException{
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```
<br>
위 코드의 문제는 DAO로직마다 상속을 통해 새로운 클래스를 만들어야 한다는 점이다. 만약 이 방식을 사용한다면 UserDao의 JDBC메소드가 4개일 경우 그림 3-1과 같이 4개의 서브클래스를 만들어서 사용해야한다. 장점보다 단점이 더 많아 보인다.
<br>
(그림 3-1 추가 예정)
<br>
또 확장구조가 이미 클래스를 설계하는 시점에서 고정되어 버린다는 점이다. 변하지 않는 코드를 가진 UserDao의 JDBC(try/catch/finally 블록과 변하는 PreparedStatement를 담고 있는 서브클래스들이 이미 클래스 레벨에서 컴파일 시점에 이미 그 관계가 결정되어있다. 따라서 관계에 대한 유연성이 떨어져 버린다.

<br>

#### 전략 패턴의 적용
개방 폐쇄 원칙(OCP)을 잘 지키는 구조이면서 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어난 것이, 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 전략 패턴이다. 그림 3-2는 전략 패턴의 구조를 나타낸다.
<br>
(그림 3-2 추가 예정)
<br>

deleteAll() 메소드에서 변하지 않는 부분이 contextMethod()가 된다. deleteAll()은 JDBC를 이용해 DB를 업데이트하는 작업이라는 변하지 않는 맥락(context)을 갖는다.
* DB 커넥션 가져오기
* PreparedStatement를 만들어줄 외부 기능 호출하기
* 전달받은 PreparedStatement 실행하기
* 예외게 발생하면 메소드 밖으로 던지기
* 모든 경우에 만들어진 PreparedStatement와 Connection을 닫아주기
<br>
두 번째 작업에서 사용하는 PreparedStatement를 만들어주는 외부 기능이 바로 전략 패턴에서 말하는 전략이라고 볼 수 
있다. 이 내용을 인터페이스로 정의하면 다음과 같다.

```java
public interface StatementStrategy{
	PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```

이 인터페이스를 상속해서 실제 전략, 즉 바뀌는 부분인 PreparedStatement를 생성하는 클래스를 만들어보자.

```java
public class DeleteAllStatement implements StatementStrategy{
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```

이제 확장된 DeleteAllStatement를 deleteAll() 메소드에서 사용하면 전략 패턴을 적용했다고 볼 수 있다.

```java
public void deleteAll() throws SQLException{
	Connection c = null;
	PrepareStatement ps = null;
	
	try{
		c = dataSource.getConnection();
		
		StatementStrategy strategy = new DeleteAllStatement();
		ps = strategy.makePreparedStatement(c);
		
		ps.executeUpdate();
	} catch (SQLException e) {
		...
	}
}
```

하지만 전략 패턴은 필요에 따라 컨텍스트는 그대로 유지되면서 전략을 바꿔 쓸 수 있다는 것인데, 이렇게 구체적인 전략 클래스인 DeleteAllStatement를 사용하도록 고정되어있다. 따라서 OCP패턴에 잘 들어맞지 않는다.

#### DI 적용을 위한 클라이언트/컨텍스트 분리
Context가 어떤 전략을 사용하게 할 것인가는 Client가 결정하는 것이 일반적이다. Client가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 Context에 전달하는 것이다.
<br>
(그림 3-3 추가 예정)
<br>
```java
public void jdbcContextWithStatementStrategy(StatementStrategy stat) throws SQLException{
	Connection c = null;
	PreparedStatement ps = null;
	
	try {
		c = dataSource.getConnection();
		ps = stat.makePreparedStatement(c);
		ps.executeUpdate();
	} catch (SQLException e){
		throw e;
	} finally {
		if (ps != null){
			try{
				ps.close();
			} catch (SQLException e){
			}
		}
		
		if (c != null){
			try{
				c.close();
			} catch (SQLException e){
			}
		}
	}
}
```

다음은 클라이언트에 해당하는 부분이다.

```java
public void deleteAll() throws SQLException{
	StatementStrategy st = new DeleteAllStatement();
	jdbcContextWithStatementStrategy(st);
}
```

아직까지는 이렇게 분리한 것에서 크게 장점이 보이지 않는다. 하지만 지금까지 해온 관심사를 분리하고 유연한 확장관계를 유지하도록 만든 작업은 매우 중요하다. 이 구조가 기반이 되어서 앞으로 진행할 UserDao 코드의 본격적인 개선 작업이 가능하다.
<br>
*마이크로 DI* : 일반적으로 DI는 의존관계에 있는 두 개의 오브젝트와 이 관계를 다이나믹하게 설정해주는 오브젝트 팩토리, 그리고 이를 사용하는 클라이언트라는 4개의 오브젝트 사이에서 일어난다. 하지만 때로는 원시적인 전략 패턴 구조를 따라 클라이언트가 오브젝트 팩토리의 책임을 함께 지고 있을 수도 있다. 이런 경우에는 DI가 매우 작은 단위의 코드와 메소드 사이에서 일어나기도 한다. 이렇게 DI의 장점을 단순화해서 IoC 컨테이너의 도움 없이 코드 내에서 적용한 경우를 마이크로 DI라고 한다.

<br>

## JDBC 전략 패턴의 최적화
### 전략 클래스의 추가 정보
이번엔 add() 메소드에도 적용해보자.
```java
public class AddStatement implements StatementStrategy {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException{
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword()); // 그런데 user는 어디서 가져올까
		
		return ps;
	}
}
```

컴파일 에러가난다. 여기서는 user라는 부가정보가 필요하다.

```java
public class AddStatement implements StatementStrategy {
	User user;
	
	public AddStatement(User user){
		this.user = user;
	}
	
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException{
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());
		
		return ps;
	}
}
```

위 코드를 add()메소드에 적용시키면 아래와 같다.

```java
public void add(User user) throws SQLException {
	StatementStrategy st = new AddStatement(user);
	jdbcContextWithStatementStrategy(st);
}
```

<br>

### 전략과 클라이언트의 동거
위 코드에도 두 가지 단점이 있다.<br>
첫 째, DAO메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 한다는 점이다. 이렇게 되면 기존 UserDao때보다 클래스 파일의 개수가 많이 늘어난다. 이렇게 되면 런타임 시에 다이나믹하게 DI 해준다는 점을 제외하면 로직마다 상속을 사용하는 템플릿 메소드 패턴을 적용했을 때보다 나을게 없다. <br>
둘 째, DAO 메소드에서 StatementStrategy에 전달할 User와 같은 부가정보가 있는 경우, 이를 위해 오브젝트를 전달받는 생성자와 이를 저장해둘 인스턴스 변수를 번거롭게 만들어야 한다는 점이다. <br>

<br>

#### 로컬 클래스
클래스 파일이 많아지는 문제는 StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의해버리는 것이다. DeleteAllStatement나 AddStatement는 UserDao 밖에서는 사용되지 않는다. 

```java
public void add(User user) throws SQLException{
	class AddStatement implements StatementStrategy {
		User user;

		public AddStatement(User user){
			this.user = user;
		}

		public PreparedStatement makePreparedStatement(Connection c) throws SQLException{
			PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
			ps.setString(1, user.getId());
			ps.setString(2, user.getName());
			ps.setString(3, user.getPassword());

			return ps;
		}
	}
	
	StatementStrategy st = new AddStatement(user);
	jdbcContextWithStatementStrategy(st);
}
```

로컬 클래스는 선언된 메소드 안에서만 사용할 수 있다. AddStatement가 사용될 곳이 add() 메소드 뿐이라면, 이렇게 사용하는것도 좋은 방법이다.
<br>
로컬 클래스에는 또 한가지 장점이 있다. 바로 로컬 클래스는 클래스가 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있다는 점이다. AddStatement는 user 정보를 필요로 하고, 이렇게 메소드 내에 AddStatement 클래스를 정의하면 번거롭게 생성자를 통해 User 오브젝트를 전달할 필요가 없다. <br>
내부 메소드는 자신이 정의된 메소드의 로컬 변수에 직접 접근할 수 있기 때문이다. 메소드 파라미터도 일종의 로컬 변수이므로 add() 메소드의 user 변수를 AddStatement에서 직접 사용할 수 있다. 다만 내부 클래스에서 외부의 변수를 사용할 때는 외부 변수는 반드시 final로 선언해줘야 한다. user파라미터는 메소드 내부에서 변경될 일이 없으므로 final로 선언해도 무방하다.

```java
public void add(final User user) throws SQLException{
	class AddStatement implements StatementStrategy {
		public PreparedStatement makePreparedStatement(Connection c) throws SQLException{
			PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
			ps.setString(1, user.getId());
			ps.setString(2, user.getName());
			ps.setString(3, user.getPassword());

			return ps;
		}
	}
	
	StatementStrategy st = new AddStatement();
	jdbcContextWithStatementStrategy(st);
}
```

<br>

#### 익명 내부 클래스
AddStatement 클래스는 add() 메소드에서만 사용할 용도로 만들어졌다. 그렇다면 좀 더 간결하게 클래스 이름도 제거할 수 있다.

```java
StatementStrategy st = new StatementStrategy() {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());

		return ps;
	}
}
```

만들어진 익명 내부 클래스의 오브젝트는 딱 한 번만 사용할 테니 굳이 변수에 담아둘 필요가 없다.

```java
public void add(final User user) throws SQLException{
	jdbcContextWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
				ps.setString(1, user.getId());
				ps.setString(2, user.getName());
				ps.setString(3, user.getPassword());

				return ps;
			}
		}
	);
}
```

마찬가지로 deleteAll()도 구현한다면 다음과 같다.

```java
public void deleteAll throws SQLException{
	jdbcContextWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement("delete from users");
			}
		}
	);
}
```

<br>

## 컨텍스트와 DI
### JdbcContext의 분리
전략 패턴의 구조로 보자면 UserDao의 메소드가 클라이언트이고, 익명 내부 클래스로 만들어지는 것이 개발적인 전략, jdbcContextWithStatementStrategy() 메소드는 컨텍스트이다. 컨텍스트 메소드는 UserDao 내의 PreparedStatement를 실행하는 기능을 가진 메소드에서 공유할 수 있다. 그런데 JDBC의 일반적인 작업 흐름을 담고 있는 jdbcContextWithStatementStrategy()는 다른 DAO에서도 사용 가능하다. 그러므로 jdbcContextWithStatementStrategy()를 UserDao 클래스 밖으로 독립시켜 모든 DAO가 사용할 수 있도록 해보자.

<br>

#### 클래스 분리
JdbcContext가 DataSource에 의존하고 있으므로 DataSource 타입 빈을 DI 받을 수 있게 해줘야 한다.
```java
public class JdbcContext{
	private DataSource dataSource;
	
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
	
	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;
		
		try {
			c = this.dataSource.getConnection();
			
			ps = stat.makePreparedStatement(c);
			
			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			if (ps != null){
				try{
					ps.close();
				} catch (SQLException e){
				}
			}

			if (c != null){
				try{
					c.close();
				} catch (SQLException e){
				}
			}
		}
	}
}
```

```java
public class UserDao {
	private JdbcContext jdbcContext;
	
	public void setJdbcContext(JdbcContext jdbcContext) {
		this.jdbcContext = jdbcContext;
	}
	
	public void add(final User user) throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() { ... }
		);
	}
	
	public deleteAll() throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() { ... }
		);
	}
}
```

<br>

#### 빈 의존관계 변경
새롭게 작성된 오브젝트를 스프링 설정에 적용해보자. <br>
스프링의 DI는 기본적으로 인터페이스를 사이에 두고 의존 클래스를 바꿔서 사용하는 게 목적이다. 하지만 위 경우 JdbcContext는 독립적인 서비스 오브젝트로서 의미가 있을 뿐이고 구현 방법이 바뀔 가능성이 없으므로 인터페이스로 구현할 필요가 없다. <br>
test-applicationContext.xml 파일을 수정하면 다음과 같다.

```xml
<beans>
	<bean id = "userDao" class="spring...UserDao">
		<property name = "dataSource" ref="dataSource" />
		<property name = "jdbcContext" ref="jdbcContext" />
	</bean>	

	<bean id = "jdbcContext" class="spring...JdbcContext">
		<property name = "dataSource" ref="dataSource" />
	</bean>	

	<bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
		...
	</bean>
<beans>
```

아직은 userDao의 모든 메소드가 jdbcContext를 사용하는것은 아니니, 기존 방법을 사용해서 동작하는 메소드를 위해 UserDao가 아직은 dataSource를 DI받도록 하고있다.

<br>

### JdbcContext의 특별한 DI
#### 스프링 빈으로 DI
이렇게 인터페이스를 사용하지 않고 DI를 적용하는 것은 문제가 된다고 생각할 수 있다. 그러나 스프링의 DI는 넓게 보자면 객체의 생성과 관계 설정에 대한 제어권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC 개념을 포괄한다. 그런 의미에서 JdbcContext를 스프링을 이용해 UserDao 객체에서 사용하게 주입했다는 건 DI의 기본을 따르고 있다고 볼 수 있다. <br>
인터페이스를 사용하지 않았지만 JdbcContext를 UserDao와 DI 구조로 만들어야 할 이유는 아래와 같다. <br>
첫 째, JdbcContext는 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되기 때문이다. JdbcContext는 그 자체로 변경되는 상태정보를 갖고있지 않다. 내부에서 사용할 dataSource라는 인스턴스 변수는 있지만, dataSource는 읽기전용이므로 JdbcContext가 싱글톤이 되는데 아무런 문제가 없다. JdbcContext는 JDBC 컨텍스트 메소드를 제공하는 일종의 서비스 오브젝트로서 의미가 있기에 싱글톤으로 등록되어 여러 오브젝트에서 쓰이는 것이 적절하다. <br>
둘 째, JdbcContext가 DI를 통해 다른 빈에 의존하고 있기 때문이다. 이것이 중요한 이유이다. JdbcContext는 datsSource 프로퍼티를 통해 DataSource 오브젝트를 주입 받도록 되어있다. DI를 위해서는 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두 스프링 빈으로 등록해야한다. 스프링이 생성하고 관리하는 IoC대상이어야 DI에 참여할 수 있기 때문이다. 따라서 JdbcContext는 다른 빈을 DI받기 위해서라도 스프링 빈으로 등록되어야 한다.
<br>
<br>
여기서 중요한 것은 인터페이스의 사용 여부다. 인터페이스가 없다는 건 UserDao와 JdbcContext가 매우 긴밀한 관계를 가지고 강하게 결합되어 있다는 의미이다. UserDao는 항상 JdbcContext와 함께 사용되어야 하기 때문이다. 비록 클래스는 구분되어있지만 이 둘은 강한 응집도를 갖고 있다. UserDao가 JDBC방식 대신 JPA나 Hibernate로 변경된다면 JdbcContext도 통째로 바꿔야 한다. JdbcContext는 DataSource와 달리 테스트에서도 다른 구현으로 대체해서 사용할 이유가 없다. 이런 경우는 굳이 인터페이스 보다는 강력한 결합을 가진 관계를 허용하는 것도 좋다. <br>

<br>

#### 코드를 이용하는 수동 DI
