# Sync binaries of helm

从 Helm 官网 [https://get.helm.sh/](https://get.helm.sh/) 同步 Helm 二进制程序,并通过 GitHub Actions 自动发布到本仓库的 Releases.

## 工作原理

同步流程由本仓库 `.github/workflows/dosync.yaml` 触发,实际同步逻辑实现于共享组合 Action `abldgsync/actions/helm`(其 `dosync.sh` 按 `CS=1~4` 阶段执行);本仓库工作流仅负责调用与传参。核心步骤如下:

1. **解析版本并生成下载清单**
   - 未指定版本时使用 GitHub API 的 `latest` 接口获取最新稳定版;指定版本时通过 `tags/<version>` 接口精确命中.
   - 若解析到预发布版本(`rc`/`beta`/`alpha`)则直接报错退出,避免误发非稳定版.
   - 以 Release 资源中 `.sha256.asc` 文件名(去掉后缀)作为二进制包名,过滤 `loong` 等不支持的架构,生成下载清单.
2. **并行下载全部二进制包**:基于 `xargs -P` 并发下载(默认 8 路并发,失败自动重试).
3. **校验 SHA256**:逐个比对官网 `.sha256` 校验值,保证文件完整性.
4. **发布到 Releases**:以版本号为 tag(如 `v3.19.2`),上传所有下载文件.

> 调用 GitHub API 时已携带 `GITHUB_TOKEN` 鉴权,将限流从 60 次/小时提升到 5000 次/小时.

## 触发方式

- **定时触发**:`schedule` 已预留(每月 10 号凌晨 2 点 `0 2 10 * *`,默认注释),取消注释即可启用,使用最新稳定版.
- **手动触发**(`workflow_dispatch`):可在 Actions 页面手动运行,并支持以下输入参数:

| 参数        | 说明                                                                | 必填 | 示例       |
| ----------- | ------------------------------------------------------------------- | ---- | ---------- |
| `binvern` | 指定 Helm 版本号(如 `3.19.2` 或 `v3.19.2`),不填则使用最新稳定版 | 否   | `3.19.2` |

## 产物

每次同步会在 Releases 中生成一个以版本号命名的发行(如 `Helm v3.19.2 Binaries`),包含对应平台的全部二进制包(`helm-*`),可直接下载使用.

## 目录结构

```
helm/
├── .github/workflows/
│   └── dosync.yaml   # 同步工作流定义
├── LICENSE
└── README.md
```
