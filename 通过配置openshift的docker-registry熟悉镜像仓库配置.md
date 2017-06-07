# 通过配置openshift的docker-registry熟悉镜像仓库配置
## 利用docker-registry以insecure-registry模式做镜像仓库
### docker-registry默认设置
openshift自带的集成docker-registry默认是insecure的,即用HTTP,且默认只有openshift集群内的节点能够访问。
```
$ oc get svc -n default
NAME                       CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
docker-registry            172.30.1.1       <none>        5000/TCP                  8d
```
openshift集群内的节点已经把--insecure-registry 172.30.0.0/16假如到的docker daemon的设置
```
ps -ef | grep dockerd
root      1687     1  0 09:07 ?        00:00:13 dockerd -D -g /var/lib/docker -H unix:// -H tcp://0.0.0.0:2376 --label provider=virtualbox --insecure-registry 172.30.0.0/16 --tlsverify --tlscacert=/var/lib/boot2docker/ca.pem --tlscert=/var/lib/boot2docker/server.pem --tlskey=/var/lib/boot2docker/server-key.pem -s aufs
```
### 将docker-registry服务用NodePort模式发布到外部
将docker-registry服务用NodePort模式发布到外部,假设其中一个Node节点的外部IP为192.168.1.100,外部docker可以通过
192.168.1.100:31000访问external-docker-registry,这个仓库仍然是insecure的。
```
$ oc expose dc/docker-registry --name=external-docker-registry1 --port=5000 --target-port=5000 --type=NodePort
service "external-docker-registry" exposed
$ oc get svc -o wide -n default
NAME                       CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE       SELECTOR
docker-registry            172.30.1.1       <none>        5000/TCP                  8d        docker-registry=default
external-docker-registry   172.30.231.67    <nodes>       5000:31000/TCP            2h        docker-registry=default
```
expose命令设定NodePort时,端口nodePort是随机生成的,如果想固定NodePort端口,一方面可以用yaml创建服务,也可以用oc edit更改
```
oc edit svc/external-docker-registry
```
### 利用docker命令行测试镜像仓库
#### 前提条件
1. docker必须用openshift识别的合法用户登录docker-registry
2. 该用户必须有权限访问openshift中相应项目的imagestream
3. 要获得登录openshift的token
4. 要把外部访问地址192.168.1.100:31000加入到客户端docker engine的--insecure-registry中
#### 使用admin账户登录docker-registry,获取它的token
```
$ oc login -u admin -p xxxxxxxxx
$ oc whoami -t
xxxxxxxxxxxxxxxxxyyyyyyyyyyyyy
```
#### 使用docker-machine创建一个新的docker engine,设置它的--insecure-registry
创建一个docker-machine
```
$ docker-machine create -d virtualbox \
    --engine-insecure-registry 192.168.1.100:31000 \
    --engine-insecure-registry 172.30.0.0/16 default
```
验证--insecure-registry参数已经设置
```
$ docker-machine ssh default
$ ps -ef | grep dockerd
/usr/local/bin/dockerd -D -g /var/lib/docker -H unix:// -H tcp://0.0.0.0:2376 --label provider=virtualbox --insecure-registry 192.168.99.100:31000 --insecure-registry 172.30.0.0/16 --tlsverify --tlscacert=/var/lib/boot2docker/ca.pem --tlscert=/var/lib/boot2docker/server.pem --tlskey=/var/lib/boot2docker/server-key.pem -s aufs
```
#### 利用admin用户和token作为密码登录docker-registry
```
$ docker login 192.168.1.100:31000 -u admin -p xxxxxxxxxxxxxxxxxyyyyyyyyyyyyy-sv4r4qhfJF8ys
Login Succeeded
```
这里192.168.1.100地址是客户端可以访问到的,而172.30.0.0/16是访问不到的。
