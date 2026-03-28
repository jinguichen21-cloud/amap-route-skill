---
name: amap-route-skill
description: >
  高德智能路径规划。当用户需要查询路线、规划出行、查找导航方案时使用。
  支持驾车、公交、地铁、骑行、步行等多种出行方式的路径规划。
  每当用户提到"怎么去"、"路线规划"、"导航"、"从A到B"、"出行方案"、"怎么走"等关键词时都应触发本技能。
metadata:
  label: 高德智能路径规划
---

# 高德智能路径规划

基于高德地图的智能路径规划服务，提供美观、易读的出行方案。

## MCP 服务配置

本技能依赖以下 MCP 服务，运行时**必须**先读取 `mcp-config.json` 获取服务 URL：

| 环境变量名 | MCP 服务 | mcpId |
|------------|----------|-------|
| `$AMAP_MAPS_URL` | 高德地图 | 1031 |

### 配置读取方式

**约束：调用方 agent 必须在执行任何 MCP 调用前，先读取 mcp-config.json 文件**

```python
import json

with open("mcp-config.json", "r", encoding="utf-8") as f:
    config = json.load(f)

# 获取高德地图服务的 URL
amap_url = config["AMAP_MAPS_URL"]["url"]
```

## 工作流程

1. **读取配置**：首先读取 `mcp-config.json` 获取高德地图服务 URL
2. **解析需求**：识别用户的出发地、目的地
3. **判断出行方式**：
   - **情况A**：用户已明确指定出行方式（如"开车去"、"坐公交"）→ 直接进入步骤4
   - **情况B**：用户未指定出行方式 → **发起反问**，让用户选择出行方式，等待用户确认后再继续
4. **地理编码**：使用 `maps_geo` 将地址转换为经纬度坐标
5. **路径规划**：根据出行方式调用对应的路径规划接口
   - 驾车：`maps_direction_driving`
   - 公交/地铁：`maps_direction_transit_integrated`
   - 骑行：`maps_direction_bicycling`
   - 步行：`maps_direction_walking`
6. **美化输出**：将规划结果格式化为美观、易读的出行方案

## 出行方式识别与反问

### 出行方式关键词识别

根据用户输入自动识别出行方式：

| 关键词 | 出行方式 | 工具 |
|--------|----------|------|
| 开车、驾车、自驾、打车 | 🚗 驾车 | `maps_direction_driving` |
| 公交、地铁、公共交通、坐公交 | 🚌 公交/地铁 | `maps_direction_transit_integrated` |
| 骑车、骑行、自行车、共享单车 | 🚴 骑行 | `maps_direction_bicycling` |
| 走路、步行、散步 | 🚶 步行 | `maps_direction_walking` |

### 出行方式反问流程

**当用户未明确指定出行方式时，必须发起反问**：

```markdown
📍 **路线规划**：从 {出发地} 到 {目的地}

请选择您的出行方式：

🚗 **驾车** - 自驾或打车
🚌 **公交/地铁** - 公共交通出行
🚴 **骑行** - 骑自行车或共享单车
🚶 **步行** - 步行前往

请回复数字 **1-4** 或出行方式名称，我将为您规划最优路线。
```

**反问规则**：
1. **必须等待用户明确选择**后才能继续执行路径规划
2. 用户回复后，根据选择执行对应的路径规划
3. 如用户回复模糊（如"都可以"），则默认推荐公交/地铁方案（最通用）
4. 如用户改变主意，可重新发起反问

**示例对话**：

```
用户：从望京SOHO怎么去首都机场？
AI：📍 路线规划：从 望京SOHO 到 首都机场

    请选择您的出行方式：
    🚗 驾车 - 自驾或打车
    🚌 公交/地铁 - 公共交通出行
    🚴 骑行 - 骑自行车或共享单车
    🚶 步行 - 步行前往

    请回复数字 1-4 或出行方式名称...

用户：1
AI：[执行驾车路径规划，输出驾车方案]

---

用户：2
AI：[执行公交路径规划，输出公交方案]
```

## 路径规划调用与结果解析

### 调用方式

#### 驾车路径规划

```bash
python3 scripts/call_mcp.py call "$AMAP_MAPS_URL" maps_direction_driving \
  --params '{"origin": "116.481028,39.989643", "destination": "116.434446,39.90816"}'
```

