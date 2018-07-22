# Android 移动应用安全加固
致力于成为业界移动应用安全专家，为移动数据安全保驾护航！


## 安全加固刻不容缓
> 目前移动领域已经出现了相当部分的安全问题，新的恶意软件层出不穷，同时，企业对敏感数据保密性意识日益提高，作为移动开发者，有责任对最终用户的隐私和安全承担更多责任。另一方面，APP被篡改、盗用，直接伤害了开发者的知识和劳动权益，移动开发者有必要保护自己的利益不受损失。

![img1](imgs/幻灯片2.PNG)

> 本文介绍成熟的安全加固方案，均已实现和使用，解决了众多企业和个人的安全问题。

## 安全加固项介绍
##### 安全项
三个维度提升应用安全能力：应用包、运行环境、应用数据。各个安全项如下：防二次打包保护、防重签名保护、插花指令混淆保护、资源指纹签名保护、开发者license保护；
ROOT检测告警、防应用界面劫持、截屏防护；安全密钥管理、本地数据安全存储、核心代码分离保护。

![img1](imgs/幻灯片4.PNG)
![img1](imgs/幻灯片5.PNG)
![img1](imgs/幻灯片6.PNG)

##### 安全加固应用方式
![img1](imgs/幻灯片7.PNG)

##  移动数据存储
Android平台实现数据存储的基本方式：

- 数据共享(SharedPreferences)
- 内部存储(File)
- SQLite数据库存储
- 外部存储
- 网络存储

这里讨论前三种数据存储类型，实现了加密解密SDK，并实现对APP的安全存储注入。首先我们来简单讨论下Android中数据存储的位置——考虑数据安全，有必要更改android应用的存储位置吗？

在非root设备上，数据已可通过沙盒得到很好地安全保护（当然，我们也不可以完全忽略设备存在内核漏洞或虚拟机漏洞时可能发生的后果）。“自定义存储位置”也是一种设计方式，使破解者难以确定存储的路径，并完全控制了存储实现，以进行加密和解密保护。但这样做，失去了Android自身完善的特权分离机制（安全沙盒），将数据文件完全暴露在沙盒外，弊大于利，实际上意义不大。

因此这里仍选择沙盒作为数据的存储位置，接下来主要讨论对于root的设备，如何增强数据的安全性。在root设备上，需要通过数据擦除或数据加密，实现数据的安全存储，降低数据暴露的可能性。

##### 加密方案选择
涉及到了加密，我们需要选择一种加密算法，加密算法的选择无穷无尽，本文采用AES加密算法，选择128位秘钥加密方式。以下是三种存储类型在AES算法下的具体实现方式：

- SharedPreferences存储加密解密方式：对key和value同时加密，存储类型都为String类型，数据读取时根据需要进行类型转换。
- File文件存储加密解密方式：对数据流进行加密解密。
- SQLite数据库存储加密解密方式：基于Sqlcipher进行实现。

##  防二次打包或重签名
##### 方案简述
- 签名运行时校验
- 资源文件运行时校验
- dex资源运行时校验
- 其他资源运行时校验

此外，针对暴力签名破解，提供对抗暴力破解方案，原理是动态注入+Hook

##  对抗应用静态分析
##### 方案简述
- 无效花字节填充
- 加密标识篡改
- 虚假资源引用定义填充
- 代码特殊语句段填充

##  对抗应用动态分析
##### 方案简述
- 监控内存相关虚拟文件
- API调用判断
- 进程状态判断
- 运行环境智能判断
- 检测代码执行时间
- 检测系统关键文件
- 检测特殊端口号
- 检测breakpoint指令
- 检测调试器特征
- 主动ptrace

## 安全加固工具方案设计
##### 注入方案比较
在”哪里”或者如何将我们提供的SDK注入到已有APP中，以及如何修改此APP中的实现。这里注入的层面可以是：java字节码（.class）或DEX字节码（.smali），二者分别对应反编译和反汇编过程。这里选择后者作为注入方案，原因如下：

(1) smali语法定义明确，易于直接替换对象类型；

(2)在反编译流程中，更直接。

##### DEX字节码
Dalvik虚拟机运行 Dalvik 字节码，由java字节码转换而来，又称为” Dalvik汇编语言”，是Dalvik指令集组成的代码，Dalvik指令集是Dalvik虚拟机为自己专门设计的一套指令集，严格说它不属于正式语言。我们来看一下它对方法的定义：

`invoke-virtual {p0, v2, v3},`
`Landroid/content/Context;->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;`
`move-result-object v1`

