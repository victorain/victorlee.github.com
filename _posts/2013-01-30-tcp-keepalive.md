---
layout: post                               
title: TCP 死连接检测 与 keepalive                
category: dev-design                       
p-indent: false                            
original: true                             
tags: tcp/ip keepalive crtmpserver
---

###什么是tcp死连接
这里的tcp死连接指的是，事实上连接已经不通了，但是服务器程序无法感知到，仍然
认为连接存在。当客户端正常断开tcp连接的时候，是不会出现死连接的，tcp协议保证
服务端能够感知到客户端连接的断开。那么什么时候会出现死连接呢？在一些异常情况下，
连接已经断开但是客户端没有机会发送断开连接的tcp包给服务器，这时候就发生了死连接。

这些异常情况包括： 

- 网线松动(可以通过拔掉客户端网线来制造这种场景) 
- 客户端主机突然掉电
- 客户端主机崩溃(注意是突然崩溃，不是正常关机)
- 中间路由器崩溃
- ...... 等等

###tcp死连接的危害

对于服务器程序来说，来自于客户端的tcp连接是一种资源，尤其是长连接。不处理死连接意味着浪费资源。

更加严重的情况是死连接导致业务逻辑上的问题。前段时间我在工作中遇到了这种情况：搞现场直播时，
将直播流推到rtmpserver上，然后用户去rtmpserver上观看直播流。rtmpserver对推流有个限制：
对于相同的流名称，只能有一个推流连接。比如cctv5，如果已经有个客户端推了cctv5到rtmpserver，
那么再有另外一个客户端向rtmpserver推cctv5这个流的时候会被rtmpserver拒绝掉。在这个场景下
如果发生了死连接，我们将无法重新向rtmpserver推流，因为rtmpserver认为之前已经有一个cctv5连
接存在了，于是不允许推流。这就导致再也无法将这个流推到rtmpserver上。
我们看到，有些场景必须要检测并释放tcp死连接。下面谈谈如何处理死连接。                  
                                                                    

###如何处理tcp死连接
处理思路大概有这么几个：  

- 1.应用程序在业务上检测这种情况        
- 2.应用程序socket层面设置keepalive参数  
- 3.操作系统层面调整keepalive参数  

个人比较倾向于思路1，应用程序在业务上检测tcp死连接。我也是用这种思路解决了rtmpserver 
遇到的推流死连接的情况。我对rtmpserver做了如下修改，30s内没有推任何数据的连接被视为死连接，
这种连接将被释放掉。修改后，如果发生了推流死链接，那么最多30s后就可以重新推流了。当然，
这里的30s是一个配置项。

比较下思路2和思路3，都是设置keepalive参数，不同的是设置的层面不同。系统层设置会影响到其他进程，
从这个角度讲，应该是socket层面的设置优于系统层设置，这可以把影响限制在本应用程序进程内。

考考虑另外一个场景：问题已经发生并且在线上环境中。这时候要求最快速解决问题，你没时间改代码，
修改系统参数的效果是立竿见影的。于是，这里对linux系统的 tcp keepalive 参数做个介绍。

###linux tcp keepalive 简介
tcp keepalive 的原理是指定时间内连接上没有数据通信时，就向对端发送探测包，这个探测包不包含任何数据。
当连续发送指定个探测包都没有回应时，就认为连接已经断了。

不过，keepalive 并不是 TCP 规范的 一部分。原因大概有下面这些:    
                      
- 在网络闪断的情况下，这可能会使一个非常好的连接释放掉
- 它们耗费不必要的带宽                  
- 在按分组计费的情况下会在互联网上花掉更多的钱

然而，在许多的实现中提供了存活定时器。   

####tcp keepalive 相关的参数及其含义
tcp_keepalive_time  
连接上长达 tcp_keepalive_time 秒没有数据时，开始向对端发送第一个探测包                                                                                      
默认值:7200s                

tcp_keepalive_intvl            
在keepalive探测包开始后,每隔 tcp_keepalive_intvl 秒发送一次探测包
默认值：75s                                                                           
                                                                            
tcp_keepalive_probes                                                        
连续 tcp_keepalive_probes 个探测包没有回应时，认为是死连接                                   
默认值:9                                                                       
                                                                                                                                     
按照上述默认设置来看,服务端检测到死连接需要7200s+8*75s=7800秒  

####查看和更改 tcp keepalive 的方法
系统层面设置tcp keepalive有两种方法，直接修改/proc/文件系统下面的设置和使用系统提供的sysctl接口。

#####proc方法：
  查看：
  {% highlight c++ %}
  # cat /proc/sys/net/ipv4/tcp_keepalive_time
  7200
  # cat /proc/sys/net/ipv4/tcp_keepalive_intvl 
  75
  # cat /proc/sys/net/ipv4/tcp_keepalive_probes
  9    
  {% endhighlight %}     
  修改：   
  {% highlight c++ %}
  # echo 600 > /proc/sys/net/ipv4/tcp_keepalive_time
  # echo 60 > /proc/sys/net/ipv4/tcp_keepalive_intvl
  # echo 20 > /proc/sys/net/ipv4/tcp_keepalive_probes
  {% endhighlight %}
    
#####sysctl方法：
  查看：
  {% highlight c++ %} 
  # sysctl \                          
  > net.ipv4.tcp_keepalive_time \   
  > net.ipv4.tcp_keepalive_intvl \  
  > net.ipv4.tcp_keepalive_probes   
  net.ipv4.tcp_keepalive_time = 7200
  net.ipv4.tcp_keepalive_intvl = 75 
  net.ipv4.tcp_keepalive_probes = 9 
  {% endhighlight %}
    
  修改：
  {% highlight c++ %} 
  # sysctl -w \                        
  > net.ipv4.tcp_keepalive_time=600 \
  > net.ipv4.tcp_keepalive_intvl=60 \
  > net.ipv4.tcp_keepalive_probes=20 
  net.ipv4.tcp_keepalive_time = 600  
  net.ipv4.tcp_keepalive_intvl = 60  
  net.ipv4.tcp_keepalive_probes = 20 
  {% endhighlight %}