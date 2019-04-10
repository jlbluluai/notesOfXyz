参考：

https://blog.csdn.net/xiaokang123456kao/article/details/76228684



### 前言

​	使用MyBatis的都知道我们只需要写个mapper接口和mapper的xml文件，我们实际使用时就能用接口对象的方法去做操作，我们都知道接口本身不能实例化，也不具备执行方法的功能，都是需要一个实现类的，这里很自然而然的就想到MyBatis肯定是使用了动态代理的方式在程序运行时动态给我们生成了接口实现类并绑定到了接口对象上，既然知道是这样，就让我们去一探究竟吧。



### demo

​	这里我们就不用Spring的方式了，那个封装的太好了，我们直接手动去调用。

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

	/*
	 * 查询一个人
	 */
	@Test
	public void testselectOnePerson() {
		try {
			PersonMapper personMapper = session.getMapper(PersonMapper.class);
            Person person = personMapper.selectOnePerson(1);
			System.out.println(person);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

}
```

​	总的来说由SqlSessionFactoryBuilder创建了一个SqlSessionFactory的对象，然后再由这个SqlSessionFactory创建了SqlSession对象，最终由SqlSession的对象获取了代理类，我们就来慢慢看。

​	该篇只针对动态代理进行研究，所以类似SqlSessionFactory还是SqlSession的分析不再过多叙述。



### 重要类分析

​	进入getMapper方法我们会看到。

```java
  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
```

```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```

​	

#### MapperRegistry类

​	上面我们已经知道了getMapper方法最终执行就是MapperRegistry中的方法，这个类的名字也挺形象的，mapper注册。

```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //获取对应接口的代理类工厂对象
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      //通过MapperProxyFactory类的方法newInstance生成接口的代理对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```



#### MapperProxyFactory类

​	这个类就负责生成代理类了，我们来看看newInstance到底怎么执行的。

```java
  protected T newInstance(MapperProxy<T> mapperProxy) {
    //通过Proxy创建代理类对象，
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  //这是MapperRegistry中调用的newInstance
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

​	这个MapperProxy类是啥呢我贴出类的信息就明了了，这不就是JDK动态代理中的处理类吗。

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
```

​	

#### MapperProxy类

​	上面也说了这是处理类，重要的就是这个方法。

```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    /*	调用MapperMethod的execute方法对SqlSession进行封装
     *	补充说下，这个处理类的实例化对象会作为动态生成的代理类的构造函数参数，
     *	也就是说代理类中执行接口对应方法都是调用的这个invoke方法
     *	而这边的这个execute就是执行sql的关键，这就不在这里详述了，
     *	该篇主要解析MyBatis的动态代理
     */
    return mapperMethod.execute(sqlSession, args);
  }
```



### 总结

​	总得来分析下，MyBatis的动态代理就是通过JDK动态代理去实现的，关于代理相关的代码都在org.apache.ibatis.binding包下，顺便说一下，MyBatis的包结构特别清晰，很适合作为研究框架源码的案例。