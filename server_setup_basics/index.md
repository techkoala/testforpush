# 服务器基础设置

> 翻译一篇服务器基本配置的好文章

> 这是我一直想写的一篇文章。虽然说明如何设置自托管应用程序很简单，但在薄弱的基础上托管应用程序毫无意义。在每篇教程的开头都介绍服务器设置是一件非常麻烦的事，所以我也为自己写了这篇文章，作为我如何为自己托管的应用程序设置服务器的参考。我会从基本的东西开始，比如使用 SSH 正确登录、非根用户设置以及为每个应用程序创建用户。我还会介绍 NGINX 设置、一些让服务器管理更轻松的生活质量工具、日志管理和基本网络安全。

## SSH

首先是登录。您需要一种安全访问设备的方法。不要使用用户名和密码。你需要使用 SSH（安全外壳），并确保 SSH 是唯一的登录方式。为此，你需要一个 SSH 密钥和一个新的用户账户。在新配置的 VPS 上，你将以根用户身份登录，你需要保护根用户账户。首先在 VPS 或远程机器上创建一个新的普通用户，并将其添加到 "sudo "组：

```shell
sudo adduser techkoala

sudo usermod -aG sudo techkoala
```

然后在本地电脑是执行：

```shell
ssh-keygen -t ed25519 -C "your_email@example.com"
```

按照说明操作，它会问你想把文件保存在哪里，以及是否需要密码。请确保设置了字符串密码。要将公钥复制到服务器上，请在本地计算机上运行：

```shell
ssh-copy-id -i ~/.ssh/id_ed25519.pub newuser@your_server_ip
```

请记住，newuser@your-server-ip 是用户名，也是你要将公钥复制到的远程设备。提示输入密码时，输入的是远程设备上的账户密码，而不是你刚刚为 SSH 密钥设置的密码。一旦验证通过，它就会复制公钥，现在你就可以通过 SSH 登录了。要关闭用户名和密码登录，请键入：

```shell
sudo vim /etc/ssh/sshd_config
```

找到下列值，并按你在这里看到的进行设置

```shell
Port 2222     # Change default port (use a number between 1024 and 65535)
PermitRootLogin no                 # Disable root login
PasswordAuthentication no          # Disable password authentication
PubkeyAuthentication yes           # Enable public key authentication
AuthorizedKeysFile .ssh/authorized_keys # Specify authorized_keys file location
AllowUsers newuser                 # Only allow specific users to login
```

这将禁止除 SSH 之外的所有登录方式，只能通过你复制了公钥的用户登录。停止以 Root 身份登录，只允许你指定的用户登录。点击 CTL+S 保存，点击 CTL+x 退出文件编辑器。重启 SSH：

```shell
sudo systemctl restart ssh
```

这可能会导致你退出 SSH 会话。如果出现这种情况，这时可以测试其他登录方法，看看是否被拒绝，然后再继续。此外，不言而喻，你需要妥善保管私钥，一旦丢失，你将无法再远程登录：

```shell
Protocol 2                 # Use only SSH protocol version 2
MaxAuthTries 3             # Limit authentication attempts
ClientAliveInterval 300    # Client alive interval in seconds
ClientAliveCountMax 2      # Maximum client alive count
```

现在，让我们再深入研究一下用户，看看如何利用他们来提高组织性和安全性。

## Users

在管理 Linux 服务器时，用户非常重要。服务器管理中有一个理念叫做 "最小权限原则"，其基本意思是，你要给予应用程序或进程完成工作所需的最小权限。Root 拥有无限的权限，没有应用程序真正需要它。为正在运行的应用程序设置用户权限有几个好处。如果正在运行的应用程序受到威胁，它可以限制潜在的损害。当运行多个应用程序时，它可以增加隔离性，有助于审计，让你知道哪些应用程序正在使用哪些系统资源。

简而言之，用户是组织系统的好帮手，在出现问题时能帮助你排除故障。要添加新用户，请运行

```shell
sudo useradd -rms /usr/sbin/nologin -c "a comment" youruser
```

该命令将创建一个用户，并为其提供一个存放应用程序数据的主目录，但不允许以该用户身份登录。-c标志是可选的，但最好能知道用户的用途，如 "运行 Nextcloud "之类。将应用程序文件克隆到 /opt 目录中：

