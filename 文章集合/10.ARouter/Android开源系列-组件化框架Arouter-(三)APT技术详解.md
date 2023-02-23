---
theme: juejin
highlight: a11y-light
---
携手创作，共同成长！这是我参与「掘金日新计划 · 8 月更文挑战」的第8天，[点击查看活动详情](https://juejin.cn/post/7123120819437322247 "https://juejin.cn/post/7123120819437322247") >>
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
最近组里需要进行**组件化框架**的改造，用到了**ARouter**这个开源框架，为了更好的对项目进行改造，笔者花了一些时间去了解了下ARouter

`ARouter`是阿里巴巴团队在17年初发布的一款针对**组件化**模块之间**无接触通讯**的一个开源框架，经过多个版本的迭代，现在已经非常成熟了。

`ARouter`**主要作用**：**组件间通讯，组件解耦，路由跳转，涉及到我们常用的Activity，Provider，Fragment等多个场景的跳转**

接下来笔者会以几个阶段来对`Arouter`进行讲解：

> - [Android开源系列-组件化框架Arouter-(一)使用方式详解](https://juejin.cn/post/7134326382565261348)
> - [Android开源系列-组件化框架Arouter-(二)深度原理解析](https://juejin.cn/post/7134632360087126023)
> - Android开源系列-组件化框架Arouter-(三)APT技术详解
> - Android开源系列-组件化框架Arouter-(四)AGP插件详解

这篇文章我们来讲解下：**APT技术详解**

## APT前置知识
### 注解基础：
#### 1.元注解

- 1.`@Target`：目标，表示注解修饰的目标

    - **ElementType.`ANNOTIONS_TYPE`**: 目标是注解，给注解设置的注解
    - **ElementType.`CONSTRUCTOR`**: 构造方法
    - **ElementType.`FIELD`**: 属性注解
    - **ElementType.`METHOD`**: 方法注解
    - **ElementType.`Type`**: 类型如：类，接口，枚举
    - **ElementType.`PACKAGE`**: 可以给一个包进行注解
    - **ElementType.`PARAMETER`**: 可以给一个方法内的参数进行注解
    - **ElementType.`LOCAL_VARIABLE`**: 可以给局部变量进行注解

- 2.`@Retention`:表示需要在什么级别保存该注解信息

    - **RetentionPolicy.`SOURCE`**:在编译阶段有用，编译之后会被丢弃，不会保存到字节码class文件中
    - **RetentionPolicy.`CLASS`**：注解在class文件中可用，但是会被VM丢弃，在类加载时会被丢弃，在字节码文件处理中有用，注解默认使用这种方式
    - **RetentionPolicy.`RUNTIME`**：运行时有效，可以通过反射获取注解信息

- 3.`@Document`：将注解包含到javaDoc中

- 4.`@Inherit`：运行子类继承父类的注解

- 5.`@Repeatable`:定义注解可重复


#### 2.元注解的使用方式
- 2.1：基本使用方式
```java
@Target(ElementType.METHOD) ://表示作用在方法中
@Retention(RetentionPolicy.SOURCE) ：//表示只在编译器有效
public @interface Demo1 {
        public int id(); //注解的值，无默认值，在创建注解的时候需要设置该值
        public String desc() default "no info";//注解默认值
}

@Demo1(id=1)
public void getData() {
}
```


- 2.2：重复注解使用方式

 `定义Persons`：

```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)
public   @interface Persons {
        Person[] value();
}
```
`定义Person`：

```java
@Repeatable(Persons.class)
public  @interface Person{
        String role() default "";
}
```
`使用使用`：

```java
@Person(role="CEO")
@Person(role="husband")
@Person(role="father")
@Person(role="son")
public   class Man {
        String name="";
}
```
`调用注解`

```java
if(Man.class.isAnnotationPresent(Persons.class)) {先判断是否存在这个注解
        Persons p2=Man.class.getAnnotation(Persons.class);获取注解
        for(Person t:p2.value()){
                System.out.println(t.role());
        }
} 
 结果：
	1
	CEO
	husband
	father
	son
```


#### 3.运行时注解
需要使用反射获取

```JAVA
@Retention(RetentionPolicy.RUNTIME)
public void getAnnoInfo() {
    Class clazz = GetAnno.class;
     //获得所有的方法
    Method[] methods = clazz.getMethods();
    for (Method method : methods) {
            method.setAccessible(true);//禁用安全机制
            if (method.isAnnotationPresent(Demo1.class)) {//检查是否使用了Demo1注解
                    Demo1 demo1 = method.getAnnotation(Demo1.class);//获得注解实例
                    String name = method.getName();//获得方法名称
    }
}
```


#### 4.编译时注解
需要使用到`APT`工具

@Retention(RetentionPolicy.`SOURCE`)或者`CLASS`注解的获取
可以使用编译期注解动态生成代码，很多优秀的开源库都是使用这个方式：如`Arouter` `ButterKnife`，`GreenDao`，`EventBus3`等

### APT知识储备
- 1.APT是一种`注解解析工具`：

**在`编译期`找出源代码中所有的注解信息，如果指定了注解器(继承`AbstractProcessor`)，那么在编译期会调用这个注解器里面的代码，我们可以在这里面做一些处理，
如根据注解信息动态生成一些代码，并将代码注入到源码中**

- 使用到的工具类：

**工具类1**：`Element`

表示程序的一个元素，它只在编译期存在。可以是`package，class，interface，method`,成员变量，函数参数，泛型类型等。

`Element`的子类介绍：

* `ExecutableElement`:类或者接口中的方法，构造器或者初始化器等元素
* `PackageElement`:代表一个包元素程序
* `VariableElement`:代表一个类或者接口中的属性或者常量的枚举类型，方法或者构造器的参数，局部变量，资源变量或者异常参数
* `TypeElement`:代表一个类或者接口元素
* `TypeParameterElement`:代表接口，类或者方法的泛型参数元素

通过`Element`可以获取什么信息呢？

    1.asType() 返回TypeMirror：
        TypeMirror是元素的类型信息，包括包名，类(或方法，或参数)名/类型
        TypeMirror的子类：
        ArrayType, DeclaredType, DisjunctiveType, ErrorType, ExecutableType, NoType, NullType, PrimitiveType, ReferenceType, TypeVariable, WildcardType
        getKind可以获取类型：
    2.equals(Object obj) 比较两个Element利用equals方法。
    3.getAnnotation(Class annotationType) 传入注解可以获取该元素上的所有注解。
    4.getAnnotationMirrors() 获该元素上的注解类型。
    5.getEnclosedElements() 获取该元素上的直接子元素，类似一个类中有VariableElement。
    6.getEnclosingElement() 获取该元素的父元素,
        如果是属性VariableElement，则其父元素为TypeElement，
        如果是PackageElement则返回null，
        如果是TypeElement则返回PackageElement，
        如果是TypeParameterElement则返回泛型Element
    7.getKind() 返回值为ElementKind,通过ElementKind可以知道是那种element，具体就是Element的那些子类。
    8.getModifiers() 获取修饰该元素的访问修饰符，public，private
    9.getSimpleName() 获取元素名，不带包名，
        如果是变量，获取的就是变量名，
        如果是定义了int age，获取到的name就是age。
        如果是TypeElement返回的就是类名
    10.getQualifiedName()：获取类的全限定名，Element没有这个方法它的子类有，例如TypeElement，得到的就是类的全类名（包名）。
    11.Elements.getPackageOf(enclosingElement).asType().toString()：获取所在的包名:


​        
**工具类2：**`ProcessingEnvironment`：

`APT运行环境`：里面提供了写新文件, 报告错误或者查找其他工具.

    1.getFiler()：返回用于创建新的源，类或辅助文件的文件管理器。
    2.getElementUtils():返回对元素进行操作的一些实用方法的实现.
    3.getMessager():返回用于报告错误，警告和其他通知的信使。
    4.getOptions():返回传递给注解处理工具的处理器特定选项。
    5.getTypeUtils():返回一些用于对类型进行操作的实用方法的实现。

**工具类3**:`ElementKind`

如何判断Element的类型呢，需要用到`ElementKind`，`ElementKind`为元素的类型，元素的类型判断不需要用`instanceof`去判断，而应该通过`getKind()`去判断对应的类型

    element.getKind()==ElementKind.CLASS;

**工具类4：**`TypeKind`

`TypeKind`为类型的属性，类型的属性判断不需要用instanceof去判断，而应该通过`getKind()`去判断对应的属性

    element.asType().getKind() == TypeKind.INT

### javapoet:生成java文件

3种生成文件的方式：
- 1.`StringBuilder`·进行拼接 
- 2.模板文件进行字段替换 
- 3.`javaPoet` 生成

> StringBuilder进行拼接，模板文件进行字段替换进行简单文件生成还好，如果是复杂文件，拼接起来会相当复杂

所以一般复杂的都使用`Square`出品的sdk：`javapoet`
		
```java
implementation "com.squareup:javapoet:1.11.1"
```

### 自己实现自定义APT工具类

`步骤`：
##### 1.创建一个单独javalib模块`lib_annotions`：
创建需要的注解类：
```
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface BindView {
    int value();
}
```
##### 2.再创建一个`javalib`模块`lib_compilers`:
在模块中创建一个继承`AbstractProcessor`的类:
```java
@AutoService(Processor.class)
public class CustomProcessorTest extends AbstractProcessor {
    public Filer filer;
    private Messager messager;
    private List<String> result = new ArrayList<>();
    private int round;
    private Elements elementUtils;
    private Map<String, String> options;

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<>();
        annotations.add(CustomBindAnnotation.class.getCanonicalName());
        return annotations;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        filer = processingEnvironment.getFiler();
        messager = processingEnvironment.getMessager();
        elementUtils = processingEnv.getElementUtils();
        options = processingEnv.getOptions();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        messager.printMessage(Diagnostic.Kind.NOTE,"process");
        Map<TypeElement, Map<Integer, VariableElement>> typeElementMap = getTypeElementMap(roundEnv);
        messager.printMessage(Diagnostic.Kind.NOTE,"2222");
        for(TypeElement key:typeElementMap.keySet()){
            Map<Integer, VariableElement> variableElementMap = typeElementMap.get(key);
            TypeSpec typeSpec = generalTypeSpec(key,variableElementMap);
            String packetName = elementUtils.getPackageOf(key).getQualifiedName().toString();
            messager.printMessage(Diagnostic.Kind.NOTE,"packetName:"+packetName);

            JavaFile javaFile = JavaFile.builder(packetName,typeSpec).build();
            try {
                javaFile.writeTo(processingEnv.getFiler());
                messager.printMessage(Diagnostic.Kind.NOTE,"3333");
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return true;
    }

    private TypeSpec generalTypeSpec(TypeElement key,Map<Integer, VariableElement> variableElementMap) {
        return TypeSpec.classBuilder(key.getSimpleName().toString()+"ViewBinding")
                .addModifiers(Modifier.PUBLIC)
                .addMethod(generalMethodSpec(key,variableElementMap))
                .build();
    }

    private MethodSpec generalMethodSpec(TypeElement typeElement, Map<Integer, VariableElement> variableElementMap) {
        ClassName className = ClassName.bestGuess(typeElement.getQualifiedName().toString());
        String parameter = "_" + toLowerCaseFirstChar(className.simpleName());
        MethodSpec.Builder builder = MethodSpec.methodBuilder("bind")
                .addModifiers(Modifier.PUBLIC,Modifier.STATIC)
                .returns(void.class)
                .addParameter(className,parameter);
        messager.printMessage(Diagnostic.Kind.NOTE,"typeElement.getQualifiedName().toString():"+typeElement.getQualifiedName().toString());

        messager.printMessage(Diagnostic.Kind.NOTE,"typeElement.className():"+className.simpleName().toString());
        messager.printMessage(Diagnostic.Kind.NOTE,"parameter:"+parameter);
        for(int viewId:variableElementMap.keySet()){
            VariableElement variableElement = variableElementMap.get(viewId);
            String elementName = variableElement.getSimpleName().toString();
            String elementType = variableElement.asType().toString();
            messager.printMessage(Diagnostic.Kind.NOTE,"elementName:"+elementName);
            messager.printMessage(Diagnostic.Kind.NOTE,"elementType:"+elementType);
//            builder.addCode("$L.$L = ($L)$L.findViewById($L);\n",parameter,elementName,elementType,parameter,viewId);
            builder.addStatement("$L.$L = ($L)$L.findViewById($L)",parameter,elementName,elementType,parameter,viewId);
        }
//        for (int viewId : varElementMap.keySet()) {
//            VariableElement element = varElementMap.get(viewId);
//            String name = element.getSimpleName().toString();
//            String type = element.asType().toString();
//            String text = "{0}.{1}=({2})({3}.findViewById({4}));";
//            builder.addCode(MessageFormat.format(text, parameter, name, type, parameter, String.valueOf(viewId)));
//        }
        return builder.build();
    }

    private Map<TypeElement, Map<Integer, VariableElement>> getTypeElementMap(RoundEnvironment roundEnv) {
        Map<TypeElement, Map<Integer, VariableElement>> typeElementMap = new HashMap<>();
        messager.printMessage(Diagnostic.Kind.NOTE,"1111");
        Set<? extends Element> variableElements = roundEnv.getElementsAnnotatedWith(CustomBindAnnotation.class);
        for(Element element:variableElements){
            VariableElement variableElement = (VariableElement) element;//作用在字段上，可以强制转换为VariableElement
            TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();
            Map<Integer, VariableElement> varElementMap = typeElementMap.get(typeElement);
            if(varElementMap == null){
                varElementMap = new HashMap<>();
                typeElementMap.put(typeElement,varElementMap);
            }
            CustomBindAnnotation customBindAnnotation = variableElement.getAnnotation(CustomBindAnnotation.class);
            int viewId = customBindAnnotation.value();
            varElementMap.put(viewId,variableElement);
        }
        return typeElementMap;
    }
    //将首字母转为小写
    private static String toLowerCaseFirstChar(String text) {
        if (text == null || text.length() == 0 || Character.isLowerCase(text.charAt(0))) return text;
        else return String.valueOf(Character.toLowerCase(text.charAt(0))) + text.substring(1);
    }
}
```
这个类中：重写以下方法

	1.getSupportedAnnotationTypes：
		该方法主要作用是：返回支持的注解类型
		public Set<String> getSupportedAnnotationTypes() {
			Set<String> hashSet = new HashSet<>();
			hashSet.add(BindView.class.getCanonicalName());
			return hashSet;
		}
	2.getSupportedSourceVersion：
		作用：返回支持的jdk版本
		public SourceVersion getSupportedSourceVersion() {
			return SourceVersion.latestSupported();
		}
	3.init(ProcessingEnvironment processingEnvironment)
		作用：返回一个ProcessingEnvironment
		这个工具内部有很多处理类
		1.getFiler()：返回用于创建新的源，类或辅助文件的文件管理器。
		2.getElementUtils():返回对元素进行操作的一些实用方法的实现.
		3.getMessager():返回用于报告错误，警告和其他通知的信使。
		4.getOptions():返回传递给注解处理工具的处理器特定选项。
		5.getTypeUtils():返回一些用于对类型进行操作的实用方法的实现。
	4.process(Set<? extends TypeElement> set, RoundEnvironment environment)：
		作用：apt核心处理方法，可以在这里面对收集到的注解进行处理，生成动态原文件等
##### 3.在模块的`build.gradle`文件中

```java
implementation "com.google.auto.service:auto-service:1.0-rc6" //使用Auto-Service来自动注册APT
//Android Plugin for Gradle >= 3.4 或者 Gradle Version >=5.0 都要在自己的annotation processor工程里面增加如下的语句
annotationProcessor 'com.google.auto.service:auto-service:1.0-rc6'

implementation "com.squareup:javapoet:1.11.1"//辅助生成文件的工具类
implementation project(':lib_annotionss')//该模块是注解存再的库中
```

##### 4.最后编译会自动生成对应的类。
然后在需要的地方加上注解就可以了。

编译器自动生成的文件：

```java
public class AnnotationActivityViewBinding {
  public static void bind(AnnotationActivity _annotationActivity) {
    _annotationActivity.btn1 = (android.widget.Button)_annotationActivity.findViewById(2131296347);
    _annotationActivity.lv = (android.widget.ListView)_annotationActivity.findViewById(2131296475);
    _annotationActivity.btn = (android.widget.Button)_annotationActivity.findViewById(2131296346);
  }
}
```

## ARouter中APT的使用
我们来看**ARouter**源码框架

![ARouter源码架构.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/814a072417a140679f71e75a8979bd0f~tplv-k3u1fbpfcp-watermark.image?)

- **app**:是ARouter提供的一个测试Demo
- **arouter-annotation**：这个lib模块中声明了很多注解信息和一些枚举类
- **arouter-api**：ARouter的核心api，转换过程的核心操作都在这个模块里面
- **arouter-compiler**：APT处理器，自动生成路由表的过程就是在这里面实现的
- **arouter-gradle-plugin**：这是一个编译期使用的Plugin插件，主要作用是用于编译器自动加载路由表，节省应用的启动时间。

我们主要看**arouter-annotation**和**arouter-compiler**这两个模块

### 1.arouter-annotation

可以看到这里面实现了几个注解类
- `Autowired`：属性注解

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.CLASS)
public @interface Autowired {

    // 标志我们外部调用使用的key
    String name() default "";

    // 如果有要求，一定要传入，不然app会crash
    // Primitive type wont be check!
    boolean required() default false;

    // 注解字段描述
    String desc() default "";
}
```

@Target({ElementType.FIELD})：指定我们注解是使用在属性字段上
@Retention(RetentionPolicy.CLASS)：指定我们注解只在编译期存在

- `Interceptor`：拦截器注解

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.CLASS)
public @interface Interceptor {
    /**
     * The priority of interceptor, ARouter will be excute them follow the priority.
     */
    int priority();

    /**
     * The name of interceptor, may be used to generate javadoc.
     */
    String name() default "Default";
}
```

@Target({ElementType.TYPE})：指定注解是在类上

@Retention(RetentionPolicy.CLASS)：指定注解在编译期存在

- `Route`：路由注解

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.CLASS)
public @interface Route {

    /**
     * Path of route
     */
    String path();

    /**
     * Used to merger routes, the group name MUST BE USE THE COMMON WORDS !!!
     */
    String group() default "";

    /**
     * Name of route, used to generate javadoc.
     */
    String name() default "";

    /**
     * Extra data, can be set by user.
     * Ps. U should use the integer num sign the switch, by bits. 10001010101010
     */
    int extras() default Integer.MIN_VALUE;

    /**
     * The priority of route.
     */
    int priority() default -1;
}
```

@Target({ElementType.TYPE})：指定注解是使用在类上

@Retention(RetentionPolicy.CLASS)：指定注解是在编译期存在

枚举类：
- `RouteType`:路由类型

```java
public enum RouteType {
    ACTIVITY(0, "android.app.Activity"),
    SERVICE(1, "android.app.Service"),
    PROVIDER(2, "com.alibaba.android.arouter.facade.template.IProvider"),
    CONTENT_PROVIDER(-1, "android.app.ContentProvider"),
    BOARDCAST(-1, ""),
    METHOD(-1, ""),
    FRAGMENT(-1, "android.app.Fragment"),
    UNKNOWN(-1, "Unknown route type");
}
TypeKind
public enum TypeKind {
    // Base type
    BOOLEAN,
    BYTE,
    SHORT,
    INT,
    LONG,
    CHAR,
    FLOAT,
    DOUBLE,

    // Other type
    STRING,
    SERIALIZABLE,
    PARCELABLE,
    OBJECT;
}
```

model类
- `RouteMeta`：路由元数据

```java
public class RouteMeta {
    private RouteType type;         // Type of route
    private Element rawType;        // Raw type of route
    private Class<?> destination;   // Destination
    private String path;            // Path of route
    private String group;           // Group of route
    private int priority = -1;      // The smaller the number, the higher the priority
    private int extra;              // Extra data
    private Map<String, Integer> paramsType;  // Param type
    private String name;

    private Map<String, Autowired> injectConfig;  // Cache inject config.
}

```

总结下**arouter-annotation**：
- 1.创建了`Autowired`：属性注解，`Interceptor`：拦截器注解，`Route`：路由注解
- 2.创建了`RouteType`:路由类型枚举，`RouteMeta`：路由元数据

### 2.arouter-compiler

- **AutowiredProcessor**：属性Autowired注解处理器
- **InterceptorProcessor**：拦截器Interceptor注解处理器
- **RouteProcessor**：路由Route注解处理器
- **BaseProcessor**：注解处理器基类，主要获取一些通用参数，上面三个都继承这个基类
- **incremental.annotation.processors**：拦截器声明，这里将我们需要使用的几个注解处理器做了声明

```java
com.alibaba.android.arouter.compiler.processor.RouteProcessor,aggregating
com.alibaba.android.arouter.compiler.processor.AutowiredProcessor,aggregating
com.alibaba.android.arouter.compiler.processor.InterceptorProcessor,aggregating
```


下面依次来看：
#### AutowiredProcessor：

```java
@AutoService(Processor.class)//使用AutoService可以将处理器自动注册到processors文件中
@SupportedAnnotationTypes({ANNOTATION_TYPE_AUTOWIRED}) //设置需要匹配的注解类："com.alibaba.android.arouter.facade.annotation.Autowired"
public class AutowiredProcessor extends BaseProcessor {
    private Map<TypeElement, List<Element>> parentAndChild = new HashMap<>();   // Contain field need autowired and his super class.
    private static final ClassName ARouterClass = ClassName.get("com.alibaba.android.arouter.launcher", "ARouter");
    private static final ClassName AndroidLog = ClassName.get("android.util", "Log");

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);

        logger.info(">>> AutowiredProcessor init. <<<");
    }
	
	//这是注解处理器的核心方法
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        if (CollectionUtils.isNotEmpty(set)) {
            try {
                //这里将所有声明Autowired注解的属性包括在parentAndChild中:parentAndChild的key为注解的类TypeElement
				//parentAndChild{List<Element>{element1,element2,element3...}}
                categories(roundEnvironment.getElementsAnnotatedWith(Autowired.class));
				//生成帮助类
                generateHelper();

            } catch (Exception e) {
                logger.error(e);
            }
            return true;
        }

        return false;
    }

    private void generateHelper() throws IOException, IllegalAccessException {
		//获取com.alibaba.android.arouter.facade.template.ISyringe的TypeElement
        TypeElement type_ISyringe = elementUtils.getTypeElement(ISYRINGE);
		//获取com.alibaba.android.arouter.facade.service.SerializationService的TypeElement
        TypeElement type_JsonService = elementUtils.getTypeElement(JSON_SERVICE);
		//获取com.alibaba.android.arouter.facade.template.IProvider的TypeMirror:元素的类型信息，包括包名，类(或方法，或参数)名/类型
        TypeMirror iProvider = elementUtils.getTypeElement(Consts.IPROVIDER).asType();
		//获取android.app.Activity的TypeMirror:元素的类型信息，包括包名，类(或方法，或参数)名/类型
        TypeMirror activityTm = elementUtils.getTypeElement(Consts.ACTIVITY).asType();
		//获取android.app.Fragment的TypeMirror:元素的类型信息，包括包名，类(或方法，或参数)名/类型
        TypeMirror fragmentTm = elementUtils.getTypeElement(Consts.FRAGMENT).asType();
        TypeMirror fragmentTmV4 = elementUtils.getTypeElement(Consts.FRAGMENT_V4).asType();

        // 生成属性参数的辅助类
        ParameterSpec objectParamSpec = ParameterSpec.builder(TypeName.OBJECT, "target").build();

        if (MapUtils.isNotEmpty(parentAndChild)) {
			//遍历parentAndChild：每个entry使用的key为当前类的TypeElement，value为当前类内部所有使用注解Autowired标记的属性
            for (Map.Entry<TypeElement, List<Element>> entry : parentAndChild.entrySet()) {
				//MethodSpec生成方法的辅助类 METHOD_INJECT = 'inject'
				/**
				方法名：inject
				方法注解：Override
				方法权限：public
				方法参数：前面objectParamSpec生成的：Object target
				
				*/
                MethodSpec.Builder injectMethodBuilder = MethodSpec.methodBuilder(METHOD_INJECT)
                        .addAnnotation(Override.class)
                        .addModifiers(PUBLIC)
                        .addParameter(objectParamSpec);
				//key为当前类的TypeElement
                TypeElement parent = entry.getKey();
				//value为当前类内部所有使用注解Autowired标记的属性
                List<Element> childs = entry.getValue();
				//类的全限定名
                String qualifiedName = parent.getQualifiedName().toString();
				//类的包名
                String packageName = qualifiedName.substring(0, qualifiedName.lastIndexOf("."));
				//类的文件名：NAME_OF_AUTOWIRED = $$ARouter$$Autowired，完整fileName = BaseActivity$$ARouter$$Autowired
                String fileName = parent.getSimpleName() + NAME_OF_AUTOWIRED;
           
				//TypeSpec生成类的辅助类
				/**
				类名：BaseActivity$$ARouter$$Autowired
				类doc："DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER."
				父类：com.alibaba.android.arouter.facade.template.ISyringe
				权限：public
				*/
                TypeSpec.Builder helper = TypeSpec.classBuilder(fileName)
                        .addJavadoc(WARNING_TIPS)
                        .addSuperinterface(ClassName.get(type_ISyringe))
                        .addModifiers(PUBLIC);
				//生成字段属性辅助类
				/**
				字段类型：SerializationService
				字段名：serializationService
				字段属性：private
				*/
                FieldSpec jsonServiceField = FieldSpec.builder(TypeName.get(type_JsonService.asType()), "serializationService", Modifier.PRIVATE).build();
                //将字段添加到类：BaseActivity$$ARouter$$Autowired中
				helper.addField(jsonServiceField);
				
				/**
				给inject方法添加语句：这里parent = BaseActivity
				1.serializationService = ARouter.getInstance().navigation(SerializationService.class);
				2.BaseActivity substitute = (BaseActivity)target;
				*/		
                injectMethodBuilder.addStatement("serializationService = $T.getInstance().navigation($T.class)", ARouterClass, ClassName.get(type_JsonService));
                injectMethodBuilder.addStatement("$T substitute = ($T)target", ClassName.get(parent), ClassName.get(parent));

                /**
				生成方法内部代码，注入属性
				
				*/
                for (Element element : childs) {
				   //获取当前element注解Autowired的属性：
                    Autowired fieldConfig = element.getAnnotation(Autowired.class);
					//获取注解的名称
                    String fieldName = element.getSimpleName().toString();
					//判断是否是iProvider的子类，说明iProvider字段如果使用Autowired注解的话，会单独处理
                    if (types.isSubtype(element.asType(), iProvider)) {  // It's provider
                        if ("".equals(fieldConfig.name())) {    // User has not set service path, then use byType.

                            // Getter
                            injectMethodBuilder.addStatement(
                                    "substitute." + fieldName + " = $T.getInstance().navigation($T.class)",
                                    ARouterClass,
                                    ClassName.get(element.asType())
                            );
                        } else {    // use byName
                            // Getter
                            injectMethodBuilder.addStatement(
                                    "substitute." + fieldName + " = ($T)$T.getInstance().build($S).navigation()",
                                    ClassName.get(element.asType()),
                                    ARouterClass,
                                    fieldConfig.name()
                            );
                        }

                        // Validator 这里如果设置了required为true，则一定要有值，否则会报错
                        if (fieldConfig.required()) {
                            injectMethodBuilder.beginControlFlow("if (substitute." + fieldName + " == null)");
                            injectMethodBuilder.addStatement(
                                    "throw new RuntimeException(\"The field '" + fieldName + "' is null, in class '\" + $T.class.getName() + \"!\")", ClassName.get(parent));
                            injectMethodBuilder.endControlFlow();
                        }
                    } else {    // It's normal intent value
						//普通属性
						/**
						假设fieldName = "name"
						originalValue = "substitute.name"
						statement = "substitute.name = substitute."
						*/
                        String originalValue = "substitute." + fieldName;
                        String statement = "substitute." + fieldName + " = " + buildCastCode(element) + "substitute.";
                        boolean isActivity = false;
						//判断是Activity 则statement += "getIntent()."
                        if (types.isSubtype(parent.asType(), activityTm)) {  // Activity, then use getIntent()
                            isActivity = true;
                            statement += "getIntent().";
						//判断是Fragment 则statement += "getArguments()."
                        } else if (types.isSubtype(parent.asType(), fragmentTm) || types.isSubtype(parent.asType(), fragmentTmV4)) {   // Fragment, then use getArguments()
                            statement += "getArguments().";
                        } else {
							//非Activity和Fragment，其他情况抛异常
                            throw new IllegalAccessException("The field [" + fieldName + "] need autowired from intent, its parent must be activity or fragment!");
                        }
						 //statement = "substitute.name = substitute.getIntent().getExtras() == null ? substitute.name : substitute.getIntent().getExtras()
                        statement = buildStatement(originalValue, statement, typeUtils.typeExchange(element), isActivity, isKtClass(parent));
                        if (statement.startsWith("serializationService.")) {   // Not mortals
                            injectMethodBuilder.beginControlFlow("if (null != serializationService)");
                            injectMethodBuilder.addStatement(
                                    "substitute." + fieldName + " = " + statement,
                                    (StringUtils.isEmpty(fieldConfig.name()) ? fieldName : fieldConfig.name()),
                                    ClassName.get(element.asType())
                            );
                            injectMethodBuilder.nextControlFlow("else");
                            injectMethodBuilder.addStatement(
                                    "$T.e(\"" + Consts.TAG + "\", \"You want automatic inject the field '" + fieldName + "' in class '$T' , then you should implement 'SerializationService' to support object auto inject!\")", AndroidLog, ClassName.get(parent));
                            injectMethodBuilder.endControlFlow();
                        } else {
							//将statement注入到injectMethodBuilder方法中
                            injectMethodBuilder.addStatement(statement, StringUtils.isEmpty(fieldConfig.name()) ? fieldName : fieldConfig.name());
                        }

                        // 添加null判断
                        if (fieldConfig.required() && !element.asType().getKind().isPrimitive()) {  // Primitive wont be check.
                            injectMethodBuilder.beginControlFlow("if (null == substitute." + fieldName + ")");
                            injectMethodBuilder.addStatement(
                                    "$T.e(\"" + Consts.TAG + "\", \"The field '" + fieldName + "' is null, in class '\" + $T.class.getName() + \"!\")", AndroidLog, ClassName.get(parent));
                            injectMethodBuilder.endControlFlow();
                        }
                    }
                }
				 //将方法inject注入到类中
                helper.addMethod(injectMethodBuilder.build());

                //生成java文件
                JavaFile.builder(packageName, helper.build()).build().writeTo(mFiler);

                logger.info(">>> " + parent.getSimpleName() + " has been processed, " + fileName + " has been generated. <<<");
            }

            logger.info(">>> Autowired processor stop. <<<");
        }
    }
    
    /**
     * Categories field, find his papa.
     *
     * @param elements Field need autowired
     */
    private void categories(Set<? extends Element> elements) throws IllegalAccessException {
        if (CollectionUtils.isNotEmpty(elements)) {
            for (Element element : elements) {
				//获取element的父元素：如果是属性，父元素就是类或者接口：TypeElement
                TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
				//如果element属性是PRIVATE，则直接报错，所以对于需要依赖注入的属性，一定不能为private
                if (element.getModifiers().contains(Modifier.PRIVATE)) {
                    throw new IllegalAccessException("The inject fields CAN NOT BE 'private'!!! please check field ["
                            + element.getSimpleName() + "] in class [" + enclosingElement.getQualifiedName() + "]");
                }
				//判断parentAndChild是否包含enclosingElement，第一次循环是空值会走到else分支，第二次才会包含
				//格式：parentAndChild{List<Element>{element1,element2,element3...}}
                if (parentAndChild.containsKey(enclosingElement)) { // Has categries
                    parentAndChild.get(enclosingElement).add(element);
                } else {
                    List<Element> childs = new ArrayList<>();
                    childs.add(element);
                    parentAndChild.put(enclosingElement, childs);
                }
            }

            logger.info("categories finished.");
        }
    }
}


