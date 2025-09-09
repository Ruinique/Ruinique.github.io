---
title: 在 ECS 上优雅地管理你的反向代理：Nginx Proxy Manager 实战教程
---

# {{ $frontmatter.title }}

## **前言**

当你在自己的云服务器上托管的服务越来越多时（例如个人博客、NAS 控制台、笔记同步服务等），一个棘手的问题便会浮现：如何让这些服务都通过易于记忆的域名访问，并且全部启用 HTTPS 加密？传统的做法是手动配置 Nginx 或 Caddy 的命令行配置文件，这不仅繁琐，而且容易出错，对新手尤其不友好。

今天，我们将介绍一款神器——Nginx Proxy Manager (NPM)。它是一个基于网页图形界面（GUI）的反向代理管理工具，能让你告别复杂的命令行，像操作桌面软件一样轻松地设置域名、配置反向代理并自动申请和续签 Let's Encrypt 的免费 SSL 证书。

本文将作为一份详尽的实战指南，手把手教你如何在阿里云 ECS 上部署 NPM，并深度结合阿里云的 DNS 服务，实现“一键化”的 SSL 证书全自动管理。

## **准备工作**

在开始之前，请确保你拥有以下“装备”：

- **一台阿里云 ECS 实例**：并已在其中安装好 Docker 和 Docker Compose。
- **一个由阿里云管理的域名**：无论是在阿里云（万网）注册，还是将 DNS 解析托管在阿里云。
- **一个阿里云 AccessKey**：这是实现 DNS 自动化验证的关键，拥有管理云解析（DNS）的权限。

### **第一步：配置阿里云环境**

工欲善其事，必先利其器。首先，我们需要为 NPM 的顺畅运行配置好云端环境。

#### **配置 ECS 安全组**    

安全组是云服务器的“防火墙”，我们需要为 NPM 和它所代理的服务打开必要的端口。
- 登录阿里云 ECS 管理控制台，进入“网络与安全” > “安全组”。    
- 找到你的 ECS 实例绑定的安全组，点击“配置规则”。    
- 在“入方向”上，手动添加以下三条规则（授权对象均设为 `0.0.0.0/0`）：
- `HTTPS (443)`: 所有经过加密的网站服务的最终访问端口。     
- `HTTP (80)`: 用于 Let's Encrypt 进行 HTTP 验证以及将 HTTP 请求重定向到 HTTPS。            
- `自定义 TCP (81)`: Nginx Proxy Manager 管理后台的专用访问端口。         

#### **创建并获取 AccessKey**

为了让 NPM 能自动在你的域名下添加 DNS 记录来证明你对域名的所有权（DNS-01 质询），从而自动化申请证书，我们需要创建一个拥有 DNS 操作权限的 AccessKey。

- 登录阿里云 RAM 访问控制台。 
- 在“身份管理” > “用户”中创建一个新用户（例如 `npm-manager`）。 
- 创建时，请勾选“为该用户自动生成 AccessKey”，并在创建完成后，**立即保存好 AccessKey ID 和 AccessKey Secret**。这个密钥对只会显示一次。 
- 回到用户列表，为刚才创建的用户“添加权限”，搜索并授予 `AliyunDNSFullAccess`（管理云解析（DNS）的权限）。 
### **第二步：一键部署 Nginx Proxy Manager**

得益于 Docker，部署 NPM 的过程简单到极致。

SSH 登录到你的 ECS 服务器，创建一个专属的工作目录。

```shell
mkdir npm && cd npm
```
 
 在该目录中，创建一个 `docker-compose.yml` 文件。

```bash
nano docker-compose.yml
```

 将以下内容复制进去。这是官方推荐的最小化配置，已经包含了数据持久化，确保你的配置在容器重启后不会丢失。

