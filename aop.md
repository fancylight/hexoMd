---
title: aop
date: 2019-05-25 11:15:39
tags:
- 框架
- spring
- springAop
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
### 概述
- springAop和AspectJ的关系
在spring的aop模块中提供了切面编程实际上都是一种动态代理机制
从实现来说一种是jdk代理,一种是cglib
从语法上讲一种是使用aopalliance定义的advice来实现,一种是spring使用aspectj语法来实现
从spring底层机制来说无论是直接使用aspectj语法还是使用advice类,过程都是将advisor转换成Interceptor的过程
- 真正的AspectJ
spring提供了`spring.aspectJ`模块与其集成,AspectJ使用参考[AspectJ](https://blog.mythsman.com/post/5d301cf2976abc05b34546be/)
### spring中aop的实现
#### 从ProxyFactory开始
{%asset_img ProxyFactory.png%}
- Advised接口表示包含了Interceptors|advice|advices|以及代理接口
- Support都是包含了有效信息,提供组装的接口
例子:
```java
 //该例子使用了jdk代理
  public void test(){
      factory.setTarget(new MyProxyInstance());
        factory.setInterfaces(ProxyInterface.class);
        factory.addAdvice(new BeforeMethod());
        factory.addAdvice(new AfterMethod());
        factory.addAdvice(new ThrowsAd());
        factory.addAdvice(new AroundMethod());
        factory.addAdvice(new IntrodcutionAdvice()); //引介增强
        this.<ProxyInterface>getProxy().test();
    }
```
##### 术语和类
- 术语
	- JoinPoint连接点,实际就是指的方法,其衍生的类有MethodInvocation表示一个方法调用
	- PointCut 切点,描述如何匹配位置,内部包含ClassFilter用来匹配类,MethodMatcher用来匹配方法
	- Advice 增强
	- Advisor 切面 包含了切点和增强,spring用PointCutAdvisor描述
	- 拦截器 就是spring将advisor转换成动态代理时调用的逻辑
##### 类
spring提供了基本五种advice可以通过实现方式不同做一个细分
- 基本的非Introduction的advice
   - MethodBeforeAdvice
     {%asset_img  BeforeAdvice.png%}
   - AfterReturningAdvice
     {%asset_img afterRunnging.png%}
   - ThrowsAdvice
     {%asset_img ThrowsAdvice.png%}
   - MethodInterceptor
     {%asset_img MethodInterceptor.png%}  
- Introduction类型的增强,实际就是添加了代理接口	 
  - 该类在初始化时会获取实现类的直接父接口,也就是用户要增强的接口,源码在构造器部分,不写出来了
{%asset_img DelegatingIntroductionInterceptor.png%}    
可以直观的看到上述5个接口其中环绕和引介本身都属于拦截器
##### advice到advisor的转换
该阶段发生在向ProxyFactory添加advice时,当然用户可以直接添加advisor
{%codeblock lang:java AdvisedSupport%}   
public void addAdvice(Advice advice) throws AopConfigException {
		int pos = this.advisors.size();
		addAdvice(pos, advice);
	}
public void addAdvice(int pos, Advice advice) throws AopConfigException {
		Assert.notNull(advice, "Advice must not be null");
		if (advice instanceof IntroductionInfo) { //实际上我们实现的引介增强提供了要代理的接口
			// We don't need an IntroductionAdvisor for this kind of introduction:
			// It's fully self-describing.
			addAdvisor(pos, new DefaultIntroductionAdvisor(advice, (IntroductionInfo) advice));
		}
		else if (advice instanceof DynamicIntroductionAdvice) { //不能直接添加这种引介
			// We need an IntroductionAdvisor for this kind of introduction.
			throw new AopConfigException("DynamicIntroductionAdvice may only be added as part of IntroductionAdvisor");
		}
		else { //非Introduction类型
			addAdvisor(pos, new DefaultPointcutAdvisor(advice));
		}
	}

//处理IntroductionInfo类型
public void addAdvisor(int pos, Advisor advisor) throws AopConfigException {
		if (advisor instanceof IntroductionAdvisor) {
			validateIntroductionAdvisor((IntroductionAdvisor) advisor);
		}
		addAdvisorInternal(pos, advisor);
	}
//从IntroductionAdvisor获取要增强的接口
private void validateIntroductionAdvisor(IntroductionAdvisor advisor) {
		advisor.validateInterfaces();
		// If the advisor passed validation, we can make the change.
		Class<?>[] ifcs = advisor.getInterfaces();
		for (Class<?> ifc : ifcs) {
			addInterface(ifc);
		}
	}
{%endcodeblock%}

引介基本Advisor
{%asset_img DefaultIntroductionAdvisor.png%}
注意增强advice匹配条件仅仅是类,因此该advisor并不是`PointCut`子类,仅仅是`ClassFilter`子类
{%codeblock lang:java DefaultIntroductionAdvisor%}
//第二个参数一般情况就是用户实现的接口
public DefaultIntroductionAdvisor(Advice advice, @Nullable IntroductionInfo introductionInfo) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
		if (introductionInfo != null) {
			Class<?>[] introducedInterfaces = introductionInfo.getInterfaces(); //获取增加的接口
			if (introducedInterfaces.length == 0) {
				throw new IllegalArgumentException("IntroductionAdviceSupport implements no interfaces");
			}
			for (Class<?> ifc : introducedInterfaces) {
				addInterface(ifc); //将信息的接口添加进set
			}
		}
	}
{%endcodeblock%}

