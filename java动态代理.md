---
title: java动态代理
date: 2019-05-22 19:09:46
tags:
- proxy
- jdk
categories: java
---
### 关于jdk提供动态代理
#### 例子
{%codeblock lang:java 例子%}
//代理对象
public interface MethodInterface {
    void doSomething();
}
//代理类
public class MethodClass implements MethodInterface,MethodInterface2{
    @Override
    public void doSomething() {
        System.out.println("原本的函数");
    }

    @Override
    public void doSomething2() {
        System.out.println("原本的函数2");
    }
}
//获取代理类
//这里的代码我是为了测试代理对象实现多个接口才这么写的
public static <T> T getProxy(MethodClass methodClass){
        return (T) Proxy.newProxyInstance(MethodClass.class.getClassLoader(), MethodClass.class.getInterfaces(), (proxy, method, args) -> {
            System.out.println("代理前");
			//这里有个问题就是这个method可以匹配proxy和target
            var ob= method.invoke(methodClass,args);
            System.out.println("代理后");
            return ob;
        });
    }
{%endcodeblock%}
#### 猜测
我认为proxy代理返回的结构如下
{%asset_img proxy.png%}
并且代理对象的接口函数逻辑都是
```java
void doSomething([args]){
//这里这个method是代理对象Class中声明的,并且该method可以匹配proxy和target对象
  h.invoke(this,method,args)
}
```
- 我认为proxy的结构也许是实现了接口,而是继承了代理类,也就是下图
{%asset_img proxy2.png%}
#### 源码
首先我看的源码是jdk11,和9之前版本不同,引入了模块概念,忽略不计
{%codeblock lang:java Proxy%}
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h) {
        Objects.requireNonNull(h);

        final Class<?> caller = System.getSecurityManager() == null
                                    ? null
                                    : Reflection.getCallerClass();

        //这里获取代理类构造器
        Constructor<?> cons = getProxyConstructor(caller, loader, interfaces);

        return newProxyInstance(caller, cons, h);
    }
//
private static Constructor<?> getProxyConstructor(Class<?> caller,
                                                      ClassLoader loader,
                                                      Class<?>... interfaces)
    {
        // optimization for single interface
        if (interfaces.length == 1) {
            Class<?> intf = interfaces[0];
            if (caller != null) {
                checkProxyAccess(caller, loader, intf);
            }
            return proxyCache.sub(intf).computeIfAbsent(//缓存容器
                loader,
                (ld, clv) -> new ProxyBuilder(ld, clv.key()).build() //仅仅注意这个lambda就行
            );
        } else {
            // interfaces cloned
            final Class<?>[] intfsArray = interfaces.clone();
            if (caller != null) {
                checkProxyAccess(caller, loader, intfsArray);
            }
            final List<Class<?>> intfs = Arrays.asList(intfsArray);
            return proxyCache.sub(intfs).computeIfAbsent(
                loader,
                (ld, clv) -> new ProxyBuilder(ld, clv.key()).build()
            );
        }
    }
//
   Constructor<?> build() {
            Class<?> proxyClass = defineProxyClass(module, interfaces); //构建代理类的Class对象
            final Constructor<?> cons;
            try {
                cons = proxyClass.getConstructor(constructorParams);
            } catch (NoSuchMethodException e) {
                throw new InternalError(e.toString(), e);
            }
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
            return cons;
        }
