# 数据格式与完整性

## 与录题 skill 的关系

复用现有题目录入 skill 的四种领域对象，以及服务端契约中的 `schemas.submission` 与 `schemas[kind]`。不另造“取题版”题面格式，不把题干展平成字符串，不剥离答案与解析。线上固定契约为准；仓库中的定义位于 `packages/question-content-contracts/src/question_content_contracts/protocol.py`。

`questions.json` 顶层恰好使用以下已有字段：

| 字段 | 获取时的值 |
| --- | --- |
| `exam` | `ielts` 或 `toefl` |
| `contract_version` | 本次固定并保存的真实契约版本 |
| `objects` | 去重后的 `{id, kind, data}` 数组；依赖大题在前，根试卷在后 |
| `source_manifest` | 每个对象的来源记录，使用现有 `object_id/namespace/source_id/source_sha256/locator` 字段 |

每条详情只投影 `{id, kind, data}`，保持内层 `data` 不变。`status`、`revision`、`created_at`、`updated_at` 属于读取元数据，放入 `fetch-report.json`，不混进上传的领域 `data` 或顶层 envelope。`taskType` 保持对外字段名，不能改成数据库的 `task_type`。

来源 `namespace` 可用 `seedline:机构站点的hostname`，`source_id` 使用真实对象 ID，`locator` 使用同站点内容详情相对路径；所有来源定位都不含秘密。保留来源快照时可填其实际摘要，未计算不伪造。请求追踪 ID、下载时间和根对象列表只写报告。

输出是已有对象的只读快照。格式一致不意味着可以直接提交新建：当前提交接口是 create-only，且新建对象要求 UUIDv7。读取时不得重编号旧对象来迎合新建校验；若旧 ID 或旧内容不符合固定契约，原样保留并记录兼容问题，不能称为可直接录入。再次录入由录题 skill 按用户另行授权处理 ID 映射、目标媒体、校验与发布确认。

## 四类领域数据

| kind | 顶层领域字段 | 保留要点 |
| --- | --- | --- |
| `ielts_question` | `skill/title/difficulty/tags/source/content` | 全部 materials、groups、items、答案、解析、译文与适用的参考音频 |
| `ielts_paper` | `title/skill/source/content` | `content.parts` 原顺序及 `question_id/label/first_number`；适用 book/test |
| `toefl_item` | `title/section/taskType/difficulty/content/solution` | 完整 materials、stimulus、subItems；`answersByKey/explanationsByKey` 覆盖全部小题 |
| `toefl_form` | `title/modules/rules/scoring` | 全部模块引用、显式规则与评分配置，包括合法的 0 值 |

以固定契约为准，不把此表当成第二份独立 Schema。原文定位依赖稳定句子 ID；不得重排生成 ID、丢弃证据引用、修改音频时间轴或 UTF-16 高亮偏移。保留契约允许的“字段未提供”与“显式空值”区别，不全局删除 null 或填默认值。

## 单题与整卷展开

单题模式的 `objects` 只有一个完整 `ielts_question` 或 `toefl_item`。题目内部的小题即使能单独展示，也不能拆成新的原子对象。

雅思单科卷：遍历 `paper.data.content.parts[].question_id`，获取全部 `ielts_question`。依赖须同科且已发布；保留分部顺序、label 与 first_number。按契约校验阅读／听力分部及连续计分位、写作任务和口语结构，不通过复制或补造题包满足完整卷规则。

托福整卷：逐一遍历以下八个平面 key 的所有 ID，而不是只跟随实际作答路径：

```text
reading.router
reading.lower
reading.upper
listening.router
listening.lower
listening.upper
writing.linear
speaking.linear
```

`objects` 对重复引用去重，但 `modules` 原始数组及顺序不改。候选分支中的共享 ID 只下载一次；不能把引用去重误用到卷结构。依赖须为当前可用题组；配额、顺序、每条路径内的重复约束用固定契约校验。

整卷读完后再次读取根卷，比较适用 revision、updated_at 和领域内容；根结构变化则重建该卷快照。现有分次 GET 不提供数据库事务快照，不宣称整卷来自同一时刻；报告保存各对象版本／读取时间，发现依赖变化或关系不一致则重取相关内容。

## 媒体与报告

题目 payload 中继续保存原媒体 ID。媒体文件另存 `media/`，映射放入报告，不能把领域 `file` 字段改成本地路径或短期 URL。相同媒体 ID 只获取一次。

`fetch-report.json` 是本技能的本地获取记录，不属于服务端题目 Schema，建议记录：

- 站点、正式身份契约确认的机构 ID、考试、固定契约版本、获取起止时间。
- 根对象列表、每个对象的 kind、真实 ID、status、revision、updated_at 与读取时间。
- `state` 为 `complete`、`partial` 或 `pending_validation`，以及具体缺失对象／媒体和校验问题。
- 分开的大题数、小题数、试卷数；托福整卷的统计口径为全部候选题组。
- 每个媒体 ID 的用途、类型、本地相对路径、实际大小、本地 SHA-256 和下载结果；仅 JSON 模式明确记录未下载。

不能把已取得部分对象写成完整整卷。允许交付部分 `questions.json`，但报告必须标明 `partial`；失败详情只保留安全的错误码和说明，不含签名、短期 URL 或原始私密配置。

## 校验与交接

保存实际 `contract.json`，使用其 submission Schema 和对应 kind Schema；进一步复用同版本 `question_content_contracts.validation` 的领域校验。仓库有可用运行环境时可直接调用共享函数，输入已获取对象与媒体元数据；不因此启动 ImportWorkflow 或请求写权限。

检查根对象、所有依赖的实际可用状态和机构读取结果，再执行共享校验。新建校验器会按创建语义处理对象状态，不能把它通过当作服务端依赖状态与权限的证明。实际媒体字段依据固定契约及现有共享媒体解析逻辑识别，不把文本中的任意 UUID 当媒体。

没有兼容校验器时只能报告格式初检或尚待校验，不宣称完成全部录入校验。取题过程中不补写未知答案、不展开源文件缺失字段来掩盖不兼容，也不降低校验规则。

用户要求预览时可以复用同版本纯渲染组件，但不能为了预览调用需要管理员身份的录题 prepare 工作流；缺少只读渲染入口时交付数据并如实说明。本版不承诺自动 HTML 预览。
