---
title: spring内省
date: 2020-03-10 20:31:26
tag:
- 框架
- spring
- spring组件
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
### 概况
jdk提供了完善的内省机制,spring的`BeanWrapper`体系使用到了该部分,参考[beanwrapper](/2020/02/11/spring常见/#beanwrapper)分析
在jdk中将该部分api归类到了java.desktop中
#### jdk内省
##### BeanInfo
表示一个类内部的信息
{%codeblock lang:java BeanInfo%}
public interface BeanInfo {
BeanDescriptor getBeanDescriptor(); //描述类本身
EventSetDescriptor[] getEventSetDescriptors();//事件
PropertyDescriptor[] getPropertyDescriptors();//属性
MethodDescriptor[] getMethodDescriptors();//方法
Image getIcon(int iconKind);//图形开发使用的
}
{%endcodeblock%}
- 使用
```java
@Test
    public void BeanInfoTest() {
        BeanInfo beanInfo = null;
        try {
            beanInfo = Introspector.getBeanInfo(User.class);
        } catch (IntrospectionException e) {
            e.printStackTrace();
        }
        var pds = beanInfo.getPropertyDescriptors();
        var beanInfos = beanInfo.getAdditionalBeanInfo();
        var bd = beanInfo.getBeanDescriptor();
        var md =beanInfo.getMethodDescriptors();
    }
```
##### PropertyDescriptor
描述一个类中的属性
{%codeblock lang:java PropertyDescriptor%}
public class PropertyDescriptor extends FeatureDescriptor {
  //分别获取set|get函数
   public synchronized Method getWriteMethod();
    public synchronized Method getReadMethod()
    getPropertyType()//属性类型
}  
{%endcodeblock%}
- 使用
```java
@Test
 public void propertyDescriptorTest() {
     PropertyDescriptor propertyDescriptor = null;
     try {
         propertyDescriptor = new PropertyDescriptor("name", User.class);
     } catch (IntrospectionException e) {
         e.printStackTrace();
     }
     System.out.println(propertyDescriptor.getShortDescription());
 }
```
#### spring部分
##### BeanWrapper关于属性设置的实现
这个体系的实现类`BeanWrapperImpl` 持有真正对象`wrappedObject`,调用setPorperty,实际就给内部Object属性设置,存在类型转换
`AbstractNestablePropertyAccessor`表示可以对一个对象进行嵌套设置
- 例子
{%codeblock lang:java 例子%}
//注意要提供get|set函数
public class Company {
    private List<Department> departments;
    private String companyName;
    private Boss boss;
}    
public class Department {
    private String departmentName;
    private List<Employee> employees;
}
public class Employee {
    private String name;
}
public class Boss{
  private String name;
}
//使用
@Test
 public void test1() {
     BeanWrapperImpl beanWrapper = new BeanWrapperImpl(Company.class);
     //表示如果中间属性未设置的情况,spring会创建默认中间值
     beanWrapper.setAutoGrowNestedPaths(true);
     beanWrapper.setPropertyValue("companyName","co.LTD");
     beanWrapper.setPropertyValue("boss.name","xx");
     beanWrapper.setPropertyValue("departments[0].departmentName","testPart");
     beanWrapper.setPropertyValue("departments[0].employees[0].name","wang2gou");
     beanWrapper.setPropertyValue("departments[1]",new Department());
        Company
     //正确设置
     Company company = (Company) beanWrapper.getWrappedInstance();
 }        
{%endcodeblock%}
##### AbstractNestablePropertyAccessor原理
顾名思义该类表示属性可嵌套,如上述例子可知,对象A.b.c的这样的情况,实际在该类每一个节点都表现为一个ANPA,源码采用递归层级的实现.
简单来说即按照分割符`.`来确定属性层级,父层级缓存次层级,以此类推
```
 Company
        |                  ---departmentName   
        ----departments----|
        |                  ---employees  
        ----companyName
        |
        ----boss--
                 |
                 ---name
```
实际上每个属性都会被转换为ANPA
{%codeblock lang:java %}
public abstract class AbstractNestablePropertyAccessor extends AbstractPropertyAccessor {
  //当前层级的对象
  Object wrappedObject;

    private String nestedPath = "";
  //最外层对象
    @Nullable
    Object rootObject;

  //当前对象的缓存的次级别对象
    /** Map with cached nested Accessors: nested path -> Accessor instance. */
    @Nullable
    private Map<String, AbstractNestablePropertyAccessor> nestedPropertyAccessors;
}
{%endcodeblock%}
这里提到的属性可能是单一对象,或者数组,集合,下文用[]表示后者情况.
###### PropertyHandler
该类是spring实现了,用来描述属性性质的封装类,用来判断能否读写,并且也由该类来完成对于真正包装对象属性的读写
{%codeblock lang:java PropertyHandler%}
//ANPA的内部类,具体由子类实现
protected abstract static class PropertyHandler {

        private final Class<?> propertyType;

        private final boolean readable;

        private final boolean writable;

        public PropertyHandler(Class<?> propertyType, boolean readable, boolean writable) {
            this.propertyType = propertyType;
            this.readable = readable;
            this.writable = writable;
        }

        public Class<?> getPropertyType() {
            return this.propertyType;
        }

        public boolean isReadable() {
            return this.readable;
        }

        public boolean isWritable() {
            return this.writable;
        }
    //...
        @Nullable
        public abstract Object getValue() throws Exception;

        public abstract void setValue(@Nullable Object value) throws Exception;
    }
  //----------------------BeanWrapperImpl实现----------------------------------
  private class BeanPropertyHandler extends PropertyHandler {
    //内部实际封装了PD
        private final PropertyDescriptor pd;

        public BeanPropertyHandler(PropertyDescriptor pd) {
            super(pd.getPropertyType(), pd.getReadMethod() != null, pd.getWriteMethod() != null);
            this.pd = pd;
        }
    //...
        @Override
        @Nullable
        public Object getValue() throws Exception {
            final Method readMethod = this.pd.getReadMethod();
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    ReflectionUtils.makeAccessible(readMethod);
                    return null;
                });
                try {
          //使用内省wrappedObject的对应属性值
                    return AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
                            readMethod.invoke(getWrappedInstance(), (Object[]) null), acc);
                }
                catch (PrivilegedActionException pae) {
                    throw pae.getException();
                }
            }
            else {
                ReflectionUtils.makeAccessible(readMethod);
                return readMethod.invoke(getWrappedInstance(), (Object[]) null);
            }
        }
    //内省设置wrappedObject的对应属性值
        @Override
        public void setValue(final @Nullable Object value) throws Exception {
            final Method writeMethod = (this.pd instanceof GenericTypeAwarePropertyDescriptor ?
                    ((GenericTypeAwarePropertyDescriptor) this.pd).getWriteMethodForActualAccess() :
                    this.pd.getWriteMethod());
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    ReflectionUtils.makeAccessible(writeMethod);
                    return null;
                });
                try {
                    AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
                            writeMethod.invoke(getWrappedInstance(), value), acc);
                }
                catch (PrivilegedActionException ex) {
                    throw ex.getException();
                }
            }
            else {
                ReflectionUtils.makeAccessible(writeMethod);
                writeMethod.invoke(getWrappedInstance(), value);
            }
        }
    }
  //-----------------获取PropertyHandler,子类实现------------------------
  protected BeanPropertyHandler getLocalPropertyHandler(String propertyName) {
    //CachedIntrospectionResults实际上封装了BeanInfo,因此获取到PropertyDescriptor不是问题
        PropertyDescriptor pd = getCachedIntrospectionResults().getPropertyDescriptor(propertyName);
    //创建BeanPropertyHandler
        return (pd != null ? new BeanPropertyHandler(pd) : null);
    }
{%endcodeblock%}
###### PropertyTokenHolder
对于一个属性的Token表示
{%codeblock lang:java PropertyTokenHolder%}
protected static class PropertyTokenHolder {

