# 基于主机名搭建NGINX 虚拟主机

## 配置nginx-yum源

```
vim /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1

```

# 容器安装nginx

```
docker run -p 80:80 --name nginx -v /root/nginx/www:/www -v /root/nginx/conf.d/nginx.conf:/etc/nginx/nginx.conf -v /root/nginx/logs:/wwwlogs  -d nginx  
```

# 反向代理和负载均衡（Server Load Balancer）

### 一、SLB产生背景:

SLB(服务器负载均衡)：在多个提供相同服务的服务器的情况下，负载均衡设备存在**虚拟服务IP地址**。当大量客户端从外部访问虚拟服务IP地址时，负载均衡设备将这些报文请求根据负载均衡算法，将流量均衡的分配给后台服务器以平衡各个服务器的负载压力，避免在还有服务器压力较小情况下其他服务达到性能临界点出现运行缓慢甚至宕机情况，从而提高服务效率和质量。因此对客户端而言，RS（real server 实际服务器）的IP地址即是负载均衡设备VIP（虚拟服务地址IP）地址，真正的RS服务器IP地址对于客户端是不可见的。

### 二、SLB的三种传输模式：

四层SLB：配置负载均衡设备上服务类型为tcp/udp，负载均衡设备将只解析到4层，负载均衡设备与client三次握手之后就会和RS建立连接；

七层SLB：配置负载均衡设备服务类型为http/ftp/https等，负载均衡设备将解析报文到7层，在负载均衡设备与client三次握手之后，只有收到对应七层报文，才会跟RS建立连接。

在负载均衡设备中，SLB主要工作在以下的三种传输模式中：

- **反向代理模式**
- **透传模式**
- **三角模式**

（根据不同的模式，负载均衡设备的工作方式也不尽相同，但无论哪种模式，客户端发起的请求报文总是需要先到达负载均衡设备进行处理，这是LB正常工作的前提。）

模拟网络拓扑环境：

Client：10.8.21.40

负载均衡设备：172.16.75.83

VIP：172.16.75.84

RS1IP：172.16.75.82

RS2IP：172.16.75.85

在整个报文交互过程中，采用Tcpdump和Wireshark分别在RS和Client处抓包，然后使用Wireshark进行报文解析。

### 三、 反向代理模式：

**反向代理**：普通的代理设备是内网用户通过代理设备出外网进行访问，而工作在这种模式下的负载均衡设备，则是外网用户通过代理设备访问内网，因此称之为反向代理。

在反向代理模式下，当负载均衡设备收到客户端请求后，会记录下此报文（ 源IP地址、目的IP地址、协议号、源端口、目的端口，服务类型以及接口索引），将报文（目的地址更改为优选后的RS设备的IP地址，目的端口号不变；源地址修改为负载均衡设备下行与对应RS设备接口的IP地址，源端口号随机）发送给RS；

当RS收到报文后，会以（RS接口IP地址为源，负载均衡设备地址为目的）回复报文；负载均衡设备将源修改为VIP，目的端口号修改为客户端的源端口号，目的IP修改为Client的源IP回复报文。

#### 查看报文解析结果：

配置完成后，Client访问RS服务器，返回成功，整个报文交互过程如下 ：

