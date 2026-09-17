# 第三方 API 集成说明

本文档记录地球Online 接入的所有第三方 API，包括调用方式、参数、错误处理和降级策略。

## 目录

- [LLM API](#llm-api)
- [和风天气 API](#和风天气-api)
- [高德地图 API](#高德地图-api)

---

## LLM API

### 用途

- 每日任务生成（核心功能）
- 任务感想分析（提炼成长关键词、情绪判断）

### 接口规范

所有 LLM 调用统一使用 OpenAI Chat Completions 格式：

```
POST {baseUrl}/chat/completions
Headers:
  Content-Type: application/json
  Authorization: Bearer {apiKey}
Body:
  {
    "model": "{model}",
    "messages": [...],
    "temperature": 0.7-0.85,
    "max_tokens": 1000-2000,
    "response_format": { "type": "json_object" }  // 可选，见下方兼容性说明
  }
```

### 调用封装

代码中通过 `callLLm(messages, opts)` 函数统一封装，位于 `index.html` 的 IIFE 内部。该函数处理了以下兼容性问题：

1. **response_format 降级**：先尝试带 `response_format: {type: "json_object"}` 调用，如果返回 400 或网络错误，则自动降级为不带该参数重试。这是因为部分 OpenAI 兼容接口（如某些代理服务、旧版本地模型）不支持该参数。

2. **JSON 解析容错**：通过 `parseJsonSafe(text)` 函数处理 LLM 返回的非严格 JSON（如包含 markdown 代码块标记、前后多余文本等），使用正则提取第一个 `{...}` 块。

3. **错误透传**：API 返回非 200 时，将状态码和响应文本前 200 字符拼入错误消息，方便排查。

### 任务生成 Prompt 结构

任务生成的 system prompt 和 user prompt 分开构建：

- **system prompt**：设定角色为「专业游戏任务设计师」，要求只输出 JSON
- **user prompt**：由 `buildTaskPrompt()` 动态构建，包含以下段落：
  1. 角色定义和产品背景
  2. 用户画像（名称、兴趣、目标、等级、属性）
  3. 今日世界状态（日期、星期、工作日/周末、天气、位置）
  4. 近期已完成任务（去重用）
  5. 生成规则（8 条约束）
  6. 输出格式说明和字段约束

### 感想分析 Prompt

```
system: 你是一个成长教练。根据用户的任务感想，提炼3-5个成长关键词，并判断情绪倾向。只输出JSON。
user: 任务：{task.title}
      感想：{reflectionContent}
      输出格式：{"keywords":["关键词1","关键词2"],"mood":"情绪描述"}
```

### 各服务商适配注意事项

| 服务商 | CORS | response_format | 备注 |
|--------|------|-----------------|------|
| OpenAI | 支持 | 支持 | 官方接口，最稳定 |
| DeepSeek | 支持 | 支持 | 性价比高，任务生成质量好 |
| 字节豆包（Ark） | 支持 | 部分支持 | model 填接入点 ID 而非模型名；部分旧接入点不支持 response_format |
| 月之暗面 | 支持 | 不支持 | 代码会自动降级，无需手动处理 |
| 通义千问（兼容模式） | 支持 | 支持 | 需使用 compatible-mode 端点 |
| Ollama | 支持（localhost） | 不支持 | 本地模型，API Key 随意填 |

### 调用频率与成本

| 操作 | 日均调用次数 | 单次 token 估算 |
|------|-------------|----------------|
| 任务生成 | 1 次（首次打开时） | 输入 1500-2500，输出 800-1500 |
| 感想分析 | 0-3 次（仅记录感想时） | 输入 200-500，输出 100-200 |
| 连接测试 | 不定 | 极少 |

按 gpt-4o-mini 定价估算，单用户月成本约 5-15 元人民币。

---

## 和风天气 API

### 用途

- 获取当前位置实时天气
- 天气信息用于 HUD 展示和任务生成 Prompt（影响任务类型推荐，如雨天优先室内任务）

### 接口调用

```
GET https://devapi.qweather.com/v7/weather/now?location={lon},{lat}&key={apiKey}
```

注意：和风天气的 location 参数格式是 `经度,纬度`（lon,lat），与大多数地图 API 的 lat,lon 顺序相反。

### 响应处理

```javascript
{
  "code": "200",           // 200 表示成功
  "now": {
    "temp": "23",          // 温度（摄氏度）
    "text": "多云",         // 天气描述
    "icon": "101",
    ...
  }
}
```

代码中仅提取 `temp` 和 `text` 两个字段。

### 错误处理

- API 返回非 `200` code 时静默失败，不影响主流程
- 定位被拒绝或超时时（8 秒超时），HUD 显示「定位被拒绝」或「定位不可用」
- 未配置天气 Key 时，整个天气模块跳过，任务生成 Prompt 中不包含天气信息

### 免费额度

个人开发者免费版每日 1000 次调用，地球Online 每日仅调用 1 次，额度充足。

---

## 高德地图 API

### 用途（当前版本：预留）

- 周边 POI 查询（用于探坑推荐、探索任务生成）
- 逆地理编码（将经纬度转为城市/区域名称）

### 当前状态

v0.1.0 版本中，高德地图 Key 已在初始化页收集，但尚未实际调用。周边 POI 功能计划在 v0.3.0 接入。

### 计划调用的接口

**周边搜索 POI：**
```
GET https://restapi.amap.com/v3/place/around?key={apiKey}&location={lon},{lat}&radius=2000&types=...&offset=20
```

**逆地理编码：**
```
GET https://restapi.amap.com/v3/geocode/regeo?key={apiKey}&location={lon},{lat}
```

### 注意事项

- 高德 Web 服务 API 的 location 参数同样是 `经度,纬度` 格式
- 需要在高德控制台申请「Web服务」类型的 Key，而非「Web端（JS API）」类型
- 个人开发者每日有免费调用额度

---

## 通用错误处理原则

1. **非核心 API 失败不阻塞主流程**：天气、地图等可选服务失败时静默降级，仅影响任务生成的丰富度，不影响核心功能
2. **核心 API（LLM）失败时明确提示**：任务生成失败时通过 toast 显示错误信息，包含状态码和简要原因
3. **网络超时**：天气定位设置 8 秒超时，LLM 调用依赖浏览器默认超时（通常无超时，建议后续添加 AbortController）
4. **用户数据安全**：所有 API Key 仅存储在 localStorage，不经过任何中间服务器
