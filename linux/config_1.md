#3.1 ssh正向反向代理
## 1. 代理的意思
- 网络连接都是CS结构，所谓代理，就是将CS不能直接访问的资源映射在C端或者S端  
- C->S 正向代理  
- S->C 反向代理  
## 1. 正向代理
- client local_client server remote_resource  
  client ssh 客户端  
  local_client 要访问远程资源的主机  
  server  ssh 服务器  
  remote_resource 要代理的资源  
- 连接  
  ssh -L client:bind_port:remote_resource:remote_resource_port user@server 



## 2. 反向代理
- local_resource client server remote_client
  local_resource 要映射到S端的服务器  	
  client ssh 客户端  
  server ssh 服务器
  remote_client S端客户端  
- 连接  
 ssh -R server:bind_port:local_resource:localport user@server  

## 3. 优缺点
- 速度比较慢
- 部署方便
- 后台运行需要额外支持


