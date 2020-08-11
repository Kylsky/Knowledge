# Maven权威指南

## 1.介绍

Maven是一个项目管理工具，它包含了一个**项目对象模型** (Project Object Model)，一组**标准集合**，一个**项目生命周期**(Project Lifecycle)，一个**依赖管理系统**(Dependency Management System)，和用来运行定义在生命周期阶段(phase)中**插件**(plugin)目标(goal)的逻辑。 当你使用Maven的时候，你用一个明确定义的项目对象模型来描述你的项目，然后 Maven 可以应用横切的逻辑，这些逻辑来自一组共享的（或者自定义的）插件。

### 1.1约定优于配置

Maven通过给项目提供明智的默认行为来融合这个概念。 在没有自定义的情况下，源代码假定是在 maven-guide-zh-to-production/workspace/content-zh/src/main/**java**，资源文件假定是在 maven-guide-zh-to-production/workspace/content-zh/src/main/**resources**。测试代码假定是在 maven-guide-zh-toproduction/
workspace/content-zh/src/**test** 。项目假定会产生一个 JAR 文件。Maven假定你想要把编译好的字节码放到 maven-guidezh-to-production/workspace/content-zh/target/classes 并且在 maven-guide-zh-to-production/workspace/content-zh/**target** 创建一个可分发的 JAR 文件。 虽然这看起来无关紧要，但是想想大部分基于 Ant 的构建必须为每个子项目定义这些目录。 Maven 对约定优于配置的应用不仅仅是简单的目录位置，Maven 的核心插件使用了一组通用的约定，以用来编译源代码，打包可分发的构件，生成 web 站点，还有许多其他的过程。 Maven 的力量来自它的"武断"，它有一个定义好的生命周期和一组知道如何构建和装配软件的通用插件。如果你遵循这些约定，Maven 只需要几乎为零的工作——**仅仅是将你的源代码放到正确的目录**，Maven 将会帮你处理剩下的事情。



## 2.安装

### 2.1验证java

```
java -version
```

### 2.2下载maven并安装

```
http://maven.apache.org/download.html
```

### 2.3设置环境变量

```
M2_HOME=  #maven文件夹所在路径

PATH=%PATH%;%M2_HOME%\bin
M2_HOME=maven所在文件夹的路径
```



## 3.核心概念

### *3.1Maven插件和目标

```
mvn archetype:create -DgroupId=org.sonatype.mavenbook.ch03 \
-DartifactId=simple \
-DpackageName=org.sonatype.mavenbook
```

如上例，**archetype**即为插件，**create**即为目标，archetype:create则表示通过archetype插件创建一个maven项目

一个**Maven插件**是一个单个或者多个目标的集合，Maven插件的例子有一些简单但核心的插件，像Jar插件，它包含了一组创建JAR文件的目标，Compiler插件，它包含了一组编译源代码和测试代码的目标。

一个**目标**是一个明确的任务，它可以作为单独的目标运行，或者作为一个大的构建的一部分和其它目标一起运行。一个目标是Maven中的一个“工作单元(unit ofwork)”。目标的例子包括Compiler插件中的compile目标，它用来编译项目中的所有源文件，或者Surefire插件中的test目标，用来运行单元测试。目标通过配置属性进行配置，以用来定制行为。



### *3.2Maven生命周期

生命周期是包含在一个项目构建中的一系列有序的阶段。Maven可以支持许多不同的生命周期，但是最常用的生命周期是默认的Maven生命周期，这个生命周期中一开始的一个阶段是验证项目的基本完整性，最后的一个阶段是把一个项目发布成产品。

插件目标可以附着在生命周期阶段上。随着Maven沿着生命周期的阶段移动，它会执
行附着在特定阶段上的目标。每个阶段可能绑定了零个或者多个目标。图示如下：

![image-20200706163713661](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200706163713661.png)

当Maven经过以package为结尾的默认生命周期的时候，下面的目标按顺序被执行：

```
resources:resources
	Resources插件的resources目标绑定到了resources 阶段。这个目标复制src/main/resources下的所有资源和其它任何配置的资源目录，到输出目录。

compiler:compile
	Compiler插件的compile目标绑定到了compile 阶段。这个目标编译src/main/java下的所有源代码和其他任何配置的资源目录，到输出目录。

resources:testResources
	Resources插件的testResources目标绑定到了test-resources 阶段。这个目标复制src/test/resources下的所有资源和其它任何的配置的测试资源目录，到测试输出目录。

compiler:testCompile
	Compiler插件的testCompile目标绑定到了test-compile 阶段。这个目标编译src/test/java下的测试用例和其它任何的配置的测试资源目录，到测试输出目录。

surefire:test
	Surefire插件的test目标绑定到了test 阶段。这个目标运行所有的测试并且创建那些捕捉详细测试结果的输出文件。默认情况下，如果有测试失败，这个目标会终止。
	
jar:jar
	Jar插件的jar目标绑定到了package 阶段。这个目标把输出目录打包成JAR文
件。
```



### *3.3Maven坐标

Maven坐标定义了一组标识，它们可以用来唯一标识一个项目，一个依赖，或者Maven
POM里的一个插件。看一下下面的POM:

![image-20200706163914057](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200706163914057.png)

groupId, artifactId, version和packaging。这些组合的标识符拼成了一个项目的坐标,就像任何其它的坐标系统，一个Maven坐标是一个地
址，即“空间”里的某个点：从一般到特殊。当一个项目通过依赖，插件或者父项目引用和另外一个项目关联的时候，Maven通过坐标来精确定位一个项目

```
groupId
	团体，公司，小组，组织，项目，或者其它团体。团体标识的约定是，它以创建这个项目的组织名称的逆向域名(reverse domain name)开头。来自Sonatype的项目有一个以com.sonatype开头的groupId，而Apache Software的项目有以org.apache开头的groupId。

artifactId
	在groupId下的表示一个单独项目的唯一标识符。
	
version
	一个项目的特定版本。发布的项目有一个固定的版本标识来指向该项目的某一个特定的版本。而正在开发中的项目可以用一个特殊的标识，这种标识给版本加上一个“SNAPSHOT”的标记。

packaging
	项目的类型，默认是jar，描述了项目打包后的输出。类型为jar的项目产生一个JAR文件，类型为war的项目产生一个web应用。项目的打包格式也是Maven坐标的重要组成部分，但是它不是项目唯一标识符的一个部分。需要注意的是：一个项目的groupId:artifactId:version使之成为一个独一无二的项目；你不能同
时有一个拥有同样的groupId, artifactId和version标识的项目。

scope
	https://www.cnblogs.com/kingsonfu/p/10342892.html
```



### 3.4Maven仓库

当你第一次运行Maven的时候，你会注意到Maven从一个远程的Maven仓库下载了许多文件。如果这个简单的项目是你第一次运行Maven，那么当触发resources:resource目标的时候，它首先会做的事情是去下载最新版本的Resources插件。在Maven中，构件和插件是在它们被需要的时候从远程的仓库取来的。初始的Maven下载包的大小相当的小（1.8兆），其中一个原因是事实上这个初始Maven不包括很多插件。

Maven从远程仓库下载构件和插件到你本机上，存储在你的本地Maven仓库里。一旦Maven已经从远程仓库下载了一个构件，它将永远不需要再下载一次，因为maven会首先在本地仓库查找插件，然后才是其它地方。在Windows 上，你的本地仓库很可能在C:\Documents and Settings\USERNAME\.m2\repository



### 3.5Maven依赖管理

一个复杂的项目将会包含很多依赖，也有可能包含依赖于其它构件的依赖。这是Maven最强大的特征之一，它支持了传递性依赖（transitive dependencies）。假如你的项目依赖于一个库，而这个库又依赖于五个或者十个其它的库（就像Spring或者Hibernate那样）。你不必找出所有这些依赖然后把它们写在你的pom.xml里，你只需
要加上你直接依赖的那些库，Maven会隐式的把这些库间接依赖的库也加入到你的项目中。Maven也会处理这些依赖中的冲突，同时能让你自定义默认行为，或者排除一些特定的传递性依赖。

当你把项目的构件安装到本地仓库时，你会发现在和JAR文件同一目录下，Maven发布了一个稍微修改过的pom.xml的版本。存储POM文件在仓库里提供给其它项目了该项目的信息，其中最重要的就是它有哪些依赖。如果项目B依赖于项目A，那么它也依赖于项目A的依赖。当Maven通过一组Maven坐标来处理依赖构件的时候，它也会获取POM，通依赖的POM来**寻找传递性依赖**。那些传递性依赖就会被添加到当前项目的依赖列表中。

在Maven中一个依赖不仅仅是一个JAR。它是一个POM文件，这个POM可能也声明了对其它构件的依赖。这些依赖的依赖叫做传递性依赖，Maven仓库不仅仅存贮二进制文件，也存储了这些构建的元数据（metadata，即pom.xml），才使传递性依赖成为可能。



### 3.6站点生成和报告

另外一个Maven的重要特征是，它能生成文档和报告。命令为**mvn site**

这将会运行site生命周期阶段。它不像默认生命周期那样，管理代码生成，操作资源，编译，打包等等。Site生命周期只关心处理在src/site目录下的site内容，还有生成报告。在这个命令运行过之后，你将会在target/site目录下看到一个项目web站点。载入target/site/index.html你会看到项目站点的基本外貌。它包含了一些报告，它们在
左手边的导航目录的“项目报告”下面。它也包含了项目相关的信息，依赖和相关开发人员信息，在“项目信息”下面。



## 4.创建Maven项目

### 4.1创建项目

略



### 4.2添加依赖

略



### 4.3编写单元测试

#### 4.3.1添加测试范围依赖

```
<dependency>
<groupId>org.apache.commons</groupId>
<artifactId>commons-io</artifactId>
<version>1.3.2</version>
<scope>test</scope>
</dependency>
```

scope:test表示只在测试编译和测试运行时会在classpath中有效，如果项目是以war形式打包，测试范围依赖就不会呗包含在项目的打包输出中。

#### 4.3.2执行单元测试

```
mvn test
```

#### 4.3.3忽略失败的单元测试

```xml
<project>
[...]
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <testFailureIgnore>true</testFailureIgnore>
                </configuration>
            </plugin>
        </plugins>
    </build>
[...]
</project>
```

#### *4.3.4跳过单元测试

```xml
<project>
[...]
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
        </plugins>
    </build>
[...]
</project>
```

#### 4.3.5构建maven项目

```xml
<project>
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                	<descriptorRefs>
                		<descriptorRef>jar-with-dependencies</descriptorRef>
                	</descriptorRefs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

执行命令

```
mvn install assembly:assembly
```

jar-with-dependencies 格式创建一个包含所有 simple-weather 项目的二进制代码以及所有依赖解压出来的二进制代码的 JAR 文件。 这个略微非常规的格式产生了一个 9MiB 大小的 JAR 文件，包含了大概 5290 个类。 但是它确实给那些使用 Maven 开发的应用程序提供了一个易于分发的格式。本书的后面会**说明如何创建一个自定义的装配描述符来生成一个更标准的分发包。**



## 5.多模块项目

### 5.1简介

一个多模块项目通过一个父POM引用一个或多个子模块来定义。父项目不像之前的项目那样创建一个JAR或者一个WAR，它仅仅是一个引用其它Maven项目的POM。父POM的正确打包类型应该是pom，举例如下：

```xml
<groupId>org.sonatype.mavenbook.ch06</groupId>
<artifactId>simple-parent</artifactId>
<packaging>pom</packaging>
<version>1.0</version>
```

当一个项目需要设定父项目时，则需要在pom中指定parent，举例如下：

```xml
<parent>
    <groupId>org.sonatype.mavenbook.ch06</groupId>
    <artifactId>simple-parent</artifactId>
    <version>1.0</version>
</parent>
```



### 5.2构建多模块项目

```
mvn clean install
```

当Maven执行构建一个带有子模块的项目的时候，Maven首先载入父POM，然后定位所有的子模块的POM。Maven然后将所有这些项目的POM放入到一个称为Maven 反应堆（Reactor）的东西中，由它负责分析模块之间的依赖关系。这个反应堆处理组件的排序，以确保相互独立的模块能以适当的顺序被编译和安装。除非需要做更改，反应堆一直维持定义在POM中的模块的顺序。为此一个有帮助的思维模型是，那些依赖于兄弟项目的项目在列表中被“向下按”，直到依赖顺序被满足。



## 6.优化和重构POM

### 6.1pom清理

优化一个多模块项目的POM最好通过几步来做，因为我们需要关注很多区域。总的来说，我们是寻找一个POM中的重复，或者多个兄弟POM中的重复。当你开始一个项目，或者项目进化得很快，有一些依赖和插件的重复是可以接受的，但随着项目的成熟以及模块的增多，你会需要花一些时间来重构共同的依赖和配置点。随着项目的变大，使你的POM更高效能很大程度的帮助你管理复杂度。不管什么时候遇到一些重复的信息片段，通常都有更好的配置方式。



### 6.2优化依赖

重复依赖声明使得很难保证一个大项目中的版本一致性。当你只有两个或者三个模块的时候，可能这不是一个大问题，但当你的组织正使用一个大型的，多模块Maven构建来管理分布于很多部门的数百个组件，一个依赖间的版本不匹配能够造成混乱。

项目中一个对于名为ASM的字节码操作包的依赖版本不一致，即使处于项目层次的三层以下，如果该模块被依赖，还是可以影响到由另一个完全不同的开发组维护的web应用。单元测试会通过因为它们是基于一个版本的依赖运行的，但产品可能会灾难性的失败，原因是包（比如这里是war）里存在有不同版本的类库。如果你拥有数十个项目使用比如Hibernate这样的东西，每个项目重复那些依赖和排除配置，那么有人搞坏构建的平均发生时间就会很短。由于你的Maven项目变得很复杂，依赖列表也会增大，就**需要在父POM中巩固版本和依赖声明**。

在子POM的依赖配置被上移到父POM之后，我们需要为每个子POM移除这些依赖的版本，否则它们会覆盖定义在父项目中的dependencyManagement。



### 6.3优化插件

如果两个子POM同时引用了同一个插件，则可以将该插件放置到父POM中管理这个插件配置。举例如下：

```xml
<project>
...
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <source>1.5</source>
                        <target>1.5</target>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>hibernate3-maven-plugin</artifactId>
                    <version>2.1</version>
                    <configuration>
                    <components>
                        <component>
                        <name>hbm2ddl</name>
                        <implementation>annotationconfiguration</implementation>
                    </component>
                    </components>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
...
</project>
```



## 7.项目对象模型

![image-20200707152326637](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20200707152326637.png)

POM包含了四类描述和配置：
1.项目总体信息
它包含了一个项目的名称，项目的URL，发起组织，以及项目的开发者贡献者列表和许可证。

2.构建设置
在这一部分，我们自定义Maven构建的默认行为。我们可以更改源码和测试代码的位置，可以添加新的插件，可以将插件目标绑定到生命周期，我们还可以自定义站点生成参数。

3.构建环境
构建环境包含了一些能在不同使用环境中激活的profile。例如，在开发过程中你可能会想要将应用部署到一个dev服务器上，而在prod环境中你会需要将应用部署到线上服务器上。构建环境为特定的环境定制了构建设置，通常它还会由~/.m2中的自定义settings.xml补充。这个settings文件将会在第 11 章构建Profile中，以及第 A.1 节 “简介”中的附录 A, 附录: Settings小节中讨论。

4.POM关系
一个项目很少孤立存在；它会依赖于其它项目，可能从父项目继承POM设置，它要定义自身的坐标，可能还会包含子模块。



### *7.1项目版本

一个Maven项目发布版本号用version编码，用来分组和排序发布。Maven中的版本包含
了以下部分：**主版本，次版本，增量版本，和限定版本号**。一个版本中，这些部分对应
如下的格式：

```
<major version>.<minor version>.<incremental version>-<qualifier>
```


例如：版本“**1.3.5**”由一个**主版本1**，一个**次版本3**，和一个**增量版本5**。而一个版本“**5**”只有主版本5，没有次版本和增量版本。限定版本用来标识里程碑构建：alpha和beta发布，限定版本通过连字符与主版本，次版本或增量版本隔离。例如，版本“**1.3-beta-01**”有一个主版本1，次版本3，和一个限定版本“beta-01”。
当你想要在你的POM中使用版本界限的时候，保持你的版本号与标准一致十分重要。如果你的版本号与格式<主版本>.<次版本>.<增量版本>-<限定版本>相匹配，它就能被正确的比较；“1.2.3”将被评价成是一个比“1.0.2”更新的构件，这种比较基于主版本，次版本，和增量版本的数值。如果你的版本发布号没有符合本节介绍的标准，那么你的版本号只会根据字符串被比较；如“1.0.1b”和“1.2.0b”会使用字符串比较。

#### 7.1.1 版本限定号

我们还需要对版本号的限定版本进行排序。以版本号“1.2.3-alpha-2”和“1.2.3-alpha-10”为例，这里“alpha-2”对应

了第二次alpha构建，而“alpha-10”对应了第十次alpha构建。虽然“alpha-10”应该被认为是比“alpha-2”更新的构

建，但Maven排序的结果是“alpha-10”比“alpha-2”更旧，问题的原因就是我们刚才讨论的Maven处理版本号的方

式。

Maven会将限定版本后面的数字认作一个构建版本。换句话说，这里限定版本是“alpha”，而构建版本是2。虽然

Maven被设计成将构建版本和限定版本分离，但目前这种解析还是失效的。因此，“alpha-2”和“alpha-10”是使用

字符串进行比较的，而根据字母和数字“alpha-10”在“alpha-2”前面。要避开这种限制，你需要对你的限定版本使用

一些技巧。如果你使用“alpha-02”和“alpha-10”，这个问题就消除了，一旦Maven能正确的解析版本构建号之后，

这种工作方式也还是能用。



#### 7.1.2 SNAPSHOT版本

如果一个版本包含字符“SNAPSHOT”，Maven就会在安装或发布这个组件的时候将该符号展开为一个日期和时间

值，转换为UTC（协调世界时）。

例如，如果你的项目有个版本为“1.0-SNAPSHOT”并且你将这个项目的构件部署到了一个Maven仓库，如果你在

UTC2008年2月7号下午11:08部署了这个版本，Maven就会将这个版本展开成“1.0-20080207-230803-1”。换句话

说，当你发布一个snapshot，你没有发布一个软件模块，你只是发布了一个特定时间的快照版本。

那么为什么要使用这种方式呢？SNAPSHOT版本在项目活动的开发过程中使用。如果你的项目依赖的一个组件正

处于开发过程中，你可以依赖于一个SNAPSHOT版本，在你运行构建的时候Maven会定期的从仓库下载最新的

snapshot。类似的，如果你系统的下一个发布版本是“1.4”你的项目需要拥有一个“1.4-SNAPSHOT”的版本，之后

它被正式发布。

作为一个默认设置，Maven不会从远程仓库检查SNAPSHOT版本，要依赖于SNAPSHOT版本，用户必须在POM中

使用repository和pluginRepository元素显式的开启下载snapshot的功能。当发布一个项目的时候，你需要解析所

有对SNAPSHOT版本的依赖至正式发布的版本。如果一个项目依赖于SNAPSHOT，那么这个依赖很不稳定，它随

时可能变化。发布到非snapshot的Maven仓库（如http://repo1.maven.org/maven2）的构件不能依赖于任何

SNAPSHOT版本，因为Maven的超级POM对于中央仓库关闭了snapshot。SNAPSHOT版本只用于开发过程。



#### 7.1.3 LATEST和RELEASE版本

当你依赖于一个插件或一个依赖，你可以使用特殊的版本值LATEST或者RELEASE。LATEST是指某个特定构件最新

的发布版或者快照版(snapshot)，最近被部署到某个特定仓库的构件。RELEASE是指仓库中最后的一个非快照版

本。总得来说，设计软件去依赖于一个构件的不明确的版本，并不是一个好的实践。如果你处于软件开发过程中，

你可能想要使用RELEASE或者LATEST，这么做十分方便，你也不用为每次一个第三方类库新版本的发布而去更新

你配置的版本号。但当你发布软件的时候，你总是应该确定你的项目依赖于某个特定的版本，以减少构建的不确定

性。



### *7.2 属性引用

一个POM可以通过一对大括弧和前面一个美元符号来包含 对属性的引用。

Maven提供了三个隐式的变量，可以用来访问环境变量，POM信息，和Maven Settings：

#### 7.2.1 env

env变量暴露了你操作系统或者shell的环境变量。例如${env.OS}获取操作系统



#### 7.2.2 project

project变量暴露了POM。你可以使用点标记（.）的路径来引用POM元素的值。例如：${project.artifactId}。



#### 7.2.3 settings

settings变量暴露了Maven settings信息。可以使用点标记（.）的路径来引用settings.xml文件中元素的值。例如，${settings.offline}会引用~/.m2/settings.xml文件中offline元素的值。



### 7.3 项目依赖

Maven可以管理内部和外部依赖。一个Java项目的外部依赖可能是如Plexus，SpringFramework，或者Log4J的类

库。一个内部的依赖就比如自己在本地封装的jar包

#### 7.3.1 范围依赖

**1.compile（编译范围）**

compile是默认的范围；如果没有提供一个范围，那该依赖的范围就是编译范围。编译范围依赖在所有的

classpath中可用，同时它们也会被打包。

**2.provided（已提供范围）**

provided依赖只有在当JDK或者一个容器已提供该依赖之后才使用。说白了就是你的插件jar里添加了一个依赖（比

如fastjson） 并且scope设置成provided  , 其中你的插件jar包里 方法用到了这个依赖（fastjson），那么你在新

项目中引用插件jar的时候也要添加 fastjson这个依赖。

**3.runtime（运行时范围）**

runtime依赖在运行和测试系统的时候需要，但在编译的时候不需要。比如，你可能在编译的时候只需要JDBC API 

JAR，而只有在运行的时候才需要JDBC驱动实现。

**4.test（测试范围）**

test范围依赖 在一般的 编译和运行时都不需要，它们只有在测试编译和测试运行阶段可用

**5.system（系统范围）**

system范围依赖与provided类似，但是你必须显式的提供一个对于本地系统中JAR文件的路径。这么做是为了允许

基于本地对象编译，而这些对象是系统类库的一部分。这样的构件应该是一直可用的，Maven也不会在仓库中去

寻找它。如果你将一个依赖范围设置成系统范围，你必须同时提供一个systemPath元素。注意该范围是不推荐使

用的（你应该一直尽量去从公共或定制的Maven仓库中引用依赖）。



#### 7.3.2 依赖版本界限

你并不是必须为依赖声明某个特定的版本，你可以指定一个满足给定依赖的版本界限。例如，你可以指定你的项目

依赖于JUnit的3.8或以上版本，或者说依赖于JUnit 1.2.10和1.2.14之间的某个版本。你可以使用如下的字符来围绕

一个或多个版本号，来实现版本界限。

```
(, )
不包含量词
[, ]
包含量词

如：
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>[3.8,4.0)</version>
    <scope>test</scope>
</dependency>
```



#### 7.3.3 传递性依赖

一个传递性依赖就是对于一个依赖的依赖。如果project-a依赖于project-b，而后者接着依赖于project-c，那么

project-c就被认为是project-a的传递性依赖。如果project-c依赖于project-d，那么project-d就也被认为是

project-a的传递性依赖。Maven的部分吸引力是由于它能够管理传递性依赖，并且能够帮助开发者屏蔽掉跟

踪所有编译期和运行期依赖的细节。你可以只依赖于一些包如Spring Framework，而不用担心Spring 

Framework的所有依赖，Maven帮你自动管理了，你不用自己去详细了解配置。
Maven是怎样完成这件事情的呢？它建立一个依赖图，并且处理一些可能发生的冲突和重叠。例如，如果Maven

看到有两个项目依赖于同样的groupId和artifactId，它会自动整理出使用哪个依赖，选择那个最新版本的依赖。虽

然这听起来很方便，但在一些边界情况中，传递性依赖会造成一些配置问题。在这种情况下，你可以使用依赖排

除。



#### 7.3.4 依赖排除

有很多时候你需要排除一个传递性依赖，比如当你依赖于一个项目，后者又继而依赖于另外一个项目，但你的希望是，要么整个的排除这个传递性依赖，要么用另外一个提供同样功能的依赖来替代这个传递性依赖。

```xml
<dependency>
    <groupId>org.sonatype.mavenbook</groupId>
    <artifactId>project-a</artifactId>
    <version>1.0</version>
    <exclusions>
        <exclusion>
            <groupId>org.sonatype.mavenbook</groupId>
            <artifactId>project-b</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```



## 8.构建profile