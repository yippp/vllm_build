# vllm_build

为 **A800（SM80 / Ampere）** 与 **H20（SM90 / Hopper）** 构建同一个 [vLLM](https://github.com/vllm-project/vllm) CUDA wheel。`TORCH_CUDA_ARCH_LIST=8.0 9.0`，一份 wheel 同时用于两种 GPU。

当前默认目标：**vLLM 0.27.1**。编译使用 CUDA 12.8 toolkit（与本机驱动 CUDA 12.8 对齐）；运行时仍使用 cu126 PyTorch wheels。cu129 / CUDA 12.9 不可用。

## 构建固定版本（0.27.x）

| 项目 | 值 |
| --- | --- |
| vLLM | `0.27.1`（可改） |
| PyTorch | `2.13.0+cu126` |
| torchvision（运行时建议） | `0.28.0+cu126` |
| torchaudio（上游 v0.27.1 固定） | `2.11.0+cu126` |
| Triton（运行时建议） | `3.7.1` |
| CUDA 构建容器 | `nvidia/cuda:12.8.0-devel-ubuntu22.04` |
| Python | `3.12` |
| GPU arch | `8.0 9.0`（A800 SM80 + H20 SM90，单一 wheel） |

构建阶段使用上游的 `requirements/build/cuda.txt`、`requirements/build/rust.txt` 与 `use_existing_torch.py`；预编译 Rust frontend 后打 wheel。GitHub-hosted runner 内存约 16GB，dual-arch 构建将 `MAX_JOBS` / `CMAKE_BUILD_PARALLEL_LEVEL` 设为 2，并在 CUDA 容器内启用 swapfile，以降低 nvcc 峰值内存。

### 驱动与 CUDA 版本

| 项目 | 说明 |
| --- | --- |
| 本机 NVIDIA 驱动 | CUDA **12.8**（用户环境） |
| 编译 toolkit | **CUDA 12.8**（`nvidia/cuda:12.8.0-devel-ubuntu22.04`） |
| 运行时 PyTorch | **cu126 / CUDA 12.6** wheels |
| 不可用 | cu129 / CUDA 12.9（超过驱动 12.8） |

vLLM dual-arch wheel 在本仓库构建完成后，从 [Releases](../../releases) 或对应 Actions Artifact 下载。

### PyTorch cu126 官方 wheel 直链（linux x86_64, cp312）

当前 pin 集合对应的可点击下载链接：

| 包 | 版本 | 直链 |
| --- | --- | --- |
| torch | `2.13.0+cu126` | [torch-2.13.0+cu126-cp312-cp312-manylinux_2_28_x86_64.whl](https://download.pytorch.org/whl/cu126/torch-2.13.0%2Bcu126-cp312-cp312-manylinux_2_28_x86_64.whl) |
| torchvision | `0.28.0+cu126` | [torchvision-0.28.0+cu126-cp312-cp312-manylinux_2_28_x86_64.whl](https://download.pytorch.org/whl/cu126/torchvision-0.28.0%2Bcu126-cp312-cp312-manylinux_2_28_x86_64.whl) |
| torchaudio | `2.11.0+cu126` | [torchaudio-2.11.0+cu126-cp312-cp312-manylinux_2_28_x86_64.whl](https://download.pytorch.org/whl/cu126/torchaudio-2.11.0%2Bcu126-cp312-cp312-manylinux_2_28_x86_64.whl) |

说明：上游 vLLM `0.27.1` 将 `torchaudio` 固定为 `2.11.0`；PyTorch cu126 索引上无 `torchaudio-2.13.0+cu126`，因此运行时使用 `2.11.0+cu126`。

## 手动触发

1. 打开 [Actions → Build vLLM CUDA 12.6 SM80+SM90 (A800+H20)](../../actions/workflows/build-vllm-cu126.yml)
2. 选择 **Run workflow**
3. 输入（可选）：
   - `vllm_version`：默认 `0.27.1`；也可填 `latest` 或具体版本如 `0.27.0`
   - `create_release`：默认开启；成功后在本仓库创建同名 GitHub Release（tag `vX.Y.Z`）并挂上 wheel

也可用 GitHub CLI：

```bash
gh workflow run build-vllm-cu126.yml \
  -R yippp/vllm_build \
  -f vllm_version=0.27.1 \
  -f create_release=true
```

## 自动构建（轮询上游 release）

Workflow：[Watch upstream vLLM releases](../../actions/workflows/watch-vllm-release.yml)

- **触发**：`schedule` 每 6 小时（`0 */6 * * *`），以及 `workflow_dispatch`
- **逻辑**：
  1. 请求 `https://api.github.com/repos/vllm-project/vllm/releases/latest`
  2. 检查本仓库是否已有同名 Release（tag `vX.Y.Z`）
  3. 若不存在，调用 `build-vllm-cu126.yml` 构建该版本的 dual-arch wheel，并创建 GitHub Release

因此：上游一旦发布新版本，本仓库在下一次轮询窗口内会构建一份同时覆盖 A800 与 H20 的 wheel；已发布过的版本不会重复构建。

也可手动跑 watcher：

```bash
gh workflow run watch-vllm-release.yml -R yippp/vllm_build
```

## 下载产物

1. **Actions Artifact**（保留 30 天）  
   打开对应 workflow run → Artifacts →  
   `vllm-<version>-torch2.13.0-cu126-cp312-sm80-sm90`

2. **GitHub Releases**（长期保留，自动构建默认启用）  
   [Releases](../../releases) 中按 vLLM 版本 tag（如 `v0.27.1`）下载 `.whl`  
   文件名通常仍为 `vllm-0.27.1-cp312-cp312-linux_x86_64.whl`；同一 tag 重建会覆盖该资源。

## 安装示例（A800 / H20）

```bash
pip install torch==2.13.0 torchvision==0.28.0 torchaudio==2.11.0 \
  --index-url https://download.pytorch.org/whl/cu126

pip install /path/to/vllm-*.whl
# 或按上游 runtime 依赖安装其余包后再装本仓库 wheel
```

也可直接使用上文「PyTorch cu126 官方 wheel 直链」中的 URL 安装。

## Workflow 文件

| 文件 | 作用 |
| --- | --- |
| `.github/workflows/build-vllm-cu126.yml` | 构建 SM80+SM90 单一 wheel；支持手动与 `workflow_call` |
| `.github/workflows/watch-vllm-release.yml` | 轮询上游 latest release 并触发 dual-arch 构建 |

校验步骤只检查已安装包的元数据版本（允许 `0.27.1+cu126` 这类 local version），**不** `import vllm`，避免在无 GPU 的 runner 上加载 CUDA kernels。
