
##　vue-router 简易版本实现

## vue-router实现原理
> 更新视图但不重新请求页面
js可以感知到url的变化，通过这一点，js动态的去修改当前要显示的组件。
在vue中的router-link标签，其实就是动态的根据Url的变化去更新此处展示的组件。在SPA单页面应用中，每次跳转就不需要在请求html了，html我们只请求一回。动态替换其中app标签内的组件。

> vue的单页面应用是基于路由和组件的，路由用于设定访问路径，并将路径和组件映射起来
### router-link是如何动态替换成组件的


## 几种模式(mode)
vue-router 提供了三种运行模式：
- hash: 使用 URL hash 值来作路由。默认模式。
- history: 依赖 HTML5 History API 和服务器配置。
- abstract: 支持所有 JavaScript 运行环境，如 Node.js 服务器端

## hash模式
> hash即浏览器url中#后面的内容，包含#。hash是URL中的锚点，代表的是网页中的一个位置，单单改变#后的部分，浏览器只会加载相应位置的内容，不会重新加载页面。

> 即#是用来指导浏览器动作的，对服务器端完全无用，HTTP请求中，不包含#。
每一次改变#后的部分，都会在浏览器的访问历史中增加一个记录，使用”后退”按钮，就可以回到上一个位置。
所以说Hash模式通过锚点值的改变，根据不同的值，渲染指定DOM位置的不同数据。

### hash模式测试
```js
window.addEventListener("hashchange",cb,false)
function cb(){
	console.log("hashchanged")
};
window.location.hash=5; // hashchanged
window.location.hash=6; // hashchanged


```
## hash模式模拟实现
### 初级版本
需要手动修改url中的 hash，来触发hashchange事件。来执行对应的回调

```js

class Routers{
    constructor(routersList){
        // 路由列表对象，用来存放所有路由与对应的回调
        this.routersList = routersList || {};
        // 页面初始化时，监听load事件，并执行当前hash对应的回调
        window.addEventListener('load',this.refresh,false);
        
        //不加下面这行监听hash的代码也可以，有个Bug,当hash改变时，并不会立即执行回调，而是需要再刷新一下页面。相当于一直在走load事件的回调。
        window.addEventListener('hashchange',this.refresh,false);
    }

    refresh(){
        this.currentUrl = location.hash.slice(1) || '/a' || '/';   //   '/a' 方便测试用
        // 执行当前路由的回调
        this.routersList[this.currentUrl] && this.routersList[this.currentUrl]();
    }
};

var routersList = { 
    "a": () => { console.log('a') }, 
    "b": () => { console.log('b') } 
}
var routers = new Routers(routersList);  

```
> 通过以上代码，我就实现了一个最简单的hash路由。
当我们手动修改 hash 的时候，就会执行对应的回调函数。

> 那么，问题来了，我们的项目中，是不会手动修改hash的，那么，我们需要提供一个方法，当用户交互时，我们调用对应的hash修改，并且渲染页面。我们本文不对渲染页面展开。重点分析hash路由的实现

### 过渡版本
新增push方法，可通过他来修改hash，并执行其回调
```js
class Routers{
    constructor(routersList){
        
        // 路由列表对象，用来存放所有路由与对应的回调
        this.routersList = routersList || {};
        // 页面初始化时，监听load事件，并执行当前hash对应的回调
        window.addEventListener('load',this.refresh,false);
        window.addEventListener('hashchange',this.refresh,false);
    }
    push(hash,cb){
        this.routersList[hash] = cb || (() =>{ });
        window.location.hash=hash;
        //this.refresh();
    }

    refresh(){
        this.currentUrl = location.hash.slice(1) || '/a' || '/';   //   '/a' 方便测试用
        // 执行当前路由的回调
        this.routersList[this.currentUrl] && this.routersList[this.currentUrl]();
    }

}

var routersList = { "a": () => { console.log('a') }, "b": () => { console.log('b') } };

var routers = new Routers(routersList);  
routers.push('c',()=>{console.log('c')});
routers.push('d',()=>{console.log('d')}); // push俩回，但是只执行了最终hash的回调，之前的 c 的回调并未执行 
console.log(routersList); // {a: ƒ, b: ƒ, c: ƒ, d: ƒ}

```
### 中级版本
>增加了，前进与后退。
依赖初始化数据。
并没有在前进后退时，完善history数组。

