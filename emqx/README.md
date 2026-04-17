# EMQX 单机部署指南

这是一份面向腾讯云 CVM 的中文实操版说明，适用于当前仓库里的单机 EMQX 部署方案。

本文按仓库内的以下配置编写：

- EMQX 镜像标签：`6.2.0`
- Dashboard 端口：`18083`
- 明文 MQTT 端口：`1883`
- TLS MQTT 端口：`8883`

当前 `compose.yaml` 的默认安全策略是：

- `18083` 只监听 `127.0.0.1`
- `1883` 只监听 `127.0.0.1`
- `8883` 只监听 `127.0.0.1`

这样做的目的，是先把 EMQX 启动起来，再通过 Dashboard 配置认证和授权，最后才把 `8883` 对公网放开。

## 目录结构

- `compose.yaml`：单机 EMQX Docker Compose 配置
- `.env.example`：环境变量示例文件，复制为 `.env` 后再修改

## 部署目标

第一阶段的目标不是“把所有能力一次性配满”，而是先完成下面这个闭环：

1. 在腾讯云服务器上启动 EMQX
2. 通过 SSH 隧道安全打开 Dashboard
3. 配好设备认证和 Topic 授权
4. 只把 `8883` 暴露给公网
5. 用电脑先完成一次 MQTT over TLS 的连通测试

## 一、上线前确认

开始之前，请确认你已经具备：

- 一台腾讯云 CVM
- Ubuntu 22.04 或 24.04
- 一个可用的公网 IP
- 可以通过 SSH 登录服务器
- 当前仓库里的 `emqx/` 目录

建议服务器最低准备：

- 2 核 CPU
- 2 GB 内存
- 20 GB 系统盘

如果只是自己测试，以上配置足够起步。

## 二、腾讯云控制台需要先做什么

### 1. 确认实例基础信息

在腾讯云控制台确认：

- 实例有公网 IP
- 你知道登录用户名
- 实例所在安全组可以编辑规则

Ubuntu 镜像常见用户名：

- `ubuntu`

CentOS 镜像常见用户名：

- `centos`

### 2. 安全组规则建议

### 启动阶段

在 EMQX 认证和授权还没配好之前，入站规则建议只放：

- `22/TCP`

来源建议填：

- 你当前的公网 IP

不要在这个阶段就放开：

- `1883/TCP`
- `18083/TCP`
- `8883/TCP`

### 放开 MQTT 阶段

当你已经在 EMQX 里配置好认证和授权后，再新增：

- `8883/TCP`

来源可以先用：

- 你当前公网 IP

等开发板接入测试稳定后，再决定是否改成：

- `0.0.0.0/0`

腾讯云官方文档：

- 安全组概述：https://cloud.tencent.com/document/product/213/112610
- 添加安全组规则：https://cloud.tencent.com/document/product/213/39740

## 三、在 Ubuntu 上安装 Docker

推荐使用 Docker 官方 `apt` 仓库方式安装。

先登录服务器：

```bash
ssh ubuntu@<你的服务器公网IP>
```

然后执行：

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

安装完成后验证：

```bash
docker --version
docker compose version
sudo systemctl status docker --no-pager
```

如果当前用户执行 `docker` 提示权限不足，第一阶段直接在命令前加 `sudo` 就可以，不急着折腾用户组。

Docker 官方文档：

- https://docs.docker.com/engine/install/ubuntu/

## 四、把部署目录放到服务器上

推荐两种方式，任选一种。

### 方式 A：从本地上传

如果你本地已经有这个仓库，直接上传 `emqx/` 目录最省事：

```bash
scp -r emqx ubuntu@<你的服务器公网IP>:/opt/
```

上传后登录服务器：

```bash
ssh ubuntu@<你的服务器公网IP>
cd /opt/emqx
```

### 方式 B：在服务器上直接拉代码

如果你的仓库是公开的，可以在服务器上直接执行：

```bash
sudo apt-get update
sudo apt-get install -y git
cd /opt
git clone https://github.com/Qing-King/MQTT-Deploy-CODEX.git
cd MQTT-Deploy-CODEX/emqx
```

如果你的仓库是私有的，建议优先用方式 A。

## 五、准备 `.env`

进入部署目录后执行：

```bash
cp .env.example .env
```

然后编辑：

```bash
vim .env
```

至少修改这一项：

```dotenv
EMQX_DASHBOARD_PASSWORD=改成一个强密码
```

第一阶段请保持下面三项不变：

```dotenv
EMQX_BIND_1883=127.0.0.1
EMQX_BIND_8883=127.0.0.1
EMQX_BIND_18083=127.0.0.1
```

原因很简单：还没配认证之前，不要把 MQTT 端口暴露出去。