DefaultPointcutAdvisor:典型的PointCutAdvisor,由 内部PointCut决定匹配
{%codeblock lang:java DefaultPointcutAdvisor%}  
public DefaultPointcutAdvisor(Advice advice) {
  this(Pointcut.TRUE, advice); //Pointcut默认匹配所有Class的任意Method
}
{%endcodeblock%}
#### spring支持的代理模式
jdk Proxy和cglib
{%codeblock lang:java ProxyFactory%}
//[1] 外部接口
public Object getProxy() {
		return createAopProxy().getProxy();
	}
//[2] aop代理
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		return getAopProxyFactory().createAopProxy(this); //ProxyFactory本身就是一个配置
	}
	//类DefaultAopProxyFactory,通过代理AopProxyFactory创建AopProxy
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
	//用户没有提供要代理的接口,即config#interfaces为空或者只有SpringProxy接口被代理
	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class<?>[] ifcs = config.getProxiedInterfaces();
		return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
	}

{%endcodeblock%}
- spring提供了两种代理 jdk和cglib,前者的实现导致只能对接口进行代理,后者可以对类进行代理
- 设置ProxyTargetClass=true,表示接受目标是类时,直接代理和代理接口一样;一般来当传递的target为类,这样就是开启cglib.
- <aop:aspectj-autoproxy proxy-target-class="true"/> | @EnableAspectJAutoProxy(proxyTargetClass = true)
##### JDK proxy
###### 开启jdk代理
- jdk代理过程中,将target作为最终原本调用的代理对象,可以参看proxy代理部分,也就是说该对象不指定或者并不是接口对象,那么必定会导致最终调用异常
- 只有主动调用setInterfaces才能启用jdk代理,否则就是cglib代理,当然除去代理对象本身就是个接口(会抛出异常),或者代理对象本身是个Proxy类(即jdk proxy生成的类)
```java
ProxyFactory factory=new ProxyFactory();
    //关于spring jdk代理
    private <T> T getProxy(){
        return  (T)factory.getProxy();
    }
    @Test
    public void test0(){
        factory.setTarget(new NoInterface());
        factory.setInterfaces(new Class[]{ProxyInterface.class,ProxyInterface2.class});
        this.<ProxyInterface>getProxy().test(); //此处会跳出异常
    }
```
###### 调用逻辑
- 注意该类本身就是一个InvocationHandler,因此其invoke函数就是支持代理执行的逻辑
{%codeblock lang:java JdkDynamicAopProxy%}
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
@Override
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

	@Override
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isTraceEnabled()) {
			logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
		}
		//advised即ProxyFactory本身,该函数生成了该代理模式下生成的所有接口
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		//确定是否重写了equals和hash方法
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		//jdk代理
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
}
{%endcodeblock%}
- spring如何获取代理接口的
	- 这个函数说明的问题是,想使用jdk proxy的动作是主动添加interface,而不是仅仅传递target,spring并没有主动解析目标类的父接口
{%codeblock lang:java AopProxyUtils%}
static Class<?>[] completeProxiedInterfaces(AdvisedSupport advised, boolean decoratingProxy) {
		Class<?>[] specifiedInterfaces = advised.getProxiedInterfaces();
		//当用户没有提供代理接口时,检查是否代理对象是一个proxy
		if (specifiedInterfaces.length == 0) {
			// No user-specified interfaces: check whether target class is an interface.
			Class<?> targetClass = advised.getTargetClass();
			if (targetClass != null) {
				if (targetClass.isInterface()) {
					advised.setInterfaces(targetClass);
				}
				else if (Proxy.isProxyClass(targetClass)) {
					advised.setInterfaces(targetClass.getInterfaces());
				}
				specifiedInterfaces = advised.getProxiedInterfaces();
			}
		}
		boolean addSpringProxy = !advised.isInterfaceProxied(SpringProxy.class);
		boolean addAdvised = !advised.isOpaque() && !advised.isInterfaceProxied(Advised.class);
		boolean addDecoratingProxy = (decoratingProxy && !advised.isInterfaceProxied(DecoratingProxy.class));
		int nonUserIfcCount = 0;
		//spring会主动添加三个代理接口 SpringProxy   | Advised   |DecoratingProxy
		if (addSpringProxy) {
			nonUserIfcCount++;
		}
		if (addAdvised) {
			nonUserIfcCount++;
		}
		if (addDecoratingProxy) {
			nonUserIfcCount++;
		}
		Class<?>[] proxiedInterfaces = new Class<?>[specifiedInterfaces.length + nonUserIfcCount];
		System.arraycopy(specifiedInterfaces, 0, proxiedInterfaces, 0, specifiedInterfaces.length);
		int index = specifiedInterfaces.length;
		if (addSpringProxy) {
			proxiedInterfaces[index] = SpringProxy.class;
			index++;
		}
		if (addAdvised) {
			proxiedInterfaces[index] =  .class;
			index++;
		}
		if (addDecoratingProxy) {
			proxiedInterfaces[index] = DecoratingProxy.class;
		}
		return proxiedInterfaces;
	}
{%endcodeblock%}
- JdkDynamicAopProxy#invoke,这是代理逻辑
{%codeblock lang:java JdkDynamicAopProxy%}
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;
    //此处体现了目标源
		TargetSource targetSource = this.advised.targetSource;
		Object target = null;

		try {
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
		       //没有重写equals,调用Object#equals
				return equals(args[0]);
			}
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				//同上
				return hashCode();
			}
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// 如果调用是DecoratingProxy#getDeclaringClass则调用如下函数
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// 这个函数的逻辑实际就是调用method.invoke,jdk反射调用函数
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			//获取代理目标
			target = targetSource.getTarget();
			Class<?> targetClass = (target != null ? target.getClass() : null);

			//获取拦截链,这是核心逻辑之一
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		     //如果拦截链为空,则调用反射
			if (chain.isEmpty()) {
				// We can skip creating a MethodInvocation: just invoke the target directly
				// Note that the final invoker must be an InvokerInterceptor so we know it does
				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				//创建一个使用拦截逻辑的方法调用,核心逻辑之一
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target &&
					returnType != Object.class && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}
{%endcodeblock%}
  - 说明此处的`targetSource`,如果使用spring中jdk代理不提供源|或者目标源和代理接口并非实现关系,那么当真正函数调用时spring会抛出异常,
  这里实际上就是我在[jdk动态代理](/2019/05/22/java动态代理/#总结)提及的若不提供源,那么代理无意义.
- 获取拦截链
这里的逻辑剥离了advisor,其中切点用来筛选,而advice用来创建拦截链
{%codeblock lang:java AdvisedSupport%}
//缓存机制  method-拦截链
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
		MethodCacheKey cacheKey = new MethodCacheKey(method);
		List<Object> cached = this.methodCache.get(cacheKey);
		if (cached == null) {
			cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);
			this.methodCache.put(cacheKey, cached);
		}
		return cached;
	}
