下面给你一份\*\*“拿来就能干”\*\*的 **ComfyUI × Flask** 多租户（用户分区）落地方案。它分四层：

1. **身份与通信** – 每个用户独立 WebSocket／HTTP 通道
2. **算力与文件** – ComfyUI 运行时与输出目录隔离
3. **数据资产** – 历史记录元数据落 MongoDB，原图进对象存储
4. **前端加载** – 摆脱全屏 `<iframe>`，按需懒加载 ESM 模块

整个方案都只在你现有的 *Flask + HTML/CSS/JS + MongoDB* 栈上做增量改动；涉及的 ComfyUI 代码位置我都标了注释，方便你直接开改。

---

## 1  身份与通信隔离

| 目标                   | 做法                                                                                                                                                                                  | 关键改动点                                                                                    |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **1.1 统一登录态**        | Flask 登录成功后签发 `access_token` (**长**，7 天)、`ws_token`(**短**，30 min)。                                                                                                                  | 新增 `/api/auth/login` & `/api/comfy/ws_token`                                             |
| **1.2 WebSocket 打标** | 前端加载 ComfyUI 时：<br>``js<br>const {ws_token}=await fetch('/api/comfy/ws_token').then(r=>r.json());<br>const ws = new WebSocket(`wss://comfy.yourdomain/ws?token=${ws_token}`);<br>`` | **server.py** `websocket_handler` 处读取 `token`，验证→解码出 `user_id`→保存到 `request['user_id']`  |
| **1.3 HTTP 路由打标**    | 所有 POST/GET 到 ComfyUI 的 `/prompt`、`/upload` 等接口都必须带 `Authorization: Bearer access_token`。                                                                                           | 在 **server.py** 里加 `@web.middleware` 做 JWT 校验，注入 `request['user_id']`                    |

---

## 2  运行时与文件隔离

### 2.1 按用户切换输出/临时目录

ComfyUI 默认把文件写到全局 `output/` 和 `temp/`；调整为 **每次收到新任务就临时改目录**：

```python
# 新 util：per_user_dirs.py
import folder_paths, os, contextlib

@contextlib.contextmanager
def user_fs(user_id: str):
    base = os.getenv("COMFY_DATA_ROOT", "/data/comfy")
    out = os.path.join(base, user_id, "output")
    tmp = os.path.join(base, user_id, "temp")
    os.makedirs(out, exist_ok=True)
    os.makedirs(tmp, exist_ok=True)
    old_out, old_tmp = folder_paths.get_output_directory(), folder_paths.get_temp_directory()
    folder_paths.set_output_directory(out)
    folder_paths.set_temp_directory(tmp)
    yield
    folder_paths.set_output_directory(old_out)
    folder_paths.set_temp_directory(old_tmp)
```

在 **execution.prompt\_worker** 里每次执行队列项时包一层：

```python
from per_user_dirs import user_fs
with user_fs(user_id):
    e.execute(...)
```

这样所有写盘节点（如 **SaveLatent** ）都会自动落到 `…/user_id/output/`，彼此完全隔离。

### 2.2 写完即推对象存储

在 `SaveLatent` / `SaveImage` 节点末尾追加钩子，把文件 `upload` 到 S3（或 MinIO）：

```python
if os.getenv("S3_ENDPOINT"):
    upload_to_s3(local_path, f"comfy/{user_id}/{ts}/{filename}")
```

元数据（prompt、参数、S3 key…）存进 `mongodb.generations`，字段参考下节。

---

## 3  业务数据模型（MongoDB）

```javascript
// users
{ _id:ObjectId, email, pwd_hash, ... }

// generations
{
  _id:ObjectId,
  user_id:ObjectId,           // 外键
  s3_key:"comfy/{uid}/2025/06/26/abc.png",
  thumb_key:"…webp",
  workflow_id:"hash-of-graph",
  prompt:"…",
  created_at:ISODate
}
```

### API 约定

| 接口                        | 作用                                                    |
| ------------------------- | ----------------------------------------------------- |
| `GET /api/history?page=n` | 查询 `generations` where `user_id==current`，分页返回缩略图 URL |
| `POST /api/upload/tmp`    | 临时素材上传，同样写 `comfy/{uid}/tmp/…`                        |

在 Toolbar 里调用即可。由于 **websocket\_handler** 已解析出 `user_id`，Flask 路由里用 `g.user_id` 就能过滤。

---

## 4  前端加载优化

你的 `create_workflow.html` 目前整页 iframe ，浏览器得 **再开一个渲染进程**，还无法共用 JS Bundle，故启动慢、占内存——换成 **懒加载 ESM 模块** 最简单：

```html
<!-- workflow 区域 -->
<div id="workflow-root"></div>
<button id="open-workflow">开始创作</button>

<script type="module">
document.getElementById('open-workflow').onclick = async ()=>{
  const [{ mountComfyUI }] = await Promise.all([
    import('https://cdn.yoursite.com/comfyui/esm/index.js'),
    fetch('/api/comfy/ws_token')
  ]);
  const { ws_token } = await r.json();
  mountComfyUI('#workflow-root',{ wsToken:ws_token });
};
</script>
```

> *如果你暂时不想改 ComfyUI 前端打包*，也可以保留 iframe，但用 **IntersectionObserver** 延迟插入元素，比直接 `src=` 好得多。

---

## 5  加速与弹性

| 层级        | 动作                                                                     | 备注                                               |
| --------- | ---------------------------------------------------------------------- | ------------------------------------------------ |
| **模型加载**  | ComfyUI Worker 启动时预热常用 checkpoint & LoRA；多 Worker 共享只读挂载，显存满自动 LRU 卸载。 | `comfy.model_management` 已有 LRU；配合 `--cache_lru` |
| **图片静态化** | 生成完即存 WebP，前端 srcset，自带缩略；Origin Shield CDN                            |                                                  |
| **水平扩容**  | 每个 `user_id` 任意 Worker 都能跑（逻辑隔离）；若想物理隔离“大客户”，按 namespace 调 HPA         | K8s 或 Docker Swarm                               |

---

## 6  里程碑排期

| 阶段     | 交付                                    | 工期  |
| ------ | ------------------------------------- | --- |
| **M0** | 登录 + JWT / ws\_token 流程打通，iframe 延迟加载 | 3 天 |
| **M1** | `user_fs()` 输出目录隔离 + S3 上传 + Mongo 记录 | 1 周 |
| **M2** | Toolbar 历史面板读取 Mongo + 预签名 S3 URL     | 3 天 |
| **M3** | 改造前端为 ESM 模块，去掉 iframe                | 1 周 |
| **M4** | 多 Worker / 自动伸缩 / 监控告警                | 1 周 |

---

### 一句话总结

> **先用 JWT + “一人一目录” 把身份和磁盘隔好，再把界面从 iframe 变模块，性能立刻飞起；后续再做对象存储、水平扩容就水到渠成。**

这样既不会推翻你现有 Flask/MongoDB 项目结构，也给后续多租户收费、算力弹性留下足够空间。祝项目顺利落地！
