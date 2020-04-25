---
title: spring-cache
tag:
- 框架
- spring
- 缓存
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
description: spring常见组件
---
### spring缓存
`spring-context`模块下功能,与之相关的jsr为jsr107
#### 配置
- xml配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:cache="http://www.springframework.org/schema/cache" xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">
    <context:component-scan base-package="cache"/>
    <cache:annotation-driven/>
    <!--定义manager-->
    <!--id必须为cacheManager-->
    <bean id="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
        <property name="caches">
            <set>
                <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="name"/>
            </set>
        </property>
    </bean>
</beans>
```
- java
```java
package cache;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class CacheTestService {
    @Cacheable("name")
    public User getUser(String name) {
        System.out.println("缓存未命中");
        return new User();
    }
}
```
#### 注解
##### Cacheable
一般使用这个使用缓存
- `value`和`cacheNames`表示使用哪一个`Cache`
- `key`和`keyGenerator`表示该使用的键生成器,若为空,源码会创建`SimpleKeyGenerator`,支持el
- `cacheManager`和`cacheResolver`实际都是处理`CacheManager`的,bf中必须有一个叫做`cacheManager`的,否则异常
- `condition`:表示满足条件执行,可以使用spring el表达式
- `sync`表示加锁执行
- `unless`:支持el,若当发生未命中时,若该条件和形参匹配,则不会发生缓存行为
##### CacheEvict
表示对于缓存进行删除,大部分属性和`Cacheable`一样
- `beforeInvocation`:表示删除操作是否在用户函数执行前就调用
- `allEntries`:`true`表示删除对应`Cache`中所有
##### CachePut
当存在返回值时会进行将返回值尝试加入缓存
- `unless`:支持el,若当发生未命中时,若该条件和返回值匹配,则不会发生缓存行为
表示执行完用户逻辑,会尝试将返回值加入缓存
##### 配置源码
- 解析器源码
{%codeblock lang:java AnnotationDrivenCacheBeanDefinitionParser%}
class AnnotationDrivenCacheBeanDefinitionParser implements BeanDefinitionParser {
private static final String CACHE_ASPECT_CLASS_NAME =
    "org.springframework.cache.aspectj.AnnotationCacheAspect";

private static final String JCACHE_ASPECT_CLASS_NAME =
    "org.springframework.cache.aspectj.JCacheCacheAspect";

//---------------这部分确认是否支持jsr107和jcache
private static final boolean jsr107Present;

private static final boolean jcacheImplPresent;

static {
  ClassLoader classLoader = AnnotationDrivenCacheBeanDefinitionParser.class.getClassLoader();
  jsr107Present = ClassUtils.isPresent("javax.cache.Cache", classLoader);
  jcacheImplPresent = ClassUtils.isPresent(
      "org.springframework.cache.jcache.interceptor.DefaultJCacheOperationSource", classLoader);
}
//=======================解析==========================
public BeanDefinition parse(Element element, ParserContext parserContext) {
        String mode = element.getAttribute("mode");
        if ("aspectj".equals(mode)) { //使用aspectJ,这里说的是spring-aspectJ模块
            // mode="aspectj"
            registerCacheAspect(element, parserContext);
        }
        else { //使用spring aop
            // mode="proxy"
            registerCacheAdvisor(element, parserContext);
        }

        return null;
    }
}

private void registerCacheAdvisor(Element element, ParserContext parserContext) {
        AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element); //设置一个自动代理器InfrastructureAdvisorAutoProxyCreator
        SpringCachingConfigurer.registerCacheAdvisor(element, parserContext);//内部类实现
        if (jsr107Present && jcacheImplPresent) {
            JCacheCachingConfigurer.registerCacheAdvisor(element, parserContext);
        }
    }
//============================SpringCachingConfigurer内部类=======================

private static class SpringCachingConfigurer {

        private static void registerCacheAdvisor(Element element, ParserContext parserContext) {
            if (!parserContext.getRegistry().containsBeanDefinition(CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)) {
                Object eleSource = parserContext.extractSource(element);

        //分别创建了AnnotationCacheOperationSource
        //CacheInterceptor
        //BeanFactoryCacheOperationSourceAdvisor
                // Create the CacheOperationSource definition.
                RootBeanDefinition sourceDef = new RootBeanDefinition("org.springframework.cache.annotation.AnnotationCacheOperationSource");
                sourceDef.setSource(eleSource);
                sourceDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
                String sourceName = parserContext.getReaderContext().registerWithGeneratedName(sourceDef);

                // Create the CacheInterceptor definition.
                RootBeanDefinition interceptorDef = new RootBeanDefinition(CacheInterceptor.class);
                interceptorDef.setSource(eleSource);
                interceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
         //设置内部的CahceResovler,实际上就返回了一个pv#value 为runtimeReferen,没有类型,只有名称
                parseCacheResolution(element, interceptorDef, false);
                parseErrorHandler(element, interceptorDef);
                CacheNamespaceHandler.parseKeyGenerator(element, interceptorDef);
                interceptorDef.getPropertyValues().add("cacheOperationSources", new RuntimeBeanReference(sourceName));
                String interceptorName = parserContext.getReaderContext().registerWithGeneratedName(interceptorDef);

                // Create the CacheAdvisor definition.
                RootBeanDefinition advisorDef = new RootBeanDefinition(BeanFactoryCacheOperationSourceAdvisor.class);
                advisorDef.setSource(eleSource);
                advisorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
                advisorDef.getPropertyValues().add("cacheOperationSource", new RuntimeBeanReference(sourceName));
                advisorDef.getPropertyValues().add("adviceBeanName", interceptorName);
                if (element.hasAttribute("order")) {
                    advisorDef.getPropertyValues().add("order", element.getAttribute("order"));
                }
                parserContext.getRegistry().registerBeanDefinition(CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME, advisorDef);

                CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), eleSource);
                compositeDef.addNestedComponent(new BeanComponentDefinition(sourceDef, sourceName));
                compositeDef.addNestedComponent(new BeanComponentDefinition(interceptorDef, interceptorName));
                compositeDef.addNestedComponent(new BeanComponentDefinition(advisorDef, CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME));
                parserContext.registerComponent(compositeDef);
            }
        }  
 {%endcodeblock%}
- 解释
  - `InfrastructureAdvisorAutoProxyCreator`:他选择的`advisor`条件是当前bd#role=2,即基础设施,关于参考[自动织入](/2020/03/04/Spring-aop织入分析/#分析源码)
#### 各部分源码逻辑
  {%asset_img cache标准实现.png%}
  - 图中`AnnotationCacheOperationSource`:自动aop处理器抽象逻辑过程会被调用来决定是否对bean进行增强处理
  - 图中`BeanFactoryCacheOperationSourceAdvisor`实际是一个`advisor`,参考[aop](/2019/05/25/aop/#术语和类)
    - advice:`CacheInterceptor`:完成缓存调用逻辑
    - pointCut:`CacheOperationSourcePointcut`:实际调用`AnnotationCacheOperationSource`实现pointCut功能
  - 实际上spring-cache的实现原理就是通过aop实现的  
##### 缓存操作的封装类
###### CacheOperation
{%asset_img cacheOp.png%}
该类实际是spring通过用户类中缓存注解创建封装了注解信息,最后供`CacheInterceptor`调用,没什么特别的
{%codeblock lang:java CacheOperation%}  
public abstract class CacheOperation implements BasicOperation {

    private final String name;

    private final Set<String> cacheNames;

    private final String key;

    private final String keyGenerator;

    private final String cacheManager;

    private final String cacheResolver;

    private final String condition;

    private final String toString;
}  
{%endcodeblock%}
#### advisor
标准实现由`BeanFactoryCacheOperationSourceAdvisor`和`CacheInterceptor`,以及切点`CacheOperationSourcePointcut`实现
##### advisor逻辑
advisor一般解决问题:`提供切点`和`提供增强`
- 代码逻辑
  {%codeblock lang:java BeanFactoryCacheOperationSourceAdvisor%}
  public abstract class AbstractBeanFactoryPointcutAdvisor extends AbstractPointcutAdvisor implements BeanFactoryAware {
    private String adviceBeanName;//advice
    private BeanFactory beanFactory;
    //获取切点特别方法,从bf中拿
    public Advice getAdvice() {
          Advice advice = this.advice;
          if (advice != null) {
              return advice;
          }

          Assert.state(this.adviceBeanName != null, "'adviceBeanName' must be specified");
          Assert.state(this.beanFactory != null, "BeanFactory must be set to resolve 'adviceBeanName'");

          if (this.beanFactory.isSingleton(this.adviceBeanName)) {
              // Rely on singleton semantics provided by the factory.
              advice = this.beanFactory.getBean(this.adviceBeanName, Advice.class);
              this.advice = advice;
              return advice;
          }
          else {
              // No singleton guarantees from the factory -> let's lock locally but
              // reuse the factory's singleton lock, just in case a lazy dependency
              // of our advice bean happens to trigger the singleton lock implicitly...
              synchronized (this.adviceMonitor) {
                  advice = this.advice;
                  if (advice == null) {
                      advice = this.beanFactory.getBean(this.adviceBeanName, Advice.class);
                      this.advice = advice;
                  }
                  return advice;
              }
          }
      }
  }    
  public class BeanFactoryCacheOperationSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

      @Nullable
      private CacheOperationSource cacheOperationSource;
    //增强
      private final CacheOperationSourcePointcut pointcut = new CacheOperationSourcePointcut() {
          @Override
          @Nullable
          protected CacheOperationSource getCacheOperationSource() {
              return cacheOperationSource;
          }
      };

      public void setCacheOperationSource(CacheOperationSource cacheOperationSource) {
          this.cacheOperationSource = cacheOperationSource;
      }
      public void setClassFilter(ClassFilter classFilter) {
          this.pointcut.setClassFilter(classFilter);
      }
    //内部切点
      @Override
      public Pointcut getPointcut() {
          return this.pointcut;
      }
  }
  {%endcodeblock%}
- 总结
  - 提供了内部切点`CacheOperationSourcePointcut`
  - 提供从bf中获取adivce的功能,而不是直接用pv引用
##### 切点
- 代码逻辑
  {%codeblock lang:java CacheOperationSourcePointcut%}
  abstract class CacheOperationSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {
    protected CacheOperationSourcePointcut() {
          setClassFilter(new CacheOperationSourceClassFilter());
      }
    //--------匹配方法-------------
    public boolean matches(Method method, Class<?> targetClass) {
          CacheOperationSource cas = getCacheOperationSource();
          return (cas != null && !CollectionUtils.isEmpty(cas.getCacheOperations(method, targetClass)));
      }

    //-----------------------内部实现的类匹配器------------------
    private class CacheOperationSourceClassFilter implements ClassFilter {

          @Override
          public boolean matches(Class<?> clazz) {
              if (CacheManager.class.isAssignableFrom(clazz)) {
                  return false;
              }
              CacheOperationSource cas = getCacheOperationSource();
              return (cas == null || cas.isCandidateClass(clazz));
          }
      }
  {%endcodeblock%}
- 总结
  - 方法匹配通过`CacheOperationSource#getCacheOperations`实现
  - 类型匹配通过`CacheOperationSource#isCandidateClass`实现