```
通过在编译器使用注解处理器AutowiredProcessor处理后，自动生成了以下文件
`BaseActivity$$ARouter$$Autowired.java`

```java
/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class BaseActivity$$ARouter$$Autowired implements ISyringe {
  private SerializationService serializationService;

  @Override
  public void inject(Object target) {
    serializationService = ARouter.getInstance().navigation(SerializationService.class);
    BaseActivity substitute = (BaseActivity)target;
    substitute.name = substitute.getIntent().getExtras() == null ? substitute.name : substitute.getIntent().getExtras().getString("name", substitute.name);
  }
}
```


生成过程：

1.使用Map<TypeElement, List<Element>> parentAndChild = new HashMap<>()存储所有被Autowired注解的属性
key：每个类的TypeElement
value：当前类TypeElement中所有的Autowired注解的属性字段

2.使用ParameterSpec生成参数

3.使用MethodSpec生成方法：METHOD_INJECT = 'inject'

```java
方法名：inject
方法注解：Override
方法权限：public
方法参数：前面objectParamSpec生成的：Object target
```


4.使用TypeSpec生成类：

```java
类名：BaseActivity$$ARouter$$Autowired
类doc："DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER."
父类：com.alibaba.android.arouter.facade.template.ISyringe
权限：public
```


5.使用addStatement给方法添加语句body

6.将方法注入到帮助类中

```java
 helper.addMethod(injectMethodBuilder.build());   