  public PropertyTokenHolder(String name) {
    this.actualName = name;
    this.canonicalName = name;
  }
  //属性的实际名称
  public String actualName;
  //用户使用时,可能的名称 如departments[0]
  public String canonicalName;
  //当访问的属性是[]这样情况,keys就是[]内部的值
  @Nullable
  public String[] keys;
}
{%endcodeblock%}
###### 读取实现逻辑
{%asset_img get逻辑.png%}
- 逻辑线
{%codeblock lang:java%}
public Object getPropertyValue(String propertyName) throws BeansException {
    //获取正确的层级ANPA
        AbstractNestablePropertyAccessor nestedPa = getPropertyAccessorForPropertyPath(propertyName);
    //创建tokens
        PropertyTokenHolder tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
    //正确的获取值
        return nestedPa.getPropertyValue(tokens);
    }

  //根据tokens获取对应属性的值
  protected Object getPropertyValue(PropertyTokenHolder tokens) throws BeansException {
        String propertyName = tokens.canonicalName;
        String actualName = tokens.actualName;
    //[1] 对应的ph获取
        PropertyHandler ph = getLocalPropertyHandler(actualName);
        if (ph == null || !ph.isReadable()) {
            throw new NotReadablePropertyException(getRootClass(), this.nestedPath + propertyName);
        }
    //[1!]
        try {
      //[2] 内省获取对应的值
            Object value = ph.getValue();
            if (tokens.keys != null) {
                if (value == null) { //这说明此时tokens获取的[]未设置默认值
                    if (isAutoGrowNestedPaths()) {
            //此处设置的value一定是一个空数组|空集合,而对应key的值还是null
            //注意此时使用actualName创建token,也就是说此处仅仅给[]类型创建默认值
                        value = setDefaultValue(new PropertyTokenHolder(tokens.actualName));
                    }
                    else {
                        throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + propertyName,
                                "Cannot access indexed value of property referenced in indexed " +
                                        "property path '" + propertyName + "': returned null");
                    }
                }
                StringBuilder indexedPropertyName = new StringBuilder(tokens.actualName);
        //[2!]
                // apply indexes and map keys
        //[3] 根据tokens的中key处理[]情况,给对应的位置创建默认值
                for (int i = 0; i < tokens.keys.length; i++) {
                    String key = tokens.keys[i];
                    if (value == null) {
                        throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + propertyName,
                                "Cannot access indexed value of property referenced in indexed " +
                                        "property path '" + propertyName + "': returned null");
                    }
                    else if (value.getClass().isArray()) {
                        int index = Integer.parseInt(key);
                        value = growArrayIfNecessary(value, index, indexedPropertyName.toString());
                        value = Array.get(value, index);
                    }
                    else if (value instanceof List) {
                        int index = Integer.parseInt(key);
                        List<Object> list = (List<Object>) value;
                        growCollectionIfNecessary(list, index, indexedPropertyName.toString(), ph, i + 1);
                        value = list.get(index);
                    }
                    else if (value instanceof Set) {
                        // Apply index to Iterator in case of a Set.
                        Set<Object> set = (Set<Object>) value;
                        int index = Integer.parseInt(key);
                        if (index < 0 || index >= set.size()) {
                            throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                                    "Cannot get element with index " + index + " from Set of size " +
                                            set.size() + ", accessed using property path '" + propertyName + "'");
                        }
                        Iterator<Object> it = set.iterator();
                        for (int j = 0; it.hasNext(); j++) {
                            Object elem = it.next();
                            if (j == index) {
                                value = elem;
                                break;
                            }
                        }
                    }
                    else if (value instanceof Map) {
                        Map<Object, Object> map = (Map<Object, Object>) value;
                        Class<?> mapKeyType = ph.getResolvableType().getNested(i + 1).asMap().resolveGeneric(0);
                        // IMPORTANT: Do not pass full property name in here - property editors
                        // must not kick in for map keys but rather only for map values.
                        TypeDescriptor typeDescriptor = TypeDescriptor.valueOf(mapKeyType);
                        Object convertedMapKey = convertIfNecessary(null, null, key, mapKeyType, typeDescriptor);
                        value = map.get(convertedMapKey);
                    }
                    else {
                        throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                                "Property referenced in indexed property path '" + propertyName +
                                        "' is neither an array nor a List nor a Set nor a Map; returned value was [" + value + "]");
                    }
                    indexedPropertyName.append(PROPERTY_KEY_PREFIX).append(key).append(PROPERTY_KEY_SUFFIX);
                }
            }
            return value;
        }
    //[3!]
        catch (IndexOutOfBoundsException ex) {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                    "Index of out of bounds in property path '" + propertyName + "'", ex);
        }
        catch (NumberFormatException | TypeMismatchException ex) {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                    "Invalid index in property path '" + propertyName + "'", ex);
        }
        catch (InvocationTargetException ex) {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                    "Getter for property '" + actualName + "' threw exception", ex);
        }
        catch (Exception ex) {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                    "Illegal attempt to get property '" + actualName + "' threw exception", ex);
        }
    }
{%endcodeblock%}
- 获取对应层次ANPA
{%codeblock lang:java %}
//该函数的递归调用会导致AbstractNestablePropertyAccessor一层一层向下创建,一直返回最后一层的
protected AbstractNestablePropertyAccessor getPropertyAccessorForPropertyPath(String propertyPath) {
    int pos = PropertyAccessorUtils.getFirstNestedPropertySeparatorIndex(propertyPath);
    // Handle nested properties recursively.
    if (pos > -1) {
      String nestedProperty = propertyPath.substring(0, pos);
      String nestedPath = propertyPath.substring(pos + 1);
      //根据当前属性在当前ANPA获取由该属性封装的ANPA
      //"departments[0].employees[0].name"为例子,第一次该函数返回封装departments[0]的AbstractNestablePropertyAccessor
      //再由此为出发点获取后边的部分,一直到最后的封装了name的的AbstractNestablePropertyAccessor,则递归返回
      AbstractNestablePropertyAccessor nestedPa = getNestedPropertyAccessor(nestedProperty);
      //获取分隔符后边的
      return nestedPa.getPropertyAccessorForPropertyPath(nestedPath);
    }
    else {//递归返回条件,即当前属性不存在.分割符,那么本身的ANPA就是正确的
      return this;
    }
  }
  public static int getFirstNestedPropertySeparatorIndex(String propertyPath) {
        return getNestedPropertySeparatorIndex(propertyPath, false);
    }
  //last == false 意思是获取第一个分隔符的index,分隔符实际上就是 .
  private static int getNestedPropertySeparatorIndex(String propertyPath, boolean last) {
    //true 表示当前下标在[]内部因此叫做inKey
    boolean inKey = false;
    int length = propertyPath.length();
    int i = (last ? length - 1 : 0); //很容易想,让开始index=0表示从左开始, length-1表示从右开始
    while (last ? i >= 0 : i < length) {
      switch (propertyPath.charAt(i)) {
        //前两个case表示必须遇到一对[] inkey才能==false
        case PropertyAccessor.PROPERTY_KEY_PREFIX_CHAR: // [ 符号
        case PropertyAccessor.PROPERTY_KEY_SUFFIX_CHAR: // ] 符号
          inKey = !inKey;
          break;
        case PropertyAccessor.NESTED_PROPERTY_SEPARATOR_CHAR: //遇到分隔符.
          if (!inKey) {
            return i;
          }
      }
      if (last) {
        i--;
      }
      else {
        i++;
      }
    }
    return -1;
  }  

