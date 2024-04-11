# 配置

## 添加环境变量设置缓存位置

此处暂无不添加环境变量的方式

GRADLE_USER_HOME=D:\Software\Gradle\.gradle

## 修改仓库镜像

在${GRADLE_USER_HOME}文件夹下创建init.gradle文件

```groovy
allprojects {
    repositories {
        maven { name "Alibaba" ; url "https://maven.aliyun.com/repository/public/" }
        maven { name "Alibaba Spring" ;url 'https://maven.aliyun.com/repository/spring/'}
        maven { name "Bstek" ; url "https://nexus.bsdn.org/content/groups/public/"}
        mavenCentral()
    }
    buildscript { 
        repositories { 
            maven { name "Alibaba" ; url 'https://maven.aliyun.com/repository/public'}
            maven { name "Bstek" ; url 'https://nexus.bsdn.org/content/groups/public/' }
            maven { name "M2" ; url 'https://plugins.gradle.org/m2/'}
        }
    }
}
```

# Task

## 自定义打包

```groovy
tasks.register("TestApplication",Jar){
    // 配置 JAR 文件的属性
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
    archiveBaseName = 'TestApplication'
    archiveVersion = '0.0.1'
    manifest {
        attributes(
                "Manifest-Version": "1.0",
                'Main-Class': 'com.fool.TestApplication' //指定main方法所在的文件
        )
    }
    // 将编译后的类文件添加到 JAR 中
    from sourceSets.main.output
    // 将项目依赖添加到 JAR 中
    from configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) }
    // 排除测试代码
    exclude 'META-INF/*.RSA', 'META-INF/*.SF', 'META-INF/*.DSA'
}
```

## 自定义Task

```groovy
// 
abstract class GreetingTask extends DefaultTask{
    @TaskAction
    def greet() {
        println 'hello from GreetingTask'
    }
}

tasks.register('hello', GreetingTask)
```