![img](https://pic2.zhimg.com/v2-c10a0ba07129662dd4ad7c8c8a696695_b.png) 

Client和负载均衡设备之间的报文交互过程 ↑

![img](https://pic1.zhimg.com/v2-3ea4b99cad578b6b3cfd29d1890a60e0_b.png) 

负载均衡设备和RS之间报文交互过程 ↑

#### 结果分析

分析整个报文交互过程：

TCP握手过程：

首先Client向负载均衡设备发送TCP SYN报文请求建立连接，源IP为Client的IP 10.8.21.40，源端口号50894，目的IP为VIP地址172.16.75.84，目的端口号80；

收到请求报文后，负载均衡设备会以源IP为VIP地址172.16.75.84，端口号80，目的IP 10.8.21.40，目的端口号50894回应SYN ACK报文；

Client收到报文后回复ACK报文，TCP三次握手成功。

HTTP报文交互过程：

当负载均衡设备与client完成三次握手后，因为配置的七层SLB，如果收到HTTP请求，就会根据负载均衡算法和服务器健康状态优选出对应的RS（在这次过程中选择的RS设备为172.16.75.82），然后与RS建立TCP连接：

负载均衡设备发送TCP SYN报文请求连接，源IP为负载均衡设备与RS相连接口IP 172.16.75.83，源端口号随机4574，目的IP为RS的IP 172.16.75.82，目的端口号80；

RS收到报文后，以源IP 172.16.75.82，端口号80，目的IP 172.16.75.83，目的端口号4574回复SYN ACK报文，负载均衡设备回复ACK报文建立三次握手；

之后，负载均衡设备再将收到的HTTP报文源IP修改为与RS相连下行接口IP地址172.16.75.83，源端口号为随机端口号，将报文发送给RS；

当RS收到报文后，使用源为本地IP 172.16.75.82，目的IP为172.16.75.83进行回复，所以报文直接回复给负载均衡设备；

当负载均衡设备收到RS的回应报文后，将报文的源修改为VIP地址172.16.75.84，目的IP为10.8.21.40发送回Client，再将目的端口号修改为HTTP请求报文中的源端口号，服务器访问成功。

![img](https://pic2.zhimg.com/v2-0ddbc3871dac8bce55d0dd26a6e5f1bd_b.png)  

由上述的过程可以看出，在RS端上，client的真实IP地址被负载设备修改成与RS相连接口的IP地址，所以RS无法记录到Client的访问记录，为了解决这个问题，可以采用在HTTP报文头中添加X-Forwarded-For字段，本文不做赘述，可以自行查询。

### 四、透传模式：

当负载均衡设备工作在透传模式中时，RS无法感知到负载均衡设备的存在，对于Client来说，RS的IP地址就是负载均衡设备的VIP地址。

在这种模式下，当负载均衡设备收到源为Client的IP，目的IP为本地VIP地址的报文时，会将报文根据负载均衡策略和健康状况发送给最优的RS设备上，继而RS设备会收到目的为本地IP，源为Client实际IP的请求报文；然后RS将会直接回应此请求，报文的目的IP地址为Client的IP地址，当负载均衡设备收到此报文后，将源IP地址修改为VIP地址，然后将报文发送给Client。

#### 报文解析结果：

同样在RS端和Client端抓取交互报文：

 ![img](https://pic3.zhimg.com/v2-35504893ad1c1c5e31cde46777b52866_b.png) 

Client和负载均衡设备之间的报文交互过程 ↑

 ![img](https://pic1.zhimg.com/v2-950fe59a2215c3aa311af25e2c16ad38_b.png) 

负载均衡设备和RS之间的报文交互过程 ↑

#### 结果分析：

TCP握手过程：

同反向代理模式交互过程

HTTP报文交互过程：

Client向负载均衡设备的VIP地址172.16.75.84以源IP 10.8.21.40发送HTTP请求，当负载均衡设备收到报文后，与优选后的RS进行TCP三次握手，过程同反向代理模式，然后将收到的HTTP报文，不改变报文的源IP地址和源/目的端口号，只修改目的IP修改为优选后的RS地址172.16.75.82；

当RS收到源来自IP 10.8.21.40的报文后，回复报文给IP地址10.8.21.40，此时要注意，必须在RS上配置回复报文经过负载均衡设备，负载均衡设备会将源IP修改为VIP地址172.16.75.84，然后转发给Client，否则Client将会收到源IP为172.16.75.82的HTTP报文，服务器访问失败。![img](https://pic1.zhimg.com/v2-919aed35b9aa3e41f1a17044aaf52344_b.png) 

### 五、 三角模式：

在三角模式下，当客户端发送请求到负载设备上时，负载均衡设备会计算出最优RS，然后直接**根据MAC地址**将报文转发给RS，在RS上配置报文的源IP为VIP地址（一般配置在loopback口上），因此在这种情况下，RS会直接将报文发送给Client，即使回复报文经过负载均衡设备，此设备不做任何处理。由于报文在整个过程中传输途径类似于三角形，因此称之为三角模式。

#### 报文解析结果：

分别在Client端和RS端抓包，内容如下：

![img](https://pic3.zhimg.com/v2-f14dda14a2584b15e6feb82d9d30bea6_b.png) 

Client和负载均衡设备之间的报文交互过程 ↑

 ![img](https://pic4.zhimg.com/v2-667f8cb715128027cfc7d41a4bd80ba7_b.png) 

RS 和负载均衡设备之间的报文交互过程 ↑

#### 结果分析：

TCP握手过程：

由于采用了4层SLB，所以在TCP握手过程中与上述的7层SLB有些不同，当Client和RS完成三次握手之后，此时负载均衡设备会直接选择RS，然后跟RS建立TCP三次握手；

在三角模式环境中，由于RS的Loopback口和负载均衡设备上都存在着VIP地址172.16.75.84，当负载均衡设备经过负载均衡算法选择出对应的RS后，会根据实际配置的RS的IP地址对应的mac地址，将报文以目的mac为RS，目的IP为VIP的方式建立TCP连接。

HTTP报文交互过程：

首先Client向负载均衡设备的VIP发送HTTP请求，源为10.8.21.40，当负载均衡设备收到报文后，将报文直接转发给RS，当RS收到源IP为10.8.21.40，目的IP为本地Loopback口IP地址172.16.75.84的报文后，直接将报文回复给10.8.21.40，同样源为IP地址172.16.75.84，由此访问服务器成功。

 ![img](https://pic4.zhimg.com/v2-ba2a56bdecbd0844a760a0486d54a56f_b.png) 

在三角模式中，由于回复报文负载均衡设备不做任何处理，所以非常适合于RS到Client方向流量较大或者连接数目较多的组网环境。采用三角模式时，必须注意RS有路由可以到达Client，并且在RS的Loopback接口上必须有负载均衡设备的VIP地址，否则即使RS设备收到Client的请求报文也会直接丢弃报文，不作回应。

### 六、总结

由于反向代理模式中在RS侧只能收到源为负载均衡设备IP的报文，因此可以使用防火墙增加安全性，只允许源IP为负载均衡设备的IP地址的报文通过，同时增加X-Forwarded-For字段也可以让RS只允许有此字段的报文进行访问，因此安全性相对较高。

![img](https://pic3.zhimg.com/v2-cbfce85101a299ea4e86a89b238efd56_b.png) 

nginx容器代理其他容器conf配置文件

```
server {
    listen 9000;
   server_name api.testcloud.com;

    access_log /var/log/nginx/api-dev.log;
    error_log /var/log/nginx/api-dev.error.log;
#    location / {
#      root /www/es65;
#      index index.html index.htm;
#      proxy_set_header    Host $host;
#      proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
#      proxy_ignore_client_abort on;
#      proxy_connect_timeout   1200s;
#      proxy_send_timeout  1200s;
#      proxy_read_timeout  1200s;
#      expires 1m;
#}

    location / {
        proxy_set_header    Host $host;
        proxy_redirect off;
        proxy_set_header    X-Real-IP $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        if ($request_method !~* GET|POST) {
            return 403;
        }

        proxy_ignore_client_abort on;
        proxy_connect_timeout   1200s;
        proxy_send_timeout  1200s;
        proxy_read_timeout  1200s;
        proxy_pass  http://172.16.0.32:9208/;
    }
}

```

​           if ($request_method !~* GET|POST) {        return 403;    }

​         上面这句：非get  非post的请求，全部转到430

C:\Windows\System32\drivers\etc     #windows解析文件地址
