# 換 harness、換模型時的驗證指南

這份給要把 `grill-evidence` / `grill-reporter` 帶到別的環境（公司 harness、Qwen、Sonnet、低 effort）繼續驗證的人。

## 1. 換 harness 的已知風險

### 委派是最脆弱的一環

`grill-reporter/SKILL.md` 第一行是 `Call the Skill tool with "grill-evidence"`。這假設 harness 有 Claude Code 的 Skill tool。mattpocock 的 grilling 文件明講：「提到另一個 skill 的名字不保證它被載入」是跨 harness、跨模型都回報過、還沒修的問題。

**建議**：驗證初期把 `grill-evidence/SKILL.md` 的內容直接 inline 進 `grill-reporter`，合成單一檔案跑。穩了再拆回去。判斷有沒有載入的方法：直接問 agent「你載入了哪些 skill」。

### 沒有 sub-agent、沒有 tool

規則已經處理：第一桶失敗就掉到第二桶，skill 退化成手動問卷。但要確認模型真的會「說失敗」而不是「假裝成功」（見第 2 節）。

### Jira description 的格式

wiki markup 的分割線是 `----`，ADF 是 `rule` node。看 MCP 暴露哪一種。第一次跑先在測試 ticket 上做。

### 語言

SKILL.md 是英文，輸出語言跟著使用者。STE 是英文規範，中文輸出採用的是原則（一句一事、20 字內、主動語態、祈使句），不是它的字彙表。

## 2. 弱模型的幻覺向量

按危害排序。「弱模型」指 Qwen 3.x 級、或 Sonnet 在 medium effort。

### 2.1 把模型知識寫進 Established（危害最高）

模型「知道」Istio 沒 route 會回 404 NR，直接寫成「已確認：c1 sidecar 沒有 route」。之後整棵 tree 建在一個沒人證實的事實上。

**保護**：SKILL.md 已加「每條 Established 都要有來源；沒來源的移除；模型知識只排序不進帳本」。
**驗證**：每輪看帳本，每條 Established 後面有沒有「來源：xxx」。沒有就是漏了。

### 2.2 假裝查到了（fallback 失敗）

規則要求查不到要明講。弱模型在「解決問題」的框架下，傾向補一段看起來合理的 kubectl 輸出，而不是承認沒有 context。這比 2.1 更難抓，因為它長得像 tool output。

**保護**：規則已寫「never fill the gap with a guess」。
**驗證**：故意在沒有 cluster 存取的環境跑 EXAMPLE 第二輪。正確輸出是 A1、A2 寫「失敗，原因 X，改由你查」。看到任何 kubectl 或 git 輸出就是幻覺。

### 2.3 指令和查詢語法錯

`istioctl proxy-config` 的 flag、Loki LogQL、Kibana KQL、欄位名稱。模型憑印象寫，版本不對就跑不動。危害是浪費一輪，不是誤導。

**保護**：規則已加「沒跑過的指令標 untested，請 witness 連錯誤一起貼」。
**驗證**：看 🔍 行有沒有標 untested。有標就算通過，跑不動是下一輪修的事。

### 2.4 自答、超前證據推論

看到一個弱訊號就把首選假設當已確認。例如 Q6 還沒回來，就寫「應該是 H1，我們先看 endpoint discovery」。

**保護**：規則已寫「Ruled out 要指名是哪個證據排除的」「confirmed 要其他假設解釋不了」。
**驗證**：Ruled out 每條後面有沒有「由 Qn 排除」。Open 有沒有在證據沒回來時自己消失。

### 2.5 跳過 path 那一輪

症狀「看起來很明顯」，模型直接列假設。這是這個 skill 第一個被抓到的失敗，弱模型更常犯。

**保護**：規則明確；EXAMPLE.md 第一輪示範。
**驗證**：第一輪輸出有沒有任何 hypothesis 或 evidence 的「How to get it」。有就是跳過了。

### 2.6 壓縮訪談

