---
title: 笔记：Spring IOC
date: 2025-03-10
categories:
  - Java
  - Spring 生态
  - Spring IOC
tags: 
author: 霸天
layout: post
---
![](image-20250704122212717.png)

不能这样做，会报错，要想在本方法用，只能 csrfTokenRepository()


![](image-20250703215302874.png)



## 两种注册方式啊


![](image-20250703103102608.png)

![](image-20250703103116668.png)


> [!NOTE] 注意事项
> 1. 如果我们是使用 `AuthenticationManager` 进行认证，它会自动将用户发送来的用户名和密码，与我们的 `CustomerUserDetailsImpl` 中返回的用户名和密码进行比对，这是我们已知的逻辑。那你可能会有疑问：它在比对前，肯定需要先用密码加密器对用户发送来的明文密码进行加密，然后再比对吧？可我并没有做任何相关配置，`AuthenticationManager` 怎么知道该使用哪个加密器？
> 2. 其实，只要你注册了一个类型为 `PasswordEncoder` 的 接口 Bean，这个 接口 Bean 有一个具体实现，`AuthenticationManager` 就会知道使用这个 `PasswordEncoder` Bean 与其具体实现，对密码进行加密，**无需我们手动配置**。
> 3. 同样的，只要你注册了一个类型为 `UserDetailsService` 的 Bean（接口 Bean），这个 接口 Bean 有一个具体的实现，`AuthenticationManager` 就会知道使用这个 `UserDetailsService` Bean 与其具体实现，去获取 `CustomerUserDetailsImpl` **无需我们手动配置**。
> 4. 上述，只限于接口 Bean 只有一个具体实现，如果有多个具体实现，那就要我们进行配置了，因为 Spring Security 虽然知道用这个 Bean，但是并不知道使用哪一个具体实现
```
/**
 * ============================================
 * Spring IoC 声明 Bean 的常用方式 1
 * --------------------------------------------
 * 概念：
 * - 同时注册了 UserDetailsService 接口类型的 Bean 和 CustomerUserDetailsImplService 实现类类型的 Bean
 * - CustomerUserDetailsImplService 是该接口的一个具体实现类，IoC 容器中可能存在多个这样的实现类 Bean
 * - 我们既可以注入 CustomerUserDetailsImplService 类 Bean，也可以注入 UserDetailsService 接口 Bean
 * - 如果注入的是 UserDetailsService，且只有一个实现类，那么调用接口方法时，实际就是调用该实现类的方法
 * - 如果存在多个实现类，则需要通过配置明确指定使用哪个实现类
 * - 简而言之，此方式支持一个接口 Bean 有多个实现类 Bean，切换实现时只需调整配置，指定使用哪一个实现即可
 * ============================================
 */
@Service
public class CustomerUserDetailsImplService implements UserDetailsService {
	......
}


/**
 * ============================================
 * Spring IoC 声明 Bean 的常用方式 2
 * --------------------------------------------
 * 概念：
 * - 仅注册了 UserDetailsService 类型的 Bean，返回的 CustomerUserDetailsImplService 实例是其具体实现类
 * - 此方式下，一个接口 Bean 只能绑定一个实现类，若要更换实现，需在此方法中直接修改返回的实例。
 * ============================================
 */
@Bean
public UserDetailsService userDetailsService() {
    return new CustomerUserDetailsImplService();
}
```


----

![](image-20250701205019198.png)

![](image-20250701205226473.png)

其实这就是 Spring IOC 的核心思想了，对吧，你的接口这些方法都已经制定好了
```
public interface PasswordEncoder {  
    String encode(CharSequence rawPassword);  
  
    boolean matches(CharSequence rawPassword, String encodedPassword);  
  
    default boolean upgradeEncoding(String encodedPassword) {  
        return false;  
    }  
}
```
然后这些实现，都会去实现你这些方法，我们将接口直接声明为 Bean，然后选择一个合适的实现类进行返回，其实这个接口呢，就可以使用这个实现类实现的方法，以后如果我们要修改实现类，只需要修改以下这个return new BCY 到其他的，然后呢，其他的代码都不用变了，如果你是直接 Bean BCY，那你如果以后像用一个其他的，那你还得把所有的BCY 找出来，一一替换


```
@Configuration
@EnableWebSecurity 
public class SecurityConfig {
    @Bean 
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // 返回合适的实现类
    }
}
```


```
@Controller  
@RequestMapping("/security")  
public class PasswordController {  
    private final PasswordEncoder passwordEncoder;  
  
    @Autowired  
    public PasswordController(PasswordEncoder passwordEncoder) {  
        this.passwordEncoder = passwordEncoder;  
    }  
      
    // 通过方法处理密码加密  
    @RequestMapping("/encode-password")  
    @ResponseBody  
    public String encodePassword() {  
        String password = "myPasswordxxxxxxx";  
        // 在方法内调用 passwordEncoder 进行密码加密  
        String encodedPassword = passwordEncoder.encode(password);  
        return "Encoded Password: " + encodedPassword;  
    }  
}
```


![](image-20250519160241335.png)

Spring 可以通过字段注入（反射）注入依赖，但这种方式**不能用在final字段上**，因为final字段一旦初始化就不能再改。Spring 反射注入时，Java编译器没法确定这个final字段被初始化过，编译期就报错。
![](image-20250519160340948.png)

![](image-20250519160346033.png)











### 一、理论 

### 0、导图：[Map：Spring IOC](../../maps/Map：SpringIoC.xmind)

---

![](image-20250518162951182.png)

![](image-20250518163131432.png)




### 1、Spring IoC

#### 1.1、IoC 概述

==1.问题：代码高耦合度==
在传统的面向对象编程中，类与类之间的依赖关系通常是直接创建的。这种直接依赖会导致代码高度耦合，使得系统的维护和测试变得困难。