##### advice
这是spring-cache的核心,由该切面逻辑决定如何执行缓存逻辑,它的执行依据是通过`CacheOpertation`,标准实现的逻辑是`CacheInterceptor`
- 前提说明
  `CacheAspectSupport`完成了逻辑,这个类源码中包含了数个内部类
- 代码逻辑
    {%codeblock lang:java CacheInterceptor%}
    public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {

    @Override
    @Nullable
    public Object invoke(final MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
    //这个匿名类仅仅是一个函数接口
        CacheOperationInvoker aopAllianceInvoker = () -> {
            try {
                return invocation.proceed();//调用原函数
            }
            catch (Throwable ex) {
                throw new CacheOperationInvoker.ThrowableWrapper(ex);
            }
        };

        try {
            return execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());
        }
        catch (CacheOperationInvoker.ThrowableWrapper th) {
            throw th.getOriginal();
        }
    }

}
    {%endcodeblock%}
###### 内部类CacheOperationCacheKey
- 代码逻辑
  {%codeblock lang:java CacheOperationCacheKey%}    
private static final class CacheOperationCacheKey implements Comparable<CacheOperationCacheKey> {

        private final CacheOperation cacheOperation;

        private final AnnotatedElementKey methodCacheKey;
    public int compareTo(CacheOperationCacheKey other) {
            int result = this.cacheOperation.getName().compareTo(other.cacheOperation.getName());
            if (result == 0) {
                result = this.methodCacheKey.compareTo(other.methodCacheKey);
            }
            return result;
        }
}    
{%endcodeblock%}
- 总结:实际上就是一个比较器,由`AnnotatedElementKey`和`CacheOperation`组成,前者实际上是由`AnnotatedElement`构成的比较器(jdk属性,method,class的父类)
###### 内部类CacheOperationMetadata
- 代码逻辑
  {%codeblock lang:java CacheOperationMetadata%}
