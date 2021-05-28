## AOP源码解析
### xml文件
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns="http://www.springframework.org/schema/beans"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd"
	   default-autowire="byName">

	<!-- aspectj end-->
	<bean id="userService" class="com.ztz.aop.UserServiceImpl"></bean>

	<!-- 定义前置通知,com.zhoujunwen.BeforeLogAdvice实现了org.springframework.aop.MethodBeforeAdvice -->
	<bean id="beforeLogAdvice" class="com.ztz.aop.BeforeLogAdvice"></bean>

    <!-- 通过userProxy方式获取代理，begin-->
	<!-- 定义代理类，名 称为userProxy，将通过userProxy访问业务类中的方法 -->
	<bean id="userProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
		<property name="proxyInterfaces">
			<value>com.ztz.aop.UserService</value>
		</property>
		<property name="interceptorNames">
			<list>
				<value>beforeLogAdvice</value>
			</list>
		</property>
		<property name="target" ref="userService"></property>
	</bean>
	<!-- 通过userProxy方式获取代理，end-->

<!-- BeanNameAutoProxyCreator方式ebegin，可以直接通过userServiceid获取代理对象-->
<!--	<bean id="myServiceAutoProxyCreator" class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">-->
<!--		<property name="interceptorNames">-->
<!--			<list>-->
<!--				<value>beforeLogAdvice</value>-->
<!--			</list>-->
<!--		</property>-->
<!--		<property name="beanNames">-->
<!--			<value>*Service</value>-->
<!--		</property>-->
<!--	</bean>-->
	<!-- BeanNameAutoProxyCreator方式end，可以直接通过userServiceid获取代理对象-->
</beans>

```

### ProxyFactoryBean方式
#### 源码分析

- org.springframework.aop.framework.ProxyFactoryBean.getObject
- 经过doGetBean走到这里，因为普通实例返回该实例，如果是FactoryBean，返回FactoryBean创建的实例
```java
public Object getObject() throws BeansException {
	initializeAdvisorChain();
	if (isSingleton()) {
        // 往下走
		return getSingletonInstance();
	}
	else {
		if (this.targetName == null) {
			logger.info("Using non-singleton proxies with singleton targets is often undesirable. " +
					"Enable prototype proxies by setting the 'targetName' property.");
		}
		return newPrototypeInstance();
	}
}
```

- org.springframework.aop.framework.ProxyFactoryBean.getSingletonInstance
```java
	private synchronized Object getSingletonInstance() {
		if (this.singletonInstance == null) {
			this.targetSource = freshTargetSource();
			if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
				// Rely on AOP infrastructure to tell us what interfaces to proxy.
				Class<?> targetClass = getTargetClass();
				if (targetClass == null) {
					throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
				}
				setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
			}
			// Initialize the shared singleton instance.
			super.setFrozen(this.freezeProxy);
            // 获取代理，先看createAopProxy，创建代理类：JDK/cglib
			this.singletonInstance = getProxy(createAopProxy());
		}
		return this.singletonInstance;
	}