尤其是在三层架构中，虽然层与层之间通过接口衔接，但直接依赖仍然存在。如果上层依赖下层，且下层发生改变时，上层也需要进行相应更改，这违背了软件开发中的开闭原则（OCP）和依赖倒转原则（DIP），例如：
``` 
// 在 Service 层中直接实例化 Dao 层的对象  
private UserDao userDao = new UserDaoImplForMySQL();  
​  
// 在 Web 层中直接实例化 Service 层的对象  
private UserService userService = new UserServiceImpl();
```


==2.IoC 解决方案==
控制反转（IoC）是一种设计模式，将对象的创建和依赖关系的管理从应用程序代码中抽离出来，交由一个外部的容器或框架进行自动管理。
``` 
// 在 IoC 容器管理下，依赖关系由容器处理
private UserDao userDao; // 不直接实例化具体实现
```

---

 
#### 1.2、IoC 的实现方式

IoC 的实现方式主要有两种：
1. <font color="#00b0f0">构造器注入</font>：通过构造函数将依赖项注入。
2. <font color="#00b0f0">Setter 注入</font>：通过 setter 方法将依赖项注入。

---


#### 1.3、Spring IoC 是什么

Spring IoC 是 Spring 框架对 IoC 设计模式的具体实现。它可以根据配置文件或注解自动创建 Bean 对象，并对其进行依赖注入，从而有效管理 Bean 对象之间的依赖关系。此外，Spring IoC 还支持为 Bean 对象的属性赋予常量值，进一步增强了其灵活性和易用性。

---


### 2、Bean

#### 2.1、Bean 概述

Bean 就是 Spring 容器所创建和管理的对象，Spring 容器负责这些对象的创建、生命周期的管理以及依赖关系的处理。

---


#### 2.2、Bean 对象的生命周期

1. ==实例化 Bean==
    - Spring 使用反射机制调用 Bean 的构造函数来创建 Bean 实例。默认使用无参构造函数，如果使用了构造注入，则使用相应的有参构造方法。

2. ==Bean 的属性赋值==
    - 在 Bean 实例化后，Spring 根据配置文件（如 XML 或注解）进行属性注入，完成依赖项的注入。

3. ==检查 Aware 接口==
    - Spring 检查 Bean 是否实现了某些特定的 Aware 接口，例如 `BeanNameAware`、`BeanClassLoaderAware` 或 `BeanFactoryAware`。若实现，则调用相应的方法完成相关依赖的设置。

4. ==Bean 后处理器 before 执行==
    - Spring 调用所有注册的 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 方法，允许对 Bean 进行自定义操作和修改。

5. ==检查 InitializingBean 接口并调用其方法==
    - 检查 Bean 是否实现了 `InitializingBean` 接口。如果实现了，Spring 调用 `afterPropertiesSet` 方法进行初始化逻辑。

6. ==初始化 Bean==
    - 如果配置中指定了初始化方法（例如通过 `@PostConstruct` 注解或 XML 配置中的 `init-method`），Spring 会在此阶段调用这些初始化方法。

7. ==Bean 后处理器 after 执行==   
    - Spring 调用所有注册的 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法，允许在 Bean 初始化后进行自定义操作和修改。

8. ==使用 Bean==
    - Bean 完成初始化后，进入就绪状态，供应用程序使用。

9. ==检查 DisposableBean 接口并调用其方法==
    - 在 Bean 被销毁之前，Spring 检查 Bean 是否实现了 `DisposableBean` 接口。如果实现了，Spring 调用 `destroy` 方法进行资源释放等清理工作。

10. ==销毁 Bean==    
    - 在 Bean 生命周期的最后阶段，如果配置了自定义的销毁方法（如通过 `@PreDestroy` 注解或 XML 配置中的 `destroy-method`），Spring 会调用这些销毁方法，完成最终的清理工作。

---


#### 2.3、Bean 对象的作用域

Bean 的作用域决定了一个 Bean 实例的实例化时机、生命周期、存储方式和共享程度。常见的作用域有以下几种：

- **Singleton（默认，单例）**：
    - **实例化时机**：Spring 容器初始化时立即创建所有的 Singleton Bean，或配置为懒加载。
        
    - **生命周期**：从 Spring 容器初始化开始，到容器关闭为止，完成所有 10 个生命周期步骤。
        
    - **存储方式**：实例存储在 Spring 容器内部的 `Map` 集合中，以 Bean 名称为键，实例为值。
        
    - **共享程度**：整个应用程序中共享同一个实例，无论有多少次对该 Bean 的请求，都会返回同一个实例。

- **Prototype（原型）**：
    - **实例化时机**：每次对 Bean 的请求都会创建一个新的实例。
        
    - **生命周期**：仅限于请求的处理期间，Spring 不管理 Prototype Bean 的销毁，需要由应用程序代码自行处理，因此只完成生命周期前九步。
        
    - **存储方式**：实例不在 Spring 容器中存储，每次请求获取的是不同的实例。
        
    - **共享程度**：每次请求都会获得不同的实例，不共享。

- **Request（请求）**（特定于 Web 应用程序）：
    
    - **实例化时机**：每次 HTTP 请求到达时创建实例。
        
    - **生命周期**：从请求到达时开始，到请求处理完成后销毁实例，完成全部 10 个生命周期步骤。
        
    - **存储方式**：实例存储在请求的上下文中。
        
    - **共享程度**：在一次 HTTP 请求内共享，跨不同请求不共享。

- **Session（会话）**（特定于 Web 应用程序）：
    
    - **实例化时机**：每次新的用户会话开始时创建实例。
        
    - **生命周期**：从用户的 HTTP 会话开始时创建实例，到会话失效时销毁实例，完成全部 10 个生命周期步骤。
        
    - **存储方式**：实例存储在会话上下文中。
        
    - **共享程度**：在同一个会话中共享，跨不同会话不共享。

- **Global session（全局会话）**（主要用于基于 Portlet 的 Web 应用）：
    
    - **实例化时机**：当 global session 开始时创建实例。
        
    - **生命周期**：与 global session 生命周期相同，从 global session 开始时创建实例，到 global session 失效时销毁实例，完成全部 10 个生命周期步骤。
        
    - **存储方式**：实例存储在 global session 上下文中。
        
    - **共享程度**：在所有 Portlet 中共享同一个实例。

