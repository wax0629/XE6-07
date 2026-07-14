# 提案：Hunyuan3D / ComfyUI 服务器目录结构整理

## 1. 整理结果

完整 ComfyUI、Hunyuan3D 主开发源码、模型、缓存、工作流和运行数据已经从 `/root` 及零散目录归入 `/srv/3d-printing-ai`。所有仍可能被旧脚本使用的路径均保留兼容软链接。

整理过程中没有重新下载、复制或替换大模型。两个主要 Comfy 模型迁移前后的设备号、inode 和文件大小完全一致，表明本次只是同一文件系统内移动：

| 模型 | 大小 | inode |
|---|---:|---:|
| `hunyuan_3d_v2.1.safetensors` | 7,365,943,290 字节 | 298170 |
| `hunyuan3d-dit-v2-0-mv-fast-fp16.safetensors` | 4,930,777,530 字节 | 299059 |

磁盘使用率仍为 76%，新增空间主要是约 203 MB 的结构迁移备份。

## 2. 当前标准目录

```text
/srv/3d-printing-ai/
├── platform/                         # 后续业务开发目录
│   ├── api/                          # FastAPI，当前预留
│   ├── comfy/
│   │   ├── workflows/
│   │   │   ├── source/               # ComfyUI 可编辑源工作流
│   │   │   ├── api/                  # 精简 API 工作流，当前预留
│   │   │   └── bindings/             # API 与节点映射，当前预留
│   │   └── custom_nodes/             # 业务自定义节点，当前预留
│   ├── config/
│   ├── deploy/
│   │   ├── systemd/
│   │   └── nginx/
│   ├── docs/
│   ├── scripts/
│   └── tests/
│
├── vendor/                           # 固定版本第三方源码
│   ├── ComfyUI/
│   └── Hunyuan3D-2.1/
│
├── models/                           # 模型权威存储位置
│   ├── comfy-hunyuan/
│   ├── hunyuan3d-2.1/
│   ├── stableDiffusion/
│   └── trellis2/
│
├── cache/
│   ├── huggingface/
│   └── hunyuan-runtime/
│
├── runtime/
│   ├── comfy/
│   │   ├── input/
│   │   ├── output/
│   │   ├── temp/
│   │   └── user/
│   ├── jobs/                         # 后续 API 任务目录
│   ├── db/
│   └── legacy-tmp/
│
├── logs/
├── legacy/
│   └── 2026-07-14-pre-structure/
├── outputs/                          # 现有团队输出；因 sidecar 正在使用而保留
├── docker/                           # 现有 sidecar；因容器正在使用而保留
├── repos/                            # 其他历史仓库及兼容链接
├── scripts/
└── venvs/                            # 后续独立 Python 环境目录
```

## 3. 路径变更

| 原路径 | 新位置 | 兼容方式 |
|---|---|---|
| `/root/comfy/ComfyUI` | `/srv/3d-printing-ai/vendor/ComfyUI` | 原路径保留软链接 |
| `ComfyUI/models` | `/srv/3d-printing-ai/models/comfy-hunyuan` | Comfy 内部保留软链接 |
| `ComfyUI/input` | `/srv/3d-printing-ai/runtime/comfy/input` | Comfy 内部保留软链接 |
| `ComfyUI/output` | `/srv/3d-printing-ai/runtime/comfy/output` | Comfy 内部保留软链接 |
| `ComfyUI/temp` | `/srv/3d-printing-ai/runtime/comfy/temp` | Comfy 内部保留软链接 |
| `ComfyUI/user` | `/srv/3d-printing-ai/runtime/comfy/user` | Comfy 内部保留软链接 |
| `user/default/workflows` | `/srv/3d-printing-ai/platform/comfy/workflows/source` | 用户目录保留软链接 |
| `/srv/3d-printing-ai/repos/Hunyuan3D-2.1` | `/srv/3d-printing-ai/vendor/Hunyuan3D-2.1` | `repos` 下保留软链接 |
| `/root/hunyuan/Hunyuan3D-2.1_model_pth` | `/srv/3d-printing-ai/models/hunyuan3d-2.1` | 原路径保留软链接 |
| `/srv/3d-printing-ai/hf-cache` | `/srv/3d-printing-ai/cache/huggingface` | 原路径保留软链接 |
| `/srv/3d-printing-ai/runtime-cache` | `/srv/3d-printing-ai/cache/hunyuan-runtime` | 原路径保留软链接 |
| `/srv/3d-printing-ai/tmp` | `/srv/3d-printing-ai/runtime/legacy-tmp` | 原路径保留软链接 |

