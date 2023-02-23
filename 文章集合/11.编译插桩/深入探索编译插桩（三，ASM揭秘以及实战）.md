> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
最近在学习一些关于编译插桩方面的知识，说到编译插桩：大家可以想到的哪些关键字：
`Gradle插件`，`ASM`，`AspectJ`，`AOP`，`JVM字节码`等。

其中`ASM，AspectJ`用于字节码插入操作，而`Gradle插件`用于在编译期传入字节码.class文件和输出重编写后的字节码.class。
而`JVM字节码`则关系我们具体插入代码如何实现。
`AOP则是一种面向切面的编程思想`，如何理解切面：看下面图

![面向AOP.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29f4e87ec7ce4ec5a83ce064018246f8~tplv-k3u1fbpfcp-watermark.image?)

这幅图中，我们可以针对所有方法起始位置和结束位置做一些操作，如`统计方法时常`等操作，`打印方法日志`。
**这里的起始位置和结束位置就是一种切换编程实现，简称AOP。**
同理：对于对象属性字段的访问也可以使用这种方式，如检测所有字段是否含有某种特定的注解，然后对该字段做一些分析统计操作。

- **ASM，AspectJ如何选择？**

- `AspectJ`：
**优点就是使用简单，但是其基于规则，切入点相对固定，做字节码的操作自由度较低，且会生成一些额外包装信息。**

- `ASM`:
**优点是基于字节码的操作，可以说只要对JVM字节码掌握的很好，可以实现任何切面方面的需求。
缺点就是上手难度大，一般开发者很难实现稍微复杂点的需求，**

> 但是话说回来，太复杂的需求大家觉得有必要用`AOP编程`么？

**基于以上几点，本篇文章讲解将由ASM展开。笔者会先讲解一些关于ASM的基本API和基本原理的讲解，最后使用一个Demo来实战下**

## ASM的两种模型

### 对象模型(ASM Tree API)
ASM对象模型使用一个树状图来描述一个类：
每个节点又有子节点，子节点又有子节点，和设计模式中的组合模式类似

![对象模型.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea3541acdf974bf1b9e5dd418e26363b~tplv-k3u1fbpfcp-watermark.image?)

对象模型优缺点：
**学习成本较低，代码量少，事宜处理简单类的修改，不适合复杂场景。**

对象模型的操作纬度
- 1)**获取节点**：获取类，字段，方法节点，注解节点等
- 2)**针对方法节点的操作码**：获取操作码，操作码的替换，删除，插入，输出字节码


#### 1.获取节点

##### 1.1)**获取指定类节点**

使用如下代码获取节点：
```java
ClassNode classNode = new ClassNode();
// 1
ClassReader classReader = new ClassReader(bytes);
// 2
classReader.accept(classNode, 0);
```
**使用ClassReader接收一个class输入字节，然后使用accept方法可以将class字节输出到classNode中，
这样就将class字节输出到了一个ClassNode类节点中。**
ClassNode类结构：


| 类型   | 名称                 | 说明                                   |
| ------ | -------------------- | -------------------------------------- |
| int    | version              | class文件的major版本（编译的java版本） |
| int    | access               | 访问级                                 |
| String | name                 | 类名，采用全地址，如java/lang/String   |
| String | signature            | 签名，通常是null                       |
| String | superName            | 父类类名，采用全地址                   |
| List   | interfaces           | 实现的接口,采用全地址                  |
| String | sourceFile           | 源文件，可能为null                     |
| String | sourceDebug          | debug源，可能为null                    |
| String | outerClass           | 外部类                                 |
| String | outerMethod          | 外部方法                               |
| String | outerMethodDesc      | 外部方法描述（包括方法参数和返回值）   |
| List   | visibleAnnotations   | 可见的注解                             |
| List   | invisibleAnnotations | 不可见的注解                           |
| List   | attrs                | 类的Attribute                          |
| List   | innerClasses         | 类的内部类列表                         |
| List   | fields               | 类的字段列表                           |
| List   | methods              | 类的方法列表                           |

##### 1.2)**获取指定字段的节点**

```java
for(FieldNode fieldNode : (List)classNode.fields) {
    // 1
    if(fieldNode.name.equals("password"))  {
        // 2
        fieldNode.access = Opcodes.ACC_PUBLIC;
    }
}
```
这里将classNode中的一个字段名为password节点的字段权限更改为了public

也可以为classNode添加一个字段：

```java
FieldNode field = new FieldNode(Opcodes.ACC_PUBLIC,"yuhb","B",null,null);
classNode.fields.add(field);
```

这样就为可以为所有或者特定的类添加一个字段：
**public byte yuhb;**

字段节点：

| 类型   | 名称                 | 说明                                                         |
| ------ | -------------------- | ------------------------------------------------------------ |
| int    | access               | 访问级                                                       |
| String | name                 | 字段名                                                       |
| String | signature            | 签名，通常是 null                                            |
| String | desc                 | 类型描述，例如 Ljava/lang/String、D（double）、F（float）Objectvalue初始值，通常为 null |
| List   | visibleAnnotations   | 可见的注解                                                   |
| List   | invisibleAnnotations | 不可见的注解                                                 |
| List   | attrs                | 字段的 Attribute                                             |

##### 1.3)**获取指定方法的节点**

```java
for(MethodNode methodNode : (List)classNode.methods) {
    // 1、判断方法名是否匹配目标方法
    if(methodNode.name.equals("getName")) {
        // 2、进行操作
    }
}
```
**methods 同 fields 一样，也是一个 ArrayList，通过遍历并判断方法名的方式即可匹配到目标方法。**

对于一个方法节点来说，它包含有如下信息：

方法节点包含的信息
| 类型     | 名称                          | 说明                               |
| -------- | ----------------------------- | ---------------------------------- |
| int      | access                        | 访问级                             |
| String   | name                          | 方法名                             |
| String   | desc                          | 方法描述，其包含方法的返回值和参数 |
| String   | signature                     | 签名，通常是null                   |
| List     | exceptions                    | 可能返回的异常列表                 |
| List     | visibleAnnotations            | 可见的注解列表                     |
| List     | invisibleAnnotations          | 不可见的注解列表                   |
| List     | attrs                         | 方法的Attribute列表                |
| Object   | annotation                    | Default默认的注解                  |
| List     | visibleParameterAnnotations   | 可见的参数注解                     |
| List     | invisibleParameterAnnotations | 不可见的参数注解列表               |
| InsnList | instructions                  | 操作码列表                         |
| List     | tryCatchBlock                 | stry-catch块列表                   |
| int      | maxStack                      | 最大操作栈的深度                   |
| int      | maxLocals                     | 最大局部变量区的大小               |
| List     | localVariables                | 本地（局部）变量节点列表           |

**这里我问下大家既然是方法节点，那么方法内部代码存储在哪里面呢？**
> 前面我们讲解Class文件结构的时候，知道在Class文件的方法表中的属性列表会有一个Code的属性，我们的代码就是存储在Code属性列表中。
> 这里我们的ASM也是一样，其将Code属性中的方法代码存储到了方法节点中的`instructions`列表中。这个列表就是ASM操控字节码的地方