>需要考虑，真正的历史记录，与路由对象列表的产生方式。与存储方式。

```js 
class Routers{
    /**构造器方便测试，默认输入两个对象
     * @输入 路由列表对象，包括 hash 与其 对应的回调, 如 { "a": () => { console.log('a') }, "b": () => { console.log('b') }... }
     * @输出 路由列表数组，默认存储的 hash 历史  ["a","b"]
     * */ 
    constructor(routersList,hashHistory){
        // 路由列表对象，用来存放所有路由与对应的回调
        this.routersList = routersList || {};
        // 存储hash历史的 数组
        this.hashHistory = hashHistory || [];
        // 当前的hash索引，默认指向hash历史数组的末尾，前进时+1，后退时-1
        this.currentHashIndex = this.hashHistory.length-1;
        // 当页的hash值
        //this.currentHash = "";
        // 页面初始化时，监听load事件，并执行当前hash对应的回调
        window.addEventListener('load',this.refresh,false);
        // 页面hash被修改时，监听 hashchange 事件，并执行当前hash对应的回调
        window.addEventListener('hashchange',this.refresh,false);
    }
    /**
     * 1. 通过当前hash索引+1,得到下一个索引。并修改hash为下一个索引
     * 2. 通过监听 hashchange 来调用refresh执行回调
     * */ 
    go(){
        this.currentHashIndex++;
        console.log('下一个hash索引 ', this.currentHashIndex)
        //边界处理，如果路由当前索引小于history数组时，才能前时，否则 当前索引减1，并报错。
        if(this.currentHashIndex < this.hashHistory.length){
            location.hash = this.hashHistory[this.currentHashIndex]
        }else{
            //前进时，当前索引增加了1，但是没有对应的路由，所以需要减去1，否则，走到这一步边界时，需要调用back两次才会生效。
            this.currentHashIndex = this.currentHashIndex -1;
            throw new Error("路由数组中，没有前进路由了");
        }
    }
    /**
     * 1. 添加新的hash，和callback
     * 2. 并跳转到新的hash，通过监听 hashchange 来调用refresh执行回调
     * */ 
    push(hash,cb){
        //新增hash到hashHistory数组
        this.hashHistory.push(hash);
        //新增hash和对应callback到routerList对象
        this.routersList[hash] = cb || (() =>{ });
        // 修改当前页面hash为新增的hash
        window.location.hash=hash;
        // 更新当前的索引为改变history长度后的索引。指向其中最后一个hash
        this.currentHashIndex = this.hashHistory.length-1;
    }
    /**
     * 1. 通过当前hash索引-1,得到上一个索引。并修改hash为上一个索引
     * */ 
    back(){
        // 如果当前的hash索引小于0 就让其等于0;
        if(this.currentHashIndex <= 0){
            this.currentHashIndex = 0;
            throw new Error("已到hashHistory最顶端")
        }else{
            this.currentHashIndex = this.currentHashIndex - 1;
        }
        console.log('当前hash索引 ',this.currentHashIndex)
        location.hash = this.hashHistory[this.currentHashIndex]
    }
    /**
     * 1. 获取Url中的hash，并执行当前hash的回调
     * */ 
    refresh(){
        this.currentUrl = location.hash.slice(1) || '/';   //   '/a' 方便测试用
        // 执行当前路由的回调   
        this.routersList[this.currentUrl] &&　this.routersList[this.currentUrl]();
    }

}

var routersList = { "a": () => { console.log('a') }, "b": () => { console.log('b') } };
var hashHistory = ["a","b"]
var routers = new Routers(routersList,hashHistory);  
routers.push('c',()=>{console.log('c')});
console.log(routers);  // 此时，我们的当前hash是c , 路由实例为：{routersList: {…}, hashHistory: Array(3), currentHashIndex: 2}
// 通过实例的go,和 back 方法来前进 、 后退, 无需传参
routers.go()
routers.back()
// 记得边界处理


```