```

7.写入java文件

```java
JavaFile.builder(packageName, helper.build()).build().writeTo(mFiler);    
```


#### RouteProcessor


```java
@AutoService(Processor.class)
@SupportedAnnotationTypes({ANNOTATION_TYPE_ROUTE, ANNOTATION_TYPE_AUTOWIRED})
//这里表示我们的RouteProcessor可以处理Route和Autowired两种注解
public class RouteProcessor extends BaseProcessor {
    private Map<String, Set<RouteMeta>> groupMap = new HashMap<>(); // ModuleName and routeMeta.
    private Map<String, String> rootMap = new TreeMap<>();  // Map of root metas, used for generate class file in order.

    private TypeMirror iProvider = null;
    private Writer docWriter;       // Writer used for write doc

	//初始化
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
		//这里如果支持generateDoc，则打开docWriter，待写入文件：generateDoc字段由模块中的build.gradle文件传入
        if (generateDoc) {
            try {
                docWriter = mFiler.createResource(
                        StandardLocation.SOURCE_OUTPUT,
                        PACKAGE_OF_GENERATE_DOCS,
                        "arouter-map-of-" + moduleName + ".json"
                ).openWriter();
            } catch (IOException e) {
                logger.error("Create doc writer failed, because " + e.getMessage());
            }
        }
		//获取IPROVIDER的类型TypeMirror
        iProvider = elementUtils.getTypeElement(Consts.IPROVIDER).asType();

