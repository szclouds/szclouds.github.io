---
title: 基于 gost 搭建网络流量转发
layout: default
nav_order: 7
tags: gost cloudflare vps freenom
permalink: /docs/gost
---
# 基于 gost 搭建网络流量转发

## 准备

1. [gost](https://github.com/ginuerzh/gost)-GO语言实现的安全隧道
   - 支持多级转发
   - 支持多种协议：http、http2、socket、websocket（本教程使用此协议中的mwss）
   - mwss—GOST 在 Websocket 基础之上扩展出具有多路复用(Multiplex)特性的传输类型
   - 支持 docker 快速部署
   - 支持自定义域名
2. 域名 freenom （免费，目前官网显示暂不可注册），其他域名服务商：[dynadot](https://www.dynadot.com/)，邮箱注册，支付宝购买即可，无需其他操作
   - 作用1：可通过 cloudflare 进行代理域名、解析 DNS 或代理流量并转发
   - 作用2：使用 [Let’s Encrypt ](https://certbot.eff.org/instructions) 来签发证书
3. vps 机器，根据个人需要选择，[bwh](https://bwh81.net/)、[vultr](https://my.vultr.com/) 等
4. cloudflare 流量代理/DNS代理
   - 代理 DNS 或**转发流量**
   - 支持免费代理域名

## 步骤

1. 注册域名

2. 使用 cloudflare 代理域名，根据提示选择免费代理即可，设置完成后需返回域名服务提供商处修改对于的 DNS 解析地址，使得域名能够正确解析到 cloudflare 上，设置完成后需要等待一段时间，5分钟左右生效，域名代理成功后 cloudflare 对应有提示。

    ![image-20230318150547801](/assets/images/cloudflare_success.png)

3. 购买 VPS

4. 在 cloudflare 中创建 A 记录，填写名称和 IP，其中 IP 为 VPS IP 地址，此时代理状态设置为仅代理 DNS

    ![image-20230318151058076](/assets/images/cloudflare_dns.png)

5. 验证 cloudflare 是否成功将域名与 VPS 绑定，`ping domain` 如 IP 为 VPS IP 则说明代理成功

6. 登陆 VPS 执行如下步骤

   - 开启 BBR TCP 拥塞算法，大部分云服务商提供的机器都具备，可先查看是否已启动 BBR

     ``` shell
     # 如果结果都有 bbr，则证明你的内核已开启 BBR。
     # 执行 lsmod | grep bbr，看到有 tcp_bbr 模块即说明 BBR 已启动，则无需下面的开启步骤。
     
     apt install --install-recommends linux-generic-hwe-16.04
     sudo modprobe tcp_bbr
     echo "tcp_bbr" | sudo tee --append /etc/modules-load.d/modules.conf
     
     echo "net.core.default_qdisc=fq" | sudo tee --append /etc/sysctl.conf
     echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee --append /etc/sysctl.conf
     
     sudo sysctl -p
     sysctl net.ipv4.tcp_available_congestion_control
     sysctl net.ipv4.tcp_congestion_control
     ```
     
   - 安装 docker，参照[官网](http://docker.io)教程即可 

   - 防火墙开启 80 端口，后续证书安装需要访问 80 端口，如果不可用将会导致证书安装失败 

   - 签发证书 

     ~~~shell
     sudo snap install core; sudo snap refresh core
     
     sudo snap install --classic certbot
     
     sudo ln -s /snap/bin/certbot /usr/bin/certbot
     
     sudo certbot certonly --standalone
     
     # 按照提示输入邮箱和域名
     ~~~

   - 新建一个 start.sh 文件编写 gost 启动脚本，选用 mwss 协议

     ~~~shell
     #!/bin/bash
     # 下面的四个参数需要改成你的
     DOMAIN="YOU.DOMAIN.NAME"
     USER="username"
     PASS="password"
     PORT=443

     BIND_IP=0.0.0.0
     CERT_DIR=/etc/letsencrypt
     CERT=${CERT_DIR}/live/${DOMAIN}/fullchain.pem
     KEY=${CERT_DIR}/live/${DOMAIN}/privkey.pem
     sudo docker run -d --name gost \
     -v ${CERT_DIR}:${CERT_DIR}:ro \
     --net=host ginuerzh/gost \
     -L "mwss://${USER}:${PASS}@:${PORT}?cert=${CERT}&key=${KEY}"
     ~~~

   - 给 start.sh 脚本执行权限，`chmod +X start.sh` 执行脚本：`sh start.sh`
   - 验证 gost 服务启动情况

     ```shell
      nc -zvw 3 DOMAIN PORT # 查看端口是否已监听
      docker logs ${your gost containerId} # 查看容器日志输出情况
     ```
   - 证书自动更新策略和定时重启 gost 服务，`crontab -e` 编辑定时任务

     ```shell
      0 0 1 * * /usr/bin/certbot renew --force-renewal
      5 0 1 * * /usr/bin/docker restart gost
     ```

7. 将 cloudflare A 记录中的代理状态改为完成代理，并设置 SSL/TLS 为完全，此步骤需要等待 5 分钟左右才会生效

8. 验证 cloudflare 是否完成代理，`ping domain` 若 IP 非 VPS IP 则说明代理成功，此处可将输出的 IP 在[在线工具](https://www.ip138.com/)中查询 IP 归属是否为 cloudflare 节点

9. 客户端配置，参考 [gost](https://github.com/ginuerzh/gost) 下载对应的客户端

   ```shell
      # windows 可使用 git-bash 客户端执行，在 git-bash 中配置 .bash_profile，在 windows 中配置 git-bash 开机自启动，这样无需每次手动操作
      ./gost-windows-386.exe -L socks5://:1080 -F  mwss://username:password@domain:443
      # macos 中同样也可设置自启动或后台运行
      nohup gost -L socks://:1080 -F mwss://username:password@domain:443 &
   ```

10. Chrome 浏览器配置，使用 Chrome 插件 [SwitchyOmega](https://github.com/FelisCatus/SwitchyOmega) ，可通过新建不同情景模式配置规则达到非全局转发目的。

    ![image-20230318152425472](/assets/images/switchOmega.png)

11. 在 Chrome 页面中选择不同的 SwitchyOmega 模式即可

     

## 参考

- [haoel](https://haoel.github.io/)
- [gost](https://github.com/ginuerzh/gost)
- [SwitchyOmega](https://github.com/FelisCatus/SwitchyOmega) 
