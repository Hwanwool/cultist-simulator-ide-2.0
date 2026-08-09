---
name: cultsim-schema-edit
description: 编辑 Cultist Simulator IDE 扩展 (tiinusen.cultsim) 的 JSON 自动补全字段定义。当用户要求给 elements/recipes/decks/legacies/endings/verbs 或共享类型添加、修改、删除补全字段（键值对），指定键名、值类型、插入位置时使用。修改的是 json-schema/ 目录下的 JSON Schema 文件。
---

# Cultist Simulator 补全字段编辑

## 背景

本扩展的 JSON 自动补全**由 `json-schema/` 目录下的 JSON Schema (draft-07) 驱动**，经 `package.json` 的 `contributes.jsonValidation` 关联到 `content/**/*.json`。增删改补全字段 = 编辑对应 schema 文件，**无需改动任何 TS/JS 代码**。

## 目标文件定位

根据用户提到的类目确定目标文件（路径相对于工程根目录）：

| 类目 | 文件 | 字段插入点 |
|---|---|---|
| elements | `json-schema/schemas/elements` | `items.oneOf` 有 **Card / Aspect 两个分支**，必须先确认加到哪个分支 |
| recipes | `json-schema/schemas/recipes` | `items.properties` |
| decks | `json-schema/schemas/decks` | `items.properties` |
| legacies | `json-schema/schemas/legacies` | `items.properties` |
| endings | `json-schema/schemas/endings` | `items.properties` |
| verbs | `json-schema/schemas/verbs` | `items.properties` |
| 共享类型 slot / expulsion / xtriggers-card / xtriggers-aspect | `json-schema/schemas/types/<同名文件>` | 顶层 `properties` |

类目未指明或含糊时，先读 `json-schema/files` 确认六类，再向用户确认。

## 值类型 → Schema 片段映射

| 用户描述 | 生成的 Schema 片段 |
|---|---|
| string / 字符串 | `"type": "string"` |
| number / 数字 | `"type": "number"` |
| integer / 整数 | `"type": "integer"` |
| boolean / 布尔 | `"type": "boolean"` |
| 字符串数组 / 元素列表 | `"type": "array", "additionalItems": false, "items": {"type": "string", "title": "element"}` |
| 整数数组 | `"type": "array", "additionalItems": false, "items": {"type": "integer"}` |
| 对象（含子字段） | `"type": "object", "additionalProperties": false, "properties": {…}` — 子字段需用户逐个提供，未提供则追问 |
| dictionary / 字典（任意键） | `"type": "object", "additionalProperties": false, "patternProperties": {".*": {"type": "integer", "title": "Aspect/Element: Amount"}}` — 字典值类型默认 integer，可按用户要求改为 string 等 |
| enum / 枚举 | `"type": "string", "oneOf": [{"const": "A"}, {"const": "B"}]` — 每个 const 可带 description；若某值另有含义，用 `{"const": "A", "description": "..."}` |
| 引用共享类型 | `"$ref": "./types/slot"` 等（相对路径从 `json-schema/schemas/` 出发） |
| 引用同文件定义 | `"$ref": "#/definitions/xxx"` — 若 `definitions` 中不存在该定义，创建之或询问用户 |

值类型描述含糊时（如只说"列表"），用 AskUserQuestion 或直接询问确认元素类型与结构。

## 插入位置（相对现有字段）

properties 的键顺序 = 补全候选显示顺序，插入位置影响补全顺序：

- 未指定位置 → **追加到 properties 末尾**
- "在 X 之后 / 之前" → 在文件中定位 `"X"` 键所在行，紧邻其后/前插入（键 X 必须真实存在于目标分支的 properties 中，否则询问）
- "第一个字段" / "开头" → properties 第一个键之前
- "末尾" → 最后一个键之后
- 位置无法解析 → 询问用户

## 添加字段工作流

1. **定位**：按类目定位文件与插入点。elements 必须确认加到 Card 还是 Aspect 分支；未指明时询问。
2. **键名**：默认 camelCase。若工程已存在同语义字段的大小写变体（如 `actionId`/`actionid`），询问用户是否同时生成变体。
3. **值类型**：按映射表生成片段；多字段一起添加时逐个确认。
4. **description**：用户提供则写入；未提供时写入空描述占位并提醒用户补充。
5. **插入**：按位置插入，缩进与相邻字段一致（4 空格），保持 JSON 逗号正确。
6. **Merge-Overwrite 变体**：仅当字段是数组/字典且用户提到"合并覆盖原版内容"时，询问是否生成 `$append`/`$prepend`/`$remove`（数组）或 `$add`/`$remove`（字典）变体。默认不生成。
7. **验证**：见下方验证节。

## 修改字段工作流

1. 定位字段所在文件与分支（elements 需确认分支）。
2. 确认修改点：值类型 / description / 位置（用"移动到 X 之后"表述）/ 键名（改名需确认无引用方）。
3. 修改后验证；若该字段被其他文件 `$ref` 引用（共享类型、definitions 中的定义），检查引用方是否受影响，受影响时向用户说明。

## 删除字段工作流

1. 定位并删除该键及其值。
2. **引用检查**：搜索其他 schema 文件是否 `$ref` 引用它；被引用时询问用户是否连带处理（删除定义或保留）。
3. 删除后验证。

## 验证（每个操作后必做）

1. 用 GetProblems 检查被编辑文件无语法错误。
2. 命令行 JSON 解析校验（在工程根目录执行）：
   ```
   node -e "JSON.parse(require('fs').readFileSync('json-schema/schemas/<目标文件>','utf8'))"
   ```
   输出无异常即合法。
3. 向用户汇报：改动文件、插入/修改/删除的字段名、值类型、所处位置。

## 格式约定

- 缩进 4 空格；逗号、引号风格与文件现有内容一致。
- 每个字段必须有 `description`；未研究透的字段用 `"$comment": "@TODO: ..."` 标注。
- 数值字段按项目惯例加 `minimum`/`maximum` 约束；枚举字段用 `oneOf`+`const` 风格（勿用 `enum` 数组，项目不使用）。
- 不要改动 `json-schema/files` 根文件的六个顶层键，除非用户明确要求新增顶层类目。
- 不要破坏既有结构：elements 的 `oneOf` 分支、recipes 的 `definitions.dictionary`、types 共享文件的 `patternProperties`。
- 项目格式细节（Merge-Overwrite 约定、大小写混用字段、共享类型引用关系）见 [reference.md](reference.md)。
- 对话示例见 [examples.md](examples.md)。
