### babel是什么

` Babel 是转码器，把es6代码转换成es5代码`

### babel怎么使用

1. 安装babel工具，使用命令

   ```shell
   # 在文件夹下初始化文件
   npm init
   # --global全局安装babel
   npm install --global babel-cli
   # 测试是否安装成功
   babel --version
   ```

2. 创建一个js文件，编写es6代码

   ```js
   let input = [1,2,3]
   // 将数组的每个元素+1
   input = input.map(item => item + 1)
   console.log(input)
   
   let iarr = [2,3,4]
   iarr = iarr.map(i => i + 1)
   console.log(iarr)
   ```

3. 在当前目录下创建babel配置文件`.babelrc`并添加内容

   ```json
   {
       "presets": ["es2015"],
       "plugins": []
   }
   ```

4. 使用命令安装es2015转码器(需要联网)

   ```shell
   npm install --save-dev babel-preset-es2015
   ```

5. 使用命令进行转码（最后）

* 根据文件转码

```shell
# 将es6的js代码转换成es5的js代码
babel src/xxx.js -o dist/xxx.js
```

* 根据文件夹转码

```shell
# 将src文件夹下的es6的js代码转换成es5的代码放到dist文件夹中
babel src -d dist
```



