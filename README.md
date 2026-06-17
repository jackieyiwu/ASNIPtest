# ASNIPtest

从 ASN 拉取 IP 段 → 端口扫描 → Cloudflare 反代节点检测 → 输出可用 CF 节点。

## 一键安装

```bash
curl -fsSL https://raw.githubusercontent.com/jackieyiwu/ASNIPtest/refs/heads/main/install.sh | bash
```

## 卸载

```bash
curl -fsSL https://raw.githubusercontent.com/jackieyiwu/ASNIPtest/refs/heads/main/uninstall.sh | bash
```

## 使用

```bash
cd ~/ASNIPtest
python3 run.py AS209242
```

多个 ASN：

```bash
python3 run.py AS209242,AS3214
```

## 挂机执行（断开 SSH 后持续运行）

为了在断开 SSH 会话后脚本能持续运行，建议使用以下三种方式之一：

### 1. 推荐：使用 Tmux (推荐，方便查看实时日志)
Tmux 允许你在一个会话中运行任务，即使断开连接，任务也会在后台继续。

*   **创建会话**：`tmux new -s asnip`
*   **执行任务**：在会话内执行 `python3 run.py AS45102`
*   **退出会话** (让任务后台运行)：按下 `Ctrl + B`，然后松开按键，再按下 `D` 键。
*   **恢复会话**：再次连接 SSH 后，输入 `tmux attach -t asnip` 即可回到运行界面查看输出。

### 2. 备选：使用 Nohup (简单粗暴)
如果不需要中途查看实时进度，可以使用 `nohup` 将任务挂载到后台。

*   **执行指令**：
    ```bash
    nohup python3 -u run.py AS45102 > output.log 2>&1 &
    ```
*   **关键点**：
    *   `-u`：强制 Python 使用无缓冲模式，确保日志实时写入文件。
    *   `> output.log 2>&1`：将标准输出和错误输出都存入 `output.log`。
    *   `&`：放入后台运行。
*   **查看状态**：执行 `tail -f output.log` 查看最新日志。

### 3. 稳健：使用 Systemd (长期守护)
适用于需要开机自启或自动重启的生产环境。
创建一个 `/etc/systemd/system/asnip.service` 文件，设置 `Restart=always` 即可保证服务永不掉线。

---

## 流程

```
ASN(命令行) → RIPEStat → CIDR → prips 展开 IP
    → masscan 端口扫描 → cf-scanner 粗筛 → API 精筛 → CSV
```

## 依赖工具

| 工具 | 用途 | 来源 |
|------|------|------|
| [masscan](https://github.com/robertdavidgraham/masscan) | 高速端口扫描 | `apt install masscan` |
| [prips](https://manpages.debian.org/prips) | CIDR IP 段展开 | `apt install prips` |
| [RIPEStat API](https://stat.ripe.net/) | ASN → CIDR 查询 | 免费公开 API |
| cf-scanner | CF 反代检测 | 内置 Go 源码，自动编译 |
| 精筛 API | 二次验证节点可用性 | 内置 |

## 输出

运行完成后自动输出 CSV 文件，并提供临时下载链接：

```
📥 下载链接 (临时, 按回车关闭):
http://1.2.3.4:8899/output_AS209242_20260616_120000.csv
```

CSV 列：IP地址, 端口, TLS, 数据中心, 地区, 城市, 网络延迟, 下载速度, ASN

> 下载链接自动检测公网出口 IP（ipify + ip.sb 双 API 备用），支持 NAT/Docker 环境。

## 硬件自适应

根据 CPU 核数/内存自动调整参数：

| 配置 | masscan | cf并发 | API并发 |
|------|---------|--------|---------|
| 2核1G | 2,000 pps | 200* | 8 |
| 4核2G | 4,000 pps | 400 | 32 |
| 16核16G | 16,000 pps | 500 | 32 |

> *cf-scanner 并发最低 200，最高 500。