#### 公交路径规划

```bash
python3 scripts/call_mcp.py call "$AMAP_MAPS_URL" maps_direction_transit_integrated \
  --params '{"origin": "116.481028,39.989643", "destination": "116.434446,39.90816", "city": "北京", "cityd": "北京"}'
```

#### 骑行路径规划

```bash
python3 scripts/call_mcp.py call "$AMAP_MAPS_URL" maps_direction_bicycling \
  --params '{"origin": "116.481028,39.989643", "destination": "116.434446,39.90816"}'
```

#### 步行路径规划

```bash
python3 scripts/call_mcp.py call "$AMAP_MAPS_URL" maps_direction_walking \
  --params '{"origin": "116.481028,39.989643", "destination": "116.434446,39.90816"}'
```

#### 地理编码（地址转坐标）

```bash
python3 scripts/call_mcp.py call "$AMAP_MAPS_URL" maps_geo \
  --params '{"address": "北京市朝阳区望京SOHO", "city": "北京"}'
```

---

### MCP 返回数据结构解析

MCP 调用返回的数据格式如下，需要从中提取关键信息：

```json
{
  "content": [
    {
      "type": "text",
      "text": "{\"status\":\"1\",\"info\":\"OK\",\"infocode\":\"10000\",\"route\":{...}}"
    }
  ],
  "isError": false
}
```

**关键：需要解析 `content[0].text` 中的 JSON 字符串，提取 `route` 字段**

### 数据提取与转换

#### 驾车数据提取

从返回的 JSON 中提取以下字段：

```python
# 解析路径数据
route = result["route"]
paths = route["paths"][0]  # 取第一条路线

distance = int(paths["distance"])  # 距离（米）
duration = int(paths["duration"])  # 时间（秒）
tolls = int(paths.get("tolls", 0))  # 过路费（元）
strategy = paths.get("strategy", "")  # 策略说明

# 步骤信息
steps = paths["steps"]
for step in steps:
    instruction = step["instruction"]  # 导航指令
    step_distance = step["distance"]  # 步骤距离
    road = step.get("road", "")  # 道路名称
    action = step.get("action", "")  # 动作（转向等）
    toll_road = step.get("toll_road", "")  # 是否收费路段
```

**转换计算**：
- 距离：`distance / 1000` → 公里（保留1位小数）
- 时间：`duration / 60` → 分钟（向上取整）

#### 公交数据提取

```python
route = result["route"]
transits = route["transits"][0]  # 取第一条方案

duration = int(transits["duration"])  # 总时间（秒）
cost = transits.get("cost", "0")  # 费用（元）
segments = transits["segments"]  # 路段信息

# 解析每个路段
for segment in segments:
    if "bus" in segment:
        # 公交/地铁段
        buslines = segment["bus"]["buslines"][0]
        line_name = buslines["name"]  # 线路名称
        start_station = buslines["departure_stop"]["name"]  # 上车站
        end_station = buslines["arrival_stop"]["name"]  # 下车站
        station_count = int(buslines.get("via_num", 0))  # 经过站数
    elif "walking" in segment:
        # 步行段
        walk_distance = segment["walking"]["distance"]  # 步行距离
        walk_duration = segment["walking"]["duration"]  # 步行时间
```

**转换计算**：
- 总时间：`duration / 60` → 分钟
- 换乘次数：`len([s for s in segments if "bus" in s]) - 1`

#### 骑行/步行数据提取

```python
route = result["route"]
paths = route["paths"][0]

distance = int(paths["distance"])  # 距离（米）
duration = int(paths["duration"])  # 时间（秒）
steps = paths["steps"]

# 骑行额外信息
calories = distance * 0.05  # 估算消耗热量（千卡）

# 步行额外信息
steps_count = distance * 1.3  # 估算步数
```

**转换计算**：
- 距离：`distance / 1000` → 公里
- 时间：`duration / 60` → 分钟
- 骑行热量：`distance / 1000 * 25` → 千卡（约每公里25千卡）
- 步行步数：`distance * 1.3` → 步（约每米1.3步）

## 输出格式规范

### 数据到模板的映射规则

将从 MCP 返回数据中提取的字段，按以下规则映射到输出模板：

