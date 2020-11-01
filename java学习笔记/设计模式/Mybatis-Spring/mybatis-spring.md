**MyBatis-spring**

## 1. 对象怎么注入spring容器？

@Bean

```java
ApplicationContext context=new AnnotationConfigApplicationContext(AppConfig.class);

//AppConfig.class为数据源

//AccountDao accountDao=context.getBean("dataSource");

//System.out.println(accountDao.query());
AccountDao accountDao=(AccountDao)Proxy.newProxyInstance(getClass().getClassLoader(),
 new Class[]{AccountDao.class},new MyInvocationHandler());

accountDao.query();
//怎么将accountDao交给spring管理
//@Comopnet +@CompnentScan
//xml
//怎么普通类交给spring管理？
//@Import，@FactoryBean，

```

@MapperScan 生成代理@Proxy对象 ，spring生成并且交给Spring IOC管理

```java
public interface AccountDao{
    @Select("select * from account")
    public List<Account>query();
}
```



```java
public class MyInvocationHandler implements InvocationHandler{

	@Override
	public Object invoke(Object proxy,Method method,Object[]args)throws Throwable{
        
        //sql
        System.out.println(method.getAnnotation(Select.class).value());
    }
}
```







