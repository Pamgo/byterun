### vue特点以及使用方式

#### vue入门

> 引入 vue.min.js

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <div id="app">
          <!-- {{}} 插值表达式，绑定vue中的data数据 -->
          {{message}}
    </div>
    <script src="vue.min.js"></script>

    <script>
        // 创建一个vue对象
        new Vue({
            el: '#app',//绑定vue作用的范围
            data: {//定义页面中显示的模型数据
                message: 'Hello Vue!'
            }
        })
    </script>
</body>
</html>
```

#### 测试代码片段

````html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">

    </div>
    <script src="vue.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            data: {
                
            }
        })
    </script>
</body>

</html>
````

#### 指令v-bind

> 单向数据绑定

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <!-- v-bind指令
            单向数据绑定
            这个指令一般用在标签属性里面，获取值
        -->
        <h1 v-bind:title="message">
            {{content}}
        </h1>

        <!--简写方式-->
        <h2 :title="message">
                {{content}}
            </h2>

    </div>
    <script src="vue.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            data: {
                content: '我是标题',
                message: '页面加载于 ' + new Date().toLocaleString()
            }
        })
    </script>
</body>

</html>
```

#### 指令v-model

> 双向数据绑定

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <input type="text" v-bind:value="searchMap.keyWord"/>
        <!--双向绑定-->
        <input type="text" v-model="searchMap.keyWord"/>

        <p>{{searchMap.keyWord}}</p>

    </div>
    <script src="vue.min.js"></script>
    <script>

        new Vue({
            el: '#app',
            data: {
                searchMap:{
                    keyWord: '开始绑定事件'
                }
            }
        })
    </script>
</body>

</html>
```

#### vue事件操作

> methods方法

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <!--vue绑定事件-->
        <button v-on:click="search()">查询</button>

        <!--vue绑定事件简写-->
        <button @click="search()">查询1</button>
    </div>
    <script src="vue.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            data: {
                searchMap:{
                    keyWord: '绑定数值'
                },
                //查询结果
                result: {}
            },
            methods:{//定义多个方法
                search() {
                    console.log('search....')
                },
                f1() {
                    console.log('f1...')
                }
            }
        })
    </script>
</body>

</html>
```

#### vue修饰符

> 防止表达form自动提交，而是使用自己定义的方法提交。 v-on:submit.prevent="onSubmit"

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <form action="save" v-on:submit.prevent="onSubmit">
            <input type="text" id="name" v-model="user.username"/>
            <button type="submit">保存</button>
        </form>
    </div>
    <script src="vue.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            data: {
                user:{}
            },
            methods:{
                onSubmit() {
                    if (this.user.username) {
                        console.log('提交表单')
                    } else {
                        alert('请输入用户名')
                    }
                }
            }
        })
    </script>
</body>

</html>
```

#### vue指令 v-if

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <input type="checkbox" v-model="ok"/>是否同意
        <!--条件指令 v-if  v-else -->
        <h1 v-if="ok">尚硅谷</h1>
        <h1 v-else>谷粒学院</h1>
    </div>
    <script src="vue.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            data: {
                ok:false
            }
        })
    </script>
</body>

</html>
```

#### vue指令 v-for

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <ul>
            <li v-for="n in 10"> {{n}} </li>
        </ul>
        <ol>
            <li v-for="(n,index) in 10">{{n}} -- {{index}}</li>
        </ol>

        <hr/>
        <table border="1">
            <tr v-for="user in userList">
                <td>{{user.id}}</td>
                <td>{{user.username}}</td>
                <td>{{user.age}}</td>
            </tr>
        </table>

    </div>
    <script src="vue.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            data: {
                userList: [
                        { id: 1, username: 'helen', age: 18 },
                        { id: 2, username: 'peter', age: 28 },
                        { id: 3, username: 'andy', age: 38 }
                    ]
            }
        })
    </script>
</body>

