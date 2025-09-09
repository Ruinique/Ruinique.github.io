---
title: 基于 Self-hosted LiveSync 和 Nginx Proxy Manager 实现的 Obsidian 笔记同步服务
---

# {{ $frontmatter.title }}

## **前言**

Obsidian 作为一款功能强大的双链笔记软件，其开放的插件生态和本地存储的特性，受到了许多用户的青睐。然而，官方的同步服务需要付费订阅。对于拥有自己的服务器，并且热爱折腾的玩家来说，自建同步服务无疑是一个更具性价比和可玩性的选择。

---

本文将手把手教你如何利用 Docker、CouchDB 和 Nginx Proxy Manager，搭建一套属于自己的、安全可靠的 Obsidian 跨平台同步服务。

---

这里的 Nginx Proxy Manager 更进一步的配置文档在 [[在 ECS 上优雅地管理你的反向代理：Nginx Proxy Manager 实战教程]] 。

---

## **准备工作**

在开始之前，请确保你拥有以下几样东西：

- 一台拥有公网 IP 的服务器（VPS）
- 一个属于你自己的域名
- 在服务器上安装好 Docker 和 Docker Compose

---

### **第一步：搭建 CouchDB 数据库**

Self-hosted LiveSync 插件使用 CouchDB 作为其后端数据库。我们将使用 Docker 来快速部署 CouchDB。

---

#### **创建 `docker-compose.yml` 文件**

在你的服务器上，创建一个新的文件夹，例如 `obsidian-sync`，并在其中创建一个名为 `docker-compose.yml` 的文件。    
```shell
mkdir obsidian-sync
cd obsidian-sync
nano docker-compose.yml
```

---
#### **编辑 `docker-compose.yml` 文件**

 将以下内容复制并粘贴到 `docker-compose.yml` 文件中。请务必修改 `COUCHDB_USER` 和 `COUCHDB_PASSWORD` 的值，设置一个安全的用户名和密码。
 
``` yaml
version: '3'
services:
  couchdb:
    image: couchdb:latest
    container_name: couchdb-for-ols
    restart: unless-stopped
    environment:
      - COUCHDB_USER=你的用户名  # 修改这里
      - COUCHDB_PASSWORD=你的密码 # 修改这里
    volumes:
      - ./couchdb-data:/opt/couchdb/data
      - ./couchdb-etc:/opt/couchdb/etc/local.d
    ports:
      - "5984:5984"
```

#### **启动 CouchDB 容器**

在 `docker-compose.yml` 文件所在的目录中，运行以下命令来启动 CouchDB 容器：

```shell
docker-compose up -d
```

现在，你的 CouchDB 数据库已经在后台运行，并监听着服务器的 5984 端口。

### **第二步：使用 Nginx Proxy Manager 设置反向代理**

为了安全起见，我们不希望将 CouchDB 的端口直接暴露在公网上。我们将使用 Nginx Proxy Manager 来设置一个反向代理，并通过域名来访问我们的同步服务，同时为它加上 HTTPS。

#### **创建 Nginx Proxy Manager 的 `docker-compose.yml` 文件**

在另一个文件夹中（例如 `npm`），创建对应的 `docker-compose.yml` 文件。

```shell
 mkdir npm
 cd npm
 nano docker-compose.yml
```

以下内容粘贴进去：

```yaml
version: '3'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```
#### **启动 Nginx Proxy Manager**

``` shell
docker-compose up -d
``` 

#### **配置 Nginx Proxy Manager**

- 通过浏览器访问 `http://你的服务器IP:81`，进入 NPM 的管理界面。
- 默认的管理员用户和密码如下，首次登录后会要求你修改：
    - **Email:** `admin@example.com`
    - **Password:** `changeme`
- 登录后，点击 "Proxy Hosts"，然后点击 "Add Proxy Host"。
- 在 "Details" 标签页中，进行如下配置：
    - **Domain Names:** 输入你准备好的域名，例如 `obsidian.yourdomain.com`。
    - **Scheme:** `http`
    - **Forward Hostname / IP:** 你的服务器的内网或公网 IP 地址。
    - **Forward Port:** `5984` (CouchDB 的端口)
    - **Block Common Exploits:** 建议勾选。
- 切换到 "SSL" 标签页：
    - 在 "SSL Certificate" 下拉菜单中，选择 "Request a new SSL Certificate"。
    - 配置 DNS Challenge
    - 勾选 "Force SSL" 和 "HTTP/2 Support"。
    - 同意 Let's Encrypt 的服务条款。
    - 点击 "Save"。

现在，你应该可以通过你设置的域名 (`https://obsidian.yourdomain.com`) 来访问你的 CouchDB 数据库了。


### **第三步：配置 Obsidian LiveSync 插件**

万事俱备，只欠东风。现在我们回到 Obsidian，配置 LiveSync 插件。

#### **安装插件**

在 Obsidian 中，进入 "设置" > "第三方插件"，关闭安全模式。然后点击 "社区插件" 旁的 "浏览"，搜索 "LiveSync" 并安装。    
#### **配置插件**
  
  安装并启用 LiveSync 插件后，打开插件的设置页面。
  
  - 在 "Sync Method" 中选择 "Self-hosted LiveSync"。
  - 在 "Remote Database configuration" 中：
	  - **URL:** 填写你刚刚设置好的域名，例如 `https://obsidian.yourdomain.com`。
      - **Username:** 填写你在第一步中为 CouchDB 设置的用户名。
      - **Password:** 填写你设置的密码。  
  - 点击 "Test" 按钮，如果一切顺利，你应该会看到连接成功的提示。  

#### **开始同步**
    
配置完成后，你可以选择立即同步。在 LiveSync 的状态栏图标上点击，可以选择同步模式，例如 "LiveSync" 模式可以实现近乎实时的同步。    

## **结语**

恭喜你！你已经成功搭建了一套完全由自己掌控的 Obsidian 同步服务。现在，你可以在你的所有设备上（Windows, macOS, Linux, iOS, Android）安装 Obsidian 和 LiveSync 插件，并使用相同的配置来进行同步，享受无缝衔接的笔记体验。

自建服务虽然需要一些前期投入，但它带来的数据安全感和自由度是无可替代的。希望这篇笔记能对你有所帮助！