protected static class CacheOperationMetadata {

        private final CacheOperation operation;

        private final Method method;

        private final Class<?> targetClass;

        private final Method targetMethod;

        private final AnnotatedElementKey methodKey;
    //key生成器,匹配缓存时使用
        private final KeyGenerator keyGenerator;
    //获取Cache的解析器
        private final CacheResolver cacheResolver;

        public CacheOperationMetadata(CacheOperation operation, Method method, Class<?> targetClass,
                KeyGenerator keyGenerator, CacheResolver cacheResolver) {

            this.operation = operation;
            this.method = BridgeMethodResolver.findBridgedMethod(method);
            this.targetClass = targetClass;
            this.targetMethod = (!Proxy.isProxyClass(targetClass) ?
                    AopUtils.getMostSpecificMethod(method, targetClass) : this.method);
            this.methodKey = new AnnotatedElementKey(this.targetMethod, targetClass);
            this.keyGenerator = keyGenerator;
            this.cacheResolver = cacheResolver;
        }
    }
{%endcodeblock%}
- 总结:该类是把`Operation`,`Method`,'Class'转化真正的处理部件
###### CacheOperationContext
- 代码逻辑
  {%codeblock lang:java CacheOperationContext%}
protected class CacheOperationContext implements CacheOperationInvocationContext<CacheOperation> {

        private final CacheOperationMetadata metadata;

        private final Object[] args;

        private final Object target;

        private final Collection<? extends Cache> caches;

        private final Collection<String> cacheNames;

        @Nullable
        private Boolean conditionPassing;

        public CacheOperationContext(CacheOperationMetadata metadata, Object[] args, Object target) {
            this.metadata = metadata;
            this.args = extractArgs(metadata.method, args);
            this.target = target;
      //调用解析器获取Cache
            this.caches = CacheAspectSupport.this.getCaches(this, metadata.cacheResolver);
            this.cacheNames = createCacheNames(this.caches);
        }
    protected Object generateKey(@Nullable Object result) {
      //...........
    }
}    
{%endcodeblock%}
- 总结:该类已经是根据源信息缓存了`Cache`,提供了key生成函数
###### 内部类CacheOperationContexts
- 代码逻辑
  {%codeblock lang:CacheOperationContexts%}
  private class CacheOperationContexts {
    private final MultiValueMap<Class<? extends CacheOperation>, CacheOperationContext> contexts;

        private final boolean sync;

        public CacheOperationContexts(Collection<? extends CacheOperation> operations, Method method,
                Object[] args, Object target, Class<?> targetClass) {

            this.contexts = new LinkedMultiValueMap<>(operations.size());
            for (CacheOperation op : operations) {
                this.contexts.add(op.getClass(), getOperationContext(op, method, args, target, targetClass));  //此处时创建CacheOperationContext的入口
            }
            this.sync = determineSyncFlag(method);
        }
  }
  {%endcodeblock%}
