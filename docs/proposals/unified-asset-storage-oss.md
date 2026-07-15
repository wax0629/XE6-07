# 提案：统一资源存储链路（OSS）

## 动机 / 用户故事

用户上传的参考图和已有模型需要被稳定保存；后台生成的概念图、模型和每次修复产生的新版本也需要统一落盘，供预览、检查和切片读取。

共同需求是：无论资源来自上传还是生成，都通过同一条存储链路进入 OSS，并返回不受物理路径变化影响的稳定引用。

## 目标用户

- 上传图片或模型的普通用户
- 生成、预览、检查和切片过程中读写资源的后台服务

## 现有做法及不足

如果上传资源和生成资源分别使用不同的落盘、路径和引用机制，业务对象会依赖具体目录或 OSS Key，资源迁移时需要同步修改大量引用，也容易出现上传可读但生成结果不可复用等分叉。

## 本期范围

本期做：

- 定义统一 `Asset` 对象。
- 客户端上传使用“申请上传 - OSS 直传 - 完成确认”的两步链路，业务服务器不转发文件字节。
- 后台生成服务使用 `putAsset(bytes, meta) -> Asset`；两种入口最终都通过同一 finalize 逻辑创建 `Asset`。
- 将图片和模型资源存入 OSS。
- 使用不可变 `assetId` 作为业务引用。
- `ossKey` 作为可迁移映射保存，不由 `assetId` 固定派生。
- `Asset` 记录 owner、visibility 和内容哈希；读取和下载必须经过访问策略校验。
- `AssetRevision` 通过 `modelAssetId` 和 `previewAssetId` 引用资源。

本期明确不做：

- 不做复杂共享权限矩阵、配额和生命周期回收；但本期必须具备 owner、visibility 和受控读取的最小授权边界。
- 不做内容去重。
- 不做版本树逻辑。
- 不做社区发布和下载。

## 关键决策与依据

### 备选方案

1. 上传资源和生成资源分别维护存储入口
   - 优点：可以针对来源分别实现。
   - 缺点：落盘、编址和读取逻辑重复，业务层需要理解资源来源。
2. 业务对象直接保存 OSS Key 或物理路径
   - 优点：读取路径直观。
   - 缺点：路径迁移会传播到所有业务对象，引用不稳定。
3. 使用统一入口和不可变 `assetId`，用 `source` 区分来源
   - 优点：上传与生成共享同一条链路，路径可以迁移而业务引用保持稳定。
   - 缺点：需要维护 `assetId` 到 OSS Key 的解析关系。

### 本期选择

选择方案 3。

原因：上传与生成的差异只在字节来源，落盘和编址应保持一致；大文件静态资源适合对象存储，稳定 `assetId` 可以隔离后续路径迁移。

## 基本概念与信息结构

### Asset

```text
Asset {
  id
  ownerId
  visibility # private | public
  kind       # image | model
  source     # upload | generated
  ext        # png | glb | stl | ...
  ossKey     # 存储层生成并持久化的可迁移映射
  contentHash
  createdAt
}
```

### 统一写入口

```text
beginUpload({ ownerId, visibility, kind, source, ext, contentHash })
  -> { uploadId, signedUrl, expiresAt }

completeUpload(uploadId, contentHash) -> Asset

putAsset(bytes, { ownerId, visibility, kind, source, ext, contentHash }) -> Asset

getAsset(assetId, principal) -> authorized Asset reference
```

客户端上传使用 `beginUpload` 与 `completeUpload`；后台生成服务可使用 `putAsset`，其落盘校验和 Asset 创建语义与 `completeUpload` 相同。业务对象只保存 `Asset.id`，不保存服务器路径。`ossKey` 由存储层生成（例如使用随机对象标识），并作为 `assetId` 的映射字段持久化。

## 原型 / 演示

本期为存储契约，无独立界面。上传入口、三维预览和版本切换是该能力的消费端。

现有原型：https://marycomplex.github.io/3d-prototype/

## 验收标准

### 例子 1：保存上传图片

- 现状：上传图片可能使用独立路径，后续服务无法获得统一引用。
- 提议后的行为：客户端申请限时直传地址，把图片直接写入 OSS，再由服务端确认对象和哈希后创建 `Asset`。
- 验收：调用 `beginUpload` 获得限时 `signedUrl`，直传成功后以同一 `uploadId` 和 `contentHash` 调用 `completeUpload`；系统返回不可变 `assetId`，保存 owner、visibility、内容哈希和实际 `ossKey`，业务服务器未接收文件字节。

### 例子 2：保存生成模型

- 现状：生成模型与上传资源可能使用不同的落盘和引用方式。
- 提议后的行为：生成模型使用同一入口，`source` 标记为 `generated`。
- 验收：后台调用 `putAsset` 后返回的 `Asset.id` 可写入 `AssetRevision.modelAssetId`，预览、检查和切片在通过访问校验后按该 id 读取同一模型。

### 例子 3：迁移 OSS Key

- 现状：业务对象直接持有路径时，资源迁移会导致历史引用失效。
- 提议后的行为：业务对象只持有 `assetId`，解析层从持久化映射读取当前 `ossKey`。
- 验收：迁移对象并更新 Asset 记录中的 `ossKey` 后，所有持有旧 `assetId` 的业务对象无需修改，仍能读取资源；`assetId` 与新 `ossKey` 不要求存在可推导关系。

### 例子 4：非法资源

- 现状：空字节、非法 kind 或扩展名与资源类型不匹配可能产生脏资源。
- 提议后的行为：系统在写入前拒绝非法元数据和空内容。
- 验收：`kind` 非法、bytes 为空或 `ext` 与 `kind` 不匹配时返回明确错误，不创建 Asset，也不在 OSS 留下对象。

### 例子 5：跨用户读取被拒绝

- 现状：若 `assetId` 被当作读取凭证，泄露或猜中 id 的用户可能访问他人的私有模型。
- 提议后的行为：预览、下载和后台读取均根据 principal、owner 和 visibility 执行访问校验，签名地址短期有效且只在校验通过后签发。
- 验收：用户 A 请求用户 B 的 private Asset 时返回 403/404，且不返回 OSS Key 或签名地址；同一 Asset 变为 public 后可按公开策略读取。
