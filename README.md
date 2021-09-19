# docker_macvlan_same_subnet_with_host_and_ipv6 
docker容器配置macvlan及设置容器与宿主机同一ipv4网段并通信，另外还配置macvlan获得公网IPv6地址 
  
前面我的repo已经分享了docker容器使用缺省bridge模式获得公网IPv6地址并路由的配置方法，但是容器还可以有其他网络模式，  
满足不同的应用发布需求，例如macvlan模式，它主要解决容器实例的ipv4需要与宿主机同一网段，而且还有互通需求，  
所以一并把我成功的操作流程记录一下  
  
操作流程假设你已经从服务供应商获得ipv6公网地址段::/64，当前局域网的ip网段10.0.0.0/24  
部署容器的宿主机对外网卡为ens33，宿主机系统为ubuntu 18  
  
首先查看宿主机网卡ipv6地址段  
#ip -f inet6 addr show ens33  
  
选取的是公网IP，也就是后面scope global指示的，下面的ipv6子网为2a01:53c0:ff0e:2e::/64  
inet6 2a01:53c0:ff0e:2e::b139/128 scope global dynamic noprefixroute  
inet6 2a01:53c0:ff0e:2e:20c:29ff:feff:1453/64 scope global dynamic mngtmpaddr noprefixroute  
  
编辑或追加配置文件内容，至少要定义一个ipv6子网段，而且这个子网段必须在::/64内，否则报错  
#nano /etc/docker/daemon.json  
写入下面内容  
  
{  
"ipv6": true,  
"fixed-cidr-v6": "2a01:53c0:ff0e:2e:2::/80"  
}  
  
保存退出并重启容器服务  
#systemctl restart docker  
  
创建容器网络模式macvlan并使用当前局域网的ip网段及公网ipv6网段，前提是必须配置好/etc/docker/daemon.json文件  
必须加上--ipv6参数，否则即使在管理界面看到网络模式名字分配了网段，而容器实例即使指定了ipv6地址，  
但容器实例ipv6地址实际为空，ipv6的网关参数可以不写，系统会自动指定::1，子网段不能用::/64，否则报错，  
也不要与已经定义的其他网络模式的网段重叠，  
--subnet：当前局域网的ipv4网段，--aux-address：排除本宿主机的ipv4地址，parent：指定使用的网卡名字  
#docker network create -d macvlan --ipv6 --subnet=2a01:53c0:ff0e:2e:3::/80 --subnet=10.0.0.0/24 --gateway=10.0.0.1 --aux-address="exclude_host=10.0.0.206" -o parent=ens33 ip6macvlan  
  
检查创建是否成功  
#docker network ls  
  
创建两个容器实例并使用macvlan模式，指定ipv6地址，指定IPv4地址，如果不指定的话会自动分配，  
建议自己指定，可避免IPv4地址冲突问题，验证指定的ipv4地址是否能够与宿主机ipv4通信   
  
#docker run -dit --restart=always --network=ip6macvlan --ip6=2a01:53c0:ff0e:2e:3::2 --ip=10.0.0.192 --name=u18ip6macvlan2 -v /data:/data ubuntu:bionic-20210827 /bin/bash -c "/etc/init.d/cron start;/etc/init.d/run;/bin/bash"  
  
#docker run -dit --restart=always --network=ip6macvlan --ip6=2a01:53c0:ff0e:2e:3::3 --ip=10.0.0.193 --name=u18ip6macvlan3 -v /data:/data ubuntu:bionic-20210827 /bin/bash -c "/etc/init.d/cron start;/etc/init.d/run;/bin/bash"  
  
  
无论如何宿主机的原生ipv4是不能直接和本机新创建的容器通信的，所以另外创建一个macvlan并把它设置成桥接组，  
至于网上很多的教程都设置一个ip地址给这个桥接组，经过验证，不需要设置桥接ip地址，只要是接口up及设置了路由，  
那么容器和宿主机就能互相通信，如果各位没有宿主机与本机容器互通需求，那么可以忽略下面的配置了，  
因为这时容器已经可以和任何局域网内10.0.0.0/24网段的机器互通，除了宿主机  
  
#ip link add link ens33 macvlan0-host type macvlan mode bridge  
  
#ip link set dev macvlan0-host up   
  
在macvlan模式下这个ip地址设置是不需要的，不过网上教程都有  
#ip addr add 10.0.0.254/24 dev macvlan0-host  
  
修改路由，使宿主机到上面两个容器ipv4地址10.0.0.192及10.0.0.193的通信全部转发给macvlan0-host进行桥接广播  
#ip route add 10.0.0.192 dev macvlan0-host  
#ip route add 10.0.0.193 dev macvlan0-host  
  
  
最后是在宿主机ping两个容器的ip地址及分别在两个容器内部ping宿主机ip地址，验证配置正确  
  
由于ip命令在linux重启后会丢失，所以需要把命令写在宿主机的定时任务@reboot或启动脚本，然后设置开机执行这个脚本  
  
但用了macvlan模式后，ping的反应明显比容器的bridge模式要慢，但看上去实际的网络操作好像没有太大影响，  
可以用下面命令检测网络反应时间作为对比  
#time curl -o -v github.com  
  
另外macvlan模式会导致一些应用不能正常使用，如一个文件传输程序croc会有问题，  
而容器的bridge、host、ipvlan模式都完全没有这个问题  
  
但macvlan比ipvlan有一个好处是每个容器外访都是一个独立的mac地址，  
而ipvlan外访都是使用宿主机的mac地址，这样也可能导致有些应用有问题  
  
对于macvlan模式下的容器ipv6地址不需要设置任何的ND proxy，ipv6互访性与普通局域网的ipv6机器一样，  
如果各位没有ipv6需求，那么只需要去掉上面流程中所有关于ipv6的配置及命令  
  

