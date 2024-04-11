# Vue

一个前端框架

## 脚手架

* 可视化创建项目

```shell
vue ui
```

## Idea开发@符号路径无法调换问题

在项目根目录下新建jsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES6",
    "module": "commonjs",
    "allowSyntheticDefaultImports": true,
    "baseUrl": "./",
    "paths": {
      "@/*": [
        "src/*"
      ]
    }
  },
  "exclude": [
    "node_modules"
  ]
}
```

## Vue Router

### 后台管理route component

数据库存储相对路径,前端渲染时使用相对路径,且component必须在views文件夹下

```js
function(component){
    return resolve => require([`@/views${component}`], resolve)
}
```



## 一些技巧

### Element UI

表格高度自适应

```html
<el-table :height="`calc(100vh - 187px)`" />
```

