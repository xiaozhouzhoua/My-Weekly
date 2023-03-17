# openEuler编译安装Redis

### 安装依赖和常用工具
`yum -y install vim net-tools wget gcc make lrzsz`

### 下载安装包
`wget https://download.redis.io/redis-stable.tar.gz`

### 解压缩安装包
`tar -zxvf redis-stable.tar.gz`

### 进入解压目录后编译安装
`cd redis-stable`

`make PREFIX=/usr/local/redis install`

### 创建配置文件目录
`mkdir /usr/local/redis/conf`

### 拷贝配置文件模板
`cp redis.conf /usr/local/redis/conf/`

### 修改配置文件
```shell
vim /usr/local/redis/conf/redis.conf
#修改绑定IP
bind 0.0.0.0
#修改启动方式为多线程模式
daemonize yes
#设置密码 requirepass foobared
# requirepass 123456
#禁止公网访问
protected-mode no
```

### 修改服务启动文件
```shell
cd /lib/systemd/system
vim redis.service

# 添加内容
[Unit]
Description=Redis
After=network.target

[Service]
Type=forking
PIDFile=/var/run/redis_6379.pid
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/conf/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
### 重新加载service文件
`systemctl daemon-reload`

### 启动redis
`systemctl start redis`

### 开机自启动redis
`systemctl enable redis`

### 修改环境变量
```shell
vim /etc/profile

alias bat="/usr/local/bat/bat"
alias exa="/usr/local/exa/bin/exa"
export REDIS_HOME=/usr/local/redis

export TMOUT=1800
export HISTSIZE=5000
export JAVA_HOME=/usr/local/jdk1.8.0_312
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
export PATH=$JAVA_HOME/bin:$REDIS_HOME/bin:$PATH
```

### 生效
`source /etc/profile`

### 验证
```shell
redis-cli
AUTH Changeme_123
info server
quit
```

### IDEA连接Redis
驱动下载地址：https://github.com/DataGrip/redis-jdbc-driver/releases

host: 10.44.153.173
username: 空
password: Changeme_123
port: 6379

scheme选择所有

可以直接拷贝redis数据库连接，然后在其它工程下import from clipboard就可以导入了
