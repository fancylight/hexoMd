---
title: tomcat
tags: 源码
date: 2018-10-06 12:54:27
categories: java
---
### 架构
{% asset_img 架构.png%}
#### server.xml相关
##### server.xml源
```xml
<!--该文件说明了tomcat的 架构,以及很多细节-->
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<!-- Note:  A "Server" is not itself a "Container", so you may not
     define subcomponents such as "Valves" at this level.
     Documentation at /docs/config/server.html
 -->
 <!--最外层的StanderServer-->
<Server port="8005" shutdown="SHUTDOWN">
 <!--5个监听器,加上NamingContextListener一共6个-->
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" /><!--init时打印信息-->
  <!-- Security listener. Documentation at /docs/config/listeners.html
  <Listener className="org.apache.catalina.security.SecurityListener" />
  --> <!-- -->
  <!--APR library loader. Documentation at /docs/apr.html -->
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />  <!--关于apr是否开启,init时触发-->
  <!-- Prevent memory leaks due to use of particular java/javax APIs-->  
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />    <!--关于防止内存溢出,init触发 -->
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />   <!-- 处理jndi start和stop触发-->
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <!-- Global JNDI resources
       Documentation at /docs/jndi-resources-howto.html
  -->
  <GlobalNamingResources>    <!--NamingResourcesImpl实例,在digister构建该对象的过程中,不单单加入了Resource,还加入其他属性如ejb...,参考源码看 -->   
    <!-- Editable user database that can also be used by
         UserDatabaseRealm to authenticate users
    -->
    <Resource name="UserDatabase" auth="Container"   
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" /> <!--这就是jndi管理的东西 -->
  </GlobalNamingResources>  

  <!-- A "Service" is a collection of one or more "Connectors" that share
       a single "Container" Note:  A "Service" is not itself a "Container",
       so you may not define subcomponents such as "Valves" at this level.
       Documentation at /docs/config/service.html
   -->
  <Service name="Catalina">

    <!--The connectors can use a shared executor, you can define one or more named thread pools-->
    <!--
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="150" minSpareThreads="4"/>
    -->


    <!-- A "Connector" represents an endpoint by which requests are received
         and responses are returned. Documentation at :
         Java HTTP Connector: /docs/config/http.html
         Java AJP  Connector: /docs/config/ajp.html
         APR (HTTP/AJP) Connector: /docs/apr.html
         Define a non-SSL/TLS HTTP/1.1 Connector on port 8080
    -->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <!-- A "Connector" using the shared thread pool-->
    <!--
    <Connector executor="tomcatThreadPool"
               port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    -->
    <!-- Define a SSL/TLS HTTP/1.1 Connector on port 8443
         This connector uses the NIO implementation. The default
         SSLImplementation will depend on the presence of the APR/native
         library and the useOpenSSL attribute of the
         AprLifecycleListener.
         Either JSSE or OpenSSL style configuration may be used regardless of
         the SSLImplementation selected. JSSE style configuration is used below.
    -->
    <!--
    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true">
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="conf/localhost-rsa.jks"
                         type="RSA" />
        </SSLHostConfig>
    </Connector>
    -->
    <!-- Define a SSL/TLS HTTP/1.1 Connector on port 8443 with HTTP/2
         This connector uses the APR/native implementation which always uses
         OpenSSL for TLS.
         Either JSSE or OpenSSL style configuration may be used. OpenSSL style
         configuration is used below.
    -->
    <!--
    <Connector port="8443" protocol="org.apache.coyote.http11.Http11AprProtocol"
               maxThreads="150" SSLEnabled="true" >
        <UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol" />
        <SSLHostConfig>
            <Certificate certificateKeyFile="conf/localhost-rsa-key.pem"
                         certificateFile="conf/localhost-rsa-cert.pem"
                         certificateChainFile="conf/localhost-rsa-chain.pem"
                         type="RSA" />
        </SSLHostConfig>
    </Connector>
    -->

    <!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />


    <!-- An Engine represents the entry point (within Catalina) that processes
         every request.  The Engine implementation for Tomcat stand alone
         analyzes the HTTP headers included with the request, and passes them
         on to the appropriate Host (virtual host).
         Documentation at /docs/config/engine.html -->

    <!-- You should set jvmRoute to support load-balancing via AJP ie :
    <Engine name="Catalina" defaultHost="localhost" jvmRoute="jvm1">
    -->
    <Engine name="Catalina" defaultHost="localhost">

      <!--For clustering, please take a look at documentation at:
          /docs/cluster-howto.html  (simple how to)
          /docs/config/cluster.html (reference documentation) -->
      <!--
      <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
      -->

      <!-- Use the LockOutRealm to prevent attempts to guess user passwords
           via a brute-force attack -->
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <!-- This Realm uses the UserDatabase configured in the global JNDI
             resources under the key "UserDatabase".  Any edits
             that are performed against this UserDatabase are immediately
             available for use by the Realm.  -->
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <!-- SingleSignOn valve, share authentication between web applications
             Documentation at: /docs/config/valve.html -->
        <!--
        <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
        -->

        <!-- Access log processes all example.
             Documentation at: /docs/config/valve.html
             Note: The pattern used is equivalent to using pattern="common" -->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

      </Host>
    </Engine>
  </Service>
</Server>
```
- 整体架构(出自tomcat内核分析)
{%asset_img 整体.png 这部分和我看我tomcat9并不是完全符合%}
##### server.xml对于java代码加载
{%codeblock lang:java Catalina中xml解析%}
    /**
     * Create and configure the Digester we will be using for startup.
     * @return the main digester to parse server.xml
     *   看来此处是用来解析server.xml的,而Digester 相当于一个规则和defaultHandler
     */
    protected Digester createStartDigester() {
        long t1=System.currentTimeMillis();
        // Initialize the digester  --->这个解析器代码比较复杂
        Digester digester = new Digester();
        digester.setValidating(false);
        digester.setRulesValidation(true);
        Map<Class<?>, List<String>> fakeAttributes = new HashMap<>();
        List<String> attrs = new ArrayList<>();
        attrs.add("className");
        fakeAttributes.put(Object.class, attrs);
        digester.setFakeAttributes(fakeAttributes);
        digester.setUseContextClassLoader(true);

        // Configure the actions we will be using

        /**
         *  1.addObjectCreate() -->将遇到的开始标签类进行创建
         *  2.addSetProperties()-->将之填充
         *  3.addSetNext-->使用栈顶次栈顶元素调用内部xx方法进行set
         *      在{@link Catalina#load()}中push(this)这一步就将catalina本身放到了栈顶
         */
        //server
        digester.addObjectCreate("Server",
                                 "org.apache.catalina.core.StandardServer",
                                 "className");
        digester.addSetProperties("Server");
        digester.addSetNext("Server",
                            "setServer",
                            "org.apache.catalina.Server");
        //NamingResourcesImpl
        digester.addObjectCreate("Server/GlobalNamingResources",
                                 "org.apache.catalina.deploy.NamingResourcesImpl");
        digester.addSetProperties("Server/GlobalNamingResources");
        digester.addSetNext("Server/GlobalNamingResources",
                            "setGlobalNamingResources",
                            "org.apache.catalina.deploy.NamingResourcesImpl");

        digester.addObjectCreate("Server/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties("Server/Listener");
        digester.addSetNext("Server/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");
        //service
        digester.addObjectCreate("Server/Service",
                                 "org.apache.catalina.core.StandardService",
                                 "className");
        digester.addSetProperties("Server/Service");
        digester.addSetNext("Server/Service",
                            "addService",
                            "org.apache.catalina.Service");
        //service::listener
        digester.addObjectCreate("Server/Service/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties("Server/Service/Listener");
        digester.addSetNext("Server/Service/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        //service::Executor
        digester.addObjectCreate("Server/Service/Executor",
                         "org.apache.catalina.core.StandardThreadExecutor",
                         "className");
        digester.addSetProperties("Server/Service/Executor");

        digester.addSetNext("Server/Service/Executor",
                            "addExecutor",
                            "org.apache.catalina.Executor");

        //service::listener::Connector
        digester.addRule("Server/Service/Connector",
                         new ConnectorCreateRule());
        digester.addRule("Server/Service/Connector", new SetAllPropertiesRule(
                new String[]{"executor", "sslImplementationName", "protocol"}));
        digester.addSetNext("Server/Service/Connector",
                            "addConnector",
                            "org.apache.catalina.connector.Connector");

        digester.addObjectCreate("Server/Service/Connector/SSLHostConfig",
                                 "org.apache.tomcat.util.net.SSLHostConfig");
        digester.addSetProperties("Server/Service/Connector/SSLHostConfig");
        digester.addSetNext("Server/Service/Connector/SSLHostConfig",
                "addSslHostConfig",
                "org.apache.tomcat.util.net.SSLHostConfig");

        digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                         new CertificateCreateRule());
        digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                         new SetAllPropertiesRule(new String[]{"type"}));
        digester.addSetNext("Server/Service/Connector/SSLHostConfig/Certificate",
                            "addCertificate",
                            "org.apache.tomcat.util.net.SSLHostConfigCertificate");

        digester.addObjectCreate("Server/Service/Connector/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties("Server/Service/Connector/Listener");
        digester.addSetNext("Server/Service/Connector/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        digester.addObjectCreate("Server/Service/Connector/UpgradeProtocol",
                                  null, // MUST be specified in the element
                                  "className");
        digester.addSetProperties("Server/Service/Connector/UpgradeProtocol");
        digester.addSetNext("Server/Service/Connector/UpgradeProtocol",
                            "addUpgradeProtocol",
                            "org.apache.coyote.UpgradeProtocol");

        // Add RuleSets for nested elements   这几个内容比较重要
        digester.addRuleSet(new NamingRuleSet("Server/GlobalNamingResources/"));  //处理全局naming资源
        digester.addRuleSet(new EngineRuleSet("Server/Service/"));			      // 加载Engine容器		
        digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));			  //Host容器

        digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));   //Context
        addClusterRuleSet(digester, "Server/Service/Engine/Host/Cluster/");
        digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/Context/"));  //向context中添加naming资源

        // When the 'engine' is found, set the parentClassLoader.
        digester.addRule("Server/Service/Engine",
                         new SetParentClassLoaderRule(parentClassLoader));
        addClusterRuleSet(digester, "Server/Service/Engine/Cluster/");

        long t2=System.currentTimeMillis();
        if (log.isDebugEnabled()) {
            log.debug("Digester for server.xml created " + ( t2-t1 ));
        }
        return digester;

    }
{%endcodeblock%}
{%codeblock lang:java NamingRuleSet%}
//关于xml中 GlobalNamingResources标签的加载,处理naming资源映射
 public void addRuleInstances(Digester digester) {

        //如果存在Ejb标签创建,并且调用addEjb添加到NamingResources中
        digester.addObjectCreate(prefix + "Ejb",
                                 "org.apache.tomcat.util.descriptor.web.ContextEjb");
        digester.addRule(prefix + "Ejb", new SetAllPropertiesRule());
        digester.addRule(prefix + "Ejb",
                new SetNextNamingRule("addEjb",
                            "org.apache.tomcat.util.descriptor.web.ContextEjb"));
        //Environment 添加
        digester.addObjectCreate(prefix + "Environment",
                                 "org.apache.tomcat.util.descriptor.web.ContextEnvironment");
        digester.addSetProperties(prefix + "Environment");
        digester.addRule(prefix + "Environment",
                            new SetNextNamingRule("addEnvironment",
                            "org.apache.tomcat.util.descriptor.web.ContextEnvironment"));
        //LocalEjb
        digester.addObjectCreate(prefix + "LocalEjb",
                                 "org.apache.tomcat.util.descriptor.web.ContextLocalEjb");
        digester.addRule(prefix + "LocalEjb", new SetAllPropertiesRule());
        digester.addRule(prefix + "LocalEjb",
                new SetNextNamingRule("addLocalEjb",
                            "org.apache.tomcat.util.descriptor.web.ContextLocalEjb"));
        //这个就是默认常见的如数据库配置等
        digester.addObjectCreate(prefix + "Resource",
                                 "org.apache.tomcat.util.descriptor.web.ContextResource");
        digester.addRule(prefix + "Resource", new SetAllPropertiesRule());
        digester.addRule(prefix + "Resource",
                new SetNextNamingRule("addResource",
                            "org.apache.tomcat.util.descriptor.web.ContextResource"));
        //下边三种我不知道啥玩意
        digester.addObjectCreate(prefix + "ResourceEnvRef",
            "org.apache.tomcat.util.descriptor.web.ContextResourceEnvRef");
        digester.addRule(prefix + "ResourceEnvRef", new SetAllPropertiesRule());
        digester.addRule(prefix + "ResourceEnvRef",
                new SetNextNamingRule("addResourceEnvRef",
                            "org.apache.tomcat.util.descriptor.web.ContextResourceEnvRef"));

        digester.addObjectCreate(prefix + "ServiceRef",
            "org.apache.tomcat.util.descriptor.web.ContextService");
        digester.addRule(prefix + "ServiceRef", new SetAllPropertiesRule());
        digester.addRule(prefix + "ServiceRef",
                new SetNextNamingRule("addService",
                            "org.apache.tomcat.util.descriptor.web.ContextService"));

        digester.addObjectCreate(prefix + "Transaction",
            "org.apache.tomcat.util.descriptor.web.ContextTransaction");
        digester.addRule(prefix + "Transaction", new SetAllPropertiesRule());
        digester.addRule(prefix + "Transaction",
                new SetNextNamingRule("setTransaction",
                            "org.apache.tomcat.util.descriptor.web.ContextTransaction"));
    }
    //最终在GlobalNamingResources中就会存在xml中配置的所有属性,这就是jndi
	
{%endcodeblock%}
{%codeblock lang:java SetNextNamingRule%}
   public void end(String namespace, String name) throws Exception {
       //这个类是处理namingRuleSet的setNext情况,和一般的不一样,花里胡哨的
        // Identify the objects to be used
        Object child = digester.peek(0);
        Object parent = digester.peek(1);

        NamingResourcesImpl namingResources = null;
        if (parent instanceof Context) {  //此处判断了栈顶类型,如果是Context 则取getNamingResources,这是一个慢加载代码
            namingResources = ((Context) parent).getNamingResources(); //第一次访问向对应context中创建namingResources,之后按照namingResources的逻辑添加资源映射

        } else {
            namingResources = (NamingResourcesImpl) parent; //否则就是向GlobalNamingResources中添加
        }

        // Call the specified method
        IntrospectionUtils.callMethod1(namingResources, methodName,
                child, paramType, digester.getClassLoader());

    }
{%endcodeblock%}
{%codeblock lang:java SetAllPropertiesRule%}
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */


package org.apache.catalina.startup;

import java.util.HashMap;

import org.apache.tomcat.util.IntrospectionUtils;
import org.apache.tomcat.util.digester.Rule;
import org.xml.sax.Attributes;

/**
 * Rule that uses the introspection utils to set properties.
 *
 * @author Remy Maucherat
 */
public class SetAllPropertiesRule extends Rule {


    // ----------------------------------------------------------- Constructors
    public SetAllPropertiesRule() {}

    public SetAllPropertiesRule(String[] exclude) {
        for (int i=0; i<exclude.length; i++ ) if (exclude[i]!=null) this.excludes.put(exclude[i],exclude[i]);
    }

    // ----------------------------------------------------- Instance Variables
    protected final HashMap<String,String> excludes = new HashMap<>();

    // --------------------------------------------------------- Public Methods


    /**
     * Handle the beginning of an XML element.
     *
     * @param attributes The attributes of this element
     *
     * @exception Exception if a processing error occurs
     */
    @Override
    public void begin(String namespace, String nameX, Attributes attributes)
        throws Exception {

        for (int i = 0; i < attributes.getLength(); i++) {
            String name = attributes.getLocalName(i);
            if ("".equals(name)) {
                name = attributes.getQName(i);
            }
            String value = attributes.getValue(i);
            if ( !excludes.containsKey(name)) {
                if (!digester.isFakeAttribute(digester.peek(), name)
                        && !IntrospectionUtils.setProperty(digester.peek(), name, value) //这里牵扯到digester解析的过程,不仅仅能够该任意对象的某属性直接调用setXX(Value)赋值
                        //还能够对某对象内部成员为Map<String,Object> ,调用对应的setProperty(String name, Object value) 进行赋值,参考ResourceBase子类的创建过程
                        && digester.getRulesValidation()) {
                    digester.getLogger().warn("[SetAllPropertiesRule]{" + digester.getMatch() +
                            "} Setting property '" + name + "' to '" +
                            value + "' did not find a matching property.");
                }
            }
        }

    }


}

