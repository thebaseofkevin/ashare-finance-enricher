### 1. 基础过滤：剔除“地雷”与平庸者

**逻辑：** 排除亏损企业、高杠杆企业及上市不足 3 年（缺乏历史均利数据）的公司。

```sql
SELECT code, code_name, market_cap, price, pe, roe
FROM stocks
WHERE 
    -- 1. 盈利性门槛：ROE 必须大于 10%，确保生意本身是赚钱的
    roe > 10 
    -- 2. 估值门槛：PE 在 0 到 25 之间（拒绝泡沫，剔除亏损）
    AND pe > 0 AND pe < 18
    -- 3. 规模门槛：市值处于 30 亿至 200 亿之间（典型的小市值成长股区间）
    AND market_cap BETWEEN 3000000000 AND 20000000000
    -- 4. 经营安全性：经营现金流必须为正
    AND operating_cash_flow > 0
    -- 5. 时间维度：上市时间超过 3 年，确保有历史数据支撑估值计算
    AND ipoDate < DATE_SUB(CURDATE(), INTERVAL 5 YEAR);

```

---

### 2. 核心估值：成长式的“净现金”筛选

**逻辑：** 寻找“账面现金充足”的公司。如果 **(市值 - 净现金) / 利润** 极低，说明市场几乎白送了该公司的商业经营部分。

```sql
SELECT 
    code, 
    code_name, 
    price,
    market_cap,
    -- 计算净现金 (Net Cash Position)
    (cash - short_term_borrowing) AS net_cash,
    -- 计算 L 值：剔除现金后的实际估值倍数
    (market_cap - (cash - short_term_borrowing)) / net_profit AS L_value,
    gross_profit_margin
FROM stocks
WHERE 
    -- 护城河指标：毛利率需大于 30%（代表具备一定的定价权或成本优势）
    CAST(REPLACE(gross_profit_margin, '%', '') AS DECIMAL) > 30
    -- 财务稳健：现金必须覆盖短期借款
    AND cash > short_term_borrowing
    -- 价值洼地：L 值小于 12（即剔除净现金后，仅需12年利润即可收回成本）
    AND (market_cap - (cash - short_term_borrowing)) / net_profit < 12
ORDER BY L_value ASC;

```

---

### 3. 终极研选：威科夫“超跌+黄金坑”潜力池

**逻辑：** 结合 PB（账面价值）与现金流，寻找那些价格接近净资产但依然在创造现金的“被冷落者”，这通常是威科夫 **Phase C (Spring)** 容易发生的区域。

```sql
SELECT 
    code, 
    code_name, 
    price, 
    pb,
    (operating_cash_flow / net_profit) AS earnings_quality -- 利润含金量
FROM stocks
WHERE 
    -- 价格底线：PB 处于 0.8 到 1.5 之间（极高的安全边际）
    pb BETWEEN 0.8 AND 1.5
    -- 盈利质量：经营现金流 / 净利润 > 1（赚的是真钱，不是应收账款）
    AND (operating_cash_flow / net_profit) >= 1.0
    -- 行业选择：排除过度拥挤的赛道（示例：排除银行金融等重资产行业，专注制造业/消费）
    AND industry NOT LIKE '%银行%' AND industry NOT LIKE '%保险%'
ORDER BY pb ASC;

```

---

### 特别提示 (Guardrails)

1. **关于 $L < R$ 的警示：** SQL 只能帮你筛出 $L$ (当前估值)，但 $R$ (行业合理中枢) 需要你调取所属行业的平均 PE。在执行我的【第二阶段】协议时，若 $L$ 显著低于行业平均 $R$，且公司**毛利率没有下滑**，那就是绝佳的左侧机会。
2. **不买理由排查：** 即使 SQL 结果完美，若 `investment_cash_flow` (投资现金流) 连续多年大幅流出且未见营收增长，需警惕“资本黑洞”。
3. **威科夫共振：** 请观察筛选出股票的 **60 月线**。若股价在 60 月线附近缩量横盘（无量震荡），配合上述财务指标，往往就是机构吸筹完成的 **Phase C** 阶段。
