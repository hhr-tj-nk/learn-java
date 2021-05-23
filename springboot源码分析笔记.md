## 概览
* 自动装配分析
* 配置类增强分析
* 单例Bean的创建分析
* 循环依赖分析

## 1、自动装配分析
### 1.1 启动类作为配置类，参与配置类解析
* 启动类的启动注解声明了@Confiuration注解，启动类可以当成一个配置类
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {}


@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {}
```
* 上下文解析时，将启动类加载，注册进Bean工厂
```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    // 省略部分代码

    // sources里面包含了启动类，
    // SpringApplication#run，创建SpringApplication，启动类赋值给primarySources属性
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
    // Class转BeanDefinition，见AnnotatedBeanDefinitionReader#doRegisterBean
		load(context, sources.toArray(new Object[0]));
		listeners.contextLoaded(context);
}
```

* 创建上下文时，注册工厂后置处理器ConfigurationClassPostProcessor
```java
public AnnotationConfigServletWebServerApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
        // 注册ConfigurationClassPostProcessor实例
        //  RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        //  def.setSource(source);
        //  beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)); 
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```
* 上下文刷新时，调用工厂后置处理器进行bean注册，则也会调用ConfigurationClassPostProcessor进行配置类的解析注册，其中就包含了启动类
```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
            //  配置类已经解析过，跳过
			if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
            // 是否包含Configuration注解？
            // 根据proxyBeanMethods属性，决定是轻量加载还是全量加载（代理增强）
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}
        // 省略部分代码

		// 配置类解析器
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {   
			parser.parse(candidates);
			parser.validate();
            // 解析后，可能会引入一些新的配置类 
			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// 注册这些新引入的配置类到工厂
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				for (String candidateName : newCandidateNames) {
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
        // 不断处理新加入的配置类
		while (!candidates.isEmpty());
		}
	}
     
    // 将配置解析器收集的 ImportRegistry注册，用于支持ImportAware @Configuration classes
    // see ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor#postProcessBeforeInitialization
    if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
			sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
     }
```
### 1.2 配置类解析
* 进行过滤、去重处理
```java
// // ConfigurationClassParser#doProcessConfigurationClass
protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
        // 基于Conditional注解过滤
        // 指定过滤阶段为ConfigurationPhase.PARSE_CONFIGURATION，如果Condition是ConfigurationCondition，则有生效阶段，阶段匹配才会进行过滤
		if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}

		ConfigurationClass existingClass = this.configurationClasses.get(configClass);
		if (existingClass != null) {
			if (configClass.isImported()) {
				if (existingClass.isImported()) {
					existingClass.mergeImportedBy(configClass);
				}
				// Otherwise ignore new imported config class; existing non-imported class overrides it.
				return;
			}
			else {
				this.configurationClasses.remove(configClass);
				this.knownSuperclasses.values().removeIf(configClass::equals);
			}
		}

		SourceClass sourceClass = asSourceClass(configClass, filter);
		do {
            // 返回父类，再尝试对父类进行解析  
			sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
		}
		while (sourceClass != null);

		this.configurationClasses.put(configClass, configClass);
}
```
* 解析
```java
// ConfigurationClassParser#doProcessConfigurationClass
protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException {

       
		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// 处理内部类
            // 如果内部类有这些注解，@Component、 @ComponentScan、@Import、@ImportResource，或者有@Bean方法
            // 当成配置类，先进行解析
			processMemberClasses(configClass, sourceClass, filter);
		}
        // 解析@PropertySource，@PropertySources注解，加载配置文件
		//  解析占位符，获取真正配置 String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);，
        //  加载文件资源，Resource resource = this.resourceLoader.getResource(resolvedLocation);
        //  转换为Properties，加入环境的配置集合，addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			}
			else {
				logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}

		// 处理@ComponentScans， @ComponentScan
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        // 基于Condition注解的过滤，当前生效阶段是 REGISTER_BEAN
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// 借助ClassPathBeanDefinitionScanner，扫描配置的包下面的所有类解析为BeanDefinition，这里也是应用内的Bean自动装配的地方
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
                    // 如果是候选的配置类，那么又是一轮递归的配置类解析
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

		// 处理@Import注解
        // 校验循环Import，配置类A import了配置类B，但是B又import了A
        // 通过importStack中记录当前正在解析Import注解的配置类进行校验
        // 1、ImportSelector ：实例化、调用Aware相关方法
        //   如果是DeferredImportSelector：先收集起来，所有配置类解析完成后，统一处理。
        //   如果是普通的mportSelector：调用selectImports，获取导入的类，递归进行processImports
        // 2、ImportBeanDefinitionRegistrar：实例化，先缓存起来，在所有配置类解析结束后，和BeanMethod、ImportResource一起进行加载ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass
        // 3、不是ImportBeanDefinitionRegistrar，也不是ImportSelector，那么当成一个配置类，再次进行递归的配置类解析
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

        // @ImportResource：引入一些xml形式定义的Bean
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}

		// 收集BeanMethod，后面统一处理ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}

		// 收集实现的接口的Bean Method，应该是以default method的形式
		processInterfaces(configClass, sourceClass);

		// 返回父类，进行父类的配置类解析
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (superclass != null && !superclass.startsWith("java") &&
					!this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				return sourceClass.getSuperClass();
			}
		}
		return null;
	}