//
 private static Class<?> defineProxyClass(Module m, List<Class<?>> interfaces) {
          //省略大量判断,如方法私有等标志

            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                    proxyName, interfaces.toArray(EMPTY_CLASS_ARRAY), accessFlags); //生成字节码
           try {
                Class<?> pc = JLA.defineClass(loader, proxyName, proxyClassFile,
                                              null, "__dynamic_proxy__");//真正定义Class的方式
                reverseProxyCache.sub(pc).putIfAbsent(loader, Boolean.TRUE);
                return pc;
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }		
{%endcodeblock%}
构造代理类字节码的类
{%codeblock lang:java ProxyGenerator%}
//在11版本中,该类位于Reflect包中,我们无法调用,因此无法查看生成的字节码
//但是通过源码我们可以查看生产的代理类情况
class ProxyGenerator {
//该变量决定jdk是否会对代理类生成文件
private static final boolean saveGeneratedFiles =
        java.security.AccessController.doPrivileged(
            new GetBooleanAction(
                "jdk.proxy.ProxyGenerator.saveGeneratedFiles")).booleanValue();
}
//
static byte[] generateProxyClass(final String name,
                                     Class<?>[] interfaces,
                                     int accessFlags)
    {
        ProxyGenerator gen = new ProxyGenerator(name, interfaces, accessFlags);
        final byte[] classFile = gen.generateClassFile(); //该函数生成了代理类文件

        if (saveGeneratedFiles) { //当开启后就会在项目下生成 com/sun/proxy/代理类.class文件
            java.security.AccessController.doPrivileged(
            new java.security.PrivilegedAction<Void>() {
                public Void run() {
                    try {
                        int i = name.lastIndexOf('.');
                        Path path;
                        if (i > 0) {
                            Path dir = Path.of(name.substring(0, i).replace('.', File.separatorChar));
                            Files.createDirectories(dir);
                            path = dir.resolve(name.substring(i+1, name.length()) + ".class");
                        } else {
                            path = Path.of(name + ".class");
                        }
                        Files.write(path, classFile);
                        return null;
                    } catch (IOException e) {
                        throw new InternalError(
                            "I/O exception saving generated file: " + e);
                    }
                }
            });
        }

        return classFile;
    }