- 总结:此类封装了'CacheOperation'->`CacheOperationContext`的信息,并且由该类的构造为如创建上述的内部类
###### 实际调用逻辑
- 代码逻辑
  {%codeblock lang:java CacheAspectSupport%}

    //缓存key->源信息
    private final Map<CacheOperationCacheKey, CacheOperationMetadata> metadataCache = new ConcurrentHashMap<>(1024);

      private final CacheOperationExpressionEvaluator evaluator = new CacheOperationExpressionEvaluator();
    //cs
      @Nullable
      private CacheOperationSource cacheOperationSource;
    //keyGen生成器,生成的是SimpleKeyGenerator,会被MetaData调用生成具体的key
      private SingletonSupplier<KeyGenerator> keyGenerator = SingletonSupplier.of(SimpleKeyGenerator::new);

      @Nullable
      private SingletonSupplier<CacheResolver> cacheResolver; //Cache解析器

      @Nullable
      private BeanFactory beanFactory;

      private boolean initialized = false;
//============================执行逻辑=======================================
  protected Object execute(CacheOperationInvoker invoker, Object target, Method method, Object[] args) {
        // Check whether aspect is enabled (to cope with cases where the AJ is pulled in automatically)
        if (this.initialized) {
            Class<?> targetClass = getTargetClass(target);
            CacheOperationSource cacheOperationSource = getCacheOperationSource();
            if (cacheOperationSource != null) {
                Collection<CacheOperation> operations = cacheOperationSource.getCacheOperations(method, targetClass);
                if (!CollectionUtils.isEmpty(operations)) {
                    return execute(invoker, method,
                            new CacheOperationContexts(operations, method, args, target, targetClass));//构造调用
                }
            }
        }

        return invoker.invoke();
    }
  private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
        // Special handling of synchronized invocation
        if (contexts.isSynchronized()) {
            CacheOperationContext context = contexts.get(CacheableOperation.class).iterator().next();
            if (isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
                Object key = generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
                Cache cache = context.getCaches().iterator().next();
                try {
                    return wrapCacheValue(method, cache.get(key, () -> unwrapReturnValue(invokeOperation(invoker))));
                }
                catch (Cache.ValueRetrievalException ex) {
                    // The invoker wraps any Throwable in a ThrowableWrapper instance so we
                    // can just make sure that one bubbles up the stack.
                    throw (CacheOperationInvoker.ThrowableWrapper) ex.getCause();
                }
            }
            else {
                // No caching required, only call the underlying method
                return invokeOperation(invoker);
            }
        }


        // Process any early evictions
    //执行@CacheEvict#beforeInvocation=true的逻辑
        processCacheEvicts(contexts.get(CacheEvictOperation.class), true,
                CacheOperationExpressionEvaluator.NO_RESULT);

        // Check if we have a cached item matching the conditions
    //查看是否@Cacheable#condition满足,并且命中缓存
    //ValueWrapper是实际value的包装
        Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));

        // Collect puts from any @Cacheable miss, if no cached item is found
    // 若发生@Cacheable缓存未命中,则根据参数创建一个CachePutRequest
        List<CachePutRequest> cachePutRequests = new LinkedList<>();
        if (cacheHit == null) {
            collectPutRequests(contexts.get(CacheableOperation.class),
                    CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
        }

        Object cacheValue;
        Object returnValue;

        if (cacheHit != null && !hasCachePut(contexts)) {
            // If there are no put requests, just use the cache hit
      // 命中但是没有@CachePut注解,则仅仅获取value
            cacheValue = cacheHit.get();
            returnValue = wrapCacheValue(method, cacheValue);
        }
        else {
            // Invoke the method if we don't have a cache hit
      // 缓存未命中,执行用户函数,并且cachevlaue改成用户返回值
            returnValue = invokeOperation(invoker);
            cacheValue = unwrapReturnValue(returnValue);
        }

        // Collect any explicit @CachePuts
    //此时cacheValue可能是用户返回值,可能是null
    //根据cacheValue为result对于CachePut满足条件创建CacheputRequests
        collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);

        // Process any collected put requests, either from @CachePut or a @Cacheable miss
    //进行缓存填充,req来自于@Cacheput条件命中回参,或者@Cacheable缓存未命中的形参
        for (CachePutRequest cachePutRequest : cachePutRequests) {
            cachePutRequest.apply(cacheValue); //把req都缓存进去
        }

        // Process any late evictions
    //执行@CacheEvict#beforeInvocation==false的情况
        processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);

        return returnValue;
    }
  //----------------------执行@CacheEvict#beforeInvocation=true--------------
  private void processCacheEvicts(
            Collection<CacheOperationContext> contexts, boolean beforeInvocation, @Nullable Object result) {

        for (CacheOperationContext context : contexts) {
            CacheEvictOperation operation = (CacheEvictOperation) context.metadata.operation;
            if (beforeInvocation == operation.isBeforeInvocation() && isConditionPassing(context, result)) { //before=ture&&condition满足,条件通过el表达式解析
                performCacheEvict(context, operation, result);
            }
        }
    }
  private void performCacheEvict(
              CacheOperationContext context, CacheEvictOperation operation, @Nullable Object result) {

          Object key = null;
          for (Cache cache : context.getCaches()) {
              if (operation.isCacheWide()) { //对应@CacheEvict#allEntries属性
                  logInvalidating(context, operation, null);
                  doClear(cache, operation.isBeforeInvocation());//调用Cache.clear||或者imde
              }
              else {
                  if (key == null) {
                      key = generateKey(context, result); //根据result和context生成key,根据key删除
                  }
                  logInvalidating(context, operation, key);
                  doEvict(cache, key, operation.isBeforeInvocation());
              }
          }
      }  
    //都是清除逻辑
    protected void doClear(Cache cache, boolean immediate) {
            try {
                if (immediate) {
                    cache.invalidate();
                }
                else {
                    cache.clear();
                }
            }
            catch (RuntimeException ex) {
                getErrorHandler().handleCacheClearError(ex, cache);
            }
        }
      //删除对应的key
      protected void doEvict(Cache cache, Object key, boolean immediate) {
              try {
                  if (immediate) {
                      cache.evictIfPresent(key);
                  }
                  else {
                      cache.evict(key);
                  }
              }
              catch (RuntimeException ex) {
                  getErrorHandler().handleCacheEvictError(ex, cache, key);
              }
          }
  //---------------------@Cacheable#condition命中-------------
  private Cache.ValueWrapper findCachedItem(Collection<CacheOperationContext> contexts) {
        Object result = CacheOperationExpressionEvaluator.NO_RESULT;
        for (CacheOperationContext context : contexts) {
            if (isConditionPassing(context, result)) {
                Object key = generateKey(context, result);
                Cache.ValueWrapper cached = findInCaches(context, key);
                if (cached != null) { //明显即使存在多个也只是从第一个返回
                    return cached;
                }
                else {
                    if (logger.isTraceEnabled()) {
                        logger.trace("No cache entry for key '" + key + "' in cache(s) " + context.getCacheNames());
                    }
                }
            }
        }
        return null;
    }      
  //--------------------根据result进行判断创建一个putReq------------
  //result的来源来自于形参,或者返回值,前者是@Cacheable处理过程中的,后者是@CachePut用户调用函数的
  private void collectPutRequests(Collection<CacheOperationContext> contexts,
            @Nullable Object result, Collection<CachePutRequest> putRequests) {

        for (CacheOperationContext context : contexts) {
            if (isConditionPassing(context, result)) {
                Object key = generateKey(context, result);
                putRequests.add(new CachePutRequest(context, key));
            }
        }
    }
  //-----------------reqput---------------------------------------
  private class CachePutRequest {

        private final CacheOperationContext context;

        private final Object key;

        public CachePutRequest(CacheOperationContext context, Object key) {
            this.context = context;
            this.key = key;
        }

        public void apply(@Nullable Object result) {
            if (this.context.canPutToCache(result)) {//满足条件则添加
                for (Cache cache : this.context.getCaches()) {
                    doPut(cache, this.key, result);
                }
            }
        }
    }
  protected boolean canPutToCache(@Nullable Object value) {
            String unless = "";
            if (this.metadata.operation instanceof CacheableOperation) {
                unless = ((CacheableOperation) this.metadata.operation).getUnless();
            }
            else if (this.metadata.operation instanceof CachePutOperation) {
                unless = ((CachePutOperation) this.metadata.operation).getUnless();
            }//获取注解#unless表达式
            if (StringUtils.hasText(unless)) {
                EvaluationContext evaluationContext = createEvaluationContext(value);
                return !evaluator.unless(unless, this.metadata.methodKey, evaluationContext); //若表达式和value匹配,则不通过
            }
            return true;
  //================contexts的创建过程=====================
  public CacheOperationContexts(Collection<? extends CacheOperation> operations, Method method,
                Object[] args, Object target, Class<?> targetClass) {

            this.contexts = new LinkedMultiValueMap<>(operations.size());
            for (CacheOperation op : operations) {
                this.contexts.add(op.getClass(), getOperationContext(op, method, args, target, targetClass));//函数调用
            }
            this.sync = determineSyncFlag(method);//决定是否加锁
        }
  //-----------context创建--------------
  protected CacheOperationContext getOperationContext(
            CacheOperation operation, Method method, Object[] args, Object target, Class<?> targetClass) {

        CacheOperationMetadata metadata = getCacheOperationMetadata(operation, method, targetClass);
        return new CacheOperationContext(metadata, args, target);
    }
  //-------------源信息创建--------------------
  protected CacheOperationMetadata getCacheOperationMetadata(
            CacheOperation operation, Method method, Class<?> targetClass) {
    //key=AnnotatedElementKey+CacheOperation    
        CacheOperationCacheKey cacheKey = new CacheOperationCacheKey(operation, method, targetClass);
        CacheOperationMetadata metadata = this.metadataCache.get(cacheKey);
        if (metadata == null) {//若缓存中没有
            KeyGenerator operationKeyGenerator;
      //[1]创建keyGen,自定义或者SimpleKeyGenerator
            if (StringUtils.hasText(operation.getKeyGenerator())) {
                operationKeyGenerator = getBean(operation.getKeyGenerator(), KeyGenerator.class);
            }
            else {
                operationKeyGenerator = getKeyGenerator();
            }
      //[1!]
      //[2]创建CacheResolver,若用户没有定义一个id=cacheManager的处理器,那么直接爆炸
            CacheResolver operationCacheResolver;
            if (StringUtils.hasText(operation.getCacheResolver())) {
                operationCacheResolver = getBean(operation.getCacheResolver(), CacheResolver.class);
            }
            else if (StringUtils.hasText(operation.getCacheManager())) {
                CacheManager cacheManager = getBean(operation.getCacheManager(), CacheManager.class);
                operationCacheResolver = new SimpleCacheResolver(cacheManager);
            }
            else {
                operationCacheResolver = getCacheResolver();
                Assert.state(operationCacheResolver != null, "No CacheResolver/CacheManager set");
            }//[2!]
            metadata = new CacheOperationMetadata(operation, method, targetClass,
                    operationKeyGenerator, operationCacheResolver);
            this.metadataCache.put(cacheKey, metadata);
        }
        return metadata;
    }
  //---------context构造器调用--------------
  public CacheOperationContext(CacheOperationMetadata metadata, Object[] args, Object target) {
            this.metadata = metadata;
            this.args = extractArgs(metadata.method, args);
            this.target = target;
            this.caches = CacheAspectSupport.this.getCaches(this, metadata.cacheResolver); //调用了解析器
            this.cacheNames = createCacheNames(this.caches);
        }
{%endcodeblock%}
###### Cache解析
- CacheResolver
内部含有`CacheManager`,用来返回`Cache`
{%asset_img CacheResolver.png%}
  - 代码逻辑
    {%codeblock lang:java CacheResolver%}
private CacheManager cacheManager; //一般用户指定

public Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context) {
        Collection<String> cacheNames = getCacheNames(context);
        if (cacheNames == null) {
            return Collections.emptyList();
        }
        Collection<Cache> result = new ArrayList<>(cacheNames.size());
        for (String cacheName : cacheNames) {
            Cache cache = getCacheManager().getCache(cacheName); //CacheManager根据cacheNames返回对应的Cache
            if (cache == null) {
                throw new IllegalArgumentException("Cannot find cache named '" +
                        cacheName + "' for " + context.getOperation());
            }
            result.add(cache);
        }
        return result;
    }
{%endcodeblock%}
- CacheManager
提供获取Cache的函数,简单来说就是通过注解中如`@Cacheable#value`属性来返回对应的`Cache`
{%asset_img CacheManager.png%}
  - 代码逻辑
    {%codeblock lang:java SimpleCacheManager%}
    public class SimpleCacheManager extends AbstractCacheManager {

      private Collection<? extends Cache> caches = Collections.emptySet(); //一般也是用户来指定
    /**
     * Specify the collection of Cache instances to use for this CacheManager.
     */
    public void setCaches(Collection<? extends Cache> caches) {
        this.caches = caches;
    }

    @Override
    protected Collection<? extends Cache> loadCaches() {
        return this.caches;
    }

}
//=============抽象逻辑=================
public abstract class AbstractCacheManager implements CacheManager, InitializingBean {
  public Cache getCache(String name) {
        // Quick check for existing cache...
        Cache cache = this.cacheMap.get(name);
        if (cache != null) {
            return cache;
        }

        // The provider may support on-demand cache creation...
        Cache missingCache = getMissingCache(name);
        if (missingCache != null) {
            // Fully synchronize now for missing cache registration
            synchronized (this.cacheMap) {
                cache = this.cacheMap.get(name);
                if (cache == null) {
                    cache = decorateCache(missingCache);
                    this.cacheMap.put(name, cache);
                    updateCacheNames(name);
                }
            }
        }
        return cache;
    }
}  
    {%endcodeblock%}