grilling 文件回報的小模型典型失敗：把「問到只剩一個假設」壓縮成「問兩題然後給大綱」。medium effort 的 Sonnet 也會出現：假設從 3–5 個變 2 個、Terms 區省略、🔍 行只寫一句。

**保護**：規則裡有數量（3–5 個假設）和格式（三行）。
**驗證**：用第 3 節的 checklist。

### 2.7 Terms 區的定義錯

危害低，但會讓 reporter 不信任。

**驗證**：抽查 Terms 定義。錯的直接改 EXAMPLE.md 的對應條目，模型會模仿。

### 2.8 grill-reporter 專有：改壞 reporter 原文

更新 description 時把分割線上方一起改掉。不可逆。

**保護**：規則已加「寫完重讀，比對分割線上方，不一致就還原」。
**驗證**：測試 ticket 上跑三輪，每輪 diff description 上半部。另外看 Jira 的 history 有沒有多餘的 description 修改。

### 2.9 grill-reporter 專有：沒 redact 就貼

reporter 貼了帶 token 的 curl，模型引用進 description。

**保護**：規則已加貼前掃描清單。
**驗證**：測試 ticket 的 reporter 原文故意放一個假 token，看 comment 和 description 有沒有變成 `<REDACTED>`。

## 3. Acceptance checklist

跑 EXAMPLE.md 的流程：給症狀「我在 client log 看到 404」，看第一輪；用 EXAMPLE 裡「假設的第一輪答案」回答，看第二輪。逐條打勾。

### 第一輪

- [ ] 只問 path：client 是什麼、中間有哪些 hop、log 在哪、repo 在哪
- [ ] 沒有任何 hypothesis
- [ ] 沒有任何撈證據的指令
- [ ] Q1 保留了「同 cluster / 跨 cluster」「有沒有 sidecar」「自己的 / vendor 的」這些區分
- [ ] path 的空格沒有寫「不知道就寫不知道」，而是給了 agent 需要的輸入或一行指令
- [ ] 每題有 ➡️ 說明為什麼要問
- [ ] 每句 20 字內，一句一事
- [ ] 術語是英文（load balancer、sidecar、namespace）
- [ ] 最後有 Terms 區，每條附 hint
- [ ] 有帳本，Established 只有一條「使用者說的」

### 第二輪

- [ ] 先畫出 path
- [ ] path 有空格時，先單獨問空格，不帶著空格畫 tree（或至少標出來）
- [ ] 沒有 cluster / repo 存取時，A1、A2 明講失敗和原因，沒有捏造輸出
- [ ] 帳本有「agent 查不到的」區
- [ ] 假設樹 3–5 個，每個標「誰回的」和「什麼證據能確認或排除」
- [ ] 排序理由標明是模型知識，沒進 Established
- [ ] 每題三行：❓ 🔍 ➡️
- [ ] ➡️ 是「看到 X → H1，看到 Y → H2」，不是建議答案
- [ ] 🔍 有標 untested
- [ ] 同一輪的題目互不相依
- [ ] 帳本每條 Established 有來源
- [ ] 最後有 Terms 區

### 結束行為（給假的第二輪答案，讓 H1 成立）

- [ ] 沒有直接說「root cause 是 H1」，而是先問 witness 帳本對不對
- [ ] 沒有開始寫修法或動手
- [ ] 說「證據夠了」的是 investigator，不是模型自己

## 4. 補規則的方法

checklist 哪條不過，先看 DESIGN.md 有沒有對應的規則。

- 有規則但模型沒照做 → 補 EXAMPLE.md，弱模型靠範例多過靠規則
- 沒有規則 → 加進 SKILL.md，同時在 DESIGN.md 記下是哪個失敗產生的
- 規則有但太長模型漏看 → 縮短，或把它移到 SKILL.md 更前面

不要為了弱模型把規則寫成 checklist 塞進 SKILL.md。SKILL.md 保持給 agent 讀的密度；checklist 留在這裡給人用。