>  此版本，并没有存储所有前进后退时的 hash访问历史。 我们需要在go 和back的时候去增加历史到 hashHistory中。

> 此版本，只是模拟了前进与后退，和浏览器真正的 history相比是没有做到全部记录。

### 完善中级版本
>需要考虑，真正的历史记录，与路由对象列表的产生方式。与存储方式。
```js

```


#### hash 路由小结
- 实现关键点，一个维护路由历史的数组，存放所有的hash
- 一个维护hash与callback的键值对 对象
- 通过当前索引（默认是指向 历史数组的最后），自增，自减来完成前进后退

#### hash 路由 需要完善
- 与浏览器的历史记录保持一致
- 在前进、后退时，都需要去改变历史记录。
- 上面的例子中，我们只是在固定了 历史记录、与callback对象的情况下，来前进，后退

## History模式

HTML5 history Api 可以实现在不刷新整个页面的情况下，修改页面的url，
```js
window.history.pushState({},"","www.aa.com");
// 此时页面不会刷新，但是url改变为
// 从 https://juejin.im/editor/drafts/5dd68d6d6fb9a05a51701491
// 改变为 https://juejin.im/editor/drafts/www.aa.com


```

## vue-router使用方式
1. 下载 npm i vue-router -S
2. 在main.js中引入 import VueRouter from 'vue-router';
3. 安装插件 Vue.use(VueRouter);
4. 创建路由对象并配置路由规则
let router = new VueRouter({routes:[{path:'/home',component:Home}]});
5. 将其路由对象传递给Vue的实例，options中加入 router:router
6. 在app.vue中留坑
<router-view></router-view>

## vue-router 其他属性
#### scrollBehavior
scrollBehavior 方法接收 to 和 from 路由对象。第三个参数 savedPosition 当且仅当 popstate 导航 (通过浏览器的 前进/后退 按钮触发) 时才可用。
```js
const router = new VueRouter({
  routes: [...],
  scrollBehavior (to, from, savedPosition) {
    // return 期望滚动到哪个的位置
  }
})
```
##### 模拟『滚动到锚点』的行为

```js
scrollBehavior (to, from, savedPosition) {
  if (to.hash) {
    return {
      selector: to.hash
    }
  }
}
```
##### 与keepAlive结合，如果keepAlive的话，保存停留的位置
```js
scrollBehavior (to, from, savedPosition) {
     if (savedPosition) {
            return savedPosition
    } else {
        if (from.meta.keepAlive) {
        　　from.meta.savedPosition = document.body.scrollTop;
        }
        return { x: 0, y: to.meta.savedPosition || 0}
    }
}
```
##### 相关的兼容性问题
使用keep-alive标签后部分安卓机返回缓存页位置不精确问题

##### 关于滚动到指定位置 相关知识点
- 获得页面向左、向上卷动的距离

为了兼容所有浏览器，封装一个函数，去获取页面向上卷曲出去的距离和向左卷曲出去的距离


```js
function getScroll(){
    return {
        left: window.pageXOffset || document.documentElement.scrollLeft || document.body.scrollLeft || 0,
        top: window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop || 0
    };
}
```



```js

```


## 总结
- hash模式和history模式都是不刷新更新页面的URL。
- history模式会走服务器，需要后端配合。

## 注意点
- history模式下，在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 index.html 页面，这个页面就是你 app 依赖的页面
如：

```js
export const routes = [ 
  {path: "/", name: "home", component:Home}
  {path: "/a", name: "a", component: A},
  {path: "/login", name: "login", component: Login},
  {path: "*", redirect: "/"}]

```

- history模式，的前端路由路由要有对应的同样的后端路由。


## 思考点
- vue路由为什么不用a标签，-> 页面刷新
- 