```shell
sudo mkdir /opt/myapp
```

该命令会创建一个用户，并为其提供一个存放应用程序数据的主目录，但不允许以该用户身份登录。-c标志是可选的，但最好能知道用户的用途，如 "运行 Nextcloud "之类。将应用程序文件克隆到 /opt 目录中：

```shell
sudo chown appuser:appuser /opt/myapp
```

好了，这样就锁定了你的登录，你也应该对如何使用用户有了一定的了解。接下来是日志。

## **Logs**

日志对系统管理至关重要。它们可以跟踪系统健康状况，帮助排除故障和检测威胁。因此，你需要设置适当的日志轮换，这样日志就不会占用系统太多空间，而且更易于阅读和管理。要设置正确的日志轮换，需要编辑位于 /etc 目录下的 logrotate.conf 文件。单个应用程序的配置通常存储在 /etc/logrotate.d/，因此 NGINX 的配置示例如下：

```shell
/var/log/nginx/*.log {
    weekly
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

该配置每周轮换日志，保留 52 周的日志，压缩旧日志，以正确的权限创建新日志，然后在轮换后向 NGINX 发出信号以重新打开日志文件。您可以使用以下配置进行测试

```shell
sudo logrotate -d /etc/logrotate.conf
```

这将显示它在不实际旋转日志的情况下会做什么。有了这些设置，你就可以开始做更高级的事情了，比如根据日志条目触发警报。现在，这对单台服务器来说还算不错，但如果你管理的服务器不止一台，最好还是了解一下 Grafana Loki、Graylog 和 Fluentd 等工具。在此我就不详细介绍了，但如果你想提高你的日志游戏水平，这些都是不错的起点。

## **Backups**

备份，更重要的是测试备份，在服务器管理中极其重要。请记住：除非经过测试，否则备份就不是备份。未经测试的备份基本上是无用的。

备份主要有三种类型。完全备份、差异备份和增量备份。完全备份是磁盘上所有数据的完整副本。这种备份占用资源最多，但最容易恢复。差异备份备份的是上次完整备份后的所有变化，这是一种在空间和恢复速度上都处于中间位置的备份策略。增量备份备份自上次备份后发生变化的数据，这是最快的备份方式，但恢复起来可能最复杂。

我是这样想的。我使用增量备份来备份照片、文档或项目文件和经常编辑的文件夹。我会使用完全备份来备份整个服务器或磁盘。差异备份用于备份完整文件夹，如 /etc、/opt 和日志文件夹。

现在该如何存储呢？如果您遵循 3-2-1 原则，您就会如虎添翼。3 份数据副本、2 种存储类型和 1 个异地备份。我想说的是，如果这看起来太多，那么 "异地 "存储是最重要的，不能省略。万一发生灾难性的数据崩溃，拥有一个带有备份的硬盘是非常宝贵的。异地/离线备份还能让您免受勒索软件的威胁。因此，请牢记这一点。现在有大量的备份软件。我使用 Sync-thing、Borg 备份和老式 FTP 的组合。

请记住，备份、日志和服务器监控是一个根据您的需求不断发展的过程。您实施的具体策略应符合您的需求和数据的关键性。

## **Basic Network Safety**

保护服务器安全的下一步是锁定那些不需要暴露在互联网上的端口，并禁止那些在不应该登录的情况下尝试登录的东西。UFW 和 Fail2Ban 是目前广泛使用的两种工具。它们简单易用，UFW 可让你为端口设置流量规则，而 Fail2Ban 则会在 IP 地址进入不应进入的端口或在某些预定义规则后仍无法登录时将其封禁。UFW 或不复杂的防火墙通常会预装在许多 VPS 服务中，Fail2Ban 也是如此，但如果你使用的是新机器且不确定，请运行：

```shell
sudo apt install ufw

sudo apt install fail2ban
```

### **UFW**

关于 Fail2Ban，我们稍后再讨论，现在让我们重点讨论 UFW 设置。首先运行一些默认策略：

```shell
sudo ufw default deny incoming
 