private AbstractNestablePropertyAccessor getNestedPropertyAccessor(String nestedProperty) {
        if (this.nestedPropertyAccessors == null) {
            this.nestedPropertyAccessors = new HashMap<>();
        }
        // Get value of bean property.
    //创建PropertyTokenHolder
        PropertyTokenHolder tokens = getPropertyNameTokens(nestedProperty);
        String canonicalName = tokens.canonicalName;
    //获取对应的值
        Object value = getPropertyValue(tokens);

        if (value == null || (value instanceof Optional && !((Optional) value).isPresent())) {
      //如果走到这里说明嵌套类型不是数组|集合如 boss.name,在处理boss部分的时候此处value返回就是null
            if (isAutoGrowNestedPaths()) {
                value = setDefaultValue(tokens);
            }
            else {
                throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + canonicalName);
            }
        }

        // 当前层级的AbstractNestablePropertyAccessor缓存次层级的
        AbstractNestablePropertyAccessor nestedPa = this.nestedPropertyAccessors.get(canonicalName);
        if (nestedPa == null || nestedPa.getWrappedInstance() != ObjectUtils.unwrapOptional(value)) {
            if (logger.isTraceEnabled()) {
                logger.trace("Creating new nested " + getClass().getSimpleName() + " for property '" + canonicalName + "'");
            }
      //根据此次层级属性创建次层级AbstractNestablePropertyAccessor,并由该层级缓存
            nestedPa = newNestedPropertyAccessor(value, this.nestedPath + canonicalName + NESTED_PROPERTY_SEPARATOR);
            // Inherit all type-specific PropertyEditors.
            copyDefaultEditorsTo(nestedPa);
            copyCustomEditorsTo(nestedPa, canonicalName);
            this.nestedPropertyAccessors.put(canonicalName, nestedPa);
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Using cached nested property accessor for property '" + canonicalName + "'");
            }
        }
        return nestedPa;
    }
{%endcodeblock%}
- 默认值
{%codeblock lang:java%}
private Object setDefaultValue(PropertyTokenHolder tokens) {
        PropertyValue pv = createDefaultPropertyValue(tokens);
    //设置
        setPropertyValue(tokens, pv);
    //获取值
        Object defaultValue = getPropertyValue(tokens);
        Assert.state(defaultValue != null, "Default value must not be null");
        return defaultValue;
    }
  //根据类型创建一个默认pv并返回
    private PropertyValue createDefaultPropertyValue(PropertyTokenHolder tokens) {
        TypeDescriptor desc = getPropertyTypeDescriptor(tokens.canonicalName);
        if (desc == null) {
            throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + tokens.canonicalName,
                    "Could not determine property type for auto-growing a default value");
        }
        Object defaultValue = newValue(desc.getType(), desc, tokens.canonicalName);
        return new PropertyValue(tokens.canonicalName, defaultValue);
    }
  //数组 和集合创建空值
    private Object newValue(Class<?> type, @Nullable TypeDescriptor desc, String name) {
        try {
            if (type.isArray()) {
                Class<?> componentType = type.getComponentType();
                // TODO - only handles 2-dimensional arrays
                if (componentType.isArray()) {
                    Object array = Array.newInstance(componentType, 1);
                    Array.set(array, 0, Array.newInstance(componentType.getComponentType(), 0));
                    return array;
                }
                else {
                    return Array.newInstance(componentType, 0);
                }
            }
            else if (Collection.class.isAssignableFrom(type)) {
                TypeDescriptor elementDesc = (desc != null ? desc.getElementTypeDescriptor() : null);
                return CollectionFactory.createCollection(type, (elementDesc != null ? elementDesc.getType() : null), 16);
            }
            else if (Map.class.isAssignableFrom(type)) {
                TypeDescriptor keyDesc = (desc != null ? desc.getMapKeyTypeDescriptor() : null);
                return CollectionFactory.createMap(type, (keyDesc != null ? keyDesc.getType() : null), 16);
            }
            else {
                Constructor<?> ctor = type.getDeclaredConstructor();
                if (Modifier.isPrivate(ctor.getModifiers())) {
                    throw new IllegalAccessException("Auto-growing not allowed with private constructor: " + ctor);
                }
                return BeanUtils.instantiateClass(ctor);
            }
        }
        catch (Throwable ex) {
            throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + name,
                    "Could not instantiate property type [" + type.getName() + "] to auto-grow nested property path", ex);
        }
    }
{%endcodeblock%}
总结
- 对于getPropertyValue(PropertyTokenHolder),存在这几种情况
  - departments[0]:此时token:{actualName:departments ;canonicalName:departments[0];keys:{0}}
  若此时departments属性为null,则会出现departments默认设置,以及对应key位置的默认设置
  - boss:此时token:{boos,boos,null},那么会直接返回null