```
### 1.3 配置类解析总结
* 基于Condition注解进行过滤，例如在配置类解析前决定是否跳过该配置类、处理@ComponentScan引入的Bean时是否跳过该Bean等
* 如果Condition注解指定的Condition类，实现的是ConfigurationCondition，代表这个Condition类存在生效阶段，需要以更细的粒度进行条件过滤
```java
// ConditionEvaluator
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
		if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
			return false;
		}

        // 实例化Condition类
		List<Condition> conditions = new ArrayList<>();
		for (String[] conditionClasses : getConditionClasses(metadata)) {
			for (String conditionClass : conditionClasses) {
				Condition condition = getCondition(conditionClass, this.context.getClassLoader());
				conditions.add(condition);
			}
		}

		AnnotationAwareOrderComparator.sort(conditions);

		for (Condition condition : conditions) {
			ConfigurationPhase requiredPhase = null;
			if (condition instanceof ConfigurationCondition) {
				requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
			}
            // 有生效阶段的话，阶段匹配才进行条件过滤
			if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
				return true;
			}
		}

		return false;
}
```
* 配置类解析时，进行了对内部类、@PropertySource，@PropertySources、@ComponentScans， @ComponentScan、@Import注解、@ImportResource、BeanMethod的处理
* @PropertySource、@PropertySources用于引入配置文件，配置的location将被解析占位符、填充环境配置得到真正的文件路径，然后作为资源加载，解析为Properties，加入到环境的配置集合
    
```java
     // 递归解析，占位符内可能还包含占位符
     // 解析到真正的属性名和默认值后，去环境配置集合获取配置，PropertySourcesPropertyResolver#getProperty
     // 并会根据目标类型进行可能的类型转换，例如集合、Map等类型支持  GenericConversionService#convert
    String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);
    Resource resource = this.resourceLoader.getResource(resolvedLocation);
    // 加入环境的配置集合
    addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
