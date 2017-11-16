---
title: WiFi环境下使用Cellular发送数据
tags: 工作
date: 2017-06-19 13:34:01
categories: 工作
---



# WiFi环境下使用Cellular发送数据 
产品需求，配合一些操作可以做到用户身份的确认。

<!-- more -->          
平常开发习惯的直接调用`NSURLConnection`或者`NSURLSession`这样的API进行网络请求，或者干脆引用`AFNetworking`这样的第三方网络框架。除非在特定的，类似下载这种场景中，才回去关心用户是处在哪一种的网络环境下。这种环境更确切的说是：是否是WiFi？（公司最开始也有类似socks代理的场景）   
如何在已经连接到WiFi的情况下，通过移动网络发送送请求？    
连接WiFi中可能会出现：    
 	--  WiFi是可以连接到网络       
 	--  WiFi不可以连接到网络(可能会影响域名解析)    
步骤如下：

## 找到cellular      
代码如下：

```
// retrieve the current interfaces - returns 0 on success
    struct ifaddrs *interfaces;
    if(!getifaddrs(&interfaces)) {
        // Loop through linked list of interfaces
        struct ifaddrs *interface;
        for(interface=interfaces; interface; interface=interface->ifa_next) {
            //make sure the interface is up
            if(!(interface->ifa_flags & IFF_UP) /* || (interface->ifa_flags & IFF_LOOPBACK) */ ) {
                continue; // deeply nested code harder to read
            }
            const struct sockaddr_in *addr = (const struct sockaddr_in*)interface->ifa_addr;
            char addrBuf[ MAX(INET_ADDRSTRLEN, INET6_ADDRSTRLEN) ];
            if(addr && (addr->sin_family==AF_INET || addr->sin_family==AF_INET6)) {
                NSString *name = [NSString stringWithUTF8String:interface->ifa_name];
                NSString *type;
                if(addr->sin_family == AF_INET) {
                    if(inet_ntop(AF_INET, &addr->sin_addr, addrBuf, INET_ADDRSTRLEN)) {
                        type = IP_ADDR_IPv4;
                    }
                } else {
                    const struct sockaddr_in6 *addr6 = (const struct sockaddr_in6*)interface->ifa_addr;
                    if(inet_ntop(AF_INET6, &addr6->sin6_addr, addrBuf, INET6_ADDRSTRLEN)) {
                        type = IP_ADDR_IPv6;
                    }
                }
                if(type) {
                    NSString *key = [NSString stringWithFormat:@"%@/%@", name, type];
                    addresses[key] = [NSString stringWithUTF8String:addrBuf];
                }
            }
        }
        // Free memory
        freeifaddrs(interfaces);
    }
```

## 建立socket&bind          
代码如下：   

```
struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));
addr.sin_len         = sizeof(addr);
addr.sin_family      = AF_INET;
addr.sin_port        = htons(0);
addr.sin_addr.s_addr = inet_addr([cellular_address UTF8String]);
NSData *data = [NSData dataWithBytes:&addr length:sizeof(addr)];
// socket
int _socket = socket(AF_INET, SOCK_STREAM, 0);
int status = bind(_socket,(const struct sockaddr *)[data bytes], (socklen_t)data.length);

```

## 域名解析

当前WiFi要联网

```
struct hostent * remoteHostEnt = gethostbyname([domain UTF8String]);
if (NULL == remoteHostEnt) {
        close(_socket);
        NSLog(@"%@",@"无法解析服务器的主机名");
        return;
}
struct in_addr * remoteInAddr = (struct in_addr *)remoteHostEnt->h_addr_list[0];
```

## sendbuffer格式

HTTP头结构
{% asset_img http_head_formate.png HTTP头 %}

### GET
如下：    

```
GET /path?query_string HTTP/1.1\r\n
...
Connection: keep-alive
\r\n
```

### POST
如下：  

```
POST /path HTTP/1.1\r\n
Content-Type: text/plain\r\n
Content-Length: 12\r\n
...
Connection: keep-alive
\r\n
query_string
```


## 大概流程 
  流程   
  
- create a socket    
- bind local
- lookup the remote address     
- open the socket    
- send the request    
- wait for the response    
- close the socket   