- 若是创建层级ANPA过程调用getPropertyValue(PropertyTokenHolder)
  - departments[0].xx:不多做处理
  - boss.name: 在boss层级创建返回null,由创建层次函数创建boss.
- getPropertyValue有对[]类型设置默认值,并处理对应key的能力
- getNestedPropertyAccessor:有对非[]类设置默认值的能力
###### 设值逻辑
{%asset_img set逻辑.png%}
- 逻辑线
{%codeblock lang:java%}
public void setPropertyValue(PropertyValue pv) throws BeansException {
        PropertyTokenHolder tokens = (PropertyTokenHolder) pv.resolvedTokens;
        if (tokens == null) {
            String propertyName = pv.getName();
            AbstractNestablePropertyAccessor nestedPa;
            try {
                nestedPa = getPropertyAccessorForPropertyPath(propertyName);
            }
            catch (NotReadablePropertyException ex) {
                throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName,
                        "Nested property in path '" + propertyName + "' does not exist", ex);
            }
            tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
            if (nestedPa == this) {
                pv.getOriginalPropertyValue().resolvedTokens = tokens;
            }
            nestedPa.setPropertyValue(tokens, pv);
        }
        else {
            setPropertyValue(tokens, pv);
        }
    }  
  //设值
  protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
          if (tokens.keys != null) { //只有最终属性是[]的情况才会调用这里如,departments[1]
              processKeyedProperty(tokens, pv);
          }
          else {
              processLocalProperty(tokens, pv);
          }
      }  


  //------------------------------最终为[]的情况处理--------------------------------  
    private void processKeyedProperty(PropertyTokenHolder tokens, PropertyValue pv) {
      //获取该属性的值,如一个List,数组等
        Object propValue = getPropertyHoldingValue(tokens);
    //对于该属性的ph
        PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
        if (ph == null) {
            throw new InvalidPropertyException(
                    getRootClass(), this.nestedPath + tokens.actualName, "No property handler found");
        }
        Assert.state(tokens.keys != null, "No token keys");
        String lastKey = tokens.keys[tokens.keys.length - 1];

    //分类讨论,实际就是根据key,将pv设置到对应位置,可能存在类型转换的问题
        if (propValue.getClass().isArray()) {
            Class<?> requiredType = propValue.getClass().getComponentType();
            int arrayIndex = Integer.parseInt(lastKey);
            Object oldValue = null;
            try {
                if (isExtractOldValueForEditor() && arrayIndex < Array.getLength(propValue)) {
                    oldValue = Array.get(propValue, arrayIndex);
                }
                Object convertedValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                        requiredType, ph.nested(tokens.keys.length));
                int length = Array.getLength(propValue);
                if (arrayIndex >= length && arrayIndex < this.autoGrowCollectionLimit) {
                    Class<?> componentType = propValue.getClass().getComponentType();
                    Object newArray = Array.newInstance(componentType, arrayIndex + 1);
                    System.arraycopy(propValue, 0, newArray, 0, length);
                    setPropertyValue(tokens.actualName, newArray);
                    propValue = getPropertyValue(tokens.actualName);
                }
                Array.set(propValue, arrayIndex, convertedValue);
            }
            catch (IndexOutOfBoundsException ex) {
                throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                        "Invalid array index in property path '" + tokens.canonicalName + "'", ex);
            }
        }

        else if (propValue instanceof List) {
            Class<?> requiredType = ph.getCollectionType(tokens.keys.length);
            List<Object> list = (List<Object>) propValue;
            int index = Integer.parseInt(lastKey);
            Object oldValue = null;
            if (isExtractOldValueForEditor() && index < list.size()) {
                oldValue = list.get(index);
            }
            Object convertedValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                    requiredType, ph.nested(tokens.keys.length));
            int size = list.size();
            if (index >= size && index < this.autoGrowCollectionLimit) {
                for (int i = size; i < index; i++) {
                    try {
                        list.add(null);
                    }
                    catch (NullPointerException ex) {
                        throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                                "Cannot set element with index " + index + " in List of size " +
                                size + ", accessed using property path '" + tokens.canonicalName +
                                "': List does not support filling up gaps with null elements");
                    }
                }
                list.add(convertedValue);
            }
            else {
                try {
                    list.set(index, convertedValue);
                }
                catch (IndexOutOfBoundsException ex) {
                    throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                            "Invalid list index in property path '" + tokens.canonicalName + "'", ex);
                }
            }
        }

        else if (propValue instanceof Map) {
            Class<?> mapKeyType = ph.getMapKeyType(tokens.keys.length);
            Class<?> mapValueType = ph.getMapValueType(tokens.keys.length);
            Map<Object, Object> map = (Map<Object, Object>) propValue;
            // IMPORTANT: Do not pass full property name in here - property editors
            // must not kick in for map keys but rather only for map values.
            TypeDescriptor typeDescriptor = TypeDescriptor.valueOf(mapKeyType);
            Object convertedMapKey = convertIfNecessary(null, null, lastKey, mapKeyType, typeDescriptor);
            Object oldValue = null;
            if (isExtractOldValueForEditor()) {
                oldValue = map.get(convertedMapKey);
            }
            // Pass full property name and old value in here, since we want full
            // conversion ability for map values.
            Object convertedMapValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                    mapValueType, ph.nested(tokens.keys.length));
            map.put(convertedMapKey, convertedMapValue);
        }

        else {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                    "Property referenced in indexed property path '" + tokens.canonicalName +
                    "' is neither an array nor a List nor a Map; returned value was [" + propValue + "]");
        }
    }
  //--------------获取[]的值----------------
    private Object getPropertyHoldingValue(PropertyTokenHolder tokens) {
        // Apply indexes and map keys: fetch value for all keys but the last one.
        Assert.state(tokens.keys != null, "No token keys");
    //仅仅根据属性名创建一个tokens
        PropertyTokenHolder getterTokens = new PropertyTokenHolder(tokens.actualName);
        getterTokens.canonicalName = tokens.canonicalName;
        getterTokens.keys = new String[tokens.keys.length - 1];
        System.arraycopy(tokens.keys, 0, getterTokens.keys, 0, tokens.keys.length - 1);

        Object propValue;
        try {
      //获取属性,具体见下文,中间可能存在创建过程
            propValue = getPropertyValue(getterTokens);
        }
        catch (NotReadablePropertyException ex) {
            throw new NotWritablePropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                    "Cannot access indexed value in property referenced " +
                    "in indexed property path '" + tokens.canonicalName + "'", ex);
        }

        if (propValue == null) {
            // null map value case
            if (isAutoGrowNestedPaths()) {
                int lastKeyIndex = tokens.canonicalName.lastIndexOf('[');
                getterTokens.canonicalName = tokens.canonicalName.substring(0, lastKeyIndex);
                propValue = setDefaultValue(getterTokens);
            }
            else {
                throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + tokens.canonicalName,
                        "Cannot access indexed value in property referenced " +
                        "in indexed property path '" + tokens.canonicalName + "': returned null");
            }
        }
        return propValue;
    }
    //---------------------------非集合或数组-------------------------------
    private void processLocalProperty(PropertyTokenHolder tokens, PropertyValue pv) {
      [1]获取ph子类实现,见下文
          PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
          if (ph == null || !ph.isWritable()) {
              if (pv.isOptional()) {
                  if (logger.isDebugEnabled()) {
                      logger.debug("Ignoring optional value for property '" + tokens.actualName +
                              "' - property not found on bean class [" + getRootClass().getName() + "]");
                  }
                  return;
              }
              else {
                  throw createNotWritablePropertyException(tokens.canonicalName);
              }
          }
      //![1]
      //[2]可能会存在类型转换
          Object oldValue = null;
          try {
              Object originalValue = pv.getValue();
              Object valueToApply = originalValue;
              if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
                  if (pv.isConverted()) {
                      valueToApply = pv.getConvertedValue();
                  }
                  else {
                      if (isExtractOldValueForEditor() && ph.isReadable()) {
                          try {
                              oldValue = ph.getValue();
                          }
                          catch (Exception ex) {
                              if (ex instanceof PrivilegedActionException) {
                                  ex = ((PrivilegedActionException) ex).getException();
                              }
                              if (logger.isDebugEnabled()) {
                                  logger.debug("Could not read previous value of property '" +
                                          this.nestedPath + tokens.canonicalName + "'", ex);
                              }
                          }
                      }
                      valueToApply = convertForProperty(
                              tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());
                  }
          //![2]类型转换
                  pv.getOriginalPropertyValue().conversionNecessary = (valueToApply != originalValue);
              }
        //[3]内省设值,见下文
              ph.setValue(valueToApply);
        //![3]
          }
          catch (TypeMismatchException ex) {
              throw ex;
          }
          catch (InvocationTargetException ex) {
              PropertyChangeEvent propertyChangeEvent = new PropertyChangeEvent(
                      getRootInstance(), this.nestedPath + tokens.canonicalName, oldValue, pv.getValue());
              if (ex.getTargetException() instanceof ClassCastException) {
                  throw new TypeMismatchException(propertyChangeEvent, ph.getPropertyType(), ex.getTargetException());
              }
              else {
                  Throwable cause = ex.getTargetException();
                  if (cause instanceof UndeclaredThrowableException) {
                      // May happen e.g. with Groovy-generated methods
                      cause = cause.getCause();
                  }
                  throw new MethodInvocationException(propertyChangeEvent, cause);
              }
          }
          catch (Exception ex) {
              PropertyChangeEvent pce = new PropertyChangeEvent(
                      getRootInstance(), this.nestedPath + tokens.canonicalName, oldValue, pv.getValue());
              throw new MethodInvocationException(pce, ex);
          }
{%endcodeblock%}
设值遇到的类型转换参考[^1]
[^1]:[属性转换](/2020/02/11/spring常见/#属性类型转换)
