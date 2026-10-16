以下是针对环境（飞牛/jellyfin公网部署 + SmartStrm + Caddy 反代）整理的完整指南。

---

## Jellyfin + SmartStrm 302直链

### Docker compose 内容

```yaml
name: jellyfin-strm-stack
services:
  # 1. Jellyfin 媒体服务器 (原生端口 8096)
  jellyfin:
    image: nyanmisaka/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host
    environment:
      - TZ=Asia/Shanghai
      - PUID=0
      - PGID=0
    volumes:
      - /opt/media-stack/jellyfin/config:/config
      - /opt/media-stack/jellyfin/cache:/cache
      - /opt/media-stack/strm:/strm

  smartstrm:
    image: cp0204/smartstrm:latest
    container_name: smartstrm
    restart: unless-stopped
    network_mode: host
    volumes:
      - /opt/media-stack/smartstrm/config:/app/config
      - /opt/media-stack/smartstrm/logs:/app/logs
      - /opt/media-stack/strm:/strm
    environment:
      - TZ=Asia/Shanghai
      - PORT=8024
      - ADMIN_USERNAME=admin
      - ADMIN_PASSWORD=密码
```

### Caddyfile

```txt
jf.example.com {
        reverse_proxy 127.0.0.1:8097 {
                flush_interval -1
                transport http {
                        read_buffer 0
                        write_buffer 0
                }
                header_up Host {host}
                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-For {remote_host}
                header_up X-Forwarded-Proto {scheme}
        }
}

strm.example.com {
        reverse_proxy 127.0.0.1:8024 {
                flush_interval -1
                transport http {
                        read_buffer 0
                        write_buffer 0
                }
                header_up Host {host}
                header_up X-Real-IP {remote_host}
                header_up X-Forwarded-For {remote_host}
                header_up X-Forwarded-Proto {scheme}
        }
}
```

### 步骤

> 需注意端口问题

1. 部署Docker容器
2. 配置caddy反代
3. 配置SmartStrm生成strm文件并打开302代理
4. 配置jellyfin

## 飞牛影视 + SmartStrm 302 直链完整部署与排错指南

### 一、核心目标

- 使用 SmartStrm 生成 STRM 文件
- 开启 302 代理，让飞牛影视播放时流量**不经过 NAS**
- 通过域名访问（Caddy 反代），实现外网正常播放

### 二、正确部署步骤

#### 1. Docker Compose 部署 SmartStrm（推荐 host 网络）

```yaml
name: smartstrm
services:
  smartstrm:
    image: cp0204/smartstrm:latest
    container_name: smartstrm
    restart: unless-stopped
    network_mode: host
    volumes:
      - /vol1/1000/docker/smartstrm/config:/app/config
      - /vol1/1000/docker/smartstrm/logs:/app/logs
      - /vol1/1000/docker/smartstrm/strm:/strm
    environment:
      - TZ=Asia/Shanghai
      - PORT=8024
      - ADMIN_USERNAME=admin
      - ADMIN_PASSWORD=强密码
      # 购买 Pro 后填入
      # - LICENSE=许可证
```

#### 2. SmartStrm 关键配置

**系统设置 → 302代理**
- 服务端口：`8097`
- 媒体服务器地址：`http://127.0.0.1:8005`（飞牛影视实际端口，请自行确认）
- 启用代理

**系统设置 → STRM 设置**
- SmartStrm 基础地址：填你的域名
- 开启「对 STRM 进行 URL 编码」
- 云盘类存储生成类型：推荐「文件编号模式」
- 启用签名

#### 3. Caddy 反代配置（最关键）

```caddy
example.domain.com {
    reverse_proxy 127.0.0.1:8097 {
        flush_interval -1          # 必须加！禁止缓冲
        transport http {
            read_buffer  0
            write_buffer 0
        }
        header_up Host {host}
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

修改后执行：
```bash
caddy reload
```

### 三、客户端连接方式

飞牛播放器 / App 中填写：

- **正确**：`https://example.domain.com`（走 Caddy → 8097）
- 或临时测试：`公网IP:8097`

**注意**：播放必须走 8097 端口（或对应域名），管理/刮削可以继续用原端口。

### 四、验证是否真正 302 成功

1. 播放时查看 SmartStrm 日志，应出现：
   ```
   STRM VOD : [存储名]文件名
   MediaItem: xxxxx 命中 STRM 302
   ```
2. 观察 NAS 上行流量：几乎没有大流量上传 → 说明已直连网盘
3. 手动用浏览器访问 STRM 文件内容，应能跳转到网盘直链并播放

### 五、常见排错顺序（建议按此执行）

1. 确认客户端连的是代理端口/域名
2. 查看 SmartStrm 日志是否出现「命中 STRM 302」
3. 手动测试 STRM 链接是否能播放
4. 对比「公网 IP:8097」与「域名」是否表现一致
5. 检查 Caddy 是否加了 `flush_interval -1`
6. 关闭「优化 STRM 首播速度」再试

按以上配置和排错顺序，基本可以稳定实现飞牛影视 302 直链播放。