# 更改rootContext 
## 什么是rootContext
http://ip:port/{rootContext}/path/to/resource

{rootContext} 是我们讨论的重点。

## 为什么要更改rootContext
rootContext跟工程名称仅仅的耦合在一起，创建什么样的工程名称，就有什么样的rootContext，有时候这个工程名起的不好，当然我们可以改一下工程名，但是这样的做法是重量级的，而且极有可能引入一些工程问题（我曾经改过工程名，eclipse工程总是莫名的产生一些报错信息）。有没有一种简单便捷的方法，改变rootContext呢？

## 更改rootContext的轻量级方法
方法当然存在，我用的是eclipse + gradle，因此主要围绕着这两方面，其他的方式请参见下面的参考连接

eclipse与gradle改变rootContext非常简单，代码如下：

```java
apply plugin: 'java'
apply plugin : 'war'
apply plugin : 'eclipse-wtp'

eclipse{
  wtp{
    component{
      contextPath = "${CONTEXT_PATH}"
    }
  }
}

```

${CONTEXT_PATH}这个变量在gradle.properties文件中，gradle.properties文件和build.properties文件在同级路径

CONTEXT_PATH=cloud

在cmd在运行如下命令：

```java
gradle eclipse
```

重新刷新一下工程即可。

## 更优雅的解决方法
到这里，已经改变了rootContext，但是用gradle打包后，包的名称为：ProjectName.war，把这个war放到tomcat的webapps下，发现rootContext还是工程名，否则报404，当然这里改变下文件名（把ProjectName.war 改成 RootContext.war）就没什么问题了。
但是这样子总感觉多道手续，能自动化的就坚决不用人工，因为人工因素会引入潜在的风险不说，让人机械的该这玩意，跟无限拧螺丝差不多。
我们在打包的时候，让gradle更改下war的名字就可以了。代码如下：

```java
apply plugin: 'java'
apply plugin : 'war'
apply plugin : 'eclipse-wtp'
apply plugin: 'idea'

[compileJava, compileTestJava]*.options*.encoding = 'UTF-8'
	
sourceCompatibility = 1.8
targetCompatibility = 1.8

//for eclipse IDE only
eclipse{
  wtp{
    component{
      contextPath = "${CONTEXT_PATH}"
    }
  }
}

//改变包的名称
war {
  archiveName = "${CONTEXT_PATH}.war";
}
```

# 参考链接
* [Eclipse – How to change web project context root](https://www.mkyong.com/eclipse/eclipse-how-to-change-web-project-context-root/)
* [How do I change the name of the war file being built?](https://discuss.gradle.org/t/how-do-i-change-the-name-of-the-war-file-being-built/9223/4)
