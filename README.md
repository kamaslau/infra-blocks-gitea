# trial-gitea

[官方网站](https://docs.gitea.com/)
[安装说明](https://docs.gitea.com/installation/install-with-docker-rootless)

# 默认服务路径

GUI [http://localhost:3000](http://localhost:3000)

## 【可选】docker 镜像源配置

### 下载配置文件

```bash
mkdir -p /etc/docker

curl -o /etc/docker/daemon.json https://disk.liuyajie.com/softwares/daemon.json

systemctl restart docker.service
```

### 手动创建

```bash
mkdir -p /etc/docker

touch /etc/docker/daemon.json

echo "
{
  "registry-mirrors": [
    "https://dockerproxy.com",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com",
    "https://ccr.ccs.tencentyun.com"
  ]
}
" | sudo tee -a /etc/docker/daemon.json

systemctl restart docker.service
```

### 确认配置

```bash
docker info
```

_Registry Mirrors_ 部分应当有相应内容。

## 启动服务

### Start with [Docker Compose](https://docs.docker.com/compose/)

```bash
# Initiate .env file
cp .env.sample .env
# Start services
docker compose up -d
```

Update existing composed containers with latest images:

```bash
docker compose pull && \
docker compose down && \
docker compose up -d
```

## 注册管理员账户

若未在初始化时配置管理员信息，则[注册第一个用户](http://localhost:3000/user/sign_up)后，该用户将自动成为管理员。

## 常用操作

- [迁入代码库](http://localhost:3000/repo/migrate)
- [镜像/推送同步](https://docs.gitea.com/next/usage/repo-mirror/)

## CI/CD/Runner(s)

- https://docs.gitea.com/usage/actions/quickstart
- https://gitea.com/gitea/act_runner/src/branch/main/README.md#run-with-docker

先从管理后台获取 `GITEA_RUNNER_REGISTRATION_TOKEN` 备用： https://your_gitea.com/admin/actions/runners

```bash
docker run --name gitea-runner \
-d \
--restart unless-stopped \
  -e GITEA_INSTANCE_URL=https://your_gitea.com \
  -e GITEA_RUNNER_REGISTRATION_TOKEN=your_token \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitea/act_runner
```
