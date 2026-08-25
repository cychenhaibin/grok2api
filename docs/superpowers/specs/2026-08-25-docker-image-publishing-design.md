# Docker 镜像发布设计

## 目标

在推送正式版本 Git tag 或手动指定已有 tag 时，为 Grok2API 发布 Docker Hub 多架构镜像。消费者继续使用 `chenyme/grok2api:<版本>` 或 `chenyme/grok2api:latest`，由镜像 manifest 自动选择 `linux/amd64` 或 `linux/arm64`。

## 触发与版本

- 推送除 `nightly*` 以外的 Git tag 时自动发布。
- `workflow_dispatch` 接收一个 tag 名，并校验仓库中存在该 tag。
- 工作流将选定 tag 写入根目录 `VERSION`，以便镜像内的版本信息与镜像 tag 一致。

## 构建与发布

1. Matrix 在原生 `ubuntu-latest`（amd64）和 `ubuntu-24.04-arm`（arm64）runner 分别构建根目录 `Dockerfile`。
2. 每个架构推送 `chenyme/grok2api:<版本>-<架构>` 与 `chenyme/grok2api:latest-<架构>`。
3. 汇总 job 用 `docker buildx imagetools create` 分别创建 `<版本>` 与 `latest` 的多架构 manifest。
4. 构建采用 GitHub Actions 缓存，并产生 build provenance 与 SBOM；每个架构镜像和最终 manifest 使用 Cosign keyless 签名。

## 认证与权限

- 使用 `DOCKERHUB_USERNAME` 和 `DOCKERHUB_TOKEN` 登录 Docker Hub。
- 构建 job 请求 `contents: read` 与 `id-token: write`；汇总 job 仅请求 `id-token: write`。
- 不向日志输出 Docker Hub 凭据。

## 错误处理与验证

- 两个架构均成功后才发布多架构 tag；任一架构失败会阻止 manifest 发布。
- 手动构建若 tag 不存在，立即失败并写出明确错误。
- 汇总 job 将 `imagetools inspect` 结果写入 GitHub Actions Summary，供人工确认两个架构均在 manifest 中。

## 范围外

- 不发布 GitHub Container Registry 镜像。
- 不为分支或 pull request 自动推送镜像。
- 不修改现有 Dockerfile、Compose 文件或运行时配置。
