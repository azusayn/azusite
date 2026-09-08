---
title: JavaScript 学习笔记
date: 2023/9/5 20:01:00
categories:
- Frontend
tags:
- js
- web
---


### 0. 随时可以访问官网查阅文档或者示例

### 1. 问题与笔记

- `undefined` 和 `null` : 前者代表声明了但没有分配的值，后者代表一个分配了的空值

- **Promise** 的用法 

	- 首先根据 [**MDN**](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Asynchronous/Promises) 文档: 

		> Promise 有三种状态：
		>
		> - **待定（pending）**：初始状态，既没有被兑现，也没有被拒绝。这是调用 `fetch()` 返回 Promise 时的状态，此时请求还在进行中。
		> - **已兑现（fulfilled）**：意味着操作成功完成。当 Promise 完成时，它的 `then()` 处理函数被调用。
		> - **已拒绝（rejected）**：意味着操作失败。当一个 Promise 失败时，它的 `catch()` 处理函数被调用。

	- 通过 **Promise** 对象传递异步函数运行的结果，对象拥有 `then` , `catch` 等方法 

	- 任何一个 声明为 `async` 的函数都应该拥有 **Promise** 类型的返回值, 下文是一个 ‘烤鱼’ 的例子

    ```javascript
    	async function cook(raw_food) {
          return new Promise((resolve, reject) => {
            if(raw_food === "fish") {
              resolve(`cooked ${raw_food}`)   // return value
            } else {
              reject(`we do not cook ${raw_food}`)
            }
          }) 
        }
    
        cook("fish")
          .then((val) => {
            console.log(`you've got a ${val}`)
          })
          .catch((reason) => {
            console.log(`you've got nothing because ${reason}`)
          })
    	// 输出：you've got a cooked fish
    ```
  
  - 同时可以使用 `Promise.all` 方法等待多个 **Promise** 执行完成，在上述例子的基础上换一个写法。注意这个方法会在所有 **Promise** 都成功后触发 `then` ，假如其中任意一个 **Promise** 失败则会触发 `catch`  返回触发该失败的原因
  
    ```javascript
        var p_0 = cook("fish")
        var p_1 = cook("laptop")
        var p_2 = cook("meat")
    
        Promise.all([p_0, p_1, p_2])
          .then((vals) => {
            for(v of vals) {
              console.log(`you've got a ${v}`)
            }
          })
          .catch((reason) => {
              console.log(`you've got nothing because ${reason}`)
          })
    	// 输出：you've got nothing because we do not cook laptop
    ```


- **getter** 和 **setter** 的使用方法，<u>注意不能看作普通函数调用</u>

    ```javascript
        class Person {
          set setInfo(val) {
            this.age = val.age
            this.name = val.name
          }
    
          get getInfo() {
            return `My name is ${this.name}, I am ${this.age} `
          }
        }
    
        let p = new Person()
    
        p.setInfo = {
          age: 12,
          name: "Tommy"
        }
    
        console.log(p.getInfo)
    ```
    
- 使用 `new` 调用构造函数：在 **JavaScript** 中除了类的构造函数，还可以将普通函数定义为构造函数并用 `new` 触发

    ```javascript
    	function Person(name, age) {
          this.name = name
          this.age = age
        }
    
        console.log(new Person("Tommy", 21))
    ```

- for 中的 `in` 和 `of`

    - 使用 `in` 用于遍历对象字段的键

    	```javascript
    		let p = {
    	      name: "Tommy",
    	      age: 21,
    	      blood_type: "A"
    	    }
    	
    	    for (const key in p) {
    	      console.log(`${key}: ${p[key]}`)
    	    }
    	```
	- `of` 一般用于遍历容器，比如数组等
		
		```javascript
			let arr = [1, 2, 5, 7]
		    for (const val of arr) {
		      console.log(val)
		    }
		```
	
	
	
- 事件捕获和冒泡：一个事件的处理会先经过事件捕获阶段，再经过事件冒泡阶段。事件捕获阶段，事件从父元素逐层传递到子元素，事件冒泡阶段则反之。当我们在给元素添加监听器的时候 `addEventListener()` 的第三个参数如果为 **true** ，则会在事件捕获阶段进行监听，不写则默认为 **false**  。

- 传入参数是值复制还是引用？一般来说可以理解为：基本类型复制值，其他对象传引用（细节可参考 [文章]([JavaScript 是传值调用还是传引用调用？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/25314908)) ）

- 函数柯里化: 将函数的参数绑定已知的值返回一个新的函数，减少重复代码。比如下列例子可以生成对 `add(1，4)(2)(5)` 这样的函数的柯里化。**需要注意的是** ，由于 **JavaScript** 独特的闭包机制 (**Closure**) , 变量可以访问作用域之外的值，同时作用域外的值会保存到不再被引用时。不像 C++ 这类函数变量压栈的语言，变量不会因为离开函数作用域就被直接释放。所以在下文的例子中 `args` 并不会在离开 `add()` 函数的作用域就被释放，而是可以被不断调用。(说实话听起来有点不靠谱。。。)

    ```javascript
    	function add() {
          let args = Array.prototype.slice.call(arguments)
          let inner = function() {
            args.push(...arguments)
            let sum = 0
            for(val of args) {
              sum += val
            }
            return inner
          }
          return inner
        }
    
        console.log(add(1)(3))
    ```

- **JavaScript** 中的 `this` 并不只限于类中，而可以出现在多个语境中。广义上讲， `this` 表示的是函数指向的某个数据。这个 `this` 一般指向的是调用当前函数的对象（不能是箭头函数，箭头函数中的 `this` 指代的是所属作用域的 `this`, 如果在全局则指代全局对象）, 称之为隐式绑定。但也可以通过显式绑定设置为自定义的对象，见下一个条目的三个方法

- 显式绑定 `call`,  `bind` 和  `apply` 。三者本质大差不差，用法略有不同

    ``` javascript
    	let tommy = {
          power: 100,
          charge: function (inc_0, inc_1)  {
            this.power += (inc_0 + inc_1)
          }
        }
    
        let john = {
          power: 50
        }
    
        tommy.charge.call(john, 10, 10)
        tommy.charge.apply(john, [10, 10])
        tommy.charge.bind(john)(10, 0)
    
        console.log(john) // {power: 100}
    ```

    

- 箭头函数与普通函数的不同

    - 箭头函数不能用作构造函数，所以 `this` 并不能指向构造的函数，而是等于所属的作用域的 `this` 的值
    - 函数没有 `proyotype` 、`arguments` 等内置对象
    - 不可用显式绑定修改绑定的对象

- **CommonJS** 和 **ECMAScript** 模块（**ESM**）是两种不同的 **JavaScript** 模块化标准。它们之间有一些主要区别，包括：

	- **语法**：**CommonJS** 使用 `require` 和 `module.exports` 来导入和导出模块，而 **ECMAScript** 模块使用 `import` 和 `export` 关键字。
	- **运行时/编译时**：**CommonJS** 模块是在运行时动态加载的，这意味着您可以在运行时根据条件来决定是否导入某个模块。而 **ECMAScript** 模块是在编译时静态加载的，这意味着您必须在代码顶部显式地声明所有要导入的模块。
	- **环境支持**：**CommonJS** 最初是为 **Node.js** 设计的，因此它在 Node.js 环境中得到了广泛支持。而 **ECMAScript** 模块是 JavaScript 语言标准的一部分，因此它在现代浏览器中得到了广泛支持。不过，**Node.js** 也支持 **ECMAScript** 模块。

	

	