# OCS 网课助手 5.2.1 接入 DeepSeek 答题

用官方 DeepSeek API（按量付费）替代题库网站来答题。**脚本主体逻辑没动**，与原版的差异只有下面 4 处。

> | 模型 | 说明 |
> | --- | --- |
> | `deepseek-flash` | 当前主模型（DeepSeek-V4.1-Flash），**支持识图**，便宜快 |
> | `deepseek-v4-pro` | 更强的推理模型，**不支持识图** |
>
> 本文配置使用 `deepseek-flash`。

## 交付文件

| 文件 | 说明 |
| --- | --- |
| `OCS 网课助手-5.2.1-DS.user.js` | 改造后的脚本 |
| `deepseek-answerer-config.json` | DeepSeek 题库配置，把里面的 key 换成你自己的即可 |

## 与原版的差异（4 处）

1. `@connect` 白名单新增一行：`// @connect api.deepseek.com`
2. 删除 `// @updateURL` 行（切断管理器自动更新提示）
3. 删除 `// @downloadURL` 行（切断管理器自动更新提示）
4. 内置更新通知 `autoNotify` 默认值 `true` → `false`（不再自动请求 `cdn.ocsjs.com` 检查版本、不再弹更新提示）

**自动更新已完全切断**：Tampermonkey 不会再提示更新，OCS 自带的更新通知也不会自动弹。
唯一还会请求 `cdn.ocsjs.com` 的地方，是你**主动**打开「📥 更新模块」页面时查一次版本（不弹窗）。

## 使用步骤

### 1. 拿 DeepSeek API key

1. 打开 <https://platform.deepseek.com>，注册/登录
2. 左侧「API keys」→「创建 API key」，复制 `sk-` 开头的字符串
3. 需要少量余额（「充值」）。按正常答题量，几块钱能用很久

### 2. 安装脚本

1. 浏览器先装好 **Tampermonkey（油猴）** 或 **ScriptCat（脚本猫）**
2. 把 `OCS 网课助手-5.2.1-DS.user.js` 拖进浏览器窗口，扩展会弹出安装页，点「安装」
3. 如果之前装过官方版 OCS 5.2.1，**先禁用或删除旧版**，避免两个脚本同时运行

### 3. 填 key

用记事本打开 `deepseek-answerer-config.json`，找到：

```json
"Authorization": "Bearer sk-REPLACE_WITH_YOUR_KEY"
```

把 `sk-REPLACE_WITH_YOUR_KEY` 换成你的 key（`Bearer ` 前缀保留）。保存。

### 4. 导入题库配置

1. 打开任意网课页面，等 OCS 悬浮窗出现
2. 悬浮窗 →「通用」→「⚙️ 全局设置」→「题库配置」
3. 把 json 文件的**全部内容**粘贴进输入框 →「应用」
4. 看到「解析到的题库」里出现 DeepSeek，确认即可

### 5. 开启图片题优化（想答图片题才需要）

OCS 默认不处理图片题。要答图片题：

1. 悬浮窗 →「通用」→「⚙️ 全局设置」→「高级设置」
2. 打开「图片题优化」
3. 开启后 OCS 会把题目里的图片下载并转成 PNG base64，配置里的 `messages` handler 会自动把它们作为 `image_url` 发给 DeepSeek

## 配置里几个关键参数

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `model` | `deepseek-flash` | 当前主模型，支持识图 |
| `thinking` | `{"type":"disabled"}` | 关闭思考模式。**不关的话默认开启且 effort=high，每题会先输出一大段推理** |
| `temperature` | `0` | 固定输出，减少随机性（仅非思考模式生效） |
| `messages` | handler | 动态把题型、题干、选项（以及图片）拼成请求 |
| `handler` | 字符串 | 把模型输出清洗成 `A` / `ABD` / `正确` / `答案===答案` 这类标准格式 |

**想更准、不介意慢和贵**：把配置里的

```json
"thinking": { "type": "disabled" },
```

改成

```json
"thinking": { "type": "enabled" },
"reasoning_effort": "high",
```

（思考模式下 `temperature` 不生效，会被忽略。）

## 先验证 key 能不能用（可选）

在 PowerShell 里跑一下（把 key 换成你的）：

```powershell
$key = "sk-你的key"
$body = '{"model":"deepseek-flash","thinking":{"type":"disabled"},"messages":[{"role":"user","content":"1+1=?只回答数字"}]}'
Invoke-RestMethod -Uri "https://api.deepseek.com/chat/completions" -Method Post `
  -Headers @{ Authorization = "Bearer $key"; "Content-Type" = "application/json" } `
  -Body $body | ConvertTo-Json -Depth 6
```

返回 `choices[0].message.content` 为 `2` 就说明 key、余额都正常。

## 图片题是怎么接上的

- 开启「图片题优化」后，OCS 会把题干/选项里的图片 URL 下载成 base64 PNG（防防盗链），并把题干里的图片 URL 替换成 `[图片1]`、`[图片2]` 标记
- 配置里的 `messages.handler` 检测到 `env.images` 非空时，把文字和图片一起组成 `content` 数组，图片以 `{"type":"image_url","image_url":{"url":"data:image/png;base64,..."}}` 传入
- DeepSeek 的限制：仅 `user` 消息可带图片；单图 ≤32MB；每张图最多按 1024 tokens 计费；单请求最多 600 张图
- 纯图片、题干里没有可识别 URL 的题（例如整块截图直接当背景）可能抓不到图，仍会按文字处理

## 以后 OCS 出新版，怎么手动合并

因为自动更新已切断，升级要手动做。步骤：

1. 从 OCS 官方（docs.ocsjs.com 或 GreasyFork）下载新版 `.user.js`
2. 用文本编辑器打开，做 3 处修改：
   - 在 `@connect` 列表里加一行 `// @connect api.deepseek.com`
   - 删除 `// @updateURL` 和 `// @downloadURL` 两行
   - 搜索 `autoNotify:`，把附近的 `defaultValue: true` 改成 `false`
3. 保存后重新安装（会覆盖旧版），**题库配置不用重配**

## 注意事项与限制

- **图片题优化默认关闭**，不开就只能答文字题（图片 URL 会以文字形式进 prompt，模型看不到图）
- **费用**：deepseek-flash 定价低；普通文字题每题几百 tokens，图片题每图最多约 1024 tokens，正常使用成本很低
- **key 安全**：json 里是明文 key，不要发给别人；不要用「公开题库配置」分享站点
- **回退**：禁用/删除本脚本，装回官方原版即可
