#AOP
## AOP联盟 aopalliance 
- 接口标准化
    - 接口 org.aopalliance.aop.Advice
    - 接口 org.aopalliance.intercept.Interceptor 继承 Advice
    - 接口 org.aopalliance.intercept.MethodInterceptor 继承 Interceptor
## Spring AOP
- 常见的增强类型
    - 接口 org.springframework.aop.BeforeAdvice 
    - 接口 org.springframework.aop.AfterAdvice
        - org.springframework.aop.AfterReturningAdvice
        - org.springframework.aop.ThrowsAdvice
- PointCut切点
    - 用于决定增强应该应用到哪些方法上（连接点）
    - MethodMatcher 用于判断对应方法是否应该应用增强（Advice） 具体会调用其 matches方法判断
- Advisor通知器
    - 作用：配置在哪些切点上应用哪些增强
- ProxyFactoryBean
    - spring aop 声明式配置
    - 配置例子
        ```
            <bean id="testAdvisor" class="com.gloryofme.aop.TestAdvisor"/>
            <bean id="demoTarget" class="com.gloryofme.aop.A"/>
            <bean id="proxyFactoryBean" class="org.springframework.aop.framework.ProxyFactoryBean">
                <property name="target" ref="demoTarget"/>
                <property name="proxyInterfaces">
                    <list>
                        <value>com.gloryofme.aop.InterfaceA</value>
                    </list>
                </property>
                <property name="interceptorNames">
                    <list>
                        <value>testAdvisor</value>
                    </list>
                </property>
            </bean>
        ```    
    - 生成代理对象的原理
        - ProxyFactoryBean 实现了 FactoryBean接口 IOC容器创建bean对象，调用getObject方法
            - getObject方法源码分析
                ```
                    public Object getObject() throws BeansException {
                        // 初始化通知器链
                        initializeAdvisorChain();
                        if (isSingleton()) {
                            // 如果是配置的ProxyFactoryBean是单例的，则生成单例代理对象
                            return getSingletonInstance();
                        }
                        else {
                            if (this.targetName == null) {
                                logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
                                        "Enable prototype proxies by setting the 'targetName' property.");
                            }
                            // 原型类型的，则创建原型类型对象
                            return newPrototypeInstance();
                        }
                    }
                ```    
            - initializeAdvisorChain 方法分析
                ```
                private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
                    if (this.advisorChainInitialized) {
                        return;
                    }
                    if (!ObjectUtils.isEmpty(this.interceptorNames)) {
                        ...
                        // Materialize interceptor chain from bean names. 根据bean名称获取配置的拦截器/通知器链
                        for (String name : this.interceptorNames) {
                            if (name.endsWith(GLOBAL_SUFFIX)) { // GLOBAL_SUFFIX = "*"
                                addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
                                        name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
                            } else {
                                // 判断配置的name对应的通知是单例还是原型类型的
                                Object advice;
                                if (this.singleton || this.beanFactory.isSingleton(name)) {
                                    // Add the real Advisor/Advice to the chain. 根据名称从IOC容器中获取对应的通知
                                    advice = this.beanFactory.getBean(name);
                                } else {
                                    // 创建PrototypePlaceholderAdvisor对象
                                    advice = new PrototypePlaceholderAdvisor(name);
                                }
                                // 往通知器链中添加通知，对应advisors，类型为LinkedList<Advisor>
                                addAdvisorOnChainCreation(advice, name);
                            }
                        }
                    }
                    this.advisorChainInitialized = true;
                }
                ```
            - ProxyFactoryBean的方法getSingletonInstance() 分析
                - 关键代码：```this.singletonInstance = getProxy(createAopProxy());```
                    ``` 
                        protected Object getProxy(AopProxy aopProxy) {
                            // 根据指定的AopProxy对象获取aop代理对象
                            return aopProxy.getProxy(this.proxyClassLoader);
                        }
                    ```
                    ```
                        protected final synchronized AopProxy createAopProxy() {
                            if (!this.active) {
                                activate();
                            }
                            // 使用AopProxyFactory创建AopProxy对象，默认使用的AopProxyFactory为DefaultAopProxyFactory
                            return getAopProxyFactory().createAopProxy(this);
                        }
                    ```   
            - DefaultAopProxyFactory 创建AopProxy对象方法 createAopProxy，返回结果为CglibAopProxy类型对象或者JdkDynamicAopProxy类型对象
                ```
                    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
                        if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
                            Class targetClass = config.getTargetClass();
                            if (targetClass == null) {
                                throw new AopConfigException("TargetSource cannot determine target class: " +
                                        "Either an interface or a target is required for proxy creation.");
                            }
                            if (targetClass.isInterface()) {
                                return new JdkDynamicAopProxy(config);
                            }
                            // 使用Cglib生成代理对象
                            return CglibProxyFactory.createCglibProxy(config);
                        }
                        else {
                            // 使用jdk动态代理生成代理对象
                            return new JdkDynamicAopProxy(config);
                        }
                    }
                ```    
            - JdkDynamicAopProxy 创建代理对象方法 getProxy 分析
                ```
                    public Object getProxy(ClassLoader classLoader) {
                        if (logger.isDebugEnabled()) {
                            logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
                        }
                        Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
                        findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
                        // 使用jdk动态代理创建代理对象：InvocationHandler对象为 JdkDynamicAopProxy 本身
                        return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
                    }
                ```   
            - JdkDynamicAopProxy invoke方法（实现InvocationHandler接口方法）
                ```
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        MethodInvocation invocation;
                        Object oldProxy = null;
                        boolean setProxyContext = false;
                        TargetSource targetSource = this.advised.targetSource;
                        Class targetClass = null;
                        Object target = null;
                        try {
                            // 如果没有配置代理 equals方法、hashCode方法，并且当前调用代理对象方法是 equals方法、hashCode方法，
                            // 则调用 JdkDynamicAopProxy 定义的相应方法
                            ...
                            Object retVal;
                            ...
                            // 获取代理的目标对象、以及目标类信息
                            target = targetSource.getTarget();
                            if (target != null) {
                                targetClass = target.getClass();
                            }
                            // Get the interception chain for this method. 获取当前被调用方法静态匹配的对应通知器链
                            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                            if (chain.isEmpty()) {
                                // 如果该方法没有对应的拦截、增强，直接使用反射调用目标对象的方法
                                retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
                            }
                            else {
                                // 如果有对该方法的增强，则创建ReflectiveMethodInvocation
                                invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                                // 递归调用ReflectiveMethodInvocation的proceed方法：依次应用通知器链中的增强，最终会调用target对象对应方法
                                retVal = invocation.proceed();
                            }
                            ...
                            return retVal;
                        }
                        finally {
                            ...
                        }
                    }
                 ```   
            - 当前目标对象该方法静态匹配到的通知器链  
            ```List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);```
            AdvisedSupport的getInterceptorsAndDynamicInterceptionAdvice方法分析：
                ``` 
                    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class targetClass) {
                        MethodCacheKey cacheKey = new MethodCacheKey(method);
                        List<Object> cached = this.methodCache.get(cacheKey);
                        if (cached == null) {
                            // 首次调用目标对象的该方法触发 静态匹配通知器链 advisorChainFactory 默认为DefaultAdvisorChainFactory类
                            cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                                    this, method, targetClass);
                            this.methodCache.put(cacheKey, cached);
                        }
                        return cached;
                    } 
                ```    
            - DefaultAdvisorChainFactory的getInterceptorsAndDynamicInterceptionAdvice方法分析：
                ```
                    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
                            Advised config, Method method, Class targetClass) {
                        List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
                        // 判断该目标对象的通知器链中是否包含匹配目标类的IntroductionAdvisor
                        boolean hasIntroductions = hasMatchingIntroductions(config, targetClass);
                        // 获取DefaultAdvisorAdapterRegistry类型的单例对象 通知器的适配器注册器，该对象配置了适配器集合、默认设置了MethodBeforeAdviceAdapter、
                        // AfterReturningAdviceAdapter、ThrowsAdviceAdapter
                        AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
                        for (Advisor advisor : config.getAdvisors()) {
                            if (advisor instanceof PointcutAdvisor) {
                                PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
                                // 通知器是否匹配目标对象类
                                if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {
                                    // 从注册器中获取通知器对应的拦截器，相当于把通知器适配成了方法拦截器
                                    MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                                    // 获取通知器对应的方法匹配器
                                    MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                                    if (MethodMatchers.matches(mm, method, targetClass, hasIntroductions)) {
                                        // 当前通知器匹配该对象方法
                                        if (mm.isRuntime()) {
                                            // 如果该通知器配置的方法匹配器是动态的，则封装成InterceptorAndDynamicMethodMatcher对象，
                                            // 用于后续Invocation proceed方法中动态判断拦截器是否匹配
                                            for (MethodInterceptor interceptor : interceptors) {
                                                interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                                            }
                                        }
                                        else {
                                            interceptorList.addAll(Arrays.asList(interceptors));
                                        }
                                    }
                                }
                            } else if (advisor instanceof IntroductionAdvisor) {
                                IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
                                if (config.isPreFiltered() || ia.getClassFilter().matches(targetClass)) {
                                    Interceptor[] interceptors = registry.getInterceptors(advisor);
                                    interceptorList.addAll(Arrays.asList(interceptors));
                                }
                            } else {
                                Interceptor[] interceptors = registry.getInterceptors(advisor);
                                interceptorList.addAll(Arrays.asList(interceptors));
                            }
                        }
                        // 返回的拦截器集合可能包括两种类型：MethodInterceptor和InterceptorAndDynamicMethodMatcher
                        return interceptorList;
                    }
                ```    
            - ReflectiveMethodInvocation的proceed方法分析
                ```
                    public Object proceed() throws Throwable {
                        //	We start with an index of -1 and increment early. currentInterceptorIndex从-1开始
                        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
                            // interceptorsAndDynamicMethodMatchers 为通知器链，通知器链已经依次执行完，最后使用反射调用目标对象对应方法
                            return invokeJoinpoint();
                        }
                        Object interceptorOrInterceptionAdvice =
                            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
                        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
                            // 校验当前通知器是否匹配该对象的方法，InterceptorAndDynamicMethodMatcher包含两个字段：MethodInterceptor
                            // 和MethodMatcher
                            InterceptorAndDynamicMethodMatcher dm =
                                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
                            if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
                                // 如果匹配，则调用对应拦截器、通知器的invoke方法 动态匹配
                                return dm.interceptor.invoke(this);
                            }
                            else {
                                // 动态匹配失败，则跳过该拦截器，调用拦截器上的下一个拦截器 （递归调用）
                                return proceed();
                            }
                        }
                        else {
                            // 方法拦截器类型的，直接调用方法拦截器的invoke方法
                            return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
                        }
                    }
                ```    