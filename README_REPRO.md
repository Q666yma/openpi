# OpenPI Docker 复现说明

## 1. 项目信息

- 原始项目：https://github.com/Physical-Intelligence/openpi
- 个人仓库：https://github.com/Q666yma/openpi
- 复现 commit：15a9616a00943ada6c20a0f158e3adb39df2ccac

本次复现 OpenPI 开源代码，使用 Docker 构建可运行环境，并运行官方 `simple_client` 示例验证 policy server 推理功能。

## 2. 运行环境

```text
OS: Ubuntu 22.04.4 LTS
GPU: NVIDIA GeForce RTX 4090 48GB
NVIDIA Driver: 580.95.05
Docker: 26.1.0
Docker Compose: v2.26.1

```

```

容器内 GPU 验证成功：

```bash
docker run --rm --gpus all nvidia/cuda:12.2.2-cudnn8-runtime-ubuntu22.04 nvidia-smi
```

## 3. Docker 镜像

已构建镜像：

```text
openpi_server:latest   12.7GB
simple_client:latest   646MB
```

已打包 Docker image：

```text
/home/featurize/work/openpi-repro-images.tar
```

导出命令：

```bash
docker save -o /home/featurize/work/openpi-repro-images.tar \
  openpi_server:latest \
  simple_client:latest
```

导入命令：

```bash
docker load -i /home/featurize/work/openpi-repro-images.tar
```

## 4. 构建命令

```bash
docker build --network=host -t openpi_server -f scripts/docker/serve_policy.Dockerfile .
docker build --network=host -t simple_client -f examples/simple_client/Dockerfile .
```

## 5. 运行命令

```bash
export OPENPI_DATA_HOME=/home/featurize/work/openpi_assets
export SERVER_ARGS="--env ALOHA_SIM"

docker compose -f examples/simple_client/compose.yml up -d openpi_server
docker compose -f examples/simple_client/compose.yml exec openpi_server bash
docker compose -f examples/simple_client/compose.yml run --rm runtime
```

## 6. 容器内检查结果

```text
/app
NVIDIA GeForce RTX 4090, 49140 MiB
openpi import ok
```

## 7. 复现结果

`simple_client` 成功连接 OpenPI policy server，并完成 20 次推理请求。

```text
Running policy: 100% 20/20

client_infer_ms      mean: 86.4 ms
policy_infer_ms      mean: 59.1 ms
server_infer_ms      mean: 81.8 ms
server_prev_total_ms mean: 87.7 ms
```

完整日志保存为：

```text
openpi_simple_client_result.log
```

## 8. 汇报展示流程

```bash
cd /home/featurize/work/openpi

export OPENPI_DATA_HOME=/home/featurize/work/openpi_assets
export SERVER_ARGS="--env ALOHA_SIM"

docker compose -f examples/simple_client/compose.yml up -d openpi_server
docker compose -f examples/simple_client/compose.yml ps
docker compose -f examples/simple_client/compose.yml exec openpi_server bash
```

进入容器后展示：

```bash
pwd
nvidia-smi
/.venv/bin/python -c "import openpi; print('openpi import ok')"
exit
```

然后展示推理结果：

```bash
docker compose -f examples/simple_client/compose.yml run --rm runtime
```

