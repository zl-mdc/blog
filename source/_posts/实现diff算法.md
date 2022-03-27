---
title: 实现diff算法
date: 2022-01-27 09:55:28
tags: diff snabbdom 最小量更新s
---




## 什么是diff算法？

diff算法就是最小量更新

实例：

```html
<div class="box">
  <h3>我是一个标题</h3>
  <ul>
    <li>牛奶</li>
    <li>咖啡</li>
    <li>可乐</li>
  </ul>
</div>
```

```html
<div class="box">
  <h3>我是一个标题</h3>
  <span>我是一个新的span</span>
  <ul>
    <li>牛奶</li>
    <li>咖啡</li>
    <li>可乐</li>
    <li>雪碧</li>
  </ul>
</div>
```
<!-- more -->
```
这种情况下，我们只需要在dom上合适的位置加上一个span标签和一个li标签即可，而不需要全部推翻重新渲染
```

## 什么是虚拟DOM？

- 虚拟 `DOM`：用 `JavaScript` 对象描述 `DOM` 的层次结构。`DOM` 中的一切属性都在虚拟 `DOM` 中有对应的属性。

- `diff` 是发生在虚拟 `DOM` 上的：新虚拟 `DOM` 和老虚拟 `DOM` 进行 `diff` （精细化比较），算出应该如何最小量更新，最后反映到真实的 `DOM` 上

  真实DOM：

  ```html
  <div class="box">
    <h3>我是一个标题</h3>
    <ul>
      <li>牛奶</li>
      <li>咖啡</li>
      <li>可乐</li>
    </ul>
  </div>
  
  ```

  虚拟DOM：

  ```json
  {
    "sel": "div",
    "data": {
      "class": { "box": true }
    },
    "children": [
      {
        "sel": "h3",
        "data": {},
        "text": "我是一个标题"
      },
      {
        "sel": "ul",
        "data": {},
        "children": [
          { "sel": "li", "data": {}, "text": "牛奶" },
          { "sel": "li", "data": {}, "text": "咖啡" },
          { "sel": "li", "data": {}, "text": "可乐" }
        ]
      }
    ]   
     
  ```

  

## 搭建测试环境

#### （1）snabbdom库：

1. `snabbdom` 是著名的虚拟 `DOM` 库，是 `diff` 算法的鼻祖，`Vue` 源码就是借鉴了 `snabbdom`
2. 官方 `Git`：https://github.com/snabbdom/snabbdom

#### (2)测试环境：

snabbdom 库是 DOM 库，当然不能在 nodejs 环境运行，所以我们需要搭建 webpack 和 webpack-dev-server 开发环境，好消息是不需要安装任何loader
这里需要注意，必须安装最新版 webpack@5，不能安装 webpack@4，这是因为 webpack@4 没有读取(package.json)中 exports 的能力，建议大家使用这样的版本：

```
npm i -D webpack@5 webpack-cli@3 webpack-dev-server@3
```

webpack.config.js文件：

```js
const path = require('path')
module.export ={
    //入口文件
    entry:"./src/index.js",
     // 出口
  output: {
    // 虚拟打包路径，就是说文件夹不会真正生成，而是在 8080 端口虚拟生成，不会真正的物理生成
    publicPath: 'xuni',
    // 打包出来的文件名
    filename: 'bundle.js'
  },
      module: {
        // 指定要价在的规则
        rules: [
            {
                // test指定的是规则生效的文件,意思是，用ts-loader来处理以ts为结尾的文件
                test: /\.ts$/,
                use: 'ts-loader',
                exclude: /node_modules/
            }
        ]
    }，
  devServer: {
    // 端口号
    port: 8081,
    // 静态资源文件夹
    contentBase: 'static'
  }
}
```

之后我们在src目录下创建-->index.ts文件

static目录下创建--->index.html文件

最终目录结构如下：

![screenshot-20220127-104359](C:\Users\14240\Desktop\screenshot-20220127-104359.png)

(由于我们使用ts，还需要配置tsconfig.json)，同时webpack.config.js同时要加上ts-loader（如上图）

```{
//tsconfig.json
{
    "compilerOptions": {
        "target": "es6",
        "module": "es6",
        "strict": true
    }
}
```

之后我们运行

```
npm run dev
```

我们就可以在localhost：8081看到index.html首页以及localhost：8081/xuni/bundle.js打包后的js文件

![image-20220127105243262](C:\Users\14240\AppData\Roaming\Typora\typora-user-images\image-20220127105243262.png)

![screenshot-20220127-105314](C:\Users\14240\Desktop\screenshot-20220127-105314.png)

![image-20220127113430733](C:\Users\14240\AppData\Roaming\Typora\typora-user-images\image-20220127113430733.png)

将snabbdom库github上库的demo复制到index.ts，打开浏览器，看到![screenshot-20220127-113530](C:\Users\14240\Desktop\screenshot-20220127-113530.png)

说明webpack运行打包成功