sudo ufw allow outgoing
```

这被认为是最佳做法，因为它遵循了我前面提到的 "最少权限 "理念。它减少了机器的攻击面，让你可以精确控制暴露的内容。简而言之，这种配置在安全性和功能性之间取得了平衡。你的服务器可以根据需要接入互联网，但外部实体只能通过你明确允许的方式连接到你的服务器。现在，让我们允许一些东西进入。

```shell
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
```

如果要运行网络服务器，则需要打开 80 端口和 443 端口。80 端口用于 HTTP，443 端口用于 HTTPS。默认情况下，22 端口是 SSH，如果更改了端口，则需要指定端口，而不是使用 "allow ssh "命令。下面是其他一些有用的命令：

````shell
#List rules with numbers:
sudo ufw status numbered
#Delete by number:
sudo ufw delete NUMBER
#Delete by rule specification:
sudo ufw delete allow 80
#You can allow connections from specific IP addresses:
sudo ufw allow from 192.168.1.100
#You can also only allow an IP to connect to a specfic port with: 
sudo ufw allow from 192.168.1.100 to any port 22
#If you neeed to allow a range of ports: 
sudo ufw allow 6000:6007/tcp
#To further protect from brut force attacks you can rate limit specific ports with: 
sudo ufw limit ssh
#This would limit port 22 to 6 connections in 30 seconds from a single IP. To see the status of the firewall you can use: 

#Adding this goves you more info
sudo ufw status verbose
#and to reset incase you need to start over: 
sudo ufw reset
#and to enable and disable: 
sudo ufw enable 
sudo ufw disable 

#finaly to enable logging and adjusting the log level: 
sudo ufw logging on
sudo ufw logging medium # levels are low, medium, high, full 
````

现在开始 Fail2Ban。

### **Fail2Ban**

主要配置文件位于 /etc/fail2ban/jail.conf，但建议创建本地配置文件：

```shell
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

sudo nano /etc/fail2ban/jail.local
```

jail.local 部分的[DEFAULT]部分有一些基本设置，它们是

```shell
bantime = 10m
findtime = 10m
maxretry = 5
```

封禁时间是指 IP 被封禁的时间。查找时间是 Fail2Ban 寻找重复失败的时间范围，最大重试次数是 IP 被禁用前的失败次数。您可以根据需要调整这些参数。您还可以设置自定义封禁，Fail2Ban 也支持 SSH 等常用服务的封禁。您还可以采取更多步骤，但我认为这已经涵盖了基本内容。

### **NGINX**

您可以使用的网络服务器有很多。Apache, Caddy, nginx, IIS 等等。我使用 Nginx。这是我所熟悉的，而且它运行得非常好。Nginx（发音为 engine-x）是一个网络服务器、反向代理和负载平衡器。作为 Web 服务器，它擅长提供静态内容，并能以相当低的资源占用率处理大量并发连接。作为反向代理，它可以位于应用程序服务器之前，将流量转发给它们，同时确保应用程序的安全。它的负载平衡功能可有效平衡服务器之间的流量，提高可靠性和可扩展性。

通过 apt 安装时，nginx 的默认位置是 /etc/nginx/，nginx.conf 主要用于全局服务器配置，包括 /etc/nginx/sites-enabled 文件夹中的文件。这种模块化结构便于管理多个站点。需要注意的两个文件夹是 sites-enabled 文件夹和 sites-available 文件夹。您可以将可用站点视为测试站点配置的暂存区，而启用站点则用于实时站点和应用程序。常见的做法是在可用站点中的站点中设置和测试配置，然后当你准备上线并获得 SSL 证书时，将文件链接到启用站点文件夹。具体方法如下

```shell
ln -s /etc/nginx/sites-available/yoursitefile /etc/nginx/sites-enabled
```

然后重新加载nginx，并再次检查nginx状态：

```shell
sudo systemctl reload nginx