        logger.info(">>> RouteProcessor init. <<<");
    }
	
	
	//核心处理api
    /**
     * {@inheritDoc}
     *
     * @param annotations
     * @param roundEnv
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (CollectionUtils.isNotEmpty(annotations)) {
            Set<? extends Element> routeElements = roundEnv.getElementsAnnotatedWith(Route.class);
            try {
                logger.info(">>> Found routes, start... <<<");
				//解析Routes
                this.parseRoutes(routeElements);

            } catch (Exception e) {
                logger.error(e);
            }
            return true;
        }

        return false;
    }
	
    private void parseRoutes(Set<? extends Element> routeElements) throws IOException {
        if (CollectionUtils.isNotEmpty(routeElements)) {
            // prepare the type an so on.

            logger.info(">>> Found routes, size is " + routeElements.size() + " <<<");

            rootMap.clear();
			//获取Activity的TypeMirror
            TypeMirror type_Activity = elementUtils.getTypeElement(ACTIVITY).asType();
			//获取Service的TypeMirror
            TypeMirror type_Service = elementUtils.getTypeElement(SERVICE).asType();
			//获取Fragment的TypeMirror
            TypeMirror fragmentTm = elementUtils.getTypeElement(FRAGMENT).asType();
            TypeMirror fragmentTmV4 = elementUtils.getTypeElement(Consts.FRAGMENT_V4).asType();

            // Interface of ARouter
			//获取IRouteGroup的TypeElement
            TypeElement type_IRouteGroup = elementUtils.getTypeElement(IROUTE_GROUP);
			////获取IProviderGroup的TypeElement
            TypeElement type_IProviderGroup = elementUtils.getTypeElement(IPROVIDER_GROUP);
			//获取RouteMeta的ClassName：权限定名
            ClassName routeMetaCn = ClassName.get(RouteMeta.class);
			//获取RouteType的ClassName：权限定名
            ClassName routeTypeCn = ClassName.get(RouteType.class);

            /*创建Map<String, Class<? extends IRouteGroup>>类型的ParameterizedTypeName
               Build input type, format as :

               ```Map<String, Class<? extends IRouteGroup>>```
             */
            ParameterizedTypeName inputMapTypeOfRoot = ParameterizedTypeName.get(
                    ClassName.get(Map.class),
                    ClassName.get(String.class),
                    ParameterizedTypeName.get(
                            ClassName.get(Class.class),
                            WildcardTypeName.subtypeOf(ClassName.get(type_IRouteGroup))
                    )
            );

            /*创建Map<String, RouteMeta>类型的ParameterizedTypeName

              ```Map<String, RouteMeta>```
             */
            ParameterizedTypeName inputMapTypeOfGroup = ParameterizedTypeName.get(
                    ClassName.get(Map.class),
                    ClassName.get(String.class),
                    ClassName.get(RouteMeta.class)
            );

            /*创建参数类型rootParamSpec，groupParamSpec，providerParamSpec
              Build input param name.
             */
            ParameterSpec rootParamSpec = ParameterSpec.builder(inputMapTypeOfRoot, "routes").build();
            ParameterSpec groupParamSpec = ParameterSpec.builder(inputMapTypeOfGroup, "atlas").build();
            ParameterSpec providerParamSpec = ParameterSpec.builder(inputMapTypeOfGroup, "providers").build();  // Ps. its param type same as groupParamSpec!

            /*创建loadInto方法的MethodSpec
              Build method : 'loadInto'
             */
            MethodSpec.Builder loadIntoMethodOfRootBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .addParameter(rootParamSpec);

            //  Follow a sequence, find out metas of group first, generate java file, then statistics them as root.
			//遍历routeElements所有的path注解对象
            for (Element element : routeElements) {
				//获取对象element的TypeMirror
                TypeMirror tm = element.asType();
				//获取element的注解Route
                Route route = element.getAnnotation(Route.class);
                RouteMeta routeMeta;

                // Activity or Fragment 如果是Activity或者Fragment：根据不同情况创建不同的routeMeta路由元数据
                if (types.isSubtype(tm, type_Activity) || types.isSubtype(tm, fragmentTm) || types.isSubtype(tm, fragmentTmV4)) {
                    // Get all fields annotation by @Autowired
                    Map<String, Integer> paramsType = new HashMap<>();
                    Map<String, Autowired> injectConfig = new HashMap<>();
					//这里是收集所有的Autowired属性参数
                    injectParamCollector(element, paramsType, injectConfig);

                    if (types.isSubtype(tm, type_Activity)) {
                        // Activity
                        logger.info(">>> Found activity route: " + tm.toString() + " <<<");
                        routeMeta = new RouteMeta(route, element, RouteType.ACTIVITY, paramsType);
                    } else {
                        // Fragment
                        logger.info(">>> Found fragment route: " + tm.toString() + " <<<");
                        routeMeta = new RouteMeta(route, element, RouteType.parse(FRAGMENT), paramsType);
                    }

                    routeMeta.setInjectConfig(injectConfig);
                } else if (types.isSubtype(tm, iProvider)) {         // IProvider
                    logger.info(">>> Found provider route: " + tm.toString() + " <<<");
                    routeMeta = new RouteMeta(route, element, RouteType.PROVIDER, null);
                } else if (types.isSubtype(tm, type_Service)) {           // Service
                    logger.info(">>> Found service route: " + tm.toString() + " <<<");
                    routeMeta = new RouteMeta(route, element, RouteType.parse(SERVICE), null);
                } else {
                    throw new RuntimeException("The @Route is marked on unsupported class, look at [" + tm.toString() + "].");
                }
				//收集路由元数据
                categories(routeMeta);
            }
			//创建IProvider注解的loadInto方法
            MethodSpec.Builder loadIntoMethodOfProviderBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .addParameter(providerParamSpec);

            Map<String, List<RouteDoc>> docSource = new HashMap<>();

            // Start generate java source, structure is divided into upper and lower levels, used for demand initialization.
            for (Map.Entry<String, Set<RouteMeta>> entry : groupMap.entrySet()) {
                String groupName = entry.getKey();
				//创建IGroupRouter的loadInto方法
                MethodSpec.Builder loadIntoMethodOfGroupBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                        .addAnnotation(Override.class)
                        .addModifiers(PUBLIC)
                        .addParameter(groupParamSpec);

                List<RouteDoc> routeDocList = new ArrayList<>();

                // 创建 group 方法的 body
                Set<RouteMeta> groupData = entry.getValue();
                for (RouteMeta routeMeta : groupData) {
                    RouteDoc routeDoc = extractDocInfo(routeMeta);

                    ClassName className = ClassName.get((TypeElement) routeMeta.getRawType());
					
                    switch (routeMeta.getType()) {
						//创建PROVIDER的loadInto方法体
                        case PROVIDER:  // Need cache provider's super class
                            List<? extends TypeMirror> interfaces = ((TypeElement) routeMeta.getRawType()).getInterfaces();
                            for (TypeMirror tm : interfaces) {
                                routeDoc.addPrototype(tm.toString());

                                if (types.isSameType(tm, iProvider)) {   // Its implements iProvider interface himself.
                                    // This interface extend the IProvider, so it can be used for mark provider
                                    loadIntoMethodOfProviderBuilder.addStatement(
                                            "providers.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, null, " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                                            (routeMeta.getRawType()).toString(),
                                            routeMetaCn,
                                            routeTypeCn,
                                            className,
                                            routeMeta.getPath(),
                                            routeMeta.getGroup());
                                } else if (types.isSubtype(tm, iProvider)) {
                                    // This interface extend the IProvider, so it can be used for mark provider
                                    loadIntoMethodOfProviderBuilder.addStatement(
                                            "providers.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, null, " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                                            tm.toString(),    // So stupid, will duplicate only save class name.
                                            routeMetaCn,
                                            routeTypeCn,
                                            className,
                                            routeMeta.getPath(),
                                            routeMeta.getGroup());
                                }
                            }
                            break;
                        default:
                            break;
                    }

                    // Make map body for paramsType
                    StringBuilder mapBodyBuilder = new StringBuilder();
                    Map<String, Integer> paramsType = routeMeta.getParamsType();
                    Map<String, Autowired> injectConfigs = routeMeta.getInjectConfig();
                    if (MapUtils.isNotEmpty(paramsType)) {
                        List<RouteDoc.Param> paramList = new ArrayList<>();

                        for (Map.Entry<String, Integer> types : paramsType.entrySet()) {
                            mapBodyBuilder.append("put(\"").append(types.getKey()).append("\", ").append(types.getValue()).append("); ");

                            RouteDoc.Param param = new RouteDoc.Param();
                            Autowired injectConfig = injectConfigs.get(types.getKey());
                            param.setKey(types.getKey());
                            param.setType(TypeKind.values()[types.getValue()].name().toLowerCase());
                            param.setDescription(injectConfig.desc());
                            param.setRequired(injectConfig.required());

                            paramList.add(param);
                        }

                        routeDoc.setParams(paramList);
                    }
                    String mapBody = mapBodyBuilder.toString();
					//创建IGroupRouter的方法体
                    loadIntoMethodOfGroupBuilder.addStatement(
                            "atlas.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, " + (StringUtils.isEmpty(mapBody) ? null : ("new java.util.HashMap<String, Integer>(){{" + mapBodyBuilder.toString() + "}}")) + ", " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                            routeMeta.getPath(),
                            routeMetaCn,
                            routeTypeCn,
                            className,
                            routeMeta.getPath().toLowerCase(),
                            routeMeta.getGroup().toLowerCase());

                    routeDoc.setClassName(className.toString());
                    routeDocList.add(routeDoc);
                }

                // Generate groups 生成IGroupRrouter的子类文件
                String groupFileName = NAME_OF_GROUP + groupName;
                JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                        TypeSpec.classBuilder(groupFileName)
                                .addJavadoc(WARNING_TIPS)
                                .addSuperinterface(ClassName.get(type_IRouteGroup))
                                .addModifiers(PUBLIC)
                                .addMethod(loadIntoMethodOfGroupBuilder.build())
                                .build()
                ).build().writeTo(mFiler);

                logger.info(">>> Generated group: " + groupName + "<<<");
                rootMap.put(groupName, groupFileName);
                docSource.put(groupName, routeDocList);
            }

            if (MapUtils.isNotEmpty(rootMap)) {
                // Generate root meta by group name, it must be generated before root, then I can find out the class of group.
                for (Map.Entry<String, String> entry : rootMap.entrySet()) {
                    loadIntoMethodOfRootBuilder.addStatement("routes.put($S, $T.class)", entry.getKey(), ClassName.get(PACKAGE_OF_GENERATE_FILE, entry.getValue()));
                }
            }

            // Output route doc
            if (generateDoc) {
				//将path关系写入doc
                docWriter.append(JSON.toJSONString(docSource, SerializerFeature.PrettyFormat));
                docWriter.flush();
                docWriter.close();
            }

            // Write provider into disk 写入provider
            String providerMapFileName = NAME_OF_PROVIDER + SEPARATOR + moduleName;
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                    TypeSpec.classBuilder(providerMapFileName)
                            .addJavadoc(WARNING_TIPS)
                            .addSuperinterface(ClassName.get(type_IProviderGroup))
                            .addModifiers(PUBLIC)
                            .addMethod(loadIntoMethodOfProviderBuilder.build())
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Generated provider map, name is " + providerMapFileName + " <<<");

            // Write root meta into disk.写入root meta
            String rootFileName = NAME_OF_ROOT + SEPARATOR + moduleName;
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                    TypeSpec.classBuilder(rootFileName)
                            .addJavadoc(WARNING_TIPS)
                            .addSuperinterface(ClassName.get(elementUtils.getTypeElement(ITROUTE_ROOT)))
                            .addModifiers(PUBLIC)
                            .addMethod(loadIntoMethodOfRootBuilder.build())
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Generated root, name is " + rootFileName + " <<<");
        }
    }

}
```

生成过程：
和上面生成AutoWried过程类似，都是使用javapoet的api生成对应的java文件
    
这里我们需要生成三种文件：
- `ARouter$$Root$$xxx`：xxx是当前模块名的缩写，存储当前模块路由组的信息：value是路由组的类名

```java
 /**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Root$$modulejava implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("m2", ARouter$$Group$$m2.class);
    routes.put("module", ARouter$$Group$$module.class);
    routes.put("test", ARouter$$Group$$test.class);
    routes.put("yourservicegroupname", ARouter$$Group$$yourservicegroupname.class);
  }
}
```

- `ARouter$$Group$$xxx`:xxx是当前路由组的组名，存储一个路由组内路由的信息：内部包含多个路由信息

```java
/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Group$$test implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/test/activity1", RouteMeta.build(RouteType.ACTIVITY, Test1Activity.class, "/test/activity1", "test", new java.util.HashMap<String, Integer>(){{put("ser", 9); put("ch", 5); put("fl", 6); put("dou", 7); put("boy", 0); put("url", 8); put("pac", 10); put("obj", 11); put("name", 8); put("objList", 11); put("map", 11); put("age", 3); put("height", 3); }}, -1, -2147483648));
    atlas.put("/test/activity2", RouteMeta.build(RouteType.ACTIVITY, Test2Activity.class, "/test/activity2", "test", new java.util.HashMap<String, Integer>(){{put("key1", 8); }}, -1, -2147483648));
    atlas.put("/test/activity3", RouteMeta.build(RouteType.ACTIVITY, Test3Activity.class, "/test/activity3", "test", new java.util.HashMap<String, Integer>(){{put("name", 8); put("boy", 0); put("age", 3); }}, -1, -2147483648));
    atlas.put("/test/activity4", RouteMeta.build(RouteType.ACTIVITY, Test4Activity.class, "/test/activity4", "test", null, -1, -2147483648));
    atlas.put("/test/fragment", RouteMeta.build(RouteType.FRAGMENT, BlankFragment.class, "/test/fragment", "test", new java.util.HashMap<String, Integer>(){{put("ser", 9); put("pac", 10); put("ch", 5); put("obj", 11); put("fl", 6); put("name", 8); put("dou", 7); put("boy", 0); put("objList", 11); put("map", 11); put("age", 3); put("height", 3); }}, -1, -2147483648));
    atlas.put("/test/webview", RouteMeta.build(RouteType.ACTIVITY, TestWebview.class, "/test/webview", "test", null, -1, -2147483648));
  }
}
```


- `ARouter$$Providers$$xxx`，xxx是模块名，存储的是当前模块中的IProvider信息，key是IProvider的名称，value是RouteMeta路由元数据


```java
/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Providers$$modulejava implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
    providers.put("com.alibaba.android.arouter.demo.service.HelloService", RouteMeta.build(RouteType.PROVIDER, HelloServiceImpl.class, "/yourservicegroupname/hello", "yourservicegroupname", null, -1, -2147483648));
    providers.put("com.alibaba.android.arouter.facade.service.SerializationService", RouteMeta.build(RouteType.PROVIDER, JsonServiceImpl.class, "/yourservicegroupname/json", "yourservicegroupname", null, -1, -2147483648));
    providers.put("com.alibaba.android.arouter.demo.module1.testservice.SingleService", RouteMeta.build(RouteType.PROVIDER, SingleService.class, "/yourservicegroupname/single", "yourservicegroupname", null, -1, -2147483648));
  }
}
```

还有其他比如拦截器的java文件生成方式就不再描述了，和前面两个注解处理器是一样的原理。
自动生成了这些帮助类之后，在编译器或者运行期，通过调用这些类的`loadInto`方法，可以将路由元信息加载到内存中。

## 总结
本文在开始主要讲解一些注解和注解处理器的前置知识，且带大家自己实现了一个APT自动生成文件的demo，最后讲解下在ARouter中APT是如何再编译器动态生成几种帮助类的。
        
到这里已经是ARouter的第三篇了
        

> 持续输出中。。你的关注和点赞是我最大的动力