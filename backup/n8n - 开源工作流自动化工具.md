![image](https://github.com/user-attachments/assets/b49a9bab-0e62-4580-8d40-849c57590e5d)

## 介绍
**n8n是一个开源的工作流自动化工具，可以实现定时、手动、webhook、或其它平台的事件进行触发，执行一系列节点组成的工作流。**
n8n内置了200+平台的各种操作节点，但基本都是国外平台，如果想要对接的平台不存在，可以使用`HTTP Request`节点进行网络请求。
在处理数据的逻辑比较复杂的情况下，n8n也有`code`节点，可以使用js或者python直接编写代码。

## 使用docker-compose部署
### 一、简单部署
1. 创建n8n文件夹
   - `mkdir n8n`
2. 进入n8n文件夹
   - `cd n8n`
3. 创建`.n8n`文件夹
   - `mkdir .n8n`
4. 创建 docker-compose.yaml 文件
   ``` yaml
   version: '3'
   services:
       n8n:
           restart: unless-stopped
           volumes:
               - './.n8n/:/home/node/.n8n'
           ports:
               - '5678:5678'
           environment:
               - GENERIC_TIMEZONE=Asia/Shanghai
               - TZ=Asia/Shanghai
               - N8N_SECURE_COOKIE=false # 是否仅在HTTPS下可用，默认为true；https://docs.n8n.io/hosting/configuration/environment-variables/security/
           container_name: n8n
           image: 'n8nio/n8n:latest'
   ```
5. 启动容器
   - `docker-compose up -d`
6. 通过[http://localhost:5678](http://localhost:5678)访问n8n

![image](https://github.com/user-attachments/assets/c8e52d76-af2b-440c-9d36-6db30496b5f9)

### 二、通过域名访问
**n8n可以配置为主域名、子域名、子目录访问**

例: subdomain/mydomain.com/path/
- 主域名: `mydomain.com`
- 子域名: `subdomain.mydomain.com`
- 子目录: `/path/`


修改`docker-compose.yaml`中的`environment`配置
```yaml
environment:
   - N8N_SECURE_COOKIE=false # 是否仅在HTTPS下可用，默认为true；https://docs.n8n.io/hosting/configuration/environment-variables/security/
   - N8N_PROTOCOL=http
   - N8N_HOST=你的域名
   - N8N_PATH=/你的二级路径/ # 默认值为/，注意这里如果想自定义，必须斜杠开头、斜杠结尾
```

#### <span style="color:red">注意</span>
<span style="color:red">1. 配置为主域名或者子域名，需要在你的dns上配置解析，并在本地配置好反向代理</span>

<span style="color:red">2. 配置为二级路径访问，需要在本地机器配置好反向代理</span>

#### 设置为二级路径访问后，nginx的配置示例
```
location ^~ /你的二级路径/ {
    proxy_pass http://127.0.0.1:5678/; 
    proxy_set_header Host $host; 
    proxy_set_header X-Real-IP $remote_addr; 
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
    proxy_set_header REMOTE-HOST $remote_addr; 
    proxy_set_header Upgrade $http_upgrade; 
    proxy_set_header Connection $http_connection; 
    proxy_set_header X-Forwarded-Proto $scheme; 
    proxy_http_version 1.1; 
    add_header X-Cache $upstream_cache_status; 
    add_header Cache-Control no-cache; 
    proxy_ssl_server_name off; 
}
```

### 三、使用postgresql存储数据
n8n默认使用sqlite存储数据，但是sqlite的速度较慢，而且如果保存的执行记录过多，删除数据之后，数据库文件默认并不会变小，[官方建议生产使用postgresql存储数据](https://community.n8n.io/t/n8n-creates-everygrowing-files-how-to-delete-unnecessary-data-via-cronjob/620)。当然针对这个问题，也可以设置`DB_SQLITE_VACUUM_ON_STARTUP`环境变量为`true`，每次启动n8n，都会执行`VACUUM`命令，清理数据库文件。

可以通过修改`docker-compose.yaml`中的`environment`配置使用postgresql存储数据。

[官方文档](https://docs.n8n.io/hosting/configuration/environment-variables/database/#postgresql)
```yaml
environment:
   - DB_TYPE=postgresdb # 数据库类型，目前支持 sqlite, postgresdb，默认为sqlite
   - DB_POSTGRESDB_DATABASE=库名 # 默认为n8n
   - DB_POSTGRESDB_HOST=数据库地址 # 默认为localhost
   - DB_POSTGRESDB_PORT=数据库端口 # 默认为5432
   - DB_POSTGRESDB_USER=数据库用户名 # 默认为postgres
   - DB_POSTGRESDB_SCHEMA=模式名 # 默认为public
   - DB_POSTGRESDB_PASSWORD=数据库用密码 # 默认为空
```