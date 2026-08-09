# 使用示例

以下是在对话中使用本 skill 的输入/输出示例。

## 示例 1：添加基础字段（未指定位置 → 追加末尾）

**用户输入：**
> 给 recipes 加一个字段 `canBeCraftedByPlayer`，boolean 类型，描述是"是否允许玩家手动制作"。

**agent 执行：**
1. 定位 `json-schema/schemas/recipes` 的 `items.properties`。
2. 生成片段 `"canBeCraftedByPlayer": { "type": "boolean", "description": "是否允许玩家手动制作" }`。
3. 追加到 properties 末尾（`comments` 之后）。
4. 运行 JSON 校验，汇报结果。

## 示例 2：相对位置插入 + 字典类型

**用户输入：**
> 在 elements 的 Card 分支，`aspects` 字段之后加一个 `tags` 字典字段，值类型为字符串数组。

**agent 执行：**
1. 定位 elements 的 Card 分支 properties 中的 `"aspects"` 键。
2. 生成片段：
   ```json
   "tags": {
       "type": "object",
       "additionalProperties": false,
       "description": "",
       "patternProperties": {
           ".*": {
               "type": "array",
               "additionalItems": false,
               "items": { "type": "string", "title": "element" }
           }
       }
   }
   ```
3. 插入到 `"aspects"` 块之后、`"lifetime"` 之前。
4. 提醒用户 description 为空待补充。

## 示例 3：枚举类型 + 指定位置

**用户输入：**
> 给 endings 加一个 `screenBackground` 枚举字段，可选值 Night（黑夜）/ Dawn（黎明），插在 `anim` 前面。

**agent 执行：**
1. 定位 endings 的 `items.properties` 中 `"anim"` 键。
2. 生成片段：
   ```json
   "screenBackground": {
       "type": "string",
       "description": "",
       "oneOf": [
           { "const": "Night", "description": "黑夜背景" },
           { "const": "Dawn", "description": "黎明背景" }
       ]
   }
   ```
3. 插入到 `"anim"` 之前。
4. 校验并汇报。

## 示例 4：修改已有字段

**用户输入：**
> 把 verbs 里 `icon` 字段的类型改成枚举，可选值 wood / stone / gold。

**agent 执行：**
1. 定位 verbs 的 `items.properties` 中 `"icon"` 键（原为纯 string + TODO 注释）。
2. 改写为 `oneOf`+`const` 枚举，移除原 `$comment`。
3. 校验并汇报。

## 示例 5：删除字段（含引用检查）

**用户输入：**
> 删除 decks 里的 `drawmessages$remove` 字段。

**agent 执行：**
1. 定位 decks 的 `items.properties` 中 `"drawmessages$remove"` 键并删除。
2. 用 Grep 搜索 `drawmessages\$remove` 确认无其他文件 `$ref` 引用。
3. 校验并汇报。

## 用户输入中的常见说法与解析

| 说法 | 解析 |
|---|---|
| "给 X 加个字段" | 添加操作，类目为 X |
| "值类型是 xxx" / "存 xxx 类型" | 值类型映射 |
| "放在 Y 后面/前面" | 相对位置插入 |
| "放开头 / 放最后" | 相对位置（第一个/最后一个） |
| "改成 Z 类型" | 修改操作 |
| "删掉 X 字段" | 删除操作 |