| 模板变量 | 数据来源 | 转换方式 |
|---------|---------|---------|
| `{出发地}` | 用户输入 | 直接使用 |
| `{目的地}` | 用户输入 | 直接使用 |
| `{X.X 公里}` | `paths["distance"]` | `round(distance/1000, 1)` |
| `{XX 分钟}` | `paths["duration"]` | `ceil(duration/60)` |
| `{约 XX 元}` | `paths["tolls"]` | 直接使用或估算 |
| `{动作}` | `step["instruction"]` | 清洗后使用 |
| `{X 米/公里}` | `step["distance"]` | 自动转换单位 |

### 驾车方案输出模板

```markdown
# 🚗 驾车路线规划

> 📍 **从**：{出发地}
> 🎯 **到**：{目的地}
> 📏 **总距离**：{X.X 公里}
> ⏱️ **预计时间**：{XX 分钟}
> 💰 **过路费**：{约 XX 元}

---

## 🛣️ 推荐路线

### 路线概览
{路线整体描述，如：途经 XX 路 → XX 高速 → XX 路}

### 详细导航

| 步骤 | 操作 | 距离 | 说明 |
|------|------|------|------|
| 1 | 🚦 {动作} | {X 米} | {详细说明} |
| 2 | ➡️ {动作} | {X 公里} | {详细说明} |
| 3 | 🛣️ {动作} | {X 公里} | {详细说明} |
| ... | ... | ... | ... |
| N | 🏁 到达目的地 | - | {目的地附近提示} |

---

💡 **温馨提示**
- {路况提示，如：高峰期 XX 路段可能拥堵}
- {其他建议，如：建议提前规划停车}
```

**导航指令清洗规则**：
1. 去除 HTML 标签（如 `<b>`、`<\/b>`）
2. 将 `"` 替换为 `"`
3. 简化冗余描述（如"沿XX路行驶XX米后"简化为"沿XX路"）
4. 关键转向信息保留（左转、右转、直行、掉头等）

### 公交/地铁方案输出模板

```markdown
# 🚌 公共交通路线规划

> 📍 **从**：{出发地}
> 🎯 **到**：{目的地}
> ⏱️ **预计时间**：{XX 分钟}
> 💰 **预计费用**：{X 元}
> 🚇 **换乘次数**：{X 次}

---

## 🎯 推荐方案

### 方案一：{方案特点，如：最快到达}
⏱️ **总时长**：{XX 分钟} | 🚇 **换乘**：{X 次} | 💰 **费用**：{X 元}

#### 🚶 行程详情

| 阶段 | 交通方式 | 行程 | 时间 |
|------|----------|------|------|
| 1 | 🚶 步行 | 从 {起点} 到 {公交站/地铁站} | {X 分钟} |
| 2 | 🚌 {公交线路} / 🚇 {地铁线路} | 乘坐 {X 站} | {X 分钟} |
| 3 | 🚇 换乘 {地铁线路} | 在 {站点} 换乘 | {X 分钟} |
| 4 | 🚶 步行 | 从 {公交站/地铁站} 到 {终点} | {X 分钟} |

**📋 详细指引**
1. 步行至 **{站点名称}**（约 {X} 分钟）
2. 乘坐 **{线路名称}** 方向，经过 {X} 站，在 **{下车站点}** 下车
3. {如有换乘：换乘 **{线路}**，经过 {X} 站}
4. 步行至目的地（约 {X} 分钟）

---

### 方案二：{方案特点，如：最少换乘}
{同上格式}

---

💡 **温馨提示**
- {发车时间提示，如：首班车 XX:XX，末班车 XX:XX}
- {拥挤度提示，如：早高峰 XX 线较拥挤，建议错峰出行}
```

**公交数据映射规则**：

| 模板变量 | 数据来源 | 示例 |
|---------|---------|------|
| `{XX 分钟}` | `transits["duration"]` / 60 | 75 分钟 |
| `{X 元}` | `transits["cost"]` | 6 元 |
| `{X 次}` | 统计 bus 类型 segments 数量 - 1 | 1 次 |
| `{公交线路}` | `buslines["name"]` | 地铁14号线 |
| `{上车站}` | `buslines["departure_stop"]["name"]` | 望京东站 |
| `{下车站}` | `buslines["arrival_stop"]["name"]` | 望京站 |
| `{X 站}` | `buslines["via_num"]` | 3 站 |

