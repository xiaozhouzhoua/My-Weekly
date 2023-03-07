# 开发中遇到的Maven问题

## 访问setting文件中仓库地址显示406，不允许以http访问，全部改成https后，无法下载相关依赖，https下，无法下载依赖的问题could not transfer artifact xxx from xx

### 解决

File-settings-> maven -> importing -> vm options for importer

```
-Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true
```

File-settings-> maven -> Runner
```
-Dmaven.wagon.http.ssl.insecure=true
-Dmaven.wagon.http.ssl.allowall=true
-Dmaven.wagon.http.ssl.ignore.validity.dates=true
-DarchetypeCatalog=internal
```
## java.lang.IllegalArgumentException: Malformed \uxxxx encoding while mvn install

> 原因：下载的包中properties文件含有乱码字符

1、设置断点到错误提示类`java.util.Properties`的Malformed \uxxxx错误抛出行

2、运行处增加maven启动，这样可以debug形式调试maven

3、debug启动maven，到断点处，断点线程堆栈下找到TrackingFileManager.read(File)

4、找到file字段，拷贝路径找到出错的文件，打开编辑器，删除掉有问题的内容，再次执行maven

## 一些maven插件无法下载，如clean插件

在中央仓库输入对应插件名称，找到插件，下载jar包和pom文件(也可以拷贝pom文件内容)，然后到本地的插件路径
如：org->apache->maven->plugins->maven-xxx-plugin->将下载的jar包和pom文件放在这

## maven依赖已下载，右侧maven栏依赖下划线还是飘红

> 有缓存导致

file -> Invalidate Cache..重启后会自动索引依赖，就不会报错了


项目仓库配置
```xml
<repositories>
    <repository>
        <id>alimaven</id>
        <name>aliyun maven</name>
        <url>https://maven.aliyun.com/repository/public</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

