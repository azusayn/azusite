---
title: Typescript 学习笔记
date: 2023/9/5 17:06:00
categories:
- Frontend
tags:
- ts
- web
---
### 0. 随时可以访问官网查阅文档或者示例

- 首页：[TypeScript: JavaScript With Syntax For Types.](https://www.typescriptlang.org/)
- 在线编辑器：[Playground](https://www.typescriptlang.org/play)
- Cheet Sheet: [TypeScript: Cheat Sheets (typescriptlang.org)](https://www.typescriptlang.org/cheatsheets)、
- 学习过程中敲的代码 [Code](./HelloWorld.ts)
### 1. 基本环境配置

- 通过 **npm** 包管理安装 **typescript** :

	```bash
		npm install typescript --save-dev
		# 使用 --save-dev 选项安装的包会限定在生产测试环境中而不会影响到部署环境
		# 如果想全局安装请使用 -g 选项
	```

- 为了更好的管理工程，我们应该使用 **tsconfig.json** 文件配置环境

	```bash
		# 这将自动生成一个tsconfig.json文件
		tsc --init 
	```
	- 将 **tsconfig.json** 中的 **rootDir** 项设置为 **ts** 源文件的文件夹
	
	- 在 **tsconfig.json** 末尾添加 **include** 项，以防将与 **tsconfig.json** 同文件夹的文件编译 
	
		```bash
			"include": ["[SRC_DIR]"] # 将[SRC_DIR]设置为源文件文件夹	
		```
	
	- 将 **outDir** 项设置为 **js** 文件的输出文件夹
	
- 接着就可以使用 **npx** 运行编译器将 **.ts(x)** 文件编译成 **.js(x)** 文件

	```bash
		npx tsc # 假如不想设置tsconfig.json, 这条指令后面应该指定ts源文件，即可跳过上述第二步
	```

- 使用 `-watch` 选项可以让 **tsc** 进程在后台自动编译文件 ( 或者 `-w` ) 

	``` bash
		npx tsc -w
	```

### 2. 问题与笔记
1)   编译选项 `--noEmitOnError` 的作用：该选项默认为 **false** ，代表源文件编译错误时仍然会输出 **js** 文件 

2. `instanceof` 和 `typeof` : 前者用于判断**类**实例与某个类的关系 (可以判断继承关系)，但是不能用于接口。后者可以用于类与接口，可以判断实例的基本类型，这两个方法属于 **javascript** 方法

3. `in` , `is` 和 `as` 

  - `in` 用于类型判断，可以判断某个字段是否存在于类型中

    ```typescript
      let p:Person = {gender:"male", swim: () => {}};
      if("gender" in p){
        console.log("It has member gender")
      }
    ```


  - `is` 用于定义类型检查函数，写在返回 `bool` 的函数返回值中，编译器根据布尔值确定变量类型

    ```typescript
        function isString(p: any): p is string {
            return true
        }
    
        if(true) {
            let s: any = "Hello"
            if(isString(s)){
                console.log(typeof s)
            }
        }
    ```
    
  - `as` 用于告诉编译器某个变量的类型以通过编译

    ```typescript
        let s: unknown = "HELLO"
        console.log((s as string).toLowerCase())
    ```

4. `keyof`: 用于返回一个类型中的字段的类型的 **Union**

5. **Meta Types** ：在 **typescript** 中操作类型的不同方法（用于操纵 **type** 和 **interface**）：

     - **Mapped Types**：意指用现有的类型生成新的类型。如下所示，根据原来的 **Person** 类型生成一个只读的 **Readonly\<Person>** 类型

        ```typescript
          type Readonly<T> = {
            readonly [P in keyof T]: T[P];
          };
        
          type Person = {
            name: string;
            age: number;
          };
        
          type ReadonlyPerson = Readonly<Person>;
        
        ```
     - **Conditional Types**：用 `?` 结合 `extends` 关键字生成类型
       
        ```typescript
            type T = String extends Object ? never: String
        	# 也可以结合泛型
            type Ty<T> = T extends Object ? never : String 
        ```
        
     - **Indexed Types**：可以用下标运算符操作某个类型以获得某个字段的类型
        ```typescript
          interface Test {
            name: string
          }
          type IDXType = Test["name"]
        ```
        
     - **Discriminate Types**：允许用键值对临时定义一个类型
        ```typescript
          let a : {name: string} = {
            name: "Tommy"
          }
        ```

6. 函数签名 ：
     - **Construct Signatures**：用于**描述构造函数类型信息**的语法 (在这里只要是生成新对象的函数就是构造函数), 在 **Call Signatures** 之前使用 `new` 关键字即可完成一个构造签名
       
        ```typescript
          type SomeConstructor = {
            new (s: string) : String; 
          }
              
          function fn_(ctor: SomeConstructor) {
            return new ctor("hello");
          }
        
          fn_(String) // 传入 String 的构造函数也就是 String
        ```
     - **Call Signatures**：用于**描述函数类型信息**的语法, 写法类似于 **lambda** 
       
        ```typescript
          function f(v: number): void {
            console.log('do nothing')
          }  
        
          let func: (val: number) => void = f
        ```
     
7. **.d.ts** 文件一般用于描述 javascript 库的类型信息，指导编译器更好地工作

8. `unknown` 、 `never` 和 `any` : 

     - `never` 表示一个永远不会发生或者用到的类型，比如一个函数永远不会正常返回时，可以将返回值设置为 `never`
     - `unknown` 代表任何值，但是需要进行类型检查或者断言才能保证通过编译，推荐使用 `unknown` 作为 `any` 的代替
     - `any` 代表任何值，可以躲过 **typescript** 的检查

9.  文档中 argument 与 parameter 的区别：前者代表函数传入的实际参数，后者代表函数定义中的参数