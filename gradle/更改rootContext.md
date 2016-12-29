# ����rootContext 
## ʲô��rootContext
http://ip:port/{rootContext}/path/to/resource

{rootContext} ���������۵��ص㡣

## ΪʲôҪ����rootContext
rootContext���������ƽ����������һ�𣬴���ʲô���Ĺ������ƣ�����ʲô����rootContext����ʱ�������������Ĳ��ã���Ȼ���ǿ��Ը�һ�¹������������������������������ģ����Ҽ��п�������һЩ�������⣨�������Ĺ���������eclipse��������Ī���Ĳ���һЩ������Ϣ������û��һ�ּ򵥱�ݵķ������ı�rootContext�أ�

## ����rootContext������������
������Ȼ���ڣ����õ���eclipse + gradle�������ҪΧ�����������棬�����ķ�ʽ��μ�����Ĳο�����

eclipse��gradle�ı�rootContext�ǳ��򵥣��������£�

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

${CONTEXT_PATH}���������gradle.properties�ļ��У�gradle.properties�ļ���build.properties�ļ���ͬ��·��

CONTEXT_PATH=cloud

��cmd�������������

```java
gradle eclipse
```

����ˢ��һ�¹��̼��ɡ�

## �����ŵĽ������
������Ѿ��ı���rootContext��������gradle����󣬰�������Ϊ��ProjectName.war�������war�ŵ�tomcat��webapps�£�����rootContext���ǹ�����������404����Ȼ����ı����ļ�������ProjectName.war �ĳ� RootContext.war����ûʲô�����ˡ�
�����������ܸо�������������Զ����ľͼ�������˹�����Ϊ�˹����ػ�����Ǳ�ڵķ��ղ�˵�����˻�е�ĸ������⣬������š��˿��ࡣ
�����ڴ����ʱ����gradle������war�����־Ϳ����ˡ��������£�

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

//�ı��������
war {
  archiveName = "${CONTEXT_PATH}.war";
}
```

# �ο�����
* [Eclipse �C How to change web project context root](https://www.mkyong.com/eclipse/eclipse-how-to-change-web-project-context-root/)
* [How do I change the name of the war file being built?](https://discuss.gradle.org/t/how-do-i-change-the-name-of-the-war-file-being-built/9223/4)