## 六、启动 EMQX

在部署目录执行：

```bash
sudo docker compose up -d
sudo docker compose ps
sudo docker logs --tail=100 emqx
```

正常情况下，你应该看到：

- 容器状态是 `Up`
- 日志里没有持续报错

## 七、检查监听端口

在服务器上执行：

```bash
sudo ss -lntp | grep -E ':(1883|8883|18083)\b'
```

第一阶段的预期结果是：

- `127.0.0.1:1883`
- `127.0.0.1:8883`
- `127.0.0.1:18083`

如果你看到的是 `0.0.0.0`，说明端口已经对外监听了，要先回头检查 `.env`。

## 八、通过 SSH 隧道安全打开 Dashboard

不要直接把 `18083` 放到公网。

在你自己的电脑上执行：

```bash
ssh -L 18083:127.0.0.1:18083 ubuntu@<你的服务器公网IP>
```

保持这个 SSH 会话不要关，然后在本机浏览器打开：

- http://127.0.0.1:18083

登录信息：

- 用户名：`admin`
- 密码：你在 `.env` 里设置的 `EMQX_DASHBOARD_PASSWORD`

EMQX Dashboard 官方快速开始：

- https://docs.emqx.com/en/emqx/latest/getting-started/getting-started.html

## 九、先配置认证，再谈公网开放

EMQX 官方文档明确说明：默认情况下，认证功能并不会自动开启。  
所以在生产环境或公网环境里，至少要先配置一种认证方式。

官方文档：

- 认证总览：https://docs.emqx.com/en/emqx/latest/access-control/authn/authn.html

### 推荐第一版方案

建议你现在就用：

- 认证方式：`Password-Based`
- 后端：`Built-in Database`
- `UserID Type`：`username`

这样最适合你当前这种“设备数量不多、先跑通链路”的阶段。

### 在 Dashboard 里创建认证器

路径：

- `Access Control -> Authentication`

操作顺序：

1. 点击 `Create`
2. 选择 `Password-Based`
3. 选择 `Built-in Database`
4. `UserID Type` 选 `username`
5. `Password Hash` 建议选 `bcrypt`
6. 保存

EMQX 内置数据库认证文档：

- https://docs.emqx.com/en/emqx/latest/access-control/authn/mnesia.html

### 创建第一台测试设备账号

建议先创建一台测试设备，比如：

- 用户名：`esp-test-01`
- 密码：`换成你自己的强密码`

这台账号先只拿来做连通性测试，不要一开始就复用到别的设备。

## 十、配置授权规则

认证解决的是“谁能连进来”，授权解决的是“连进来之后能发什么、收什么”。

EMQX 官方文档：

- 授权总览：https://docs.emqx.com/en/emqx/latest/access-control/authz/authz.html
- 内置数据库授权：https://docs.emqx.com/en/emqx/latest/access-control/authz/mnesia.html

### 推荐的第一版 Topic 约定

先统一约定设备 Topic：

- 设备上行：`device/<device_id>/up/#`
- 设备下行：`device/<device_id>/down/#`

例如测试设备 `esp-test-01`：

- 上行：`device/esp-test-01/up/#`
- 下行：`device/esp-test-01/down/#`

### 在 Dashboard 里创建授权器

路径：

- `Access Control -> Authorization`

操作顺序：

1. 点击 `Create`
2. 选择 `Built-in Database`
3. 保持默认 `Max Rules`
4. 保存

### 给测试设备加规则

建议先按“用户名维度”给 `esp-test-01` 建三条规则，顺序如下：

1. `allow subscribe device/esp-test-01/down/#`
2. `allow publish device/esp-test-01/up/#`
3. `deny all #`

这里的意思是：

- 这台设备只允许订阅自己的下行命令
- 这台设备只允许发布自己的上行状态
- 其他 Topic 一律拒绝

如果你在 Dashboard 里是按 `Username` 维度添加权限，就把用户名填成：

- `esp-test-01`

然后逐条添加上面的规则。

第一阶段先不要做太复杂的通配和多租户规则，先把一台设备跑通最重要。

## 十一、只把 `8883` 开放到公网

当认证和授权都已经配好之后，再修改 `.env`：

```dotenv
EMQX_BIND_8883=0.0.0.0
```

然后重启：

```bash
sudo docker compose up -d
sudo ss -lntp | grep 8883
```

此时预期结果应该是：

- `0.0.0.0:8883`

注意，下面两项继续保持本机监听：

- `1883`
- `18083`

也就是说：

- `1883` 不对公网开放
- `18083` 不对公网开放
- 只有 `8883` 可以对外

然后回到腾讯云控制台，在安全组新增一条入站规则：