原 `/root/Hunyuan3D-2.1`、`/root/hunyuan/Hunyuan3D-2.1` 和 `/root/HY` 含有不同本地改动或不完整模型文件，没有覆盖合并，已移动到：

```text
/srv/3d-printing-ai/legacy/2026-07-14-pre-structure/
```

对应的 `/root` 原路径仍为软链接，因此旧命令暂时仍可使用。

## 4. 工作流位置

当前源工作流统一放在：

```text
/srv/3d-printing-ai/platform/comfy/workflows/source/
```

文件：

| 文件 | 节点 | 连线 | JSON 版本 |
|---|---:|---:|---:|
| `Hunyuan3D-2.1-Image-to-GLB.json` | 11 | 12 | 0.4 |
| `hy3d_multiview_example.json` | 81 | 95 | 0.4 |

ComfyUI 中原来的工作流路径已经链接到该目录，继续从 UI 保存时仍会写入统一位置。

## 5. 备份与本地修改保护

迁移前备份位于：

```text
/srv/3d-printing-ai/legacy/2026-07-14-pre-structure/backups/
```

包含：

- `comfy-config-workflows-custom-nodes.tgz`：Comfy 用户配置、工作流和自定义节点，约 203 MB。
- `comfy-sidecar-source.tgz`：当前 Hunyuan sidecar 源码。
- `hunyuan-srv-local-changes.patch`：主 Hunyuan3D-2.1 开发版本的四个纹理文件修改。
- `hunyuan-root-docker-local-changes.patch`：Docker 变体修改。
- `hunyuan-root-env-local-changes.patch`：Python 环境变体修改。
- `comfy-process-env.filtered`：迁移前 Comfy 进程的非敏感运行环境摘要。

两个压缩备份均已通过 `gzip -t` 完整性校验。

## 6. 运行验证

整理完成后已验证：

- 完整 ComfyUI 正常响应 `/system_stats`，版本为 0.27.0。
- PyTorch 为 2.6.0+cu124，8 张 RTX 4090 均可识别。
- `Hy3DGenerateMeshMultiView` 节点正常加载。
- `Hy3DExportMesh` 节点正常加载且仍为输出节点。
- ComfyUI 当前队列为空。
- 两份工作流均能正常解析。
- 旧路径与新路径均可访问相同模型文件。
- `comfyui-x-dev` sidecar 仍运行在 `127.0.0.1:8189`。
- Zero123++ 仍运行在 7868。
- SDXL 服务仍运行在 7860。

本次未执行新的模型生成任务，避免在纯目录迁移阶段引入额外 GPU 工作和新产物。

## 7. 有意保留的现状

以下问题属于下一阶段服务/API 改造，不在本次目录整理中修改：

1. 完整 ComfyUI 仍以 root 用户从 SSH 会话人工启动。
2. 完整 ComfyUI 仍监听 `0.0.0.0:8188`，尚未收口到本机地址。
3. `comfyui-x-dev` sidecar 仍保留，尚未退役。
4. `/srv/3d-printing-ai/outputs` 和 `/srv/3d-printing-ai/docker` 仍被活动容器使用，因此没有移动。
5. Stable Diffusion 的代码和日志仍混在 `models/stableDiffusion`，因进程正在运行而未处理。
6. Python 环境仍位于 `/root/miniconda3/envs/comfy_hunyuan`，尚未迁移到 `venvs`。
7. API 工作流、自定义 Artifact Output 节点、FastAPI、systemd 和 Nginx 尚未实施。

## 8. 后续开发规则

- 业务代码放入 `platform`，不要继续写入 `/root`。
- 第三方固定版本源码放入 `vendor`。
- 模型只放入 `models`。
- Hugging Face、Torch 和运行缓存只放入 `cache`。
- API 任务输入输出使用 `runtime/jobs/<job_id>`。
- 可长期交付的产物放入 `outputs` 或后续对象存储。
- 临时试验文件放入 `runtime`，不得放到源码和模型目录。
- 新服务配置放入 `platform/deploy`，敏感环境变量最终放入 `/etc/hunyuan3d-platform`。

## 9. 下一阶段

下一阶段可基于当前结构依次实施：

1. 裁剪 geometry/textured 两份 API 工作流。
2. 修复 Front/Back/Left/Right 的独立输入绑定。
3. 增加可被 Comfy history 读取的 Artifact Output 节点。
4. 将 ComfyUI、任务 Worker 和 FastAPI 纳入 systemd，并通过 Nginx 仅暴露受控 API。