---


#### 2.4、Bean 对象的实例化方式

##### 2.4.1、前言：实例化方式 概述

Bean 的实例化由 Spring 框架负责。了解 Spring 如何实例化 Bean 以及如何配置 Bean 的实例化方式是配置和优化 Spring 应用的重要部分。Bean 的实例化方式包括：
1. <font color="#00b0f0">构造方法（本质）</font>
2. <font color="#00b0f0">简单工厂模式</font>
3. <font color="#00b0f0">实例工厂模式</font>

> [!NOTE] 注意事项
> 1. 这三种实例化方法，本质上都是通过调用类的构造方法实现的

---


##### 2.4.2、构造方法

Spring 默认使用无参构造方法创建 Bean 实例，如果使用了构造注入，则使用相应的有参构造方法。

---

##### 2.4.3、简单工厂模式

==1.工厂类==
``` 
public class WeaponFactory {  
​  
	public static Weapon createWeapon(String weaponType) {  
		if (weaponType == null || weaponType.trim().isEmpty()) {  
			throw new IllegalArgumentException("Weapon type must not be null or empty");  
		}  
		  
		switch (weaponType.toLowerCase()) {  
			case "sword":  
				return new Sword();  
			case "bow":  
				return new Bow();  
			default:  
				throw new IllegalArgumentException("Unknown weapon type");  
		}  
	}  
}
```

==2.XML 配置==
``` 
<!--
声明产品 Bean
class：指向工厂类
factroy-method：指向工厂类的方法，Spring 将调用这个方法创建 Bean 实例
<constructor-arg>：为工厂类的静态方法传递参数
-->

<bean id="weapon" class="com.example.WeaponFactory" factory-method="createWeapon"> 
	<!-- 通过 <constructor-arg> 元素传递工厂方法参数 -->  
	<constructor-arg value="sword"/>  
</bean>
```

---


##### 2.4.4、实例工厂模式

==1.具体工厂角色==
``` 
public class TankFactory implements WeaponFactory {
	@Override
	public Weapon get() {
		return new Tank(); 
	} 
}
```

==2.XML 配置==
```
<!-- 声明具体工厂 Bean -->
<bean id="gunFactory" class="com.powernode.spring6.bean.GunFactory"/>  

<!-- 
声明产品 Bean
factory-bean：指向具体工厂 Bean，通过该工厂 Bean 创建目标 Bean
factory-method：指向具体工厂的方法，Spring 将调用这个方法创建 Bean 实例
-->
<bean id="gun" factory-bean="gunFactory" factory-method="get"/>
```

---


# 二、实操

### 1、基本开发步骤


#### 1.1、环境准备

---


#### 1.2、创建 Spring IoC 项目