下面我们来讲解下操作码。
##### 1.4)**操控操作码**

方法节点中`instructions列表`是用于存储操作码的地方，其中 **每一个元素都代表一行操作码。**	

**ASM 将一行字节码封装为一个 xxxInsnNode（Insn 表示的是 Instruction 的缩写，即指令/操作码），例如 ALOAD/ARestore 指令被封装入变量操作码节点  VarInsnNode，INVOKEVIRTUAL 指令则会被封入方法操作码节点 MethodInsnNode 之中。**

操作码中的`xxxInsnNode`都继承自`AbstractInsnNode`。下面列表列出其所有派生类的情况

**所有的指令码节点说明**
| 名称                      | 说明                                                         | 参数                                                         |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| FieldInsnNode             | 用于 GETFIELD 和 PUTFIELD 之类的字段操作的字节码             | String owner 字段所在的类String name 字段的名称String desc 字段的类型 |
| FrameNode                 | 栈映射帧的对应的帧节点                                       | 待补充                                                       |
| IincInsnNode              | 用于 IINC 变量自加操作的字节码                               | int var：目标局部变量的位置int incr： 要增加的数             |
| InsnNode                  | 一切无参数值操作的字节码，例如 ALOAD_0，DUP（注意不包含 POP） | 无                                                           |
| IntInsnNode               | 用于 BIPUSH、SIPUSH 和 NEWARRAY 这三个直接操作整数的操作     | int operand：操作的整数值                                    |
| InvokeDynamicInsnNode     | 用于 Java7 新增的 INVOKEDYNAMIC 操作的字节码                 | String name：方法名称String desc：方法描述Handle bsm：句柄Object bsmArgs：参数常量 |
| JumpInsnNode              | 用于 IFEQ 或 GOTO 等跳转操作字节码                           | LabelNode lable：目标lable                                   |
| LabelNode                 | 一个用于表示跳转点的 Label 节点                              | 无                                                           |
| LdcInsnNode               | 使用 LDC 加载常量池中引用值并进行插入的字节码                | Object cst：引用值                                           |
| LineNumberNode            | 表示行号的节点                                               | int line：行号LabelNode start：对应的第一个                  |
| LabelLookupSwitchInsnNode | 用于实现 LOOKUPSWITCH 操作的字节码                           | LabelNode dflt：default 块对应的 LableList keys 键列表List labels：对应的 Label 节点列表 |
| MethodInsnNode            | 用于 INVOKEVIRTUAL 等传统方法调用操作的字节码，              | 不适用于 Java7 新增的 INVOKEDYNAMICString owner ：方法所在的类String name ：方法名称String desc：方法描述 |
| MultiANewArrayInsnNode    | 用于 MULTIANEWARRAY 操作的字节码                             | String desc：类型描述int dims：维数                          |
| TableSwitchInsnNode       | 用于实现 TABLESWITCH 操作的字节码                            | int min：键的最小值int max：键的最大值LabelNode dflt：default 块对应的 LableList labels：对应的 Label 节点列表 |
| TypeInsnNode              | 用于实现 NEW、ANEWARRAY 和 CHECKCAST 等类型相关操作的字节码  | String desc：类型VarInsnNode用于实现 ALOAD、ASTORE 等局部变量操作的字节码int var：局部变量 |

下面来看常见的几种对操作码的处理。

- 1.**遍历当前方法中的所有操作码**：
使用下面方法遍历instructions：

```java
ClassNode classNode = new ClassNode();
ClassReader reader = new ClassReader(is);
reader.accept(classNode,0);
List<MethodNode> methodNodeList = classNode.methods;
for(MethodNode methodNode:methodNodeList){
	for(AbstractInsnNode node:methodNode.instructions.toArray()){
		Log.d("TAG",node.toString());
	}
}
```
打印结果：可以看到这里出现了很多操作码，每个操作码都代表了当前方法的某一行代码

```java
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@17520e0b
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@2c974016
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@3eb19446
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@4b4bf90c
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@3854f881
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@2ed5986d
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@1c742a55
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@8e3234e
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@17ec5223
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@1578b643
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@2863003b
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@390bbf2a
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@4c038f62
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@1c99e20f
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@2c4b5b7f
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@48dd5333
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@18f5add5
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@4bafad5e
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@3dc997f7
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@1faf0fb6
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@2a018293
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@1cf20fd0
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@6d754596
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@a402531
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@6c0f4d3d
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@5996d7f7
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@60b5e341
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@6b79dd7e
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@7d090389
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@124f5ab0
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@5d2f0c53
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@7898b8a5
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@4d38e974
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@254ab0b9
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@226dd7
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@3dc2a0e6
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@37bb8d63
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@47042c55
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@545badb7
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@7c36e24a
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@178da83a
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@710d481a
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@78327599
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@33ddfc67
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@496b4e5e
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@52bd92f1
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@3948ab82
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@65aa4608
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@76a0ed5d
[DEBUG][TAG]org.objectweb.asm.tree.LineNumberNode@48b4d362
[DEBUG][TAG]org.objectweb.asm.tree.VarInsnNode@345c754d
[DEBUG][TAG]org.objectweb.asm.tree.MethodInsnNode@3f1ffc8
[DEBUG][TAG]org.objectweb.asm.tree.InsnNode@724a20ee
[DEBUG][TAG]org.objectweb.asm.tree.LabelNode@1c8bd206

```
- 2)**获取操作码的位置**

用下面指令过滤了所有操作码为ALOAD的指令：

```java
List<MethodNode> methodNodeList = classNode.methods;
for(MethodNode methodNode:methodNodeList){
	for(AbstractInsnNode node:methodNode.instructions.toArray()){
		if(node.getOpcode()==Opcodes.ALOAD){
			VarInsnNode varNode = (VarInsnNode) node;
			Log.d("TAG",varNode.toString()+"var:"+varNode.var);
		}
	}
}
```

- 3)**替换操作**

使用下面操作将当前VarInsnNode替换为新的VarInsnNode且操作数+1

```java
List<MethodNode> methodNodeList = classNode.methods;
for(MethodNode methodNode:methodNodeList){
	for(AbstractInsnNode node:methodNode.instructions.toArray()){
		if(node.getOpcode()==Opcodes.ALOAD){
			VarInsnNode varNode = (VarInsnNode) node;
			Log.d("TAG",varNode.toString()+"var:"+varNode.var);
			methodNode
			methodNode.instructions.set(node,new VarInsnNode(Opcodes.ALOAD,varNode.var+1));
			
		}
	}
}

```

- 4)**插入操作**

`InsnList` 主要提供了 **四类** 方法用于插入字节码，如下所示：
InsnList内部使用的是链表的数据结构对字节节点进行存储

1）、`add(AbstractInsnNode insn)`： 将一个操作码添加到 InsnList 的末尾。