public class DefaultAdvisorChainFactory implements AdvisorChainFactory, Serializable {

	@Override
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {

		//AdvisorAdapterRegistry用来创建拦截器
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
		Advisor[] advisors = config.getAdvisors();
		List<Object> interceptorList = new ArrayList<>(advisors.length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		Boolean hasIntroductions = null;

		for (Advisor advisor : advisors) {
			if (advisor instanceof PointcutAdvisor) { ////默认的前置  后置 抛出 环绕 都时这种类型
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) { //做一次类型匹配
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher(); //获取方法匹配器
					boolean match;
					if (mm instanceof IntroductionAwareMethodMatcher) { //不清楚的方法匹配其类型
						if (hasIntroductions == null) {
							hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
						}
						match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
					}
					else {
						match = mm.matches(method, actualClass); //匹配一次
					}
					if (match) {
						MethodInterceptor[] interceptors = registry.getInterceptors(advisor); //创造拦截器,返回数组说明用户实现的部分可以属于多个增强
						if (mm.isRuntime()) { //动态切点则创建动态拦截器
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) { //引介增强
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}

	/**
	 * Determine whether the Advisors contain matching introductions.
	 */
	private static boolean hasMatchingIntroductions(Advisor[] advisors, Class<?> actualClass) {
		for (Advisor advisor : advisors) {
			if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (ia.getClassFilter().matches(actualClass)) {
					return true;
				}
			}
		}
		return false;
	}

}
//拦截器创建
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
		List<MethodInterceptor> interceptors = new ArrayList<>(3);
		Advice advice = advisor.getAdvice();
		if (advice instanceof MethodInterceptor) { //环绕 引介本身就是拦截器
			interceptors.add((MethodInterceptor) advice);
		}
		for (AdvisorAdapter adapter : this.adapters) { //其他三种通过适配器创建
			if (adapter.supportsAdvice(advice)) {
				interceptors.add(adapter.getInterceptor(advisor));
			}
		}
		if (interceptors.isEmpty()) {
			throw new UnknownAdviceTypeException(advisor.getAdvice());
		}
		return interceptors.toArray(new MethodInterceptor[0]);
	}
}
{%endcodeblock%}
- 拦截器调用逻辑
{%codeblock lang:java 拦截器调用%}
invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				retVal = invocation.proceed();
//创建ReflectiveMethodInvocation
public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {
protected ReflectiveMethodInvocation(
			Object proxy, @Nullable Object target, Method method, @Nullable Object[] arguments,
			@Nullable Class<?> targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {

		this.proxy = proxy; //代理对象
		this.target = target; //代理目标
		this.targetClass = targetClass; //目标class类型
		this.method = BridgeMethodResolver.findBridgedMethod(method); //方法
		this.arguments = AopProxyUtils.adaptArgumentsIfNecessary(method, arguments); //参数
		this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers; //该方法对应的拦截链
	}
}
//拦截器的调用
	public Object proceed() throws Throwable {
		//	当拦截器调用结束则使用反射,去调用函数本身,实际就是递归的返回条件
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) { //动态切点就会匹配到这里
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) { //做一次动态匹配,如果匹配成功就调用
				return dm.interceptor.invoke(this); //调用拦截器中的增强
			}
			else {
				//如果动态拦截器失败则进行下一个拦截
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}		
{%endcodeblock%}
此处对于非动态拦截器采取了一种十分有趣的写法,具体实现不好描述,简单的理解为递归调用
默认的5种类型拦截器放置顺序都能保证正确的执行,只是具体顺序有些不同
```java
     factory.setTarget(new MyProxyInstance());
        factory.setInterfaces(ProxyInterface.class);
        factory.addAdvice(new AroundMethod());
        factory.addAdvice(new AfterMethod());
        factory.addAdvice(new BeforeMethod());
        this.<ProxyInterface>getProxy().test();
//结果:
//test环绕前
//test前执行
//采取jdk proxy
//test执行后
//test环绕后
	  factory.setTarget(new MyProxyInstance());
        factory.setInterfaces(ProxyInterface.class);
        factory.addAdvice(new ThrowsAd());
		factory.addAdvice(new AfterMethod());
        factory.addAdvice(new AroundMethod());
        this.<ProxyInterface>getProxy().test();
//结果
test环绕前
test前执行
采取jdk proxy
test环绕后
test执行后
```
###### 拦截器
```java
//前置拦截器
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {
@Override
	public Object invoke(MethodInvocation mi) throws Throwable { //mi表示方法调用
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis()); //调用用户实现
		return mi.proceed();//可以理解为递归,等待此处循环调用结束,因此before实现的部分一定会在方法调用前被调用
	}
}
//后置拦截
public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {
@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		Object retVal = mi.proceed(); //继续递归执行
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());//只有递归结束后才能调用,因此该逻辑必然在函数调用后才能被调用
		return retVal;
	}
}
//环绕拦截实际是一个普通的拦截器
public interface MethodInterceptor extends Interceptor {
	Object invoke(MethodInvocation invocation) throws Throwable;
}
//用户实现
public class AroundMethod implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method= invocation.getMethod();
        Object arg[]= invocation.getArguments();
        System.out.println(method.getName()+"环绕前");
//        Object re=method.invoke(invocation.getThis(),arg);  这句代码就会导致递归返回,如果该method拦截链没有调用完,那么就会到此为止
        var re= invocation.proceed(); //如此调用就会继续进入拦截链逻辑
        System.out.println(method.getName()+"环绕后");
        return re;
    }
}
```
- 异常拦截器,由于ThrowsAdvice是个标记接口,因此在该拦截器中定义用户能够实现的函数模式
`public void afterThrowing(Exception ex)`
`public void afterThrowing(RemoteException)`
`public void afterThrowing(Method method, Object[] args, Object target, Exception ex)`
`public void afterThrowing(Method method, Object[] args, Object target, ServletException ex)`
函数名必须时`afterThrowing`
{%codeblock lang:java ThrowsAdviceInterceptor%}
public class ThrowsAdviceInterceptor implements MethodInterceptor, AfterAdvice {
public ThrowsAdviceInterceptor(Object throwsAdvice) {
		Assert.notNull(throwsAdvice, "Advice must not be null");
		this.throwsAdvice = throwsAdvice;

		Method[] methods = throwsAdvice.getClass().getMethods();
		for (Method method : methods) {
			if (method.getName().equals(AFTER_THROWING) &&
					(method.getParameterCount() == 1 || method.getParameterCount() == 4)) {//检查函数名和参数
				Class<?> throwableParam = method.getParameterTypes()[method.getParameterCount() - 1];
				if (Throwable.class.isAssignableFrom(throwableParam)) { //若符合
					// An exception handler to register...
					this.exceptionHandlerMap.put(throwableParam, method); //缓存 exception-method的 map
					if (logger.isDebugEnabled()) {
						logger.debug("Found exception handler method on throws advice: " + method);
					}
				}
			}
		}

		if (this.exceptionHandlerMap.isEmpty()) {
			throw new IllegalArgumentException(
					"At least one handler method must be found in class [" + throwsAdvice.getClass() + "]");
		}
	}
}
//拦截链调用
public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed(); //递归逻辑
		}
		catch (Throwable ex) { //当递归返回,并捕获异常后处理
			Method handlerMethod = getExceptionHandler(ex); //匹配对应异常的处理函数
			if (handlerMethod != null) {
				invokeHandlerMethod(mi, ex, handlerMethod); //调用用户实现的handlerMethod
			}
			throw ex;
		}
	}
private void invokeHandlerMethod(MethodInvocation mi, Throwable ex, Method method) throws Throwable {
		Object[] handlerArgs;
		if (method.getParameterCount() == 1) { //处理handler会用到的参数
			handlerArgs = new Object[] {ex};
		}
		else {
			handlerArgs = new Object[] {mi.getMethod(), mi.getArguments(), mi.getThis(), ex};
		}
		try {
			method.invoke(this.throwsAdvice, handlerArgs); //调用异常处理handler
		}
		catch (InvocationTargetException targetEx) {
			throw targetEx.getTargetException();
		}
	}
{%endcodeblock%}
- 引介拦截器
{%codeblock lang:java DelegatingIntroductionInterceptor%}
public class DelegatingIntroductionInterceptor extends IntroductionInfoSupport
		implements IntroductionInterceptor {
//链接链调用		
public Object invoke(MethodInvocation mi) throws Throwable {
		if (isMethodOnIntroducedInterface(mi)) { //判断此次method是不是增强的函数
		    //如果是则调用反射,只是反射的instance是this,就能调用用户实现增强
			Object retVal = AopUtils.invokeJoinpointUsingReflection(this.delegate, mi.getMethod(), mi.getArguments());

			//如果返回的是this则中断递归逻辑
			if (retVal == this.delegate && mi instanceof ProxyMethodInvocation) {
				Object proxy = ((ProxyMethodInvocation) mi).getProxy();
				if (mi.getMethod().getReturnType().isInstance(proxy)) {
					retVal = proxy;
				}
			}
			return retVal;
		}
        //递归逻辑
		return doProceed(mi);
	}		
}
{%endcodeblock%}
###### 切点
切点可以分为 静态切点 | 动态切点 | 流程切点 | 以及默认的全匹配切点|复合切点|匹配name的切点
切点实际上就是实现了PointCut,MethodMathcher的类
```java
//methodMathcher
public interface MethodMatcher {
//创建拦截链时调用
boolean matches(Method method, Class<?> targetClass);
//该条件反应了该切点是不是动态的,动态切点会在每次函数调用再做一次匹配
	boolean isRuntime();
//动态匹配条件
boolean matches(Method method, Class<?> targetClass, Object... args);
//默认的全匹配方法拦截器
MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}
//pointcut
public interface Pointcut {
//拦截链条件之一
ClassFilter getClassFilter(); //存在默认的全Class匹配器
//
	MethodMatcher getMethodMatcher();
//默认的全匹配切点
Pointcut TRUE = TruePointcut.INSTANCE;
}
```


- spring提供了一个静态advisor,可以直接使用StaticMethodMatcherPointcutAdvisor,或者我们实现静态切点StaticMethodMatcherPointcut使用默认defaultAdvisor
- 动态则提供了一个切点
```java
public abstract class DynamicMethodMatcherPointcut extends DynamicMethodMatcher implements Pointcut {

	@Override
	public ClassFilter getClassFilter() {
		return ClassFilter.TRUE;
	}

	@Override
	public final MethodMatcher getMethodMatcher() {
		return this;
	}

}
//用户实现
/**
 * 实际上这是一个切点
 * @see org.springframework.aop.MethodMatcher#matches(Method, Class, Object...)
 * 第三个参数表示当函数运行时判断形参,也就是说该切面判定在于运行期,而不是Context解析的过程中
 */
public class DynamicPointcut extends DynamicMethodMatcherPointcut {
    private static List<Integer> list;
    static {
        list = Stream.of(1,2,3).collect(Collectors.toList());
    }
    //匹配类
    @Override
    public ClassFilter getClassFilter() {
        return super.getClassFilter(); // 默认匹配所有,这个测试中只有该切面只对应了一个target,所以我不写了
    }
    //匹配函数
    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        //首先因该判断类,省略
        //匹配函数,省略,因为只有一个
        return super.matches(method, targetClass);
    }
	//该函数会在method调用的时候再做一次,形参args的匹配,也就是说每次函数调用传递的参数不同,也许就不满足条件,这就是动态匹配的意思
    @Override
    public boolean matches(Method method, Class<?> targetClass, Object... args) {
        //此刻就能对于函数进行动态匹配,也就是说在参数不同的情况下对于目标做出不同的处理,默认不进行处理
        return list.contains(args[0]);
    }
}
```
- 流程切点	ControlFlowPointcut
{%codeblock lang:java 流程切点%}
//匹配条件有 类 和 方法
public ControlFlowPointcut(Class<?> clazz, @Nullable String methodName) {
		Assert.notNull(clazz, "Class must not be null");
		this.clazz = clazz;
		this.methodName = methodName;
	}
public boolean isRuntime() {
		return true;
	}
//动态匹配条件
public boolean matches(Method method, Class<?> targetClass, Object... args) {
		this.evaluations++;
        //获取调用方法栈,当类和方法匹配时启动代理
		for (StackTraceElement element : new Throwable().getStackTrace()) {
			if (element.getClassName().equals(this.clazz.getName()) &&
					(this.methodName == null || element.getMethodName().equals(this.methodName))) {
				return true;
			}
		}
		return false;
	}
{%endcodeblock%}
- 复合切点
{% codeblock lang:java ComposablePointcut%}
public class ComposablePointcut implements Pointcut, Serializable {
//默认为全匹配
public ComposablePointcut(MethodMatcher methodMatcher) {
		Assert.notNull(methodMatcher, "MethodMatcher must not be null");
		this.classFilter = ClassFilter.TRUE;
		this.methodMatcher = methodMatcher;
	}
//交
public ComposablePointcut intersection(ClassFilter other) {
		this.classFilter = ClassFilters.intersection(this.classFilter, other);
		return this;
	}
public ComposablePointcut intersection(MethodMatcher other) {
		this.methodMatcher = MethodMatchers.intersection(this.methodMatcher, other);
		return this;
	}
public ComposablePointcut intersection(Pointcut other) {
		this.classFilter = ClassFilters.intersection(this.classFilter, other.getClassFilter());
		this.methodMatcher = MethodMatchers.intersection(this.methodMatcher, other.getMethodMatcher());
		return this;
	}
}
{%endcodeblock%}
如何实现 交逻辑 | 并逻辑类似
{%codeblock lang:java %}
private static class IntersectionMethodMatcher implements MethodMatcher, Serializable {
protected final MethodMatcher mm1;

