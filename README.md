# vllm_build

> **2026-09-23 起：推荐改用官方 vLLM，不再自动自编译。**

本仓库曾为 **A800（SM80）** 与 **H20（SM90）** 自编 dual-arch CUDA wheel。用户驱动已升至 **CUDA 13.2** 后，官方预编译 wheel 已可用，**默认请直接装官方 cu130**。

## 推荐安装（官方 cu130）

官方 **v0.30.0** 没有 `cu132` wheel。驱动 13.2 ≥ 运行时 toolkit，可直接用：

| 变体 | 说明 | 安装 |
| --- | --- | --- |
| **cu130（推荐）** | PyPI 默认 / Release 无 `+cu` 后缀 | `pip install vllm==0.30.0` |
| cu129 | 显式 CUDA 12.9 | 见下方直链 |

```bash
# 推荐：官方 cu130
pip install vllm==0.30.0

# 或 uv（按驱动自动选 torch backend）
uv pip install vllm==0.30.0 --torch-backend=auto
```

### 官方 wheel 直链（linux x86_64）

| 变体 | 直链 |
| --- | --- |
| cu130（默认） | [vllm-0.30.0-cp38-abi3-manylinux_2_28_x86_64.whl](https://github.com/vllm-project/vllm/releases/download/v0.30.0/vllm-0.30.0-cp38-abi3-manylinux_2_28_x86_64.whl) |
| cu129 | [vllm-0.30.0+cu129-cp38-abi3-manylinux_2_28_x86_64.whl](https://github.com/vllm-project/vllm/releases/download/v0.30.0/vllm-0.30.0%2Bcu129-cp38-abi3-manylinux_2_28_x86_64.whl) |

上游说明见 [vLLM CUDA 安装文档](https://docs.vllm.ai/en/latest/getting_started/installation/gpu.html)。

## 自编译状态（legacy）

| 项目 | 状态 |
| --- | --- |
| `Watch upstream vLLM releases` | **已停用**：无 `schedule`；Actions 中已 `disabled_manually` |
| `build-vllm-cu126.yml` | 仍保留，**仅手动**；非推荐路径 |
| 历史 Release | `v0.27.1` / `v0.28.0` / `v0.29.0`（dual-arch SM80+SM90，torch cu126） |
| v0.30.0 自编 | 已失败（deepgemm 缺 `libdw-dev`），**不再修复** |

历史自编 pin（仅作归档说明）：torch `2.13.0+cu126`，编译容器 CUDA 12.8，`TORCH_CUDA_ARCH_LIST=8.0 9.0`，可选 cooperative_topk 关闭。

### 手动自编（不推荐）

仅在官方 wheel 不可用、或必须定制架构时：

```bash
gh workflow run build-vllm-cu126.yml \
  -R yippp/vllm_build \
  -f vllm_version=0.29.0 \
  -f create_release=true
```

## 历史产物下载

- [Releases](../../releases)：`v0.27.1` / `v0.28.0` / `v0.29.0`
- Artifact 命名形如 `vllm-<version>-torch2.13.0-cu126-cp312-sm80-sm90`

## Workflow 文件

| 文件 | 作用 |
| --- | --- |
| `.github/workflows/watch-vllm-release.yml` | 原上游轮询；**已去掉 schedule** |
| `.github/workflows/build-vllm-cu126.yml` | 遗留手动 dual-arch 构建 |