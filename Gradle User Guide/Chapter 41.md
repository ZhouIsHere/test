# Chapter 41. Writing Custom Plugins  编写自定义插件


[原文地址 GUG v4.4.1](https://docs.gradle.org/current/userguide/custom_plugins.html)

## Table of Contents 目录

1. [创建插件](#创建插件)
2. [编写一个简单的插件](#编写一个简单的插件)
3. [使插件可以配置](#使插件可以配置)
4. [在自定义插件或者任务的文件中工作](#在自定义插件或者任务的文件中工作)
5. [适配扩展属性到任务属性](#适配扩展属性到任务属性)
6. [在独立项目中使用](#在独立项目中使用)
7. [为插件提供DSL配置](#为插件提供DSL配置)


## 创建插件

你可以把插件的源码放在以下几个地方
* Build script  编译脚本以内  

    你可以把源码直接放在build脚本里面，这种方式的好处是插件会被自动的编译和引入到脚本的classpath，你什么别的东西都不用做。可是，这个插件在这个build脚本以外就不可见了，因此你不能在定义这个插件的build脚本以外的地方复用它
* buildSrc project  buildSrc文件夹内

    你可以把插件源码放在rootProjectDir/buildSrc/src/main/groovy 目录下.Gradle也将会自动编译和测试这个插件并且把它添加到编译脚本的classpath里。这个插件对编译过程中用到的build脚本都可见。可是，它依然对build脚本以外的地方不可见，这种情况下你也不能复用它。
    在第43章对buildSrc文件夹有更详细的描述。
* Standalone project 单独的项目

    你可以为你的插件单独创建一个项目。这个项目可以导出和发布一个jar，你可以在各种各样的情形下和别人共享这个插件。通常，这个jar可能包含多个插件，or bundle several related task classes into a single library.或者两者都有。

在我们的例子中，会从第一种情况开始。为了简化，我们会先创建一个单独的项目。

## 编写一个简单的插件

要创建一个插件，你需要编写一个类实现插件的Plugin接口。当这个插件在项目中应用的时候，gradle会创建一个插件类的实例然后调用它的Plugin.apply(T)方法。这个项目对象会作为参数传递给插件，用来按照自己的需要来配置。下面是一个插件示例，添加了一个hello任务到项目中。

*Example 41.1. A custom plugin*
build.gradle
~~~Groovy
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
        }
    }
}
// Apply the plugin
apply plugin: GreetingPlugin
~~~
 gradle -q hello 的输出
> Hello from the GreetingPlugin

值得一提的是对于每一个应用这个插件的项目，都创建了一个插件类实例，而且，插件类是一个泛型类。这个例子中它接受Project类型作为类型参数。插件还可以接受Setting类型，这种情况下插件会被应用到settings脚本，或者是Gradle类型，这种情况下插件会被应用到初始化脚本中。


## 使插件可以配置
大部分的插件都需要从build脚本中获取某些配置。我们可以通过使用extension对象来完成这个功能。Gradle项目会自动和一个ExtensionContainer对象关联，这个对象包含了项目所使用的插件的所有设置和属性。你可以通过添加一个extension对象到这个container来为你的插件提供配置功能。一个extension对象通常是一个简单的java veab对象。Groovy对于实现这种对象很方便，因为它不需要设置getter和setter方法。Java和Kotlin也是可选项。
在这里我们添加了一个extension对象到项目中，用来配置greeting插件。    


*Example 41.2. A custom plugin extension*

build.gradle
~~~Groovy
class GreetingPluginExtension {
    String message = 'Hello from GreetingPlugin'
}

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        // Add the 'greeting' extension object
        def extension = project.extensions.create('greeting', GreetingPluginExtension)
        // Add a task that uses configuration from the extension object
        project.task('hello') {
            doLast {
                println extension.message
            }
        }
    }
}

apply plugin: GreetingPlugin

// Configure the extension
greeting.message = 'Hi from Gradle'
~~~
 gradle -q hello 的输出
> Hi from Gradle

在这个例子中，GreetingPluginExtension是一个包含message属性的Groovy对象，这个extension对象被用为greeting这个名字添加到插件中，然后它就在插件项目中可以作为extension对象使用了。
通常，在一个单独的插件中，你会需要特别设置一些相关的属性，Gradle会为每一个extension对象添加一个配置用的闭包块，这样你可以把一组设置放到一起。下面是示例     

*Example 41.3. A custom plugin with configuration closure*

build.gradle
~~~Groovy
class GreetingPluginExtension {
    String message
    String greeter
}

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        def extension = project.extensions.create('greeting', GreetingPluginExtension)
        project.task('hello') {
            doLast {
                println "${extension.message} from ${extension.greeter}"
            }
        }
    }
}

apply plugin: GreetingPlugin

// Configure the extension using a DSL block
greeting {
    message = 'Hi'
    greeter = 'Gradle'
}
~~~

 gradle -q hello 的输出

> Hi from Gradle

在这个例子中，多个设置作为一组放到了greeting闭包内。这个闭包的名字需要和extension对象的名字一致。然后，当这个闭包被执行，extension对象的相应属性就会被设置为闭包里的值。

## 在自定义插件或者任务的文件中工作

开发自定义任务或者插件的时候，指定一个文件地址作为插件配置会显得比较灵活。你可以使用Project.file方法来尽可能迟的解析它的值。

*Example 41.4. Evaluating file properties lazily*

build.gradle
~~~Groovy
class GreetingToFileTask extends DefaultTask {

    def destination

    File getDestination() {
        project.file(destination)
    }

    @TaskAction
    def greet() {
        def file = getDestination()
        file.parentFile.mkdirs()
        file.write 'Hello!'
    }
}

task greet(type: GreetingToFileTask) {
    destination = { project.greetingFile }
}

task sayGreeting(dependsOn: greet) {
    doLast {
        println file(greetingFile).text
    }
}

ext.greetingFile = "$buildDir/hello.txt"
~~~
gradle -q sayGreeting 的输出

~~~ 
> gradle -q sayGreeting
Hello!
~~~

在这个例子中，我们在配置了greet任务的destination属性为一个闭包，在这个闭包里，Project.file方法把闭包的值返回到一个文件对象里面。在上面的例子里，我们指定了greetingFile属性，并且在使用它之前就把它配置到了任务里面。当我们设置一个文件属性然后解析文件里的值的时候，这种懒赋值是一个很重要的优点。


## 适配扩展属性到任务属性
在build脚本里面通过extension来获取用户输入并且把它映射到自定义任务的相应属性，是一个比较好的方式。终端用户只需要和extension暴露出来的DSL交互就可以了，重要的逻辑都隐藏在插件的实现里面。
在build脚本里面声明的extension和自定义任务的属性之间的映射发生在Gradle配置的整个构建周期以内。为了避免赋值的顺序问题，实际映射属性的解析必须放在执行阶段内。更详细的细节再Section 22.1 “构建阶段” 里面有描述。Gradle API提供了一些类型用来表示属性应该在执行阶段被懒赋值。查看<lazyConfiguration>获得更多信息

下面展示了映射extension属性到task属性的过程

*Example 41.5. Mapping extension properties to task properties*

build.gradle
~~~Groovy
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        def extension = project.extensions.create('greeting', GreetingPluginExtension, project)
        project.tasks.create('hello', Greeting) {
            message = extension.message
            outputFiles = extension.outputFiles
        }
    }
}