2）、`insert(AbstractInsnNode insn)`： 将一个操作码插入到这个 InsnList 的开头。

3）、`insert(AbstractInsnNode insnNode，AbstractInsnNode insn)`： 将一个操作码插入到另一个操作码的下面。

4）、`insertBefore(AbstractInsnNode insnNode，AbstractInsnNode insn)` 将一个操作码插入到另一个操作码的上面

```java
//添加到instructions的末尾
methodNode.instructions.add(new VarInsnNode(Opcodes.ILOAD,2));
//插入到instructions的开头
methodNode.instructions.insert(new VarInsnNode(Opcodes.ILOAD,0) );
//插入到instructions的node节点后面
methodNode.instructions.insert(node,new VarInsnNode(Opcodes.ILOAD,0));
//插入到instructions的node节点前面
methodNode.instructions.insertBefore(node,new VarInsnNode(Opcodes.ILOAD,0));
```

- 5)**删除操作**

methodNode.instructions.remove(node);
直接调用remove方法移除节点。

### ASM事件模型（ASM Core API）

**首先要了解ASM的对象模型是在事件模型的基础上封装而成，所以事件模型更偏向底层，上手难度肯定也更大**

事件模型采用的是`访问者模式`实现：
访问者模式主要用来解决：
> 比如当前有N个元素，每个元素对应不同的访问者有不同的处理方法，于是将所有元素的处理方法抽象为一个接口，
> 外部访问者实现这个接口，然后传入元素处理的核心类中，内部元素需要处理就调用传入的访问者的方法即可。

要理解 ASM 的事件模型，我们就需要对其中的 两个重要成员的工作原理 有较深的了解。它们便是 **类访问者 ClassVisitor 与 类读取（解析）者 ClassReader。**

从字节码的视角中，一个 Java 类由很多组件凝聚而成，而这之中便包括超类、接口、属性、域和方法等等。当我们在使用 ASM 进行操控时，可以将它们视为一个个与之对应的事件。因此 ASM 提供了一个 类访问者 ClassVisitor，以通过它来访问当前类的各个组件，当解析器 ClassReader 依次遇到上述的各个组件时，ClassVisitor 上对应的 visitor 事件处理器方法均会被一一调用。

与类相似，方法也是由多个组件凝聚而成的，其对应着方法属性、注解及编译后的代码（Class 字节码）。ASM 的 MethodVisitor 提供了一种 hook（钩子）机制，以便能够访问方法中的每一个操作码，这样我们便能够对字节码文件进行细粒度地修改。

下面，我们便来一一分析下它们。

#### 1.ClassVisitor
我们在使用ASM处理字节码操作的时候，通常会有下面模板方式：

```java
InputStream is = new FileInputStream(classFile);
// 1
ClassReader classReader = new ClassReader(is);
// 2
ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
// 3
ClassVisitor classVisitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);
// 4
classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);

```

- **步骤1**：将文件输入流传递给ClassReader，包裹住FileInputStream
- **步骤2**：创建一个ClassWriter，其参数 COMPUTE_MAXS 的作用是将自动计算本地变量表最大值和操作数栈最大值的任务托付给了ASM
- **步骤3**：将classWriter传递给TraceClassAdapter对象，包裹住classWriter对象，内部调用其实还是通过classWriter实现
- **步骤4**：调用accept将classVisitor传递给classReader，这里有个EXPAND_FRAMES参数，旨在说明在读取 class 的时候需要同时展开栈映射帧（StackMap Frame），如果我们需要使用自定义的 MethodVisitor 去修改方法中的指令时必须要指定这个参数，。



#### 2.ClassReader
前面分析了，我们模板流程最终会调用ClassReader的accept方法
我们进入这个方法看看：

```java
/**
 * Makes the given visitor visit the Java class of this {@link ClassReader}
 * . This class is the one specified in the constructor (see
 * {@link #ClassReader(byte[]) ClassReader}).
 * 
 * @param classVisitor
 *            the visitor that must visit this class.
 * @param flags
 *            option flags that can be used to modify the default behavior
 *            of this class. See {@link #SKIP_DEBUG}, {@link #EXPAND_FRAMES}
 *            , {@link #SKIP_FRAMES}, {@link #SKIP_CODE}.
 */
public void accept(final ClassVisitor classVisitor, final int flags) {
    accept(classVisitor, new Attribute[0], flags);
}
```


在 accept 方法中又继续调用了 classReader 的另一个 accept 重载方法，如下所示：