- 协议端口：`TCP:8883`
- 来源：先填你当前公网 IP
- 策略：允许

等开发板接入测试稳定后，再决定是否改成更宽的来源范围。

## 十二、第一轮外部连通测试

在开发板接入之前，先用电脑做一轮 MQTT over TLS 的烟雾测试。

推荐使用 MQTTX CLI。

MQTTX：

- https://mqttx.app/

### 先测试连接是否成功

在你自己的电脑上执行：

```bash
mqttx conn -h <你的服务器公网IP或域名> -p 8883 --protocol mqtts -u esp-test-01 -P '<你的密码>' --insecure
```

这里加 `--insecure` 的原因是：

- 你当前大概率还没有给 `8883` 换成正式域名证书
- 第一轮只是验证 TLS 通道、账号密码和 ACL 是否工作

### 测试订阅权限

```bash
mqttx sub -h <你的服务器公网IP或域名> -p 8883 --protocol mqtts -u esp-test-01 -P '<你的密码>' -t 'device/esp-test-01/down/#' --insecure
```

这条命令不一定立刻收到消息，但它应该可以成功建立订阅，而不是被鉴权拒绝。

### 测试发布权限

再开一个终端执行：

```bash
mqttx pub -h <你的服务器公网IP或域名> -p 8883 --protocol mqtts -u esp-test-01 -P '<你的密码>' -t 'device/esp-test-01/up/status' -m '{"online":true,"ts":"2026-04-17T00:00:00Z"}' --insecure
```

这一步的目标是验证：

- TLS 握手正常
- 用户名密码正常
- 发布 ACL 正常

如果这一步不报权限错误，就说明 Broker 这一侧已经基本跑通。

## 十三、如果你启用了系统防火墙

腾讯云安全组之外，如果你自己在系统里启用了 `ufw`，还要额外检查：

```bash
sudo ufw status
```

如果 `ufw` 是启用状态，记得补放：

```bash
sudo ufw allow 22/tcp
sudo ufw allow 8883/tcp
```

不要放行这两个端口：

- `18083/tcp`
- `1883/tcp`

如果你没有启用 `ufw`，这一步可以跳过。

## 十四、常见排错命令

查看容器状态：

```bash
sudo docker compose ps
```

查看最近日志：

```bash
sudo docker logs --tail=200 emqx
```

实时看日志：

```bash
sudo docker logs -f emqx
```

查看端口监听：

```bash
sudo ss -lntp | grep -E ':(1883|8883|18083)\b'
```

### 常见问题 1：外网连不上 8883

优先检查：

1. `.env` 里是否已经把 `EMQX_BIND_8883` 改成 `0.0.0.0`
2. 腾讯云安全组是否已经放通 `8883/TCP`
3. 系统防火墙是否拦截
4. 容器日志里是否有 TLS 或监听报错

### 常见问题 2：可以连上，但用户名密码不对

优先检查：

1. 是否已经在 `Access Control -> Authentication` 里创建认证器
2. 设备连接时用的是不是 `username`
3. 你创建用户时填的用户名和客户端使用的用户名是否一致

### 常见问题 3：可以连接，但订阅或发布被拒绝

优先检查：

1. 是否已经创建授权器
2. 规则是否加在正确的用户名下
3. 规则顺序是否把 `deny` 放到了前面
4. Topic 是否和你约定的 `device/<device_id>/up/#`、`device/<device_id>/down/#` 一致

## 十五、下一阶段建议

当这一版跑通之后，下一步建议按这个顺序继续：

1. 给 `8883` 换成正式域名证书，不再依赖 `--insecure`
2. 给每块开发板分配独立 `device_id`
3. 在云端后端里保存设备状态、命令日志和回执
4. 再开始接 ESP 侧 MQTT 客户端代码

当前这份 README 的目标，是先把 Broker 这一层做稳，而不是一次把整套平台做完。

## 官方参考

- Docker 安装 Ubuntu：
  https://docs.docker.com/engine/install/ubuntu/
- EMQX 快速开始：
  https://docs.emqx.com/en/emqx/latest/getting-started/getting-started.html
- EMQX 认证总览：
  https://docs.emqx.com/en/emqx/latest/access-control/authn/authn.html
- EMQX 内置数据库认证：
  https://docs.emqx.com/en/emqx/latest/access-control/authn/mnesia.html
- EMQX 授权总览：
  https://docs.emqx.com/en/emqx/latest/access-control/authz/authz.html
- EMQX 内置数据库授权：
  https://docs.emqx.com/en/emqx/latest/access-control/authz/mnesia.html
- 腾讯云安全组概述：
  https://cloud.tencent.com/document/product/213/112610
- 腾讯云添加安全组规则：
  https://cloud.tencent.com/document/product/213/39740