**线路名称清洗**：
- 去除方向信息（如 "地铁14号线(善各庄方向)" → "地铁14号线"）
- 保留线路类型标识（地铁、公交、快速公交等）

### 骑行方案输出模板

```markdown
# 🚴 骑行路线规划

> 📍 **从**：{出发地}
> 🎯 **到**：{目的地}
> 📏 **总距离**：{X.X 公里}
> ⏱️ **预计时间**：{XX 分钟}
> 🔥 **消耗热量**：{约 XXX 千卡}

---

## 🛣️ 推荐路线

### 路线概览
{路线整体描述}

### 详细指引

| 步骤 | 操作 | 距离 | 路况 |
|------|------|------|------|
| 1 | 🚴 {动作} | {X 米} | {路况说明} |
| 2 | ➡️ {动作} | {X 公里} | {路况说明} |
| ... | ... | ... | ... |

---

💡 **温馨提示**
- {路况提示，如：XX 路段有上坡，注意安全}
- {设施提示，如：途经 XX 处有共享单车停放点}
```

**骑行数据映射规则**：

| 模板变量 | 数据来源 | 转换计算 |
|---------|---------|---------|
| `{X.X 公里}` | `paths["distance"]` / 1000 | round(distance/1000, 1) |
| `{XX 分钟}` | `paths["duration"]` / 60 | ceil(duration/60) |
| `{约 XXX 千卡}` | `distance` * 0.025 | round(distance/1000 * 25) |
| `{动作}` | `step["instruction"]` | 清洗后使用 |
| `{路况说明}` | `step["road"]` + 分析 | 根据道路类型推断 |

### 步行方案输出模板

```markdown
# 🚶 步行路线规划

> 📍 **从**：{出发地}
> 🎯 **到**：{目的地}
> 📏 **总距离**：{X.X 公里}
> ⏱️ **预计时间**：{XX 分钟}
> 👣 **步数**：{约 XXXX 步}

---

## 🛣️ 推荐路线

### 路线概览
{路线整体描述}

### 详细指引

| 步骤 | 操作 | 距离 | 地标 |
|------|------|------|------|
| 1 | 🚶 {动作} | {X 米} | {途经地标} |
| 2 | ➡️ {动作} | {X 米} | {途经地标} |
| ... | ... | ... | ... |

---

💡 **温馨提示**
- {步行友好度提示，如：XX 路段有步行道，适合散步}
- {设施提示，如：途经 XX 公园，可休息}
```

**步行数据映射规则**：

| 模板变量 | 数据来源 | 转换计算 |
|---------|---------|---------|
| `{X.X 公里}` | `paths["distance"]` / 1000 | round(distance/1000, 1) |
| `{XX 分钟}` | `paths["duration"]` / 60 | ceil(duration/60) |
| `{约 XXXX 步}` | `distance` * 1.3 | int(distance * 1.3) |
| `{动作}` | `step["instruction"]` | 清洗后使用 |
| `{途经地标}` | `step["road"]` | 直接使用 |

## 输出示例

### 驾车方案示例

```markdown
# 🚗 驾车路线规划

> 📍 **从**：北京市朝阳区望京SOHO
> 🎯 **到**：北京首都国际机场T3航站楼
> 📏 **总距离**：约 18.5 公里
> ⏱️ **预计时间**：35 分钟
> 💰 **过路费**：约 5 元

---

## 🛣️ 推荐路线

### 路线概览
途经 望京街 → 京密路 → 机场高速 → 机场第二高速

### 详细导航

| 步骤 | 操作 | 距离 | 说明 |
|------|------|------|------|
| 1 | 🚦 从望京SOHO出发，沿望京街向北 | 1.2 公里 | 注意望京街与阜安西路交叉口 |
| 2 | ➡️ 右转进入京密路 | 3.5 公里 | 沿京密路向东北方向行驶 |
| 3 | 🛣️ 上机场高速（收费） | 10.8 公里 | 保持左侧车道，注意限速 120km/h |
| 4 | ⬅️ 从机场高速出口下，进入机场第二高速 | 2.5 公里 | 跟随机场T3航站楼指示牌 |
| 5 | 🏁 到达首都机场T3航站楼 | - | 建议停靠在出发层或停车楼 |

---

💡 **温馨提示**
- 机场高速早高峰（7:00-9:00）可能拥堵，建议预留充足时间
- T3航站楼停车楼收费标准：前30分钟免费，之后每30分钟6元
- 如送机可直接停靠出发层，即停即走
```

