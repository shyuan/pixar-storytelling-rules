# Skill 評估結果

紀錄 `pixar-storytelling-rules` skill 經過量化對照測試後的表現。

| 指標 | With Skill | Without Skill | Delta |
|---|---|---|---|
| **Pass Rate** | **100%** (28/28) | **57.2%** (16/28) | **+42.8%** |
| 平均 token 用量 | 19,744 | 11,666 | +8,078 |

> Iteration 1，執行於 2026-04-30。模型：claude-sonnet-4。

---

## 為什麼做這個評估

評估是用來驗證 skill 的設計是否**真的改變 Claude 的行為**——而不只是讓回應「看起來」更專業。關心兩個問題：

1. 啟用 skill 後，回應有沒有更貼近 Pixar 22 法則的精神？
2. 沒裝 skill 的 base model 會在哪些地方判錯？

## 測試方法

5 個測試 case，每個跑兩組：
- **with_skill**：subagent 載入 SKILL.md，依其指引回應
- **without_skill**：subagent 直接回應，完全不知道 skill 存在

兩組對同一份 assertion 評分（共 28 條），檢查**可客觀驗證**的行為：法則編號引用、模式模板遵循、特定 Gotcha 規避、繁體中文使用等。

## 5 個測試 Case

| # | 測什麼 | Prompt 摘要 |
|---|---|---|
| 0 | Mode 1 構思期（含 Gotcha：單句概念催結構） | 「我想寫一個關於時間旅行的故事」 |
| 1 | Mode 1 卡住時 | 「主角到了城市但不知道接下來怎麼發展」 |
| 2 | Mode 2 完整審閱 | 200-300 字短篇大綱（內含 #19 巧合違規等多個結構問題） |
| 3 | 誤觸發測試 | 「幫我寫一封求職信，應徵後端工程師」 |
| 4 | 角色設計請求 | 「幫我設計一個有缺陷的反派角色」 |

## 各 Case 對照

| # | 測什麼 | With | Without | Delta |
|---|---|---|---|---|
| 0 | Mode 1 構思期 | 6/6 | 3/6 | +50% |
| 1 | Mode 1 卡住時 | 5/5 | 3/5 | +40% |
| 2 | **Mode 2 review** | **7/7** | **3/7** | **+57%** |
| 3 | 求職信誤觸發 | 4/4 | 4/4 | **0%（等價）** |
| 4 | 反派角色設計 | 6/6 | 2/6 | +67% |

## 三個關鍵發現

### 1. Skill 修正了 base model 的真實判斷錯誤

Eval 2 baseline 把測試大綱裡「警察剛好巡邏」這個明顯的 #19 違規（巧合解決衝突）**稱讚為**「設計得很聰明，讓主題自然呈現，而非說教」。Skill 版本則把這個列為「可以更好的部分」優先順序 #2，引用法則原文 *Coincidences to get them out of trouble are cheating* 直接打臉。

> 這證明 skill 不只是術語層的包裝——它修正了 base model 對結構問題的**真實誤判**。

### 2. 沒有 false positive 成本

Eval 3（求職信）兩組都 100% 通過，輸出品質等價。Skill 正確識別這不是說故事任務、不出手干預。代表**裝這個 skill 不會干擾使用者其他類型的請求**。

### 3. 對「直接創作請求」的 tradeoff

Eval 4（設計反派）baseline 直接給了一個完整角色（含名字、背景、行為對照表）；Skill 給的是引導性的法則 + 開放問題。

**這不是孰優孰劣，而是兩種模式**：
- 想「現成材料」 → baseline 風格更實用
- 想「自己思考過」 → skill 風格更貼合創作引導目的

這個發現已寫進 SKILL.md 的 Gotchas section，補了「對直接創作請求應額外給拋磚引玉的具體範例」一條。

## Tokens 多用了 8K，值得嗎？

With_skill 平均比 baseline 多用 ~70% token（主要花在讀 SKILL.md 與 references/22-rules.md），換到的是 +42.8% pass rate。對於需要結構性引導的創作任務，這個比例合理；對於誤觸發場景（Eval 3），skill 不會被啟用，所以沒有額外成本。

## 重現方法

測試定義：`skills/pixar-storytelling-rules/evals/evals.json`

執行流程使用 Anthropic 官方 [skill-creator](https://github.com/anthropics/skills/) 的 eval 工具：

1. 為每個 case spawn 兩個 subagent（with_skill / without_skill）
2. 收集 timing 與 outputs
3. 對每個輸出評分 28 條 assertion
4. 用 `aggregate_benchmark.py` 產生 benchmark.json/.md
5. 用 `eval-viewer/generate_review.py` 開啟對照 viewer

執行結果（grading.json、outputs、benchmark.json/.md）會放在 `skills/pixar-storytelling-rules-workspace/iteration-N/`，這個目錄已加入 `.gitignore`——eval run 結果是工作 artifact，不是 source of truth。

## 已知限制

- **單次執行**：每個 case 只跑一次，無法估計 variance。下一個 iteration 可改成每 case 跑 3 次以驗證穩定性
- **Assertion 偏向行為驗證**：檢查「有沒有引用 #14」「有沒有用模板」這類客觀條件，但「引導品質好不好」這種主觀面向需要人眼判讀（已透過 viewer 互動完成）
- **單一模型**：目前只在 sonnet 上測過。Opus 與 Haiku 表現可能不同
