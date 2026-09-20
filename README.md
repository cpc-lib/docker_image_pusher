# Docker Images Pusher

使用 GitHub Actions 将境外 Docker 镜像自动转存到阿里云容器镜像服务（个人版），供国内服务器快速拉取。免费、免服务器、全自动。

**特性**

- 支持任意来源仓库：Docker Hub、gcr.io、ghcr.io、quay.io、registry.k8s.io、k8s.gcr.io 等
- 支持最大 40GB 的大型镜像（构建时自动扩容磁盘）
- 走阿里云官方线路，国内拉取速度快
- 提交即构建：修改 `images.txt` 后自动触发转存

> 本项目 Fork 自 [tech-shrimp/docker_image_pusher](https://github.com/tech-shrimp/docker_image_pusher)（Apache-2.0 License），视频教程见[原仓库](https://github.com/tech-shrimp/docker_image_pusher)。

## 使用步骤

### 1. 配置阿里云容器镜像服务

登录 [阿里云容器镜像服务控制台](https://cr.console.aliyun.com/)：

1. 开通**个人实例**（免费）
2. 创建一个**命名空间**，即 `ALIYUN_NAME_SPACE`
3. 在 **访问凭证** 页面获取以下三个值（需设置固定密码）：

| 环境变量 | 说明 |
|---|---|
| `ALIYUN_NAME_SPACE` | 命名空间名称 |
| `ALIYUN_REGISTRY` | 仓库地址，如 `registry.cn-hangzhou.aliyuncs.com` |
| `ALIYUN_REGISTRY_USER` | 访问凭证用户名 |
| `ALIYUN_REGISTRY_PASSWORD` | 访问凭证密码 |

### 2. Fork 并配置本项目

1. Fork 本项目，进入自己的仓库，在 **Actions** 页启用 GitHub Actions
2. 进入 **Settings → Secrets and variables → Actions → New repository secret**
3. 将上表中的 **4 个值** 分别配置为 Repository secrets

### 3. 添加镜像

编辑 [images.txt](images.txt)，每行一个镜像，提交后自动触发构建：

```
quay.io/minio/mc:latest
nginx:1.25.3
gcr.io/google-containers/pause:3.9
```

**语法说明**

| 写法 | 说明 |
|---|---|
| `nginx` | 不带 tag 默认拉取 `latest` |
| `nginx:1.25.3` | 指定 tag |
| `nginx@sha256:xxxx` | 支持摘要，转存时自动去掉摘要后缀 |
| `--platform=linux/arm64 nginx:1.25.3` | 指定架构，多架构镜像需手动指定 |
| `# 注释` | `#` 开头的行及空行会被忽略 |

### 4. 使用镜像

构建完成后，在阿里云容器镜像服务中即可看到转存好的镜像（可将仓库设为公开，拉取免登录）。在国内服务器上拉取：

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/<命名空间>/<镜像名>:<tag>
```

其中 `registry.cn-hangzhou.aliyuncs.com` 即 `ALIYUN_REGISTRY`，`<命名空间>` 即 `ALIYUN_NAME_SPACE`。

## 进阶用法

### 多架构镜像

在镜像前加 `--platform` 参数，指定后的架构会以**前缀**形式加在镜像名前：

```
--platform=linux/arm64 nginx:1.25.3
```

转存后镜像名为 `linux_arm64_nginx:1.25.3`。

### 镜像重名

不同命名空间下存在同名镜像时，程序自动将命名空间作为前缀加以区分：

```
xhofe/alist
xiaoyaliu/alist
```

转存后镜像名分别为 `xhofe_alist` 和 `xiaoyaliu_alist`。

### 定时执行

默认只在推送到 `main` 分支或手动触发时运行。如需定时同步，在 [.github/workflows/docker.yaml](.github/workflows/docker.yaml) 中添加 `schedule`（cron 为 UTC 时区）：

```yaml
on:
  workflow_dispatch:
  push:
    branches: [ main ]
  schedule:
    - cron: '0 1 * * *'  # 每天 UTC 1:00（北京时间 9:00）
```

## 常见问题

**Q: 拉取报 `pull access denied ... repository does not exist`？**

先确认镜像和 tag 真实存在。注意部分厂商已不再将镜像托管在 Docker Hub，需写完整的仓库地址。例如 MinIO 已迁移到 quay.io，应写 `quay.io/minio/mc:latest` 而不是 `minio/mc:latest`。

**Q: 支持哪些来源仓库？**

任意可通过 `docker pull` 拉取的仓库，包括 Docker Hub、gcr.io、ghcr.io、quay.io、registry.k8s.io 等。

**Q: 构建失败提示磁盘空间不足？**

工作流已默认清理 runner 磁盘并挂载 40GB 构建空间。如仍不够用，可打开 `docker.yaml` 中 `remove-android`、`remove-codeql` 注释进一步释放空间。

## 致谢

- 原项目：[tech-shrimp/docker_image_pusher](https://github.com/tech-shrimp/docker_image_pusher)，作者 [技术爬爬虾](https://github.com/tech-shrimp/me)
- 参考视频：https://www.bilibili.com/video/BV1Zn4y19743/
- License: [MIT](LICENSE)