```yaml
version: '3'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # 公开的网络端口
      - '80:80'   # HTTP
      - '443:443' # HTTPS
      # 管理后台端口
      - '81:81'
    volumes:
      # 将配置和证书持久化到当前目录下的 data 和 letsencrypt 文件夹
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
  ```
  
 在 `docker-compose.yml` 文件所在目录，运行以下命令启动服务：
 
 ```shell
 docker-compose up -d
 ```

 **首次登录**

- 稍等片刻，通过浏览器访问 `http://你的ECS公网IP:81`。
    - 使用默认管理员账号登录：
	    - **Email:** `admin@example.com`
        - **Password:** `changeme`    
    - 首次登录后，系统会引导你修改邮箱和密码，请务必修改为你的常用邮箱和强密码。

### **第三步：核心实战：添加反向代理与自动化 SSL**

现在，让我们通过一个实际案例来体验 NPM 的强大之处。假设我们有一个内部服务（比如一个 Portainer 容器）运行在服务器的 `9000` 端口，我们希望通过 `portainer.yourdomain.com` 这个域名以 HTTPS 方式访问它。

**配置阿里云 DNS 提供商（一次性设置）**

- 在 NPM 管理界面，点击顶部 "SSL Certificates"，然后点击 "Add SSL Certificate"，选择 "Let's Encrypt"。    
- 这一步我们不是真的要申请证书，而是为了预先配置好 DNS 提供商。      
- 勾选 **"Use a DNS Challenge"**。      
- 在 `DNS Provider` 下拉菜单中，找到并选择 **`Alibaba Cloud`**。      
- 在 `Credentials File Content` 输入框中，按照以下格式填入你第一步中获取的 AccessKey 信息：
```json
{
"dns_aliyun_access_key_id": "你的AccessKey ID",
"dns_aliyun_access_key_secret": "你的AccessKey Secret"
}
```

- 点击 "Save" 保存。NPM 现在已经知道如何与你的阿里云账户“对话”了。你可以直接关闭这个添加证书的弹窗。

 **创建反向代理主机**
 
- 回到主界面，点击 "Hosts" > "Proxy Hosts"，然后点击 "Add Proxy Host"。
- **在 "Details" 标签页中，填写路由信息：**
    - **Domain Names**: 输入你的完整域名，例如 `portainer.yourdomain.com`。
    - **Scheme**: `http`（因为 NPM 和目标容器通常在同一台服务器，内网无需加密）。
    - **Forward Hostname / IP**: 填写你的 **ECS 内网 IP 地址**（推荐，更安全高效）或 `127.0.0.1`。
    - **Forward Port**: `9000`（你的内部服务实际监听的端口）。
- **切换到 "SSL" 标签页，配置证书：**
    - 在 `SSL Certificate` 下拉菜单中，选择 **"Request a new SSL Certificate"**。
    - **勾选 "Force SSL"**，强制所有访问都使用 HTTPS。
    - **勾选 "HTTP/2 Support"**，开启性能更强的协议。
    - 勾选同意 Let's Encrypt 服务条款。
- **点击 "Save"**。

此时，奇迹发生了！NPM 会自动通过你配置好的阿里云 AccessKey，在你的域名下创建一条临时的 TXT 解析记录，然后向 Let's Encrypt 证明你对该域名的所有权。验证通过后，证书将自动签发并配置好。整个过程全自动，通常在 30 秒内完成。
    
大功告成！现在，你可以直接在浏览器中输入 `https://portainer.yourdomain.com` 来访问你的服务了，一把安全的小绿锁已经出现在了地址栏。

#### **结语**

你已经成功地在阿里云上部署了 Nginx Proxy Manager，并掌握了其最核心、最强大的自动化 HTTPS 功能。从此以后，无论你需要在服务器上部署多少个网站或服务，都可以通过这个优雅的图形界面，在几分钟内为它们分配独立的域名并配置好加密访问，无需再触碰任何复杂的配置文件。

NPM 不仅是一个反向代理工具，它更像是一个网站管家，将你从繁琐的运维工作中解放出来，让你能更专注于创造和分享。去探索它的更多功能（如访问列表、重定向等），尽情享受云端折腾的乐趣吧！