### 公交方案示例

```markdown
# 🚌 公共交通路线规划

> 📍 **从**：北京市朝阳区望京SOHO
> 🎯 **到**：北京首都国际机场T3航站楼
> ⏱️ **预计时间**：约 75 分钟
> 💰 **预计费用**：6 元
> 🚇 **换乘次数**：1 次

---

## 🎯 推荐方案

### 方案一：地铁+机场快轨（推荐）
⏱️ **总时长**：约 75 分钟 | 🚇 **换乘**：1 次 | 💰 **费用**：6 元

#### 🚶 行程详情

| 阶段 | 交通方式 | 行程 | 时间 |
|------|----------|------|------|
| 1 | 🚶 步行 | 从望京SOHO到望京东站 | 8 分钟 |
| 2 | 🚇 地铁14号线 | 乘坐 3 站到望京站 | 10 分钟 |
| 3 | 🚇 换乘机场线 | 在望京站换乘机场线 | 5 分钟 |
| 4 | 🚇 机场线 | 乘坐到T3航站楼站 | 35 分钟 |
| 5 | 🚶 步行 | 从T3航站楼站到出发大厅 | 5 分钟 |

**📋 详细指引**
1. 步行至 **望京东地铁站**（约 8 分钟）
2. 乘坐 **地铁14号线（善各庄方向）**，经过 3 站，在 **望京站** 下车
3. 站内换乘 **机场线**，经过 4 站，在 **T3航站楼站** 下车
4. 按指示牌步行至出发大厅（约 5 分钟）

---

💡 **温馨提示**
- 机场线首班车 06:00，末班车 22:30，发车间隔约 10 分钟
- 早高峰（7:30-9:00）地铁较拥挤，建议错峰出行
- 机场线票价 25 元，可使用北京一卡通或手机支付
```

## 使用示例

### 示例1：用户未指定出行方式（发起反问）

**用户输入**："从望京SOHO怎么去首都机场？"

**技能执行**：
1. 读取 `mcp-config.json` 获取高德地图 URL
2. 解析出发地：望京SOHO，目的地：首都机场
3. **检测到未指定出行方式 → 发起反问**：

```markdown
📍 **路线规划**：从 望京SOHO 到 首都机场

请选择您的出行方式：

🚗 **1. 驾车** - 自驾或打车
🚌 **2. 公交/地铁** - 公共交通出行
🚴 **3. 骑行** - 骑自行车或共享单车
🚶 **4. 步行** - 步行前往

请回复数字 1-4 或出行方式名称，我将为您规划最优路线。
```

**用户回复**："1" 或 "驾车"

**技能继续执行**：
4. 调用地理编码获取坐标
5. 调用驾车路径规划接口
6. 输出美观的驾车出行方案

---

### 示例2：用户已指定出行方式（直接执行）

**用户输入**："从望京SOHO开车去首都机场"

**技能执行**：
1. 读取 `mcp-config.json` 获取高德地图 URL
2. 解析出发地：望京SOHO，目的地：首都机场
3. 识别出行方式：🚗 驾车（用户已明确指定）
4. 调用地理编码获取坐标
5. 调用驾车路径规划接口
6. 直接输出驾车出行方案

---

### 示例3：用户回复模糊

**用户输入**："从公司到客户那里怎么走？"

**技能执行**：发起反问...

**用户回复**："都可以，怎么方便怎么来"

**技能处理**：用户未明确选择，默认推荐公交/地铁方案（最通用），同时可简要说明：
> "为您推荐公交/地铁方案（最经济便捷），如需其他方式可随时告知。"

## 完整处理流程示例

### 驾车路线规划完整流程