sudo systemctl status nginx
```

您的网站现在应该已经上线。

下面，我将向您展示一些模板化的 Nginx 网站配置。请务必考虑您的应用程序或网站需求，因为这些只是起点。对于静态网站，这是一个不错的起点。

基本静态网站配置：

```shell
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    root /var/www/example.com/html;
    index index.html index.htm;
    location / {
        try_files $uri $uri/ =404;
    }
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

    # Logging
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log warn;

    # SSL configuration (uncomment after running Certbot)
    # listen 443 ssl http2;
    # listen [::]:443 ssl http2;
    # ssl_protocols TLSv1.2 TLSv1.3;
    # ssl_prefer_server_ciphers on;
    # ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # Certbot will add its own SSL certificate paths
    # ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
}
```

代理通行证配置：

```shell
server {
    listen 80;
    listen [::]:80;
    server_name app.example.com;
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

    # Logging
    access_log /var/log/nginx/app.example.com.access.log;
    error_log /var/log/nginx/app.example.com.error.log warn;

    # SSL configuration (uncomment after running Certbot)
    # listen 443 ssl http2;
    # listen [::]:443 ssl http2;
    # ssl_protocols TLSv1.2 TLSv1.3;
    # ssl_prefer_server_ciphers on;
    # ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # Certbot will add its own SSL certificate paths
    # ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
}
```

WebSocket 升级配置：

```shell
server {
    listen 80;
    listen [::]:80;
    server_name ws.example.com;
    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

    # WebSocket timeout settings
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
    # Logging
    access_log /var/log/nginx/ws.example.com.access.log;
    error_log /var/log/nginx/ws.example.com.error.log warn;

    # SSL configuration (uncomment after running Certbot)
    # listen 443 ssl http2;
    # listen [::]:443 ssl http2;
    # ssl_protocols TLSv1.2 TLSv1.3;
    # ssl_prefer_server_ciphers on;
    # ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # Certbot will add its own SSL certificate paths
    # ssl_certificate /etc/letsencrypt/live/ws.example.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/ws.example.com/privkey.pem;
}
```

基本配置用于为简单的静态网站提供服务。它指定了域名，IPv4 和 IPv6 都使用 80 端口监听，设置了网站的根目录，使用 try\_files 配置了错误处理，添加了一些防止常见网络漏洞的基本标头，设置了访问和错误日志，还包括一个注释掉的 SSL 部分。大部分 SSL 配置将由 certbot 处理，但其中有几行添加了一些 SSL 安全性，可以在运行 certbot 后取消注释。

代理通行配置与基本配置类似，但它不是直接提供文件，而是将请求代理到本地应用程序（在本例中，运行于 3000 端口）。

第三个配置文件面向需要网站连接的应用程序，它与代理通行证配置很相似，只是在允许网络套接字方面做了一些改动。

如果不谈 SSL，任何有关网络服务器的内容都是不完整的。对于普通用户来说，certbot 是他们最好的朋友。它免费，速度快，还他妈的好用。我使用的是 python 版本的 certbot。安装方法如下：

```shell
sudo apt install certbot python3-certbot-nginx
```

安装完成后，你只需在终端运行 "certbot"，它就会检测你的站点启用文件夹中的配置，并询问你要做什么（续费、重发等......）。按照 certbot 提供的步骤操作就可以了，非常简单。

因此，现在 certbot 在获取新证书时会为你设置自动续费，所以这只是个 "坐等 "任务。但为了确保成功，你可以运行

如果运行正常，使用 systemd 的用户就可以放心使用了。

## **Quality Of Life Tools**

关于 "让系统管理更轻松的工具 "这个话题，我将介绍一些我在服务器上使用的工具，我认为它们能让管理变得更轻松。我不会深入介绍任何工具。所有这些都是可选的，没有特定的顺序。其中很多都是我在网站上找到的，如果你和我一样是个终端迷，那么这个网站很值得浏览。

第一个工具，这是我的个人必备清单。Btop 是一款资源终端监控器。它能实时显示电脑的 CPU、内存、磁盘、网络和运行附魔的可视化使用统计信息，它由 C++ 编写，可通过大多数软件包管理器安装。

对于有大量外部连接的服务器（如 nostr 中继站），类似的工具很有帮助。Neoss 的目标是取代常用的 ss 命令，满足基本使用需求。它提供了一个使用中的 TCP 和 UDP 套接字列表及其各自的统计信息。与 SS 原始输出相比，它的主要优势在于清晰简洁的 TUI（终端用户界面），允许你对连接到机器的内容进行排序、刷新和导航。它通过 NPM 安装，这意味着你需要安装 JavaScript。

是一款基于终端的网络服务器日志分析器。它非常适合在终端上快速实时查看日志，还能实时生成 HTML、JSON 和 CSV 报告。GoAccess 可通过大多数软件包管理器安装，适用于所有平台。

接下来要介绍的是 Its，它是一款功能强大的基于文本的文件管理器，具有双面显示屏和许多操作文件和目录的功能。它还具有跨平台特性，可通过大多数软件包管理器进行安装。

与服务器文件管理同属一个主题的是 .NET Framework。这是我的必备清单。它是一款磁盘使用分析器，专门用于查找占用空间的文件。它运行速度快，操作简单。它可以安装在大多数系统和软件包管理器上。Windows 需要安装 Linux 子系统才能使用它。

希望你能从中找到一些有用的东西。我想谈的最后一个话题是 DNS，这是个比较大的话题，所以我不会做大规模的深入探讨，但如果你是自助托管，掌握一些 DNS 的基本知识还是有帮助的。

## **DNS**

DNS 或域名系统是我们所熟知的互联网运作方式的核心部分。不管你喜欢还是讨厌，如果你想访问更广泛的互联网，我们就必须使用它。(我不喜欢它现在的样子，但我不会在这里说这个）。基本上，把 DNS 想象成电话簿。它可以让你在每次需要搜索互联网时输入 duckduckgo.com，而不是 "52.250.42.157"。它将人类容易记住的东西转化为计算机所需的信息，从而真正到达 "duckduckgo.com"。

如果您使用的是 VPS 主机，您唯一需要知道的就是在决定使用某个域名后，如何将 A 记录指向您的服务器 IP。几乎所有的 VPS 主机都可以为你提供一个静态 IP，所以这主要是一种设置和遗忘类型的交易。

在家托管会遇到一些挑战。一个突出的问题是（我经常听到的一个有效问题）没有静态 IP 地址。如今，由于需要 IP 地址的在线设备数量众多，我们要做的事情很多，而且大多数 IP 地址都是动态分配的，除非你从 ISP 付费购买。但还是有解决办法的。这就是动态 DNS 或 DDNS。每当 IP 地址发生变化时，DNS 服务器就会自动更新。设置动态 DNS 的方法多种多样。您可以托管自己的服务或使用主机。下面是一些主机和项目的链接，可供参考。

简而言之，它是这样工作的。你可以选择一个服务提供商，也可以自己设置。你可以获得一个域名，将客户端安装在家庭路由器或服务器上，客户端会定期检查 IP 地址是否发生变化，如果发生变化，它就会更新该域名的 DNS 记录。

## **Docker**

在这里我不会介绍如何安装 docker。无论如何，最好还是按照官方的安装指南来安装。但我想谈几点。首先，docker 在测试新应用程序时非常有用。但我认为也仅此而已。我个人不太喜欢使用 docker，而是尽可能直接运行应用程序。以下是一些值得注意的利弊。

### 优点

如果你的系统可以运行 docker，你就可以运行大多数 docker 应用程序。它有助于隔离，减少应用程序之间的冲突。在某些情况下，它可以帮助提高效率，因为它比传统的虚拟机占用更少的资源。微服务架构也很有用，因为它可以将应用程序分解成更小的可管理服务，从而实现服务的独立扩展。最后，该社区规模庞大，文档完善，社区支持总是很有帮助，而且还有大量现成的 docker 映像可供部署。

### 缺点

首先是开销。虽然它比传统的虚拟机要好，但它比直接在主机上运行要耗费更多的资源，而且输入/输出操作可能会更慢。docker 共享系统内核的事实意味着，一个受损的应用程序可能会影响系统。持久化数据是可行的，但增加了一层复杂性，可能会导致新用户的数据丢失，也会使备份变得更加复杂。使用 docker 时，网络连接也会变得更加复杂，因此不会那么简单。值得注意的是，如果使用 UFW 或 firewalld 作为防火墙，docker 会绕过这些规则。Docker 只与 iptables 兼容。此外，管理良好的 docker 容器有助于管理服务器资源，但管理不当也会对资源造成损害。容器可能会变得过大，从而影响磁盘大小，而错误的配置则会占用过多的服务器资源。在监控和调试应用程序时，尤其是跨多个容器时，它还会增加额外的复杂性。

## 总结

好了，关于服务器设置和工具的基础知识就介绍到这里。有一个工具可以帮你完成大部分工作。我写它是为了让自己的服务器设置更快。你可以在这里获得它，它包含了我所有的必备工具，并做了一些基本配置。请根据自己的需要进行调整，并一如既往地注意安全。

## 原文

- [1] [Server Setup Basics](https://becomesovran.com/blog/server-setup-basics.html)

