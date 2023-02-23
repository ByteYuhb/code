---
theme: smartblue
highlight: a11y-light
---
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
这段时间在写一些`组件化`相关的文章，其中有用到开源库`ARoute`相关知识，查看了下源码，内部使用了`APT`动态生成类的方式,于是就有了这篇文章，记录下自己对APT注解处理器的一些理解。

注解在我们`android`开发和`java`开发中有很多作用，今天我们就来介绍下他的一种高级用法：**注解处理器**

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
可以使用编译期注解动态生成代码，很多优秀的开源库都是使用这个方式：如`Aroute` `ButterKnife`，`GreenDao`，`EventBus3`等

`APT知识储备`：

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

## 总结
APT用法在android的高级中是你一定要去了解的东西，后期学习Aroute源码，对组件化思路的理解都有很大帮助，快乐学习。。gogogo
		
	