

# 下载安装包

```
wget https://dist.apache.org/repos/dist/release/pulsar/pulsar-manager/pulsar-manager-0.2.0/apache-pulsar-manager-0.2.0-bin.tar.gz
```



# 启动部署

```
tar -zxvf apache-pulsar-manager-0.2.0-bin.tar.gz
cd pulsar-manager
tar -xvf pulsar-manager.tar
cd pulsar-manager
cp -r ../dist ui

nohup ./bin/pulsar-manager > pulsar-manager.log 2>&1 < /dev/null &
```



# 添加超级用户

```
CSRF_TOKEN=$(curl http://10.69.5.171:7750/pulsar-manager/csrf-token)
curl \
    -H "X-XSRF-TOKEN: $CSRF_TOKEN" \
    -H "Cookie: XSRF-TOKEN=$CSRF_TOKEN;" \
    -H 'Content-Type: application/json' \
    -X PUT http://10.69.5.171:7750/pulsar-manager/users/superuser \
    -d '{"name": "admin", "password": "apachepulsar", "description": "super user", "email": "liangxxx@xxx.com"}'
```



# 访问Broker集群监控

```
http://x.x.x.x:7750/ui/index.html
admin/apachepulsar
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210319133256.png)

# **添加Environment** 



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210318195916.png)



# 访问Bookeeper监控

```
http://x.x.x.x:7750/bkvm
admin/admin
```



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210319132450.png)