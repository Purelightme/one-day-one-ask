### 简介

[Hprose](https://hprose.com/#)（High Performance Remote Object Service Engine）是一款先进的轻量级、跨语言、跨平台、无侵入式、高性能动态远程对象调用引擎库。它不仅简单易用，而且功能强大。你无需专门学习，只需看上几眼，就能用它轻松构建分布式应用系统。

### 实践

##### tcp-server （php）

- 新建目录，cd进去，composer init，然后require hprose就可以了。

  ![9.1](images/9.1.png)

- 编写业务代码。

  ![9.2](images/9.2.png)

- 运行服务。这里运行两个实例，分别监听不同的端口：

  默认的1314端口： ![9.3](images/9.3.png)另一个1315端口：![9.4](images/9.4.png)

#### tcp-client（php）

- 新建目录，引入composer依赖同server。

- 编写业务代码。

  ![9.5](images/9.5.png)

  ！！！这里设置了两个地址，有***负载均衡***的作用。

- 运行客户端：

  ![9.6](images/9.6.png)

  查看server端输出：

  ![9.7](images/9.7.png)

#### tcp-client（node-js）

- 安装npm依赖。

  ![9.8](images/9.8.png)

- 编写业务代码

  ![9.10](images/9.10.png)

- 执行：

  ![9.9](images/9.9.png)

# 总结

至此，完成了php的server，并且通过php，nodejs都成功调用了服务。

```2019-08-06```