```java
public void accept(final ClassVisitor classVisitor,
        final Attribute[] attrs, final int flags) {
    int u = header; // current offset in the class file
    char[] c = new char[maxStringLength]; // buffer used to read strings

    Context context = new Context();
    context.attrs = attrs;
    context.flags = flags;
    context.buffer = c;
    
    // 1、读取类的描述信息，例如 access、name 等等
    int access = readUnsignedShort(u);
    String name = readClass(u + 2, c);
    String superClass = readClass(u + 4, c);
    String[] interfaces = new String[readUnsignedShort(u + 6)];
    u += 8;
    for (int i = 0; i < interfaces.length; ++i) {
        interfaces[i] = readClass(u, c);
        u += 2;
    }
    
    // 2、读取类的属性信息，例如签名 signature、sourceFile 等等。
    String signature = null;
    String sourceFile = null;
    String sourceDebug = null;
    String enclosingOwner = null;
    String enclosingName = null;
    String enclosingDesc = null;
    int anns = 0;
    int ianns = 0;
    int tanns = 0;
    int itanns = 0;
    int innerClasses = 0;
    Attribute attributes = null;
    
    u = getAttributes();
    for (int i = readUnsignedShort(u); i > 0; --i) {
        String attrName = readUTF8(u + 2, c);
        // tests are sorted in decreasing frequency order
        // (based on frequencies observed on typical classes)
        if ("SourceFile".equals(attrName)) {
            sourceFile = readUTF8(u + 8, c);
        } else if ("InnerClasses".equals(attrName)) {
            innerClasses = u + 8;
        } else if ("EnclosingMethod".equals(attrName)) {
            enclosingOwner = readClass(u + 8, c);
            int item = readUnsignedShort(u + 10);
            if (item != 0) {
                enclosingName = readUTF8(items[item], c);
                enclosingDesc = readUTF8(items[item] + 2, c);
            }
        } else if (SIGNATURES && "Signature".equals(attrName)) {
            signature = readUTF8(u + 8, c);
        } else if (ANNOTATIONS
                && "RuntimeVisibleAnnotations".equals(attrName)) {
            anns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeVisibleTypeAnnotations".equals(attrName)) {
            tanns = u + 8;
        } else if ("Deprecated".equals(attrName)) {
            access |= Opcodes.ACC_DEPRECATED;
        } else if ("Synthetic".equals(attrName)) {
            access |= Opcodes.ACC_SYNTHETIC
                    | ClassWriter.ACC_SYNTHETIC_ATTRIBUTE;
        } else if ("SourceDebugExtension".equals(attrName)) {
            int len = readInt(u + 4);
            sourceDebug = readUTF(u + 8, len, new char[len]);
        } else if (ANNOTATIONS
                && "RuntimeInvisibleAnnotations".equals(attrName)) {
            ianns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeInvisibleTypeAnnotations".equals(attrName)) {
            itanns = u + 8;
        } else if ("BootstrapMethods".equals(attrName)) {
            int[] bootstrapMethods = new int[readUnsignedShort(u + 8)];
            for (int j = 0, v = u + 10; j < bootstrapMethods.length; j++) {
                bootstrapMethods[j] = v;
                v += 2 + readUnsignedShort(v + 2) << 1;
            }
            context.bootstrapMethods = bootstrapMethods;
        } else {
            Attribute attr = readAttribute(attrs, attrName, u + 8,
                    readInt(u + 4), c, -1, null);
            if (attr != null) {
                attr.next = attributes;
                attributes = attr;
            }
        }
        u += 6 + readInt(u + 4);
    }
    
    // 3、访问类的描述信息
    classVisitor.visit(readInt(items[1] - 7), access, name, signature,
            superClass, interfaces);
    
    // 4、访问源码和 debug 信息
    if ((flags & SKIP_DEBUG) == 0
            && (sourceFile != null || sourceDebug != null)) {
        classVisitor.visitSource(sourceFile, sourceDebug);
    }
    
    // 5、访问外部类
    if (enclosingOwner != null) {
        classVisitor.visitOuterClass(enclosingOwner, enclosingName,
                enclosingDesc);
    }
    
    // 6、访问类注解和类型注解
    if (ANNOTATIONS && anns != 0) {
        for (int i = readUnsignedShort(anns), v = anns + 2; i > 0; --i) {
            v = readAnnotationValues(v + 2, c, true,
                    classVisitor.visitAnnotation(readUTF8(v, c), true));
        }
    }
    if (ANNOTATIONS && ianns != 0) {
        for (int i = readUnsignedShort(ianns), v = ianns + 2; i > 0; --i) {
            v = readAnnotationValues(v + 2, c, true,
                    classVisitor.visitAnnotation(readUTF8(v, c), false));
        }
    }
    if (ANNOTATIONS && tanns != 0) {
        for (int i = readUnsignedShort(tanns), v = tanns + 2; i > 0; --i) {
            v = readAnnotationTarget(context, v);
            v = readAnnotationValues(v + 2, c, true,
                    classVisitor.visitTypeAnnotation(context.typeRef,
                            context.typePath, readUTF8(v, c), true));
        }
    }
    if (ANNOTATIONS && itanns != 0) {
        for (int i = readUnsignedShort(itanns), v = itanns + 2; i > 0; --i) {
            v = readAnnotationTarget(context, v);
            v = readAnnotationValues(v + 2, c, true,
                    classVisitor.visitTypeAnnotation(context.typeRef,
                            context.typePath, readUTF8(v, c), false));
        }
    }
    
    // 7、访问类的属性
    while (attributes != null) {
        Attribute attr = attributes.next;
        attributes.next = null;
        classVisitor.visitAttribute(attributes);
        attributes = attr;
    }
    
    // 8、访问内部类
    if (innerClasses != 0) {
        int v = innerClasses + 2;
        for (int i = readUnsignedShort(innerClasses); i > 0; --i) {
            classVisitor.visitInnerClass(readClass(v, c),
                    readClass(v + 2, c), readUTF8(v + 4, c),
                    readUnsignedShort(v + 6));
            v += 8;
        }
    }
    
    // 9、访问字段和方法
    u = header + 10 + 2 * interfaces.length;
    for (int i = readUnsignedShort(u - 2); i > 0; --i) {
        u = readField(classVisitor, context, u);
    }
    u += 2;
    for (int i = readUnsignedShort(u - 2); i > 0; --i) {
        u = readMethod(classVisitor, context, u);
    }
    
    // 访问当前类结束时调用
    classVisitor.visitEnd();
}
```


**首先，在 classReader 实例的 accept 方法中的

注释1和注释2处，我们会 先开始进行类相关的字节码解析的工作：读**取了类的描述和属性信息**。接着，

在注释3 ~ 注释8处，我们**调用了 classVisitor 一系列的 visitxxx 方法访问 classReader 解析完字节码后保存在内存的信息**。然后，

在注释9处，分别调用了 **readField 方法和 readMethod** 方法去访问类中的方法和字段。最后，调用 classVisitor 的 visitEnd 标识已访问结束。

- 1.**访问字段**

readField

```java
/**
 * Reads a field and makes the given visitor visit it.
 * 
 * @param classVisitor
 *            the visitor that must visit the field.
 * @param context
 *            information about the class being parsed.
 * @param u
 *            the start offset of the field in the class file.
 * @return the offset of the first byte following the field in the class.
 */
private int readField(final ClassVisitor classVisitor,
        final Context context, int u) {
    // 1、读取字段的描述信息
    char[] c = context.buffer;
    int access = readUnsignedShort(u);
    String name = readUTF8(u + 2, c);
    String desc = readUTF8(u + 4, c);
    u += 6;

    // 2、读取字段的属性
    String signature = null;
    int anns = 0;
    int ianns = 0;
    int tanns = 0;
    int itanns = 0;
    Object value = null;
    Attribute attributes = null;

    for (int i = readUnsignedShort(u); i > 0; --i) {
        String attrName = readUTF8(u + 2, c);
        // tests are sorted in decreasing frequency order
        // (based on frequencies observed on typical classes)
        if ("ConstantValue".equals(attrName)) {
            int item = readUnsignedShort(u + 8);
            value = item == 0 ? null : readConst(item, c);
        } else if (SIGNATURES && "Signature".equals(attrName)) {
            signature = readUTF8(u + 8, c);
        } else if ("Deprecated".equals(attrName)) {
            access |= Opcodes.ACC_DEPRECATED;
        } else if ("Synthetic".equals(attrName)) {
            access |= Opcodes.ACC_SYNTHETIC
                    | ClassWriter.ACC_SYNTHETIC_ATTRIBUTE;
        } else if (ANNOTATIONS
                && "RuntimeVisibleAnnotations".equals(attrName)) {
            anns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeVisibleTypeAnnotations".equals(attrName)) {
            tanns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeInvisibleAnnotations".equals(attrName)) {
            ianns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeInvisibleTypeAnnotations".equals(attrName)) {
            itanns = u + 8;
        } else {
            Attribute attr = readAttribute(context.attrs, attrName, u + 8,
                    readInt(u + 4), c, -1, null);
            if (attr != null) {
                attr.next = attributes;
                attributes = attr;
            }
        }
        u += 6 + readInt(u + 4);
    }
    u += 2;

    // 3、访问字段的声明
    FieldVisitor fv = classVisitor.visitField(access, name, desc,
            signature, value);
    if (fv == null) {
        return u;
    }

    // 4、访问字段的注解和类型注解
    if (ANNOTATIONS && anns != 0) {
        for (int i = readUnsignedShort(anns), v = anns + 2; i > 0; --i) {
            v = readAnnotationValues(v + 2, c, true,
                    fv.visitAnnotation(readUTF8(v, c), true));
        }
    }
    if (ANNOTATIONS && ianns != 0) {
        for (int i = readUnsignedShort(ianns), v = ianns + 2; i > 0; --i) {
            v = readAnnotationValues(v + 2, c, true,
                    fv.visitAnnotation(readUTF8(v, c), false));
        }
    }
    if (ANNOTATIONS && tanns != 0) {
        for (int i = readUnsignedShort(tanns), v = tanns + 2; i > 0; --i) {
            v = readAnnotationTarget(context, v);
            v = readAnnotationValues(v + 2, c, true,
                    fv.visitTypeAnnotation(context.typeRef,
                            context.typePath, readUTF8(v, c), true));
        }
    }
    if (ANNOTATIONS && itanns != 0) {
        for (int i = readUnsignedShort(itanns), v = itanns + 2; i > 0; --i) {
            v = readAnnotationTarget(context, v);
            v = readAnnotationValues(v + 2, c, true,
                    fv.visitTypeAnnotation(context.typeRef,
                            context.typePath, readUTF8(v, c), false));
        }
    }

    // 5、访问字段的属性
    while (attributes != null) {
        Attribute attr = attributes.next;
        attributes.next = null;
        fv.visitAttribute(attributes);
        attributes = attr;
    }

    // 访问字段结束时调用
    fv.visitEnd();

    return u;
}
```
同读取类信息的时候类似，首先，

