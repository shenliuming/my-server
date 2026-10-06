# 我的服务器（my-server）

统一管理个人服务器上各项目的部署配置、构建与发布脚本，以及环境搭建说明。

## 目录约定

```text
projects/
  <项目名>/
    README.md
    dev/
      README.md
      nginx/                  # 开发环境站点配置
    test/
      README.md
      nginx/                  # 测试环境站点配置
    production/
      README.md
      nginx/                  # 正式环境站点配置
```

每个环境目录集中存放该环境的 Docker Compose、`.env.example`、应用配置、Nginx 站点配置和操作说明。部署脚本和配置随实际环境搭建逐步补充。

Nginx 站点配置统一归属 `projects/<项目名>/<环境>/nginx/`，域名、静态目录和 API 转发目标随环境独立配置。服务器上的 Nginx 加载需要启用的站点配置。

## 公有仓库的内容边界

- 提交可公开的配置模板、脚本与说明。
- 真实环境变量保存在服务器上的 `.env`，模板使用占位值。
- 密码、访问令牌、SSH 私钥、证书私钥和数据库备份不提交。
- JAR、容器镜像和运行数据使用制品存储或服务器目录管理。

## 项目

- [mjedu-ai](projects/mjedu-ai/README.md)：[开发环境](projects/mjedu-ai/dev/README.md)、[测试环境](projects/mjedu-ai/test/README.md)、[正式环境](projects/mjedu-ai/production/README.md)。
