# Spring

## IOC主要流程解析

### 以ClassPathXmlApplicationContext为例解析

#### 类图

- 图片暂不提供......TODO
- ClassPathXmlApplicationContext虽然实现了BeanFactory接口，但是自身保留了一个beanFactory属性，用来对bean进行操作。
- ClassPathXmlApplicationContext持有的beanFactory为ConfigurableListableBeanFactory,因为ConfigurableListableBeanFactory实现了所有的beanfactory相关接口，是功能最强大的beanfactory
- `TODO` 循环依赖
- `TODO` AOP
- `TODO` 事物
#### 主流程
- ClassPathXmlApplicationContext构造方法
```java
public ClassPathXmlApplicationContext(
		String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
		throws BeansException {
	super(parent);
	// 根据提供的路径，处理成配置文件数组 【注释】
	setConfigLocations(configLocations);
	if (refresh) {
		// 核心方法【注释】
		refresh();
	}
}
```

- org.springframework.context.support.AbstractApplicationContext.refresh
```java
// 【注释】，为什么不叫init呢，因为调用refresh方法会将原来的ApplicationContext销毁，然后重新执行一次初始化操作
@Override
public void refresh() throws BeansException, IllegalStateException {
	// 加锁
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		// 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符 【注释】
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		// 这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
		// 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
		// 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		// 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			// 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
			// 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】

			// 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
			// 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			// 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			// 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
			// 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
			// 两个方法分别在 Bean 初始化之前和初始化之后得到执行。注意，到这里 Bean 还没初始化
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			// 初始化当前 ApplicationContext 的 MessageSource，国际化这里就不展开说了，不然没完没了了
			initMessageSource();

			// Initialize event multicaster for this context.
			// 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			// 从方法名就可以知道，典型的模板方法(钩子方法)，
			// 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
			onRefresh();

			// Check for listener beans and register them.
			// 注册事件监听器，监听器需要实现 ApplicationListener 接口。这也不是我们的重点，过
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			// 重点，重点，重点
			// 初始化所有的 singleton beans
			//（lazy-init 的除外）
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			// 最后，广播事件，ApplicationContext 初始化完成
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

- org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization
```java
// 初始化剩余的 singleton beans
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// Initialize conversion service for this context.
	// 首先，初始化名字为 conversionService 的 Bean。本着送佛送到西的精神，我在附录中简单介绍了一下 ConversionService，因为这实在太实用了
	// 什么，看代码这里没有初始化 Bean 啊！
	// 注意了，初始化的动作包装在 beanFactory.getBean(...) 中，这里先不说细节，先往下看吧
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	// Register a default embedded value resolver if no bean post-processor
	// (such as a PropertyPlaceholderConfigurer bean) registered any before:
	// at this point, primarily for resolution in annotation attribute values.
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}

	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	// 先初始化 LoadTimeWeaverAware 类型的 Bean
	// 一般用于织入第三方模块，在 class 文件载入 JVM 的时候动态织入，这里不展开说
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// Stop using the temporary ClassLoader for type matching.
	beanFactory.setTempClassLoader(null);

	// Allow for caching all bean definition metadata, not expecting further changes.
	// 没什么别的目的，因为到这一步的时候，Spring 已经开始预初始化 singleton beans 了，
	// 肯定不希望这个时候还出现 bean 定义解析、加载、注册
	beanFactory.freezeConfiguration();

	// Instantiate all remaining (non-lazy-init) singletons.
	// 开始初始化剩下的
	beanFactory.preInstantiateSingletons();
}
```

- org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons

```java
public void preInstantiateSingletons() throws BeansException {
	if (logger.isTraceEnabled()) {
		logger.trace("Pre-instantiating singletons in " + this);
	}

	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

	// Trigger initialization of all non-lazy singleton beans...
	// 触发所有的非懒加载的 singleton beans 的初始化操作
	for (String beanName : beanNames) {
		// 合并父 Bean 中的配置，注意 <bean id="" class="" parent="" /> 中的 parent，用的不多吧，
		// 考虑到这可能会影响大家的理解，我在附录中解释了一下 "Bean 继承"，请移步
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		// 非抽象、非懒加载的 singletons。如果配置了 'abstract = true'，那是不需要初始化的
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			// 处理 FactoryBean(读者如果不熟悉 FactoryBean，请移步附录区了解)
			if (isFactoryBean(beanName)) {
				// FactoryBean 的话，在 beanName 前面加上 ‘&’ 符号。再调用 getBean，getBean 方法别急
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					FactoryBean<?> factory = (FactoryBean<?>) bean;
					// 判断当前 FactoryBean 是否是 SmartFactoryBean 的实现，此处忽略，直接跳过
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(
								(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
			}
			else {
				// 对于普通的 Bean，只要调用 getBean(beanName) 这个方法就可以进行初始化了
				getBean(beanName);
			}
		}
	}

	// Trigger post-initialization callback for all applicable beans...
	// 到这里说明所有的非懒加载的 singleton beans 已经完成了初始化
	// 如果我们定义的 bean 是实现了 SmartInitializingSingleton 接口的，那么在这里得到回调，忽略
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					smartSingleton.afterSingletonsInstantiated();
					return null;
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```
- org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean 获取bean，没初始化先初始化
```java
// 我们在剖析初始化 Bean 的过程，但是 getBean 方法我们经常是用来从容器中获取 Bean 用的，注意切换思路，
// 已经初始化过了就从容器中直接返回，否则就先初始化再返回
@SuppressWarnings("unchecked")
protected <T> T doGetBean(
		String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
		throws BeansException {

	// 获取一个 “正统的” beanName，处理两种情况，一个是前面说的 FactoryBean(前面带 ‘&’)，
	// 一个是别名问题，因为这个方法是 getBean，获取 Bean 用的，你要是传一个别名进来，是完全可以的
	String beanName = transformedBeanName(name);

	// 注意跟着这个，这个是返回值
	Object bean;

	// Eagerly check singleton cache for manually registered singletons.
	// 检查下是不是已经创建过了
	Object sharedInstance = getSingleton(beanName);

	// 这里说下 args 呗，虽然看上去一点不重要。前面我们一路进来的时候都是 getBean(beanName)，
	// 所以 args 其实是 null 的，但是如果 args 不为空的时候，那么意味着调用方不是希望获取 Bean，而是创建 Bean
	if (sharedInstance != null && args == null) {
		if (logger.isTraceEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		// 下面这个方法：如果是普通 Bean 的话，直接返回 sharedInstance，
		// 如果是 FactoryBean 的话，返回它创建的那个实例对象
		// (FactoryBean 知识，读者若不清楚请移步附录)
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		if (isPrototypeCurrentlyInCreation(beanName)) {
			// 当前线程已经创建过了此 beanName 的 prototype 类型的 bean，那么抛异常
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
		// 检查一下这个 BeanDefinition 在容器中是否存在
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			// 如果当前容器不存在这个 BeanDefinition，试试父容器中有没有
			String nameToLookup = originalBeanName(name);
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
			else if (args != null) {
				// 返回父容器的查询结果
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else if (requiredType != null) {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
			else {
				return (T) parentBeanFactory.getBean(nameToLookup);
			}
		}

		if (!typeCheckOnly) {
			// typeCheckOnly 为 false，将当前 beanName 放入一个 alreadyCreated 的 Set 集合中
			markBeanAsCreated(beanName);
		}

		/*
		 * 稍稍总结一下：
		 * 到这里的话，要准备创建 Bean 了，对于 singleton 的 Bean 来说，容器中还没创建过此 Bean；
		 * 对于 prototype 的 Bean 来说，本来就是要创建一个新的 Bean。
		 */
		try {
			RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
			// 先初始化依赖的所有 Bean，这个很好理解。
			// 注意，这里的依赖指的是 depends-on 中定义的依赖
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					// 检查是不是有循环依赖，这里的循环依赖和我们前面说的循环依赖又不一样，这里肯定是不允许出现的，不然要乱套了，读者想一下就知道了
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					// 注册一下依赖关系
					registerDependentBean(dep, beanName);
					try {
						// 先初始化被依赖项
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}

			// Create bean instance.
			// 创建 singleton 的实例
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, () -> {
					try {
						// 执行创建 Bean，详情后面再说
						return createBean(beanName, mbd, args);
					}
					catch (BeansException ex) {
						// Explicitly remove instance from singleton cache: It might have been put there
						// eagerly by the creation process, to allow for circular reference resolution.
						// Also remove any beans that received a temporary reference to the bean.
						destroySingleton(beanName);
						throw ex;
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			// 创建 prototype 的实例
			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					// 执行创建 Bean
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			// 如果不是 singleton 和 prototype 的话，需要委托给相应的实现类来处理
			else {
				String scopeName = mbd.getScope();
				if (!StringUtils.hasLength(scopeName)) {
					throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
				}
				Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, () -> {
						beforePrototypeCreation(beanName);
						try {
							// 执行创建 Bean
							return createBean(beanName, mbd, args);
						}
						finally {
							afterPrototypeCreation(beanName);
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
	// 最后，检查一下类型对不对，不对的话就抛异常，对的话就返回了
	if (requiredType != null && !requiredType.isInstance(bean)) {
		try {
			T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
			if (convertedBean == null) {
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
			return convertedBean;
		}
		catch (TypeMismatchException ex) {
			if (logger.isTraceEnabled()) {
				logger.trace("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```

- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])
```java
/**
 * Central method of this class: creates a bean instance,
 * populates the bean instance, applies post-processors, etc.
 * @see #doCreateBean
 */
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {

	if (logger.isTraceEnabled()) {
		logger.trace("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;

	// Make sure bean class is actually resolved at this point, and
	// clone the bean definition in case of a dynamically resolved Class
	// which cannot be stored in the shared merged bean definition.
	// 确保 BeanDefinition 中的 Class 被加载 【注释】
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	// Prepare method overrides.
	// 准备方法覆写，这里又涉及到一个概念：MethodOverrides，它来自于 bean 定义中的 <lookup-method />
	// 和 <replaced-method />，如果读者感兴趣，回到 bean 解析的地方看看对这两个标签的解析。
	// 我在附录中也对这两个标签的相关知识点进行了介绍，读者可以移步去看看
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
		// 让 BeanPostProcessor 在这一步有机会返回代理，而不是 bean 实例，
		// 要彻底了解清楚这个，需要去看 InstantiationAwareBeanPostProcessor 接口，这里就不展开说了
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	try {
		// 重头戏，创建 bean
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isTraceEnabled()) {
			logger.trace("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
	catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
		// A previously detected exception with proper bean creation context already,
		// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
		throw ex;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
	}
}
```

- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean 这里有创建bean，解决循环依赖，属性赋值，触发监听事件等操作
```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {

	// Instantiate the bean.
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		// 说明不是 FactoryBean，这里实例化 Bean，这里非常关键，细节之后再说
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	// 这个就是 Bean 里面的 我们定义的类 的实例，很多地方我描述成 "bean 实例"
	Object bean = instanceWrapper.getWrappedInstance();
	// 类型
	Class<?> beanType = instanceWrapper.getWrappedClass();
	if (beanType != NullBean.class) {
		mbd.resolvedTargetType = beanType;
	}

	// Allow post-processors to modify the merged bean definition.
	// 建议跳过吧，涉及接口：MergedBeanDefinitionPostProcessor
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
				// MergedBeanDefinitionPostProcessor，这个我真不展开说了，直接跳过吧，很少用的
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Post-processing of merged bean definition failed", ex);
			}
			mbd.postProcessed = true;
		}
	}

	// Eagerly cache singletons to be able to resolve circular references
	// even when triggered by lifecycle interfaces like BeanFactoryAware.
	// 下面这块代码是为了解决循环依赖的问题，以后有时间，我再对循环依赖这个问题进行解析吧 【TODO】
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isTraceEnabled()) {
			logger.trace("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}

	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
		// 这一步也是非常关键的，这一步负责属性装配，因为前面的实例只是实例化了，并没有设值，这里就是设值
		populateBean(beanName, mbd, instanceWrapper);
		// 还记得 init-method 吗？还有 InitializingBean 接口？还有 BeanPostProcessor 接口？
		// 这里就是处理 bean 初始化完成后的各种回调
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}

	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}

	// Register bean as disposable.
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}

	return exposedObject;
}
```
- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition) 各种回调
```java
// 属性注入完成后，这一步其实就是处理各种回调了 【注释】,AOP的实现也会在这里触发applyBeanPostProcessorsAfterInitialization（不包括aspectj）
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
			invokeAwareMethods(beanName, bean);
			return null;
		}, getAccessControlContext());
	}
	else {
		// 如果 bean 实现了 BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口，回调
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		// BeanPostProcessor 的 postProcessBeforeInitialization 回调
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
		// 处理 bean 中定义的 init-method，
		// 或者如果 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}
	if (mbd == null || !mbd.isSynthetic()) {
		// BeanPostProcessor 的 postProcessAfterInitialization 回调
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}

	return wrappedBean;
}
```

#### 注册bean流程
- org.springframework.context.support.AbstractApplicationContext.refresh的obtainFreshBeanFactory() -> org.springframework.context.support.AbstractApplicationContext.obtainFreshBeanFactory
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		// 关闭旧的 BeanFactory (如果有)，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等等
		refreshBeanFactory();
		// 返回刚刚创建的 BeanFactory
		return getBeanFactory();
	}

```

- org.springframework.context.support.AbstractRefreshableApplicationContext.refreshBeanFactory
```java
protected final void refreshBeanFactory() throws BeansException {
	// 如果 当前ApplicationContext 中已经加载过 BeanFactory 了，销毁所有 Bean，关闭 BeanFactory
	// 注意，应用中 BeanFactory 本来就是可以多个的，这里可不是说应用全局是否有 BeanFactory，而是当前
	// ApplicationContext 是否有 BeanFactory 【注释】
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		// 初始化一个 DefaultListableBeanFactory，为什么用这个，我们马上说
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		// 用于 BeanFactory 的序列化，我想大部分人应该都用不到
		beanFactory.setSerializationId(getId());
		// 下面这两个方法很重要，别跟丢了，具体细节之后说
		// 设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
		customizeBeanFactory(beanFactory);
		// 加载 Bean 到 BeanFactory 中
		loadBeanDefinitions(beanFactory);
		this.beanFactory = beanFactory;
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

- org.springframework.context.support.AbstractXmlApplicationContext.loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)
```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		// 我们可以看到，此方法将通过一个 XmlBeanDefinitionReader 实例来加载各个 Bean 【注释】
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		// 重点来了，继续往下
		loadBeanDefinitions(beanDefinitionReader);
}
```

- org.springframework.context.support.AbstractXmlApplicationContext.loadBeanDefinitions(org.springframework.beans.factory.xml.XmlBeanDefinitionReader)

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			// 往下看 【注释】
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			// 这个分支经过路径解析转换后，会走到上边分支调用的方法
			reader.loadBeanDefinitions(configLocations);
		}
	}
```

- org.springframework.beans.factory.support.AbstractBeanDefinitionReader.loadBeanDefinitions(org.springframework.core.io.Resource...)
```java
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int count = 0;
		// 注意这里是个 for 循环，也就是每个文件是一个 resource
		for (Resource resource : resources) {
			// 继续往下看
			count += loadBeanDefinitions(resource);
		}
		return count;
	}
```

- org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(org.springframework.core.io.support.EncodedResource)
```java
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}
		// 用一个 ThreadLocal 来存放所有的配置文件资源
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}

		try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			// 核心部分
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```

- org.springframework.beans.factory.xml.XmlBeanDefinitionReader.doLoadBeanDefinitions
```java
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
			// resource转换为doc对象
			Document doc = doLoadDocument(inputSource, resource);
			// 继续
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
```

- org.springframework.beans.factory.xml.XmlBeanDefinitionReader.registerBeanDefinitions
```java
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		// 往下看
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

- org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader.registerBeanDefinitions
```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	this.readerContext = readerContext;
	// 经过漫长的链路，一个配置文件终于转换为一颗 DOM 树了，注意，这里指的是其中一个配置文件，不是所有的，读者可以看到上面有个 for 循环的。下面从根节点开始解析：
	doRegisterBeanDefinitions(doc.getDocumentElement());
}
```

- org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader.doRegisterBeanDefinitions
```java
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.

		// 我们看名字就知道，BeanDefinitionParserDelegate 必定是一个重要的类，它负责解析 Bean 定义， 【注释】
		// 这里为什么要定义一个 parent? 看到后面就知道了，是递归问题，
		// 因为 <beans /> 内部是可以定义 <beans /> 的，所以这个方法的 root 其实不一定就是 xml 的根节点，也可以是嵌套在里面的 <beans /> 节点，从源码分析的角度，我们当做根节点就好了
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			// 这块说的是根节点 <beans ... profile="dev" /> 中的 profile 是否是当前环境需要的，
			// 如果当前环境配置的 profile 不包含此 profile，那就直接 return 了，不对此 <beans /> 解析
			// 不熟悉 profile 为何物，不熟悉怎么配置 profile 读者的请移步附录区
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root); // 钩子
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root); // 钩子

		this.delegate = parent;
	}
```

- org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader.parseBeanDefinitions
```java
// default namespace 涉及到的就四个标签 <import />、<alias />、<bean /> 和 <beans />，【注释】
// 其他的属于 custom 的
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	if (delegate.isDefaultNamespace(root)) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) {
					// parseDefaultElement(ele, delegate) 代表解析的节点是 <import />、<alias />、<bean />、<beans /> 这几个。
					parseDefaultElement(ele, delegate);
				}
				else {
					// 而对于其他的标签，将进入到 delegate.parseCustomElement(element) 这个分支。如我们经常会使用到的 <mvc />、<task />、<context />、<aop />等
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
		delegate.parseCustomElement(root);
	}
}
```

- org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader.parseDefaultElement
```java
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			// 处理 <import /> 标签
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			// 处理 <alias /> 标签定义
			// <alias name="fromName" alias="toName"/>
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			// 处理 <bean /> 标签定义，这也算是我们的重点吧
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			// 如果碰到的是嵌套的 <beans /> 标签，需要递归
			doRegisterBeanDefinitions(ele);
		}
	}
```

- org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader.processBeanDefinition
```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	// 将 <bean /> 节点中的信息提取出来，然后封装到一个 BeanDefinitionHolder 中，细节往下看 【注释】
	// TODO 【TODO】 bean解析细节
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		// 如果有自定义属性的话，进行相应的解析，先忽略
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// Register the final decorated instance.
			// 我们把这步叫做 注册Bean 吧
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to register bean definition with name '" +
					bdHolder.getBeanName() + "'", ele, ex);
		}
		// Send registration event.
		// 发送注册事件
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```

- org.springframework.beans.factory.support.BeanDefinitionReaderUtils.registerBeanDefinition
```java
public static void registerBeanDefinition(
		BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {

	// Register bean definition under primary name.

	String beanName = definitionHolder.getBeanName();
	// 注册这个 Bean 【注释】
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

	// Register aliases for bean name, if any.
	// 如果还有别名的话，也要根据别名统统注册一遍，不然根据别名就找不到 Bean 了，这我们就不开心了
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String alias : aliases) {
			// alias -> beanName 保存它们的别名信息，这个很简单，用一个 map 保存一下就可以了，
			// 获取的时候，会先将 alias 转换为 beanName，然后再查找
			registry.registerAlias(beanName, alias);
		}
	}
}
```

- org.springframework.beans.factory.support.DefaultListableBeanFactory.registerBeanDefinition
```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {

	Assert.hasText(beanName, "Bean name must not be empty");
	Assert.notNull(beanDefinition, "BeanDefinition must not be null");

	if (beanDefinition instanceof AbstractBeanDefinition) {
		try {
			((AbstractBeanDefinition) beanDefinition).validate();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
					"Validation of bean definition failed", ex);
		}
	}
	// existingDefinition? 还记得 “允许 bean 覆盖” 这个配置吗？allowBeanDefinitionOverriding
	// 之后会看到，所有的 Bean 注册后会放入这个 beanDefinitionMap 中
	BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
	if (existingDefinition != null) {
		if (!isAllowBeanDefinitionOverriding()) {
			// 不允许覆盖，则抛异常
			throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
		}
		else if (existingDefinition.getRole() < beanDefinition.getRole()) {
			// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
			if (logger.isInfoEnabled()) {
				// 用框架定义的 Bean 覆盖用户自定义的 Bean
				logger.info("Overriding user-defined bean definition for bean '" + beanName +
						"' with a framework-generated bean definition: replacing [" +
						existingDefinition + "] with [" + beanDefinition + "]");
			}
		}
		else if (!beanDefinition.equals(existingDefinition)) {
			if (logger.isDebugEnabled()) {
				// 用新的 Bean 覆盖旧的 Bean
				logger.debug("Overriding bean definition for bean '" + beanName +
						"' with a different definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				// 用同等的 Bean 覆盖旧的 Bean，这里指的是 equals 方法返回 true 的 Bean
				logger.trace("Overriding bean definition for bean '" + beanName +
						"' with an equivalent definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		this.beanDefinitionMap.put(beanName, beanDefinition);
	}
	else {
		// 判断是否已经有其他的 Bean 开始初始化了.
		// 注意，"注册Bean" 这个动作结束，Bean 依然还没有初始化，我们后面会有大篇幅说初始化过程，
		// 在 Spring 容器启动的最后，会 预初始化 所有的 singleton beans
		if (hasBeanCreationStarted()) {
			// Cannot modify startup-time collection elements anymore (for stable iteration)
			synchronized (this.beanDefinitionMap) {
				this.beanDefinitionMap.put(beanName, beanDefinition);
				List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
				updatedDefinitions.addAll(this.beanDefinitionNames);
				updatedDefinitions.add(beanName);
				this.beanDefinitionNames = updatedDefinitions;
				removeManualSingletonName(beanName);
			}
		}
		else {
			// 最正常的应该是进到这里。
			// Still in startup registration phase
			// 将 BeanDefinition 放到这个 map 中，这个 map 保存了所有的 BeanDefinition
			this.beanDefinitionMap.put(beanName, beanDefinition);
			// 这是个 ArrayList，所以会按照 bean 配置的顺序保存每一个注册的 Bean 的名字
			this.beanDefinitionNames.add(beanName);
			// 这是个 LinkedHashSet，代表的是手动注册的 singleton bean，
			// 注意这里是 remove 方法，到这里的 Bean 当然不是手动注册的
			// 手动指的是通过调用以下方法注册的 bean ：
			//     registerSingleton(String beanName, Object singletonObject)
			//         这不是重点，解释只是为了不让大家疑惑。Spring 会在后面"手动"注册一些 Bean，如 "environment"、"systemProperties" 等 bean
			removeManualSingletonName(beanName);
		}
		// 这个不重要，在预初始化的时候会用到，不必管它。
		this.frozenBeanDefinitionNames = null;
	}

	if (existingDefinition != null || containsSingleton(beanName)) {
		resetBeanDefinition(beanName);
	}
	else if (isConfigurationFrozen()) {
		clearByTypeCache();
	}
}
```
- 就是把beanname->BeanDefinition 放到一个map中beanDefinitionMap