在注释1和注释2处，会 先开始进行字段相关的字节码解析的工作：**读取了字段的描述和属性信息**。然后，

在注释3 ~ 注释5处 **按顺序访问了字段的描述、注解、类型注解及其属性信息**。最后，调用了 FieldVisitor 实例的 visitEnd 方法结束了字段信息的访问。

- 2.**访问方法**

readMethod

```java
/**
 * Reads a method and makes the given visitor visit it.
 * 
 * @param classVisitor
 *            the visitor that must visit the method.
 * @param context
 *            information about the class being parsed.
 * @param u
 *            the start offset of the method in the class file.
 * @return the offset of the first byte following the method in the class.
 */
private int readMethod(final ClassVisitor classVisitor,
        final Context context, int u) {
    // 1、读取方法描述信息
    char[] c = context.buffer;
    context.access = readUnsignedShort(u);
    context.name = readUTF8(u + 2, c);
    context.desc = readUTF8(u + 4, c);
    u += 6;

    // 2、读取方法属性信息
    int code = 0;
    int exception = 0;
    String[] exceptions = null;
    String signature = null;
    int methodParameters = 0;
    int anns = 0;
    int ianns = 0;
    int tanns = 0;
    int itanns = 0;
    int dann = 0;
    int mpanns = 0;
    int impanns = 0;
    int firstAttribute = u;
    Attribute attributes = null;

    for (int i = readUnsignedShort(u); i > 0; --i) {
        String attrName = readUTF8(u + 2, c);
        // tests are sorted in decreasing frequency order
        // (based on frequencies observed on typical classes)
        if ("Code".equals(attrName)) {
            if ((context.flags & SKIP_CODE) == 0) {
                code = u + 8;
            }
        } else if ("Exceptions".equals(attrName)) {
            exceptions = new String[readUnsignedShort(u + 8)];
            exception = u + 10;
            for (int j = 0; j < exceptions.length; ++j) {
                exceptions[j] = readClass(exception, c);
                exception += 2;
            }
        } else if (SIGNATURES && "Signature".equals(attrName)) {
            signature = readUTF8(u + 8, c);
        } else if ("Deprecated".equals(attrName)) {
            context.access |= Opcodes.ACC_DEPRECATED;
        } else if (ANNOTATIONS
                && "RuntimeVisibleAnnotations".equals(attrName)) {
            anns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeVisibleTypeAnnotations".equals(attrName)) {
            tanns = u + 8;
        } else if (ANNOTATIONS && "AnnotationDefault".equals(attrName)) {
            dann = u + 8;
        } else if ("Synthetic".equals(attrName)) {
            context.access |= Opcodes.ACC_SYNTHETIC
                    | ClassWriter.ACC_SYNTHETIC_ATTRIBUTE;
        } else if (ANNOTATIONS
                && "RuntimeInvisibleAnnotations".equals(attrName)) {
            ianns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeInvisibleTypeAnnotations".equals(attrName)) {
            itanns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeVisibleParameterAnnotations".equals(attrName)) {
            mpanns = u + 8;
        } else if (ANNOTATIONS
                && "RuntimeInvisibleParameterAnnotations".equals(attrName)) {
            impanns = u + 8;
        } else if ("MethodParameters".equals(attrName)) {
            methodParameters = u + 8;
        } else {
            Attribute attr = readAttribute(context.attrs, attrName, u + 8,
                    readInt(u + 4), c, -1, null);
            if (attr != null) {
                attr.next = attributes;
                attributes = attr;
            }
        }
        u += 6 + readInt(u + 4);
    }
    u += 2;

    // 3、访问方法描述信息
    MethodVisitor mv = classVisitor.visitMethod(context.access,
            context.name, context.desc, signature, exceptions);
    if (mv == null) {
        return u;
    }

    /*
     * if the returned MethodVisitor is in fact a MethodWriter, it means
     * there is no method adapter between the reader and the writer. If, in
     * addition, the writers constant pool was copied from this reader
     * (mw.cw.cr == this), and the signature and exceptions of the method
     * have not been changed, then it is possible to skip all visit events
     * and just copy the original code of the method to the writer (the
     * access, name and descriptor can have been changed, this is not
     * important since they are not copied as is from the reader).
     */
    if (WRITER && mv instanceof MethodWriter) {
        MethodWriter mw = (MethodWriter) mv;
        if (mw.cw.cr == this && signature == mw.signature) {
            boolean sameExceptions = false;
            if (exceptions == null) {
                sameExceptions = mw.exceptionCount == 0;
            } else if (exceptions.length == mw.exceptionCount) {
                sameExceptions = true;
                for (int j = exceptions.length - 1; j >= 0; --j) {
                    exception -= 2;
                    if (mw.exceptions[j] != readUnsignedShort(exception)) {
                        sameExceptions = false;
                        break;
                    }
                }
            }
            if (sameExceptions) {
                /*
                 * we do not copy directly the code into MethodWriter to
                 * save a byte array copy operation. The real copy will be
                 * done in ClassWriter.toByteArray().
                 */
                mw.classReaderOffset = firstAttribute;
                mw.classReaderLength = u - firstAttribute;
                return u;
            }
        }
    }

    // 4、访问方法参数信息
    if (methodParameters != 0) {
        for (int i = b[methodParameters] & 0xFF, v = methodParameters + 1; i > 0; --i, v = v + 4) {
            mv.visitParameter(readUTF8(v, c), readUnsignedShort(v + 2));
        }
    }

    // 5、访问方法的注解信息
    if (ANNOTATIONS && dann != 0) {
        AnnotationVisitor dv = mv.visitAnnotationDefault();
        readAnnotationValue(dann, c, null, dv);
        if (dv != null) {
            dv.visitEnd();
        }
    }
    if (ANNOTATIONS && anns != 0) {
        for (int i = readUnsignedShort(anns), v = anns + 2; i > 0; --i) {
            v = readAnnotationValues(v + 2, c, true,
                    mv.visitAnnotation(readUTF8(v, c), true));
        }
    }
    if (ANNOTATIONS && ianns != 0) {
        for (int i = readUnsignedShort(ianns), v = ianns + 2; i > 0; --i) {
            v = readAnnotationValues(v + 2, c, true,
                    mv.visitAnnotation(readUTF8(v, c), false));
        }
    }
    if (ANNOTATIONS && tanns != 0) {
        for (int i = readUnsignedShort(tanns), v = tanns + 2; i > 0; --i) {
            v = readAnnotationTarget(context, v);
            v = readAnnotationValues(v + 2, c, true,
                    mv.visitTypeAnnotation(context.typeRef,
                            context.typePath, readUTF8(v, c), true));
        }
    }
    if (ANNOTATIONS && itanns != 0) {
        for (int i = readUnsignedShort(itanns), v = itanns + 2; i > 0; --i) {
            v = readAnnotationTarget(context, v);
            v = readAnnotationValues(v + 2, c, true,
                    mv.visitTypeAnnotation(context.typeRef,
                            context.typePath, readUTF8(v, c), false));
        }
    }
    if (ANNOTATIONS && mpanns != 0) {
        readParameterAnnotations(mv, context, mpanns, true);
    }
    if (ANNOTATIONS && impanns != 0) {
        readParameterAnnotations(mv, context, impanns, false);
    }

    // 6、访问方法的属性信息
    while (attributes != null) {
        Attribute attr = attributes.next;
        attributes.next = null;
        mv.visitAttribute(attributes);
        attributes = attr;
    }

    // 7、访问方法代码对应的字节码信息
    if (code != 0) {
        mv.visitCode();
        readCode(mv, context, code);
    }

    // 8、visits the end of the method
    mv.visitEnd();

    return u;
}
```

