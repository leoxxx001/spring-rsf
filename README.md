#Spring RSF RPC Framework


Spring RSF 是基于 RESTEasy & Spring 构建的 RESTful 风格、轻量级 RPC 框架。


### Introduction

采用 Spring AOP + RESTEasy 实现。基于 HTTP RESTFul 风格，JSON & XML 数据协议格式转输。代码基于注解零侵入性，对于调用者无需了解具体协议及URL和请求参数，完全基于远程对象接口方式进行数据交互。

客户端对多 Server 提供负载均衡机制，可轻松实现自定义的路由策略(目前支持随机和轮询)。当集群远程服务器不可用时支持重试机制，动态监测和管理 Server 列表。能实时从远程获取及更新 Server 列表。

**你可以通过下载 [示例代码](https://github.com/denger/spring-rsf/tree/master/samples/HelloSpringRSF) 或阅读下面的说明来快速了解该框架的使用。**


## Quick Start


#### 服务提供者

__1. 加入工程依赖__

```xml
<dependency>
	<groupId>org.springstack.rsf</groupId>
	<artifactId>spring-rsf</artifactId>
	<version>1.0.1</version>
<dependency>
```
*在 Server 端和 Clinet 的工程都需添加该依赖。*


__2.添加至 Server 端的 web.xml__

```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>classpath:applicationContext.xml</param-value>
</context-param>
<listener>
	<listener-class>
		org.jboss.resteasy.plugins.server.servlet.ResteasyBootstrap
	</listener-class>
</listener>

<listener>
	<listener-class>
		org.jboss.resteasy.plugins.spring.SpringContextLoaderListener
	</listener-class>
</listener>

<servlet>
	<servlet-name>resteasy-servlet</servlet-name>
	<servlet-class>
		org.jboss.resteasy.plugins.server.servlet.HttpServletDispatcher
	</servlet-class>
</servlet>

<!-- 指定是 API 的根路径 -->
<servlet-mapping>
	<servlet-name>resteasy-servlet</servlet-name>
	<url-pattern>/*</url-pattern>
</servlet-mapping>
```

__3.编写业务服务接口__


```java

@Path("/product")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface ProductService {

	@GET
	@Path("/{id}")
	public Product get(@PathParam("id") Long id);

	@DELETE
	@Path("/{id}")
	public boolean delete(@PathParam("id") Long id);

	@PUT
	public Product create(Product product);

	@GET
	public List<Product> findByCategory(@QueryParam("category") String category);
}
```

*该接口一般以 jar 包形式发布给调用者。*

__4. 编写业务服务实现及 @Bean 声明__


```java
public class ProductServiceImpl implements ProductService{

	public Product get(Long id) {
		// do somthing
	}
	// ...代码略...
}
```

添加实现至 Spring 配置 `applicationContext.xml`，当然也可以注解(@Service)的方式声明该服务实现。
```xml
<bean class="org.springstack.rsf.example.service.ProductServiceImpl">
</bean>
```



#### 服务调用者

__1. 添加依赖__

通常对于服务提供者会将接口对外暴漏且以 jar 包的方式提供给服务的调用者，所以对于调用者来说需要引入服务API接口(e.g: product-api.jar) 及 spring-rsf 的包依赖。


__2. 将服务接口注入__

将如下配置添加至 Spring 配置文件 `applicationContext-rpc-test.xml`，将 `PorudctService` 通过 Spring 以 Bean 方式注入。
 
```xml
<bean id="productService" class="org.springstack.rsf.proxy.RSFClientFactoryBean">
    <!-- 指定业务服务接口 -->
    <property name="serviceInterface" value="org.springstack.rsf.example.rpc.ProductService" />
    
    <!-- HTTP Clinet 参数设置-->
    <property name="connectionTimeout" value="5000" />
    <property name="readTimeout" value="5000" />
    <property name="maxTotalConnections" value="30" />
    <property name="defaultMaxConnectionsPerHost" value="20" />
    
    <!-- 重试次数 -->
    <property name="retry" value="1" />
    
    <!-- 服务列表(ip:port/context-root) -->
    <property name="serverList">
    	<list>
        	<value>127.0.0.1:8080/HelloSpringRSF</value>
        </list>
    </property>
</bean>
```

__3.调用产品服务__

将该 `PoructService` 注入到如下 Test 中，即可完成对该接口所有服务的调用。


```java
@ContextConfiguration(locations = "classpath:applicationContext-rpc-test.xml")
@RunWith(SpringJUnit4ClassRunner.class)
public class ProductServiceTest {

	@Autowired
	private ProductService productService;

	@Test
	public void testInvokeRpcMethod() {
		Product product = productService.get(1L);
		assertNotNull(product);
		assertEquals(product.getId(), new Long(1L));

		assertTrue(productService.delete(1L));

		List<Product> products = productService.findByCategory("all");
		assertTrue(products.size() > 0);
	}
}

```

[查看完整示例代码](https://github.com/denger/spring-rsf/tree/master/samples/HelloSpringRSF)




