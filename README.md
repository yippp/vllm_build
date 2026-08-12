# vllm_build

为 **A800 / A100（SM80 / Ampere）** 构建精简的 [vLLM](https://github.com/vllm-project/vllm) CUDA wheel。只编译 `TORCH_CUDA_ARCH_LIST=8.0`，不构建其他 GPU 架构。

当前默认目标：**vLLM 0.27.1**，对齐上游官方 `cu129` 发布线。

## 构建固定版本（0.27.x）

| 项目 | 值 |
| --- | --- |
| vLLM | `0.27.1`（可改） |
| PyTorch | `2.13.0+cu129` |
| torchvision（运行时建议） | `0.28.0+cu129` |
| Triton（运行时建议） | `3.7.1` |
| CUDA 构建容器 | `nvidia/cuda:12.9.0-devel-ubuntu22.04` |
| Python | `3.12` |
| GPU arch | `8.0`（SM80） |

构建阶段使用上游的 `requirements/build/cuda.txt`、`requirements/build/rust.txt` 与 `use_existing_torch.py`；预编译 Rust frontend 后打 wheel。

## 手动触发

1. 打开 [Actions → Build vLLM CUDA 12.9 SM80 (A800)](../../actions/workflows/build-vllm-cu129.yml)
2. 选择 **Run workflow**
3. 输入（可选）：
   - `vllm_version`：默认 `0.27.1`；也可填 `latest` 或具体版本如 `0.27.0`
   - `create_release`：默认开启；成功后在本仓库创建同名 GitHub Release（tag `vX.Y.Z`）并挂上 wheel

也可用 GitHub CLI：

```bash
gh workflow run build-vllm-cu129.yml \
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
  3. 若不存在，调用 `build-vllm-cu129.yml` 构建该版本，并创建 GitHub Release

因此：上游一旦发布新版本，本仓库在下一次轮询窗口内会构建一次；已发布过的版本不会重复构建。

也可手动跑 watcher：

```bash
gh workflow run watch-vllm-release.yml -R yippp/vllm_build
```

## 下载产物

1. **Actions Artifact**（保留 30 天）  
   打开对应 workflow run → Artifacts →  
   `vllm-<version>-torch2.13.0-cu129-cp312-sm80`

2. **GitHub Releases**（长期保留，自动构建默认启用）  
   [Releases](../../releases) 中按 vLLM 版本 tag（如 `v0.27.1`）下载 `.whl`

## 安装示例（A800）

```bash
pip install torch==2.13.0 torchvision==0.28.0 \
  --index-url https://download.pytorch.org/whl/cu129

pip install /path/to/vllm-*.whl
# 或按上游 runtime 依赖安装其余包后再装本仓库 wheel
```

## Workflow 文件

| 文件 | 作用 |
| --- | --- |
| `.github/workflows/build-vllm-cu129.yml` | 构建 SM80 wheel；支持手动与 `workflow_call` |
| `.github/workflows/watch-vllm-release.yml` | 轮询上游 latest release 并触发构建 |

校验步骤只检查已安装包的元数据版本（允许 `0.27.1+cu129` 这类 local version），**不** `import vllm`，避免在无 GPU 的 runner 上加载 CUDA kernels。