//真正生成字节码的函数
//想要分析这个函数还要了解class文件结构,我基本记不清楚了
 private byte[] generateClassFile() {
  /* ============================================================
         * Step 1: Assemble ProxyMethod objects for all methods to
         * generate proxy dispatching code for.
         */

        /*
         * Record that proxy methods are needed for the hashCode, equals,
         * and toString methods of java.lang.Object.  This is done before
         * the methods from the proxy interfaces so that the methods from
         * java.lang.Object take precedence over duplicate methods in the
         * proxy interfaces.
         */
        addProxyMethod(hashCodeMethod, Object.class);
        addProxyMethod(equalsMethod, Object.class);
        addProxyMethod(toStringMethod, Object.class);

        /*
         * Now record all of the methods from the proxy interfaces, giving
         * earlier interfaces precedence over later ones with duplicate
         * methods.
         */
        for (Class<?> intf : interfaces) {
            for (Method m : intf.getMethods()) {
                if (!Modifier.isStatic(m.getModifiers())) {
                    addProxyMethod(m, intf);
                }
            }
        }

        /*
         * For each set of proxy methods with the same signature,
         * verify that the methods' return types are compatible.
         */
        for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
            checkReturnTypes(sigmethods);
        }

        /* ============================================================
         * Step 2: Assemble FieldInfo and MethodInfo structs for all of
         * fields and methods in the class we are generating.
         */
        try {
            methods.add(generateConstructor());

            for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
                for (ProxyMethod pm : sigmethods) {

                    // add static field for method's Method object
                    fields.add(new FieldInfo(pm.methodFieldName,
                        "Ljava/lang/reflect/Method;",
                         ACC_PRIVATE | ACC_STATIC));

                    // generate code for proxy method and add it
                    methods.add(pm.generateMethod());
                }
            }

            methods.add(generateStaticInitializer());

        } catch (IOException e) {
            throw new InternalError("unexpected I/O Exception", e);
        }

        if (methods.size() > 65535) {
            throw new IllegalArgumentException("method limit exceeded");
        }
        if (fields.size() > 65535) {
            throw new IllegalArgumentException("field limit exceeded");
        }

        /* ============================================================
         * Step 3: Write the final class file.
         */

        /*
         * Make sure that constant pool indexes are reserved for the
         * following items before starting to write the final class file.
         */
        cp.getClass(dotToSlash(className));
        cp.getClass(superclassName);
        for (Class<?> intf: interfaces) {
            cp.getClass(dotToSlash(intf.getName()));
        }

        /*
         * Disallow new constant pool additions beyond this point, since
         * we are about to write the final constant pool table.
         */
        cp.setReadOnly();

        ByteArrayOutputStream bout = new ByteArrayOutputStream();
        DataOutputStream dout = new DataOutputStream(bout);
        //此处就是写的过程,总之还是很有意思的
        try {
            /*
             * Write all the items of the "ClassFile" structure.
             * See JVMS section 4.1.
             */
                                        // u4 magic; 魔数
            dout.writeInt(0xCAFEBABE);
                                        // u2 minor_version;  主版本
            dout.writeShort(CLASSFILE_MINOR_VERSION);
                                        // u2 major_version;   此版本
            dout.writeShort(CLASSFILE_MAJOR_VERSION);

            cp.write(dout);             // (write constant pool) 常量池

                                        // u2 access_flags;  访问标志
            dout.writeShort(accessFlags);
                                        // u2 this_class;  类对象
            dout.writeShort(cp.getClass(dotToSlash(className)));
                                        // u2 super_class;
            dout.writeShort(cp.getClass(superclassName));

                                        // u2 interfaces_count;
            dout.writeShort(interfaces.length);
                                        // u2 interfaces[interfaces_count];
            for (Class<?> intf : interfaces) {
                dout.writeShort(cp.getClass(
                    dotToSlash(intf.getName())));
            }

                                        // u2 fields_count;
            dout.writeShort(fields.size());
                                        // field_info fields[fields_count];
            for (FieldInfo f : fields) {
                f.write(dout);
            }

                                        // u2 methods_count;
            dout.writeShort(methods.size());
                                        // method_info methods[methods_count];
            for (MethodInfo m : methods) {
                m.write(dout);
            }

                                         // u2 attributes_count;
            dout.writeShort(0); // (no ClassFile attributes for proxy classes)

        } catch (IOException e) {
            throw new InternalError("unexpected I/O Exception", e);
        }

        return bout.toByteArray();
 }
{%endcodeblock%}
```java
//不能使用junit测试,因为该框架是通过proxy启动的,它会导致我们设置的property无效
public class Test  {
    public static void main(String[] args) {
	//启用代理类文件
            System.getProperties().setProperty("jdk.proxy.ProxyGenerator.saveGeneratedFiles","true");
            var inter= MyProxy.<MethodInterface>getProxy(new MethodClass());
//        var method= inter.getClass().getDeclaredMethods();
            inter.doSomething();
    }
}
```
#### 代理类文件
{%codeblock lang:java 代理类%}
//代理类并没有继承目标对象
public final class $Proxy0 extends Proxy implements MethodInterface, MethodInterface2 {
    private static Method m1;
    private static Method m4;
    private static Method m3;
    private static Method m2;
    private static Method m0;

   static {
        try {
		   //获取代理method类对象
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
			//注意method获取的是接口类中的Method对象,因此proxy可以通过invoke调用
            m4 = Class.forName("proxy.MethodInterface2").getMethod("doSomething2");
            m3 = Class.forName("proxy.MethodInterface").getMethod("doSomething");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void doSomething2() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void doSomething() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }


}
{%endcodeblock%}
#### 总结
- 研究这个玩意可以更加深刻的意识到从java的角度来看这本语言,就是一个动态性语言,一切的动态性来源于类的加载方式,
在程序运行期间,可以很大程度上修改class
- 依然不能完全想清楚的还是动态加载时如何确定符号地址,引申的问题还是C的动态链接相关问题
- Proxy.newProxyInstance(类加载器,接口,hander),实际上从生成的Class文件和这个传递参数来看jdk Proxy仅仅对于接口进行代理,
即生成实现了接口的临时类对象.
- 生成的Class结构符合我第一个猜想,跟传递的类无关
- jdk的proxy生成的代理类为何不直接继承代理类?这样不就可以对代理类函数进行增强
- 实际jdk代理产生的匿名类结构是第一种[参想](#猜测),若hander不持有代理对象,那么新产生的代理对象其实没有多大作用,这里思考
可以和spring中的`JdkDynamicAopProxy`对比看看
