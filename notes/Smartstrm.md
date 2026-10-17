以下是针对环境（Alist + jellyfin公网部署 + SmartStrm + Caddy 反代）整理的完整指南。

---

## Jellyfin + SmartStrm 302直链

### Docker compose 内容

**Jellyfin + Smartstrm**
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

**Alist**
```yaml
version: '3.3'
services:
  alist:
    image: 'xhofe/alist:beta'
    container_name: alist
    volumes:
      - './data:/opt/alist/data'
    ports:
      - '5244:5244'
    environment:
      - PUID=0
      - PGID=0
      - UMASK=022
    restart: unless-stopped
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
                header_up User-Agent "pan.baidu.com"
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

alist.example.com {
        reverse_proxy 127.0.0.1:5244 {
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
3. 配置Alist(如需要)
4. 配置SmartStrm生成strm文件并打开302代理
5. 配置jellyfin

#### 百度网盘有关问题(Alist + Smartstrm + Jellyfin)

```
Jellyfin 客户端 
  ↓ 播放视频
通过 jf.example.com (Caddy 代理 8097 端口)
  ↓
Smartstrm 的 302 代理 (8097 端口)
  ↓ 302 重定向
直接访问百度网盘的真实下载地址
  ✗ 百度网盘检查 User-Agent → 不匹配,拒绝播放
```

**问题出在哪?**  
当 Jellyfin 客户端播放视频时,它向 `jf.example.com` 发送请求,这个请求最终会被 302 重定向到百度网盘的真实地址。但 Jellyfin 客户端的 User-Agent 不是百度网盘认可的,所以播放失败。

**解决方案的原理:**  
在 Caddy 这一层,把所有通过 `jf.example.com` 的请求的 User-Agent 统一改成百度网盘认可的 UA。这样当请求到达百度网盘时,UA 就是正确的了。

```
Jellyfin 客户端 (UA: Jellyfin/...)
  ↓
Caddy (jf.example.com)
  ↓ [在这里把 UA 改成百度网盘认可的]
  ↓
Smartstrm 的 302 代理 (UA 已被修改)
  ↓ 302 重定向
百度网盘 (检查 UA → ✓ 匹配,允许播放)
```

**具体操作方法**  
修改 `/etc/caddy/Caddyfile`,在 `jf.example.com` 配置块中添加一行 `header_up User-Agent`:

```caddy
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
                header_up User-Agent "pan.baidu.com"
        }
}
```
