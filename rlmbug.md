# rlm 'get ethernet address' bug： Wrong host for license (-4) Can't get ethernet address (-114): syserr: -19

## bug描述
  -  当license mac地址所在的网卡接口的ifindex>4999时rlm客户端so库）无法正常checkout license，抛出如下异常信息：
  ```
  Wrong host for license (-4) Can't get ethernet address (-114): syserr: -19
  ```
  版本:  rlm 14
    
  license 类型： node locked

## bug重现方法

- 将系统的ifindex增加到大于4999
    - 创建容器的过程会创建network interface，因为network interface的ifindex的分配时递增的，所以最终ifindex会大于4999。
    宿主机和容器的ifindex的值分配都是有宿主机操作系统统一分配的。
    这里使用2500是因为创建一个容器ifindex会增加2，因为容器内和宿主机上各增加一个网络interface
```bash
for ((i=1; i<=2500; i++))
do
    docker run --rm alpine  cat /sys/class/net/eth0/ifindex

done
echo "over"

```
- 运行一个容器执行license chekout脚本(这里使用python调用rlm客户端so库)，此时的容器里的eth0的ifindex是大于4999的，所以license checkout 会报错

```
    docker run --mac-address  00:12:34:56:78:90 -w /app -v "$(pwd)":/app -d --name forever python:3.11.4-slim-bullseye python checkout.py
```


## bug原因定位

分析发现在另一台机器上没有出现该故障
所以通过strace命令获取脚本在两台机器上执行是的系统调用日志

```
strace -o strace.log python checkout.py
```


结合报错信息和系统调用日志对比发现，异常的日志没有获取到eth0网卡的地址
日志里有大量的这种系统调用
```
ioctl(3, SIOCGIFNAME, {ifr_index=1})  = -1 ENODEV (No such device)
ioctl(3, SIOCGIFNAME, {ifr_index=2})  = -1 ENODEV (No such device)
ioctl(3, SIOCGIFNAME, {ifr_index=3})  = -1 ENODEV (No such device)
···
ioctl(3, SIOCGIFNAME, {ifr_index=4999})  = -1 ENODEV (No such device)
```
推测这些调用的目的是，从0遍历网卡接口的ifr_index，给系统调用ioctl来获取 网卡接口信息进而获取到网卡地址

但是发现 该遍历到 4999
```
ioctl(3, SIOCGIFNAME, {ifr_index=4999}) = -1 ENODEV (No such device)
```
就不再继续遍历了

这时我们通过如下命令
```
cat /sys/class/net/eth0/ifindex
```
查看了异常和正常网卡的ifr_index的实际值,发现发生异常的机器的网卡ifindex的实际值为5233 大于4999,而正常机器的网卡ifindex实际值为750小于4999
显然这里明白so库无法获取到网卡的原因是，没有遍历ifindex的全部范围[0,65535]导致

通过 ```objdump -d librlmclient.so``` 生成汇编代码 发现


有如下根据4999 0x1387作为比较条件的代码：
```
5daae:	81 7d b8 87 13 00 00 	cmpl   $0x1387,-0x48(%rbp)
5dab5:	0f 8e 2e fe ff ff    	jle    5d8e9 <get_linux_ether_addr+0x61>
5dabb:	eb 01                	jmp    5dabe <get_linux_ether_addr+0x236>
```

## 进一步印证了推测：

所以当license指定的mac地址对应的机器的网卡接口ifindex值大于4999时license无法使用
rlm库遍历的网卡索引值不够大导致了该bug的发生

