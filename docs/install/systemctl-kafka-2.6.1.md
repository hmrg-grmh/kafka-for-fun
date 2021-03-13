# install (simple)

对于下面三者：

- jdk
- scala-sdk
- kafka

分别予以：

1. 解压 (建议在/opt下)
2. 配环境变量 (要有XXX_HOME)\
  
  示例：
  
  - jdk : `echo -e '\nJAVA_HOME=/opt/sdk/jdk/zulu-openjdk/zulu-8xxxx ;\nexport JAVA_HOME ; export PATH=$PATH:$JAVA_HOME/bin ;' > /etc/profile.d/jdk-path.sh`
  - scala-sdk : `echo -e '\nSCALA_HOME=/opt/sdk/scala/offical/scala-2.12.xx ;\nexport SCALA_HOME ; export PATH=$PATH:$SCALA_HOME/bin ;' > /etc/profile.d/scala-path.sh`
  - kafka : `echo -e '\nKAFKA_HOME=/opt/bigdatasoft/queue/kafka/kafka_2.12-2.6.1 ;\nexport KAFKA_HOME ; export PATH=$PATH:$KAFKA_HOME/bin ;' > /etc/profile.d/kafka-path.sh`



# service - systemctl

## 用户

增加用户组

```bash
groupadd bigdataservice
```

增加用户

```bash
useradd -m -s /bin/false -g bigdataservice zookeeper
useradd -m -s /bin/false -g bigdataservice kafka
```

更换拥有者

```bash
chown -chvR kafka:bigdataservice "$KAFKA_HOME"
```

## 启停脚本

<!-- ExecStart=$KAFKA_HOME/bin/zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties -->
<!-- ExecStart=$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties -->
在 `$KAFKA_HOME/bin` 下增加服务管理脚本 `kserverctl.sh` ，内容：

```bash
#! /bin/bash

#[ for: kafka-2.6.1 ]#

homepath_var_checker ()
{
    [[ "$1" != "" && -d "$1" ]] ;
} ;

homepath_var_checker "$KAFKA_HOME" ||
{
    rtc=$? ;
    echo '['"$(date +%FT%T.%3N%:::z)"'] ERROR : HOME Dir '"$KAFKA_HOME"' not found !!' >&2 ;
    exit $rtc ;
} ;

#[ make sure value of XXX_HOME is a legal dir ]#


source /etc/profile ||
{
    rtc=$? ;
    echo '['"$(date +%FT%T.%3N%:::z)"'] ERROR : run : source /etc/profile failed !!' >&2 ;
    exit $rtc ;
} ;

#[ make sure /etc/profile can be source ]#


service_type="${2:-kafka}" ;

case "$service_type" in
kafka|kfk) 
    service_name=kafka properties_file_name=server ;;
zookeeper|zk) 
    service_name=zookeeper properties_file_name=$service_name ;;
*) 
    {
        echo '['"$(date +%FT%T.%3N%:::z)"'] ERROR : para2 only can be : kafka/kfk/zookeeper/zk ' >&2 ;
        exit 6 ;
    } ;;
esac ;

#[ make sure para2 is legal , and init the needed vals ]#

#[ then , use values to run shell script ]#

{
    case "$1" in
    start) 
        ( $KAFKA_HOME/bin/${service_name}-server-start.sh $KAFKA_HOME/config/${properties_file_name}.properties & ) ;;
    stop) 
        $KAFKA_HOME/bin/${service_name}-server-stop.sh >&2 ;;
    restart) 
        { bash "$0" stop $service_name ; bash "$0" start $service_name ; } ;;
    *) 
        { echo '['"$(date +%FT%T.%3N%:::z)"'] ERROR : para1 only can be : start/stop/restart ' >&2 ; exit 3 ; } ;;
    esac
} ||
{
    rtc=$? ;
    echo '['"$(date +%FT%T.%3N%:::z)"'] ERROR : something might wrong ... '"($rtc)" >&2 ;
    exit $rtc ;
} ;

# end-of-file
```