</html>
```

#### vue组件

> 自定义标签模版 components

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <Navbar></Navbar>
    </div>
    <script src="vue.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            //定义vue使用的组件
            components: {
                //组件的名字
                'Navbar': {
                    //组件的内容
                    template: '<ul><li>首页</li><li>学员管理</li></ul>'
                }
            }
        })
    </script>
</body>

</html>
```

#### vue 组件全局组件

> 1、定义一个js文件来作为全局组件Navbar.js

```js
// 定义全局组件
Vue.component('Navbar', {
    template: '<ul><li>首页</li><li>学员管理</li><li>讲师管理</li></ul>'
})
```

> 2、引入组件

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <Navbar></Navbar>
    </div>
    <script src="vue.min.js"></script>
    <script src="components/Navbar.js"></script>
    <script>
        new Vue({
            el: '#app',
            data: {
                
            }
        })
    </script>
</body>

</html>
```

#### vue生命周期

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
           hello
    </div>
    <script src="vue.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            data: {
            },
            created() {
                debugger
                //在页面渲染之前执行
                console.log('created....')
            },
            mounted() {
                debugger
                //在页面渲染之后执行
                console.log('mounted....')
            }
        })
    </script>
</body>

</html>
```

#### vue路由

> 使用vue-router.min.js

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
            <h1>Hello App!</h1>
            <p>
                <!-- 使用 router-link 组件来导航. -->
                <!-- 通过传入 `to` 属性指定链接. -->
                <!-- <router-link> 默认会被渲染成一个 `<a>` 标签 -->
                <router-link to="/">首页</router-link>
                <router-link to="/student">会员管理</router-link>
                <router-link to="/teacher">讲师管理</router-link>
            </p>
            <!-- 路由出口 -->
            <!-- 路由匹配到的组件将渲染在这里 -->
            <router-view></router-view>
    </div>

    <script src="vue.min.js"></script>
    <script src="vue-router.min.js"></script>

    <script>
            // 1. 定义（路由）组件。
    // 可以从其他文件 import 进来
    const Welcome = { template: '<div>欢迎</div>' }
    const Student = { template: '<div>student list</div>' }
    const Teacher = { template: '<div>teacher list</div>' }

    // 2. 定义路由
    // 每个路由应该映射一个组件。
    const routes = [
        { path: '/', redirect: '/welcome' }, //设置默认指向的路径
        { path: '/welcome', component: Welcome },
        { path: '/student', component: Student },
        { path: '/teacher', component: Teacher }
    ]

    // 3. 创建 router 实例，然后传 `routes` 配置
    const router = new VueRouter({
        routes // （缩写）相当于 routes: routes
    })

    // 4. 创建和挂载根实例。
    // 从而让整个应用都有路由功能
    const app = new Vue({
        el: '#app',
        router
    })
    </script>
</body>

</html>
```

#### axios简单使用

> 引入axios.min.js

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>

<body>
    <div id="app">
        <!--把userList数组里面数据显示 使用v-for指令 -->
        <div v-for="user in userList">
            {{user.name}} -- {{user.age}}
        </div>
    </div>
    <script src="vue.min.js"></script>
    <script src="axios.min.js"></script>
    <script>
        new Vue({
            el: '#app',
            //固定的结构
            data: { //在data定义变量和初始值
                //定义变量，值空数组
                userList:[]
            },
            created() { //页面渲染之前执行
                //调用定义的方法
                this.getUserList()
            },
            methods:{//编写具体的方法
                //创建方法 查询所有用户数据
                getUserList() {
                    //使用axios发送ajax请求
                    //axios.提交方式("请求接口路径").then(箭头函数).catch(箭头函数)
                    axios.get("data.json")
                        .then(response =>{//请求成功执行then方法
                            //response就是请求之后返回数据
                            //console.log(response)
                            //通过response获取具体数据，赋值给定义空数组
                            this.userList = response.data.data.items
                            console.log(this.userList)
                        }) 
                        .catch(error =>{
                        }) //请求失败执行catch方法
                }
            }
        })
    </script>
</body>

</html>
```