```
* @Import注解引入Bean，包含对ImportSelector、DeferredImportSelector、ImportBeanDefinitionRegistrar、普通类的处理。DeferredImportSelector都是所有配置类进行完成后进行处理，ImportBeanDefinitionRegistrar在所有配置类解析完成后处理。
   * 典型应用了DeferredImportSelector的配置类，@EnableAutoConfiguration注解，将spring.factory下所有EnableAutoConfiguration对应的配置类引入，所以三方包配置了EnableAutoConfiguration可以将自己的bean注册
* @ComponentScan：启动类作为配置类解析时，ComponentScan注解配置的包路径默认值即为所在配置类的包路径，所以应用内和配置类同包的Bean都会被自动装配
     
## 2、配置类增强
* 配置类解析后，会尝试进行配置类增强
```java
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
		Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();

        for (String beanName : beanFactory.getBeanDefinitionNames()) {
			BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
            //  获取配置类时，会根据其proxyBeanMethods属性，进行ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE的属性赋值
            //  ConfigurationClassUtils#checkConfigurationClassCandidate
			Object configClassAttr = beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE);
			MethodMetadata methodMetadata = null;
			if (beanDef instanceof AnnotatedBeanDefinition) {
				methodMetadata = ((AnnotatedBeanDefinition) beanDef).getFactoryMethodMetadata();
            }
            // 不是轻量配置类，需要进行配置类增强
            if (ConfigurationClassUtils.CONFIGURATION_CLASS_FULL.equals(configClassAttr)) {		
	            configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
            }
	    }
        
        // 省略部分代码

		ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
		for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
			AbstractBeanDefinition beanDef = entry.getValue();
			// If a @Configuration class gets proxied, always proxy the target class
			beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			// Set enhanced subclass of the user-specified bean class
			Class<?> configClass = beanDef.getBeanClass();
			Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
			if (configClass != enhancedClass) {
				if (logger.isTraceEnabled()) {
					logger.trace(String.format("Replacing bean definition '%s' existing class '%s' with " +
							"enhanced class '%s'", entry.getKey(), configClass.getName(), enhancedClass.getName()));
				}
				beanDef.setBeanClass(enhancedClass);
			}
		}
}
```
* 代理逻辑
```java
class ConfigurationClassEnhancer {
        private Enhancer newEnhancer(Class<?> configSuperClass, @Nullable ClassLoader classLoader) {
		Enhancer enhancer = new Enhancer();
		enhancer.setSuperclass(configSuperClass);
        // 标识配置类增强过，需要再后置处理器中调用EnhancedConfiguration#setBeanFactory
		enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
		enhancer.setUseFactory(false);
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        // 为了只创建一次，需要增加先尝试从Bean工厂获取Bean的逻辑
        // 给配置类的代理类增加一个Bean Factory的属性和set方法，属性名为$$beanFactory
        // 在Bean后置处理器ImportAwareBeanPostProcessor中，判断如果当前配置类实现了EnhancedConfiguration接口，则调用setBeanFactory方法
		enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
		enhancer.setCallbackFilter(CALLBACK_FILTER);
		enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
		return enhancer;
	}