**同类和字段的读取、访问套路一样**，

首先，在**注释1和注释2**处，会 先开始进行方法相关的字节码解析的工作：读取了方法的描述和属性信息。

然后，在**注释3 ~ 注释7**处 按顺序访问了方法的描述、参数、注解、属性、方法代码对应的字节码信息。需要注意的是，在 readCode 方法中，也是先读取了方法内部代码的字节码信息，例如头部、属性等等，然后，便会访问对应的指令集。

最后，在**注释8处** 调用了 MethodVisitor 实例的 visitEnd 方法结束了方法信息的访问。

从以上对 ClassVisitor 与 ClassReader 的分析看来，ClassVisitor 被定义为了一个能接收并解析 ClassReader 传入信息的类。当在 accpet 方法中 ClassVisitor 访问 ClassReader 时，ClassReader 便会先开始字节码的解析工作，并将保存在内存中的结果源源不断地通过调用各种 visitxxx 方法传入到 ClassVisitor 之中。

需要注意的是，其中 
> 只有 visit 这个方法一定会被调用一次，因为它 获取了类头部的描述信息，显然易见，它必不可少，而对于其它的 visitxxx 方法来说都不能确定。例如其中的 visitMethod 方法，只有当 ClassReader 解析出一个方法的字节码时，才会调用一次 visitMethod 方法，并由此生成一个方法访问者 MethodVisitor 的实例。

然后，这个 MethodVisitor 的实例便会同 ClassVisitor 一样开始访问当前方法的属性信息，对于 ClassVisitor 来说，它只处理和类相关的事，而方法的事情被外包给了 MethodVisitor 进行处理。这正是访问者的一大优势：**将访问一个复杂事物的职责通过各个不同类型但又相互关联的访问者分割开来。**

由前可知，对象模型是事件模型的一个封装。其中的 ClassNode 其实就是 ClassVisitor 的一个子类，它负责将 ClassReader 传进来的信息进行分类储存。同样，MethodNode 也是 MethodVisitor 的一个子类，它负责将 ClassReader 传进来的操作码指令信息连接成一个列表并保存其中

而 ClassWriter 也是 ClassVisitor 的一个子类，但是，它并不会储存信息，而是马上会将传入的信息转译成字节码，并在之后随时输出它们。对于 ClassReader 这个被访问者来说，它负责读取我们传入的类文件中的字节流数据，并提供解析流中包含的一切类属性信息的操作。

最后，为了更进一步地将我们上面所讲解的 ClassReader 与 ClassVisitor 的工作机制更加形象化，这里借用 hakugyokurou 的一张流程图用于回顾梳理，如下所示：
注意：第二个"实例化，通过构造函数..."需要去掉

 **小结**
- 1）、**ClassReader**：用于读取已经编译好的 .class 文件。
- 2）、**ClassWriter**：用于重新构建编译后的类，如修改类名、属性以及方法，也可以生成新的类的字节码文件。
- 3）、各种 Visitor 类：如上所述，Core API 根据字节码从上到下依次处理，对于字节码文件中不同的区域有不同的 Visitor，比如用于访问方法的 MethodVisitor、用于访问类变量的 FieldVisitor、用于访问注解的 AnnotationVisitor 等等。为了实现 AOP，其重点是要灵活运用 MethodVisitor。


## 实战训练
在实战之前我们这里先提下一个工具：`ASM Bytecode Viewer`

![ASMBytecodeViewer.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4d722f2a3be48f695fa4ea3d221a4c0~tplv-k3u1fbpfcp-watermark.image?)
这个工具可以给我们提供需要插入的字节码的ASM操作码

什么意思呢？我们在实现类访问者的时候，**需要实现一系列 visitXXXXInsn() 方法、如果要手写每行的ASM的代码，需要对ASM和JVM字节码有很高的要求，
于是这里的ASM Bytecode Viewer工具就是帮助我们实现这个繁琐的转换过程**.

在AS中安装好ASM Bytecode Viewer插件后，重启就可以使用我们的ASM Bytecode Viewer插件了。
插件实现的功能如图：

![Viewer效果.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d51efa03b2134deb872a12712fad79ba~tplv-k3u1fbpfcp-watermark.image?)
还可以对比前后两次版本差异

![对比.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58d5163871bb42a8bb64167d1448a849~tplv-k3u1fbpfcp-watermark.image?)
然后就可以根据前后两次差距在类访问的时候，写入差异元素即可

我们使用两个Demo来测试ASM的事件模型

### 1.在所有方法前面插入日志统计方法耗时：

要实现字节码插庄需要有几个步骤
- 1.自定义Gradle Plugin，并注入一个Transform。
- 2.将Transform挂接到Gradle生命周期中
- 3.遍历所有的class文件和jar包中的class文件
- 4.使用ASM修改字节码，并写入新的class文件中
- 5.Gradle构建工具使用新的class文件打包到dex文件中

