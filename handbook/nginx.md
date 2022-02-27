# Nginx相关问题及常用命令

## 1.相关问题

1. 创建环境变量  

   方法一：

   ```text
   创建一个软连接
   ln -s /usr/local/nginx/sbin/nginx /usr/local/bin/
   
   /usr/local/bin/就是环境变量目录
   ```

   方法二：

   1、编辑/etc/profile

   ```bash
   vim /etc/profile
   ```
	2、在最后一行添加配置，:wq保存

	```ruby
	PATH=$PATH:/usr/local/nginx/sbin
	export PATH
	```

	3、使配置立即生效
	
	```text
	source /etc/profile
	```

## 2.常用命令

**1.启动nginx**

```javascript
service nginx start
```

**2.停止nginx**

```javascript
nginx -s stop
```

**3.查看nginx进程**

```javascript
ps -ef | grep nginx 
```

**4.平滑启动nginx**

```javascript
nginx -s reload
```

平滑启动的意思是在不停止nginx的情况下，重启nginx，重新加载配置文件，启动新的工作线程，完美停止旧的工作线程。

**5.强制停止nginx**

```javascript
pkill -9 nginx
```

**6.检查对nginx.conf文件的修改是否正确**

```javascript
nginx -t -c /etc/nginx/nginx.conf
or 
nginx -t
```

**7.查看nginx的版本**

```javascript
nginx -v
or
nginx -V
```