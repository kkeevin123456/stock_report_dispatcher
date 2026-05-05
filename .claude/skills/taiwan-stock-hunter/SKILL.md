---
name: taiwan-stock-hunter
description: 台股起漲潛力股獵人。找出近期「即將起漲」或「尚未被市場充分反映」的台股標的。
---

# 台股起漲潛力股獵人

> 執行前請讀取補充資料：
> - `.claude/skills/taiwan-stock-hunter/references/leading-indicators.md`
> - `.claude/skills/taiwan-stock-hunter/references/sector-scoring.md`

## 執行流程

```
Phase 1：市場雷達掃描 → 族群打分 → 選 2~3 個潛力族群
Phase 2：個股篩選 → 七大起漲訊號核查 → 假新聞過濾
Phase 3：套用 taiwan-stock-analysis 深度分析 → 最終推薦
Phase 4：報告切分 → GitHub Actions 推送至 Line Bot
```

**重要原則**：
- 所有數字必須有來源，無法查證標注「⚠️ 未核實」
- 優先找「消息面已改善但股價未反映」的族群
- 每 Phase 都執行實際 web_search，不靠記憶

---

## Phase 1：市場雷達掃描

搜尋時間窗口：近 2 週（預設），不足則延伸至近 1 個月。
關鍵字必須帶當前年月，例如 `2026年5月`。

### 必做搜尋（至少 5 次，並行進行）
```
搜尋 A："{當前年月} 台股 法人 外資 買超 族群 潛力"
搜尋 B："{當前年月} 台股 新催化劑 題材 未反映"
搜尋 C："{當前年月} 台股 投信 連買 布局 哪些"
搜尋 D："{當前年月} 台股 哪些族群 外資 上調目標價"
搜尋 E："{當前年月} 台灣 產業 新訂單 長約 需求回升"
```

### 族群強度打分（5 維度，各 0~4 分，滿分 20 分）

| 維度 | 0分 | 2分 | 4分 |
|------|-----|-----|-----|
| 新催化劑新鮮度 | 超過 1 個月舊消息 | 近 2~4 週新事件 | 近 2 週結構性新事件 |
| 法人態度轉變 | 無報告或中立 | 1~2 篇報告提及 | 多篇上調評級/目標價 |
| 外資籌碼動向 | 賣超或無動作 | 零星買超 | 連續買超 5 天以上 |
| 股價 vs 消息落差 | 近 1 月漲 30%+ | 小漲，消息略超前 | 橫盤/微跌但消息明顯改善 |
| 基本面支撐 | 純題材無獲利 | 有獲利但趨緩 | 三率三升、合約負債激增 |

選總分 ≥ 12 的族群進入 Phase 2。

---

## Phase 2：個股篩選與起漲訊號核查

### 初篩（每族群 2~3 檔，優先選漲幅落後但基本面不差者）
```
搜尋："{族群名} 2026 哪些個股 尚未 跟漲 低估 補漲"
```

### 七大起漲前訊號核查（每項 ✅/❌/⚠️）

| 信號 | 搜尋關鍵字 | 強訊號條件 |
|------|-----------|-----------|
| 1. 合約負債季增 | `{公司} {股號} 合約負債 季增` | QoQ > 10% |
| 2. 月營收加速 | `{公司} {股號} 2026 月營收 月增` | 連 2 月加速 |
| 3. 毛利率拐點 | `{公司} {股號} 毛利率 趨勢 季` | 下滑轉回升 |
| 4. 外資籌碼轉向 | `{公司} {股號} 外資 買超 2026` | 轉買超且連續 |
| 5. 業外干擾消除 | `{公司} {股號} 業外 匯損 一次性` | 匯損消除 |
| 6. 資本支出有支撐 | `{公司} {股號} 擴產 客戶 長約` | 有客戶名單 |
| 7. 地雷掃描 | `{公司} {股號} 現增 轉讓 處置 2026` | 無重大地雷 |

計分：✅=1分、✅✅=2分、⚠️=0分、❌=-1分、🔴=-3分
每族群選得分最高的 1 檔進入 Phase 3。

### 假新聞過濾
出現具體數字時必須查證：官方公告 > 法說會原文 > 主流財經媒體

---

## Phase 3：深度分析

對每檔個股，讀取並執行：
`.claude/skills/taiwan-stock-analysis/SKILL.md`

輸出最終推薦：

```markdown
## 🎯 本次潛力標的

### 🟢 起漲布局（N 檔）
- 股價 vs 消息面落差說明
- 預計訊號更明確的時間點
- 建議觀察的關鍵指標

### 🟡 持續追蹤（N 檔）

### 🔴 本次排除（N 檔）
```

---

## Phase 4：傳送報告至 Line Bot

Phase 3 完成後執行以下步驟：

**訊息格式規則**：
- 第一則開頭：`📊 台股每日潛力股報告 YYYY/MM/DD`（填今天日期）
- 每則不超過 4500 字元，在段落換行邊界切分
- 最後一則結尾加上：
  ```
  ---
  ⚠️ 以上為 AI 分析參考，投資有風險，請自行判斷。
  ```

**執行腳本**（將 FULL_REPORT 替換為完整報告後執行）：

```python
import json, urllib.request, os

FULL_REPORT = 'REPLACE_WITH_FULL_REPORT'

def split_messages(text, max_len=4500):
    messages = []
    current = ''
    for line in text.split('\n'):
        candidate = current + line + '\n'
        if len(candidate) <= max_len:
            current = candidate
        else:
            if current.strip():
                messages.append(current.rstrip())
            current = line + '\n'
    if current.strip():
        messages.append(current.rstrip())
    return messages

messages = split_messages(FULL_REPORT)

payload = {
    'event_type': 'stock-report',
    'client_payload': {
        'userId': 'U62acfc9bd553026db7aa5e1bc3c58afd',
        'messages': messages
    }
}

body = json.dumps(payload, ensure_ascii=False).encode('utf-8')
req = urllib.request.Request(
    'https://api.github.com/repos/kkeevin123456/stock_report_dispatcher/dispatches',
    data=body,
    headers={
        'Authorization': f"token {os.environ.get('GITHUB_TOKEN', '')}",
        'Content-Type': 'application/json',
        'Accept': 'application/vnd.github.v3+json'
    },
    method='POST'
)
with urllib.request.urlopen(req, timeout=30) as resp:
    print(f'Status: {resp.status}')
    print(f'✅ 已觸發 GitHub Actions，共 {len(messages)} 則訊息推送中')

```

**注意**：`GITHUB_TOKEN` 需在 cloud environment 的 Environment Variables 中設定。

---

## 注意事項

1. 起漲 vs 追高：族群近 1 月漲 30%+ 則「股價落差」得 0 分
2. 禁止憑空捏造：找不到數據誠實標注「無法取得」
3. 全程繁體中文輸出
4. 查證優先順序：公開資訊觀測站 > 法說會 > 主流財經媒體