		protected final MethodMatcher mm2;

		public IntersectionMethodMatcher(MethodMatcher mm1, MethodMatcher mm2) {
			Assert.notNull(mm1, "First MethodMatcher must not be null");
			Assert.notNull(mm2, "Second MethodMatcher must not be null");
			this.mm1 = mm1;
			this.mm2 = mm2;
		}
		//静态方法匹配条件改为&&满足
		@Override
		public boolean matches(Method method, Class<?> targetClass) {
			return (this.mm1.matches(method, targetClass) && this.mm2.matches(method, targetClass));
		}
        //是否为动态则取决于存在就进行动态匹配
		@Override
		public boolean isRuntime() {
			return (this.mm1.isRuntime() || this.mm2.isRuntime());
		}
        //动态匹配条件
		@Override
		public boolean matches(Method method, Class<?> targetClass, Object... args) {
			// Because a dynamic intersection may be composed of a static and dynamic part,
			// we must avoid calling the 3-arg matches method on a dynamic matcher, as
			// it will probably be an unsupported operation.
			boolean aMatches = (this.mm1.isRuntime() ?
					this.mm1.matches(method, targetClass, args) : this.mm1.matches(method, targetClass));
			boolean bMatches = (this.mm2.isRuntime() ?
					this.mm2.matches(method, targetClass, args) : this.mm2.matches(method, targetClass));
			return aMatches && bMatches;
		}
}
{%endcodeblock%}
- 引介(引介不是匹配方法而是类),因此并不是实现PointCut,在第一部分有 advice->advisor的转换中就提到了引介切面DefaultIntroductionAdvisor
引介增强DelegatingIntroductionInterceptor,本身就属于一个拦截器,用户实现该接口,直接添加该advice,spring会自动创建DefaultIntroductionAdvisor
我们也可以手动创建
{%asset_img DefaultIntroductionAdvisor.png%}
我们可以看见该切面并不是Pointcut也不是MethodMatcher
- A B类循环引用,最终问题情况
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <!--关于早期引用代理问题-->
    <bean id="a" class="ioc.problem.A" autowire="byName"/>
    <bean id="b" class="ioc.problem.B" autowire="byName"/>