class GreetingPluginExtension {
    final Property<String> message
    final ConfigurableFileCollection outputFiles

    GreetingPluginExtension(Project project) {
        message = project.objects.property(String)
        message.set('Hello from GreetingPlugin')
        outputFiles = project.files()
    }

    void setOutputFiles(FileCollection outputFiles) {
        this.outputFiles.setFrom(outputFiles)
    }
}

class Greeting extends DefaultTask {
    final Property<String> message = project.objects.property(String)
    final ConfigurableFileCollection outputFiles = project.files()

    void setOutputFiles(FileCollection outputFiles) {
        this.outputFiles.setFrom(outputFiles)
    }

    @TaskAction
    void printMessage() {
        outputFiles.each {
            logger.quiet "Writing message 'Hi from Gradle' to file"
            it.text = message.get()
        }
    }
}

apply plugin: GreetingPlugin

greeting {
    message = 'Hi from Gradle'
    outputFiles = files('a.txt', 'b.txt')
}
~~~

> Note: The code for this example can be found at samples/userguide/tasks/mapExtensionPropertiesToTaskProperties in the ‘-all’ distribution of Gradle.

Output of gradle -q hello

~~~
> gradle -q hello
Writing message 'Hi from Gradle' to file
Writing message 'Hi from Gradle' to file
~~~


## 在独立项目中使用

