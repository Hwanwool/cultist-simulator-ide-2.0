# 项目格式细节参考

本文件记录 Cultist Simulator IDE (tiinusen.cultsim-1.1.1) 的 JSON Schema 补全定义格式细节，供编辑字段时保持一致。

## 六大类 schema 结构要点

| 文件 | 结构 |
|---|---|
| elements | `items.oneOf` 两个分支：**Card**（必填 `id`，`isAspect` const false）与 **Aspect**（必填 `id`+`isAspect`，const true）。Card 独有 `aspects`/`lifetime`/`resaturate`/`slots`/`unique`/`uniquenessgroup`/`burnTo`；Aspect 独有 `noartneeded`。共享 `definitions`：id/label/description/isAspect/icon/induces/decayTo/verbicon/aspects/lifetime/resaturate/comments |
| recipes | 单分支 `items.properties`，必填 `id`。字典类字段（requirements/tablereqs/extantreqs/effects/purge/aspects/deckeffects/haltverb/deleteverb）均 `$ref` 到 `#/definitions/dictionary`。`alt` 数组项引用 `./types/expulsion`；`slots` 数组项引用 `./types/slot` 并叠加 `greedy` |
| decks | 单分支，必填 `id`。`spec` 为字符串数组；`drawmessages` 为字典（值 string） |
| legacies | 单分支，必填 `id`。`effects` 为字典（值 integer ≥0）；`statusbarelements`/`excludesOnEnding` 为字符串数组 |
| endings | 单分支，必填 `id`+`achievement`。`flavour`/`anim` 为 `oneOf`+`const` 枚举 |
| verbs | 单分支，必填 `id`。`slot` 直接 `$ref: "./types/slot"` |

## Merge-Overwrite 约定

游戏支持用同 ID 合并覆盖原版内容，schema 为此生成配套变体字段（含变体时字段名以 `$` 分隔）：

- 数组字段：`field$append`（追加）、`field$prepend`（前置）、`field$remove`（移除，值为字符串数组）
- 字典字段：`field$add`（新增/覆盖键，结构与原字段相同）、`field$remove`（移除键，值为字符串数组）
- 示例：decks 的 `spec` / `spec$append` / `spec$prepend` / `spec$remove`；slot 的 `required` / `required$add` / `required$remove`

## 大小写混用字段（核心原版内容存在两种拼写）

- recipes：`actionId` 与 `actionid`（均引用 `#/definitions/actionId`）
- legacies：`startingVerbId` 与 `startingverbid`；`fromEnding` 与 `fromending`
- endings：`achievementid`（有定义无实例）与 `achievement`（有实例无定义）

新增字段时默认 camelCase；若用户意图是补全已知混用字段的另一拼写，按上表生成。

## 共享类型引用关系

| 类型文件 | 被谁引用 | 叠加字段 |
|---|---|---|
| types/slot | elements.slots、recipes.slots、verbs.slot | elements 叠加 `actionid`；recipes 叠加 `greedy` |
| types/expulsion | recipes.alt[].expulsion | — |
| types/xtriggers-card | elements Card 分支的 `xtriggers` | — |
| types/xtriggers-aspect | elements Aspect 分支的 `xtriggers` | — |

## 字典字段标准样式

```json
"aspects": {
    "type": "object",
    "additionalProperties": false,
    "description": "…",
    "patternProperties": {
        ".*": {
            "type": "integer",
            "title": "Aspect/Element: Amount"
        }
    }
}
```

要点：`patternProperties` 用 `".*"` 接受任意键；值类型默认 integer（数量语义），字符串字典如 decks.drawmessages 用 string；标题格式为"<元素>: <含义>"。

## 枚举字段标准样式

```json
"flavour": {
    "type": "string",
    "oneOf": [
        { "const": "Grand" },
        { "const": "Melancholy", "description": "红特效" }
    ]
}
```

要点：用 `oneOf`+`const` 而非 `enum`；每个 const 可附 description 解释该值效果（补全时展示）。

## 其他惯例

- 缩进 4 空格，全部使用双引号。
- `$comment` 用于标注未研究透的字段（`"@TODO: Research and provide description"` 风格）。
- 数值约束：概率类 `minimum: 0, maximum: 100`；计数类 `minimum: 0`；有默认值的字段写 `"default": ...`。
- 根文件 `json-schema/files` 只含六个顶层键（elements/recipes/decks/legacies/endings/verbs），各 `$ref` 到 `./schemas/<类目>`。
