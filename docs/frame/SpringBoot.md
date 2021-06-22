# 配置启动参数

* 在application.yml 中配置
* Jar包运行,在命令后面添加
  > java -jar springboot.jar --spring.profiles.active=dev
* 添加JVM参数
  > java -jar springboot.jar -Dspring.profiles.active=dev
* Tomcat war包启动,修改/bin/catalina.bat(Linux下为catalina.sh)
  > @echo off
  >
  >setlocal
  >
  >set "JAVA_OPTS=%JAVA_OPTS% -Dspring.profiles.active=dev"
  >
  >if not ""%1"" == ""run"" goto mainEntry