    <bean id="flowHelp" class="aop.jdkProxy.FlowHelp"/>
    <bean id="myInstance" class="aop.jdkProxy.MyProxyInstance"/>
    <bean id="advice" class="aop.BeforeMethod"/>
    <bean id="myInstanceProxy" class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator"
          p:proxyTargetClass="true"
          p:interceptorNames="advice" p:beanNames="a"/>
</beans>

```
```java
 @Test
    public void test3(){
        xml.loadBeanDefinitions("aop2.xml");
        beanFactory.addBeanPostProcessor((BeanPostProcessor) beanFactory.getBean("myInstanceProxy"));
        beanFactory.getBean("myInstance",MyProxyInstance.class).test();
        //早期引用代理
        A a= (A) beanFactory.getBean("a");
    }
	//结果并不是一个循环,而是(代理A)->null

//-----------------------------------------------------

//原因:
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
			//当早期对象循环结束回到当前bean时
		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);  //会从缓存最终获取早期bean
			if (earlySingletonReference != null) { //若早期bean和当前不同则会导致覆盖
				if (exposedObject == bean) {
				//不同的原因就在于smart接口创建了一个早期代理对象,反正最终结果很奇怪
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
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}
	}
```
#### springAop技术总结
##### 底层实现原理
{%asset_img aop技术.png%}
- 字节码操作技术: ASM JDKPROXY JAVASIST
- 不使用`Instrumentation`的情况基本都是运行时织入
  - 运行织入技术如`cglib` ,'jdkProxy'都会生成目标类的子类代理对象,这是为了解决类加载器重复加载问题.
- 关于上述几个框架,我暂时不做源码分析,使用参考[字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)  
- spring中无论使用jdk代理还是cglib实际上都是将advice->advisor,分别在InvocationHandler | cglib的MethodIntertor中实现拦截链
##### 语法角度
{%asset_img springAop语法.png springAop语法%}
- spring支持接口或类代理,分别使用了jdk和cglib实现
- 为了拓展语法,spring也支持了aspectJ语法,在不使用`LTW`技术的情况下,原理还是上述两种
##### cglib在spring中的源码
{%asset_img ObjenesisCglibAopProxy.png%}
这个大体框架和JdkDynamicAopProxy是类似的
{%codeblock lang:java CglibAopProxy%}
public CglibAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
		this.advisedDispatcher = new AdvisedDispatcher(this.advised);
	}
//实际创建代理
public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isTraceEnabled()) {
			logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
		}

		try {
			Class<?> rootClass = this.advised.getTargetClass(); //代理源对象
			Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

			Class<?> proxySuperClass = rootClass;
			if (ClassUtils.isCglibProxyClass(rootClass)) { //处理接口
				proxySuperClass = rootClass.getSuperclass();
				Class<?>[] additionalInterfaces = rootClass.getInterfaces();
				for (Class<?> additionalInterface : additionalInterfaces) {
					this.advised.addInterface(additionalInterface);
				}
			}

			// Validate the class, writing log messages as necessary.
			validateClassIfNecessary(proxySuperClass, classLoader);

			// Configure CGLIB Enhancer... cglib Enhancer设置
			Enhancer enhancer = createEnhancer();
			if (classLoader != null) {
				enhancer.setClassLoader(classLoader);
				if (classLoader instanceof SmartClassLoader &&
						((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
					enhancer.setUseCache(false);
				}
			}
			enhancer.setSuperclass(proxySuperClass);
			enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

			Callback[] callbacks = getCallbacks(rootClass); //创建Callbcak
			Class<?>[] types = new Class<?>[callbacks.length];
			for (int x = 0; x < types.length; x++) {
				types[x] = callbacks[x].getClass();
			}
			// fixedInterceptorMap only populated at this point, after getCallbacks call above
			enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
			enhancer.setCallbackTypes(types);

			// Generate the proxy class and create a proxy instance.
			return createProxyClassAndInstance(enhancer, callbacks);
		}
		catch (CodeGenerationException | IllegalArgumentException ex) {
			throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
					": Common causes of this problem include using a final class or a non-visible class",
					ex);
		}
		catch (Throwable ex) {
			// TargetSource.getTarget() failed
			throw new AopConfigException("Unexpected AOP exception", ex);
		}
	}
//---------------------------getCallbacks---------------------------------
private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
		// Parameters used for optimization choices...
		boolean exposeProxy = this.advised.isExposeProxy();
		boolean isFrozen = this.advised.isFrozen();
		boolean isStatic = this.advised.getTargetSource().isStatic();

		// Choose an "aop" interceptor (used for AOP calls).
    //DynamicAdvisedInterceptor 是cglib中的MethodInteceptor实现类比于jdkProxy的InvocationHandler
		Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

		// Choose a "straight to target" interceptor. (used for calls that are
		// unadvised but can return this). May be required to expose the proxy.
		Callback targetInterceptor;
		if (exposeProxy) {
			targetInterceptor = (isStatic ?
					new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
		}
		else {
			targetInterceptor = (isStatic ?
					new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
		}

		// Choose a "direct to target" dispatcher (used for
		// unadvised calls to static targets that cannot return this).
		Callback targetDispatcher = (isStatic ?
				new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());

		Callback[] mainCallbacks = new Callback[] {
				aopInterceptor,  // for normal advice
				targetInterceptor,  // invoke target without considering advice, if optimized
				new SerializableNoOp(),  // no override for methods mapped to this
				targetDispatcher, this.advisedDispatcher,
				new EqualsInterceptor(this.advised),
				new HashCodeInterceptor(this.advised)
		};

		Callback[] callbacks;

		// If the target is a static one and the advice chain is frozen,
		// then we can make some optimizations by sending the AOP calls
		// direct to the target using the fixed chain for that method.
		if (isStatic && isFrozen) {
			Method[] methods = rootClass.getMethods();
			Callback[] fixedCallbacks = new Callback[methods.length];
			this.fixedInterceptorMap = new HashMap<>(methods.length);

			// TODO: small memory optimization here (can skip creation for methods with no advice)
			for (int x = 0; x < methods.length; x++) {
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(methods[x], rootClass);
				fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
						chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
				this.fixedInterceptorMap.put(methods[x].toString(), x);
			}

			// Now copy both the callbacks from mainCallbacks
			// and fixedCallbacks into the callbacks array.
			callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
			System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
			System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
			this.fixedInterceptorOffset = mainCallbacks.length;
		}
		else {
			callbacks = mainCallbacks;
		}
		return callbacks;
	}
//---------------------------------DynamicAdvisedInterceptor----------------------------------------
private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

		private final AdvisedSupport advised;

		public DynamicAdvisedInterceptor(AdvisedSupport advised) {
			this.advised = advised;
		}

		@Override
		@Nullable
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				// Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
				target = targetSource.getTarget();
				Class<?> targetClass = (target != null ? target.getClass() : null);
        //注意这里 获取拦截链,这里的逻辑和jdk就是相同的了,后边没什么值得分析的了
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				// Check whether we only have one InvokerInterceptor: that is,
				// no real advice, but just reflective invocation of the target.
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					// We can skip creating a MethodInvocation: just invoke the target directly.
					// Note that the final invoker must be an InvokerInterceptor, so we know
					// it does nothing but a reflective operation on the target, and no hot
					// swapping or fancy proxying.
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// We need to create a method invocation...
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null && !targetSource.isStatic()) {
					targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}


{%endcodeblock%}
##### Aspectj使用
- AspectJProxyFactory使用
```
public void test2(){
        AspectJProxyFactory aspectJProxyFactory = new AspectJProxyFactory();
        aspectJProxyFactory.setTarget(new AocTarget2());
        aspectJProxyFactory.addAspect(AnnotationAspect.class);
        AocTarget2 aocTarget2 =  aspectJProxyFactory.getProxy();
        aocTarget2.annotationAspectTest();
    }