- Cache
真正进行缓存的实现

##### 确定增强的类
这部分代码完成确认用户类何时增强,以及返回增强信息.
###### CacheOperationSource
在自动织入时实际上调用了这里的逻辑来判断当前类是否应该被增强,并且通过该类生成了缓存操作类`CacheOpertation`
- 代码逻辑
    {%codeblock lang:java CacheOperationSource%}
    public interface CacheOperationSource {
      default boolean isCandidateClass(Class<?> targetClass) {
            return true;
        }
      Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass);
    }  

    //=============================AbstractFallbackCacheOperationSource实现
    public abstract class AbstractFallbackCacheOperationSource implements CacheOperationSource {
      //表示空,aop自动织入处理器也有这种属性
      private static final Collection<CacheOperation> NULL_CACHING_ATTRIBUTE = Collections.emptyList();
      //缓存
      private final Map<Object, Collection<CacheOperation>> attributeCache = new ConcurrentHashMap<>(1024);
      //=========================尝试获取缓存操作对象=================================
      public Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass) {
            if (method.getDeclaringClass() == Object.class) {
                return null;
            }

            Object cacheKey = getCacheKey(method, targetClass);//用户对象method+class表示key
            Collection<CacheOperation> cached = this.attributeCache.get(cacheKey);

            if (cached != null) {
                return (cached != NULL_CACHING_ATTRIBUTE ? cached : null);
            }
            else {//创建逻辑
                Collection<CacheOperation> cacheOps = computeCacheOperations(method, targetClass);
                if (cacheOps != null) {
                    if (logger.isTraceEnabled()) {
                        logger.trace("Adding cacheable method '" + method.getName() + "' with attribute: " + cacheOps);
                    }
                    this.attributeCache.put(cacheKey, cacheOps);
                }
                else {
                    this.attributeCache.put(cacheKey, NULL_CACHING_ATTRIBUTE);
                }
                return cacheOps;
            }
        }
      private Collection<CacheOperation> computeCacheOperations(Method method, @Nullable Class<?> targetClass) {
            // Don't allow no-public methods as required.
            if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {//判断是否仅仅支持public函数
                return null;
            }

            // The method may be on an interface, but we need attributes from the target class.
            // If the target class is null, the method will be unchanged.
            Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

            // First try is the method in the target class.
            Collection<CacheOperation> opDef = findCacheOperations(specificMethod); //当前调用函数中寻找
            if (opDef != null) {
                return opDef;
            }

            // Second try is the caching operation on the target class.
            opDef = findCacheOperations(specificMethod.getDeclaringClass()); //用户类中找  
            if (opDef != null && ClassUtils.isUserLevelMethod(method)) {
                return opDef;
            }

            if (specificMethod != method) {
                // Fallback is to look at the original method.
                opDef = findCacheOperations(method);
                if (opDef != null) {
                    return opDef;
                }
                // Last fallback is the class of the original method.
                opDef = findCacheOperations(method.getDeclaringClass());
                if (opDef != null && ClassUtils.isUserLevelMethod(method)) {
                    return opDef;
                }
            }

            return null;
        }
      //子类实现
      protected abstract Collection<CacheOperation> findCacheOperations(Class<?> clazz);
      protected abstract Collection<CacheOperation> findCacheOperations(Method method);
      protected boolean allowPublicMethodsOnly() {
            return false;
        }
    }  
    //====================AnnotationCacheOperationSource实现
    public class AnnotationCacheOperationSource extends AbstractFallbackCacheOperationSource implements Serializable {
        private final boolean publicMethodsOnly;

          private final Set<CacheAnnotationParser> annotationParsers;//决定是否匹配的类

        public AnnotationCacheOperationSource() {
              this(true);
          }
        public AnnotationCacheOperationSource(boolean publicMethodsOnly) {
              this.publicMethodsOnly = publicMethodsOnly;
              this.annotationParsers = Collections.singleton(new SpringCacheAnnotationParser());
          }  
        //====================是否应该增强=========================
        public boolean isCandidateClass(Class<?> targetClass) {
            for (CacheAnnotationParser parser : this.annotationParsers) {
                if (parser.isCandidateClass(targetClass)) {
                    return true;
                }
            }
            return false;
        }
      //=====================创造CacheOperation=======================
      protected Collection<CacheOperation> findCacheOperations(Class<?> clazz) {
            return determineCacheOperations(parser -> parser.parseCacheAnnotations(clazz));
        }
      protected Collection<CacheOperation> determineCacheOperations(CacheOperationProvider provider) {
            Collection<CacheOperation> ops = null;
            for (CacheAnnotationParser parser : this.annotationParsers) {
                Collection<CacheOperation> annOps = provider.getCacheOperations(parser);
                if (annOps != null) {
                    if (ops == null) {
                        ops = annOps;
                    }
                    else {
                        Collection<CacheOperation> combined = new ArrayList<>(ops.size() + annOps.size());
                        combined.addAll(ops);
                        combined.addAll(annOps);
                        ops = combined;
                    }
                }
            }
            return ops;
        }
      //函数对象类
      protected interface CacheOperationProvider {

            /**
             * Return the {@link CacheOperation} instance(s) provided by the specified parser.
             * @param parser the parser to use
             * @return the cache operations, or {@code null} if none found
             */
            @Nullable
            Collection<CacheOperation> getCacheOperations(CacheAnnotationParser parser);
        }
        @Override
        @Nullable
        protected Collection<CacheOperation> findCacheOperations(Method method) {
            return determineCacheOperations(parser -> parser.parseCacheAnnotations(method));
        }
    }
{%endcodeblock%}    
- 总结
  - 由该类提供判断那些用户应该增强为缓存函数
  - 由该类创建`Opertation`
  - 该类实际委托给了`CacheAnnotationParser`实现