	// The callbacks to use. Note that these callbacks must be stateless.
	private static final Callback[] CALLBACKS = new Callback[] {
            // 增强Bean Method，只创建一次
			new BeanMethodInterceptor(),
            // 代理setBeanFactory方法，为增加的$$beanFactory属性赋值
            // 如果本身就实现了BeanFactoryAware，也调用下父类（被增强的配置类）的setBeanFactory方法
			new BeanFactoryAwareMethodInterceptor(),
			NoOp.INSTANCE
     }
};
```
* 总结：全量配置类的Bean Method只会创建一次Bean，轻量配置类则会创建多次。
* 怎样的配置类是轻量配置类？
  * Configuration注解中proxyBeanMethods属性为false
  * 有@Component、@ComponentScan、@Import、@ImportResource、Bean Method的类，也会被认为是配置类，但是默认作为轻量配置类处理
      ```java
        Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
		if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
		}
        // isConfigurationCandidate，即包含@Component、@ComponentScan、@Import、@ImportResource、Bean Method的类
		else if (config != null || isConfigurationCandidate(metadata)) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
		}
      ```

## 3、单例Bean的创建
### 3.1 创建时机
* 上下文刷新
```java
                // Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// 创建所有非懒加载单例Bean
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
```
### 3.2 实例化
* 实例化，可能使用工厂方法或者选择一个构造器进行实例化
```java
// AbstractAutowireCapableBeanFactory#createBeanInstance
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
        //  instanceSupplier方式实例化
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
        // 有工厂方法，使用工厂方法实例化，Bean Method就是一种工厂方法 
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// 接下来要选择构造器进行，对于多例Bean增加一些缓存逻辑，下次实例化，不用再选择构造器
        // 省略部分代码

        // 通过Bean的后置处理器选择构造器
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// 没有选中的构造器，使用无参构造器进行实例化
		return instantiateBean(beanName, mbd);
}
```
* 通过Bean后置处理器选择构造器，实现了SmartInstantiationAwareBeanPostProcessor的后置处理器，通过重写determineCandidateConstructors，可以指定实例化所用的构造器。这里的实现是AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors，寻找Autowired注解修饰的构造器
```java
        Constructor<?>[] rawCandidates = beanClass.getDeclaredConstructors();
		
		List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
		Constructor<?> requiredConstructor = null;
		Constructor<?> defaultConstructor = null;
		int nonSyntheticConstructors = 0;
		for (Constructor<?> candidate : rawCandidates) {
          	// 编译器通过生成一些在源代码中不存在的synthetic方法和类的方式，实现了对private级别的字段和类的访问，从而绕开了语言限制
          	// 这里好像是编译器为了父类访问子类私有构造器的访问而生成的Synthetic构造器, private内部类的默认构造器也是private的
			if (!candidate.isSynthetic()) {
				nonSyntheticConstructors++;
			}
			// @Autowired、@Value、@Inject
			MergedAnnotation<?> ann = findAutowiredAnnotation(candidate);
			if (ann == null) {
              	// 当前类可能是代理类，那么去父类（被代理类）的构造器尝试找下相关注解
				Class<?> userClass = ClassUtils.getUserClass(beanClass);
				if (userClass != beanClass) {
					try {
						Constructor<?> superCtor =
								userClass.getDeclaredConstructor(candidate.getParameterTypes());
						ann = findAutowiredAnnotation(superCtor);
					}
					catch (NoSuchMethodException ex) {
						// Simply proceed, no equivalent superclass constructor found...
					}
				}
			}
			if (ann != null) {
              	// requiredConstructor用来存储相关注解required属性为true的构造器
              	// 这样的构造器，必须是唯一的构造器
				if (requiredConstructor != null) {
					throw new BeanCreationException("........");
				}
              	// 尝试寻找注解的required属性
				boolean required = determineRequiredStatus(ann);
				if (required) {
                // 当前构造器的相关注解声明了required属性为true，那么不允许有其他候选
					if (!candidates.isEmpty()) {
						throw new BeanCreationException("........");
					}
					requiredConstructor = candidate;
				}
				candidates.add(candidate);
			}
			else if (candidate.getParameterCount() == 0) {
                // 无参构造器
				defaultConstructor = candidate;
			}
		}
		if (!candidates.isEmpty()) {
          	// 没有必须唯一的requiredConstructor，则将无参构造器加入候选构造器列表
            if (requiredConstructor == null) {
				if (defaultConstructor != null) {
					candidates.add(defaultConstructor);
				}
			}
			candidateConstructors = candidates.toArray(new Constructor<?>[0]);
		}
  		// 如果没有声明了相关注解的构造器，如果该类只有一个构造器并且是有参构造器，那么也可以加入候选
  		// 即：构造器注入不声明@Autowired注解也可以
		else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
			candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
		}
		else {
			candidateConstructors = new Constructor<?>[0];
		}
```
* 有参构造器实例化时，进行入参解析
```java
// ConstructorResolver#createArgumentArray
protected Object resolveAutowiredArgument(MethodParameter param, String beanName,
			@Nullable Set<String> autowiredBeanNames, TypeConverter typeConverter, boolean fallback) {
		// 省略了部分没看懂的代码
  
		Class<?> paramType = param.getParameterType();
  		// 解析bean
		return this.beanFactory.resolveDependency(
					new DependencyDescriptor(param, true), beanName, autowiredBeanNames, typeConverter);
}

