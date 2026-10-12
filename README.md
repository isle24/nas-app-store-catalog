# NAS 应用商店目录

通过声明式 JSON 清单上架 Docker 应用。商店后端统一管理 Docker，容器引用固定权限模板；新增应用只增加数据，无需逐应用修改商店代码。

本仓库维护应用清单与协议。应用软件从其原发布者的镜像仓库或经核验的作者交付获取，应用许可证记录在各自清单中。

## 当前内容

- `schemas/app-manifest.schema.json`：应用清单，支持单/多容器、依赖、TCP/UDP 端口、挂载、环境参数、健康检查和生命周期声明。
- `schemas/catalog.schema.json`：带递增 sequence 的版本目录。
- `schemas/catalog-envelope.schema.json`：目录原始字节的 SHA256、长度与 Ed25519 签名信封。
- `schemas/openapi.json`：商店通用后端 `/api/store/v3` 的接口契约；正常管理员鉴权，原 v2 接口保持兼容。
- `drafts/datatotal/`：固定 registry digest、两个 host listener 和管理员口令参数的清单草案。
- `drafts/nasreader/`：固定历史服务器归档身份、只读书库与原生应用授权的清单草案。

**当前是协议与清单草案仓库。通用后端正在实现，签名目录尚未发布；drafts 不能被当作可安装应用源。** 本仓库没有上传 nasReader 服务器归档、用户资料或发布私钥。

## 后续上架流程

1. 复制清单模板，填写 appId、交付 ID、递增 releaseNumber、名称和版本信息。
2. 声明各服务镜像、端口、逻辑卷、依赖、安装参数和固定权限模板；镜像需锁定准确平台 digest，不能以移动 tag 作为安装身份。
3. 运行通用发布工具进行结构、引用、权限和软件身份校验，并生成目录/资源校验和。
4. 离线签名目录并发布静态文件；NAS 商店验证后刷新目录。

自定义图标可使用内容寻址素材，也可直接用清单的 monogram 图标。版本发布后不覆盖；配置或软件变更新增交付。下架只阻止新安装，已有实例与数据保留。

固定模板包括 `standard-v1`、`restricted-v1`、`host-monitor-v1`。选择模板不能自行扩权：宿主监控仍需要后端的明确策略授予。发布私钥不进入本仓库；秘密参数不写进清单、图标或版本目录。
Docker application manifests and publishing protocol for the NAS app store
