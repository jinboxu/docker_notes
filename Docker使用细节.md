## Docker使用细节

#### 1. Docker的网络模式

参考文档:    https://www.cnblogs.com/gispathfinder/p/5871043.html

我们在使用docker run创建Docker容器时，可以用--net选项指定容器的网络模式，Docker有以下4种网络模式：

**· host模式，使用--net=host指定。**

**· container模式，使用--net=container:NAME_or_ID指定。**

**· none模式，使用--net=none指定。**

**· bridge模式，使用--net=bridge指定，默认设置。**



#### 2. 使用 docker exec 代替docker attach

https://www.cnblogs.com/keithtt/p/6997560.html



