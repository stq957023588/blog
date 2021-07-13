# Hoppscotch

类似Postman的一个工具,是一个前端项目,需要运行在Tomcat中,可以连接web socket

## 部署安装

* 从[GitHub](https://github.com/hoppscotch/hoppscotch )下载源码文件
* 使用Idea打开,运行nuxt可直接运行项目,运行nuxt generate --modern可用于打包生成用于部署在Tomcat的文件夹
* 运行nuxt generate --modern后在会项目根目录下生成一个dist文件夹,将该文件夹放入Tomcat的webapps下,启动tomcat即可
> PS. 如果用于Tomcat下,需要在nuxt.config文件中export default 中添加router
> ```js
> export default {
>   //  ...
>   router: {
>     base: "/hoppscotch/"
>   },
>   // ...
> }
> ```