```

- org.springframework.aop.framework.DefaultAopProxyFactory.createAopProxy
```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	// isOptimize() :是否使用优化的代理策略(ThreadLocal线程变量 线程隔离)，默认就是false，可以通过expose-proxy="true"改变
	// isProxyTargetClass()：是否直接代理目标类以及任何接口。默认false，可以通过proxy-target-class="true"改变
	// hasNoUserSuppliedProxyInterfaces :确定提供的切面类是否只有SpringProxy指定的接口，或者根本没有指定代理接口
	if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
		Class<?> targetClass = config.getTargetClass();
		if (targetClass == null) {
			throw new AopConfigException("TargetSource cannot determine target class: " +
					"Either an interface or a target is required for proxy creation.");
		}
		if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
			// JDK分支
			return new JdkDynamicAopProxy(config);
		}
		// CGLIB分支
		return new ObjenesisCglibAopProxy(config);
	}
	else {
		// 默认走JDK分支
		return new JdkDynamicAopProxy(config);
	}
}
```

- org.springframework.aop.framework.JdkDynamicAopProxy.getProxy(java.lang.ClassLoader) JDK动态代理，返回代理对象
```java
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isTraceEnabled()) {
			logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
		}
		// 得到目标类接口，SpringProxy,Advised,DecoratingProxy,都要进行反向代理
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		// 排除equals,hashCode等基类方法的增强，如果有这种增强，直接结束
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		// 进入JDK动态代理方法,这个就不说了，你不可再改了
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
```

### BeanNameAutoProxyCreator方式
#### 源码分析

- 从initializeBean方法走到 org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator.postProcessAfterInitialization
```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
	// 对象实例化、属性注入完成后
	// 在initializeBean.applyBeanPostProcessorsAfterInitialization方法中触发
	if (bean != null) {
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		if (this.earlyProxyReferences.remove(cacheKey) != bean) {
			// 生成动态代理对象并且包装起来
			return wrapIfNecessary(bean, beanName, cacheKey);
		}
	}
	return bean;
}
```

- org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator.wrapIfNecessary
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
		return bean;
	}
	if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
		return bean;
	}
	if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

	// Create proxy if we have advice.
	// 意思就是如果该类有advice（增强,还有可能有多个）则创建proxy，这个里面还对所有增强进行了排序
	Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	if (specificInterceptors != DO_NOT_PROXY) {
		this.advisedBeans.put(cacheKey, Boolean.TRUE);
		// 将普通的Bean替换成代理Bean
		Object proxy = createProxy(
				bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
		this.proxyTypes.put(cacheKey, proxy.getClass());
		return proxy;
	}

	this.advisedBeans.put(cacheKey, Boolean.FALSE);
	return bean;
}

```

-org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator.createProxy
```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
		@Nullable Object[] specificInterceptors, TargetSource targetSource) {

	// 保存代理类真实的类型，代理对象执行反射调用的时候还得用
	// public static final String ORIGINAL_TARGET_CLASS_ATTRIBUTE =
	//			Conventions.getQualifiedAttributeName(AutoProxyUtils.class, "originalTargetClass");
	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
	}

	// 1.创建ProxyFactory，proxy对象的生产主要就是在ProxyFactory做的（FactoryBean）
	ProxyFactory proxyFactory = new ProxyFactory();
	proxyFactory.copyFrom(this);

	// 判断有没有设置proxy-target-class="true" ,默认为false,true表示强制使用CGLIB代理
	if (!proxyFactory.isProxyTargetClass()) {
		// 配置了还不行，还得判断该Bean是否包含接口
		// 如果不包含，将来要使用CGLIB方式
		// 如果包含，还是走JDK方式，并且将接口设置到ProxyFactory代理工厂
		if (shouldProxyTargetClass(beanClass, beanName)) {
			proxyFactory.setProxyTargetClass(true);
		}
		else {
			evaluateProxyInterfaces(beanClass, proxyFactory);
		}
	}

	// 2.将当前bean适合的advice，重新封装下，封装为Advisor类，然后添加到ProxyFactory中
	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
	proxyFactory.addAdvisors(advisors);
	proxyFactory.setTargetSource(targetSource);
	customizeProxyFactory(proxyFactory);

	// 冻结代理工厂，此时不允许其他线程来对该对象再次代理
	proxyFactory.setFrozen(this.freezeProxy);
	if (advisorsPreFiltered()) {
		proxyFactory.setPreFiltered(true);
	}

	// 3.调用getProxy获取bean对应的proxy（命名怎么跟getBean一个德行?）
	return proxyFactory.getProxy(getProxyClassLoader());
}

```
- org.springframework.aop.framework.ProxyFactory.getProxy(java.lang.ClassLoader)，接下来的调用和通过factoryBean创建方式一样

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
	// 这里有两个方法
	// createAopProxy()-->得到代理工具类对象：JdkDynamicAopProxy or ObjenesisCglibAopProxy
	// getProxy() --->得到原始bean的代理对象
	return createAopProxy().getProxy(classLoader);
}
```

### 两种创建方式区别
- 第一种是将factoryBean放入缓存中，getBean时通过factoryBean的getObject方法创建代理对象
- 第二种是在被代理对象属性赋值之后，通过initializeBean方法调用org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator.postProcessAfterInitialization，创建代理对象
- 两种对象的创建时机不一致
