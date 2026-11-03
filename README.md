# seedline skill

seedline 的统一 Agent 技能入口。第一期聚焦获取雅思／托福题目，后续再逐项增加其他能力。

**当前版本为设计预览版。** 已提供完整的技能指引、接口说明、数据格式及机构接入条件；机构只读取题服务端尚待实现与验收，也没有独立取题客户端。安装本 skill 不代表当前已经能通过机构账号获取题目。

## 第一期范围

- 通过机构账号获准的只读开放接口检索、获取题目。
- 以完整一道大题为最小单位，保留全部材料、小题、参考答案和解析。
- 获取整套试卷时展开全部依赖大题；托福包含全部候选分支，雅思沿用单科卷结构。
- 获取相关图片、音频和参考音频，保存数据与媒体映射。
- 沿用现有录题 skill 的 `exam / contract_version / objects / source_manifest` 格式。

题目上传、发布及其他能力不属于第一期，不会因安装本 skill 自动启用。

## 安装与使用

在 Codex 中发送：

```text
$skill-installer 请安装 https://github.com/dream-lyc/seedline-skill/tree/main/skills/seedline-skill
```

安装后在新对话中调用：

```text
使用 $seedline-skill，通过我的机构账号获取指定的一道雅思完整大题。
```

或者：

```text
使用 $seedline-skill，获取指定的托福整套试卷，包含全部候选分支与关联媒体。
```

技能会先检查正式机构接入条件。当前站点未开通时，应明确说明尚不可用，不能借用平台管理员凭据。

## 接入状态

当前服务网站为 [edu.learningiland.com](https://edu.learningiland.com)。截至本版设计核对，现有机器凭据和内容读取接口仍仅面向平台管理员。机构取题需要补齐机构凭据、当前机构身份、内容范围及关联媒体授权，并完成服务端验收。

本仓库不包含业务服务端实现、凭据、真实题目或学生数据。签名传输和兼容校验器的依赖及尚未具备的能力均在技能说明中列出。不要向 GitHub 提交 AKSK、Cookie、私有配置或题目导出数据。

## 文件说明

- [技能入口](skills/seedline-skill/SKILL.md)：第一期范围与执行流程。
- [只读接口](skills/seedline-skill/references/api.md)：现有接口形状、传输复用和失败处理。
- [数据格式](skills/seedline-skill/references/data-format.md)：完整大题与整卷展开、校验及交付格式。
- [机构接入条件](skills/seedline-skill/references/institution-access.md)：现有约束、待确认的服务端调整与验收场景。

技能名称保持为 `seedline-skill`，界面显示名称为 `seedline skill`。后续能力按实际需求加入，不把规划中的能力标为已实现。