#### 步骤1：创建一个ASMPlugin用于注入我们的ASMTraceTransform

```java
class ASMPlugin implements Plugin<Project> {
    private static final String TAG = "ASMPlugin"

    @Override
    void apply(Project project) {

        if (!project.plugins.hasPlugin('com.android.application')) {
            throw new GradleException('ASM Plugin, Android Application plugin required')
        }

        project.afterEvaluate {
            def android = project.extensions.android
            android.applicationVariants.all { variant ->
                ASMTraceTransform.inject(project, variant)
            }
        }
    }
}
```

#### 步骤2：将Transform挂接到Gradle生命周期中

这里使用：为每个变种的afterEvaluate阶段注入了一个ASMTraceTransform

```java
project.afterEvaluate {
	def android = project.extensions.android
	android.applicationVariants.all { variant ->
		ASMTraceTransform.inject(project, variant)
	}
}

public static void inject(Project project, def variant) {

	String hackTransformTaskName = getTransformTaskName(
			 "",
			"",variant.name
	)

	String hackTransformTaskNameForWrapper = getTransformTaskName(
			 "",
			"Builder",variant.name
	)

	project.logger.info("prepare inject dex transform :" + hackTransformTaskName +" hackTransformTaskNameForWrapper:"+hackTransformTaskNameForWrapper)

	project.getGradle().getTaskGraph().addTaskExecutionGraphListener(new TaskExecutionGraphListener() {
		@Override
		public void graphPopulated(TaskExecutionGraph taskGraph) {
			for (Task task : taskGraph.getAllTasks()) {
				if ((task.name.equalsIgnoreCase(hackTransformTaskName) || task.name.equalsIgnoreCase(hackTransformTaskNameForWrapper))
						&& !(((TransformTask) task).getTransform() instanceof ASMTraceTransform)) {
					project.logger.warn("find dex transform. transform class: " + task.transform.getClass() + " . task name: " + task.name)
					project.logger.info("variant name: " + variant.name)
					Field field = TransformTask.class.getDeclaredField("transform")
					field.setAccessible(true)
					field.set(task, new ASMTraceTransform(project, variant, task.transform))
					project.logger.warn("transform class after hook: " + task.transform.getClass())
					break
				}
			}
		}
	})
}
```

获取我们需要hack的TransformTask名称：
`transformClassesWithDexFordebug` 或者`transformClassesWithDexBuilderFordebug`

然后使用反射:将我们的ASMTraceTransform替换为TransformTask中的transform字段属性,这样任务在执行到transformClassesWithDexFordebug或者transformClassesWithDexBuilderFordebug的时候就会执行当前ASMTraceTransform的transform方法

```java
field.set(task, new ASMTraceTransform(project, variant, task.transform))
```

#### 步骤3：遍历所有的class文件和jar包中的class文件
来看ASMTraceTransform的transform方法

```java
public void transform(TransformInvocation transformInvocation){
	...
	//这里将输入的class文件进行路径复制
	transformInvocation.inputs.each { TransformInput input ->
		input.directoryInputs.each { DirectoryInput dirInput ->
			collectAndIdentifyDir(scrInputMap, dirInput, rootOutput, isIncremental)
		}
		input.jarInputs.each { JarInput jarInput ->
			if (jarInput.getStatus() != Status.REMOVED) {
				collectAndIdentifyJar(jarInputMap, scrInputMap, jarInput, rootOutput, isIncremental)
			}
		}
	}
	
	MethodTracer methodTracer = new MethodTracer()
	//这个方法内部会对字节码进行校验
	methodTracer.trace(scrInputMap, jarInputMap)
	origTransform.transform(transformInvocation)
	...

}
```

#### 步骤4：使用ASM修改字节码，并写入新的class文件中
MethodTracer内部会对所有class文件进行遍历操作：修改字节码

```java
private void innerTraceMethodFromSrc(File input, File output) {
	is = new FileInputStream(classFile);
	//核心方法ASMCode.run
	ClassWriter classWriter = ASMCode.run(is);
	if (output.isDirectory()) {
		os = new FileOutputStream(changedFileOutput);
	} else {
		os = new FileOutputStream(output);
	}
	os.write(classWriter.toByteArray());
}
run方法:这就是前面说的模板方法
public static ClassWriter run(InputStream is) throws IOException {
	ClassReader classReader = new ClassReader(is);
	ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
	ClassVisitor classVisitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);
	classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);
	return classWriter;
}
这里使用了TraceClassAdapter，也是继承ClassVisitor
public static class TraceClassAdapter extends ClassVisitor {

	private String className;

	TraceClassAdapter(int i, ClassVisitor classVisitor) {
		super(i, classVisitor);
	}


	@Override
	public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
		super.visit(version, access, name, signature, superName, interfaces);
		this.className = name;

	}

	@Override
	public void visitInnerClass(final String s, final String s1, final String s2, final int i) {
		super.visitInnerClass(s, s1, s2, i);
	}

	@Override
	public MethodVisitor visitMethod(int access, String name, String desc,
									 String signature, String[] exceptions) {

		MethodVisitor methodVisitor = cv.visitMethod(access, name, desc, signature, exceptions);
		return new TraceMethodAdapter(api, methodVisitor, access, name, desc, this.className);
	}


	@Override
	public void visitEnd() {
		super.visitEnd();
	}
}

```

**主要来看visitMethod方法**：
使用了一个包裹类TraceMethodAdapter将methodVisitor封装在其内部

```java
public static class TraceMethodAdapter extends AdviceAdapter {

	private final String methodName;
	private final String className;
	private boolean find = false;


	protected TraceMethodAdapter(int api, MethodVisitor mv, int access, String name, String desc, String className) {
		super(api, mv, access, name, desc);
		this.className = className;
		this.methodName = name;
	}

	@Override
	public void visitTypeInsn(int opcode, String s) {
		super.visitTypeInsn(opcode, s);
	}

	@Override
	public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {
		super.visitMethodInsn(opcode, owner, name, desc, itf);
	}

	private int timeLocalIndex = 0;

	@Override
	protected void onMethodEnter() {
		mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
		timeLocalIndex = newLocal(Type.LONG_TYPE); //this is a function promotes by LocalVariablesSorter ,you can use old LocalVariables
		mv.visitVarInsn(LSTORE, timeLocalIndex);
	}

	@Override
	protected void onMethodExit(int opcode) {
		mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
		mv.visitVarInsn(LLOAD, timeLocalIndex);
		mv.visitInsn(LSUB);//its on the top of stack table
		mv.visitVarInsn(LSTORE, timeLocalIndex);//you will use this value a time later,so you can store it to the LocalVariables info table


		int stringBuilderIndex = newLocal(Type.getType("java/lang/StringBuilder"));
		mv.visitTypeInsn(Opcodes.NEW, "java/lang/StringBuilder");
		mv.visitInsn(Opcodes.DUP);
		mv.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
		mv.visitVarInsn(Opcodes.ASTORE, stringBuilderIndex);//store the stringbuilder of the stack top avoid to can not find it
		mv.visitVarInsn(Opcodes.ALOAD, stringBuilderIndex);
		mv.visitLdcInsn(className + "." + methodName + " time:");
		mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
		mv.visitInsn(Opcodes.POP);
		mv.visitVarInsn(Opcodes.ALOAD, stringBuilderIndex);
		mv.visitVarInsn(Opcodes.LLOAD, timeLocalIndex);
		mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);
		mv.visitInsn(Opcodes.POP);
		mv.visitLdcInsn("Geek");
		mv.visitVarInsn(Opcodes.ALOAD, stringBuilderIndex);
		mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
		mv.visitMethodInsn(Opcodes.INVOKESTATIC, "android/util/Log", "d", "(Ljava/lang/String;Ljava/lang/String;)I", false);
		//caution：Log.d has return value ,should pop
		mv.visitInsn(Opcodes.POP);
		//After inserting the bytecode, it is necessary to ensure that the stack is clean and does not affect the original logic. Otherwise, an exception will occur, and it will also affect the processing of bytecode by other frameworks.
	}
}

```

