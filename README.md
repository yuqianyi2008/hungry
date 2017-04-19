# hungry

### 模块

* 对于就业方向出发,学习Node.js需要掌握的内容有一下几点:

  1. 模块机制,内置功能一些原理

     每个 node 进程只有一个 VM 的上下文, 不会跟浏览器相差多少, 模块机制在文档中也描述的非常清楚了:

     ```
     function require(...) {
       var module = { exports: {} };
       ((module, exports) => {
         // Your module code here. In this example, define a function.
         function some_func() {};
         exports = some_func;
         // At this point, exports is no longer a shortcut to module.exports, and
         // this module will still export an empty default object.
         module.exports = some_func;
         // At this point, the module will now export some_func, instead of the
         // default object.
       })(module, module.exports);
       return module.exports;
     }
     ```




> Q:如果 a.js require 了 b.js, 那么在 b 中定义全局变量 `t = 111` 能否在 a 中直接打印出来?
>
> A:可以,因为每个模块node都在外面包了一个自调用,当require这个模块,模块中的全局变量就会在全局赋值成功,在别的js文件中就能拿到(自己理解).

2. 关于`commonJs`的一些问题. 比如比较 `AMD`, `CMD`, `commonJs`三者的区别, 包括询问关于 node 中 `require` 的实现原理等

3. 热更新

   在 Node.js 中做热更新代码, 牵扯到的知识点可能主要是 `require` 会有一个 `cache`, 有这个 `cache` 在, 即使你更新了 `.js` 文件, 在代码中再次 `require` 还是会拿到之前的编译好缓存在 v8 内存 (code space) 中的的旧代码. 但是如果只是单纯的清除掉 `require` 中的 `cache`, 再次 `require` 确实能拿到新的代码, 但是这时候很容易碰到各地维持旧的引用依旧跑的旧的代码的问题. 如果还要继续推行这种热更新代码的话, 可能要推翻当前的架构, 从头开始从新设计一下目前的框架.

   不过热更新 json 之类的配置文件的话, 还是可以简单的实现的, 更新 `require` 的 `cache` 可以实现, 不会有持有旧引用的问题, 但是如果旧的引用一直被持有很容易出现内存泄漏, 而要热更新配置的话, 为什么不存数据库? 或者用 `zookeeper` 之类的服务? 通过更新文件还要再发布一次, 但是存数据库直接写个接口配个界面

4. 上下文防止被污染:通过创建新的上下文沙盒 (sandbox) 可以避免上下文被污染

   Q:`为什么 Node.js 不给每一个`.js`文件以独立的上下文来避免作用域被污染?`

   A:关键代码就这么几行。

   content 可以认为是你的 .js 文件源码，例如，我们简单点：

   ```
   'console.log(module)'

   ```

   Module.wrap(content) 后：

   ```
   '(function (exports, require, module, __filename, __dirname) { console.log(module)\n});'

   ```

   > 注意：上面是字符串操作。

   vm.runInThisContext 后：

   ```
   [Function]

   ```

   将上面的字符串输出变成了可执行的 JS 函数。实际上这个函数就是：

   ```
   function(exports, require, module, __filename, __dirname) {
     console.log(module)
   });

   ```

   最后执行这个函数，也就是 compiledWrapper.call(this.exports, this.exports, require, this, filename, dirname) 就是执行了这个模块。

   以上，就是 Node.js 对一个文本的 .js 模块转换成一个可使用的 JS 模块的大致过程。

   好了，回答题主问题，**显然，.js 文件的代码都是包裹在一个函数里执行的，并不会产生作用域污染。**

   我们再追问一下，**如果你不小心没有写var，定义了全局变量怎么办？**，例如一不小心写了这行代码：

   ```
   globalVar = 1

   ```

   包裹之后变成了：

   ```
   function(exports, require, module, __filename, __dirname) {
     globalVar = 1
   });

   ```

   那这个 globalVar 是啥样子？这其实是由 vm.runInThisContext 决定的，[查看 Node.js 的文档**](https://link.zhihu.com/?target=https%3A//nodejs.org/api/vm.html%23vm_vm_runinthiscontext_code_options)：

   > vm.runInThisContext() compiles code, runs it within the context of the current global and returns the result. Running code does not have access to local scope, but does have access to the current global object.

   1. vm.runInThisContext 使得包裹函数执行时无法影响本地作用域；
   2. 但 global 对象是可以访问的，因此 globalVar = 1 等价于 global.globalVar = 1

   如何避免这种对全局作用域的污染呢？

   ```
   'use strict';
   globalVar = 1

   ```

   添加 'use strict';，禁止这样意外创建全局变量，代码执行时将抛出 globalVar 未定义的错误。

   更准确地回答题主的问题：**Node.js 模块正常情况对作用域不会造成污染，意外创建全局变量是一种例外**，可以采用严格模式来避免。



* 模块章节的新知识点
  1. require.cache 删除已经加载的模块的缓存区,require.cache['模块名']可删除指定缓存模块
  2. VM主机简称VM, 又称VM服务器. VM主机是灵动网络利用虚拟机(Virtual Machine)技术, 将一台服务器分割成多个虚拟机(VM主机)的优质服务. 这些VM主机以最大化的效率共享硬件、软件许可证以及管理资源. 对其用户和应用程序来讲, 每一个VM主机平台的运行和管理都与一台独立主机完全相同。
  3. 热更新的时候不需要关闭服务器，直接重新部署项目就行。冷的自然就是关闭服务器后再操作,简单举例:就是说你的卡车开到了150KM/H,然后，有个轮胎，爆了,然后，司机说，你就直接换吧，我不停车。你小心点换









