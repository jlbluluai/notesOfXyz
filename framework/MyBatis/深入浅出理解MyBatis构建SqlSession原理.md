参考：



### 前言

​	SqlSession是应用程序和数据库之间交互的一个单线程对象，阔以这么说，MyBatis能和数据库交互最终的关键点就在SqlSession。提及SqlSession就不得不提SqlSessionFactory，这是创建SqlSession的工厂。该篇就就围绕如何构建SqlSession为主题展开叙述。



### demo

​	Sqlsession要做到及时关闭，用前构建，用完关闭，不然容易资源泄漏。

```java
public class TestPerson {

	private SqlSession session;
	private static SqlSessionFactory sqlSessionFactory;

	@BeforeClass
	public static void testBeforeClass() throws Exception{
		String resource = "mybatis-config.xml";
		InputStream inputStream = Resources.getResourceAsStream(resource);
		sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
	}

	@Before
	public void testBefore() {
		try {
			session = sqlSessionFactory.openSession();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	@After
	public void testAfter() {
		session.close();
	}

}
```



### 类图分析

​	无论Sqlsession还是SqlSessionFactory都只是接口，显然不是直接用的，MyBatis也不玩虚的，接口在org.apache.ibatis.session包下，对应的实现类就在子包org.apache.ibatis.session.defaults下。（时间紧张，先拿张别人画的顶着，后面自己画个换掉）

![](https://raw.githubusercontent.com/jlbluluai/notesOfXyz/master/img/framework/mybatis/mu001.png)



### SqlSessionFactoryBuilder构建SqlSessionFactory分析

​	从demo中阔以看出，SqlSessionFactory的实例化对象是通过SqlSessionFactoryBuilder的方法build实现的，参数传入了配置文件的流。这个SqlSessionFactoryBuilder类里面是标准的运用了建造者模式（模式讲解就不在该篇进行了）。

​	根据demo的build方式我们一步步看。

第一步

```java
  public SqlSessionFactory build(InputStream inputStream) {
    return build(inputStream, null, null);
  }
```

第二步

```java
  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      //XMLConfigBuilder类是MyBatis提供的解析xml流的类
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      //传入解析完的配置继续构建
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
```

第三步

```java
  public SqlSessionFactory build(Configuration config) {
    //传入配置正式构建SqlSessionFactory
    return new DefaultSqlSessionFactory(config);
  }
```



### XMLConfigBuilder构建配置对象分析

...



### SqlSessionFactory分析

​	前文也提了SqlSessionFactory只是个接口，DefaultSqlSessionFactory是MyBastis提供的实现，意思是如若开发者想自己扩展那就自己实现一个新的即可。废话不多说，先看看接口到底提供了什么规范。

```java
public interface SqlSessionFactory {

  SqlSession openSession();

  SqlSession openSession(boolean autoCommit);
  SqlSession openSession(Connection connection);
  SqlSession openSession(TransactionIsolationLevel level);

  SqlSession openSession(ExecutorType execType);
  SqlSession openSession(ExecutorType execType, boolean autoCommit);
  SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);
  SqlSession openSession(ExecutorType execType, Connection connection);

  Configuration getConfiguration();

}
```

​	基本上就是个围绕构建SqlSession运作的工厂...

​	我们就看看不带花里胡哨的参数的openSession在默认实现类中如何实现的吧。

```java
  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }
```

​	这个openSessionFromDataSource是默认实现类DefaultSqlSessionFactory下私有的方法，所有的openSession都是调这个方法，只是参数设置不一样。

```java
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
     
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      //构建SqlSession的实例化对象
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```



### 总结

​	构建SqlSession实例化对象是由SqlSessionFactory实现，而创建SqlSessionFactory又是由SqlSessionFactoryBuilder实现，这么一步步下来的。

​	