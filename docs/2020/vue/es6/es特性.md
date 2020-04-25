### es6特性

> EMCAScript为标准，javascript是其的一种实现

#### let变量作用范围

```js
<script>
   //es6如何定义变量，定义变量特点
   // js定义：var a = 1;
   // es6写法定义变量，使用关键字 let  let a = 10;
   //let定义变量有作用范围
   //1 创建代码块，定义变量
   {
       var a = 10
       let b = 20
   }
   //2在代码块 外面输出
   console.log(a)
   console.log(b)  //Uncaught ReferenceError: b is not defined

</script>
```

#### let定义变量特点

let定义的变量不能重复定义

````js
<script>
   var a = 1
   var a = 2

   let m = 10
   let m = 20 //Uncaught SyntaxError: Identifier 'm' has already been declared

    console.log(a)
    console.log(m) 
</script>
````

#### const定义变量

```js
<script>
    //定义常量
    const PI = "3.1415"
    //常量值一旦定义，不能改变
    //PI = 3  //Uncaught TypeError: Assignment to constant variable.

    //定义常量必须初始化
    const AA  //Uncaught SyntaxError: Missing initializer in const declaration
</script>
```

#### 数组结构

```js
<script>
    //传统写法
    let a=1,b=2,c=3
    console.log(a, b, c)

    //es6写法
    let [x,y,z] = [10,20,30]
    console.log(x, y, z)
</script>
```

#### 对象结构

```js
<script>
    //定义对象
    let user = {"name":"lucy","age":20}

    //传统从对象里面获取值
    let name1 = user.name
    let age1 = user.age
    console.log(name1+"=="+age1)

    //es6获取对象值
    let {name,age} = user
    console.log(name+"**"+age)

</script>
```

#### 模版字符串

```js
<script>
    //1 使用`符号实现换行
    let str1 = `hello,
        es6 demo up!`
    //console.log(str1)

    //2 在`符号里面使用表达式获取变量值
    let name = "Mike"
    let age = 20

    let str2 = `hello,${name},age is ${age+1}`
    //console.log(str2)

    //3 在`符号调用方法
    function f1() {
        return "hello f1"
    }

    let str3 = `demo, ${f1()}`
    console.log(str3)
</script>
```

#### 声明对象

```js
<script>
    const age = 12
    const name = "lucy"

    //传统方式定义对象
    const p1 = {name:name,age:age}
   // console.log(p1)

    //es6定义变量
    const p2 = {name,age}
    console.log(p2)

</script>
```

#### 定义方法简写

```js
<script>
    //传统方式定义的方法
    const person1 = {
        sayHi:function(){
            console.log("Hi")
        }
    }

    //调用
    person1.sayHi()

    //es6
    const person2 = {
        sayHi(){
            console.log("Hi")
        }
    }

</script>
```

#### 对象拓展运算符

```js
<script>
     //1 对象复制
     let person1 = {"name":"lucy","age":20}
     let person2 = {...person1}
     //console.log(person2)   //Uncaught SyntaxError: Unexpected token ...

     //2 对象合并
     let name = {name:'mary'}
     let age = {age:30}
     let p2 = {...name,...age}
     console.log(p2)
</script>
```

#### 箭头函数

```js
<script>
    //1 传统方式创建方法
    //参数 => 函数体
    var f1 = function(m) {
        return m
    }
    //console.log(f1(2))

    //使用箭头函数改造
    var f2 = m => m
   // console.log(f2(8))

   //2 复杂一点方法
   var f3 = function(a,b) {
       return a+b
   }
   //console.log(f3(1,2))

   //箭头函数简化
   var f4 = (a,b) => a+b
   console.log(f4(2,2))
</script>
```