引入 [Spring Context 依赖](https://mvnrepository.com/artifact/org.springframework/spring-context)
```
<dependencies>
    <!-- 1. Spring Context -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>xxxxx</version>
    </dependency>
</dependencies>
```

---


#### 1.3、编写 Bean 类
```
public class QianDaYe {
    // 1. 对象类型
    private Object userDao;
    private Object bankDao;
    
    // 2. 简单数组
	private int[] ages; // 一维数组
	private int[][] ages2D; // 二维数组
    
    // 3. 复杂数组
    private Object[] womens; // 一维数组
    private Object[][] womens2D;  // 二维数组
    
    // 4. 简单 List 集合
    private List<Integer> ageList;  // Integer 类型 List
    
    // 5. 复杂 List 集合
    private List<Object> womenList; 
    
    // 6. 简单 Set 集合
    private Set<Integer> ageSet;  // Integer 类型 Set
    
    // 7. 复杂 Set 集合
    private Set<Object> womenSet; 
    
	// 8. 简单 Map 集合
    private Map<String, Integer> ageMap;  // Integer 类型的 Map
    
    // 9. 复杂 Map 集合
    private Map<String, Object> phones; // Map 集合
    
    // 10. 简单数据类型
    
    // 10.1 基本数据类型
	private int age;  
	private boolean isActive; 
	private double balance;  
	private char grade; 
	private long id;  
	private float weight; 
	
	// 10.2 包装类型
    private Integer ageWrapper;  
    private Boolean isActiveWrapper;  
    private Double balanceWrapper; 
    private Character gradeWrapper;  
    private Long idWrapper; 
    private Float weightWrapper; 
    
	// 10.3 字符串类型
    private String name;  
    
    
    // 有参构造
    // 无参构造
    // Getter 方法
    // Setter 方法
    // toString 方法
    
	// 其他方法
}
```

---


#### 1.4、配置与使用 Bean

##### 1.4.1、配置概述

在 Spring IoC 中，配置 Bean 的方式有三种：

1. ==XML 配置文件 配置方式==：
	1. 在 `applicationContext.xml` 配置文件中进行声明 Bean 对象和配置属性注入
		1. 优点：同一 Bean 类可以配置多个实例
		2. 缺点：仅适用于管理对象类型的 Bean
	2. <font color="#00b0f0">构造注入</font>：
		1. 通过构造器进行依赖注入，需要在 XML 中手动配置属性注入
	3. <font color="#00b0f0">Setter 注入</font>：
		1. 通过 setter 方法进行依赖注入，同样需要在 XML 中手动配置属性注入
	4.  <font color="#00b0f0">自动装配</font>：
		1. 启用自动装配后，容器可自动匹配依赖，但无法注入基本数据类型及其集合（如简单数组、简单 List、简单 Set、简单 Map）
2. ==注解 + 扫描 配置方式（推荐）==：
	1. 结合注解（如 `@Component`、`@Resource`、`@Autowired`、`@Value` 等）和类路径扫描（XML 文件扫描或配置类扫描）实现声明 Bean 对象和配置属性注入
		1. 优点：配置简单、减少 XML 文件，自动扫描和自动装配更加方便。
		2. 缺点：
			1. 仅适用于管理对象类型的 Bean
			2. 同一 Bean 类只能声明一个实例（即同一个注解标注的对象）
	2. 在 Bean 类中声明 Bean 对象和配置属性注入
		1. 对于对象类型及其集合（如复杂数组、复杂 List 、复杂 Set 、复杂 Map ），常通过 `@Autowired`、`@Resource`、`@Qualifier` 注解进行自动装配
		2. 对于基本数据类型及其集合（如简单数组、简单 List 、简单 Set 、简单 Map ），常通过 `@Value` 注解进行手动注入（需要加载外部 `properties` 文件）
	3. 在配置类 或 XML 配置文件中扫描这些 Bean 类，使其被 Spring 容器管理。
3. ==配置类 + 注解 + 扫描 配置方式==
	1. 在 Java 配置类（例如 `ApplicationConfig`）中，通过 `@Bean` 注解声明 Bean，同时配合注解（如 `@Component`、`@Resource`、`@Autowired`、`@Value` 等）和扫描机制（配置类扫描）管理组件。
		1. 优点：
			1. 不仅可以管理对象类型的 Bean，还支持数组、List、Set、Map 等多种类型的 Bean
			2. 允许为同一 Bean 类声明多个实例
		2. 缺点：
			1. 在配置类中，需要手动配置属性注入
	2. 在 `ApplicationConfig` 配置类中，使用 `@Bean` 注解声明 Bean 并配置属性输入，同时启用组件扫描以管理其他组件类。
	3. 在其他组件类可通过 `@Autowired`、`@Resource` 或 `@Qualifier` 注解自动装配这些 Bean，实现依赖注入。

---


##### 1.4.2、配置方式1：XML 配置文件 配置方式

###### 1.4.2.1、创建 Spring IoC XML 配置文件
```
<!-- applicationContext.xml -->

<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

</beans>
```



###### 1.4.2.2、采用构造注入

==1.为 Bean 类增添有参构造方法==


==2.声明 Bean 对象，并配置该 Bean 的属性注入==
```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
       
    <!-- 1. 声明 QianDaYe Bean 对象，一个 <bean> 标签就是一个 Bean 对象 -->
    <bean 
        id="myBean"                          <!-- Bean 对象的唯一标识符，可以通过此 ID 引用该 Bean，未指定时默认为类的全类名，例如 com.example.MyBean，需要注意的是 ID 的首字母要小写 -->
        class="com.example.MyBean"           <!-- 必填项，Bean 对象的母 Bean 类的全类名，Spring 会使用这个类来创建 Bean 的实例 -->
        primary="true"                       <!-- 根据类型装配时，若有多个实现，该 Bean 为主要候选者 -->
        lazy-init="true"                     <!-- 是否开始懒加载，默认为 false，若为 true，Bean 对象将在第一次请求时被创建 -->
        scope="prototype"                    <!-- 指定 Bean 的作用域，默认为 singleton -->
        init-method="initMethod"             <!-- 初始化方法，该方法会在依赖注入完成后自动调用 -->
        destroy-method="cleanup"             <!-- 销毁方法，该方法会在 Spring 容器销毁 Bean 之前自动调用，通常用于清理资源 -->
        factory-bean="myFactoryBean"         <!-- 工厂 Bean，与 Bean 的实例化方式有关 -->
        factory-method="createInstance"      <!-- 工厂方法，与 Bean 的实例化方式有关 -->
        autowire="byName">                   <!-- 采用自动装配方式 -->

        <!-- 2. 配置该 Bean 的属性注入 -->
        
        <!-- 2.1 对象类型 -->
        <constructor-arg ref="userDao" />
        <constructor-arg ref="bankDao" />
        
        <!-- 2.2 简单数组 -->
        <!-- 2.2.1 一维数组 -->
        <constructor-arg>
            <array>
                <value>25</value> 
                <value>30</value>
                <value>35</value>
            </array>
        </constructor-arg>
        
        <!-- 2.2.2 二维数组 -->
        <constructor-arg>
            <array>
                <array>
                    <value>25</value>
                    <value>30</value>
                </array>
                <array>
                    <value>35</value>
                    <value>40</value>
                </array>
            </array>
        </constructor-arg>
        
        <!-- 2.3 复杂数组 -->
        <!-- 2.3.1 一维数组 -->
        <constructor-arg>
            <array>
                <value>bean1</value>
                <value>bean2</value>
            </array>
        </constructor-arg>
        
        <!-- 2.3.2 二维数组 -->
        <constructor-arg>
            <array>
                <array>
                    <value>Object1</value>
                    <value>Object2</value>
                </array>
                <array>
                    <value>Object3</value>
                    <value>Object4</value>
                </array>
            </array>
        </constructor-arg>
        
        <!-- 2.4 简单 List 集合 -->
        <constructor-arg>
            <list>
                <value>25</value>
                <value>30</value>
                <value>35</value>
            </list>
        </constructor-arg>
        
        <!-- 2.5 复杂 List 集合 -->
        <constructor-arg>
            <list>
                <value>Object1</value>
                <value>Object2</value>
            </list>
        </constructor-arg>
        
        <!-- 2.6 简单 Set 集合 -->
        <constructor-arg>
            <set>
                <value>25</value> 
                <value>30</value>
            </set>
        </constructor-arg>

        <!-- 2.7 复杂 Set 集合 -->
        <constructor-arg>
            <set>
                <value>Object1</value>
                <value>Object2</value>
            </set>
        </constructor-arg>
        
        <!-- 2.8 简单 Map 集合 -->
        <constructor-arg>
            <map>
                <entry key="age1" value="25"/> 
                <entry key="age2" value="30"/>
            </map>
        </constructor-arg>
        
        <!-- 2.9 复杂 Map 集合 -->
        <constructor-arg>
            <map>
                <!-- 2.9.1 值是对象 -->
                <entry key="1">
                    <bean class="AnotherBean" />
                </entry>

                <!-- 2.9.2 值是 List 集合 -->
                <entry key="2">
                    <list>
                        <value>Item1</value>
                        <value>Item2</value>
                        <value>Item3</value>
                        <ref bean="bean1" />
                        <ref bean="bean2" />
                        <ref bean="bean3" />
                    </list>
                </entry>

                <!-- 2.9.3 值是 Set 集合 -->
                <entry key="6">
                    <set>
                        <value>Value1</value>
                        <value>Value2</value>
                        <value>Value3</value>
                        <ref bean="bean1" />
                        <ref bean="bean2" />
                        <ref bean="bean3" />
                    </set>
                </entry>
            </map>
        </constructor-arg>
        
        <!-- 2.10 注入简单数据类型 -->
        <!-- 2.10.1 基本数据类型 -->
        <constructor-arg value="25" />  <!-- int 类型 -->
        <constructor-arg value="true" />  <!-- boolean 类型 -->
        <constructor-arg value="99.99" />  <!-- double 类型 -->
        <constructor-arg value="A" />  <!-- char 类型 -->
        <constructor-arg value="1000L" />  <!-- long 类型 -->
        <constructor-arg value="3.14" />  <!-- float 类型 -->

        <!-- 2.10.2 包装类型 -->
        <constructor-arg value="30" />  <!-- Integer 类型 -->
        <constructor-arg value="false" />  <!-- Boolean 类型 -->
        <constructor-arg value="200.50" />  <!-- Double 类型 -->
        <constructor-arg value="B" />  <!-- Character 类型 -->
        <constructor-arg value="5000" />  <!-- Long 类型 -->
        <constructor-arg value="2.71" />  <!-- Float 类型 -->

        <!-- 2.10.3 字符串类型 -->
        <constructor-arg value="John Doe" />  <!-- String 类型 -->
    </bean>

    <!-- 声明其他 Bean 对象 -->
    <bean class="com.example.bean.QianDaYe" id="qianDaYe" />
    
</beans>
```

---


###### 1.4.2.3、采用 Setter 注入

==1.为 Bean 类增添 Setter 方法==


==2.声明 Bean 对象，并配置该对象的属性注入==
要将构造注入改为 setter 注入，你需要使用 `<property>` 标签来替代 `<constructor-arg>` 标签
```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- 1. 声明 QianDaYe Bean 对象 -->
    <bean 
        id="myBean" 
        class="com.example.MyBean" 
        lazy-init="true" 
        scope="prototype" 
        init-method="initMethod" 
        destroy-method="cleanup" 
        factory-bean="myFactoryBean" 
        factory-method="createInstance" 
        autowire="byName">

        <!-- 2. 配置 Bean 对象的依赖关系 -->
        
        <!-- 2.1 对象类型 -->
        <property name="userDao" ref="userDao" />
        <property name="bankDao" ref="bankDao" />
        
        <!-- 2.2 简单数组 -->
        <property name="simpleArray">
            <array>
                <value>25</value> 
                <value>30</value>
                <value>35</value>
            </array>
        </property>
        
        <!-- 2.3 二维数组 -->
        <property name="twoDimArray">
            <array>
                <array>
                    <value>25</value>
                    <value>30</value>
                </array>
                <array>
                    <value>35</value>
                    <value>40</value>
                </array>
            </array>
        </property>
        
        <!-- 2.4 简单 List 集合 -->
        <property name="simpleList">
            <list>
                <value>25</value>
                <value>30</value>
                <value>35</value>
            </list>
        </property>
        
        <!-- 2.5 复杂 List 集合 -->
        <property name="complexList">
            <list>
                <value>Object1</value>
                <value>Object2</value>
            </list>
        </property>
        
        <!-- 2.6 简单 Set 集合 -->
        <property name="simpleSet">
            <set>
                <value>25</value>
                <value>30</value>
            </set>
        </property>

        <!-- 2.7 复杂 Set 集合 -->
        <property name="complexSet">
            <set>
                <value>Object1</value>
                <value>Object2</value>
            </set>
        </property>
        
        <!-- 2.8 简单 Map 集合 -->
        <property name="simpleMap">
            <map>
                <entry key="age1" value="25"/>
                <entry key="age2" value="30"/>
            </map>
        </property>
        
        <!-- 2.9 复杂 Map 集合 -->
        <property name="complexMap">
            <map>
                <!-- 2.9.1 值是对象 -->
                <entry key="1">
                    <bean class="AnotherBean" />
                </entry>

                <!-- 2.9.2 值是 List 集合 -->
                <entry key="2">
                    <list>
                        <value>Item1</value>
                        <value>Item2</value>
                        <value>Item3</value>
                        <ref bean="bean1" />
                        <ref bean="bean2" />
                        <ref bean="bean3" />
                    </list>
                </entry>

                <!-- 2.9.3 值是 Set 集合 -->
                <entry key="6">
                    <set>
                        <value>Value1</value>
                        <value>Value2</value>
                        <value>Value3</value>
                        <ref bean="bean1" />
                        <ref bean="bean2" />
                        <ref bean="bean3" />
                    </set>
                </entry>
            </map>
        </property>
        
        <!-- 2.10 注入简单数据类型 -->
        <!-- 2.10.1 基本数据类型 -->
        <property name="simpleInt" value="25" /> <!-- int 类型 -->
        <property name="simpleBoolean" value="true" /> <!-- boolean 类型 -->
        <property name="simpleDouble" value="99.99" /> <!-- double 类型 -->
        <property name="simpleChar" value="A" /> <!-- char 类型 -->
        <property name="simpleLong" value="1000L" /> <!-- long 类型 -->
        <property name="simpleFloat" value="3.14" /> <!-- float 类型 -->

        <!-- 2.10.2 包装类型 -->
        <property name="wrapperInteger" value="30" /> <!-- Integer 类型 -->
        <property name="wrapperBoolean" value="false" /> <!-- Boolean 类型 -->
        <property name="wrapperDouble" value="200.50" /> <!-- Double 类型 -->
        <property name="wrapperCharacter" value="B" /> <!-- Character 类型 -->
        <property name="wrapperLong" value="5000" /> <!-- Long 类型 -->
        <property name="wrapperFloat" value="2.71" /> <!-- Float 类型 -->

        <!-- 2.10.3 字符串类型 -->
        <property name="stringValue" value="John Doe" /> <!-- String 类型 -->
    </bean>

    <!-- 声明其他 Bean 对象 -->
    <bean class="com.example.bean.QianDaYe" id="qianDaYe" />
    
</beans>
```

---


###### 1.4.2.4、采用自动装配

==1.为 Bean 类增添有参构造方法和 Setter 方法==


==2.声明 Bean 对象，并配置该对象的属性注入==
Spring 提供了五种主要的自动装配模式，这些模式包括：`no`（默认）、`byName`、`byType`、`constructor`、`autodetect`。
``` 
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="foo" class="com.example.Foo" autowire="autodetect"/>
	
</beans>
```

1. <font color="#00b0f0">no（默认）</font>
默认情况下，Spring 不会进行自动装配，需要我们手动注入依赖，例如通过构造方法注入或 Setter 注入

2. <font color="#00b0f0">byName</font>
使用这种方式，Spring 会查找系统中与**属性名相同**的组件，自动进行注入。这种方式基于 Setter 注入，所以要为属性提供 Setter 方法
``` 
public class Foo {
    private Bar bar;  // 看 bar

    // Setter
    public void setBar(Bar bar) {
        this.bar = bar;
    }
}
```

3. <font color="#00b0f0">byType</font>
使用这种方式，Spring 会查找系统中与属性**类型相同**的组件，自动进行注入。这种方式基于 Setter 注入，所以要为属性提供 Setter 方法

``` 
public class Foo {
    private Bar bar;  // 看 Bar

    // Setter
    public void setBar(Bar bar) {
        this.bar = bar;
    }
}
```

> [!NOTE] 注意：
> 1. 如果系统中存在多个属性类型匹配的组件，Spring 会无法决定注入那个组件，就会抛出 `NoUniqueBeanDefinitonException` 异常。
> 2. 通过标记主要候选者可以解决此问题，例如：
> `<bean id="primaryBar" class="com.example.Bar" primary="true"/>` 或在 Bean 类上使用 `@Primary` 注解。

4. <font color="#00b0f0">constructor</font>
使用这种方式，Spring 会自动查找系统中与构造函数参数**类型匹配**的组件，自动进行注入。这种方式基于构造方法注入，所以要为属性提供构造方法
``` 
public class Foo {
    private Bar bar;

    // 构造方法
    public Foo(Bar bar) { // 看这里的 Bar
        this.bar = bar;
    }
}
```

> [!NOTE] 注意
> 如果有多个构造函数，Spring 会选择参数最多的那个（前提是所有参数类型都能在容器中找到对应的 bean）。

5. <font color="#00b0f0">autodetect</font>
使用这种方式，Spring 会首先尝试通过 constructor 方式进行注入，如果失败，则使用 byType 方式。这种方式基于 Setter 注入和 构造注入，所以既要为属性提供 Setter 方法又要为属性提供构造方法
``` 
public class Foo {
    private Bar bar;

    // 构造方法
    public Foo() {
        // Default constructor
    }

    public Foo(Bar bar) {
        this.bar = bar;
    }
	
    // Setter
    public void setBar(Bar bar) {
        this.bar = bar;
    }
}
```


---


##### 1.4.3、配置方式2：注解 + 扫描 配置方式

###### 1.4.3.1、声明 Bean 对象

在类上标注 `@Component` 及其衍生注解（如 `@Service`, `@Repository`, `@Controller`），以指示该类是一个 Bean类并声明唯一个Bean 对象。
- `@Component`：标注通用组件
- `@Service`：标注业务逻辑层（service 层）
- `@Repository`：标注数据访问层（dao 层、mapper 层）
- `@Controller`：标注表现层（Web 层）
- `@RestController`：`@RestController` = `@Controller` + `@ResponseBody` ，使得每个方法的返回值都直接作为 HTTP 响应体返回。
```
@Component("myBean1") // 必填项，标注 Bean 类和指定 ID，若未指定 ID，默认为类名的首字母小写形式（不是全类名）
@Scope("prototype")   // 指定作用域，默认为单例(singleton)
@Lazy                 // 延迟加载，默认不开启，设置为 true 则会在第一次访问时创建 Bean
@DependsOn("myBean2") // 指定依赖的其他 Bean，确保 myBean2 初始化完成后再实例化当前 Bean
@Primary              // 根据类型装配时，若有多个实现，该 Bean 为主要候选者
public class MyBean {

    // @PostConstruct：初始化方法，依赖注入完成后自动调用
    @PostConstruct
    public void initMethod() {
        System.out.println("MyBean initialized with all dependencies.");
        // 可以在这里放置资源初始化逻辑
    }

    // @PreDestroy：销毁方法，在 Bean 被容器销毁前调用
    @PreDestroy
    public void cleanup() {
        System.out.println("MyBean is being destroyed. Cleaning up resources.");
        // 可以在这里执行清理操作
    }
    
}
```

---


###### 1.4.3.2、配置属性注入
```
@Component
public class MyBean {

    // @Autowired：实现按类型自动装配（byType）
    @Autowired
    private MyRepository myRepository;
    
    // @Autowired + @Qualifier：实现按名称自动装配（byName）
	@Autowired
    @Qualifier("specificRepository") // 指定注入名称
    private MyRepository specificRepository;
    
    // @Resource：先按名称装配（byName），再按类型装配（byType）
	@Resource(name = "specificService") 
    private MyService myService;
    
    // 1. 对象类型
    @Autowired
    private Object userDao;
    
    @Autowired
    private Object bankDao;
    
    // 2. 简单数组
    @Value("${ages}")
    private int[] ages; // 一维数组

    @Value("${ages2D}")
    private int[][] ages2D; // 二维数组
    
    // 3. 复杂数组
    @Autowired
    private Object[] womens; // 一维数组
    
    @Value("${womens2D}")  
    private Object[][] womens2D;  // 二维数组
    
    // 4. 简单 List 集合
    @Value("${ageList}")
    private List<Integer> ageList;
    
    // 5. 复杂 List 集合
    @Autowired
    private List<Object> womenList; 
    
    // 6. 简单 Set 集合
    @Value("${ageSet}")
    private Set<Integer> ageSet; 
    
    // 7. 复杂 Set 集合
    @Autowired
    private Set<ComplexObject> womenSet; 
    
	// 8. 简单 Map 集合
    @Value("${ageMap}")
    private Map<String, Integer> ageMap; 
    
    // 9. 复杂 Map 集合
	@Value("${phones}")
    private Map<String, Object> phones; 
    
	// 10. 简单数据类型
    
    // 10.1 基本数据类型
    @Value("${age}")  
    private int age;  
    
    @Value("${isActive}") 
    private boolean isActive; 
    
    @Value("${balance}")  
    private double balance;  
    
    @Value("${grade}")  
    private char grade; 
    
    @Value("${id}")  
    private long id;  
    
    @Value("${weight}")  
    private float weight; 
    
    // 10.2 包装类型
    @Value("${ageWrapper}")  
    private Integer ageWrapper;  
    
    @Value("${isActiveWrapper}")  
    private Boolean isActiveWrapper;  
    
    @Value("${balanceWrapper}") 
    private Double balanceWrapper; 
    
    @Value("${gradeWrapper}")  
    private Character gradeWrapper;  
    
    @Value("${idWrapper}") 
    private Long idWrapper; 
    
    @Value("${weightWrapper}")  
    private Float weightWrapper; 
    
    // 10.3 字符串类型
    @Value("${name}")  
    private String name;  
}
```

---


###### 1.4.3.3、配置外部 properties 文件

如果需要为基本数据类型及其集合进行注入，通常会通过配置外部 `properties` 文件，并将相应的属性值写入该文件中。
```
# 1. 对象类型
userDao=com.example.UserDao
bankDao=com.example.BankDao

# 2. 简单数组
ages=25,30,35 
ages2D=25,30|35,40 //{25,30} {35,40}

# 3. 复杂数组
womens=Jane,Alice,Mary,Emily
womens2D=Jane,Alice|Mary,Emily

# 4. 简单 List 集合
ageList=25,30,35

# 5. 复杂 List 集合
womenList=Jane,Alice,Mary,Emily

# 6. 简单 Set 集合
ageSet=25,30,35

# 7. 复杂 Set 集合
womenSet=Jane,Alice,Mary,Emily

# 8. 简单 Map 集合
ageMap=age1:25,age2:30,age3:35

# 9. 复杂 Map 集合
phones=home:1234567890,office:0987654321

# 10. 简单数据类型

#10.1 基本数据类型
age=25
isActive=true
balance=99.99
grade=A
id=12345
weight=70.5

# 10.2 包装类型
ageWrapper=30
isActiveWrapper=false
balanceWrapper=200.75
gradeWrapper=B
idWrapper=67890
weightWrapper=80.5

# 10.3 字符串类型
name=John Doe
```

---


###### 1.4.3.4、扫描 Bean 对象 + 加载外部 properties 文件

==1.XML 配置文件 扫描方式==
``` 
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd">
	
	<!-- 加载外部 properties 文件 -->
	<context:property-placeholder location="classpath:application.properties,classpath:myconfig-config.properties" />
	
	<!-- 扫描包 -->
    <context:component-scan base-package="com.example" />
</beans>
```

==2.配置类 扫描方式==
``` j
@Configuration
@ComponentScan(basePackages = "com.example") // 扫描包
@PropertySource({"classpath:application.properties", "classpath:myconfig.properties"}) // 加载外部 properties 文件
public class ApplicationContextConfig {
	......
}
```

> [!NOTE] 注意事项：关于 properties 文件
> 1. 如果有多个 `properties` 文件，要确保文件加载的顺序，因为 Spring 会按顺序加载文件，后加载的文件会覆盖前面文件中相同的属性值。例如，`application.properties` 文件中的某个配置值如果在 `myconfig.properties` 中被重新定义，后者的值会覆盖前者。

---


##### 1.4.4、配置方式3：配置类 +注解 + 扫描 配置方式

###### 1.4.4.1、创建 Spring IOC 配置类

创建配置类 `ApplicationContextConfig`，并使用 `@Configuration` 标注此类为一个配置类。
```
// ApplicationContextConfig.java

@Configuration // 表名这是个配置类
public class ApplicationContextConfig {
	......
}
```

---


###### 1.4.4.2、声明 Bean 对象，并配置属性注入 + 组件扫描
```
@Configuration 
public class ApplicationContextConfig {

	@Bean（                 // 必填项，表示返回的结果是一个 Bean
    name="myService",       // 指定 Bean 的id，可省略，默认首字母小写
    initMehtod="init",      // 初始化方法，该方法会在依赖注入完成后自动调用
    destoryMethod="cleanup" // 销毁方法，该方法会销毁 Bean 之前自动调用
    ）
    @ComponentScan(basePackages = "com.example") // 扫描包
	@Lazy                   // 懒加载，默认不开启
    @Scope("prototype")     // 指定作用域
    @DependsOn("myRepository") // 指定初始化顺序，在 myRepository 之后进行实例化
    @Primart                // 根据类型装配时，若有多个实现，该 Bean 为主要候选者
    public MyService myService() {
	    // 手动使用构造方法 new Bean 实例
	    // 配置该实例的属性注入
	    // 最后返回该实例
    }
    
	// 1. 声明 Bean 对象
	@Bean
    public QianDaYe qianDaYe() {
	    // 2. 手动使用构造方法 new Bean 实例
        QianDaYe qianDaYe = new QianDaYe();
        // 3. 配置该实例的属性注入
        qianDaYe.setUserDao(userDao());
        qianDaYe.setBankDao(bankDao());
        qianDaYe.setWomens(womens());
        qianDaYe.setWomens2D(womens2D());
        qianDaYe.setWomenList(womenList());
        qianDaYe.setWomenSet(womenSet());
        qianDaYe.setPhones(phones());
        // 4. 最后返回该实例
        return qianDaYe;
    }
    
    // 2. 声明复杂数组 Bean
    // 2.1 声明复杂一维数组 Bean
    @Bean
    public Object[] womens() {
        return new Object[]{
	        bean1(), bean2(), bean3()
        };
	}
	// 2.2 声明复杂二维数组 Bean
    @Bean
    public Object[][] womens2D() {
        return new Object[][]{
            {bean1(), bean2()},
            {bean3(), bean4()}
        };
    }

    // 3. 声明复杂 List 类型 Bean
	@Bean
	public List<Object> womenList() {
	    List<Object> womenList = new ArrayList<>(); 
	    womenList.add(bean1()); 
	    womenList.add(bean2());
	    womenList.add(bean3());
	    return womenList;
	}

    // 4. 声明复杂 Set 类型 Bean
    @Bean
    public Set<Object> womenSet() {
        Set<Object> womenSet = new HashSet<>();
        womenSet.add(bean1());
        womenSet.add(bean2());
        womenSet.add(bean3());
        return womenSet;
    }

    // 5. 声明复杂 Map 类型 Bean
    @Bean
    public Map<String, Object> phones() {
        Map<String, Object> phones = new HashMap<>();
        // 若值是对象
        phones.put("home", anotherBean());
        // 若值是 List 集合
        phones.put("work", Arrays.asList("Item1", "Item2", "Item3"));
        // 若值是 Set 集合
        phones.put("mobileSet", new HashSet<>(Arrays.asList("Value1", "Value2", "Value3")));
        return phones;
    }
}
```


> [!NOTE] 注意事项
> 1. 在使用配置类声明 Bean 时，如果需要传入参数，Spring 会根据类型（byType）自动进行注入。如果存在多个实现，它会选择标记为主要候选者的 Bean。例如，下面的示例会自动注入 `AuthenticationConfiguration` 类型的 Bean：
```
@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
    return configuration.getAuthenticationManager();
}
```

---


###### 1.4.4.3、组件类中注入这些 Bean 对象

在组件类中可通过 `@Autowired`、`@Resource` 或 `@Qualifier` 注解自动装配这些 Bean，实现依赖注入。
```
@Component
public class MyComponent {

    @Autowired
    private AuthenticationManager authenticationManager; // 无需再次传参
    
	// 使用注入的 Bean 进行操作
    public void authenticateUser() {
        System.out.println("AuthenticationManager injected: " + authenticationManager);
    }
}
```

---


### 2、Boot 简化开发

#### 2.1、自动扫描 Bean 类

在 `src/main/java/com/example` 目录下创建一个名为 `Application` 的主类，并为其添加 `@SpringBootApplication` 注解。这个注解标识该类为 Spring Boot 应用的入口点，同时它还会自动扫描当前包及其子包中的组件，无需在配置文件中手动配置扫描路径。
```
/**
* @SpringBootApplication 是组合注解，包含
*    1. @SpringBootConfigutation：标识为配置类
*    2. @EnableAutoConfiguration：启用自动配置
*    3. @ComponentScan：扫描当前包及子包的组件
*/

@SpringBootApplication
public class SpringBootDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootDemoApplication.class, args);
    }
}
```

---


#### 2.2、属性绑定

在 Spring Boot 中，无需在不同文件中重复使用 `@Value` 进行注入。Spring Boot 提供了属性绑定功能，可以将配置文件（`application.properties` 或 `application.yaml`）中的属性值一键绑定到组件类的属性上，从而简化了配置和注入过程，提高了开发效率。

==1.组件类==
```
@Component    // 注入类声明为组件是必须的
@ConfigurationProperties(prefix = "app")    // Boot 属性绑定
public class AppProperties {
	// 简单类型
    private String name;  // 简单类型
    private String emptyString;  // 空字符串
    private String nullValue;  // null 值
    private String specialChar;  // 特殊字符

	// 简单数组 
	private String[] names;  // 一维数组
	private int[][] coordinates;  // 二维数组

	// 简单 List 集合
	private List<String> users;

	// 简单 Set 集合
	private Set<String> roles;

	// 简单 Map 集合
	private Map<String,List<String>> mapp
} 
```


==2.配置文件：application.properties==
```
# 简单类型
app.name=MyApplication

# 空字符串
app.emptyString=

# null 值
app.nullValue=null

# 特殊字符(直接写特殊字符就好)
app.specialChar=2<3

# 一维数组
app.names=Alice,Bob,Charlie

# 二维数组
app.coordinates=1,2;3,4;5,6

# 简单 List 集合
app.users=Alice,Bob,Charlie

# 简单 Set 集合
app.roles=admin,user,guest

# 简单 Map 集合
app.mapp.users=Alice,Bob,Charlie   # users 是键，后面的是值
app.mapp.roles=admin,user,guest    # roles 是键，后面的是值
```


==3.配置文件：application.yml==
```
app:
  name: MyApplication           # 简单类型
  emptyString: ""               # 空字符串
  nullValue: null               # null 值
  specialChar: "2<3"            # 特殊字符，建议使用引号

  names:                        # 一维数组
    - Alice
    - Bob
    - Charlie

  coordinates:                  # 二维数组
    - - 1
      - 2
    - - 3
      - 4
    - - 5
      - 6

  users:                        # 简单 List 集合
    - Alice
    - Bob
    - Charlie

  roles:                        # 简单 Set 集合
    - admin
    - user
    - guest

  settings:                     # 简单 Map 集合，值为 List
    users:                      # 键：users，值为数组
      - Alice
      - Bob
      - Charlie
    roles:                      # 键：roles，值为数组
      - admin
      - user
      - guest
```

---


#### 2.3、唯一构造机制

在 Spring Boot 开发中，如果 Bean 类只有一个构造函数，则无需额外使用 `@Autowired` 等注解，Spring Boot 会自动根据构造函数中的参数类型（byType）进行装配：
```
@Component
public class MyService {

    private final MyRepository myRepository;

    // 只有一个构造函数，Spring 会自动根据参数类型注入 MyRepository
    public MyService(MyRepository myRepository) {
        this.myRepository = myRepository;
    }

    public void performAction() {
        myRepository.save();
    }
}
```

---