```python
# 步骤1：调用 MCP 获取原始数据
result = call_mcp("maps_direction_driving", origin, destination)

# 步骤2：解析返回的 JSON
import json
content_text = result["content"][0]["text"]
data = json.loads(content_text)

# 步骤3：提取关键信息
route = data["route"]
paths = route["paths"][0]

distance_m = int(paths["distance"])  # 18500 米
duration_s = int(paths["duration"])  # 2100 秒
tolls = int(paths.get("tolls", 0))   # 5 元

# 步骤4：数据转换
distance_km = round(distance_m / 1000, 1)  # 18.5 公里
duration_min = (duration_s + 59) // 60      # 35 分钟

# 步骤5：解析步骤信息
steps = paths["steps"]
formatted_steps = []
for i, step in enumerate(steps, 1):
    instruction = step["instruction"]
    # 清洗指令文本
    instruction = instruction.replace("<b>", "").replace("<\/b>", "")
    instruction = instruction.replace('"', '"')
    
    step_distance = int(step["distance"])
    if step_distance >= 1000:
        distance_str = f"{round(step_distance/1000, 1)} 公里"
    else:
        distance_str = f"{step_distance} 米"
    
    formatted_steps.append({
        "step": i,
        "instruction": instruction,
        "distance": distance_str,
        "road": step.get("road", "")
    })

# 步骤6：生成温馨提示
hints = []
if any("高速" in s["road"] for s in steps):
    hints.append("途经高速公路，请注意限速标识")
if duration_min > 30:
    hints.append("行程较长，建议提前规划休息点")

# 步骤7：填充模板输出
output = f"""
# 🚗 驾车路线规划

> 📍 **从**：{origin_name}
> 🎯 **到**：{destination_name}
> 📏 **总距离**：{distance_km} 公里
> ⏱️ **预计时间**：{duration_min} 分钟
> 💰 **过路费**：约 {tolls} 元

...
"""
```

### 公交路线规划完整流程

```python
# 步骤1：调用 MCP
result = call_mcp("maps_direction_transit_integrated", origin, destination, city, cityd)

# 步骤2：解析数据
data = json.loads(result["content"][0]["text"])
route = data["route"]
transits = route["transits"][0]  # 取最优方案

# 步骤3：提取信息
duration_s = int(transits["duration"])
cost = transits.get("cost", "0")
segments = transits["segments"]

# 步骤4：计算换乘次数
bus_segments = [s for s in segments if "bus" in s]
transfer_count = len(bus_segments) - 1 if len(bus_segments) > 0 else 0

# 步骤5：解析每个路段
formatted_segments = []
for segment in segments:
    if "walking" in segment:
        walk = segment["walking"]
        walk_distance = int(walk["distance"])
        walk_duration = int(walk["duration"]) // 60
        formatted_segments.append({
            "type": "walk",
            "distance": walk_distance,
            "duration": walk_duration
        })
    elif "bus" in segment:
        buslines = segment["bus"]["buslines"][0]
        line_name = buslines["name"].split("(")[0]  # 去除方向
        station_count = int(buslines.get("via_num", 0))
        formatted_segments.append({
            "type": "bus",
            "line": line_name,
            "stations": station_count,
            "start": buslines["departure_stop"]["name"],
            "end": buslines["arrival_stop"]["name"]
        })

# 步骤6：生成输出
output = generate_transit_output(formatted_segments, duration_s, cost, transfer_count)
```

## 注意事项

1. **出行方式反问**：这是本技能的核心交互设计，**必须**在未指定出行方式时发起反问，不可直接默认选择
2. **数据解析**：MCP 返回的数据在 `content[0].text` 中，是 JSON 字符串，需要先 `json.loads()` 解析
3. **单位转换**：距离（米→公里）、时间（秒→分钟）需要正确转换
4. **文本清洗**：导航指令中的 HTML 标签（`<b>`）和转义字符（`"`）需要清洗
5. **地址解析**：如用户提供的地址不够精确，先使用 `maps_geo` 进行地理编码
6. **跨城规划**：使用公交规划时，跨城场景必须传入起点城市和终点城市
7. **坐标格式**：经纬度坐标使用 `"经度,纬度"` 格式，如 `"116.481028,39.989643"`
8. **备选方案**：公交规划返回多个方案时，优先展示前 2-3 个最优方案
9. **实时性**：路径规划结果仅供参考，实际路况可能有所变化
10. **友好提示**：根据路线特点提供个性化提示（拥堵提醒、停车建议、换乘提示等）