现在我们将移动我的插件到一个独立的项目中去，这样就可以发布它并且与别人共享。这个项目是一个简单的Groovy项目，它会产生一个包含若干插件class类的JAR文件。这里是一个简单的示例脚本，它适用于Groovy插件，并且添加了Gradle API作为编译期依赖

*Example 41.6. A build for a custom plugin*

build.gradle
~~~Groovy
apply plugin: 'groovy'

dependencies {
    compile gradleApi()
    compile localGroovy()
}
~~~
> Note: The code for this example can be found at samples/customPlugin/plugin in the ‘-all’ distribution of Gradle.

因此Gradle是怎么找到插件的实现的？答案是你需要提供一个properties文件在jar的META-INF/gradle-plugins文件夹下，它的名字和你的插件id相对应

*Example 41.7. Wiring for a custom plugin*

src/main/resources/META-INF/gradle-plugins/org.samples.greeting.properties
~~~
implementation-class=org.gradle.GreetingPlugin
~~~
需要一提的是properties文件名和插件id是一致的，并且它被放在特定的资源文件下，implementation-class属性则指明了插件的实现类

### 41.6.1. 构造一个插件id

合格的插件id的命名和java的包名类似（也就是域名反过来)，这样可以避免一些冲突并且为有类似拥有者的插件提供了分组的方法。
你的插件id应该是反映了命名空间(你的组织的合理指向)和插件名字的字段的组合。比如你有一个github账号名字叫foo，然后你的插件名字叫做bar，一个合适的插件id可以是com.github.foo.bar.类似的，如果这个插件是baz组织开发的，它的id应该是org.baz.bar

插件id应该符合一下要求：

    可以包含任意的字母数字字符, '.', and '-'.

    必须至少包含一个 ”.“ 用来隔开插件的命名空间和名字

    通常使用小写的反过来的域名

    通常只使用小写的名字

    不能使用org.gradle 和 com.gradleware 命名空间.

    不能以.字符开头

    不能包含连续的"."字符

虽然通常插件id和包名有些类似，但是包名一般比插件id提供了更多的细节。比如，增加gradle作为你的插件id的一部分看起来似乎行得通，但是因为插件id只用于Gradle插件，这样的名字会显得很多余。通常，一个指明了拥有者和名字的命名空间就是一个好的插件id所需要的

### 41.6.2. 发布插件

