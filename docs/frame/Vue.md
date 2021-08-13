# Vue

一个前端框架

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