`onMethodEnter`:**在方法开始位置插入字节码**

`onMethodExit`：**在方法结束位置插入字节码**

这个字节码就是前面使用ASM Bytecode Viewer生成的差异代码
![对比.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a82e15481c24d2e934f223a365cb978~tplv-k3u1fbpfcp-watermark.image?)


来看下生成的字节码：

```java
public class SampleApplication extends Application {
    public SampleApplication() {
        long var1 = System.currentTimeMillis();
        var1 = System.currentTimeMillis() - var1;
        StringBuilder var3 = new StringBuilder();
        var3.append("com/sample/asm/SampleApplication.<init> time:");
        var3.append(var1);
        Log.d("Geek", var3.toString());
    }

    public void onCreate() {
        long var1 = System.currentTimeMillis();
        super.onCreate();
        var1 = System.currentTimeMillis() - var1;
        StringBuilder var3 = new StringBuilder();
        var3.append("com/sample/asm/SampleApplication.onCreate time:");
        var3.append(var1);
        Log.d("Geek", var3.toString());
    }
	...
}
```

可以看到其在所有的类的所有方法中都插入了以下代码：

```java
long var1 = System.currentTimeMillis();
var1 = System.currentTimeMillis() - var1;
StringBuilder var3 = new StringBuilder();
var3.append("com/sample/asm/SampleApplication.onCreate time:");
var3.append(var1);
Log.d("Geek", var3.toString());

```
### 2.将类中所有的Thread替换为自定义的CustomThread
这个只需要改变TraceMethodAdapter中的访问方法即可，如下所示：

```java
private final String methodName;
private final String className;
// 标识是否遇到了 new 指令
private boolean find = false;

protected TraceMethodAdapter(int api, MethodVisitor mv, int access, String name, String desc, String className) {
    super(api, mv, access, name, desc);
    this.className = className;
    this.methodName = name;
}

@Override
public void visitTypeInsn(int opcode, String s) {
    // 方法可以像类一样就行转换，例如，通过使用一个方法适配器来转发 
    // 那些带有修改的方法调用:改变参数可以被用来变更指令，不转发
    // 某个方法调用可以删除一 个指令，插入新的调用可以添加新的指令
    if (opcode == Opcodes.NEW && "java/lang/Thread".equals(s)) {
        // 遇到 new 指令
        find = true;
        mv.visitTypeInsn(Opcodes.NEW, "com/sample/asm/CustomThread");
            return;
    }
    super.visitTypeInsn(opcode, s);
}

@Override
public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {
    //需要排查 CustomThread 自己
    if ("java/lang/Thread".equals(owner) && !className.equals("com/sample/asm/CustomThread") && opcode == Opcodes.INVOKESPECIAL && find) {
        find = false;
        mv.visitMethodInsn(opcode, "com/sample/asm/CustomThread", name, desc, itf);
        Log.e("asmcode", "className:%s, method:%s, name:%s", className, methodName, name);
            return;
    }
    super.visitMethodInsn(opcode, owner, name, desc, itf);
}
```

在使用 ASM 进行插桩的时候，我们尤其需要注意以下 两点：
- 1）、**当我们使用 ASM 处理字节码时，需要 逐步小量的修改、验证，切记不要编写大量的字节码并希望它们能够立即通过验证并且可以马上执行。比较稳妥的做法是，每编写一行字节码时就考虑一下操作数栈与局部变量表之间的变化情况，确定无误之后再写下一行。此外，除了 JVM 的验证器之外，ASM 还维护了一个单独的字节码验证器，它也会检查你的字节码实现是否符合 JVM 规范**。
- 2）、**注意本地变量表和操作数栈的数据交换以及 try catch blcok 的处理，关于异常处理可以使用 ASM 提供的 CheckClassAdapter，可以在修改完成后验证一下字节码是否正常**。
## 7.0 以后Transform被废弃
`AGP7.0`中`Transform`已经被标记为废弃了，并且将在`AGP8.0`中移除，那我们如何去适配呢？
使用Transform Action或者## `AsmClassVisitorFactory`
Transform Action单独使用起来比较麻烦，和我们事件模型类似，我们这里可以使用AsmClassVisitorFactory来处理，AsmClassVisitorFactory是在Transform Action做的一层封装，可以不需要再处理增量编译相关。
根据官方的说法，`AsmClassVisitoFactory`会带来约18%的性能提升，同时可以减少约5倍代码

**其实核心处理那块都没有变，只是注册方式改变了，且不需要单独处理增量编译相关知识**


具体代码适配可以参考这篇文章：
[Transform 被废弃，ASM 如何适配?](https://juejin.cn/post/7105925343680135198)


## 总结

**在 ASM Bytecode Outline 工具的帮助下，我们能够完成很多场景下的 ASM 插桩的需求，但是，当我们使用其处理字节码的时候还是需要考虑很多种可能出现的情况。如果想要具备这方面的深度思考能力，我们就 必须对每一个操作码的特征都有较深的了解，如果还不了解的同学可以去看看 《深入探索编译插桩技术（三、JVM字节码）。因此，要具备实现一个复杂 ASM 插桩的能力，我们需要对 JVM 字节码、ASM 字节码以及 ASM 源码中的核心工具类的实现 做到了然于心，并且在不断地实践与试错之后，我们才能够成为一个真正的 ASM 插桩高手。**


[编译插桩DEMO地址](https://github.com/AndroidAdvanceWithGeektime/Chapter-ASM)
## 参考

1、[ASM官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fasm.ow2.io%2F "https://asm.ow2.io/")

2.[极客时间之Android开发高手课 练习Sample跑起来 | ASM插桩强化练习](https://link.juejin.cn/?target=https%3A%2F%2Ftime.geekbang.org%2Fcolumn%2Farticle%2F83148 "https://time.geekbang.org/column/article/83148")

3. [ASM解谜](https://juejin.cn/post/6844904118700474375)




​		