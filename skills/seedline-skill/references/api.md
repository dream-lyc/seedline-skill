# 只读接口与调用方式

先满足 [机构接入条件](institution-access.md)。下表是仓库中已存在的接口形状；机构权限支持尚未完成，不能用表中路径绕过该限制。正式机构版本若调整路径或响应字段，以实际获准契约为准，并同步维护本 skill。

| 方法与路径 | 用途 | scope |
| --- | --- | --- |
| `GET /api/integration/v1/me` | 验证本次机构身份、环境和有效权限 | 有效机器凭据 |
| `GET /api/integration/v1/contracts/{exam}/latest` | 固定 IELTS／TOEFL 契约 | `contracts:read` |
| `GET /api/integration/v1/contracts/{exam}/versions/{version}` | 恢复已固定版本 | `contracts:read` |
| `GET /api/integration/v1/content/{kind}` | 分页概要检索 | `content:read` |
| `GET /api/integration/v1/content/{kind}/{id}` | 完整题目／试卷详情 | `content:read` |
| `GET /api/integration/v1/media/{media_id}` | 引用媒体的元数据 | `media:read` |
| `GET /api/integration/v1/media/{media_id}/access-url` | 获取短期只读下载地址 | `media:read` |

`exam` 为 `ielts` 或 `toefl`；`kind` 为 `ielts_question`、`ielts_paper`、`toefl_item`、`toefl_form`。

## 请求与响应

列表仅使用已支持的 `page`、`page_size`、`q`、`status`。默认第 1 页、20 条，最大 100 条。`q` 当前是标题子串匹配，不支持假定的全文、题号、科目或 book/test 查询参数。需要精准筛选时查看候选概要／详情，不自行扩展参数。

默认状态：IELTS 题包与试卷、TOEFL 试卷为 `published`，TOEFL 题组为 `available`。请求的状态过滤不能替代服务端机构授权；机构接口必须自己排除草稿、下架及范围外内容。

成功包络为 `{"request_id": "…", "data": …}`；失败为 `{"request_id": "…", "error": {"code": "…", "message": "…", "retryable": false}}`。列表 `data` 包含 `total/page/page_size/records`；详情 `data` 包含 `id/kind/status/data/revision/created_at/updated_at`。明确区分包络 `data` 与对象内层领域 `data`，不能多包或少包一层。

## 复用传输

从已安装技能清单定位现有题目录入 skill 的实际目录，不按品牌名猜测目录。可复用其 `scripts/request.sh` 与相邻 `signing.sh`，只调用 GET，并显式传入机构专用 `--profile`。这仅复用签名与 curl 传输，不执行该 skill 的管理员接入、录入或发布流程。

技术执行示例（路径变量须从实际安装位置与用户已有安全配置确定，不能照抄不存在的文件）：

```sh
bash "$import_skill/scripts/request.sh" GET '/api/integration/v1/me' \
  --profile "$institution_profile" > "$work_dir/identity.private.json"
```

先检查命令退出状态，再解析结果；机构核对完成后仅保留报告所需身份信息。凭据配置结构沿用 `origin/access_key/secret_key`，文件归当前用户、权限 0600，不复制到任务交付目录。用独立配置，例如 `~/.seedline-skill/profile.json`，不能默认读录题管理员配置。

签名沿用 RFC 9421 的 `sig1/hmac-sha256`，覆盖实际 GET 方法与完整目标 URI（含编码后的查询串）；每次请求新 nonce 和短期签名。由现有脚本处理签名，不重新实现一套。查询参数先编码再签名，签名后不修改顺序。禁止输出秘密、签名头、Cookie、开启调试追踪或跟随跨站重定向转发认证。

没有既有签名传输时，先说明缺少调用工具；不编造本 skill 自带 `fetch`、`connect` 等命令。本版仅包含指引，没有可执行取题客户端。后续独立分发可复用同一签名实现并增加只读取题入口，不必复制整个录题运行时。

## 媒体

按题目契约定义的媒体引用取得元数据，仅处理当前大题／整卷依赖的媒体。媒体应为 `confirmed`，purpose、类型和大小符合契约；不使用猜测的 OSS key 或直接列举桶内容。

短期访问 URL 有效期不超过 300 秒。下载时不附带平台 AKSK、Cookie 或签名头；敏感 URL 不出现在命令行、日志或持久文件，使用私密内存或受限短期传输配置并在完成后清理。记录本地相对路径、实际字节大小和下载字节的 SHA-256。

`source_sha256` 可能是上传源文件摘要，而下载图片可能经过转换；没有明确相同字节语义时，不拿源摘要冒充下载完整性证明。记录本地摘要，并按响应提供的实际文件大小／摘要语义校验。失效 URL 可重新获取，不能沿用到下一轮交付。

现有题目详情不含完整的独立解析视频绑定列表。默认获取领域 payload 中引用的材料；用户要求全部解析视频时，必须有正式绑定查询接口，否则列出能力缺口，不能宣称已获取全部视频。

## 失败处理

| 情况 | 处理 |
| --- | --- |
| 401／凭据失效 | 停止取题，按正式机构授权入口恢复，不借用其他账号 |
| 403 | 说明机构或凭据无权访问；不换管理员、不试探其他 ID |
| 404 | 说明对象不存在或不可访问；整卷中标记依赖缺失 |
| 429／网络超时／5xx | GET 可有限重试，最多额外 2 次，每次新签名；尊重服务端等待提示，无期限轮询不可用 |
| 契约／校验器不兼容 | 保留原始数据并标为待校验，不改字段或答案来强行通过 |

达到重试上限就保存进度并说明未完成项。服务端要求长时间等待时保存可恢复信息，不持续占用会话。无论 `retryable` 值如何，认证与授权拒绝都不扩大权限重试。