{%endcodeblock%}
{%codeblock lang:java Engine复杂添加%}
//EngineRuleSet中逻辑
 public void addRuleInstances(Digester digester) {
        //创建StandardEngine
        digester.addObjectCreate(prefix + "Engine",
                                 "org.apache.catalina.core.StandardEngine",
                                 "className");
        digester.addSetProperties(prefix + "Engine");
        digester.addRule(prefix + "Engine",
                         new LifecycleListenerRule
                         ("org.apache.catalina.startup.EngineConfig",
                          "engineConfigClass"));   //当此处有参数的时候,创建一个 engineConfigClass 如次的类,调用addLifecycleListener加入,明显这是属于Engine的一个监听器
        digester.addSetNext(prefix + "Engine",
                            "setContainer",
                            "org.apache.catalina.Engine");     //将engine加入到Service中

        //Cluster configuration start                                    //添加cluster
        digester.addObjectCreate(prefix + "Engine/Cluster",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Cluster");
        digester.addSetNext(prefix + "Engine/Cluster",
                            "setCluster",
                            "org.apache.catalina.Cluster");
        //Cluster configuration end
         //创建并添加监听器
        digester.addObjectCreate(prefix + "Engine/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Listener");
        digester.addSetNext(prefix + "Engine/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        //处理关于Realm 域添加
        digester.addRuleSet(new RealmRuleSet(prefix + "Engine/"));
        //创建关于Valve标签的创建,并添加到engine中
        digester.addObjectCreate(prefix + "Engine/Valve",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Valve");
        digester.addSetNext(prefix + "Engine/Valve",
                            "addValve",
                            "org.apache.catalina.Valve");
    }
{%endcodeblock%}
{%codeblock lang:java RealmRuleSet%}
//补充关于RealmRuleSet中逻辑
      @Override
    public void addRuleInstances(Digester digester) {
        StringBuilder pattern = new StringBuilder(prefix);
        //这里的逻辑还是用来动态生成关于xml中engine/Realm/Realm这中嵌套结构的规则,也就是说
        Realm.
        //对于container子类使用set  CombinedRealm使用add,这个类子类就是默认xml中的LockOutRealm
        for (int i = 0; i < MAX_NESTED_REALM_LEVELS; i++) {
            if (i > 0) {
                pattern.append('/');
            }
            pattern.append("Realm");
            addRuleInstances(digester, pattern.toString(), i == 0 ? "setRealm" : "addRealm");
        }
    }

    private void addRuleInstances(Digester digester, String pattern, String methodName) {
        //上边的函数用来判断使用哪个函数,这里就是创建org.apache.catalina.Realm,并且添加到Engine中
        digester.addObjectCreate(pattern, null /* MUST be specified in the element */,
                "className");
        digester.addSetProperties(pattern);
        digester.addSetNext(pattern, methodName, "org.apache.catalina.Realm");
        digester.addRuleSet(new CredentialHandlerRuleSet(pattern + "/"));
    }
{%endcodeblock%}
{%codeblock lang:java SetAllPropertiesRule%}
//这个类是关于某些属性设置的
public SetAllPropertiesRule() {}

    public SetAllPropertiesRule(String[] exclude) {
        for (int i=0; i<exclude.length; i++ ) if (exclude[i]!=null) this.excludes.put(exclude[i],exclude[i]);   ///根据exclude配置此加载器,要去除的属性
    }
    
     public void begin(String namespace, String nameX, Attributes attributes)
        throws Exception {

        for (int i = 0; i < attributes.getLength(); i++) {
            String name = attributes.getLocalName(i);
            if ("".equals(name)) {
                name = attributes.getQName(i);
            }
            String value = attributes.getValue(i);
            if ( !excludes.containsKey(name)) {  //exclude中不包含的属性进行下面逻辑
                if (!digester.isFakeAttribute(digester.peek(), name)
                        && !IntrospectionUtils.setProperty(digester.peek(), name, value)
                        && digester.getRulesValidation()) {
                    digester.getLogger().warn("[SetAllPropertiesRule]{" + digester.getMatch() +
                            "} Setting property '" + name + "' to '" +
                            value + "' did not find a matching property.");
                }
            }
        }

    }
{%endcodeblock%}
{%codeblock lang:java ConnectorCreateRule%}
  @Override
  //创建connector的逻辑
    public void begin(String namespace, String name, Attributes attributes)
            throws Exception {
        Service svc = (Service)digester.peek();
        Executor ex = null;
        //根据connector标签的中"executor" ,从Service中取executor,并且加入到来连接器中,默认情况Service是没有executor的
        if ( attributes.getValue("executor")!=null ) {
            ex = svc.getExecutor(attributes.getValue("executor"));
        }
        Connector con = new Connector(attributes.getValue("protocol"));
        if (ex != null) {
            setExecutor(con, ex);
        }
        //sslImplementationName这个属性默认也是没有的
        String sslImplementationName = attributes.getValue("sslImplementationName");
        if (sslImplementationName != null) {
            setSSLImplementationName(con, sslImplementationName);
        }
        digester.push(con);  //将连接器放到栈顶,是因为接下来对于连接器标签内部也可以进行加载处理
    }

{%endcodeblock%}
{%codeblock lang:java host加载%}
public class HostRuleSet implements RuleSet {
 public void addRuleInstances(Digester digester) {

   //标准三套:创建StandardHost
        digester.addObjectCreate(prefix + "Host",
                                 "org.apache.catalina.core.StandardHost",
                                 "className");
        digester.addSetProperties(prefix + "Host");
        digester.addRule(prefix + "Host",
                         new CopyParentClassLoaderRule()); //处理父加载器

        digester.addRule(prefix + "Host",
                         new LifecycleListenerRule
                         ("org.apache.catalina.startup.HostConfig",
                          "hostConfigClass")); //添加HostConfig监听器
        digester.addSetNext(prefix + "Host",
                            "addChild",
                            "org.apache.catalina.Container");

        digester.addCallMethod(prefix + "Host/Alias",
                               "addAlias", 0);

        //Cluster configuration start
        digester.addObjectCreate(prefix + "Host/Cluster",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Host/Cluster");
        digester.addSetNext(prefix + "Host/Cluster",
                            "setCluster",
                            "org.apache.catalina.Cluster");
        //Cluster configuration end

        digester.addObjectCreate(prefix + "Host/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Host/Listener");
        digester.addSetNext(prefix + "Host/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        digester.addRuleSet(new RealmRuleSet(prefix + "Host/"));

        digester.addObjectCreate(prefix + "Host/Valve",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Host/Valve");
        digester.addSetNext(prefix + "Host/Valve",
                            "addValve",
                            "org.apache.catalina.Valve");
    }
}    
{%endcodeblock%}
{%codeblock lang:java ContextRuleSet%}
//当遇到Server/Service/Engine/Host/Content 时进行创建,这是通过server.xml的方式配置项目,总体而言我不是要说具体的解析过程而是强调
//除了content.xml配置,可以通过server.xml进行配置
public void addRuleInstances(Digester digester) {
//这其中包含了许多不太了解的类,但是可以反映出content也就是表示项目的结构
        if (create) {
            digester.addObjectCreate(prefix + "Context",
                    "org.apache.catalina.core.StandardContext", "className");
            digester.addSetProperties(prefix + "Context");
        } else {
            digester.addRule(prefix + "Context", new SetContextPropertiesRule());
        }
 //创建content,并加入到父容器Host中
        if (create) {
            digester.addRule(prefix + "Context",
                             new LifecycleListenerRule
                                 ("org.apache.catalina.startup.ContextConfig",
                                  "configClass"));
            digester.addSetNext(prefix + "Context",
                                "addChild",
                                "org.apache.catalina.Container");
        }

        digester.addObjectCreate(prefix + "Context/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Listener");
        digester.addSetNext(prefix + "Context/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        digester.addObjectCreate(prefix + "Context/Loader",
                            "org.apache.catalina.loader.WebappLoader",
                            "className");
        digester.addSetProperties(prefix + "Context/Loader");
        digester.addSetNext(prefix + "Context/Loader",
                            "setLoader",
                            "org.apache.catalina.Loader");

        digester.addObjectCreate(prefix + "Context/Manager",
                                 "org.apache.catalina.session.StandardManager",
                                 "className");
        digester.addSetProperties(prefix + "Context/Manager");
        digester.addSetNext(prefix + "Context/Manager",
                            "setManager",
                            "org.apache.catalina.Manager");

        digester.addObjectCreate(prefix + "Context/Manager/Store",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Manager/Store");
        digester.addSetNext(prefix + "Context/Manager/Store",
                            "setStore",
                            "org.apache.catalina.Store");

        digester.addObjectCreate(prefix + "Context/Manager/SessionIdGenerator",
                                 "org.apache.catalina.util.StandardSessionIdGenerator",
                                 "className");
        digester.addSetProperties(prefix + "Context/Manager/SessionIdGenerator");
        digester.addSetNext(prefix + "Context/Manager/SessionIdGenerator",
                            "setSessionIdGenerator",
                            "org.apache.catalina.SessionIdGenerator");

        digester.addObjectCreate(prefix + "Context/Parameter",
                                 "org.apache.tomcat.util.descriptor.web.ApplicationParameter");
        digester.addSetProperties(prefix + "Context/Parameter");
        digester.addSetNext(prefix + "Context/Parameter",
                            "addApplicationParameter",
                            "org.apache.tomcat.util.descriptor.web.ApplicationParameter");

        digester.addRuleSet(new RealmRuleSet(prefix + "Context/"));

        digester.addObjectCreate(prefix + "Context/Resources",
                                 "org.apache.catalina.webresources.StandardRoot",
                                 "className");
        digester.addSetProperties(prefix + "Context/Resources");
        digester.addSetNext(prefix + "Context/Resources",
                            "setResources",
                            "org.apache.catalina.WebResourceRoot");

        digester.addObjectCreate(prefix + "Context/Resources/PreResources",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Resources/PreResources");
        digester.addSetNext(prefix + "Context/Resources/PreResources",
                            "addPreResources",
                            "org.apache.catalina.WebResourceSet");

        digester.addObjectCreate(prefix + "Context/Resources/JarResources",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Resources/JarResources");
        digester.addSetNext(prefix + "Context/Resources/JarResources",
                            "addJarResources",
                            "org.apache.catalina.WebResourceSet");

        digester.addObjectCreate(prefix + "Context/Resources/PostResources",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Resources/PostResources");
        digester.addSetNext(prefix + "Context/Resources/PostResources",
                            "addPostResources",
                            "org.apache.catalina.WebResourceSet");


        digester.addObjectCreate(prefix + "Context/ResourceLink",   //关于资源引用标签的解析就在此处
                "org.apache.tomcat.util.descriptor.web.ContextResourceLink");
        digester.addSetProperties(prefix + "Context/ResourceLink");
        digester.addRule(prefix + "Context/ResourceLink",
                new SetNextNamingRule("addResourceLink",
                        "org.apache.tomcat.util.descriptor.web.ContextResourceLink"));

        digester.addObjectCreate(prefix + "Context/Valve",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Valve");
        digester.addSetNext(prefix + "Context/Valve",
                            "addValve",
                            "org.apache.catalina.Valve");

        digester.addCallMethod(prefix + "Context/WatchedResource",
                               "addWatchedResource", 0);

        digester.addCallMethod(prefix + "Context/WrapperLifecycle",
                               "addWrapperLifecycle", 0);

        digester.addCallMethod(prefix + "Context/WrapperListener",
                               "addWrapperListener", 0);

        digester.addObjectCreate(prefix + "Context/JarScanner",
                                 "org.apache.tomcat.util.scan.StandardJarScanner",
                                 "className");
        digester.addSetProperties(prefix + "Context/JarScanner");
        digester.addSetNext(prefix + "Context/JarScanner",
                            "setJarScanner",
                            "org.apache.tomcat.JarScanner");

        digester.addObjectCreate(prefix + "Context/JarScanner/JarScanFilter",
                                 "org.apache.tomcat.util.scan.StandardJarScanFilter",
                                 "className");
        digester.addSetProperties(prefix + "Context/JarScanner/JarScanFilter");
        digester.addSetNext(prefix + "Context/JarScanner/JarScanFilter",
                            "setJarScanFilter",
                            "org.apache.tomcat.JarScanFilter");

        digester.addObjectCreate(prefix + "Context/CookieProcessor",
                                 "org.apache.tomcat.util.http.Rfc6265CookieProcessor",
                                 "className");
        digester.addSetProperties(prefix + "Context/CookieProcessor");
        digester.addSetNext(prefix + "Context/CookieProcessor",
                            "setCookieProcessor",
                            "org.apache.tomcat.util.http.CookieProcessor");
    }
{%endcodeblock%}
#### 接口
##### 1. Lifecycle 接口
 - 该接口的子类要按照一定状态和顺序,完成init start stop destory 函数,并且该子类可以作为lifeEvent的事件源,并且调整life部件的状态
 {%codeblock lang:java lifeBase%}
 //在生命周期初始化开端 改变状态,调用子类internal
  public final synchronized void init() throws LifecycleException {
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
        }

        try {
            setStateInternal(LifecycleState.INITIALIZING, null, false);   // INITIALIZING(false, Lifecycle.BEFORE_INIT_EVENT),
            initInternal();
            setStateInternal(LifecycleState.INITIALIZED, null, false); //INITIALIZED(false, Lifecycle.AFTER_INIT_EVENT),
        } catch (Throwable t) {
            handleSubClassException(t, "lifecycleBase.initFail", toString());
        }
    }
  private synchronized void setStateInternal(LifecycleState state, Object data, boolean check)
            throws LifecycleException {

        if (log.isDebugEnabled()) {
            log.debug(sm.getString("lifecycleBase.setState", this, state));
        }

        if (check) {
            // Must have been triggered by one of the abstract methods (assume
            // code in this class is correct)
            // null is never a valid state
            if (state == null) {
                invalidTransition("null");
                // Unreachable code - here to stop eclipse complaining about
                // a possible NPE further down the method
                return;
            }

            // Any method can transition to failed
            // startInternal() permits STARTING_PREP to STARTING
            // stopInternal() permits STOPPING_PREP to STOPPING and FAILED to
            // STOPPING
            if (!(state == LifecycleState.FAILED ||
                    (this.state == LifecycleState.STARTING_PREP &&
                            state == LifecycleState.STARTING) ||
                    (this.state == LifecycleState.STOPPING_PREP &&
                            state == LifecycleState.STOPPING) ||
                    (this.state == LifecycleState.FAILED &&
                            state == LifecycleState.STOPPING))) {
                // No other transition permitted
                invalidTransition(state.name());
            }
        }

        this.state = state;
        String lifecycleEvent = state.getLifecycleEvent();
        if (lifecycleEvent != null) {  //此处事件源触发事件
            fireLifecycleEvent(lifecycleEvent, data);  //这个逻辑是当发生状态改变时,调用监听器,可以理解为事件发生,触发该事件源的监听器
        }
    }
{%endcodeblock%}

- lifecycleMbeanBase实现了其下属子类的共同接口函数,实现共同逻辑,类似的还有ContainBase

 {%codeblock lang:java lifecycleMbeanBase%}
 //该类只在init和destory有用
 protected void initInternal() throws LifecycleException {
   //将life部件加入到mServer
        // If oname is not null then registration has already happened via
        // preRegister().
        if (oname == null) {
            mserver = Registry.getRegistry(null, null).getMBeanServer();

            oname = register(this, getObjectNameKeyProperties());
        }
    }
    //卸除mbean
     protected void destroyInternal() throws LifecycleException {
        unregister(oname);
    }
    //没有实现startInternal
 {%endcodeblock%}
 -  衍生出了大部分第三层父类, ContainBase,LifecycleMBeanBase则实现了internal
##### ContainerBase 
{% asset_img ContainerBase.png %}
此类加载过程通常都是 container.start()->ContainerBase.startInternal()->((导致子类容器start) &&( 自身pieple start->关联valve->start)->启动线程)
{%codeblock lang:java%}
// init 逻辑
 protected void initInternal() throws LifecycleException {
        reconfigureStartStopExecutor(getStartStopThreadsInternal());  //设置线程池
        super.initInternal();  //父类是mbean 所以加入到jmx
    }
  //关于线程的逻辑,这里要清楚要自己看看这几个线程类的继承关系
    protected ExecutorService startStopExecutor; //startStopExecutor是该抽象类的属性
        private void reconfigureStartStopExecutor(int threads) {
        if (threads == 1) {
        //这里的逻辑是,如果startStopExecutor为null,那么就实例化为InlineExecutorService
            if (!(startStopExecutor instanceof InlineExecutorService)) {  
                startStopExecutor = new InlineExecutorService();   
            }
        } else {
        //其他情况说明该startStopExecutor接口被实例化了,比如第二次调用该函数就调用如下逻辑
        //当为线程池时进行设置,否则将该startStopExecutor实例化为线程池类型,当再一次调用时就会进行改变
            if (startStopExecutor instanceof ThreadPoolExecutor) {
                ((ThreadPoolExecutor) startStopExecutor).setMaximumPoolSize(threads);
                ((ThreadPoolExecutor) startStopExecutor).setCorePoolSize(threads);
            } else {
                BlockingQueue<Runnable> startStopQueue = new LinkedBlockingQueue<>();
                ThreadPoolExecutor tpe = new ThreadPoolExecutor(threads, threads, 10,
                        TimeUnit.SECONDS, startStopQueue,
                        new StartStopThreadFactory(getName() + "-startStop-"));
                tpe.allowCoreThreadTimeOut(true);
                startStopExecutor = tpe;
            }
        }
    }
//被engine调用的获取realm逻辑    
   public Realm getRealm() {

        Lock l = realmLock.readLock();
        l.lock();
        try {
            if (realm != null)
                return realm;
            if (parent != null)
                return parent.getRealm();  //如果该容器体系不存在realm则返回null
            return null;
        } finally {
            l.unlock();
        }
    }
 //设置Realm
 public void setRealm(Realm realm) {

        Lock l = realmLock.writeLock();
        l.lock();
        try {
            // Change components if necessary
            Realm oldRealm = this.realm;
            if (oldRealm == realm)
                return;
            this.realm = realm;

            // Stop the old component if necessary
            if (getState().isAvailable() && (oldRealm != null) &&  //对于持有作用域的组件,只能持有一个域,并且域也是一种生命周期组件
                (oldRealm instanceof Lifecycle)) {
                try {
                    ((Lifecycle) oldRealm).stop();
                } catch (LifecycleException e) {
                    log.error("ContainerBase.setRealm: stop: ", e);
                }
            }

            // Start the new component if necessary
            if (realm != null)
                realm.setContainer(this);
            if (getState().isAvailable() && (realm != null) &&
                (realm instanceof Lifecycle)) {
                try {
                    ((Lifecycle) realm).start();  //启动域
                } catch (LifecycleException e) {
                    log.error("ContainerBase.setRealm: start: ", e);
                }
            }

            // Report this property change to interested listeners
            support.firePropertyChange("realm", oldRealm, this.realm);  //类似于javafx的作用,当发生改变时触发事件,实际还是通过监听模式实现的
        } finally {
            l.unlock();
        }

    }
    
//容器start    
    protected synchronized void startInternal() throws LifecycleException {

        // Start our subordinate components, if any
        logger = null;
        getLogger();
        Cluster cluster = getClusterInternal();  //(cluster)是和集群相关的
        if (cluster instanceof Lifecycle) {
            ((Lifecycle) cluster).start();
        }
        Realm realm = getRealmInternal();
        /**
         * 此处关于默认情况的逻辑进行lockoutRealm的生命周期init到start,看看lifeBase中的init/start函数并非在对应的函数其组件一定会执行对应生命周期函数,执行
         * 对应的函数取决于容器的state
         * 在处理lockoutRealm的过程中,也遍历启动了其内部realm的生命周期函数 默认就是UserDataRealm
         */
        if (realm instanceof Lifecycle) {
            ((Lifecycle) realm).start();   //此处激活容器所属域,默认此处启动的是lockoutRealm,其内部可以含有子类realm,默认是userDataRealm
        }

        // Start our child containers, if any
        Container children[] = findChildren();  //xml中的host等都是子类容器
        /**
         * ExecutorService::<T> Future<T> submit(Callable<T> task);
         * 返回的Future.get表示执行结果
         * 也就是说实际上此处执行的就是自容器的start函数,不过是通过线程执行的
         */
        List<Future<Void>> results = new ArrayList<>();
        for (int i = 0; i < children.length; i++) {
            results.add(startStopExecutor.submit(new StartChild(children[i])));  //通过线程池来执行StartChild(Call子类)中的call函数
        }

        boolean fail = false;
        for (Future<Void> result : results) {
            try {
                result.get(); //get表示
            } catch (Exception e) {
                log.error(sm.getString("containerBase.threadedStartFailed"), e);
                fail = true;
            }

        }
        if (fail) {
            throw new LifecycleException(
                    sm.getString("containerBase.threadedStartFailed"));
        }

        // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle)
            ((Lifecycle) pipeline).start(); //启动pipeline


        setState(LifecycleState.STARTING);

        // Start our thread
        threadStart(); //启动线程

    }
 //关于后台线程,一个内部线程类
  protected class ContainerBackgroundProcessor implements Runnable {

        @Override
        public void run() {
            Throwable t = null;
            String unexpectedDeathMessage = sm.getString(
                    "containerBase.backgroundProcess.unexpectedThreadDeath",
                    Thread.currentThread().getName());
            try {
                while (!threadDone) { //当线程没有结束时，不断执行
                    try {
                        Thread.sleep(backgroundProcessorDelay * 1000L);
                    } catch (InterruptedException e) {
                        // Ignore
                    }
                    if (!threadDone) {
                        processChildren(ContainerBase.this);
                    }
                }
            } catch (RuntimeException|Error e) {
                t = e;
                throw e;
            } finally {
                if (!threadDone) {
                    log.error(unexpectedDeathMessage, t);
                }
            }
        }

        protected void processChildren(Container container) {
            ClassLoader originalClassLoader = null;

            try {
                if (container instanceof Context) {
                    Loader loader = ((Context) container).getLoader();
                    // Loader will be null for FailedContext instances
                    if (loader == null) {
                        return;
                    }

                    // Ensure background processing for Contexts and Wrappers
                    // is performed under the web app's class loader
                    originalClassLoader = ((Context) container).bind(false, null);
                }
                container.backgroundProcess(); //实际上执行的是个这个函数
                Container[] children = container.findChildren();
                for (int i = 0; i < children.length; i++) {
                    if (children[i].getBackgroundProcessorDelay() <= 0) {
                        processChildren(children[i]);
                    }
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error("Exception invoking periodic operation: ", t);
            } finally {
                if (container instanceof Context) {
                    ((Context) container).unbind(false, originalClassLoader);
               }
            }
        }
    }
    
    public void backgroundProcess() {
     //不断执行内部这些成员的back，不代表所有成员都实现了该函数
        if (!getState().isAvailable())
            return;

        Cluster cluster = getClusterInternal();
        if (cluster != null) {
            try {
                cluster.backgroundProcess();
            } catch (Exception e) {
                log.warn(sm.getString("containerBase.backgroundProcess.cluster",
                        cluster), e);
            }
        }
        Realm realm = getRealmInternal();
        if (realm != null) {
            try {
                realm.backgroundProcess();
            } catch (Exception e) {
                log.warn(sm.getString("containerBase.backgroundProcess.realm", realm), e);
            }
        }
        Valve current = pipeline.getFirst();
        while (current != null) {
            try {
                current.backgroundProcess();
            } catch (Exception e) {
                log.warn(sm.getString("containerBase.backgroundProcess.valve", current), e);
            }
            current = current.getNext();
        }
        fireLifecycleEvent(Lifecycle.PERIODIC_EVENT, null);
    }
{%endcodeblock%}
{%codeblock lang:java ContainBase容器部分功能%}
//添加子容器
 public void addChild(Container child) {
        if (Globals.IS_SECURITY_ENABLED) {
            PrivilegedAction<Void> dp =
                new PrivilegedAddChild(child);
            AccessController.doPrivileged(dp);
        } else {
            addChildInternal(child);
        }
    }

    private void addChildInternal(Container child) {

        if( log.isDebugEnabled() )
            log.debug("Add child " + child + " " + this);
        synchronized(children) {
            if (children.get(child.getName()) != null)
                throw new IllegalArgumentException("addChild:  Child name '" +
                                                   child.getName() +
                                                   "' is not unique");
            child.setParent(this);  // May throw IAE
            children.put(child.getName(), child);
        }

        // Start child
        // Don't do this inside sync block - start can be a slow process and
        // locking the children object can cause problems elsewhere
        try {
            if ((getState().isAvailable() ||
                    LifecycleState.STARTING_PREP.equals(getState())) &&
                    startChildren) {
                child.start();  //并且会启动此时加入的子容器,  content的加入后 就是在此时启动的,根据触发条件来看,只有该父容器处于有效 或者准备阶段会启动父容器, mxl解析创建的时候是不会的
            }
        } catch (LifecycleException e) {
            log.error("ContainerBase.addChild: start: ", e);
            throw new IllegalStateException("ContainerBase.addChild: start: " + e);
        } finally {
            fireContainerEvent(ADD_CHILD_EVENT, child); //会触发监听器
        }
    }
		//移除
	    public void removeChild(Container child) {

        if (child == null) {
            return;
        }

        try {
            if (child.getState().isAvailable()) {
                child.stop();
            }
        } catch (LifecycleException e) {
            log.error("ContainerBase.removeChild: stop: ", e);
        }

        try {
            // child.destroy() may have already been called which would have
            // triggered this call. If that is the case, no need to destroy the
            // child again.
            if (!LifecycleState.DESTROYING.equals(child.getState())) {
                child.destroy();
            }
        } catch (LifecycleException e) {
            log.error("ContainerBase.removeChild: destroy: ", e);
        }

        synchronized(children) {
            if (children.get(child.getName()) == null)
                return;
            children.remove(child.getName());
        }

        fireContainerEvent(REMOVE_CHILD_EVENT, child);
    }
	
	
  //触发
	public void fireContainerEvent(String type, Object data) {

        if (listeners.size() < 1)
            return;

        ContainerEvent event = new ContainerEvent(this, type, data);  //ContainerEvent 和lifeEvent都是EventObject子类,它不记录容器状态
        // Note for each uses an iterator internally so this is safe
        for (ContainerListener listener : listeners) {
            listener.containerEvent(event); //触发监听器
        }
    }
{%endcodeblock%}
#### tomcat关键部分及各部分作用
##### 1. server.xml 该配置文件实际非常详实的说明了tomcat的架构
##### 2. 类加载器
{% asset_img classLoader.png %}

##### 3. Server继承图
{% asset_img Server.png %}
-  init阶段
```java
 protected void initInternal() throws LifecycleException {

        super.initInternal();  //将该对象加入到jmxServer中

        // Register global String cache
        // Note although the cache is global, if there are multiple Servers
        // present in the JVM (may happen when embedding) then the same cache
        // will be registered under multiple names
        onameStringCache = register(new StringCache(), "type=StringCache");  //将StringCache加入

        // Register the MBeanFactory
        MBeanFactory factory = new MBeanFactory();
        factory.setContainer(this);
        onameMBeanFactory = register(factory, "type=MBeanFactory");

        // Register the naming resources
        globalNamingResources.init(); // 此处完成了对于资源的处理,要理解就要看看
globalNamingResources在digsiter的初始化
        // Populate the extension validator with JARs from common and shared
        // class loaders
        if (getCatalina() != null) {
            ClassLoader cl = getCatalina().getParentClassLoader();
            // Walk the class loader hierarchy. Stop at the system class loader.
            // This will add the shared (if present) and common class loaders
            while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
                if (cl instanceof URLClassLoader) {
                    URL[] urls = ((URLClassLoader) cl).getURLs();
                    for (URL url : urls) {
                        if (url.getProtocol().equals("file")) {
                            try {
                                File f = new File (url.toURI());
                                if (f.isFile() &&
                                        f.getName().endsWith(".jar")) {
                                    ExtensionValidator.addSystemResource(f);
                                }
                            } catch (URISyntaxException e) {
                                // Ignore
                            } catch (IOException e) {
                                // Ignore
                            }
                        }
                    }
                }
                cl = cl.getParent();
            }
        }
        // Initialize our defined Services
        for (int i = 0; i < services.length; i++) {
            services[i].init();  //改变状态 加入jmx
        }
    }
```
```java
//globalNamingResources init时触发情况,将server.xml GlobalNamingResources标签中的所有Resource envs resourceLinks 加入到jmx中
  protected void initInternal() throws LifecycleException {
        super.initInternal();

        // Set this before we register currently known naming resources to avoid
        // timing issues. Duplication registration is not an issue.
        resourceRequireExplicitRegistration = true;

        for (ContextResource cr : resources.values()) {
            try {
                MBeanUtils.createMBean(cr);
            } catch (Exception e) {
                log.warn(sm.getString(
                        "namingResources.mbeanCreateFail", cr.getName()), e);
            }
        }

        for (ContextEnvironment ce : envs.values()) {
            try {
                MBeanUtils.createMBean(ce);
            } catch (Exception e) {
                log.warn(sm.getString(
                        "namingResources.mbeanCreateFail", ce.getName()), e);
            }
        }

        for (ContextResourceLink crl : resourceLinks.values()) {
            try {
                MBeanUtils.createMBean(crl);
            } catch (Exception e) {
                log.warn(sm.getString(
                        "namingResources.mbeanCreateFail", crl.getName()), e);
            }
        }
    }
```
-  start阶段
{%codeblock lang:java%}
 protected void startInternal() throws LifecycleException {
//首先lifeBase将该部件state由INITIALIZED->STARTING_PREP(调用对应监听器),再由server
        fireLifecycleEvent(CONFIGURE_START_EVENT, null); //仅仅触发了NamingContentListenter的CONFIGURE_START_EVENT阶段
        setState(LifecycleState.STARTING);

        globalNamingResources.start(); //触发STARTING_PREP时候globalNamingResources的监听器,默认没有

        // Start our defined Services
        synchronized (servicesLock) {
            for (int i = 0; i < services.length; i++) {
                services[i].start();
            }
        }
    }
{%endcodeblock%}
##### 4. Service继承图
{% asset_img Service.png %} 
{%codeblock lang:java Service init%}
//默认没有监听器
- init阶段
 protected void initInternal() throws LifecycleException {

        super.initInternal();  //加入到jmx

        if (engine != null) {
            engine.init();  //
        }

        // Initialize any Executors   ,关于Executor 实际上就是线程池,而线程底层只想runnable还是通过
        //Thread,tomcat实现了一个StandardThreadExecutor,该对象时xml解析时创建的,默认xml这部分被注释了,同样属于生命周期组件 
        //拥有ThreadPoolExecutor,该对象继承于jdk中的ThreadPoolExecutor,这是线程池之一
        for (Executor executor : findExecutors()) {
            if (executor instanceof JmxEnabled) {
                ((JmxEnabled) executor).setDomain(getDomain());
            }
            executor.init();  //
        }

        // Initialize mapper listener
        mapperListener.init();  //加入jmx,这个监听器不会在Engine组件的生命周期被激活

        // Initialize our defined Connectors
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                connector.init();  //加入jmx
            }
        }
    }
{%endcodeblock%}
- start阶段
{%codeblock lang:java%}
    protected void startInternal() throws LifecycleException {

        if(log.isInfoEnabled())
            log.info(sm.getString("standardService.start.name", this.name));  //明显的service没有在start前要执行的监听器
        setState(LifecycleState.STARTING);

        // Start our defined Container first  engine也是容器子类
        if (engine != null) {
            synchronized (engine) {
                engine.start();
            }
        }

        synchronized (executors) {
            for (Executor executor: executors) { //默认不存在
                executor.start();
            }
        }

        mapperListener.start();  //mapper就是对于url的映射,当容器engine及其子容器启动后,mapper作为life监听器和container监听器加入到各个子容器

        // Start our defined Connectors second
        synchronized (connectorsLock) {
            for (Connector connector: connectors) {
                // If it has already failed, don't try and start it
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            }
        }
    }
{%endcodeblock%}
##### 5. Engine继承图
{% asset_img Engine.png %}
- init阶段
```java
//Engine init时的逻辑
 protected void initInternal() throws LifecycleException {
        // Ensure that a Realm is present before any attempt is made to start
        // one. This will create the default NullRealm if necessary.
        getRealm();
        super.initInternal();  //父类 : ContainerBase-->LifecycleMBeanBase
    }
    
    //关于域的获取,realm是一个has a的一个成员变量
 public Realm getRealm() {
      //当自身不存在realm,则表示为NullRealm
        Realm configured = super.getRealm();
        // If no set realm has been called - default to NullRealm
        // This can be overridden at engine, context and host level
        if (configured == null) {
            configured = new NullRealm();
            this.setRealm(configured);
        }
        return configured;
    }
```
补充关于StandardThreadExecutor
{% asset_img StandardThreadExecutor.png %}
- start阶段
{%codeblock lang:java Engine%}
 protected synchronized void startInternal() throws LifecycleException {

        // Log our server identification information
        if(log.isInfoEnabled())
            log.info( "Starting Servlet Engine: " + ServerInfo.getServerInfo());

        // Standard container startup
        super.startInternal();  //engine是容器类们的父容器,通过此处进行containerBase::init,完成子类容器的加载,比较复杂
    }
{%endcodeblock%}
//子容器host的start触发
{%asset_img StandardHost.png%}
{%codeblock lang:java Host%}
//在这过程中触发了其初始化阶段,如继承图可易知加入jmx,启动后台线程池
  protected synchronized void startInternal() throws LifecycleException {

        // Set error report valve  下边的逻辑只是给host.pipe加入了ReportValve这个阀
        String errorValve = getErrorReportValveClass();
        if ((errorValve != null) && (!errorValve.equals(""))) {
            try {
                boolean found = false;
                Valve[] valves = getPipeline().getValves();
                for (Valve valve : valves) {
                    if (errorValve.equals(valve.getClass().getName())) {
                        found = true;
                        break;
                    }
                }
                if(!found) {
                    Valve valve =
                        (Valve) Class.forName(errorValve).getConstructor().newInstance();
                    getPipeline().addValve(valve);
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error(sm.getString(
                        "standardHost.invalidErrorReportValveClass",
                        errorValve), t);
            }
        }
        super.startInternal(); //执行本身容器逻辑,仅仅是开启了内部pipe.start
    }
{%endcodeblock%}
##### 6. connector
{% asset_img connector.png %}
{%codeblock lang:java Connector%}
- init阶段
 public Connector(String protocol) {
        boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
                AprLifecycleListener.getUseAprConnector();  //根据之前Server apr监听器的处理,判断是否能够开启apr io,默认情况是无法开启的

        if ("HTTP/1.1".equals(protocol) || protocol == null) {
            if (aprConnector) {
                protocolHandlerClassName = "org.apache.coyote.http11.Http11AprProtocol";
            } else {
                protocolHandlerClassName = "org.apache.coyote.http11.Http11NioProtocol";  //对于http1.1连接器,tomcat 9版本使用nio
            }
        } else if ("AJP/1.3".equals(protocol)) {
            if (aprConnector) {
                protocolHandlerClassName = "org.apache.coyote.ajp.AjpAprProtocol";
            } else {
                protocolHandlerClassName = "org.apache.coyote.ajp.AjpNioProtocol";   //对于ajp 使用ajpNio
            }
        } else {
            protocolHandlerClassName = protocol;
        }

        // Instantiate protocol handler
        ProtocolHandler p = null;
        try {
            Class<?> clazz = Class.forName(protocolHandlerClassName); //这里体现了类加载的作用之一,当tomcat发布出去后,lib包的加载能力取决于类加载器
            p = (ProtocolHandler) clazz.getConstructor().newInstance();  //加载不同ProtocolHandler,
        } catch (Exception e) {
            log.error(sm.getString(
                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);
        } finally {
            this.protocolHandler = p;
        }

        // Default for Connector depends on this system property
        setThrowOnFailure(Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"));
    }
{%endcodeblock%}
{%asset_img Http11NioProtocol.png%}
{%codeblock lang:java ProtocolHandler%}
//关于handler
//创建时 创建了 endpoint  以及ConnectionHandler
  public Http11NioProtocol() {
        super(new NioEndpoint());
    }
    //
       public AbstractHttp11JsseProtocol(AbstractJsseEndpoint<S,?> endpoint) {
        super(endpoint);
    }
    //明显这里的endpoint很重要
    public AbstractHttp11Protocol(AbstractEndpoint<S,?> endpoint) {
        super(endpoint);
        setConnectionTimeout(Constants.DEFAULT_CONNECTION_TIMEOUT);
        ConnectionHandler<S> cHandler = new ConnectionHandler<>(this); 
        setHandler(cHandler);  //设置ConnectionHandler
        getEndpoint().setHandler(cHandler);  //endpoint.setHandler
    }
    public AbstractProtocol(AbstractEndpoint<S,?> endpoint) {
        this.endpoint = endpoint; //endpoint
        setConnectionLinger(Constants.DEFAULT_CONNECTION_LINGER);
        setTcpNoDelay(Constants.DEFAULT_TCP_NO_DELAY);
    }
{%endcodeblock%}
{%codeblock lang:java 初始化(Connector)%}
protected void initInternal() throws LifecycleException {

        super.initInternal();  //加入jmx

        if (protocolHandler == null) {
            throw new LifecycleException(
                    sm.getString("coyoteConnector.protocolHandlerInstantiationFailed"));
        }

        // Initialize adapter
        adapter = new CoyoteAdapter(this);  //这个adapter貌似是处理servlet的
        protocolHandler.setAdapter(adapter);

        // Make sure parseBodyMethodsSet has a default
        if (null == parseBodyMethodsSet) {
            setParseBodyMethods(getParseBodyMethods()); //当post请求解析body
        }

        if (protocolHandler.isAprRequired() && !AprLifecycleListener.isAprAvailable()) {
            throw new LifecycleException(sm.getString("coyoteConnector.protocolHandlerNoApr",
                    getProtocolHandlerClassName()));
        }
        if (AprLifecycleListener.isAprAvailable() && AprLifecycleListener.getUseOpenSSL() &&
                protocolHandler instanceof AbstractHttp11JsseProtocol) {
            AbstractHttp11JsseProtocol<?> jsseProtocolHandler =
                    (AbstractHttp11JsseProtocol<?>) protocolHandler;
            if (jsseProtocolHandler.isSSLEnabled() &&
                    jsseProtocolHandler.getSslImplementationName() == null) {
                // OpenSSL is compatible with the JSSE configuration, so use it if APR is available
                jsseProtocolHandler.setSslImplementationName(OpenSSLImplementation.class.getName());
            }
        }

        try {
            protocolHandler.init();  //protocolHandler的初始化
        } catch (Exception e) {
            throw new LifecycleException(
                    sm.getString("coyoteConnector.protocolHandlerInitializationFailed"), e);
        }
    }
{%endcodeblock%}
{%codeblock lang:java protocolHandler的初始化 %}
//AbstractHttp11Protocol
  public void init() throws Exception {
        // Upgrade protocols have to be configured first since the endpoint
        // init (triggered via super.init() below) uses this list to configure
        // the list of ALPN protocols to advertise
        for (UpgradeProtocol upgradeProtocol : upgradeProtocols) {
            configureUpgradeProtocol(upgradeProtocol);  //如果连接器没有配置upgradeProtocol  改良的协议,那么此处无效
        }

        super.init();
    }
    //AbstractProtocol
     public void init() throws Exception {
        if (getLog().isInfoEnabled()) {
            getLog().info(sm.getString("abstractProtocolHandler.init", getName()));
        }

        if (oname == null) {
            // Component not pre-registered so register it
            oname = createObjectName();
            if (oname != null) {
                Registry.getRegistry(null, null).registerComponent(this, oname, null);  //本身加入jmx
            }
        }

        if (this.domain != null) {
            try {
                tpOname = new ObjectName(domain + ":type=ThreadPool,name=" + getName());
                Registry.getRegistry(null, null).registerComponent(endpoint, tpOname, null);  //将endpoint加入 这个玩意是个重点
            } catch (Exception e) {
                getLog().error(sm.getString( "abstractProtocolHandler.mbeanRegistrationFailed",
                        tpOname, getName()), e);
            }
            rgOname = new ObjectName(domain + ":type=GlobalRequestProcessor,name=" + getName());
            Registry.getRegistry(null, null).registerComponent(
                    getHandler().getGlobal(), rgOname, null);    //ConnectorHandler.global 加入jmx

            for (SSLHostConfig sslHostConfig : getEndpoint().findSslHostConfigs()) {   //默认此处应该没有数据
                ObjectName sslOname = new ObjectName(domain + ":type=SSLHostConfig,ThreadPool=" +
                        getName() + ",name=" + ObjectName.quote(sslHostConfig.getHostName()));
                Registry.getRegistry(null, null).registerComponent(sslHostConfig, sslOname, null);
                sslOnames.add(sslOname);
                for (SSLHostConfigCertificate sslHostConfigCert : sslHostConfig.getCertificates()) {
                    ObjectName sslCertOname = new ObjectName(domain +
                            ":type=SSLHostConfigCertificate,ThreadPool=" + getName() +
                            ",Host=" + ObjectName.quote(sslHostConfig.getHostName()) +
                            ",name=" + sslHostConfigCert.getType());
                    Registry.getRegistry(null, null).registerComponent(
                            sslHostConfigCert, sslCertOname, null);
                    sslCertOnames.add(sslCertOname);
                }
            }
        }

        String endpointName = getName();
        endpoint.setName(endpointName.substring(1, endpointName.length()-1));

        endpoint.init(); //endpoint的初始化
    }
  //AbstractEndpoint
    public final void init() throws Exception {
        if (bindOnInit) {
            bind();
            bindState = BindState.BOUND_ON_INIT;
        }
    }
  //这里我看的http11 所以是NioEndpoint
  public void bind() throws Exception {
        initServerSocket();  //初始化socket,使用的nio的api,要看一下

        // Initialize thread count defaults for acceptor, poller
        if (acceptorThreadCount == 0) {
            // FIXME: Doesn't seem to work that well with multiple accept threads
            acceptorThreadCount = 1;
        }
        if (pollerThreadCount <= 0) {
            //minimum one poller thread
            pollerThreadCount = 1;
        }
        setStopLatch(new CountDownLatch(pollerThreadCount)); //coutDownLatch.wait会计算内部cout,若>0则该线程继续await

        // Initialize SSL if needed
        initialiseSsl();   //初始化ssl

        selectorPool.open();  //nioApi selector 关于复用器
    }
      protected void initServerSocket() throws Exception {
        serverSock = ServerSocketChannel.open();  //打开nioServerSocket
        socketProperties.setProperties(serverSock.socket());
        InetSocketAddress addr = (getAddress()!=null?new InetSocketAddress(getAddress(),getPort()):new InetSocketAddress(getPort()));  //地址
        serverSock.socket().bind(addr,getAcceptCount()); //绑定
        serverSock.configureBlocking(true); //mimic APR behavior  //并且tomcat中这个serverSock默认blocking
    }
    
    //NioSelectorPool.java  selectorPool(NioEndPoint的成员)
   public void open() throws IOException {
        enabled = true;
        getSharedSelector();
        if (SHARED) {
            blockingSelector = new NioBlockingSelector();
            blockingSelector.open(getSharedSelector());
        }
    }
    //
     protected Selector getSharedSelector() throws IOException {
        if (SHARED && SHARED_SELECTOR == null) {
            synchronized ( NioSelectorPool.class ) {
                if ( SHARED_SELECTOR == null )  {
                    SHARED_SELECTOR = Selector.open(); //创建一个选择器
                    log.info("Using a shared selector for servlet write/read");
                }
            }
        }
        return  SHARED_SELECTOR;
    }
    //NioBlockingSelect.java 无继承的类
     public void open(Selector selector) {
        sharedSelector = selector;  //设置seletcor
        poller = new BlockPoller(); //这个类关键,为该类静态内部类
        poller.selector = sharedSelector;
        poller.setDaemon(true);
        poller.setName("NioBlockingSelector.BlockPoller-"+(++threadCounter));
        poller.start();
    }
{%endcodeblock%}
{%codeblock lang:java BlockPoller%}
//该方法时BlockPoller外部类BioBlockingSelector,只有阻塞情况使用以下逻辑
//该函数是连接器真正向外写数据的代码
 public int write(ByteBuffer buf, NioChannel socket, long writeTimeout)  //阻塞方式写出逻辑
            throws IOException {
        SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());  //获取此处这个channel对应的key
        if ( key == null ) throw new IOException("Key no longer registered");
        KeyReference reference = keyReferenceStack.pop();
        if (reference == null) {
            reference = new KeyReference();
        }
        NioSocketWrapper att = (NioSocketWrapper) key.attachment();
        int written = 0;
        boolean timedout = false;
        int keycount = 1; //assume we can write
        long time = System.currentTimeMillis(); //start the timeout timer
        try {
            while ( (!timedout) && buf.hasRemaining()) {
                if (keycount > 0) { //only write if we were registered for a write
                    int cnt = socket.write(buf); //write the data   我记得这个socket默认是非阻塞写出
                    if (cnt == -1)
                        throw new EOFException();
                    written += cnt;
                    if (cnt > 0) { //有数据写出则继续写
                        time = System.currentTimeMillis(); //reset our timeout timer
                        continue; //we successfully wrote, try again without a selector
                    }
                }
                try {
                    if ( att.getWriteLatch()==null || att.getWriteLatch().getCount()==0) att.startWriteLatch(1); //设置countDown
                    poller.add(att,SelectionKey.OP_WRITE,reference); //加入写事件,创建一个addEvent,addEvent线程逻辑就是注册写操作到selector中
                    if (writeTimeout < 0) {
                        att.awaitWriteLatch(Long.MAX_VALUE,TimeUnit.MILLISECONDS);  //countDown停留到Long.MAX_VALUE毫秒,等待就是BlockPoller中的线程结束
                    } else {
                        att.awaitWriteLatch(writeTimeout,TimeUnit.MILLISECONDS);//停留最大writeTimeout时间
                    }
                } catch (InterruptedException ignore) {
                    // Ignore
                }
                if ( att.getWriteLatch()!=null && att.getWriteLatch().getCount()> 0) {
                    //we got interrupted, but we haven't received notification from the poller.
                    keycount = 0;
                }else {
                    //latch countdown has happened
                    keycount = 1;
                    att.resetWriteLatch();
                }

                if (writeTimeout > 0 && (keycount == 0))
                    timedout = (System.currentTimeMillis() - time) >= writeTimeout;
            } //while
            if (timedout)
                throw new SocketTimeoutException();
        } finally {
            poller.remove(att,SelectionKey.OP_WRITE);
            if (timedout && reference.key!=null) {
                poller.cancelKey(reference.key);
            }
            reference.key = null;
            keyReferenceStack.push(reference);
        }
        return written;
    }
//读取逻辑    
//代码逻辑和写相同
    public int read(ByteBuffer buf, NioChannel socket, long readTimeout) throws IOException {
        SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
        if ( key == null ) throw new IOException("Key no longer registered");
        KeyReference reference = keyReferenceStack.pop();
        if (reference == null) {
            reference = new KeyReference();
        }
        NioSocketWrapper att = (NioSocketWrapper) key.attachment();
        int read = 0;
        boolean timedout = false;
        int keycount = 1; //assume we can read
        long time = System.currentTimeMillis(); //start the timeout timer
        try {
            while(!timedout) {
                if (keycount > 0) { //only read if we were registered for a read
                    read = socket.read(buf);
                    if (read != 0) {
                        break;
                    }
                }
                try {
                    if ( att.getReadLatch()==null || att.getReadLatch().getCount()==0) att.startReadLatch(1);
                    poller.add(att,SelectionKey.OP_READ, reference);
                    if (readTimeout < 0) {
                        att.awaitReadLatch(Long.MAX_VALUE, TimeUnit.MILLISECONDS);
                    } else {
                        att.awaitReadLatch(readTimeout, TimeUnit.MILLISECONDS);
                    }
                } catch (InterruptedException ignore) {
                    // Ignore
                }
                if ( att.getReadLatch()!=null && att.getReadLatch().getCount()> 0) {
                    //we got interrupted, but we haven't received notification from the poller.
                    keycount = 0;
                }else {
                    //latch countdown has happened
                    keycount = 1;
                    att.resetReadLatch();
                }
                if (readTimeout >= 0 && (keycount == 0))
                    timedout = (System.currentTimeMillis() - time) >= readTimeout;
            } //while
            if (timedout)
                throw new SocketTimeoutException();
        } finally {
            poller.remove(att,SelectionKey.OP_READ);
            if (timedout && reference.key!=null) {
                poller.cancelKey(reference.key);
            }
            reference.key = null;
            keyReferenceStack.push(reference);
        }
        return read;
    }
//该类是NioBlockingSelecotr内部,NioSelectorPool持有NioBlockingSelecotr对象


//实际上此类的作用用来处理当进行读写操作时,选择器对读写channel的管理NioBlockingSelecotr这套逻辑,可以选择不使用,而使用
//NioSelectorPool的读写逻辑,就没有selector
//逻辑就是当读写channel进行一次操作后加入selector中,等待下一次读写,默认情况是通过
 protected static class BlockPoller extends Thread {
   @Override
        public void run() {
            while (run) {
                try {
                    events();  //每次循环将所有有的事件取出并执行,事件轮询
                    int keyCount = 0;
                    try {
                        int i = wakeupCounter.get();
                        if (i>0)  //根据这个原子性int值,决定立即select还是超时1s
                            keyCount = selector.selectNow();
                        else {
                            wakeupCounter.set(-1);
                            keyCount = selector.select(1000);
                        }
                        wakeupCounter.set(0);  //当select返回counter设为0
                        if (!run) break;
                    }catch ( NullPointerException x ) {
                        //sun bug 5076772 on windows JDK 1.5
                        if (selector==null) throw x;
                        if ( log.isDebugEnabled() ) log.debug("Possibly encountered sun bug 5076772 on windows JDK 1.5",x);
                        continue;
                    } catch ( CancelledKeyException x ) {
                        //sun bug 5076772 on windows JDK 1.5
                        if ( log.isDebugEnabled() ) log.debug("Possibly encountered sun bug 5076772 on windows JDK 1.5",x);
                        continue;
                    } catch (Throwable x) {
                        ExceptionUtils.handleThrowable(x);
                        log.error("",x);
                        continue;
                    }

                    Iterator<SelectionKey> iterator = keyCount > 0 ? selector.selectedKeys().iterator() : null;

                    // Walk through the collection of ready keys and dispatch
                    // any active event.
                    while (run && iterator != null && iterator.hasNext()) {
                        SelectionKey sk = iterator.next();
                        NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();  //返回与该key附加的对象
                        try {
                            iterator.remove();
                            sk.interestOps(sk.interestOps() & (~sk.readyOps()));
                            if ( sk.isReadable() ) {
                                countDown(attachment.getReadLatch());  //这里就是一个countdown逻辑,代码到了这里说明此key是readkey,减少count,上边的read代码的await可以执行,继续进行read逻辑
                            }
                            if (sk.isWritable()) {
                                countDown(attachment.getWriteLatch());
                            }
                        }catch (CancelledKeyException ckx) {
                            sk.cancel();
                            countDown(attachment.getReadLatch());
                            countDown(attachment.getWriteLatch());
                        }
                    }//while
                }catch ( Throwable t ) {
                    log.error("",t);
                }
            }
            events.clear();
            try {
                selector.selectNow();//cancel all remaining keys
            }catch( Exception ignore ) {
                if (log.isDebugEnabled())log.debug("",ignore);
            }
            try {
                selector.close();//Close the connector
            }catch( Exception ignore ) {
                if (log.isDebugEnabled())log.debug("",ignore);
            }
        }

        public void countDown(CountDownLatch latch) {
            if ( latch == null ) return;
            latch.countDown();
        }
        //下边三个类都是poller的私有子类,为了完成注册,取消
  private class RunnableAdd implements Runnable {  

            private final SocketChannel ch;
            private final NioSocketWrapper key;
            private final int ops;
            private final KeyReference ref;

            public RunnableAdd(SocketChannel ch, NioSocketWrapper key, int ops, KeyReference ref) {
                this.ch = ch;
                this.key = key;
                this.ops = ops;
                this.ref = ref;
            }

            @Override
            public void run() {
                SelectionKey sk = ch.keyFor(selector);
                try {
                    if (sk == null) {
                        sk = ch.register(selector, ops, key);  //附加key就是此处加上的
                        ref.key = sk;
                    } else if (!sk.isValid()) {
                        cancel(sk, key, ops);
                    } else {
                        sk.interestOps(sk.interestOps() | ops);
                    }
                } catch (CancelledKeyException cx) {
                    cancel(sk, key, ops);
                } catch (ClosedChannelException cx) {
                    cancel(sk, key, ops);
                }
            }
        }


        private class RunnableRemove implements Runnable {

            private final SocketChannel ch;
            private final NioSocketWrapper key;
            private final int ops;

            public RunnableRemove(SocketChannel ch, NioSocketWrapper key, int ops) {
                this.ch = ch;
                this.key = key;
                this.ops = ops;
            }

            @Override
            public void run() {
                SelectionKey sk = ch.keyFor(selector);
                try {
                    if (sk == null) {
                        if (SelectionKey.OP_WRITE==(ops&SelectionKey.OP_WRITE)) countDown(key.getWriteLatch());
                        if (SelectionKey.OP_READ==(ops&SelectionKey.OP_READ))countDown(key.getReadLatch());
                    } else {
                        if (sk.isValid()) {
                            sk.interestOps(sk.interestOps() & (~ops));
                            if (SelectionKey.OP_WRITE==(ops&SelectionKey.OP_WRITE)) countDown(key.getWriteLatch());
                            if (SelectionKey.OP_READ==(ops&SelectionKey.OP_READ))countDown(key.getReadLatch());
                            if (sk.interestOps()==0) {
                                sk.cancel();
                                sk.attach(null);
                            }
                        }else {
                            sk.cancel();
                            sk.attach(null);
                        }
                    }
                }catch (CancelledKeyException cx) {
                    if (sk!=null) {
                        sk.cancel();
                        sk.attach(null);
                    }
                }
            }

        }


        public static class RunnableCancel implements Runnable {

            private final SelectionKey key;

            public RunnableCancel(SelectionKey key) {
                this.key = key;
            }

            @Override
            public void run() {
                key.cancel();
            }
        }
    }
 }   
{%endcodeblock%}
 - start阶段
 {%codeblock lang:java%}
 //AbstarctProtocol,connector在start_pre状态下没有监听器
  public void start() throws Exception {
        if (getLog().isInfoEnabled()) {
            getLog().info(sm.getString("abstractProtocolHandler.start", getName()));
        }

        endpoint.start(); //真正的重点

        // Start async timeout thread
        asyncTimeout = new AsyncTimeout();
        Thread timeoutThread = new Thread(asyncTimeout, getNameInternal() + "-AsyncTimeout");
        int priority = endpoint.getThreadPriority();
        if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
            priority = Thread.NORM_PRIORITY;
        }
        timeoutThread.setPriority(priority);
        timeoutThread.setDaemon(true);
        timeoutThread.start();
    }
 //NioEndPoint 
     public void startInternal() throws Exception {

        if (!running) {
            running = true;
            paused = false;
//           SynchronizedStack 是tomcat实现的简单stack,动态数组,操作都是同步函数
            processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,  //默认值都是128
                    socketProperties.getProcessorCache());  //处理器最大500
            eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                            socketProperties.getEventCache()); //事件最大500
            nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getBufferPool()); //通道最大500

            // Create worker collection
            if ( getExecutor() == null ) {
                createExecutor();
            }

            initializeConnectionLatch();

            // Start poller threads
            pollers = new Poller[getPollerThreadCount()]; //创建并启动轮询线程,代码放在机制部分,太长了
            for (int i=0; i<pollers.length; i++) {
                pollers[i] = new Poller();
                Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
                pollerThread.setPriority(threadPriority);
                pollerThread.setDaemon(true);
                pollerThread.start();
            }

            startAcceptorThreads();//启动接受器线程
        }
    }
    //创建线程池对象ThreadPoolExecutor这也是tomcat实现的
    //如果按照一般线程池来理解,那么这个结构就是不停执行的线程从任务队列中不断取task的过程
    public void createExecutor() {
        internalExecutor = true;
        TaskQueue taskqueue = new TaskQueue();
        TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
        executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
        taskqueue.setParent( (ThreadPoolExecutor) executor);
    }
 {%endcodeblock%}
##### 7.Context 
Context的标准实现为StandardContext, 代表着一个web项目,在HostConfig监听器创建
#### 各部分作用
1. LifecycleMBeanBase

{% codeblock lang:java initInternal%}
protected void initInternal() throws LifecycleException {
//该函数的作用就是将当前对象加入到jmx中
//并且该类仅仅实现了init阶段
        // If oname is not null then registration has already happened via
        // preRegister().
        if (oname == null) {
            mserver = Registry.getRegistry(null, null).getMBeanServer();

            oname = register(this, getObjectNameKeyProperties());
        }
    }
{% endcodeblock %}
#### 监听器
LifecycleListener子类,在life组件各个生命周期根据事件触发
##### VersionLoggerListener
 ```java
 //打印日志,在Server.init触发 条件
  @Override
    public void lifecycleEvent(LifecycleEvent event) {
   BEFORE_INIT_EVENT
        if (Lifecycle.BEFORE_INIT_EVENT.equals(event.getType())) {
            log();
        }
    }
 ```
##### AprLifecycleListener   
{% codeblock lang:java apr%}
//触发时间: 由Server.init触发,当正确配置,并且本地有jni库的情况,就会加载apr模式,并且后边会创建apr io
  @Override
    public void lifecycleEvent(LifecycleEvent event) { Lifecycle.BEFORE_INIT_EVENT
        if (Lifecycle.BEFORE_INIT_EVENT.equals(event.getType())) {
            synchronized (lock) {
                init(); //加载tcn本地库 当不能加载时 aprAvailable保持false
                for (String msg : initInfoLogMessages) {
                    log.info(msg);
                }
                initInfoLogMessages.clear();
                if (aprAvailable) {  //只有init中加载了库,才会导致此处有效,也影响了protocolHandler的创建
                    try {
                        initializeSSL();
                    } catch (Throwable t) {
                        t = ExceptionUtils.unwrapInvocationTargetException(t);
                        ExceptionUtils.handleThrowable(t);
                        log.error(sm.getString("aprListener.sslInit"), t);
                    }
                }
                // Failure to initialize FIPS mode is fatal
                if (!(null == FIPSMode || "off".equalsIgnoreCase(FIPSMode)) && !isFIPSModeActive()) {
                    Error e = new Error(
                            sm.getString("aprListener.initializeFIPSFailed"));
                    // Log here, because thrown error might be not logged
                    log.fatal(e.getMessage(), e);
                    throw e;
                }
            }
        } else if (Lifecycle.AFTER_DESTROY_EVENT.equals(event.getType())) {  //destroy 时触发
            synchronized (lock) {
                if (!aprAvailable) {
                    return;   //一般不是apr模式,就没有事情发生
                }
                try {
                    terminateAPR();  //这里是Library的native函数调用,也就是说library貌似就是控制tcn native的
                } catch (Throwable t) {
                    t = ExceptionUtils.unwrapInvocationTargetException(t);
                    ExceptionUtils.handleThrowable(t);
                    log.info(sm.getString("aprListener.aprDestroy"));
                }
            }
        }
      //这里 init  和terminateAPR 都是调用Library这个类的
{% endcodeblock %}
##### JreMemoryLeakPreventionListener
{%codeblock lang:java %}
//该监听器是用来防止某些jre类的加载导致对应类加载器无法被回收的情况
//首先tomcat的类加载器结构为 comm  server shaded 以及webAppclassLoader
//1.内存泄漏的原因:  由于某个对象引用链不断开,这里说的就是webAppClassLoader ||由于锁文件的导致,具体发生在urlConnection的缓冲机制
//2.在tomcat加载某些类 如:DriverManager 会由于该对象加载之后在java程序结束前都是以单例存在,而任意对象加载后会引用其类加载器对象,这就导致了加载器webAppClassLoader无法被回收
// 如:Disposer类被加载后会启动一个循环线程,也将导致webAppClassLoader无法被回收
//因此tomcat的解决方式是,在使用webAppClassLoader前使用系统类加载器将这些类加载一次,后边使用就不会发生这些事情,使用的时机就是Server.init触发的监听器
  //这里激活函数我没有全部截取
public void lifecycleEvent(LifecycleEvent event) {
        // Initialise these classes when Tomcat starts
        if (Lifecycle.BEFORE_INIT_EVENT.equals(event.getType())) {

            ClassLoader loader = Thread.currentThread().getContextClassLoader(); 

            try
            {
                // Use the system classloader as the victim for all this
                // ClassLoader pinning we're about to do.
                Thread.currentThread().setContextClassLoader(
                        ClassLoader.getSystemClassLoader());   //设置系统类加载器为上下文加载器

                /*
                 * First call to this loads all drivers in the current class
                 * loader
                 */
                if (driverManagerProtection) {
                    DriverManager.getDrivers(); //加载DriverManager
                }

                // Trigger the creation of the AWT (AWT-Windows, AWT-XAWT,
                // etc.) thread.
                // Note this issue is fixed in Java 8 update 05 onwards.
                if (awtThreadProtection && !JreCompat.isJre9Available()) {
                    java.awt.Toolkit.getDefaultToolkit();
                }

                /*
                 * Several components end up calling
                 * sun.misc.GC.requestLatency(long) which creates a daemon
                 * thread without setting the TCCL.
                 *
                 * Those libraries / components known to trigger memory leaks
                 * due to eventual calls to requestLatency(long) are:
                 * - javax.management.remote.rmi.RMIConnectorServer.start()
                 *
                 * Note: Long.MAX_VALUE is a special case that causes the thread
                 *       to terminate
                 *
                 * Fixed in Java 9 onwards (from early access build 130)
                 */
                if (gcDaemonProtection && !JreCompat.isJre9Available()) {
                    try {
                        Class<?> clazz = Class.forName("sun.misc.GC");
                        Method method = clazz.getDeclaredMethod(
                                "requestLatency",
                                new Class[] {long.class});
                        method.invoke(null, Long.valueOf(Long.MAX_VALUE - 1));
                    } catch (ClassNotFoundException e) {
                        if (JreVendor.IS_ORACLE_JVM) {
                            log.error(sm.getString(
                                    "jreLeakListener.gcDaemonFail"), e);
                        } else {
                            log.debug(sm.getString(
                                    "jreLeakListener.gcDaemonFail"), e);
                        }
                    } catch (SecurityException | NoSuchMethodException | IllegalArgumentException |
                            IllegalAccessException e) {
                        log.error(sm.getString("jreLeakListener.gcDaemonFail"),
                                e);
                    } catch (InvocationTargetException e) {
                        ExceptionUtils.handleThrowable(e.getCause());
                        log.error(sm.getString("jreLeakListener.gcDaemonFail"),
                                e);
                    }
					
                }
				  if (tokenPollerProtection && !JreCompat.isJre9Available()) {
                    java.security.Security.getProviders();
                }

                /*
                 * Several components end up opening JarURLConnections without
                 * first disabling caching. This effectively locks the file.
                 * Whilst more noticeable and harder to ignore on Windows, it
                 * affects all operating systems.
                 *
                 * Those libraries/components known to trigger this issue
                 * include:
                 * - log4j versions 1.2.15 and earlier
                 * - javax.xml.bind.JAXBContext.newInstance()
                 *
                 * https://bugs.openjdk.java.net/browse/JDK-8163449
                 */

                // Set the default URL caching policy to not to cache
                if (urlCacheProtection) {
                    try {
                        // Doesn't matter that this JAR doesn't exist - just as
                        // long as the URL is well-formed
                        URL url = new URL("jar:file://dummy.jar!/");
                        URLConnection uConn = url.openConnection();
                        uConn.setDefaultUseCaches(false); //取消缓冲机制
                    } catch (MalformedURLException e) {
                        log.error(sm.getString(
                                "jreLeakListener.jarUrlConnCacheFail"), e);
                    } catch (IOException e) {
                        log.error(sm.getString(
                                "jreLeakListener.jarUrlConnCacheFail"), e);
                    }
                }

{%endcodeblock%}
##### GlobalResourcesLifecycleListener
{%codeblock lang:java %}
//触发时机 start 和stop
public void lifecycleEvent(LifecycleEvent event) {

        if (Lifecycle.START_EVENT.equals(event.getType())) {
            component = event.getLifecycle();
            createMBeans();
        } else if (Lifecycle.STOP_EVENT.equals(event.getType())) {
            destroyMBeans();
            component = null;
        }
    }
    //
      protected void createMBeans() {
        // Look up our global naming context
        Context context = null;
        try {
            context = (Context) (new InitialContext()).lookup("java:/");
        } catch (NamingException e) {
            log.error("No global naming context defined for server");
            return;
        }

        // Recurse through the defined global JNDI resources context
        try {
            createMBeans("", context);
        } catch (NamingException e) {
            log.error("Exception processing Global JNDI Resources", e);
        }
    }
{%endcodeblock%}
##### ThreadLocalLeakPreventionListener
{%codeblock lang:java ThreadLocalLeakPreventionListener%}
//该监听器作用是处理由ThreadLocal 引起的内存泄漏问题
//原因: 当连接器获取req后,会创建执行器Process线程,参照EndPoint源码,此处如果是通过线程池创建的,那么当该线程一直继续进行后续处理,获取到web Context后,如果此线程中使用了ThreadLocal<AA>,当该context重新创建后,会通过重新创建
//一个 webAppClassLoader 来加载,而由于线程池的实现是将线程对象不断缓冲的模式,因此之前的Threadlocal引用,导致了之前webApploader不能释放
public class ThreadLocalLeakPreventionListener implements LifecycleListener,
        ContainerListener { //触发清理时机在任意context的销毁阶段
		 public void lifecycleEvent(LifecycleEvent event) { //该监听器开始位于Server中
        try {
            Lifecycle lifecycle = event.getLifecycle();
            if (Lifecycle.AFTER_START_EVENT.equals(event.getType()) &&
                    lifecycle instanceof Server) {
                // when the server starts, we register ourself as listener for
                // all context
                // as well as container event listener so that we know when new
                // Context are deployed
                Server server = (Server) lifecycle;
                registerListenersForServer(server);  //当server.start后,说明context已经部署,一层层的加入监听器
            }

            if (Lifecycle.BEFORE_STOP_EVENT.equals(event.getType()) &&
                    lifecycle instanceof Server) {
                // Server is shutting down, so thread pools will be shut down so
                // there is no need to clean the threads
                serverStopping = true;
            }

            if (Lifecycle.AFTER_STOP_EVENT.equals(event.getType()) &&
                    lifecycle instanceof Context) {
                stopIdleThreads((Context) lifecycle); //当context.stop后进行处理
            }
        } catch (Exception e) {
            String msg =
                sm.getString(
                    "threadLocalLeakPreventionListener.lifecycleEvent.error",
                    event);
            log.error(msg, e);
        }
    }

    @Override
    public void containerEvent(ContainerEvent event) { //当添加子容器,或移除,进行监听器注册/去除
        try {
            String type = event.getType();
            if (Container.ADD_CHILD_EVENT.equals(type)) {
                processContainerAddChild(event.getContainer(),
                    (Container) event.getData());
            } else if (Container.REMOVE_CHILD_EVENT.equals(type)) {
                processContainerRemoveChild(event.getContainer(),
                    (Container) event.getData());
            }
        } catch (Exception e) {
            String msg =
                sm.getString(
                    "threadLocalLeakPreventionListener.containerEvent.error",
                    event);
            log.error(msg, e);
        }

    }

    private void registerListenersForServer(Server server) { //逐层注册
        for (Service service : server.findServices()) {
            Engine engine = service.getContainer();
            engine.addContainerListener(this);
            registerListenersForEngine(engine);
        }
    }

    private void registerListenersForEngine(Engine engine) {
        for (Container hostContainer : engine.findChildren()) {
            Host host = (Host) hostContainer;
            host.addContainerListener(this);
            registerListenersForHost(host);
        }
    }

    private void registerListenersForHost(Host host) {
        for (Container contextContainer : host.findChildren()) {
            Context context = (Context) contextContainer;
            registerContextListener(context);
        }
    }

    private void registerContextListener(Context context) {
        context.addLifecycleListener(this);
    }

    protected void processContainerAddChild(Container parent, Container child) {
        if (log.isDebugEnabled())
            log.debug("Process addChild[parent=" + parent + ",child=" + child +
                "]");

        if (child instanceof Context) {
            registerContextListener((Context) child);
        } else if (child instanceof Engine) {
            registerListenersForEngine((Engine) child);
        } else if (child instanceof Host) {
            registerListenersForHost((Host) child);
        }

    }

    protected void processContainerRemoveChild(Container parent,
        Container child) {

        if (log.isDebugEnabled())
            log.debug("Process removeChild[parent=" + parent + ",child=" +
                child + "]");

        if (child instanceof Context) {
            Context context = (Context) child;
            context.removeLifecycleListener(this);
        } else if (child instanceof Host || child instanceof Engine) {
            child.removeContainerListener(this);
        }
    }

    /**
     * Updates each ThreadPoolExecutor with the current time, which is the time
     * when a context is being stopped.
     *
     * @param context
     *            the context being stopped, used to discover all the Connectors
     *            of its parent Service.
     */
    private void stopIdleThreads(Context context) {
        if (serverStopping) return;

        if (!(context instanceof StandardContext) ||
            !((StandardContext) context).getRenewThreadsWhenStoppingContext()) {
            log.debug("Not renewing threads when the context is stopping. "
                + "It is not configured to do it.");
            return;
        }

        Engine engine = (Engine) context.getParent().getParent();
        Service service = engine.getService();
        Connector[] connectors = service.findConnectors();
        if (connectors != null) {
            for (Connector connector : connectors) {
                ProtocolHandler handler = connector.getProtocolHandler();
                Executor executor = null;
                if (handler != null) {
                    executor = handler.getExecutor();  //这个线程池实际上就是Endpoint中的线程池,该线程池用来创建执行器线程来处理请求
                }

                if (executor instanceof ThreadPoolExecutor) {
                    ThreadPoolExecutor threadPoolExecutor =
                        (ThreadPoolExecutor) executor;
                    threadPoolExecutor.contextStopping(); //处理线程池,处理的方式就是阻塞线程池,并且停止线程池中线程,就能使的引用链断开
                } else if (executor instanceof StandardThreadExecutor) {
                    StandardThreadExecutor stdThreadExecutor =
                        (StandardThreadExecutor) executor;
                    stdThreadExecutor.contextStopping();
                }

            }
        }
    }
}
{%endcodeblock%}
##### NamingContextListener
{%codeblock lang:java NamingContextListener%}
//该监听器作用用于作为生命监听器和容器监听器
//1.作为生命监听器处理Server和Context中的naming资源,向jndi中注册,以及当生命组件destory时进行jndi解除,关于tomcat中jdni的实现部分在机制部分说明
public class NamingContextListener implements LifecycleListener, ContainerListener, PropertyChangeListener{
 public void lifecycleEvent(LifecycleEvent event)
    {

        container = event.getLifecycle();
        //当Server 或者 context 分别获取内部的namingResouces,以及token(用来标识使用的Object对象)
        if (container instanceof Context)
        {
            namingResources = ((Context) container).getNamingResources();
            token = ((Context) container).getNamingToken();
        }
        else if (container instanceof Server)
        {
            namingResources = ((Server) container).getGlobalNamingResources();
            token = ((Server) container).getNamingToken();
        }
        else
        {
            return;
        }

        if (Lifecycle.CONFIGURE_START_EVENT.equals(event.getType()))
        {

            if (initialized)
            {
                return;
            }

            try
            {
                Hashtable<String, Object> contextEnv = new Hashtable<>();
                namingContext = new NamingContext(contextEnv, getName()); //第一次创建上线文, 环境map为空,context对应的路径名为"/",就是表示根节点
                //ContextAccessController| ContextBindings这两个类中的变量都是map,并且都是static,这两个类是用来做标识给jndi系统使用的
                ContextAccessController.setSecurityToken(getName(), token); //使用验证map,
                ContextAccessController.setSecurityToken(container, token);
                //Hashtable<Object,Context> objectBindings(container,namingContext) ,token是用来检查SecurityToken的
                ContextBindings.bindContext(container, namingContext, token); //验证 container,token存在,put container,namingContext
                if (log.isDebugEnabled())
                {
                    log.debug("Bound " + container);
                }

                // Configure write when read-only behaviour
                namingContext.setExceptionOnFailedWrite(getExceptionOnFailedWrite());

                // Setting the context in read/write mode
                ContextAccessController.setWritable(getName(), token);

                try
                {
                    createNamingContext(); //进行jndi的初始化以及bind,目前看来加入的新的子context 或者ref都是存在于上层context中

                }
                catch (NamingException e)
                {
                    log.error(sm.getString("naming.namingContextCreationFailed", e));
                }

                namingResources.addPropertyChangeListener(this);  //添加改变监听器

                // Binding the naming context to the class loader
                if (container instanceof Context)
                {
                    // Setting the context in read only mode
                    ContextAccessController.setReadOnly(getName());
                    try
                    {
                        //验证container,token  put(ClassLoader,container->context) put(ClassLoader,container)
                        // 将类加载器webAppClassLoader和context以及对应tomcat context联系
                        ContextBindings.bindClassLoader(container, token, ((Context) container).getLoader().getClassLoader());  //完成不同context的隔离工作
                    }
                    catch (NamingException e)
                    {
                        log.error(sm.getString("naming.bindFailed", e));
                    }
                }

                if (container instanceof Server)
                {
                    org.apache.naming.factory.ResourceLinkFactory.setGlobalContext(namingContext); //ResourceLink设置全局context,应该是该tomcat context访问全局资源使用的
                    try
                    {
                        ContextBindings.bindClassLoader(container, token, this.getClass().getClassLoader());
                        //测试
                        // env map 使用该给NamingManager创建工厂使用的变量,tomcat在系统初始化的时候将:
                        // Context.INITIAL_CONTEXT_FACTORY=org.apache.naming.java.javaURLContextFactory
                        // Context.URL_PKG_PREFIXES=org.apache.naming 放到了system.props中,当这两个变量被InitialContext对象创建时调用init(),被NamingManager使用
                        // 将defaultFactory设置了
                        InitialContext initialContext = new InitialContext();
                        initialContext.lookup("java:UserDatabase");
                    }
                    catch (NamingException e)
                    {
                        log.error(sm.getString("naming.bindFailed", e));
                    }
                    if (container instanceof StandardServer)
                    {
                        ((StandardServer) container).setGlobalNamingContext(namingContext);
                    }
                }

            }
            finally
            {
                // Regardless of success, so that we can do cleanup on configure_stop
                initialized = true;
            }

        }
        else if (Lifecycle.CONFIGURE_STOP_EVENT.equals(event.getType()))  //卸载
        {

            if (!initialized)
            {
                return;
            }

            try
            {
                // Setting the context in read/write mode
                ContextAccessController.setWritable(getName(), token);
                ContextBindings.unbindContext(container, token);

                if (container instanceof Context)
                {
                    ContextBindings.unbindClassLoader(container, token, ((Context) container).getLoader().getClassLoader());
                }

                if (container instanceof Server)
                {
                    namingResources.removePropertyChangeListener(this);
                    ContextBindings.unbindClassLoader(container, token, this.getClass().getClassLoader());
                }

                ContextAccessController.unsetSecurityToken(getName(), token);
                ContextAccessController.unsetSecurityToken(container, token);

                // unregister mbeans.
                if (!objectNames.isEmpty())
                {
                    Collection<ObjectName> names = objectNames.values();
                    Registry registry = Registry.getRegistry(null, null);
                    for (ObjectName objectName : names)
                    {
                        registry.unregisterComponent(objectName);
                    }
                }

                javax.naming.Context global = getGlobalNamingContext();
                if (global != null)
                {
                    ResourceLinkFactory.deregisterGlobalResourceAccess(global);
                }
            }
            finally
            {
                objectNames.clear();

                namingContext = null;
                envCtx = null;
                compCtx = null;
                initialized = false;
            }

        }

    }
	
	
	/**
     * Create and initialize the JNDI naming context.
     * tomcat jdni 的过程:
     * 都在server.xml中配置,也就是作为 Server全局资源并且在任意context中配置ref进行使用|或者作为Context中局部资源
     * 1.创建nameRersouce
     *  Server中namingResource创建于构造函数, context namingResource创建参考
     * @see org.apache.catalina.startup.SetNextNamingRule#end(String, String) 导致在server.xml中配置的context内部namingResource的加载
     * 2.通过namingContextListener作为生命周期监听器调用该函数,将nameRersouce中资源进行注册
     *  Server的监听器在构造创建 ,context的在startInternal创建,并激活
     * --------
     * 至少在tomcat9中不存在在context.xml中配置任意web项目,具体源码参照Hostconfig解析xml过程
     * tomcat9支持在web.xml中配置局部资源,解析过程参见context的部分
     */
    private void createNamingContext()  //该函数是Server /Context 这两种全局和局部jndi进行bind的入口,该函数具体对不同资源进行注册,特殊的有ref类型
            throws NamingException
    {

        // Creating the comp subcontext
        if (container instanceof Server)
        {
            compCtx = namingContext;
            envCtx = namingContext;
        }
        else
        {
            compCtx = namingContext.createSubcontext("comp");
            envCtx = compCtx.createSubcontext("env");
        }

        int i;

        if (log.isDebugEnabled())
        {
            log.debug("Creating JNDI naming context");
        }

        if (namingResources == null)
        {
            namingResources = new NamingResourcesImpl();
            namingResources.setContainer(container);
        }

        // Resource links
        ContextResourceLink[] resourceLinks = namingResources.findResourceLinks();
        for (i = 0; i < resourceLinks.length; i++)
        {
            addResourceLink(resourceLinks[i]);
        }

        // Resources
        ContextResource[] resources = namingResources.findResources();
        for (i = 0; i < resources.length; i++)
        {
            addResource(resources[i]);
        }

        // Resources Env
        ContextResourceEnvRef[] resourceEnvRefs = namingResources.findResourceEnvRefs();
        for (i = 0; i < resourceEnvRefs.length; i++)
        {
            addResourceEnvRef(resourceEnvRefs[i]);
        }

        // Environment entries
        ContextEnvironment[] contextEnvironments = namingResources.findEnvironments();
        for (i = 0; i < contextEnvironments.length; i++)
        {
            addEnvironment(contextEnvironments[i]);
        }

        // EJB references
        ContextEjb[] ejbs = namingResources.findEjbs();
        for (i = 0; i < ejbs.length; i++)
        {
            addEjb(ejbs[i]);
        }

        // WebServices references
        ContextService[] services = namingResources.findServices();
        for (i = 0; i < services.length; i++)
        {
            addService(services[i]);
        }

        // Binding a User Transaction reference
        if (container instanceof Context)
        {
            try
            {
                Reference ref = new TransactionRef();
                compCtx.bind("UserTransaction", ref);
                ContextTransaction transaction = namingResources.getTransaction();
                if (transaction != null)
                {
                    Iterator<String> params = transaction.listProperties();
                    while (params.hasNext())
                    {
                        String paramName = params.next();
                        String paramValue = (String) transaction.getProperty(paramName);
                        StringRefAddr refAddr = new StringRefAddr(paramName, paramValue);
                        ref.add(refAddr);
                    }
                }
            }
            catch (NameAlreadyBoundException e)
            {
                // Ignore because UserTransaction was obviously
                // added via ResourceLink
            }
            catch (NamingException e)
            {
                log.error(sm.getString("naming.bindFailed", e));
            }
        }

        // Binding the resources directory context
        if (container instanceof Context)
        {
            try
            {
                compCtx.bind("Resources", ((Context) container).getResources());
            }
            catch (NamingException e)
            {
                log.error(sm.getString("naming.bindFailed", e));
            }
        }

    }
	
	//以添加ContextResource 为例:
	 public void addResource(ContextResource resource)
    {

        // Create a reference to the resource.  创建reference
        Reference ref = new ResourceRef(resource.getType(), resource.getDescription(), resource.getScope(), resource.getAuth(), resource.getSingleton());
        // Adding the additional parameters, if any 添加额外的内容,也就是存在于map中而不是直接成员变量的属性
        Iterator<String> params = resource.listProperties();
        while (params.hasNext())
        {
            String paramName = params.next();
            String paramValue = (String) resource.getProperty(paramName);
            StringRefAddr refAddr = new StringRefAddr(paramName, paramValue);
            ref.add(refAddr);
        }
        try
        {
            if (log.isDebugEnabled())
            {
                log.debug("  Adding resource ref " + resource.getName() + "  " + ref);
            }
            createSubcontexts(envCtx, resource.getName());  //创建子树上下文节点
            envCtx.bind(resource.getName(), ref);  //真正绑定资源对象到树结构上,至此完成资源在jdni中的绑定
        }
        catch (NamingException e)
        {
            log.error(sm.getString("naming.bindFailed", e));
        }

        if ("javax.sql.DataSource".equals(ref.getClassName()) && resource.getSingleton())
        {
            try
            {
                ObjectName on = createObjectName(resource);
                Object actualResource = envCtx.lookup(resource.getName());
                Registry.getRegistry(null, null).registerComponent(actualResource, on, null);
                objectNames.put(resource.getName(), on);
            }
            catch (Exception e)
            {
                log.warn(sm.getString("naming.jmxRegistrationFailed", e));
            }
        }

    }
	
{%endcodeblock%}
##### MapperListener
该监听器默认为Service成员,Service.startInternal() Service.initInternal()  都会触发
{%codeblock lang:java %}
//该监听器用于向mapper中田间context信息
//此监听器为 contrainerListener&&LifeListener&&LifeMbean 子类
//该监听器实体对象处于Service中,但是并不是Service的监听器,也属于生命组件和容器组件

    public MapperListener(Service service) {
        this.service = service;
        this.mapper = service.getMapper();  //mapper是service的属性
    }


 public void startInternal() throws LifecycleException {

        setState(LifecycleState.STARTING);

        Engine engine = service.getContainer();
        if (engine == null) {
            return;
        }

        findDefaultHost();

        addListeners(engine); //递归将该监听器作为lifeListern和contianerlistern加如engine以及其子类中

        Container[] conHosts = engine.findChildren();  //engine的直接子容器,实际就是host
        for (Container conHost : conHosts) {
            Host host = (Host) conHost;
            if (!LifecycleState.NEW.equals(host.getState())) {
                // Registering the host will register the context and wrappers
                registerHost(host); //注册host
            }
        }
    }
	
	  private void registerHost(Host host) {

        String[] aliases = host.findAliases();
        mapper.addHost(host.getName(), aliases, host);
        for (Container container : host.findChildren()) {
            if (container.getState().isAvailable()) {
                registerContext((Context) container);  //对于host的子容器都是context,进行context在mapper的注册
            }
        }
        if(log.isDebugEnabled()) {
            log.debug(sm.getString("mapperListener.registerHost",
                    host.getName(), domain, service));
        }
    }
	
	private void registerContext(Context context) {

        String contextPath = context.getPath();
        if ("/".equals(contextPath)) {
            contextPath = "";
        }
        Host host = (Host)context.getParent();

        WebResourceRoot resources = context.getResources();
        String[] welcomeFiles = context.findWelcomeFiles();
        List<WrapperMappingInfo> wrappers = new ArrayList<>();

        for (Container container : context.findChildren()) {
            prepareWrapperMappingInfo(context, (Wrapper) container, wrappers);

            if(log.isDebugEnabled()) {
                log.debug(sm.getString("mapperListener.registerWrapper",
                        container.getName(), contextPath, service));
            }
        }

        mapper.addContextVersion(host.getName(), host, contextPath,    //添加信息,在CoyotesAdaptor处理res和rep的过程,识别url和映射就是通过mapper做的
                context.getWebappVersion(), context, welcomeFiles, resources,
                wrappers);

        if(log.isDebugEnabled()) {
            log.debug(sm.getString("mapperListener.registerContext",
                    contextPath, service));
        }
    }
{%endcodeblock%}
##### Hostconfig 
{%codeblock lang:java HostConfig%}
//该监听器加入时机在xml解析host的时候,该监听器触发的时机是Host被start的时候
public void lifecycleEvent(LifecycleEvent event) {

        // Identify the host we are associated with
        try {
            host = (Host) event.getLifecycle();
            if (host instanceof StandardHost) {
                setCopyXML(((StandardHost) host).isCopyXML());
                setDeployXML(((StandardHost) host).isDeployXML());
                setUnpackWARs(((StandardHost) host).isUnpackWARs());
                setContextClass(((StandardHost) host).getContextClass());
            }
        } catch (ClassCastException e) {
            log.error(sm.getString("hostConfig.cce", event.getLifecycle()), e);
            return;
        }
        
        // Process the event that has occurred 在host不同情况下触发不同函数
        if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {  //这种事件发生暂时没见过
            check();
        } else if (event.getType().equals(Lifecycle.BEFORE_START_EVENT)) { //host.state=beforeStart
            beforeStart();
        } else if (event.getType().equals(Lifecycle.START_EVENT)) {  //触发的时机就是Host变成startiing状态,也就是其子类容器已经初始化过的情况
            start();
        } else if (event.getType().equals(Lifecycle.STOP_EVENT)) {
            stop();
        }
        
        public void beforeStart() { 
        if (host.getCreateDirs()) { //判断是否创建文件
            File[] dirs = new File[] {host.getAppBaseFile(),host.getConfigBaseFile()};  //实际就是webapp路径和localhost路径
            for (int i=0; i<dirs.length; i++) {
                if (!dirs[i].mkdirs() && !dirs[i].isDirectory()) {
                    log.error(sm.getString("hostConfig.createDirs",dirs[i]));
                }
            }
        }
        
            public void start() {
        if (log.isDebugEnabled())
            log.debug(sm.getString("hostConfig.start"));

        try {
            ObjectName hostON = host.getObjectName();
            oname = new ObjectName
                (hostON.getDomain() + ":type=Deployer,host=" + host.getName());
            Registry.getRegistry(null, null).registerComponent
                (this, oname, this.getClass().getName());
        } catch (Exception e) {
            log.error(sm.getString("hostConfig.jmx.register", oname), e);
        }

        if (!host.getAppBaseFile().isDirectory()) {
            log.error(sm.getString("hostConfig.appBase", host.getName(),
                    host.getAppBaseFile().getPath()));
            host.setDeployOnStartup(false);
            host.setAutoDeploy(false);
        }

        if (host.getDeployOnStartup())
            deployApps();  //部署app
    }
    }
        
           protected void deployApps() {

        File appBase = host.getAppBaseFile();
        File configBase = host.getConfigBaseFile();//表示 %CATALINA)HOMT%/conf/[EngineName](catalnia)/[HostName](localhost) 目录,如果没有配置则没用
        String[] filteredAppPaths = filterAppPaths(appBase.list());//表示webapps文件夹下一级目录
        // Deploy XML descriptors from configBase
        deployDescriptors(configBase, configBase.list());  //根据localhost有xml进行一部分配置
        // Deploy WARs
        deployWARs(appBase, filteredAppPaths);   //遍历webapp下文件,进行war的部署
        // Deploy expanded folders
        deployDirectories(appBase, filteredAppPaths); //根据文件夹进行部署

    }
      //上边三个判断逻辑都是这样的,复合条件则通过container中的startStopExe进行线程处理
      protected void deployDirectories(File appBase, String[] files) {

        if (files == null)
            return;

        ExecutorService es = host.getStartStopExecutor();
        List<Future<?>> results = new ArrayList<>();

        for (int i = 0; i < files.length; i++) {

            if (files[i].equalsIgnoreCase("META-INF"))
                continue;
            if (files[i].equalsIgnoreCase("WEB-INF"))
                continue;
            File dir = new File(appBase, files[i]);
            if (dir.isDirectory()) {
                ContextName cn = new ContextName(files[i], false);

                if (isServiced(cn.getName()) || deploymentExists(cn.getName()))
                    continue;

                results.add(es.submit(new DeployDirectory(this, cn, dir))); //有三个这样线程类,负责调用HostConfig中的不同部署方法
            }
        }

        for (Future<?> result : results) {
            try {
                result.get();
            } catch (Exception e) {
                log.error(sm.getString(
                        "hostConfig.deployDir.threaded.error"), e);
            }
        }
    }
    //处理线程
        private static class DeployDirectory implements Runnable {

        private HostConfig config;
        private ContextName cn;
        private File dir;

        public DeployDirectory(HostConfig config, ContextName cn, File dir) {
            this.config = config;
            this.cn = cn;
            this.dir = dir;
        }

        @Override
        public void run() {
            config.deployDirectory(cn, dir);  //其他两个实际上也都是调用了hostConfig中其他对应函数
        }
    }
    //其中部署非war的文件夹
    protected void deployDirectory(ContextName cn, File dir) {


        long startTime = 0;
        // Deploy the application in this directory
        if( log.isInfoEnabled() ) {
            startTime = System.currentTimeMillis();
            log.info(sm.getString("hostConfig.deployDir",
                    dir.getAbsolutePath()));
        }

        Context context = null;
        //META-INFO/content.xml文件表示web项目,也就是说可以通过此处的xml文件进行配置content,如果没有该xml tomcat自行配置
        //例如: F:\apache-tomcat-9.0.0.M26-src\webapps\demo\META-INF/context.xml  对demo项目进行配置
        File xml = new File(dir, Constants.ApplicationContextXml);
        File xmlCopy =
                new File(host.getConfigBaseFile(), cn.getBaseName() + ".xml"); //复制一份到catalina/localhost/目录下

        boolean b=xml.exists();
        DeployedApplication deployedApp;
        boolean copyThisXml = isCopyXML();
        boolean deployThisXML = isDeployThisXML(dir, cn);

        try {
            if (deployThisXML && xml.exists()) { //当content存在解析创建content
                synchronized (digesterLock) {
                    try {
                        context = (Context) digester.parse(xml); //这个digester较为简单,说明content的结构也是简单的
                    } catch (Exception e) {
                        log.error(sm.getString(
                                "hostConfig.deployDescriptor.error",
                                xml), e);
                        context = new FailedContext();
                    } finally {
                        digester.reset();
                        if (context == null) {
                            context = new FailedContext();
                        }
                    }
                }

                if (copyThisXml == false && context instanceof StandardContext) { //拷贝xml
                    // Host is using default value. Context may override it.
                    copyThisXml = ((StandardContext) context).getCopyXML();
                }

                if (copyThisXml) {
                    Files.copy(xml.toPath(), xmlCopy.toPath());
                    context.setConfigFile(xmlCopy.toURI().toURL());
                } else {
                    context.setConfigFile(xml.toURI().toURL());
                }
            } else if (!deployThisXML && xml.exists()) {
                // Block deployment as META-INF/context.xml may contain security
                // configuration necessary for a secure deployment.
                log.error(sm.getString("hostConfig.deployDescriptor.blocked",
                        cn.getPath(), xml, xmlCopy));
                context = new FailedContext();
            } else {
                context = (Context) Class.forName(contextClass).newInstance(); //不存在xml时创建org.apache.catalina.core.StandardContext作为contentxt
            }

            Class<?> clazz = Class.forName(host.getConfigClass()); //创建contentConfig监听器
            LifecycleListener listener =
                (LifecycleListener) clazz.newInstance();
            context.addLifecycleListener(listener);
            //此处表示这几个属性,即使在content.xml 配置了,此处还是会被改写成正确的,那么也就是说即使在content写了path doc都是没有用的
            context.setName(cn.getName());
            context.setWebappVersion(cn.getVersion());
//            if(!cn.getName().equalsIgnoreCase("/demo")){  //做个实验,如果此处更改,就能按照content.xml进行配置path 和docBase
                context.setPath(cn.getPath());
                context.setDocBase(cn.getBaseName());
//            }

            host.addChild(context);  //host持有对应的content,此处还启动了content.start,这部分应该很重要
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            log.error(sm.getString("hostConfig.deployDir.error",
                    dir.getAbsolutePath()), t);
        } finally {
            deployedApp = new DeployedApplication(cn.getName(),
                    xml.exists() && deployThisXML && copyThisXml); //创建DeployedApplication 表示部署应用

            // Fake re-deploy resource to detect if a WAR is added at a later
            // point
            deployedApp.redeployResources.put(dir.getAbsolutePath() + ".war",
                    Long.valueOf(0));
            deployedApp.redeployResources.put(dir.getAbsolutePath(),
                    Long.valueOf(dir.lastModified()));
            if (deployThisXML && xml.exists()) {
                if (copyThisXml) {
                    deployedApp.redeployResources.put(
                            xmlCopy.getAbsolutePath(),
                            Long.valueOf(xmlCopy.lastModified()));
                } else {
                    deployedApp.redeployResources.put(
                            xml.getAbsolutePath(),
                            Long.valueOf(xml.lastModified()));
                    // Fake re-deploy resource to detect if a context.xml file is
                    // added at a later point
                    deployedApp.redeployResources.put(
                            xmlCopy.getAbsolutePath(),
                            Long.valueOf(0));
                }
            } else {
                // Fake re-deploy resource to detect if a context.xml file is
                // added at a later point
                deployedApp.redeployResources.put(
                        xmlCopy.getAbsolutePath(),
                        Long.valueOf(0));
                if (!xml.exists()) {
                    deployedApp.redeployResources.put(
                            xml.getAbsolutePath(),
                            Long.valueOf(0));
                }
            }
            addWatchedResources(deployedApp, dir.getAbsolutePath(), context);  //监视这部分,当该部分改变时,重新加载content,此处源码我没有看
            // Add the global redeploy resources (which are never deleted) at
            // the end so they don't interfere with the deletion process
            addGlobalRedeployResources(deployedApp);
        }

        deployed.put(cn.getName(), deployedApp);  //将deployedApp加入到hostConfig的一个hashMap中

        if( log.isInfoEnabled() ) {
            log.info(sm.getString("hostConfig.deployDir.finished",
                    dir.getAbsolutePath(), Long.valueOf(System.currentTimeMillis() - startTime)));
        }
    }
	
	//关于HostConfig内部的解析器,仅仅创建content,以及设置器属性,并不包含子属性,对比servelt.xml中content的解析
	 protected static Digester createDigester(String contextClassName) {
        Digester digester = new Digester();
        digester.setValidating(false);
        // Add object creation rule
        digester.addObjectCreate("Context", contextClassName, "className");
        // Set the properties on that object (it doesn't matter if extra
        // properties are set)
        digester.addSetProperties("Context");
        return digester;
    }

}
    
    
{%endcodeblock%}
### 相关机制
####  life
此机制实际上是通过观察者模式实现,将属于lifecycle接口的子类分成不通的`生命周期`,来处理不同的监听器.
1.LifeBase子类属于事件源,state表示当前生命状态
2.lifeListen持有fire函数,根据不同event判断是否调用对应的处理函数
3.实际上tomcat中的事件类型就是通过字符串分别不过通过EventObject做了层封装,LifeState则是枚举类型
{%codeblock lang:java LifecycleEvent%}
public final class LifecycleEvent extends EventObject {
  //典型的java观察者模式中的EventObject
    private static final long serialVersionUID = 1L;

    public LifecycleEvent(Lifecycle lifecycle, String type, Object data) {
        super(lifecycle);
        this.type = type; //使用type来分别事件类型,这个数量和lifecycle中string数量相同
        this.data = data;
    }
    private final Object data;
    private final String type;
    public Object getData() {
        return data;
    }
    public Lifecycle getLifecycle() {
        return (Lifecycle) getSource();
    }
    public String getType() {
        return this.type;
    }
}
{%endcodeblock%}
{%codeblock lang:java LifecycleState%}
public enum LifecycleState {
//生命周期状态
    NEW(false, null),
    INITIALIZING(false, Lifecycle.BEFORE_INIT_EVENT),
    INITIALIZED(false, Lifecycle.AFTER_INIT_EVENT),
    STARTING_PREP(false, Lifecycle.BEFORE_START_EVENT),
    STARTING(true, Lifecycle.START_EVENT),      //只有调用start和start结束后组件有效,其他时期组件处于无效状态
    STARTED(true, Lifecycle.AFTER_START_EVENT),
    STOPPING_PREP(true, Lifecycle.BEFORE_STOP_EVENT),
    STOPPING(false, Lifecycle.STOP_EVENT),
    STOPPED(false, Lifecycle.AFTER_STOP_EVENT),
    DESTROYING(false, Lifecycle.BEFORE_DESTROY_EVENT),
    DESTROYED(false, Lifecycle.AFTER_DESTROY_EVENT),
    FAILED(false, null);

    private final boolean available;  //此处的状态在enum定义后是无法改变的,也就是说该变量代表了组件当前的状态,表示其是否有效
    private final String lifecycleEvent;

    private LifecycleState(boolean available, String lifecycleEvent) {
        this.available = available;
        this.lifecycleEvent = lifecycleEvent;
    }

    /**
     * May the public methods other than property getters/setters and lifecycle
     * methods be called for a component in this state? It returns
     * <code>true</code> for any component in any of the following states:
     * <ul>
     * <li>{@link #STARTING}</li>
     * <li>{@link #STARTED}</li>
     * <li>{@link #STOPPING_PREP}</li>
     * </ul>
     *
     * @return <code>true</code> if the component is available for use,
     *         otherwise <code>false</code>
     */
    public boolean isAvailable() {
        return available;
    }

    public String getLifecycleEvent() {
        return lifecycleEvent;
    }
}
{%endcodeblock%}
{%codeblock lang:java Lifecycle%}
 /* The valid state transitions for components that support {@link Lifecycle}
 * are:
 * <pre>
 *            start()
 *  -----------------------------
 *  |                           |
 *  | init()                    |
 * NEW -»-- INITIALIZING        |
 * | |           |              |     ------------------«-----------------------
 * | |           |auto          |     |                                        |
 * | |          \|/    start() \|/   \|/     auto          auto         stop() |
 * | |      INITIALIZED --»-- STARTING_PREP --»- STARTING --»- STARTED --»---  |
 * | |         |                                                            |  |
 * | |destroy()|                                                            |  |
 * | --»-----«--    ------------------------«--------------------------------  ^
 * |     |          |                                                          |
 * |     |         \|/          auto                 auto              start() |
 * |     |     STOPPING_PREP ----»---- STOPPING ------»----- STOPPED -----»-----
 * |    \|/                               ^                     |  ^
 * |     |               stop()           |                     |  |
 * |     |       --------------------------                     |  |
 * |     |       |                                              |  |
 * |     |       |    destroy()                       destroy() |  |
 * |     |    FAILED ----»------ DESTROYING ---«-----------------  |
 * |     |                        ^     |                          |
 * |     |     destroy()          |     |auto                      |
 * |     --------»-----------------    \|/                         |
 * |                                 DESTROYED                     |
 * |                                                               |
 * |                            stop()                             |
 * ----»-----------------------------»------------------------------
 *
 */
 //一下就是event的字符串标识
    * The LifecycleEvent type for the "component before init" event.
     */
    public static final String BEFORE_INIT_EVENT = "before_init";


    /**
     * The LifecycleEvent type for the "component after init" event.
     */
    public static final String AFTER_INIT_EVENT = "after_init";


    /**
     * The LifecycleEvent type for the "component start" event.
     */
    public static final String START_EVENT = "start";


    /**
     * The LifecycleEvent type for the "component before start" event.
     */
    public static final String BEFORE_START_EVENT = "before_start";


    /**
     * The LifecycleEvent type for the "component after start" event.
     */
    public static final String AFTER_START_EVENT = "after_start";


    /**
     * The LifecycleEvent type for the "component stop" event.
     */
    public static final String STOP_EVENT = "stop";


    /**
     * The LifecycleEvent type for the "component before stop" event.
     */
    public static final String BEFORE_STOP_EVENT = "before_stop";


    /**
     * The LifecycleEvent type for the "component after stop" event.
     */
    public static final String AFTER_STOP_EVENT = "after_stop";


    /**
     * The LifecycleEvent type for the "component after destroy" event.
     */
    public static final String AFTER_DESTROY_EVENT = "after_destroy";


    /**
     * The LifecycleEvent type for the "component before destroy" event.
     */
    public static final String BEFORE_DESTROY_EVENT = "before_destroy";


    /**
     * The LifecycleEvent type for the "periodic" event.
     */
    public static final String PERIODIC_EVENT = "periodic";


    /**
     * The LifecycleEvent type for the "configure_start" event. Used by those
     * components that use a separate component to perform configuration and
     * need to signal when configuration should be performed - usually after
     * {@link #BEFORE_START_EVENT} and before {@link #START_EVENT}.
     */
    public static final String CONFIGURE_START_EVENT = "configure_start";


    /**
     * The LifecycleEvent type for the "configure_stop" event. Used by those
     * components that use a separate component to perform configuration and
     * need to signal when de-configuration should be performed - usually after
     * {@link #STOP_EVENT} and before {@link #AFTER_STOP_EVENT}.
     */
    public static final String CONFIGURE_STOP_EVENT = "configure_stop";
{%endcodeblock%}
- 关于life组件未必一定要通过.init start()这样的调用顺序
{%codeblock lang:java lifecyleBae%}
  public final synchronized void start() throws LifecycleException {

        if (LifecycleState.STARTING_PREP.equals(state) || LifecycleState.STARTING.equals(state) ||
                LifecycleState.STARTED.equals(state)) {

            if (log.isDebugEnabled()) {
                Exception e = new LifecycleException();
                log.debug(sm.getString("lifecycleBase.alreadyStarted", toString()), e);
            } else if (log.isInfoEnabled()) {
                log.info(sm.getString("lifecycleBase.alreadyStarted", toString()));
            }

            return;
        }

        if (state.equals(LifecycleState.NEW)) {  //若通过start进入生命周期也会检查,并且可以执行一次init
            init();
        } else if (state.equals(LifecycleState.FAILED)) {
            stop();
        } else if (!state.equals(LifecycleState.INITIALIZED) &&
                !state.equals(LifecycleState.STOPPED)) {
            invalidTransition(Lifecycle.BEFORE_START_EVENT);
        }
    ...省略
    }
{%endcodeblock%}

{%codeblock lang:java life和对应监听器%}
//tomcat 中 lifeCycle的子类属于 生命周期组件,不同时刻表示出不同的状态,由LifecycleState 类来表示
//1.通过 init start stop destory 来改变其状态
//2.由抽象类lifeBase 中setInternalState 来改变当前状态,并且在不同状态对应着不同事件,由LifecycleEvent
public final class LifecycleEvent extends EventObject {
   //type 是lifeCycle中常量的子集,其数量和lifeState对应
    private static final long serialVersionUID = 1L;


    /**
     * Construct a new LifecycleEvent with the specified parameters.
     *
     * @param lifecycle Component on which this event occurred
     * @param type Event type (required)
     * @param data Event data (if any)
     */
    public LifecycleEvent(Lifecycle lifecycle, String type, Object data) {
        super(lifecycle);
        this.type = type; //type表示事件的类型,监听器是根据该类型进行调用逻辑的
        this.data = data;
    }


    /**
     * The event data associated with this event.
     */
    private final Object data;


    /**
     * The event type this instance represents.
     */
    private final String type;


    /**
     * @return the event data of this event.
     */
    public Object getData() {
        return data;
    }


    /**
     * @return the Lifecycle on which this event occurred.
     */
    public Lifecycle getLifecycle() {
        return (Lifecycle) getSource();
    }


    /**
     * @return the event type of this event.
     */
    public String getType() {
        return this.type;
    }
}
{%endcodeblock%}
#### Realm 域
关于realm顾名思义,暂时理解对于不同组件的分类
{% asset_img Realm.png 大致分类%}
{% asset_img realm1.png Realm的继承%}
如上图`CombinedRealm`具有list成员可以含有多个realm对象
{%codeblock lang:java RealmBase%}
//该类是Mben成员所以其共有 init过程已知,不存在共有start
//以下是该抽象的逻辑
@Override
    protected void initInternal() throws LifecycleException {

        super.initInternal(); //jmx

        // We want logger as soon as possible
        if (container != null) { //realm基本都是container所持有的
            this.containerLog = container.getLogger();//记录
        }

        x509UsernameRetriever = createUsernameRetriever(x509UsernameRetrieverClassName); //不清楚
    }

    /**
     * Prepare for the beginning of active use of the public methods of this
     * component and implement the requirements of
     * {@link org.apache.catalina.util.LifecycleBase#startInternal()}.
     *
     * @exception LifecycleException if this component detects a fatal error
     *  that prevents this component from being used
     */
    @Override
    protected void startInternal() throws LifecycleException {
        if (credentialHandler == null) {  //结束
            credentialHandler = new MessageDigestCredentialHandler();
        }

        setState(LifecycleState.STARTING);
    }
{%endcodeblock%}
{%codeblock lang:java UserDataRealm%}
//以此为例,combine的逻辑就是依次处理list中realm的start
 protected void startInternal() throws LifecycleException {

         protected void startInternal() throws LifecycleException {

        try {
            Context context = getServer().getGlobalNamingContext(); //从Naming中取得database这个Resource
            database = (UserDatabase) context.lookup(resourceName);
        } catch (Throwable e) {
            ExceptionUtils.handleThrowable(e);
            containerLog.error(sm.getString("userDatabaseRealm.lookup",
                                            resourceName),
                               e);
            database = null;
        }
        if (database == null) {
            throw new LifecycleException
                (sm.getString("userDatabaseRealm.noDatabase", resourceName));
        }

        super.startInternal(); //执行父类逻辑
    }
{%endcodeblock%}
#### pipeline
管道模式是tomcat中对于请求逻辑处理的分布化的方式,通过不同容器组件持有不同pipe,逐层传递,每一个pipe调用内部Valve处理一部分数据逻辑
{% asset_img  StandarPipeline.png 继承图%}

{%codeblock lang:java StandardPipeline %}
// coyoteAdapter进行res/rep处理->Engine.pipe->Host.pipe->Context.pipe->Wrapper.pipe 逐层处理逻辑
// 实际上pipe是拥有Valve的组件,每层容器调用自身pipe中Valve节点,由Valve节点完成真正的复杂逻辑工作

//pipe是containerBase对象成员
//basic是对应容器构造函数中添加的
/*
如:
* public StandardHost() {

        super();
        pipeline.setBasic(new StandardHostValve());
    }
*
/
protected Valve basic = null;
//当调用addValve时加入
 protected Valve first = null;
 //保持的关系是 first->Valve->Valve-.....->basic,也就是说basic在这个node结构的最后
//该函数使用在xml解析的过程中,创建新的Valve,加入到对应container的pipeline中
//pipeline 
public void addValve(Valve valve) {

        // Validate that we can add this Valve
        if (valve instanceof Contained)
            ((Contained) valve).setContainer(this.container);

        // Start the new component if necessary
        if (getState().isAvailable()) {
            if (valve instanceof Lifecycle) {
                try {
                    ((Lifecycle) valve).start();
                } catch (LifecycleException e) {
                    log.error("StandardPipeline.addValve: start: ", e);
                }
            }
        }

        // Add this Valve to the set associated with this Pipeline
        if (first == null) {
            first = valve;
            valve.setNext(basic);
        } else {
        //维持上述的节点关系
            Valve current = first;
            while (current != null) {
                if (current.getNext() == basic) {
                    current.setNext(valve);
                    valve.setNext(basic);
                    break;
                }
                current = current.getNext();
            }
        }

        container.fireContainerEvent(Container.ADD_VALVE_EVENT, valve);
    }
    
      protected synchronized void startInternal() throws LifecycleException {

        // Start the Valves in our pipeline (including the basic), if any
        Valve current = first;
        if (current == null) {
            current = basic;
        }
        //按照节点关系顺序执行对应的start,pipe的启动在对应容器start阶段
        while (current != null) {
            if (current instanceof Lifecycle)
                ((Lifecycle) current).start();
            current = current.getNext();
        }

        setState(LifecycleState.STARTING);
    }
{%endcodeblock%}

#### Valve 
{% asset_img StandardHostValve.png 继承%}
<p>A <b>Valve</b> is a request processing component associated with a
 * particular Container.  A series of Valves are generally associated with
 * each other into a Pipeline.  The detailed contract for a Valve is included
 * in the description of the <code>invoke()</code> method below.</p>
如注释所言,valve是容器执行请求的组件,valve与pipeline关联。
{%codeblock lang:java Valve%}
//上层life各个阶段只改变状态
 @Override
    protected void initInternal() throws LifecycleException {
        super.initInternal();
        containerLog = getContainer().getLogger();
    }


    /**
     * Start this component and implement the requirements
     * of {@link org.apache.catalina.util.LifecycleBase#startInternal()}.
     *
     * @exception LifecycleException if this component detects a fatal error
     *  that prevents this component from being used
     */
    @Override
    protected synchronized void startInternal() throws LifecycleException {
        setState(LifecycleState.STARTING);
    }


    /**
     * Stop this component and implement the requirements
     * of {@link org.apache.catalina.util.LifecycleBase#stopInternal()}.
     *
     * @exception LifecycleException if this component detects a fatal error
     *  that prevents this component from being used
     */
    @Override
    protected synchronized void stopInternal() throws LifecycleException {
        setState(LifecycleState.STOPPING);
    }
{%endcodeblock%}

#### tomcat部署

{%codeblock lang:java%}
    public static final String Package = "org.apache.catalina.startup";
    public static final String ApplicationContextXml = "META-INF/context.xml";
    public static final String ApplicationWebXml = "/WEB-INF/web.xml";
    public static final String TomcatWebXml = "/WEB-INF/tomcat-web.xml";
    public static final String DefaultContextXml = "conf/context.xml";
    public static final String DefaultWebXml = "conf/web.xml";
    public static final String HostContextXml = "context.xml.default";
    public static final String HostWebXml = "web.xml.default";
    public static final String WarTracker = "/META-INF/war-tracker";
{%endcodeblock%}
#### NioEndPoint底层机制
##### 连接请求到执行器处理
- uml关系图
{% asset_img NioEndPoint.jpg %}
- acceptor:用来接受新的连接,使用blocking方式接受,将新的连接通过注册到轮询事件队列中
{%codeblock lang:java acceptor%}
//该线程启动部分在connector.start
public class Acceptor<U> implements Runnable {
public void run() {

        int errorDelay = 0;

        // Loop until we receive a shutdown command
        while (endpoint.isRunning()) {

            // Loop if endpoint is paused
            while (endpoint.isPaused() && endpoint.isRunning()) {
                state = AcceptorState.PAUSED;
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    // Ignore
                }
            }

            if (!endpoint.isRunning()) {
                break;
            }
            state = AcceptorState.RUNNING;

            try {
                //if we have reached max connections, wait
                endpoint.countUpOrAwaitConnection();  //管理连接数量,到达数量此处阻塞,就不会接受新的连接了

                // Endpoint might have been paused while waiting for latch
                // If that is the case, don't accept new connections
                if (endpoint.isPaused()) { //point暂停时不要接受任何输入
                    continue;
                }

                U socket = null; //此处看NioEndPoint代码 SocketChannel
                try {
                    // Accept the next incoming connection from the server
                    // socket
                    socket = endpoint.serverSocketAccept();  //执行accept,那么看来对于接受外界socket还是要通过这种方式,并且此处是阻塞的,返回socketChanel
                } catch (Exception ioe) {
                    // We didn't get a socket
                    endpoint.countDownConnection();
                    if (endpoint.isRunning()) {
                        // Introduce delay if necessary
                        errorDelay = handleExceptionWithDelay(errorDelay);
                        // re-throw
                        throw ioe;
                    } else {
                        break;
                    }
                }
                // Successful accept, reset the error delay
                errorDelay = 0;

                // Configure the socket
                if (endpoint.isRunning() && !endpoint.isPaused()) {
                    // setSocketOptions() will hand the socket off to
                    // an appropriate processor if successful
                    if (!endpoint.setSocketOptions(socket)) { //调用对应endpoint,将接受到的socket通过轮询poller加入到注册事件中
                        endpoint.closeSocket(socket);
                    }
                } else {
                    endpoint.destroySocket(socket);
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                String msg = sm.getString("endpoint.accept.fail");
                // APR specific.
                // Could push this down but not sure it is worth the trouble.
                if (t instanceof Error) {
                    Error e = (Error) t;
                    if (e.getError() == 233) {
                        // Not an error on HP-UX so log as a warning
                        // so it can be filtered out on that platform
                        // See bug 50273
                        log.warn(msg, t);
                    } else {
                        log.error(msg, t);
                    }
                } else {
                        log.error(msg, t);
                }
            }
        }
        state = AcceptorState.ENDED;
    }

}
//NioEndPoint的实现
protected boolean setSocketOptions(SocketChannel socket) {
        // Process the connection
        try {
            //disable blocking, APR style, we are gonna be polling it
            socket.configureBlocking(false);
            Socket sock = socket.socket();
            socketProperties.setProperties(sock);

            NioChannel channel = nioChannels.pop(); //从nio队列中取出,并进行封装
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler( //构造buffer
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {  //分别处理是否是ssl
                    channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
                } else {
                    channel = new NioChannel(socket, bufhandler);
                }
            } else {
                channel.setIOChannel(socket);
                channel.reset();
            }
            getPoller0().register(channel); //构造注册事件
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            try {
                log.error("",t);
            } catch (Throwable tt) {
                ExceptionUtils.handleThrowable(tt);
            }
            // Tell to close the socket
            return false;
        }
        return true;
    }
{%endcodeblock%}

- pollerEvent:轮询事件的种类, 1.注册 2.改变ops
{%codeblock lang:java%}
//为nioEndPoint内部类,poller事件队列,描述该线程队列如何处理
 public static class PollerEvent implements Runnable {

        private NioChannel socket;  //关联的Channel
        private int interestOps;    //该事件的操作类型
        private NioSocketWrapper socketWrapper; //包装

        public PollerEvent(NioChannel ch, NioSocketWrapper w, int intOps) {
            reset(ch, w, intOps);
        }

        public void reset(NioChannel ch, NioSocketWrapper w, int intOps) {
            socket = ch;
            interestOps = intOps;
            socketWrapper = w;
        }

        public void reset() {
            reset(null, null, 0);
        }

        /**
         *  key实际上只有这四种关心的操作
         * @see SelectionKey#OP_WRITE
         * @see SelectionKey#OP_READ
         * @see SelectionKey#OP_ACCEPT
         * @see SelectionKey#OP_CONNECT
         *   因此pollEvent事件线程执行的逻辑
         *    1.出现新的连接,将该channel注册到selector中
         *    2.由于key被取消,进行连接数的减少
         *    3.对某个channel对应的key,增加其interestOps
         *    4.socketWrapper=null,取消对应的key
         */
        @Override
        public void run() {
            if (interestOps == OP_REGISTER) {  //注册事件
                try {
                    socket.getIOChannel().register(
                            socket.getPoller().getSelector(), SelectionKey.OP_READ, socketWrapper);
                    //将该channel以read的方式注册到poller线程中的selector,也就是此key关心的是OP_READ
                } catch (Exception x) {
                    log.error(sm.getString("endpoint.nio.registerFail"), x);
                }
            } else {
                final SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());  //找到此channel对应的key
                try {
                    if (key == null) {
                        // The key was cancelled (e.g. due to socket closure)
                        // and removed from the selector while it was being
                        // processed. Count down the connections at this point
                        // since it won't have been counted down when the socket
                        // closed. 此处表示该该key被取消,因此要减少一个连接数
                        socket.socketWrapper.getEndpoint().countDownConnection(); //连接数和limitLatch有关,此处调用countDownConnection减少一个
                    } else {
                        final NioSocketWrapper socketWrapper = (NioSocketWrapper) key.attachment();
                        if (socketWrapper != null) {
                            //we are registering the key to start with, reset the fairness counter.
                            int ops = key.interestOps() | interestOps;  //增加该key关心的操作
                            socketWrapper.interestOps(ops);
                            key.interestOps(ops); //改变key关心的操作类型
                        } else {
                            socket.getPoller().cancelledKey(key); //取消该key
                        }
                    }
                } catch (CancelledKeyException ckx) {
                    try {
                        socket.getPoller().cancelledKey(key);
                    } catch (Exception ignore) {}
                }
            }
        }

        @Override
        public String toString() {
            return "Poller event: socket [" + socket + "], socketWrapper [" + socketWrapper +
                    "], interestOps [" + interestOps + "]";
        }
    }
{%endcodeblock%}


- poller:轮询线程,当acceptor注册,对应nio改变ops,都会导致事件的添加,该线程不断处理事件队列中的线程对象,并且进行selecotr的阻塞,判断有无就绪的SocketChnanel
{%codeblock lang:java%}
public class Poller implements Runnable {
  private Selector selector; //选择器
        private final SynchronizedQueue<PollerEvent> events =
                new SynchronizedQueue<>(); //事件队列

        private volatile boolean close = false;
        private long nextExpiration = 0;//optimize expiration handling
        //具体的逻辑就是当有添加evnent时增加1,若==0,则唤醒
        //
        private AtomicLong wakeupCounter = new AtomicLong(0); //用来唤醒selector.accpet部分辅助的量
        

        private volatile int keyCount = 0;
        
        private void addEvent(PollerEvent event) {
            events.offer(event);
            if ( wakeupCounter.incrementAndGet() == 0 ) selector.wakeup(); //此处是唤醒selector,使得run代码向下循环,执行事件线程
        }
         public boolean events() { //执行队列中事件,注意这个并发队列不是以前我看的没有内容线程暂停的,是tomcat自己实现的加了
            //synchronized的队列,代码易理解
            boolean result = false;

            PollerEvent pe = null;
            while ( (pe = events.poll()) != null ) {
                result = true;
                try {
                    pe.run();
                    pe.reset();
                    if (running && !paused) {
                        eventCache.push(pe);
                    }
                } catch ( Throwable x ) {
                    log.error("",x);
                }
            }

            return result;
        }
        
         public void run() {
            // Loop until destroy() is called
            while (true) {

                boolean hasEvents = false;

                try {
                    if (!close) {
                        hasEvents = events(); //不停执行事件
                        if (wakeupCounter.getAndSet(-1) > 0) {
                            //获取当前值,并且置于-1,如果当前值>0,那么说明有好几个事件加入了
                            // ,那么就要及时唤醒,进入下一次循环处理事件
                            //if we are here, means we have other stuff to do
                            //do a non blocking select
                            keyCount = selector.selectNow();  //返回的是就绪key 的数量
                        } else {
                            //当且仅当-1才会进入这里,于此同时如果addEvent操作,就会唤醒,并count+1,且唤醒
                            keyCount = selector.select(selectorTimeout);
                        }
                        //因为一次循环就会创建事件队列中所有的线程,因此此处置为0
                        wakeupCounter.set(0);
                    }
                    if (close) {
                        events();
                        timeout(0, false);
                        try {
                            selector.close();
                        } catch (IOException ioe) {
                            log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                        }
                        break;
                    }
                } catch (Throwable x) {
                    ExceptionUtils.handleThrowable(x);
                    log.error("",x);
                    continue;
                }
                //either we timed out or we woke up, process events first
                //假设超时或者被主动唤醒,说明此处没有获得能使用键,执行事件,并且获取此时系统中有无事件线程
                if ( keyCount == 0 ) hasEvents = (hasEvents | events());
                //当且仅当有readyKey才进行遍历
                Iterator<SelectionKey> iterator =
                    keyCount > 0 ? selector.selectedKeys().iterator() : null;
                // Walk through the collection of ready keys and dispatch
                // any active event.
                while (iterator != null && iterator.hasNext()) {  //遍历
                    SelectionKey sk = iterator.next();
                    NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
                    // Attachment may be null if another thread has called
                    // cancelledKey()
                    if (attachment == null) {
                        iterator.remove();
                    } else {
                        iterator.remove();
                        processKey(sk, attachment);  //执行
                    }
                }//while

                //process timeouts
                timeout(keyCount,hasEvents);
            }//while

            getStopLatch().countDown(); //减少latch,对应的await在stop中,实际上说明await调用只要在对应线程的创建线程中就可以,await在stop中
        }
        
        protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
            try {
                if ( close ) {
                    cancelledKey(sk);
                } else if ( sk.isValid() && attachment != null ) {  //真正获得连接后如何执行
                    if (sk.isReadable() || sk.isWritable() ) {
                        if ( attachment.getSendfileData() != null ) { //如果有fileData,执行发送文件
                            processSendfile(sk,attachment, false);
                        } else {
                            unreg(sk, attachment, sk.readyOps());  //取  ~sk.readyOps &sk.interestOps  改为interestOps
                            boolean closeSocket = false;
                            // Read goes before write
                            /**
                             * @see AbstractEndpoint#processSocket(SocketWrapperBase, SocketEvent, boolean) 处理
                             */
                            if (sk.isReadable()) {  
                                if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                                    closeSocket = true;
                                }
                            }
                            if (!closeSocket && sk.isWritable()) {
                                if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                                    closeSocket = true;
                                }
                            }
                            if (closeSocket) {
                                cancelledKey(sk);
                            }
                        }
                    }
                } else {
                    //invalid key
                    cancelledKey(sk);
                }
            } catch ( CancelledKeyException ckx ) {
                cancelledKey(sk);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error("",t);
            }
        }
    }
    //AbstractEndpoint中
    public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {
            if (socketWrapper == null) {
                return false;
            }
            SocketProcessorBase<S> sc = processorCache.pop();  //SocketProcessorBase 执行器也用了队列缓冲
            if (sc == null) {
                sc = createSocketProcessor(socketWrapper, event);//子类创建,这个sc就是下文中的SocketProcessor
            } else {
                sc.reset(socketWrapper, event);
            }
            Executor executor = getExecutor();
            if (dispatch && executor != null) { //该执行器线程执行
                executor.execute(sc);
            } else {
                sc.run();
            }
        } catch (RejectedExecutionException ree) {
            getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
            return false;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This means we got an OOM or similar creating a thread, or that
            // the pool and its queue are full
            getLog().error(sm.getString("endpoint.process.fail"), t);
            return false;
        }
        return true;
    }
{%endcodeblock%}

- SocketProcessor: NioEndpoint内部类,实际是一个线程类,用来调用对应的processor来进行处理
{%codeblock lang:java%}
    protected class SocketProcessor extends SocketProcessorBase<NioChannel> {

        public SocketProcessor(SocketWrapperBase<NioChannel> socketWrapper, SocketEvent event) {
            super(socketWrapper, event);
        }

        @Override
        protected void doRun() {
            NioChannel socket = socketWrapper.getSocket();
            SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());

            try {
                int handshake = -1;

                try {
                    if (key != null) {
                        if (socket.isHandshakeComplete()) {// 这几个判断都是针对于ssl进行的
                            // No TLS handshaking required. Let the handler
                            // process this socket / event combination.
                            handshake = 0;
                        } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                                event == SocketEvent.ERROR) {
                            // Unable to complete the TLS handshake. Treat it as
                            // if the handshake failed.
                            handshake = -1;
                        } else {
                            handshake = socket.handshake(key.isReadable(), key.isWritable());
                            // The handshake process reads/writes from/to the
                            // socket. status may therefore be OPEN_WRITE once
                            // the handshake completes. However, the handshake
                            // happens when the socket is opened so the status
                            // must always be OPEN_READ after it completes. It
                            // is OK to always set this as it is only used if
                            // the handshake completes.
                            event = SocketEvent.OPEN_READ;
                        }
                    }
                } catch (IOException x) {
                    handshake = -1;
                    if (log.isDebugEnabled()) log.debug("Error during SSL handshake",x);
                } catch (CancelledKeyException ckx) {
                    handshake = -1;
                }
                if (handshake == 0) {
                    SocketState state = SocketState.OPEN;
                    // Process the request from this socket
                    if (event == null) {
                        state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                    } else {
                        state = getHandler().process(socketWrapper, event);  //执行对应的event逻辑
                    }
                    if (state == SocketState.CLOSED) {
                        close(socket, key);
                    }
                } else if (handshake == -1 ) {
                    close(socket, key);
                } else if (handshake == SelectionKey.OP_READ){
                    socketWrapper.registerReadInterest();
                } else if (handshake == SelectionKey.OP_WRITE){
                    socketWrapper.registerWriteInterest();
                }
            } catch (CancelledKeyException cx) {
                socket.getPoller().cancelledKey(key);
            } catch (VirtualMachineError vme) {
                ExceptionUtils.handleThrowable(vme);
            } catch (Throwable t) {
                log.error("", t);
                socket.getPoller().cancelledKey(key);
            } finally {
                socketWrapper = null;
                event = null;
                //return to cache
                if (running && !paused) {
                    processorCache.push(this);
                }
            }
        }
    }

{%endcodeblock%}
- ConnectionHandler: 将请求连接就绪的socket进行处理,创建执行器,这部分已经进入了子线程
{%codeblock lang:java ConnectionHandler%}
 public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
            if (getLog().isDebugEnabled()) {
                getLog().debug(sm.getString("abstractConnectionHandler.process",
                        wrapper.getSocket(), status));
            }
            if (wrapper == null) {
                // Nothing to do. Socket has been closed.
                return SocketState.CLOSED;
            }

            S socket = wrapper.getSocket();

            Processor processor = connections.get(socket);
            if (getLog().isDebugEnabled()) {
                getLog().debug(sm.getString("abstractConnectionHandler.connectionsGet",
                        processor, socket));
            }

            if (processor != null) {
                // Make sure an async timeout doesn't fire
                getProtocol().removeWaitingProcessor(processor);
            } else if (status == SocketEvent.DISCONNECT || status == SocketEvent.ERROR) {
                // Nothing to do. Endpoint requested a close and there is no
                // longer a processor associated with this socket.
                return SocketState.CLOSED;
            }

            ContainerThreadMarker.set();

            try {
                if (processor == null) {
                    String negotiatedProtocol = wrapper.getNegotiatedProtocol();
                    if (negotiatedProtocol != null) {
                        UpgradeProtocol upgradeProtocol =
                                getProtocol().getNegotiatedProtocol(negotiatedProtocol);
                        if (upgradeProtocol != null) {
                            processor = upgradeProtocol.getProcessor(
                                    wrapper, getProtocol().getAdapter());
                        } else if (negotiatedProtocol.equals("http/1.1")) {
                            // Explicitly negotiated the default protocol.
                            // Obtain a processor below.
                        } else {
                            // TODO:
                            // OpenSSL 1.0.2's ALPN callback doesn't support
                            // failing the handshake with an error if no
                            // protocol can be negotiated. Therefore, we need to
                            // fail the connection here. Once this is fixed,
                            // replace the code below with the commented out
                            // block.
                            if (getLog().isDebugEnabled()) {
                                getLog().debug(sm.getString(
                                    "abstractConnectionHandler.negotiatedProcessor.fail",
                                    negotiatedProtocol));
                            }
                            return SocketState.CLOSED;
                            /*
                             * To replace the code above once OpenSSL 1.1.0 is
                             * used.
                            // Failed to create processor. This is a bug.
                            throw new IllegalStateException(sm.getString(
                                    "abstractConnectionHandler.negotiatedProcessor.fail",
                                    negotiatedProtocol));
                            */
                        }
                    }
                }
                /**
                 *  这几种执行器
                 * @see org.apache.coyote.http11.Http11Processor
                 * @see org.apache.coyote.ajp.AjpProcessor
                 * @see org.apache.coyote.http2.StreamProcessor
                 */
                if (processor == null) {
                    processor = recycledProcessors.pop();
                    if (getLog().isDebugEnabled()) {
                        getLog().debug(sm.getString("abstractConnectionHandler.processorPop",
                                processor));
                    }
                }
                if (processor == null) {
                    processor = getProtocol().createProcessor();  ////实际上这个processor 由对应的protocalHandler创建
                    register(processor); //加入到jmx
                }

                processor.setSslSupport(
                        wrapper.getSslSupport(getProtocol().getClientCertProvider()));  //还是对ssl有用,也就是 SecureNioChannel这样的信道

                // Associate the processor with the connection
                connections.put(socket, processor);  //缓冲

                SocketState state = SocketState.CLOSED;
                do {
                    state = processor.process(wrapper, status);  //真正tomcat对于连接进行处理的入口点
                    //之后的代码都是根据socketState进行的处理

                    if (state == SocketState.UPGRADING) {
                        // Get the HTTP upgrade handler
                        UpgradeToken upgradeToken = processor.getUpgradeToken();
                        // Retrieve leftover input
                        ByteBuffer leftOverInput = processor.getLeftoverInput();
                        if (upgradeToken == null) {
                            // Assume direct HTTP/2 connection
                            UpgradeProtocol upgradeProtocol = getProtocol().getUpgradeProtocol("h2c");
                            if (upgradeProtocol != null) {
                                processor = upgradeProtocol.getProcessor(
                                        wrapper, getProtocol().getAdapter());
                                wrapper.unRead(leftOverInput);
                                // Associate with the processor with the connection
                                connections.put(socket, processor);
                            } else {
                                if (getLog().isDebugEnabled()) {
                                    getLog().debug(sm.getString(
                                        "abstractConnectionHandler.negotiatedProcessor.fail",
                                        "h2c"));
                                }
                                return SocketState.CLOSED;
                            }
                        } else {
                            HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                            // Release the Http11 processor to be re-used
                            release(processor);
                            // Create the upgrade processor
                            processor = getProtocol().createUpgradeProcessor(wrapper, upgradeToken);
                            if (getLog().isDebugEnabled()) {
                                getLog().debug(sm.getString("abstractConnectionHandler.upgradeCreate",
                                        processor, wrapper));
                            }
                            wrapper.unRead(leftOverInput);
                            // Mark the connection as upgraded
                            wrapper.setUpgraded(true);
                            // Associate with the processor with the connection
                            connections.put(socket, processor);
                            // Initialise the upgrade handler (which may trigger
                            // some IO using the new protocol which is why the lines
                            // above are necessary)
                            // This cast should be safe. If it fails the error
                            // handling for the surrounding try/catch will deal with
                            // it.
                            if (upgradeToken.getInstanceManager() == null) {
                                httpUpgradeHandler.init((WebConnection) processor);
                            } else {
                                ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                                try {
                                    httpUpgradeHandler.init((WebConnection) processor);
                                } finally {
                                    upgradeToken.getContextBind().unbind(false, oldCL);
                                }
                            }
                        }
                    }
                } while ( state == SocketState.UPGRADING);

                if (state == SocketState.LONG) {
                    // In the middle of processing a request/response. Keep the
                    // socket associated with the processor. Exact requirements
                    // depend on type of long poll
                    longPoll(wrapper, processor);
                    if (processor.isAsync()) {
                        getProtocol().addWaitingProcessor(processor);
                    }
                } else if (state == SocketState.OPEN) {
                    // In keep-alive but between requests. OK to recycle
                    // processor. Continue to poll for the next request.
                    connections.remove(socket);
                    release(processor);
                    wrapper.registerReadInterest();
                } else if (state == SocketState.SENDFILE) {
                    // Sendfile in progress. If it fails, the socket will be
                    // closed. If it works, the socket either be added to the
                    // poller (or equivalent) to await more data or processed
                    // if there are any pipe-lined requests remaining.
                } else if (state == SocketState.UPGRADED) {
                    // Don't add sockets back to the poller if this was a
                    // non-blocking write otherwise the poller may trigger
                    // multiple read events which may lead to thread starvation
                    // in the connector. The write() method will add this socket
                    // to the poller if necessary.
                    if (status != SocketEvent.OPEN_WRITE) {
                        longPoll(wrapper, processor);
                    }
                } else if (state == SocketState.SUSPENDED) {
                    // Don't add sockets back to the poller.
                    // The resumeProcessing() method will add this socket
                    // to the poller.
                } else {
                    // Connection closed. OK to recycle the processor. Upgrade
                    // processors are not recycled.
                    connections.remove(socket);
                    if (processor.isUpgrade()) {
                        UpgradeToken upgradeToken = processor.getUpgradeToken();
                        HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                        InstanceManager instanceManager = upgradeToken.getInstanceManager();
                        if (instanceManager == null) {
                            httpUpgradeHandler.destroy();
                        } else {
                            ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                            try {
                                httpUpgradeHandler.destroy();
                            } finally {
                                try {
                                    instanceManager.destroyInstance(httpUpgradeHandler);
                                } catch (Throwable e) {
                                    ExceptionUtils.handleThrowable(e);
                                    getLog().error(sm.getString("abstractConnectionHandler.error"), e);
                                }
                                upgradeToken.getContextBind().unbind(false, oldCL);
                            }
                        }
                    } else {
                        release(processor);
                    }
                }
                return state;
            } catch(java.net.SocketException e) {
                // SocketExceptions are normal
                getLog().debug(sm.getString(
                        "abstractConnectionHandler.socketexception.debug"), e);
            } catch (java.io.IOException e) {
                // IOExceptions are normal
                getLog().debug(sm.getString(
                        "abstractConnectionHandler.ioexception.debug"), e);
            } catch (ProtocolException e) {
                // Protocol exceptions normally mean the client sent invalid or
                // incomplete data.
                getLog().debug(sm.getString(
                        "abstractConnectionHandler.protocolexception.debug"), e);
            }
            // Future developers: if you discover any other
            // rare-but-nonfatal exceptions, catch them here, and log as
            // above.
            catch (Throwable e) {
                ExceptionUtils.handleThrowable(e);
                // any other exception or error is odd. Here we log it
                // with "ERROR" level, so it will show up even on
                // less-than-verbose logs.
                getLog().error(sm.getString("abstractConnectionHandler.error"), e);
            } finally {
                ContainerThreadMarker.clear();
            }

            // Make sure socket/processor is removed from the list of current
            // connections
            connections.remove(socket);
            release(processor);
            return SocketState.CLOSED;
        }

{%endcodeblock%}
##### nio读写逻辑相关
- uml
{%asset_img NioChannel.jpg%}
- NioSocketWrapper:封装了nioChannel,实现了如何进行读写操作,要么通过nioApi直接读写,要么通过SelectorPool

{%codeblock lang:java %}
 public static class  NioSocketWrapper extends SocketWrapperBase<NioChannel> {
        //其父类封装了NioChannel/SocketBufferHandler相关的包装操作,实际进行读写的逻辑交给子类是实现,也就是NioSocketWrapper这种子类实现,实现有三种
        //也就是说父类提供公共接口由外部调用,由子类实现读写逻辑
        //fillReadBuffer和doWrite 实现从读取数据到buffer和将构造好的buffer写出去
        //父类提供写接口 为write函数,有许多重写
        //读接口因该是read
        
        //分别对应nio nio2 apr,这样的逻辑和c++ io相似,和java io包装的方式有些相反
        //这个类有许多跟CountDownLatch相关的
        private final NioSelectorPool pool;

        private Poller poller = null;
        private int interestOps = 0;
        private CountDownLatch readLatch = null; //读写latch,用于NioBlockSelector
        private CountDownLatch writeLatch = null;
        private volatile SendfileData sendfileData = null;
        private volatile long lastRead = System.currentTimeMillis();
        private volatile long lastWrite = lastRead;

        public NioSocketWrapper(NioChannel channel, NioEndpoint endpoint) {
            super(channel, endpoint);
            pool = endpoint.getSelectorPool();
            socketBufferHandler = channel.getBufHandler();
        }
        
        //写逻辑
        @Override
        protected void doWrite(boolean block, ByteBuffer from) throws IOException {
            long writeTimeout = getWriteTimeout();  //超时数
            Selector selector = null;
            try {
                selector = pool.get();  //获取selectorPoll的一个
            } catch (IOException x) {
                // Ignore
            }
            try {
                pool.write(from, getSocket(), selector, writeTimeout, block);
                if (block) {
                    // Make sure we are flushed
                    do {
                        if (getSocket().flush(true, selector, writeTimeout)) {  //非ssl此处没有用
                            break;
                        }
                    } while (true);
                }
                updateLastWrite();
            } finally {
                if (selector != null) {
                    pool.put(selector); //并没有销毁selector
                }
            }
            // If there is data left in the buffer the socket will be registered for
            // write further up the stack. This is to ensure the socket is only
            // registered for write once as both container and user code can trigger
            // write registration.
        }
        //读
        public int read(boolean block, byte[] b, int off, int len) throws IOException {
            int nRead = populateReadBuffer(b, off, len); //向buffer中取数据
            if (nRead > 0) { ////这个逻辑是内部的buffer已经有数据了,此时将buffer数据
            //放置到b中就可以了
                return nRead;
                /*
                 * Since more bytes may have arrived since the buffer was last
                 * filled, it is an option at this point to perform a
                 * non-blocking read. However correctly handling the case if
                 * that read returns end of stream adds complexity. Therefore,
                 * at the moment, the preference is for simplicity.
                 */
            }

            // Fill the read buffer as best we can.
            nRead = fillReadBuffer(block);   //这个函数是NioSocketWrapper实现的真正读取逻辑,也就是向buffer中添加
            //数据
            updateLastRead();

            // Fill as much of the remaining byte array as possible with the
            // data that was just read
            if (nRead > 0) {
                socketBufferHandler.configureReadBufferForRead();
                nRead = Math.min(nRead, len);
                socketBufferHandler.getReadBuffer().get(b, off, nRead);
            }
            return nRead;
        }
        
        private int fillReadBuffer(boolean block) throws IOException {
            socketBufferHandler.configureReadBufferForWrite();
            return fillReadBuffer(block, socketBufferHandler.getReadBuffer());
        }


        private int fillReadBuffer(boolean block, ByteBuffer to) throws IOException {
            int nRead;
            NioChannel channel = getSocket();
            if (block) {
                Selector selector = null;
                try {
                    selector = pool.get(); //block 读则调用
                } catch (IOException x) {
                    // Ignore
                }
                try {
                    NioEndpoint.NioSocketWrapper att = (NioEndpoint.NioSocketWrapper) channel
                            .getAttachment();
                    if (att == null) {
                        throw new IOException("Key must be cancelled.");
                    }
                    nRead = pool.read(to, channel, selector, att.getReadTimeout()); //通过pool来处理
                } finally {
                    if (selector != null) {
                        pool.put(selector);
                    }
                }
            } else {
                nRead = channel.read(to); //非blocking读则调用该函数,此处代码就是真正的 channel.read(buffer)
                if (nRead == -1) {
                    throw new EOFException();
                }
            }
            return nRead;
        }
{%endcodeblock%}
- NioChannel:封装了实际的socketChannel,并且是endPoint|procol的泛型参数之一
{%codeblock lang:java NioChannel%}
public class NioChannel implements ByteChannel {

    protected static final StringManager sm = StringManager.getManager(NioChannel.class);

    protected static final ByteBuffer emptyBuf = ByteBuffer.allocate(0);

    protected SocketChannel sc = null; //真正的socketChannel
    protected SocketWrapperBase<NioChannel> socketWrapper = null;

    protected final SocketBufferHandler bufHandler; //持有读写buffer和相关调整方法

    protected Poller poller;

    public NioChannel(SocketChannel channel, SocketBufferHandler bufHandler) {
        this.sc = channel;
        this.bufHandler = bufHandler;
    }
	
	 public void free() {
        bufHandler.free(); //处理内部buffer
    }
	public void close() throws IOException { //关闭sc
        getIOChannel().socket().close();
        getIOChannel().close();
    }
	 public int write(ByteBuffer src) throws IOException { //实际进行读写操作
        checkInterruptStatus();
        return sc.write(src);
    }
	 public int read(ByteBuffer dst) throws IOException {
        return sc.read(dst);
    }
{%endcodeblock%}
- SocketBufferHandler:持有读写buffer,并有调整方法
{%codeblock lang:java SocketBufferHandler%}
   private volatile boolean readBufferConfiguredForWrite = true;
    private volatile ByteBuffer readBuffer;

    private volatile boolean writeBufferConfiguredForWrite = true;
    private volatile ByteBuffer writeBuffer;

    private final boolean direct;

    public SocketBufferHandler(int readBufferSize, int writeBufferSize,  //由连接器获取连接之后创建对应channel时创建,并被wrapper和channel持有
            boolean direct) {
        this.direct = direct;
        if (direct) {
            readBuffer = ByteBuffer.allocateDirect(readBufferSize);
            writeBuffer = ByteBuffer.allocateDirect(writeBufferSize);
        } else {
            readBuffer = ByteBuffer.allocate(readBufferSize);
            writeBuffer = ByteBuffer.allocate(writeBufferSize);
        }
    }
{%endcodeblock%}
- NioSelectorPool:读写使用了selector多路复用,这里说的blocking应该指的是selecotr的blocking
{%codeblock lang:java NioSelectorPool%}
//该类实现处理读写channel缓冲逻辑,其中NioBlockingSelector部分逻辑在connector部分写了

public class NioSelectorPool {
 protected ConcurrentLinkedQueue<Selector> selectors =
            new ConcurrentLinkedQueue<>(); //不使用block的选择器缓冲
  protected NioBlockingSelector blockingSelector; //blocking使用的选择器      
  
    //写逻辑,这里有部分代码没看,就是tomcat从Servlet,还是什么组件进行的写回操作
     public int write(ByteBuffer buf, NioChannel socket, Selector selector,
                     long writeTimeout, boolean block) throws IOException {
        if ( SHARED && block ) {
            return blockingSelector.write(buf,socket,writeTimeout);  //只有阻塞并且SHARED=true,调用逻辑在connector部分写了
        }
        SelectionKey key = null;
        int written = 0;
        boolean timedout = false;
        int keycount = 1; //assume we can write
        long time = System.currentTimeMillis(); //start the timeout timer  //下边代码实际相似
        try {
            while ( (!timedout) && buf.hasRemaining() ) {  //Remain =limit-position
                int cnt = 0;
                if ( keycount > 0 ) { //only write if we were registered for a write
                    cnt = socket.write(buf); //write the data
                    if (cnt == -1) throw new EOFException();

                    written += cnt;  //写出的数量
                    if (cnt > 0) {
                        time = System.currentTimeMillis(); //reset our timeout timer
                        continue; //we successfully wrote, try again without a selector//可以理解为如果能写出数据,那就说明此时写channel是可用的
                    }
                    if (cnt==0 && (!block)) break; //don't block  如果=0且非阻塞则跳出
                }
                //只有当这一次没有写出,并且还是阻塞,才执行以下逻辑,意思就是如果没有写出数据,并且服务器允许对此次写进行等待,那么执行以下逻辑
                if ( selector != null ) {
                    //register OP_WRITE to the selector
                    if (key==null) key = socket.getIOChannel().register(selector, SelectionKey.OP_WRITE); //注册
                    else key.interestOps(SelectionKey.OP_WRITE);
                    if (writeTimeout==0) {
                        timedout = buf.hasRemaining();//假设返回true[1]->[2]  这个逻辑是只能进行一次写操作,如果没写完就超时异常
                    } else if (writeTimeout<0) { //阻塞逻辑
                        keycount = selector.select(); //这个逻辑说明一直进行写操作,必须等到buffer写完,否则一直继续循环
                    } else {
                        keycount = selector.select(writeTimeout);//[3]->[4]
                    }
                }
                //[2]:抛出异常  [4]:判断写代码用的时间是否>writeTimeout 最大超时时间,大于就抛出异常
                if (writeTimeout > 0 && (selector == null || keycount == 0) ) timedout = (System.currentTimeMillis()-time)>=writeTimeout;
            }//while
            if ( timedout ) throw new SocketTimeoutException();
        } finally {
            if (key != null) {
                key.cancel();
                if (selector != null) selector.selectNow();//removes the key from this selector
            }
        }
        return written;
    }
    
 //读取的逻辑   
  public int read(ByteBuffer buf, NioChannel socket, Selector selector, long readTimeout) throws IOException {
        return read(buf,socket,selector,readTimeout,true);  //idea的提示功能来进行这种读取的代码调用就这一种,也就是  //说如果是这种调用,那么必定是阻塞式除非使用反射
    }
    public int read(ByteBuffer buf, NioChannel socket, Selector selector, long readTimeout, boolean block) throws IOException {
        if ( SHARED && block ) {
            return blockingSelector.read(buf,socket,readTimeout);
        }
        SelectionKey key = null;
        int read = 0;
        boolean timedout = false;
        int keycount = 1; //assume we can write
        long time = System.currentTimeMillis(); //start the timeout timer
        try {
            while ( (!timedout) ) {
                int cnt = 0;
                if ( keycount > 0 ) { //only read if we were registered for a read
                    cnt = socket.read(buf);
                    if (cnt == -1) { //某一次读取-1
                        if (read == 0) {//说明第一次读取到-1
                            read = -1; //返回值说明此处读取到末尾
                        }
                        break;
                    }
                    read += cnt;
                    if (cnt > 0) continue; //read some more  //继续读取
                    if (cnt==0 && (read>0 || (!block) ) ) break; //we are done reading  非阻塞或者此次读取为0并且已经不是第一次跳出
                }
                if ( selector != null ) {//perform a blocking read    不使用SHARED的阻塞方式,超时逻辑和write相同
                    //register OP_WRITE to the selector
                    if (key==null) key = socket.getIOChannel().register(selector, SelectionKey.OP_READ);
                    else key.interestOps(SelectionKey.OP_READ);
                    if (readTimeout==0) {
                        timedout = (read==0);
                    } else if (readTimeout<0) {
                        keycount = selector.select();
                    } else {
                        keycount = selector.select(readTimeout);
                    }
                }
                if (readTimeout > 0 && (selector == null || keycount == 0) ) timedout = (System.currentTimeMillis()-time)>=readTimeout;
            }//while
            if ( timedout ) throw new SocketTimeoutException();
        } finally {
            if (key != null) {
                key.cancel();
                if (selector != null) selector.selectNow();//removes the key from this selector
            }
        }
        return read;
    }
}
{%endcodeblock%}

#### processor 和adaptor
##### Http11Processor 一般处理http请求的process
- 继承图
![](http://pmftd1xvt.bkt.clouddn.com/Http11Processor.png)
{%codeblock lang:java Http11Processor::构造器%}
public Http11Processor(AbstractHttp11Protocol<?> protocol)
    {
        super();
        this.protocol = protocol;

        userDataHelper = new UserDataHelper(log);

        inputBuffer = new Http11InputBuffer(request, protocol.getMaxHttpHeaderSize());
        request.setInputBuffer(inputBuffer);

        outputBuffer = new Http11OutputBuffer(response, protocol.getMaxHttpHeaderSize());
        response.setOutputBuffer(outputBuffer);
         //... 设置in/outBuffer
    }
public AbstractProcessor() {
        this(new Request(), new Response()); //实例化req和res对象
    }	
{%endcodeblock%}
- 关于in/outBuffer
![](http://pmftd1xvt.bkt.clouddn.com/Http11InputBuffer.png)
#### mapper
- mapper 的构造初始填充
mapper 创建于service,连接使用都是公用的mapper,填充由mapperListener导致,发生在content构造之后
首先理解关于路由的过程 tomcat将url 分为三部分 如: localhost:8080/demo/index.hmlt
localhost 为host
demo 为content
index.html Wrapper
tomcat 按照多级映射 mapper包含host[] ->context[] ->wrapper[]
{%codeblock lang:java Mapper%}
//原型
 // ------------------------------------------------- MapElement Inner Class


    protected abstract static class MapElement<T> {

        public final String name;
        public final T object;  //这个T就是应该存放的对象

        public MapElement(String name, T object) {
            this.name = name;
            this.object = object;
        }
    }
	
	 protected static final class MappedHost extends MapElement<Host> {  //host映射,它持有ContextList 只不过是对数组的封装

        public volatile ContextList contextList;

        /**
         * Link to the "real" MappedHost, shared by all aliases.
         */
        private final MappedHost realHost;

        /**
         * Links to all registered aliases, for easy enumeration. This field
         * is available only in the "real" MappedHost. In an alias this field
         * is <code>null</code>.
         */
        private final List<MappedHost> aliases;

        /**
         * Constructor used for the primary Host
         *
         * @param name The name of the virtual host
         * @param host The host
         */
        public MappedHost(String name, Host host) {
            super(name, host);
            realHost = this;
            contextList = new ContextList();
            aliases = new CopyOnWriteArrayList<>();
        }

		
		
		
	protected static final class ContextList {  ContextList实现

        public final MappedContext[] contexts;
        public final int nesting;

        public ContextList() {
            this(new MappedContext[0], 0);
        }

        private ContextList(MappedContext[] contexts, int nesting) {
            this.contexts = contexts;
            this.nesting = nesting;
        }

        public ContextList addContext(MappedContext mappedContext,
                int slashCount) {
            MappedContext[] newContexts = new MappedContext[contexts.length + 1];
            if (insertMap(contexts, newContexts, mappedContext)) {
                return new ContextList(newContexts, Math.max(nesting,
                        slashCount));
            }
            return null;
        }

        public ContextList removeContext(String path) {
            MappedContext[] newContexts = new MappedContext[contexts.length - 1];
            if (removeMap(contexts, newContexts, path)) {
                int newNesting = 0;
                for (MappedContext context : newContexts) {
                    newNesting = Math.max(newNesting, slashCount(context.name));
                }
                return new ContextList(newContexts, newNesting);
            }
            return null;
        }
    }
	
	 protected static final class MappedContext extends MapElement<Void> { //context映射,内部含有ContextVersion对象,该对象是对MappedWrapper的封装
        public volatile ContextVersion[] versions;

        public MappedContext(String name, ContextVersion firstVersion) {
            super(name, null);
            this.versions = new ContextVersion[] { firstVersion };
        }
    }
	
	 protected static final class ContextVersion extends MapElement<Context> {
        public final String path;
        public final int slashCount;
        public final WebResourceRoot resources;
        public String[] welcomeResources;
        public MappedWrapper defaultWrapper = null;
        public MappedWrapper[] exactWrappers = new MappedWrapper[0];
        public MappedWrapper[] wildcardWrappers = new MappedWrapper[0];
        public MappedWrapper[] extensionWrappers = new MappedWrapper[0];
        public int nesting = 0;
        private volatile boolean paused;

        public ContextVersion(String version, String path, int slashCount,
                Context context, WebResourceRoot resources,
                String[] welcomeResources) {
            super(version, context);
            this.path = path;
            this.slashCount = slashCount;
            this.resources = resources;
            this.welcomeResources = welcomeResources;
        }

        public boolean isPaused() {
            return paused;
        }

        public void markPaused() {
            paused = true;
        }
    }
	
	 protected static class MappedWrapper extends MapElement<Wrapper> { //wrapper是最小容器

        public final boolean jspWildCard;
        public final boolean resourceOnly;

        public MappedWrapper(String name, Wrapper wrapper, boolean jspWildCard,
                boolean resourceOnly) {
            super(name, wrapper);
            this.jspWildCard = jspWildCard;
            this.resourceOnly = resourceOnly;
        }
    }
------------------------------------------------------------------------------------------------------------------
mapper 通过url 映射到正确的Wrap过程

private final void internalMap(CharChunk host, CharChunk uri,
            String version, MappingData mappingData) throws IOException {
        //mapper中查找使用了二分法
        //[0]: 确保host不为空
        if (mappingData.host != null) {
            // The legacy code (dating down at least to Tomcat 4.1) just
            // skipped all mapping work in this case. That behaviour has a risk
            // of returning an inconsistent result.
            // I do not see a valid use case for it.
            throw new AssertionError();
        }

        uri.setLimit(-1);

        // Virtual host mapping [1]: 虚拟主机映射
        MappedHost[] hosts = this.hosts;
        MappedHost mappedHost = exactFindIgnoreCase(hosts, host); //根据host寻找
        if (mappedHost == null) {
            // Note: Internally, the Mapper does not use the leading * on a
            //       wildcard host. This is to allow this shortcut.
            int firstDot = host.indexOf('.');
            if (firstDot > -1) {
                int offset = host.getOffset();
                try {
                    host.setOffset(firstDot + offset);
                    mappedHost = exactFindIgnoreCase(hosts, host);
                } finally {
                    // Make absolutely sure this gets reset
                    host.setOffset(offset);
                }
            }
            if (mappedHost == null) {
                mappedHost = defaultHost;
                if (mappedHost == null) {
                    return;
                }
            }
        }
        mappingData.host = mappedHost.object;   //设置host,该host就是StandardHost

        // Context mapping [2]:获取context映射
        ContextList contextList = mappedHost.contextList; // 获取对应host中的contextList映射
        MappedContext[] contexts = contextList.contexts;  // 该host对应的所有contextMapper 数组
        int pos = find(contexts, uri);  //根据uri 寻找
        if (pos == -1) {
            return;
        }

        int lastSlash = -1;
        int uriEnd = uri.getEnd();
        int length = -1;
        boolean found = false;
        MappedContext context = null;
        while (pos >= 0) {
            context = contexts[pos];
            if (uri.startsWith(context.name)) {
                length = context.name.length();
                if (uri.getLength() == length) {
                    found = true;
                    break;
                } else if (uri.startsWithIgnoreCase("/", length)) {
                    found = true;
                    break;
                }
            }
            if (lastSlash == -1) {
                lastSlash = nthSlash(uri, contextList.nesting + 1);
            } else {
                lastSlash = lastSlash(uri);
            }
            uri.setEnd(lastSlash);
            pos = find(contexts, uri); //根据uri寻找对应的contentMap
        }
        uri.setEnd(uriEnd);

        if (!found) {
            if (contexts[0].name.equals("")) {
                context = contexts[0];
            } else {
                context = null;
            }
        }
        if (context == null) {
            return;
        }
        // 找到了context
        mappingData.contextPath.setString(context.name); //设置根据context name设置
        //[3]:获取版本映射
        ContextVersion contextVersion = null;  //[]要找到的
        ContextVersion[] contextVersions = context.versions;  //获取关于warp的映射数组,tomcat9 貌似对于一个context做了一个版本区别,按理此处不该是数组,
        //明显是通过不同数组代表不同的版本,默认使用的话应该是1
        final int versionCount = contextVersions.length;
        if (versionCount > 1) {
            Context[] contextObjects = new Context[contextVersions.length];
            for (int i = 0; i < contextObjects.length; i++) {
                contextObjects[i] = contextVersions[i].object;
            }
            mappingData.contexts = contextObjects;
            if (version != null) {
                contextVersion = exactFind(contextVersions, version);
            }
        }
        if (contextVersion == null) {
            // Return the latest version
            // The versions array is known to contain at least one element
            contextVersion = contextVersions[versionCount - 1];
        }
        mappingData.context = contextVersion.object; //设置context
        mappingData.contextSlashCount = contextVersion.slashCount;

        // Wrapper mapping [4]:按照正确的版本寻找对应的 Wrapper
        if (!contextVersion.isPaused()) {
            internalMapWrapper(contextVersion, uri, mappingData);
        }
    }
	
{%endcodeblock%}
#### tomcat中jdni
为了容易理解,我在各部分尽量加上uml或能够说明的图,我也发现单纯看代码比较麻烦
- jdni的实现逻辑:以name路径,找到对应的对象;以路径存储对象;存储类型分为直接对象,和对象信息,后者在取出时按照信息进行实例化
{%asset_img JDNI1.png%}
spi 则是jdni具体实现,所有操作都要围绕NamingManager操作,用户入口接口一般时initialContext
- jdni模型
{%asset_img JDNI2.png%}
jdni的组织结构是一个树结构,Context类型表示上下文,资源节点必须存在于context节点中,一个Context存在任意子节点,节点类型可以为Context或者资源,任意节点都有name,充当路径,根节点一般为""
如: tomcat中  lookup("java:UserDatabase") 表示以根节点""开始寻找UserDatabase表示的资源
- jdni相关类
NamingManager 属于一个静态类,是jdni的核心,基本操作都是调用该类进行中转发起的
ObjectFacotry 寻找对象后创建对象的类,但并不代表实际创建对象的操作是该类处理的,在org.apache.naming包下实现了部分子类,用以处理不同的jndi类型,该工厂决定了对象按照Refence如何创建
initialContext 用于用户接口 lookup 和bind 函数分别用于获取和存储对象
StateFactory 用于绑定时如何处理的工厂,tomcat貌似没有实现这种子类
{%codeblock lang:java JNDI%}

public interface Context {  //该接口是jdk上下文接口,这里只提及一个常量
//jndi 使用hashTable作为供namingManager使用的环境变量 
public Hashtable<?,?> getEnvironment() throws NamingException;
String INITIAL_CONTEXT_FACTORY = "java.naming.factory.initial";  //key,value 此处value表示一个factory的类位置,供namingManager初始化 initFactory使用,此处tomcat也使用了该变量,用于指定initFactory,放到了system.pero中
}
-------------------------------------------------------------------------------------------------

//用户接口
public class InitialContext implements Context {

  public InitialContext() throws NamingException {
        init(null);
    }
	
	 protected void init(Hashtable<?,?> environment)
        throws NamingException
    {
        myProps = (Hashtable<Object,Object>)
                ResourceManager.getInitialEnvironment(environment); //此处代码最终会在system.pero中寻找和Context中常量匹配的key,value 当作env

        if (myProps.get(Context.INITIAL_CONTEXT_FACTORY) != null) { //如果设置了,则进行初始化context
            // user has specified initial context factory; try to get it
            getDefaultInitCtx();
        }
    }
	
	protected Context getDefaultInitCtx() throws NamingException{
        if (!gotDefault) {
            defaultInitCtx = NamingManager.getInitialContext(myProps); //NamingManager通过env获得initFactory,然后通过Fac创建出initContext
            gotDefault = true;
        }
        if (defaultInitCtx == null)
            throw new NoInitialContextException();

        return defaultInitCtx;
    }
	//lookup和bind
	public Object lookup(Name name) throws NamingException {
        return getURLOrDefaultInitCtx(name).lookup(name);
    }
	public void bind(Name name, Object obj) throws NamingException {
        getURLOrDefaultInitCtx(name).bind(name, obj);
    }
	
	 protected Context getURLOrDefaultInitCtx(Name name)
        throws NamingException {
        if (NamingManager.hasInitialContextFactoryBuilder()) {  //当设置FactoryBuilder,则通过builder获取fac,再获取initContext
            return getDefaultInitCtx();
        }
        if (name.size() > 0) {
            String first = name.get(0);
            String scheme = getURLScheme(first);
            if (scheme != null) {
                Context ctx = NamingManager.getURLContext(scheme, myProps); //否则说明要通过urlFactory来获取initContext,tomcat就采取的这种方式,实际上tomcat也创建了initContext,但是它没有用,而非要使用url路径来返回一个新的context
                if (ctx != null) {
                    return ctx;
                }
            }
        }
        return getDefaultInitCtx();
    }

-------------------------------------------------------------------------------------------------------------------------------	
	
	//核心NamingManager
	public class NamingManager{
	//获取initContext的方式
	public static Context getInitialContext(Hashtable<?,?> env) throws NamingException {
        InitialContextFactory factory = null;

        InitialContextFactoryBuilder builder = getInitialContextFactoryBuilder(); 尝试通过builder获取
        if (builder == null) {
            // No builder installed, use property
            // Get initial context factory class name

            String className = env != null ?
                (String)env.get(Context.INITIAL_CONTEXT_FACTORY) : null;
            if (className == null) {
                NoInitialContextException ne = new NoInitialContextException(
                    "Need to specify class name in environment or system " +
                    "property, or in an application resource file: " +
                    Context.INITIAL_CONTEXT_FACTORY);
                throw ne;
            }

            ServiceLoader<InitialContextFactory> loader =
                    ServiceLoader.load(InitialContextFactory.class);

            Iterator<InitialContextFactory> iterator = loader.iterator();
            try {
                while (iterator.hasNext()) {
                    InitialContextFactory f = iterator.next();
                    if (f.getClass().getName().equals(className)) {
                        factory = f;
                        break;
                    }
                }
            } catch (ServiceConfigurationError e) {
                NoInitialContextException ne =
                        new NoInitialContextException(
                                "Cannot load initial context factory "
                                        + "'" + className + "'");
                ne.setRootCause(e);
                throw ne;
            }

            if (factory == null) {
                try {
                    @SuppressWarnings("deprecation")
                    Object o = helper.loadClass(className).newInstance();  //通过INITIAL_CONTEXT_FACTORY 对应的class位置,反射出factory
                    factory = (InitialContextFactory) o;
                } catch (Exception e) {
                    NoInitialContextException ne =
                            new NoInitialContextException(
                                    "Cannot instantiate class: " + className);
                    ne.setRootCause(e);
                    throw ne;
                }
            }
        } else {
            factory = builder.createInitialContextFactory(env);
        }

        return factory.getInitialContext(env); //通过factory创建initContext
    }
	
	    //试图使用url来创建
		 public static Context getURLContext(String scheme,
                                        Hashtable<?,?> environment)  throws NamingException
    {
        // pass in 'null' to indicate creation of generic context for scheme
        // (i.e. not specific to a URL).

            Object answer = getURLObject(scheme, null, null, null, environment);
            if (answer instanceof Context) {
                return (Context)answer;
            } else {
                return null;
            }
    }
	
	private static Object getURLObject(String scheme, Object urlInfo,
                                       Name name, Context nameCtx,
                                       Hashtable<?,?> environment)
            throws NamingException {

        // e.g. "ftpURLContextFactory"
        ObjectFactory factory = (ObjectFactory)ResourceManager.getFactory(
            Context.URL_PKG_PREFIXES, environment, nameCtx,
            "." + scheme + "." + scheme + "URLContextFactory", defaultPkgPrefix);  //创建url工厂

        if (factory == null)
          return null;

        // Found object factory
        try {
            return factory.getObjectInstance(urlInfo, name, nameCtx, environment);  //通过url工厂返回context
        } catch (NamingException e) {
            throw e;
        } catch (Exception e) {
            NamingException ne = new NamingException();
            ne.setRootCause(e);
            throw ne;
        }
	
    //实例化对象的接口
	public static Object getObjectInstance(Object refInfo, Name name, Context nameCtx,
                          Hashtable<?,?> environment) throws Exception {
        ObjectFactory factory;

        // Use builder if installed
        ObjectFactoryBuilder builder = getObjectFactoryBuilder(); //有builder使用builder创建fac
        if (builder != null) {
            // builder must return non-null factory
            factory = builder.createObjectFactory(refInfo, environment);
            return factory.getObjectInstance(refInfo, name, nameCtx,
                environment);
        }

        // Use reference if possible  否则使用Reference中的factoryClass来创建工厂
        Reference ref = null;
        if (refInfo instanceof Reference) {
            ref = (Reference) refInfo;
        } else if (refInfo instanceof Referenceable) {
            ref = ((Referenceable)(refInfo)).getReference();
        }

        Object answer;

        if (ref != null) {
            String f = ref.getFactoryClassName();
            if (f != null) {
                // if reference identifies a factory, use exclusively

                factory = getObjectFactoryFromReference(ref, f); //获取工厂
                if (factory != null) {
                    return factory.getObjectInstance(ref, name, nameCtx, //通过工厂创建对象
                                                     environment);
                }
                // No factory found, so return original refInfo.
                // Will reach this point if factory class is not in
                // class path and reference does not contain a URL for it
                return refInfo;

            } else {
                // if reference has no factory, check for addresses
                // containing URLs

                answer = processURLAddrs(ref, name, nameCtx, environment);
                if (answer != null) {
                    return answer;
                }
            }
        }

        // try using any specified factories
        answer =
            createObjectFromFactories(refInfo, name, nameCtx, environment);
        return (answer != null) ? answer : refInfo;
    }	
}
-------------------------------------------------------------------------------------------------------------------------------

// 大概流程:
  initialContext创建->根据env创建initContext->lookup->NamingManager判断并返回一个Context->由该context进行lookup->根据此类型信息中的factory进行创建
{%endcodeblock%}
- tomcat
{%codeblock lang:java tomcat实现%}
//该context作为节点存放资源和context
public class NamingContext implements Context
{
protected final Hashtable<String, Object> env; //环境变量

protected final HashMap<String, NamingEntry> bindings;//实际上绑定的资源就在此处,实际上绑定较为简单,lookup存在创建对象的复杂性

 protected Object lookup(Name name, boolean resolveLinks) throws NamingException
    {

        // Removing empty parts
        while ((!name.isEmpty()) && (name.get(0).length() == 0)) name = name.getSuffix(1);
        if (name.isEmpty())
        {
            // If name is empty, a newly allocated naming context is returned
            return new NamingContext(env, this.name, bindings);
        }

        NamingEntry entry = bindings.get(name.get(0));

        if (entry == null)
        {
            throw new NameNotFoundException(sm.getString("namingContext.nameNotBound", name, name.get(0)));
        }

        if (name.size() > 1)  //这说明寻找的位置必定是 /xx/xxx/xxxx 这种,说明存在子context,此时的this是作为当前根节点,
        {
            // If the size of the name is greater that 1, then we go through a
            // number of subcontexts.
            if (entry.type != NamingEntry.CONTEXT)
            {
                throw new NamingException(sm.getString("namingContext.contextExpected"));
            }
            return ((Context) entry.value).lookup(name.getSuffix(1)); //从相对位置1开始继续寻找
        }
        else  //否则说明该context节点就是包含资源的context,从此处map寻找就可以
        {
            if ((resolveLinks) && (entry.type == NamingEntry.LINK_REF)) //这是tomcat为了实现局部和全局资源而做的,在局部context(指的是tomcat context)ref就是这么实现的
            {
                String link = ((LinkRef) entry.value).getLinkName();
                if (link.startsWith("."))
                {
                    // Link relative to this context
                    return lookup(link.substring(1));
                }
                else
                {
                    return new InitialContext(env).lookup(link);
                }
            }
            else if (entry.type == NamingEntry.REFERENCE)  //REFERENCE 就是jndi推荐存放的数据类型
            {
                try
                {
                    Object obj = NamingManager.getObjectInstance(entry.value, name, this, env);  //按照jndi的流程进行创建
                    if (entry.value instanceof ResourceRef) //ResourceRef类型存放着大量信息
                    {
                        boolean singleton = Boolean.parseBoolean((String) ((ResourceRef) entry.value).get("singleton").getContent());
                        if (singleton) //假设是单例模式,就改变entry信息
                        {
                            entry.type = NamingEntry.ENTRY;
                            entry.value = obj;
                        }
                    }
                    return obj;
                }
                catch (NamingException e)
                {
                    throw e;
                }
                catch (Exception e)
                {
                    log.warn(sm.getString("namingContext.failResolvingReference"), e);
                    throw new NamingException(e.getMessage());
                }
            }
            else
            {
                return entry.value; //这中情况说明是单例,上次已经创建了一次,这一次返回就好了
            }
        }

    }
	    protected void bind(Name name, Object obj, boolean rebind) throws NamingException
    {

        if (!checkWritable())
        {
            return;
        }

        while ((!name.isEmpty()) && (name.get(0).length() == 0)) name = name.getSuffix(1);
        if (name.isEmpty())
        {
            throw new NamingException(sm.getString("namingContext.invalidName"));
        }

        NamingEntry entry = bindings.get(name.get(0));  //获取当前路径的当前根节点

        if (name.size() > 1)//说明当前路径为 /xx/xxx/xxxx..
        {
            if (entry == null)  //说明xx按理说是一个context,但是对于this来说并没有把它当作子context存放,也就是说tomcat的NamingContext.bind方法不支持直接创建子context
            {
                throw new NameNotFoundException(sm.getString("namingContext.nameNotBound", name, name.get(0)));
            }
            if (entry.type == NamingEntry.CONTEXT)
            {
                if (rebind)
                {
                    ((Context) entry.value).rebind(name.getSuffix(1), obj);  //重新调用bind函数,实际就是树结构的向下节点查询
                }
                else
                {
                    ((Context) entry.value).bind(name.getSuffix(1), obj);
                }
            }
            else
            {
                throw new NamingException(sm.getString("namingContext.contextExpected"));
            }
        }
        else //说明到了最后一层context节点
        {
            if ((!rebind) && (entry != null)) //说明this节点存放着name对应的数据,但是并不进行rebind,因此alreadyBound异常
            {
                throw new NameAlreadyBoundException(sm.getString("namingContext.alreadyBound", name.get(0)));
            }
            else
            {
                // Getting the type of the object and wrapping it within a new
                // NamingEntry
                Object toBind = NamingManager.getStateToBind(obj, name, this, env);  //调用状态工厂进行处理,但是tomcat并没有实现状态工厂,而是每次存储都是符合逻辑的数据
                if (toBind instanceof Context)
                {
                    entry = new NamingEntry(name.get(0), toBind, NamingEntry.CONTEXT);
                }
                else if (toBind instanceof LinkRef)
                {
                    entry = new NamingEntry(name.get(0), toBind, NamingEntry.LINK_REF);
                }
                else if (toBind instanceof Reference)
                {
                    entry = new NamingEntry(name.get(0), toBind, NamingEntry.REFERENCE);
                }
                else if (toBind instanceof Referenceable)
                {
                    toBind = ((Referenceable) toBind).getReference();
                    entry = new NamingEntry(name.get(0), toBind, NamingEntry.REFERENCE);
                }
                else
                {
                    entry = new NamingEntry(name.get(0), toBind, NamingEntry.ENTRY);
                }
                bindings.put(name.get(0), entry); // 真正进行的绑定操作
            }
        }

    }
]
---------------------------------------------------------------------------------------------------------------------
//tomcat实现的url工厂,用于返回context来进行look操作和bind操作,实际上该类只返回selectorContext
public class javaURLContextFactory
    implements ObjectFactory, InitialContextFactory {
	   public Object getObjectInstance(Object obj, Name name, Context nameCtx,
                                    Hashtable<?,?> environment)
        throws NamingException {
        if ((ContextBindings.isThreadBound()) ||  //判断当前线程或者类加载器是否被ContextBindings存储,这是实现隔离机制的方式
            (ContextBindings.isClassLoaderBound())) {
            return new SelectorContext((Hashtable<String,Object>)environment);
        }
        return null;
    }


    /**
     * Get a new (writable) initial context.
     */
    @SuppressWarnings("unchecked")
    @Override
    public Context getInitialContext(Hashtable<?,?> environment)
        throws NamingException {
        if (ContextBindings.isThreadBound() ||
            (ContextBindings.isClassLoaderBound())) { //当前线程满足条件,返回SelectorContext
            // Redirect the request to the bound initial context 
            return new SelectorContext(
                    (Hashtable<String,Object>)environment, true);
        }

        // If the thread is not bound, return a shared writable context
        if (initialContext == null) { //如果不满足,那么就会以initialContext做为上下文的根处理,并且是会被共享的
            synchronized(javaURLContextFactory.class) {
                if (initialContext == null) {
                    initialContext = new NamingContext(
                            (Hashtable<String,Object>)environment, MAIN);
                }
            }
        }
        return initialContext;  //对于没有的被绑定的线程,都会使用该context作为起始点
    }
}
-------------------------------------------------------------------------------------------------------------------
//该类也是static类,使用了大量的map进行保持线程,类加载器:context的映射,Server和Context就是通过类加载器区分的context,从而实现了全局和局部资源
//绑定的过程发生在NamingContextListener.lifeEvnet过程中
public class ContextBindings {

    // -------------------------------------------------------------- Variables

    /**
     * Bindings object - naming context. Keyed by object.
     */
    private static final Hashtable<Object,Context> objectBindings = new Hashtable<>();


    /**
     * Bindings thread - naming context. Keyed by thread.
     */
    private static final Hashtable<Thread,Context> threadBindings = new Hashtable<>();


    /**
     * Bindings thread - object. Keyed by thread.
     */
    private static final Hashtable<Thread,Object> threadObjectBindings = new Hashtable<>();


    /**
     * Bindings class loader - naming context. Keyed by class loader.
     */
    private static final Hashtable<ClassLoader,Context> clBindings = new Hashtable<>();


    /**
     * Bindings class loader - object. Keyed by class loader.
     */
    private static final Hashtable<ClassLoader,Object> clObjectBindings = new Hashtable<>();
---------------------------------------------------------------------------------------------------------------------
public class SelectorContext implements Context {
 public Object lookup(Name name)
        throws NamingException {

        if (log.isDebugEnabled()) {
            log.debug(sm.getString("selectorContext.methodUsingName", "lookup",
                    name));
        }

        // Strip the URL header
        // Find the appropriate NamingContext according to the current bindings
        // Execute the lookup on that context
        return getBoundContext().lookup(parseName(name)); //通过ContextBindings获取对应范围的NamingContext,再使用context.lookup 取得实际对象
    }
	
 public void bind(Name name, Object obj) //同理
        throws NamingException {
        getBoundContext().bind(parseName(name), obj);
    }	
}
--------------------------------------------------------------------------------------------------------------------
//tomcat中实现的一部分object工厂源码
	/**
 * <p>Object factory for resource links.</p>  处理ResouceLink的关键
 *
 * @author Remy Maucherat
 */
public class ResourceLinkFactory implements ObjectFactory {
private static Context globalContext = null; //该类持有全局context的引用
public Object getObjectInstance(Object obj, Name name, Context nameCtx,
            Hashtable<?,?> environment) throws NamingException {

        if (!(obj instanceof ResourceLinkRef)) {
            return null;
        }

        // Can we process this request?
        Reference ref = (Reference) obj;

        // Read the global ref addr
        String globalName = null;
        RefAddr refAddr = ref.get(ResourceLinkRef.GLOBALNAME);  //获取global标签到全局context.lookUp就能获取到全局资源
        if (refAddr != null) {
            globalName = refAddr.getContent().toString();
            // Confirm that the current web application is currently configured
            // to access the specified global resource
            if (!validateGlobalResourceAccess(globalName)) {
                return null;
            }
            Object result = null;
            result = globalContext.lookup(globalName);
            // Check the expected type
            String expectedClassName = ref.getClassName();
            if (expectedClassName == null) {
                throw new IllegalArgumentException(
                        sm.getString("resourceLinkFactory.nullType", name, globalName));
            }
            try {
                Class<?> expectedClazz = Class.forName(
                        expectedClassName, true, Thread.currentThread().getContextClassLoader());
                if (!expectedClazz.isAssignableFrom(result.getClass())) {
                    throw new IllegalArgumentException(sm.getString("resourceLinkFactory.wrongType",
                            name, globalName, expectedClassName, result.getClass().getName()));
                }
            } catch (ClassNotFoundException e) {
                throw new IllegalArgumentException(sm.getString("resourceLinkFactory.unknownType",
                        name, globalName, expectedClassName), e);
            }
            return result;
        }

        return null;
    }
}	
--------------------------------------------------------------------------------------------------------------------
/**
 * <p>Object factory for resource links for shared data sources.</p>
 * 用来处理: javax.sql.DataSource数据类型,我这里是一个错误例子 ,正确的server.xml例子见后文
				  <GlobalNamingResources>
						<!-- Editable user database that can also be used by
							 UserDatabaseRealm to authenticate users
						-->
						<Resource name="UserDatabase" auth="Container"
								  type="org.apache.catalina.UserDatabase"  //此处是UserDatabase
								  description="User database that can be updated and saved"
								  factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
								  pathname="conf/tomcat-users.xml"/>
					</GlobalNamingResources>
			
					<ResourceLink
                            name="UserData"
                            global="UserDatabase"
                            type="org.apache.catalina.UserDatabase" 
                            factory="org.apache.naming.factory.DataSourceLinkFactory">
                    </ResourceLink>
	该工厂创建的对象必须要有userName 和password两个属性,否则错误				
 */
public class DataSourceLinkFactory extends ResourceLinkFactory {
	 public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?,?> environment) throws NamingException { //该函数由selector.lookup传递过来
        Object result = super.getObjectInstance(obj, name, nameCtx, environment); //获取全局资源
        // Can we process this request?
        if (result!=null) {
            Reference ref = (Reference) obj;
            RefAddr userAttr = ref.get("username"); //根据userName和password标签处理
            RefAddr passAttr = ref.get("password"); 
            if (userAttr.getContent()!=null && passAttr.getContent()!=null) { //很明显,当userName和password两个标签为null,此处异常
                result = wrapDataSource(result,userAttr.getContent().toString(), passAttr.getContent().toString());
            }
        }
        return result;
    }
	//这里使用了proxy,以前写自定义数据源的时候用过,主要是DataSourceHandler.invoke的实现
	 protected Object wrapDataSource(Object datasource, String username, String password) throws NamingException {
        try {
            Class<?> proxyClass = Proxy.getProxyClass(datasource.getClass().getClassLoader(), datasource.getClass().getInterfaces());
            Constructor<?> proxyConstructor = proxyClass.getConstructor(new Class[] { InvocationHandler.class });
            DataSourceHandler handler = new DataSourceHandler((DataSource)datasource, username, password);
            return proxyConstructor.newInstance(handler);
        }catch (Exception x) {
            if (x instanceof InvocationTargetException) {
                Throwable cause = x.getCause();
                if (cause instanceof ThreadDeath) {
                    throw (ThreadDeath) cause;
                }
                if (cause instanceof VirtualMachineError) {
                    throw (VirtualMachineError) cause;
                }
                if (cause instanceof Exception) {
                    x = (Exception) cause;
                }
            }
            if (x instanceof NamingException) throw (NamingException)x;
            else {
                NamingException nx = new NamingException(x.getMessage());
                nx.initCause(x);
                throw nx;
            }
        }
    }
	//扩展:
	public static class DataSourceHandler implements InvocationHandler {
	  private final DataSource ds;
        private final String username;
        private final String password;
        private final Method getConnection;
        public DataSourceHandler(DataSource ds, String username, String password) throws Exception {
            this.ds = ds;
            this.username = username;
            this.password = password;
            getConnection = ds.getClass().getMethod("getConnection", String.class, String.class);
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

            if ("getConnection".equals(method.getName()) && (args==null || args.length==0)) { //当调用getConection
                args = new String[] {username,password};
                method = getConnection; //将设置为ds的getConnection(user,word)这个函数 
            } else if ("unwrap".equals(method.getName())) {
                return unwrap((Class<?>)args[0]);
            }
            try {
                return method.invoke(ds,args); //调用代理函数
            }catch (Throwable t) {
                if (t instanceof InvocationTargetException
                        && t.getCause() != null) {
                    throw t.getCause();
                } else {
                    throw t;
                }
            }
        }
	}
------------------------------------------------------------------
//其他工厂等用到的时候再看

{%endcodeblock%}
- tomcat9 关于context的naming资源在web.xml中jndi的解析
在tomcat web.xml配置的信息和在Server.xml 中<Context> 配置的资源一样都是属于局部Context的,注意Tomcat默认给的Server.xml中UserData的type和实际使用的类型不一样,因此默认的那么创建要通过factory这个属性
在context中的资源必须通过 "java:comp/env/..."的路径进行寻找,因为在NamingContextListener处理context类型的时候创建了了两个子上下文
{%codeblock lang:xml web.xml%}
//1.解析web.xml的过程是contextConfig监听器做的,并且发生在context.startInternal,该life监听器激活的顺序大于context.NamingContextListener
//2.当解析web.xml会构建一个WebXml类,存放了关于所有web.xml的信息,在contextConfig某一步过程中将解析到到的关于naming标签创建的Resouce子类加入到context.namingResource中,并随着namingContextListener的激活,注册到对应的jndi中
//3.web.xml解析的resource就属于局部jdni并不是引用,不过应该可以解析引用的
<web-app>

</web-app>
{%endcodeblock%}

{%codeblock lang:java contextConfig%}
//该监听器进行context加载时复杂的处理,这里截取关于naming的部分

//1. 构建WebXml 对象,使用degister,并且栈顶第一个元素就是WebXml
public class ContextConfig implements LifecycleListener
protected void webConfig()
    {
		WebXmlParser webXmlParser = new WebXmlParser(context.getXmlNamespaceAware(), context.getXmlValidation(), context.getXmlBlockExternal());

        Set<WebXml> defaults = new HashSet<>();
        defaults.add(getDefaultWebXmlFragment(webXmlParser));

        Set<WebXml> tomcatWebXml = new HashSet<>();
        tomcatWebXml.add(getTomcatWebXmlFragment(webXmlParser));

        WebXml webXml = createWebXml();  //创建WebXml对象
		   // Parse context level web.xml
        InputSource contextWebXml = getContextWebXmlSource();
        if (!webXmlParser.parseWebXml(contextWebXml, webXml, false)) //解析,这里将WebXml push进了栈顶
        {
            ok = false;
        }
		.............
		// Step 9. Apply merged web.xml to Context
            if (ok)
            {
                configureContext(webXml); //将webXml中关于naming的资源加入到对应context的namingResource中
            }
	}
	
	//处理解析出来的WebXml中数据
	
	private void configureContext(WebXml webxml){  //此处不仅仅处理naming资源
	 //解析web.xml中配置的naming资源,并加入到context的namingResources中
        for (Entry<String, String> entry : webxml.getContextParams().entrySet())
        {
            context.addParameter(entry.getKey(), entry.getValue());
        }
        context.setDenyUncoveredHttpMethods(webxml.getDenyUncoveredHttpMethods());
        context.setDisplayName(webxml.getDisplayName());
        context.setDistributable(webxml.isDistributable());
		//总之将这些资源添加到context.namingResource中,并且因为这个监听器触发的比NamingContextListener早,因此是一次性bind的
        for (ContextLocalEjb ejbLocalRef : webxml.getEjbLocalRefs().values())
        {
            context.getNamingResources().addLocalEjb(ejbLocalRef);
        }
        for (ContextEjb ejbRef : webxml.getEjbRefs().values())
        {
            context.getNamingResources().addEjb(ejbRef);
        }
        for (ContextEnvironment environment : webxml.getEnvEntries().values())
        {
            context.getNamingResources().addEnvironment(environment);
        }
		....
		for (MessageDestinationRef mdr : webxml.getMessageDestinationRefs().values())
        {
            context.getNamingResources().addMessageDestinationRef(mdr);
        }
		 // Name is just used for ordering
        for (ContextResourceEnvRef resource : webxml.getResourceEnvRefs().values())
        {
            context.getNamingResources().addResourceEnvRef(resource);
        }
        for (ContextResource resource : webxml.getResourceRefs().values())
        {
            context.getNamingResources().addResource(resource);
        }
		....
		 for (ContextService service : webxml.getServiceRefs().values())
        {
            context.getNamingResources().addService(service);
        }
	}
	
}	
// webxml解析规则部分
public class WebRuleSet implements RuleSet {
protected void configureNamingRules(Digester digester) { //关于解析naming的部分
 //resource-ref
        digester.addObjectCreate(fullPrefix + "/resource-ref",
                                 "org.apache.tomcat.util.descriptor.web.ContextResource");  //明显是创建的ContextResource不是ContextResourceLink类型,也就是说创建的和在Server.xml配置的情况类似
        digester.addSetNext(fullPrefix + "/resource-ref",  //添加到WebXml子类
                            "addResourceRef",
                            "org.apache.tomcat.util.descriptor.web.ContextResource");
        digester.addCallMethod(fullPrefix + "/resource-ref/description",  //设置属性
                               "setDescription", 0);
        digester.addCallMethod(fullPrefix + "/resource-ref/res-auth",
                               "setAuth", 0);
        digester.addCallMethod(fullPrefix + "/resource-ref/res-ref-name",
                               "setName", 0);
        digester.addCallMethod(fullPrefix + "/resource-ref/res-sharing-scope",
                               "setScope", 0);
        digester.addCallMethod(fullPrefix + "/resource-ref/res-type",
                               "setType", 0);
        digester.addRule(fullPrefix + "/resource-ref/mapped-name",  //这个规则实际上就是设置内部prop为 mapped-name:value 的键值对,在Server.xml解析过程中除了上述几个属性,多余的也是这么解析的
                         new MappedNameRule());
        configureInjectionRules(digester, "web-app/resource-ref/");
		............
		//结束之后在WebXml对象就有许多关于资源的key-value对
}	
{%endcodeblock%}
- 如何引用全局资源 https://tomcat.apache.org/tomcat-9.0-doc/config/context.html#Resource_Links
 1.目前来说我在源码HostConfig位置,tomcat是不会解析Meta-INF中context.xml多余的标签,因此局部引用方式我觉得在context.xml中写没有用
 2.在Server.xml <Context>标签配置<ResourceLink>
 3.ResourceLink标签解析在server.xml解析完成
 ```xml
 <GlobalNamingResources>
  ...
  <Resource name="sharedDataSource"
            global="sharedDataSource"
            type="javax.sql.DataSource"
            factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
            alternateUsernameAllowed="true"
            username="bar"
            password="barpass"
            ...
  ...
</GlobalNamingResources>

<Context path="/foo"...>
  ...
  <ResourceLink
            name="appDataSource"  //表示在context的ContextJdni中创建的名字
            global="sharedDataSource"  //引用的全局位置
            type="javax.sql.DataSource"
            factory="org.apache.naming.factory.DataSourceLinkFactory"  //注意如果填了此项,后边的两个也要填,否则异常
            username="foo"  
            password="foopass"
  ...
</Context>
<Context path="/bar"...>
  ...
  <ResourceLink
            name="appDataSource"
            global="sharedDataSource"
            type="javax.sql.DataSource"
  ...
</Context>
 ```
