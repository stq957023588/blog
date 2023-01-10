# 常用标签介绍

**project**：包含pom一些约束的信息 
**modelVersion**： 指定当前pom的版本 
**groupId**：（主项目标示，定义当前maven属于哪个项目，反写公司网址+项目名）、 
**artifactId**：（实际项目模块标识，项目名+模块名）、 
**version**：（当前项目版本号，第一个0标识大版本号，第二个0标示分支版本号，第三个0标识小版本号，0.0.1，snapshot快照，alpha内部测试，beta公测，release稳定，GA正式发布） 
**name**：项目描述名 
**url**：项目地址 
**description**：项目描述 
**developers**：开发人员列表 
**licenses**：许可证 
**organization**：组织 
**dependencies**：依赖列表 
**dependency**：依赖项目 里面放置坐标 
**scope**：包的依赖范围 test 
**optional** ：设置依赖是否可选 
**exclusions**：排除依赖传递列表 
**dependencyManagement** 依赖的管理 
**build**：为构建行为提供支持 
**plugins**：插件列表 
**parent**：子模块对父模块的继承 
**modules**：聚合多个maven项目

# dependency中的scope

maven中三种classpath 
编译，测试，运行 

1. compile：**默认范围**，编译测试运行都有效 
2. provided：在编译和测试时有效 
3. runtime：在测试和运行时有效 
4. test:只在测试时有效 
5. system:在编译和测试时有效，与本机系统关联，可移植性差

# 常用的打包可执行Jar

**Springboot项目推荐**

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
            <configuration>
                <classifier>spring-boot</classifier>
                <mainClass>
                 com.test.Application
                </mainClass>
            </configuration>
        </execution>
    </executions>
</plugin>
```

> * `goal`要写成`repackage`
> * `classifier`要写成`spring-boot`

### 使用`maven-assembly-plugin`

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
            <configuration>
                <archive>
                <manifest>
                    <mainClass>
                        com.test.Application
                    </mainClass>
                </manifest>
                </archive>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
        </execution>
    </executions>
</plugin>
```