public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

		descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
  		// 对Optional类型的支持
		if (Optional.class == descriptor.getDependencyType()) {
			return createOptionalDependency(descriptor, requestingBeanName);
		}
  		// 对ObjectProvider类型的支持，好像是类似于Optional
		else if (ObjectFactory.class == descriptor.getDependencyType() ||
				ObjectProvider.class == descriptor.getDependencyType()) {
			return new DependencyObjectProvider(descriptor, requestingBeanName);
		}
		else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
			return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
		}
		else {
          	// 包含了对懒加载的支持
			Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
              			descriptor, requestingBeanName);
			if (result == null) {
              	// 真正去解析bean
              	// 1、如果类型是集合类型，转换为相应的集合类型，resolveMultipleBeans
              	// 2、拿到Bean之后，如果有多个候选，进行determineAutowireCandidate，
              	//    基于@Primary注解、@Order注解排序、Qualified注解过滤
				result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
			}
			return result;
		}
}
```

### 3.3 属性收集
* 实例化后，调用后置处理器`MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition`的回调，下面列出了一些实现
* `AutowiredAnnotationBeanPostProcessor`，收集声明了`@Autowired`、`@Value`、`@Inject`注解的字段或者方法，并记录下来
```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
		// 收集声明了Autowired、Value、Inject的字段、方法，转换为InjectedElement
		List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
		Class<?> targetClass = clazz;

		do {
			final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
			// 遍历所有字段
			ReflectionUtils.doWithLocalFields(targetClass, field -> {
              	// 寻找相关注解
				MergedAnnotation<?> ann = findAutowiredAnnotation(field);
				if (ann != null) {
                  	// 不支持static属性
					if (Modifier.isStatic(field.getModifiers())) {
						return;
					}
                  	// 获取注解的required属性
					boolean required = determineRequiredStatus(ann);
					currElements.add(new AutowiredFieldElement(field, required));
				}
			});
			
          	// 遍历方法
			ReflectionUtils.doWithLocalMethods(targetClass, method -> {
              	// 找到被桥接的方法
				Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
				if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
					return;
				}
				MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
				if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                  	// 同样不支持静态方法
					if (Modifier.isStatic(method.getModifiers())) {
						return;
					}
					// required属性
					boolean required = determineRequiredStatus(ann);
                  	// method可能是setter或者getter，寻找包含这个setter或者getter的PropertyDescriptor
					PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
					currElements.add(new AutowiredMethodElement(method, required, pd));
				}
			});

			elements.addAll(0, currElements);
          	// 再去父类寻找，这里比较重要
			targetClass = targetClass.getSuperclass();
		}
		while (targetClass != null && targetClass != Object.class);

		return InjectionMetadata.forElements(elements, clazz);
}
```
* `CommonAnnotationBeanPostProcessor#postProcessMergedBeanDefinition`收集生命周期相关元素并缓存，例如`@PostConstruct`、`PreDestroy`注解，和`@Resource`注解的收集，实现和`AutowiredAnnotationBeanPostProcessor`大体相似


