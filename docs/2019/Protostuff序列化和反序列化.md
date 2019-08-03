## Protostuff序列化和反序列化
序列化和反序列化是在应对网络编程最常见的问题之一。
序列化就是将java对象转化成byte[];反序列化就是将byte[]转化成java对象，Protostuff属于一个更高效的序列化库。
## 引入依赖
引入依赖如下：

	        <dependency>
                <groupId>io.protostuff</groupId>
                <artifactId>protostuff-core</artifactId>
                <version>1.6.0</version>
            </dependency>
            <dependency>
                <groupId>io.protostuff</groupId>
                <artifactId>protostuff-runtime</artifactId>
                <version>1.6.0</version>
            </dependency>

![类方法](https://github.com/Pamgo/byterun/blob/master/docs/_media/ProstuffIOUtil.png)