## 注册服务

执行以将如下内容写入服务文件(并载入并启动)

<!-- ExecReload={ $KAFKA_HOME/bin/zookeeper-server-stop.sh >&2 ; $KAFKA_HOME/bin/zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties ; } -->
<!-- ExecStartPost=/usr/bin/bash -c '"'"'/usr/bin/echo && /usr/bin/echo '"'"'"'"'"'"'"'"'[OK] zookeeper.service is Started !'"'"'"'"'"'"'"'"' '"'"' -->
<!-- ExecStopPost=/usr/bin/bash -c '"'"'/usr/bin/echo && /usr/bin/echo '"'"'"'"'"'"'"'"'[Done] zookeeper.service has Stopped !'"'"'"'"'"'"'"'"' '"'"' -->

```bash
echo '
[Unit]
Description=ZooKeeper Service (in kafka-2.6.1)
After=network.target syslog.target

[Service]
Type=forking

User=kafka
Group=bigdataservice

ExecStart=/usr/bin/bash -c '"'"'source /etc/profile ; bash $KAFKA_HOME/bin/kserverctl.sh start zk'"'"'
ExecStop=/usr/bin/bash -c '"'"'source /etc/profile ; bash $KAFKA_HOME/bin/kserverctl.sh stop zk'"'"'
ExecReload=/usr/bin/bash -c '"'"'source /etc/profile ; bash $KAFKA_HOME/bin/kserverctl.sh restart zk'"'"'

# ExecStartPre=/usr/bin/echo '"'"'starting zookeeper.service ...'"'"'
# PrivateTmp=True

[Install]
WantedBy=bigdataservice.target
' > /usr/lib/systemd/system/zookeeper.service

systemctl daemon-reload

systemctl start zookeeper.service
systemctl status zookeeper.service

systemctl enable zookeeper.service

echo '
[Unit]
Description=Kafka Service (2.6.1)
After=zookeeper.service

[Service]
Type=forking

User=kafka
Group=bigdataservice

ExecStart=/usr/bin/bash -c '"'"'source /etc/profile ; bash $KAFKA_HOME/bin/kserverctl.sh start kfk'"'"'
ExecStop=/usr/bin/bash -c '"'"'source /etc/profile ; bash $KAFKA_HOME/bin/kserverctl.sh stop kfk'"'"'
ExecReload=/usr/bin/bash -c '"'"'source /etc/profile ; bash $KAFKA_HOME/bin/kserverctl.sh restart kfk'"'"'

# ExecStartPre=/usr/bin/echo '"'"'starting kafka.service ...'"'"'
# PrivateTmp=True

[Install]
WantedBy=bigdataservice.target
' > /usr/lib/systemd/system/kafka.service

systemctl daemon-reload

systemctl start kafka.service
systemctl status kafka.service

systemctl enable kafka.service

```


用法示例：

- 重新加载配置信息: `systemctl daemon-reload`
- 重载指定配置: `systemctl reload zookeeper.service`
- 启动 zookeeper: `systemctl start zookeeper.service`
- 关掉 zookeeper: `systemctl stop zookeeper.service`
- 查看进程状态及日志: `systemctl status zookeeper.service`
- 开机自启动: `systemctl enable zookeeper.service`
- 关闭自启动: `systemctl disable zookeeper.service`



**注意：如果你用root或者别的什么用户启动过kfk/zk的话，在他们的datadir(见它们的配置文件)中就会已经生成一些东西，而这些东西会成为你用别的用户启动服务的阻碍！！所以测试过后请删除。2.6.1的kafka以及其自带zk的配置文件在$KAFKA_HOME/config/xx.properties里，**

service文件里的`WantedBy=`字段的值应该是需要写成`multi-user.target`的，如果需要开机启动的话。