### 3.4 属性传播
#### 3.4.1 后置处理器回调
* `InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`：可以自定义属性注入方式，并且可以决定是否跳过spring的属性传播
* `InstantiationAwareBeanPostProcessor#postProcessProperties`：可以修改PropertyValues（xml标签解析中的property标签封装的对象）或者bean实例。例如`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`对之前收集到的注入属性进行注入，`ImportAwareBeanPostProcessor`对所有增强过的配置类调用`setBeanFactory`存储工厂等
#### 3.4.2 `AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`的注入逻辑
* `AutowiredAnnotationBeanPostProcessor`之前收集到的可以注入的字段和属性被封装为`AutowiredFieldElement`和`AutowiredMethodElement`，这两个类同时也实现了具体的注入逻辑
```java
// AutowiredFieldElement
// AutowiredMethodElement#实现，像是AutowiredFieldElement的循环版，从工厂解析所有入参对应的bean，然后反射method进行
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
  		// 收集到的属性
		Field field = (Field) this.member;
		
		DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
		desc.setContainingClass(bean.getClass());
		Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
		TypeConverter typeConverter = beanFactory.getTypeConverter();
		// 从工厂解析bean，进行可能的类型转换，并包含了对一些特殊类型的支持，例如集合、Optional等
		value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
		
  		// 省略了一些缓存相关的代码
  		
		// 反射注入属性
		if (value != null) {
			ReflectionUtils.makeAccessible(field);
			field.set(bean, value);
		}
}


```

### 3.5 初始化
* 调用Aware Method
* 初始化前回调：`BeanPostProcessor#postProcessBeforeInitialization`
  * `ConfigurationPropertiesBindingPostProcessor`对声明了`@ConfigurationProperties`的bean，进行属性赋值
  * `CommonAnnotationBeanPostProcessor`对之前收集的`@PostConstruct`方法进行反射调用
  * `ImportAwareBeanPostProcessor`对实现了`ImportAware`接口的bean，调用`setImportMetadata`
  * `ApplicationContextAwareProcessor`对其他一些Aware接口，进行Aware方法调用
  * `BeanValidationPostProcessor`对Bean进行校验
* init method：`InitializingBean#afterPropertiesSet`
* 初始化后回调：`BeanPostProcessor#postProcessAfterInitialization`，此时Bean基本已经是完整的Bean了
  * `ScheduledAnnotationBeanPostProcessor`收集所有声明了`@Scheduled`注解的方法，基于注解信息创建调度任务，投入调度器执行
  * `AbstractAutoProxyCreator`获取工厂内所有切面，创建aop代理

## 4、单例循环依赖的解决
### 4.1 触发循环依赖的时机
* bean的创建大体步骤可以分为：实例化，属性收集和属性传播，初始化。在属性传播阶段可能发生循环依赖
### 4.2 大致解决思路
* 缓存提前引用，当产生循环依赖时，让正在创建的bean先使用提前引用完成自己的属性传播
#### 4.2.1 三层缓存
* 单例池：缓存完成创建的bean
* 提前引用缓存：缓存提前暴露的引用
* 提前引用工厂缓存：缓存创建提前暴露引用的工厂方法
#### 4.2.2 一些问题及思考
* 为什么不能直接实例化bean后，直接存入单例池，这样只用一层缓存不就可以了？
  * bean初始化时，可能会替换bean，即实例化后的bean引用，可能是不是最终要使用的引用
  * 那么就提供一个后置处理器`SmartInstantiationAwareBeanPostProcessor`，需要在初始化阶段替换bean的后置处理器，同样要实现`SmartInstantiationAwareBeanPostProcessor`，返回提前暴露的引用，并且要记录下来，从而在初始化阶段避免再次创建，导致提前引用失效
* 二级缓存和三级缓存
  * 三级缓存提供工厂方法，以创建提前引用，如果发生了提前引用的创建，就会从三级缓存移除工厂方法，并将提前引用缓存到二级缓存
  * 二级缓存可以避免提前引用的重复创建
  * 同时二级缓存中如果存在引用，则代表发生了循环依赖，已经创建了循环引用，那么在bean初始化后，再次检查bean，确保没有出现不规范的实现，在初始化阶段再次创建新的引用，导致循环引用失效
* 返回的提前引用，未经历初始化等操作，如果bean本身实现了Aware系列接口，是否还能生效？
  * Aop代理可以生效，因为Aop代理对象中，未代理的方法还是走被代理对象本身的方法逻辑 