```
- 原理
{%codeblock lang:java AspectJProxyFactory%}
//创建advisor的工厂
private final AspectJAdvisorFactory aspectFactory = new ReflectiveAspectJAdvisorFactory();

public void addAspect(Class<?> aspectClass) {
		String aspectName = aspectClass.getName();
    //关于Metadata是 ascpetTool中的概念
		AspectMetadata am = createAspectMetadata(aspectClass, aspectName);
		MetadataAwareAspectInstanceFactory instanceFactory = createAspectInstanceFactory(am, aspectClass, aspectName);
    //根据源信息创建advisor
		addAdvisorsFromAspectInstanceFactory(instanceFactory);
	}
  //-------------------addAdvisorsFromAspectInstanceFactory------------------------------
  private void addAdvisorsFromAspectInstanceFactory(MetadataAwareAspectInstanceFactory instanceFactory) {
		List<Advisor> advisors = this.aspectFactory.getAdvisors(instanceFactory);
		Class<?> targetClass = getTargetClass();
		Assert.state(targetClass != null, "Unresolvable target class");
		advisors = AopUtils.findAdvisorsThatCanApply(advisors, targetClass);
		AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary(advisors);
		AnnotationAwareOrderComparator.sort(advisors);
		addAdvisors(advisors);
	}

//-------------------------------  ReflectiveAspectJAdvisorFactory-------------------------------------
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
    //Aspect类信息
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new ArrayList<>();
    //遍历Aspect类中函数,除了被标记为@PointCut注解的
		for (Method method : getAdvisorMethods(aspectClass)) {
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
    // 引介处理
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}

//---------------getAdvisorMethods---------------------
private List<Method> getAdvisorMethods(Class<?> aspectClass) {
		final List<Method> methods = new ArrayList<>();
		ReflectionUtils.doWithMethods(aspectClass, method -> {
			// Exclude pointcuts
			if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
				methods.add(method);
			}
		});
		methods.sort(METHOD_COMPARATOR);
		return methods;
	}
//------------------getAdvisor----------------------------------
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) {

		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
    //根据符合条件的函数创建切点
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}
    //创建advisor,这标准的PointcutAdvisor
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}  
//--------------------getDeclareParentsAdvisor------------------------
private Advisor getDeclareParentsAdvisor(Field introductionField) {
  //根据aspect中的属性,创建引介advisor
		DeclareParents declareParents = introductionField.getAnnotation(DeclareParents.class);
		if (declareParents == null) {
			// Not an introduction field
			return null;
		}

		if (DeclareParents.class == declareParents.defaultImpl()) {
			throw new IllegalStateException("'defaultImpl' attribute must be set on DeclareParents");
		}

		return new DeclareParentsAdvisor(
				introductionField.getType(), declareParents.value(), declareParents.defaultImpl());
	}  
{%endcodeblock%}