注意：要记得在index.html中加入一个容器container

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <h1>首页</h1>
    <div id="container"></div>
    <script src="./xuni/bundle.js"></script>
</body>

</html>
```

## h函数的使用：

作用：用来生成虚拟DOM（vnode）

#### 使用方式：

###### 单个使用

```
h('a',{props:{href:'www.baidu.com'}},'百度')
```

产生这样的虚拟节点

```json
{
   'sel':'a',
   'data':{
       "props":{
           "href":"www.baidu.com"
       }
   },
   "text":"百度",
}
```

他将会产生这样的真实dom

```html
<a href="www.baidu.com">百度</a>
```

一个虚拟dom有哪些属性：

```json
{
  children: undefined, // 子节点，undefined表示没有子节点
  data: {}, // 属性样式等
  elm: undefined, // 该元素对应的真正的DOM节点，undefined表示它还没有上树
  key: undefined, // 节点唯一标识
  sel: 'div', // selector选择器 节点类型（现在它是一个div）
  text: '我是一个盒子' // 文字
}
```

###### 嵌套使用：

```html
h('ul', {}, [
  h('li', {}, '牛奶'),
  h('li', {}, '咖啡'),
  h('li', {}, '可乐')
])
```

生成的虚拟dom节点：

```json
{
  "sel": "ul",
  "data": {},
  "children": [
    { "sel": "li", "data": {}, "text": "牛奶" },
    { "sel": "li", "data": {}, "text": "咖啡" },
    { "sel": "li", "data": {}, "text": "可乐" }
  ]
}
```

index.ts:

```ts

  const {  init, classModule, propsModule,styleModule,  eventListenersModule, h,} =require('snabbdom')

  const patch = init([
    // Init patch function with chosen modules
    classModule, // makes it easy to toggle classes
    propsModule, // for setting properties on DOM elements
    styleModule, // handles styling on elements with support for animations
    eventListenersModule, // attaches event listeners
  ]);
  
  const container = document.getElementById("container");
// 创建虚拟节点
const myVNode3 = h('ul', [
  h('li', '牛奶'),
  h('li', '咖啡'),
  h('li', [h('div', [h('p', '可口可乐'), h('p', '百事可乐')])]),
  h('li', h('p', '雪碧'))
])
// 让虚拟节点上树
patch(container, myVNode3)
//patch函数是将虚拟dom，转换为真实dom，并且将转换后的真实dom渲染到页面上
```

## 手写实现h函数

**首先我们要明确h函数的作用是生成虚拟dom(vnode)**

```js
/*
//我们只考虑三种形态
  形态①：h('div', {}, '文字')
  形态②：h('div', {}, [])
  形态③：h('div', {}, h())
*/
h函数本身有很多种使用方式，我们这里只考虑h函数的第三个参数为
1.文字字符串
2.拥有多个子节点的数组
3.只有一个子节点
```

我们此处需要实现两个函数：

- vnode函数--->主要是将h函数传入的参数，转换成一个虚拟dom对象
- h函数---->根据传入的第三个参数的不同，判断需要返回的虚拟dom

vnode函数：

```ts
//  ./snabbdom/vnode.ts

import { VNode} from "./h"
interface vnodeType{
    (sel:string,data:object,children:Array<VNode>|undefined,elm:Node|undefined,text:string|undefined):VNode
}

const vnode:vnodeType=function(sel,data,children,elm,text){
    return {sel,data,children,elm,text}
}
export default vnode
```

h函数：

```ts
//  ./snabbdom/h.ts
import vnode from "./vnode";
export type Key = string | number | symbol;
export type CType= Array<VNode> |string |undefined  |VNode
export interface VNode {
    sel: string | undefined;
    data: object| undefined;
    children: Array<VNode | string> | undefined;
    elm: Node | undefined;
    text: string | undefined;
    key?: Key | undefined;
  }
/*
//我们只考虑三种形态
  形态①：h('div', {}, '文字')
  形态②：h('div', {}, [])
  形态③：h('div', {}, h())
*/
//函数重载
export default function h(sel:string,data:object,c:string):VNode;
export default function h(sel:string,data:object,c:Array<VNode>):VNode; 
export default function h(sel:string,data:object,c:VNode):VNode;
export default function h(sel:string,data:object,c:CType):VNode |null{
    if(arguments.length!=3) {
        throw new Error("参数必须为3")
    }
    if(typeof c =='string'){
        return vnode(sel,data,undefined,undefined,c)
    }else if(Array.isArray(c)){
        let children:Array<VNode>=[]
        for (let i = 0; i < c.length; i++) {
            if (!(typeof c[i] == 'object' && c[i].hasOwnProperty('sel'))) {
                throw new Error("不是object")
            }
            children.push(c[i])
        }
        return vnode(sel,data,children,undefined,undefined)
    }else if(typeof c =='object'){
        let children:Array<VNode>=[c]
        return vnode(sel,data,children,undefined,undefined)
    }else{
        throw new Error("参数的类型不对")
    }
}
```

<!-- more -->