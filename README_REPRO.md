# OpenPI LIBERO Benchmark 复现说明

## 1. 复现内容

本次复现基于 OpenPI 官方源码和官方配置，构建可运行环境，并运行官方 LIBERO benchmark 对 `pi05_libero` checkpoint 进行评估。

本次复现内容包括：

- 构建 OpenPI 运行环境
- 启动 OpenPI policy server
- 进入 Docker container 验证运行环境
- 运行官方 LIBERO benchmark
- 完成 4 个 LIBERO task suite，共 2000 次 episode 评估

## 2. 项目信息

- 原始项目：https://github.com/Physical-Intelligence/openpi
- 个人仓库：https://github.com/Q666yma/openpi
- OpenPI upstream commit：`15a9616a00943ada6c20a0f158e3adb39df2ccac`

## 3. 运行环境

```text
OS: Ubuntu 22.04.4 LTS
GPU: NVIDIA GeForce RTX 4090 48GB
Docker: 26.1.0
Docker Compose: v2.26.1
CUDA container base image: nvidia/cuda:12.2.2-cudnn8-runtime-ubuntu22.04
Rendering backend for LIBERO: EGL
```

## 4. Docker Image

本次构建得到以下 Docker image：

```
openpi_server:latest
libero:latest
```
同时已上传到 Docker Hub 作为备份：

```
morgan11/openpi-server:latest
morgan11/openpi-libero:latest
```

## 5. 启动 Policy Server

```
cd /home/featurize/work/openpi

docker rm -f libero_openpi_server_proxy 2>/dev/null || true

docker run -d --name libero_openpi_server_proxy \
  --gpus all \
  --network host \
  -v /home/featurize/work/openpi:/app \
  -v /home/featurize/data/openpi_assets:/openpi_assets \
  -e SERVER_ARGS="--env LIBERO" \
  -e OPENPI_DATA_HOME=/openpi_assets \
  -e IS_DOCKER=true \
  openpi_server:latest
```

查看 server 日志：

```
docker logs -f libero_openpi_server_proxy
```

当日志中出现以下内容时，policy server 启动成功：

```
server listening on 0.0.0.0:8000
```

## 6. 进入 Docker Container

进入 OpenPI policy server container：

```
docker exec -it libero_openpi_server_proxy bash
```

在 container 内验证 GPU 和 Python 环境：

```
pwd
nvidia-smi
/.venv/bin/python -c "import openpi; print('openpi import ok')"
```

退出 container：

```
exit
```

## 7. 运行 LIBERO Benchmark

本次运行官方 LIBERO benchmark，包括以下 4 个 task suite：

```
libero_spatial
libero_object
libero_goal
libero_10
```

每个 suite 包含 10 个任务，每个任务运行 50 次 trial，因此总计：

```
4 suites × 10 tasks × 50 trials = 2000 episodes
```

运行命令：

```
cd /home/featurize/work/openpi

mkdir -p /home/featurize/data/libero_results_egl
mkdir -p /home/featurize/data/libero_videos_egl

for suite in libero_spatial libero_object libero_goal libero_10; do
  echo "===== Running ${suite}, 50 trials per task with EGL ====="
  mkdir -p "/home/featurize/data/libero_videos_egl/${suite}"

  export CLIENT_ARGS="--args.task-suite-name ${suite} --args.num-trials-per-task 50 --args.video-out-path /tmp/libero/videos"

  docker compose -f examples/libero/compose.yml run --rm --no-deps \
    -e MUJOCO_GL=egl \
    -e PYOPENGL_PLATFORM=egl \
    -e MUJOCO_EGL_DEVICE_ID=0 \
    -e NVIDIA_DRIVER_CAPABILITIES=compute,utility,graphics,display \
    -e CLIENT_ARGS="$CLIENT_ARGS" \
    -v "/home/featurize/data/libero_videos_egl/${suite}:/tmp/libero/videos" \
    runtime 2>&1 | tee "/home/featurize/data/libero_results_egl/${suite}_50trials.log"
done
```

查看 benchmark 结果：

```
for f in /home/featurize/data/libero_results_egl/*_50trials.log; do
  echo "==== $(basename "$f") ===="
  grep -E "Total success rate|Total episodes" "$f" | tail -2
done
```

## 8. 复现结果

本次 LIBERO benchmark 复现结果如下：

```
libero_spatial: Total success rate = 0.976, Total episodes = 500
libero_object:  Total success rate = 0.988, Total episodes = 500
libero_goal:    Total success rate = 0.972, Total episodes = 500
libero_10:      Total success rate = 0.920, Total episodes = 500
```

平均成功率：

```
Average success rate = (0.976 + 0.988 + 0.972 + 0.920) / 4 = 0.964
```

即平均成功率为 **96.4%**。

## 9. 复现结论

本次复现成功构建了 OpenPI 运行环境，并在 Docker container 中启动 OpenPI policy server。

基于官方 `pi05_libero` checkpoint，完成了 OpenPI 官方 LIBERO benchmark 评估复现。最终完成 4 个 task suite、40 个任务、共 2000 次 episode，平均成功率为 96.4%。
