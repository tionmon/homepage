---
name: chengdu-metro-api
description: 为成都地铁 WebUI 或家庭仪表盘接入实时数据，查询书房站双向到站时间、指定方向后续班次，以及 2 号线列车所在区段。
---

# 成都地铁实时到站 API

以下请求来自 2026-11-26 的成都地铁 App 抓包。基础地址为 `https://app.cdmetro.chengdurail.cn`。三个接口均使用 GET，参数放在查询字符串中。书房站的两个到站接口已用普通 HTTPS 请求从电脑直连成功，无需手机、签名、时间戳或登录令牌。接入新项目时用一次实际请求复核响应结构。

## 按页面选择请求

| 页面需要的数据 | 路径及书房站示例参数 | 读取位置 |
|---|---|---|
| 同屏显示两个方向的下一班 | `/op/trains/nearby-station?stationNo=0257` | 遍历 `data.list`，按 `direction` 分组，用 `endStationName` 标记方向，从各组 `trainList` 取到站预报 |
| 某个方向的下一班及后续班次 | `/op/trains/direction-nearby-station?lineNo=02&direction=01&stationNo=0257` | 同样读取各组的 `trainList`。抓包中曾返回两班，按实际条数渲染 |
| 2 号线列车位置图 | `/op/trains/all-locations?lineNo=02` | 遍历 `data.list[].trainsLos[]`，用 `trainId` 标识列车，读取当前站和下一站绘制区段 |

书房站的内部编号是 `0257`，线路编号是 `02`。把编号当字符串，保留前导零。换站时使用站点数据中的 `stationNo`；站牌上的 `02 03` 是线路与站序，不能用来推算内部编号。书房站的 `01` 往犀浦，`02` 往龙泉驿。其他线路的方向名称以响应为准。

## 把响应映射到仪表盘

- 先检查 HTTP 请求成功且 JSON 的 `code === 0`，再读取 `data.list`。用方向字段匹配卡片，避免依赖数组顺序。
- `trainList[].duration` 是列车到**所查询车站**的预计剩余秒数。保留非负数值，按秒数升序展示。数值 `0` 显示「列车进站」，正数可用 `Math.ceil(duration / 60)` 显示整分钟。空数组显示「暂无预报」，缺失值应在数值转换前处理，以免误报进站。
- `trainStationName`、`trainNextStationName` 是该列车的当前站和下一站，对应编号为 `trainStationNo`、`trainNextStationNo`。例如往犀浦的列车可显示「龙平路 → 书房」，而乘客从书房上车后的下一站是界牌。两种“下一站”在页面上应明确标注。
- 双向看板每轮只调用一次双向接口。可由 WebUI 后端请求上游并缓存约 15 秒，让多个屏幕共享结果。15 秒是家庭仪表盘的刷新选择。记录成功拉取时间，遇到超时或业务错误时显示「更新失败」，若保留旧值则标出上次更新时间。
- 首末班时间、出入口信息和设施图曾在站点详情页看到，其数据接口尚待核实。需要这些内容时单独接入来源。

在 Windows 终端可用以下请求核对书房站两组终点及秒数，再接入页面：

```powershell
$url = 'https://app.cdmetro.chengdurail.cn/op/trains/nearby-station?stationNo=0257'
$result = Invoke-RestMethod -Uri $url -TimeoutSec 8
if ($result.code -ne 0) { throw "成都地铁接口返回错误：$($result.msg)" }
$result.data.list | Select-Object direction, endStationName,
    @{Name='arrivalsSeconds'; Expression={ @($_.trainList | ForEach-Object { $_.duration }) }}
```