如果你要在你的组织内部使用插件，你可以像其他代码集合一样发布它，查看ivy 和 maven 章节获得更多关于发布集合的信息。
入宫你想要把你的插件发布到更广泛的Gradle社区，你可以把它发布到[Gradle plugin portal](https://plugins.gradle.org/)。这个网站提供了搜索和收集关于Gradle社区贡献的插件信息的能力。你可以在[这里](https://plugins.gradle.org/docs/submit)找到关于怎么使你的插件在该网站可用的信息。

### 41.6.3. 在别的项目里使用你的插件

为了在build脚本里面使用插件，你需要添加插件类到build脚本的classpath路径。你可以像在“Applying plugins with the buildscript block”这一节里面描述的那样使用一个"buildscript { }"块。下面的例子展示了在包含插件的JAR发布到了本地repository的时候，你怎么去使用它

*Example 41.8. Using a custom plugin in another project*

build.gradle
~~~Groovy
buildscript {
    repositories {
        maven {
            url uri('../repo')
        }
    }
    dependencies {
        classpath group: 'org.gradle', name: 'customPlugin',
                  version: '1.0-SNAPSHOT'
    }
}
apply plugin: 'org.samples.greeting'
~~~

或者，如果你的插件发布到了 plugin portal 网站上面，你可以使用插件DSL来应用它

*Example 41.9. Applying a community plugin with the plugins DSL*

build.gradle
~~~
plugins {
    id 'com.jfrog.bintray' version '0.4.1'
}
~~~

### 41.6.4. 为你的插件编写测试代码

当你测试你的插件实现的时候， 你可以使用ProjectBuilder类来创建插件实体。

*Example 41.10. Testing a custom plugin*

src/test/groovy/org/gradle/GreetingPluginTest.groovy
~~~Groovy
class GreetingPluginTest {
    @Test
    public void greeterPluginAddsGreetingTaskToProject() {
        Project project = ProjectBuilder.builder().build()
        project.pluginManager.apply 'org.samples.greeting'

        assertTrue(project.tasks.hello instanceof GreetingTask)
    }
}
~~~
### 41.6.5. 使用Java Gradle Plugin development plugin

你可以使用开发中的Java Gradle Plugin development plugin来消除build脚本里面一些模板化的声明，它还提供了插件元数据的一些基本验证。这个插件会自动应用Java插件，添加gradleApi()依赖到编译配置里，就可以执行插件元数据的验证作为执行jar任务的一部分了。

*Example 41.11. Using the Java Gradle Plugin Development plugin*

build.gradle
~~~Groovy
plugins {
    id 'java-gradle-plugin'
}
~~~
当发布插件到自定义的插件ivy repositories或者maven resositories时，Java Gradle Plugin development plugin也会生成基于插件id和插件实现的产出生成plugin marker artifacts。


## 为插件提供一个DSL配置

正如我们在上面所看到的，你可以使用一个extension对象来为你的插件提供配置功能。使用extension对象也扩展了DSL，使它可以为插件增加项目属性和DSL代码块。一个extension对象就是一个普通的对象，通过为它添加
属性和方法你可以提供DSL元素和内部代码块

### 41.7.1. DSL内部代码块

当Gradle创建了一个任务或者extension对象的时候，Gradle装饰了它的实现来提供DSL支持。为了创建一个DSL内部块，你可以使用ObjectFactory类型来创建类似装饰过的对象。通过插件的extension对象的属性和方法，这些装饰过的对象对DSL可见。

*Example 41.12. Nested DSL elements*

build.gradle
~~~Groovy
class Person {
    String name
}

class GreetingPluginExtension {
    String message
    final Person greeter

    @javax.inject.Inject
    GreetingPluginExtension(ObjectFactory objectFactory) {
        // Create a Person instance
        greeter = objectFactory.newInstance(Person)
    }

    void greeter(Action<? super Person> action) {
        action.execute(greeter)
    }
}

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        // Create the extension, passing in an ObjectFactory for it to use
        def extension = project.extensions.create('greeting', GreetingPluginExtension, project.objects)
        project.task('hello') {
            doLast {
                println "${extension.message} from ${extension.greeter.name}"
            }
        }
    }
}

apply plugin: GreetingPlugin

greeting {
    message = 'Hi'
    greeter {
        name = 'Gradle'
    }
}
~~~
Output of gradle -q hello

> gradle -q hello
Hi from Gradle

在这个例子里，插件通过extension的构造函数传递了Project的一个ObjectFactory对象给它，这个构造函数用它创造了一个内部对象，并且使它可以通过greeter的属性，由DSL来设置。


### 41.7.2. 配置对象的集合

Gradle提供了一些实用类用来维持对象集合，使她们和Gradle DSL协调工作

*Example 41.13. Managing a collection of objects*

build.gradle
~~~Groovy
class Book {
    final String name
    File sourceFile

    Book(String name) {
        this.name = name
    }
}

class DocumentationPlugin implements Plugin<Project> {
    void apply(Project project) {
        // Create a container of Book instances
        def books = project.container(Book)
        books.all {
            sourceFile = project.file("src/docs/$name")
        }
        // Add the container as an extension object
        project.extensions.books = books
    }
}

apply plugin: DocumentationPlugin

// Configure the container
books {
    quickStart {
        sourceFile = file('src/docs/quick-start')
    }
    userGuide {

    }
    developerGuide {

    }
}

task books {
    doLast {
        books.each { book ->
            println "$book.name -> $book.sourceFile"
        }
    }
}
~~~
Output of gradle -q books

> gradle -q books
developerGuide -> /home/user/gradle/samples/userguide/organizeBuildLogic/customPluginWithDomainObjectContainer/src/docs/developerGuide
quickStart -> /home/user/gradle/samples/userguide/organizeBuildLogic/customPluginWithDomainObjectContainer/src/docs/quick-start
userGuide -> /home/user/gradle/samples/userguide/organizeBuildLogic/customPluginWithDomainObjectContainer/src/docs/userGuide

Project.container(java.lang.Class)方法构造了一些NamedDomainObjectContainer实体，它们包含管理和配置对象的许多有用的方法，为了使用包含某个类型的project.container的方法，它必须暴露一个叫做name的属性作为唯一的，不变的名字。project.container(Class)包含了许多创造对象的方法，通过一个String参数来构造它们，也就是对象的类名。project.container方法允许自定义初始化的策略，你可以查看上面的链接获得相关信息。