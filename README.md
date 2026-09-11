# Python 脚本运行镜像

使用 Python 3.12，支持 `linux/amd64`（x86_64）和 `linux/arm64`（ARM64）。不包含 ARMv7 / 32 位支持配置。

Debian 软件源为清华，pip 源为阿里云。基础镜像 `python:3.12-slim-bookworm` 仍从 Docker Hub 拉取；更换 apt / pip 源不会加速基础镜像拉取。

## 构建当前机器架构的镜像

将脚本所需依赖填写到本目录的 `requirements.txt`，然后在本目录执行：

```sh
docker build -t python-runner:3.12 .
```

如需将 pip 源改为清华，将 Dockerfile 中的 `PIP_INDEX_URL` 改为 `https://pypi.tuna.tsinghua.edu.cn/simple`。

镜像保留精简运行环境，没有预装编译器。若依赖没有对应架构的 wheel，可能需要额外安装编译工具和系统库。

## 运行脚本

Linux / macOS，在脚本所在目录执行：

```sh
docker run --rm -v "$(pwd):/scripts" python-runner:3.12 main.py
```

Windows PowerShell，在脚本所在目录执行，Docker Desktop 需使用 Linux 容器：

```powershell
docker run --rm -v "${PWD}:/scripts" python-runner:3.12 main.py
```

脚本参数追加在 `main.py` 后。进入 Python 交互环境可执行 `docker run --rm -it python-runner:3.12`，并在命令末尾追加 `-i`。

挂载目录可读写，脚本生成的文件会保留在宿主机目录内。镜像默认使用基础镜像的 root 用户；Linux 下如需文件归属当前用户，可增加 `--user "$(id -u):$(id -g)"`。

## 使用 GitHub Actions 自动发布

仓库包含 `.github/workflows/docker-image.yml`。推送到 `main`、推送 `v*` 标签或手动运行工作流时，会构建两种架构并发布到 GitHub Container Registry（GHCR）。工作流使用自带的 `GITHUB_TOKEN`，无需添加个人访问令牌。

镜像地址为 `ghcr.io/<仓库所有者>/<仓库名>`，名称统一转为小写。`main` 分支发布 `latest` 和 `main` 标签；例如 Git 标签 `v1.0.0` 发布同名镜像标签，每次构建还生成提交 SHA 标签。

发布成功后，在脚本目录运行（PowerShell）：

```powershell
docker run --rm -v "${PWD}:/scripts" ghcr.io/OWNER/REPOSITORY:latest main.py
```

请将 `OWNER/REPOSITORY` 替换为实际的小写仓库路径。GHCR 包首次发布通常为私有；需要匿名拉取时，在 GitHub 包设置中将可见性改为 Public。私有包需要先使用具备 `read:packages` 权限的令牌登录 GHCR。

仓库或组织需要允许 GitHub Actions 创建包。如果已有同名包，还需要授予本仓库对该包的写入权限。

## 手动发布到阿里云镜像仓库（可选）

需要 Docker Buildx，且构建器应支持这两种架构（通过原生节点或 QEMU 模拟）。Docker Desktop 通常已提供模拟支持。

先登录镜像仓库，再创建构建器；下列地址需替换为实际仓库地址：

```sh
docker login registry.cn-hangzhou.aliyuncs.com
docker buildx create --name python-runner-builder --driver docker-container --use
docker buildx inspect --bootstrap
docker buildx build --platform linux/amd64,linux/arm64 -t registry.cn-hangzhou.aliyuncs.com/YOUR_NAMESPACE/python-runner:3.12 --push .
```

构建器只需创建一次，后续使用 `docker buildx use python-runner-builder`。阿里云 ACR 实例的实际登录地址可能不同，请以控制台为准。

验证远端镜像包含两种架构：

```sh
docker buildx imagetools inspect registry.cn-hangzhou.aliyuncs.com/YOUR_NAMESPACE/python-runner:3.12
```

发布后，两种架构的机器都使用同一个镜像地址，Docker 自动选择对应架构。默认采用 `--push` 发布多架构镜像，避免依赖本地镜像存储对多架构加载的支持。

当前生成环境没有 Docker 命令，尚未执行镜像构建和容器运行验证。