这里调用了类android.content.Context的getSharedPreferences方法，其中p0寄存器存储context实例，方法的参数是两个：一个是java.lang.String类型，一个是int类型，两个参数分别保存在寄存器v2和v3中，返回值是android.content.SharedPreferences类型，并把返回值保存到v1寄存器中。

Dalvik字节码中使用了寄存器保存变量和参数。我们知道，Java虚拟机基于栈架构，程序需频繁从栈上读取或写入数据，会耗费CPU(指令分派、内存访问次数)；Dalvik虚拟机基于寄存器架构，数据访问通过寄存器直接传递。两者都为每个线程维护一个PC计数器与调用栈，PC计数器以字节为单位记录当前运行位置距离方法开头的偏移量。不同的是，Java栈记录Java方法调用的活动记录，以帧为单位保存线程的运行状态，每调用一个方法就会分配新的栈帧压入Java栈上，方法返回则弹出并撤销相应的栈帧；而Dalvik栈维护一份寄存器表。

每个Dalvik寄存器都是32位，对于大于这个长度的类型，用两个相邻寄存器存储。寄存器被设计为两种表示方法：v命名法和p命名法。具体请参考相关资料。

##### 加固流程设计
App加固总流程如下：

apk反汇编 -> sdk注入 -> Smali文件扫描 -> 代码修正 -> 正向汇编 -> apk签名

##### 加固流程实现
这里以SharedPreferences存储为例，例举SDK和代码修正的实现方式。(其他步骤实现略)

##### 安全存储SDK
SDK中提供了针对SharedPreferences存储类型的加密解密方式，我们对外提供了一个SecureSharedPreferences类，它实现了android.content.SharedPreferences接口，并实现了接口所提供的各个方法，将android.content.SharedPreferences类型实例作为参数，传递给构造方法。当读取数据时，使用这个参数进行查询并返回值，当存储数据时，在edit方法处，返回一个实现android.content.SharedPreferences.Editor接口的类，进行数据的添加。

这样的SDK设计使得代码注入可行，并只需要较少的修正量。

##### 代码修正
在调用SharedPreferences存储时，某种实现方式可以是(java代码)：

`SharedPreferences sp =context.getSharedPreferences("config", 0);`

我们需要将其修正为调用我们提供的SDK方式来实例化sp对象。DEX字节码如下：

`invoke-virtual {p0, v2, v3},`

`Landroid/content/Context;->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;`

我们首先将这个结果保存到一个寄存器下，这里可以使用v2或者v3寄存器，它们分别对应两个类型的参数：java.lang.String和int，而且之后不会再对它们进行使用，因而它们是可用的两个寄存器。这里我们选择v3寄存器：

`move-result-object v3`

接下来新建一个SecureSharedPreferences对象，并将它放到某寄存器下，这个寄存器需要是java中sp对象存储的寄存器，并且用这个寄存器的值进行SharedPreferences方法的调用，如v1寄存器：

`new-instance v1,Lcom/…/SecureSharedPreferences;`

继而需要调用这个对象的构造函数，并把之前保存在v3寄存器中的值作为参数传递(SharedPreferences类型)：

`invoke-direct {v1, v3},`

`Lcom/…/SecureSharedPreferences;->(Landroid/content/SharedPreferences;)V`

这样就完成了修正过程。

其他存储类型以及调用方式的修正方式与以上类似，这里不再赘述。

##### 验证实践
这里我们将整个流程所需的各个依赖包、库文件、各个工具、代码修正实现及我们提供的SDK集成到一起，形成一个一键式的加固工具(bat形式)。以一个DEMO为例，DEMO中实现了以上三种存储方式的数据存储和读取，其中存储都以默认方式(明文)实现，没有进行安全加密。

将DEMO APK放到工作目录下，启动并得到加固后的安全APK（secure_demo.apk），其中demo文件夹为中间过程产生的数据文件。

将它运行到一个root的Android测试机上，并使用R.E.Explorer工具查看存储，可见数据已经过了加密存储，并能够正确的解密并获得原始数据。SQLite数据库对存储本身进行了加密，因而加密后，通过工具(android内置查看器或SQLite Expert Professional查看工具)已无法打开db文件。

![img1](imgs/2425324-5c8d0d2bdb53fc07.png)
![img1](imgs/2425324-72e97c906fff4d26.png)


# 总结
以上介绍了各加固方案的实现，并将持续更新和升级，以确保安全能力的持续提升。有需要安全加固的本地工具或源码的朋友，请联系。欢迎交流

![img1](imgs/WechatIMG13.png)