##### CacheAnnotationParser
该类决定是否匹配以及如何创建`CacheOperation`
- 代码逻辑
  {%codeblock lang:java CacheAnnotationParser%}
  //==================接口实现
  public interface CacheAnnotationParser {
    default boolean isCandidateClass(Class<?> targetClass) {
        return true;
    }
  Collection<CacheOperation> parseCacheAnnotations(Class<?> type);
  Collection<CacheOperation> parseCacheAnnotations(Method method);    
  }  
  //================SpringCacheAnnotationParser实现
  public class SpringCacheAnnotationParser implements CacheAnnotationParser, Serializable {
    private static final Set<Class<? extends Annotation>> CACHE_OPERATION_ANNOTATIONS = new LinkedHashSet<>(8);

    static { //cache注解
        CACHE_OPERATION_ANNOTATIONS.add(Cacheable.class);
        CACHE_OPERATION_ANNOTATIONS.add(CacheEvict.class);
        CACHE_OPERATION_ANNOTATIONS.add(CachePut.class);
        CACHE_OPERATION_ANNOTATIONS.add(Caching.class);
    }
  //================外部调用函数=================
  public boolean isCandidateClass(Class<?> targetClass) {
        return AnnotationUtils.isCandidateClass(targetClass, CACHE_OPERATION_ANNOTATIONS);
    }

    @Override
    @Nullable
    public Collection<CacheOperation> parseCacheAnnotations(Class<?> type) {
        DefaultCacheConfig defaultConfig = new DefaultCacheConfig(type);
        return parseCacheAnnotations(defaultConfig, type);
    }
  //==============针对不同的注解添加不同的信息===========================
  private Collection<CacheOperation> parseCacheAnnotations(
            DefaultCacheConfig cachingConfig, AnnotatedElement ae, boolean localOnly) {

        Collection<? extends Annotation> anns = (localOnly ?
                AnnotatedElementUtils.getAllMergedAnnotations(ae, CACHE_OPERATION_ANNOTATIONS) :
                AnnotatedElementUtils.findAllMergedAnnotations(ae, CACHE_OPERATION_ANNOTATIONS));
        if (anns.isEmpty()) {
            return null;
        }

        final Collection<CacheOperation> ops = new ArrayList<>(1);
        anns.stream().filter(ann -> ann instanceof Cacheable).forEach(
                ann -> ops.add(parseCacheableAnnotation(ae, cachingConfig, (Cacheable) ann)));
        anns.stream().filter(ann -> ann instanceof CacheEvict).forEach(
                ann -> ops.add(parseEvictAnnotation(ae, cachingConfig, (CacheEvict) ann)));
        anns.stream().filter(ann -> ann instanceof CachePut).forEach(
                ann -> ops.add(parsePutAnnotation(ae, cachingConfig, (CachePut) ann)));
        anns.stream().filter(ann -> ann instanceof Caching).forEach(
                ann -> parseCachingAnnotation(ae, cachingConfig, (Caching) ann, ops));
        return ops;
    }
  private CacheableOperation parseCacheableAnnotation(
            AnnotatedElement ae, DefaultCacheConfig defaultConfig, Cacheable cacheable) {

        CacheableOperation.Builder builder = new CacheableOperation.Builder();

        builder.setName(ae.toString());
        builder.setCacheNames(cacheable.cacheNames());
        builder.setCondition(cacheable.condition());
        builder.setUnless(cacheable.unless());
        builder.setKey(cacheable.key());
        builder.setKeyGenerator(cacheable.keyGenerator());
        builder.setCacheManager(cacheable.cacheManager());
        builder.setCacheResolver(cacheable.cacheResolver());
        builder.setSync(cacheable.sync());

        defaultConfig.applyDefault(builder);
        CacheableOperation op = builder.build();
        validateCacheOperation(ae, op);

        return op;
    }
    @Override
    @Nullable
    public Collection<CacheOperation> parseCacheAnnotations(Method method) {
        DefaultCacheConfig defaultConfig = new DefaultCacheConfig(method.getDeclaringClass());
        return parseCacheAnnotations(defaultConfig, method);
    }
  //==================内部封装类===================
  private static class DefaultCacheConfig {

        private final Class<?> target;

        @Nullable
        private String[] cacheNames;

        @Nullable
        private String keyGenerator;

        @Nullable
        private String cacheManager;

        @Nullable
        private String cacheResolver;

        private boolean initialized = false;

        public DefaultCacheConfig(Class<?> target) {
            this.target = target;
        }

        /**
         * Apply the defaults to the specified {@link CacheOperation.Builder}.
         * @param builder the operation builder to update
         */
        public void applyDefault(CacheOperation.Builder builder) {
            if (!this.initialized) {
                CacheConfig annotation = AnnotatedElementUtils.findMergedAnnotation(this.target, CacheConfig.class);
                if (annotation != null) {
                    this.cacheNames = annotation.cacheNames();
                    this.keyGenerator = annotation.keyGenerator();
                    this.cacheManager = annotation.cacheManager();
                    this.cacheResolver = annotation.cacheResolver();
                }
                this.initialized = true;
            }

            if (builder.getCacheNames().isEmpty() && this.cacheNames != null) {
                builder.setCacheNames(this.cacheNames);
            }
            if (!StringUtils.hasText(builder.getKey()) && !StringUtils.hasText(builder.getKeyGenerator()) &&
                    StringUtils.hasText(this.keyGenerator)) {
                builder.setKeyGenerator(this.keyGenerator);
            }

            if (StringUtils.hasText(builder.getCacheManager()) || StringUtils.hasText(builder.getCacheResolver())) {
                // One of these is set so we should not inherit anything
            }
            else if (StringUtils.hasText(this.cacheResolver)) {
                builder.setCacheResolver(this.cacheResolver);
            }
            else if (StringUtils.hasText(this.cacheManager)) {
                builder.setCacheManager(this.cacheManager);
            }
        }
    }

  }  
  {%endcodeblock%}
