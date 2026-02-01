# ts3-to-ts6-guide

# TeamSpeak 3 → TeamSpeak 6 私人服务器迁移指南
本文由gpt生成，但绝大多数操作已经经过UP本人验证有效
**（部署完成 + 提取旧频道结构为止）**

> 适用对象
>
> * 私人 / 小团体服务器
> * 原 TS3 跑在 Linux（CentOS 7）
> * 目标是 TS6（Docker 部署）
> * **不做自动迁移**，只手动复刻结构与图标

---

## 一、环境说明

* 系统：CentOS 7.9 x86_64
* 原服务：TeamSpeak 3 Server（本地部署）
* 新服务：TeamSpeak 6 Server（官方 Docker 镜像）
* Docker 来源：腾讯云镜像源（国内可用）

---

## 二、停止并备份 TS3 服务器

### 1️⃣ 停止 TS3 服务

```bash
kill $(cat ts3server.pid)
```

确认进程已停止：

```bash
ps -ef | grep ts3server
```

---

### 2️⃣ 备份 TS3 文件资源（重点是图标/文件）

进入 TS3 目录（示例）：

```bash
cd /home/lighthouse/teamspeak3
```

打包 `files` 目录：

```bash
tar -czvf ts3-files-backup-$(date +%F).tar.gz files/
```

该压缩包包含：

* 频道图标
* 用户头像
* 文件频道历史内容

---

### 3️⃣ （可选）单独导出图标资源

```bash
mkdir ~/ts3_icons_export
cp -r files/virtualserver_1/internal/icons ~/ts3_icons_export/
cp files/virtualserver_1/internal/icon_* ~/ts3_icons_export/ 2>/dev/null
cp files/virtualserver_1/internal/avatar_* ~/ts3_icons_export/ 2>/dev/null
```

---

## 三、安装 Docker（国内环境）

### 1️⃣ 配置 Docker 官方仓库（替换为腾讯云）

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo sed -i 's|https://download.docker.com|https://mirrors.cloud.tencent.com/docker-ce|g' /etc/yum.repos.d/docker-ce.repo
```

---

### 2️⃣ 安装 Docker 与 Compose

```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

启动并设为开机启动：

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

验证：

```bash
docker --version
docker compose version
```

---

## 四、部署 TeamSpeak 6（Docker）

### 1️⃣ 拉取官方 TS6 镜像

```bash
sudo docker pull teamspeaksystems/teamspeak6-server:latest
```

---

### 2️⃣ 创建部署目录

```bash
mkdir ~/teamspeak6
cd ~/teamspeak6
```

---

### 3️⃣ 创建 `docker-compose.yml`

```yaml
version: "3.8"

services:
  teamspeak:
    image: teamspeaksystems/teamspeak6-server:latest
    container_name: teamspeak6
    restart: unless-stopped
    ports:
      - "9987:9987/udp"
      - "10080:10080/tcp"
      - "30033:30033/tcp"
    environment:
      - TSSERVER_LICENSE_ACCEPTED=accept
    volumes:
      - ts6-data:/var/tsserver

volumes:
  ts6-data:
```

---

### 4️⃣ 启动 TS6

```bash
sudo docker compose up -d
```

查看状态：

```bash
sudo docker ps
sudo docker compose logs -f --tail=100
```

---

### 5️⃣ 记录关键信息（首次启动输出）

日志中会显示：

* Server Query Admin 账号
* ServerAdmin token（权限密钥）

**务必保存**。

---

## 五、验证 TS6 服务

* UDP 9987：语音主端口
* TCP 10080：Web / 管理
* TCP 30033：文件传输

确认防火墙 / 云安全组已放行。

---

## 六、提取 TS3 频道结构（不迁移数据，只读结构）

### 1️⃣ 打开 TS3 SQLite 数据库

```bash
cd /home/lighthouse/teamspeak3
sqlite3 ts3server.sqlitedb
```

---

### 2️⃣ 频道基础表

```sql
PRAGMA table_info(channels);
```

字段说明：

* `channel_id`
* `channel_parent_id`
* `server_id`

---

### 3️⃣ 频道属性表（名称在这里）

```sql
PRAGMA table_info(channel_properties);
```

关键字段：

* `id` → channel_id
* `ident` → 属性名
* `value` → 属性值

---

### 4️⃣ 提取频道名称

```bash
sqlite3 ts3server.sqlitedb \
"SELECT id AS channel_id, value AS name FROM channel_properties WHERE ident='channel_name';"
```

---

### 5️⃣ 提取频道父子关系

```bash
sqlite3 ts3server.sqlitedb \
"SELECT channel_id, channel_parent_id FROM channels;"
```

---

### 6️⃣ 合并为可复刻结构（示例）

最终整理成：

```csv
channel_id,pid,name,sort_order
1,0,"大厅",0
14,0,"工会",2
25,14,"聊天室",0
53,14,"讨论室",25
...
```

> 到这里为止，你已经：
>
> * 完整保留了 **频道树结构**
> * 不依赖 TS6 自动迁移
> * 可以在 TS6 中 **手动、干净地重建频道**

---
