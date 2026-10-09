# MCP_INTEGRATION.md — 麦门健康通行证 × 麦当劳 MCP

本文档说明本项目实际使用的麦当劳 MCP Server、Tool、调用流程和业务价值。

## 接入信息

- **Server**: `mcd-mcp`（麦当劳中国官方 MCP Server）
- **地址**: `https://mcp.mcd.cn`（Streamable HTTP）
- **鉴权**: `Authorization: Bearer <MCP_TOKEN>`（Token 通过环境变量/客户端配置注入，仓库中不含真实凭证）
- **申请入口**: https://open.mcd.cn/mcp

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_TOKEN"
      }
    }
  }
}
```

## 工具清单与用途

| Tool | 在本项目中的用途 | 调用时机 |
|---|---|---|
| `list-nutrition-foods` | 获取全量餐品营养数据（能量/蛋白质/脂肪/碳水/钠/钙） | 每次生成健康推荐时，一次调用拿全量约160个单品 |
| `query-nearby-stores` | 到店自取场景查询附近门店（beType=1） | 用户决定去门店时 |
| `query-meals` | 查询所选门店当前可售餐品与商品编码（orderType=1, beType=1） | 下单前确认单品在售 |
| `query-store-coupons` | 查询当前门店可用优惠券 | 算价前主动查券 |
| `calculate-price` | 计算商品合计金额（含券优惠） | 用户确认组合后 |
| `create-order` | 创建到店自取订单 | 用户明确确认后 |
| `query-order` | 查询订单状态 | 用户询问订单进度时 |
| `cancel-order` | 取消订单 | 用户要求取消时 |

## 核心调用流程

### 流程一：健康推荐（本项目核心价值）

```
用户健康档案 (health_profile.json)
        │
        ▼
list-nutrition-foods  ──►  全量营养数据
        │
        ▼
rules/health_rules.yaml  ──►  状况→营养素限额映射
        │
        ▼
计算每个单品「占每日限额百分比」→ 三色分级（🟢🟡🔴）
        │
        ▼
生成 2-3 个套餐组合 + 控糖替换建议（含糖饮料→无糖）
```

**业务价值**：麦当劳菜单对慢病人群是"信息黑箱"——巨无霸含钠 961mg，占高血压管理日限额（1500mg）的 64%，但菜单页面上看不见。本项目把官方营养数据翻译成"你的身体状况下这是什么概念"，让慢病人群第一次能带着体检报告的结论做点餐决策。

### 流程二：到店自取下单（v1 仅支持到店自取）

```
query-nearby-stores(beType=1, searchType=2, city, keyword)
   └─► 用户选择门店 → storeCode
query-meals(storeCode, orderType=1, beType=1)
   └─► 确认在售商品 → productCode
query-store-coupons → 识别可用券
calculate-price(storeCode, orderType=1, beType=1, items[+coupon])
   └─► 展示价格（注意返回单位为"分"，展示需÷100）
用户确认 → create-order → 取餐码
```

**业务价值**：健康推荐与真实交易闭环。慢病用户从"看到推荐"到"拿到餐"不需要跳出对话；券的自动匹配让健康选择不额外花冤枉钱。

## 数据边界说明（诚实披露）

- 麦当劳官方营养数据字段为：能量(kJ/kcal)、蛋白质、脂肪、碳水、钠、钙。**不含**糖、嘌呤、膳食纤维、过敏原。
- 因此：高血压/血脂/控糖/能量四类状况可做精确限额映射；高尿酸/痛风仅能依据公开食养指南做定性提示（已在规则库中标注）。
- 本项目所有限额数字来自《中国居民膳食指南(2022)》及国家卫健委慢病食养指南，规则库文件中逐条注明来源。

## 限流与容错

- MCP 限流 600 次/分钟。本项目营养数据单次全量拉取，正常单次会话总调用 <10 次，远低于限额。
- 429 错误时提示用户稍后重试；401 提示检查 Token 配置。
