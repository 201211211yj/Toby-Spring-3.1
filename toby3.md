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

