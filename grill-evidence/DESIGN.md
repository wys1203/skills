# grill-evidence 設計紀錄

每一條規則都是從一個具體失敗長出來的。要刪或改某條規則之前，先看它擋的是哪個坑。

格式：**規則** → 產生它的失敗 → 不這樣做的後果。

## 1. 先畫 path，再列 hypothesis tree

**失敗**：第一版第一輪就問「貼出 DevTools → Network 的那個請求」。使用者說 client 可能不是瀏覽器。
**後果**：「How to get it」寫給錯的環境，使用者可能照做卻拿錯證據，而且不會察覺。更深一層：tree 的分支是照 path 上的 hop 列的（誰回的 404），path 不知道，tree 就會有不存在的分支、缺真正存在的分支。例如 SPA router 只在瀏覽器有；Envoy `404 NR` 只在有 sidecar 時有。

## 2. path 問題不接受「不知道」

**失敗**：第二版 Q2 寫「不確定的 hop 直接標不確定」。使用者指出：client 到 server 的鏈路不知道，這題等於沒問。
**後果**：path 有空格 = tree 少一個分支。「不知道」在 `grilling` 裡是合法答案，因為那是決策；在證據蒐集裡，「不知道一個 hop」是一個待查的事實，要嘛 agent 查，要嘛給使用者指令查。

## 3. agent 查不到的 path 空格，單獨成一輪

**失敗**：模擬時 A1（查 c2 有沒有 east-west gateway）失敗，我把它塞進第二輪 Q9 第四行，旁邊都是證據題，然後帶著空格畫了 tree。
**後果**：如果 east-west gateway 存在，它也可能回 404，tree 少這個分支，第二輪的證據題有一部分要重問。多一輪來回，比對著錯的 tree 問一輪便宜。

## 4. 事實三桶；第一桶要先試，失敗掉到第二桶，絕不用猜的補

**失敗**：使用者問「現實環境 agent 常沒有權限，沒有 tool 可以查時 skill 怎麼辦」。原版把「agent 能查」當標籤而不是測試，查不到會卡住，或更糟：用模型知識補一個看起來合理的「事實」。
**後果**：猜出來的事實進了帳本，之後整棵 tree 都建在它上面。修法：明講試了什麼、為什麼失敗；改由 witness 查，附精確指令；帳本記下權限缺口。完全沒有 tool 時，skill 退化成手動問卷，仍然成立。

## 5. 問題用 ASD-STE100 寫

**失敗**：第二版的問題被使用者評「表述法太專業」。讀者可能是值班的人、junior、或 reporter。
**後果**：讀者誤讀問題 → 從錯的地方撈證據。STE 的四條：一句一事、20 字內、主動語態、指令用祈使句。

## 6. STE 限制句型，不限制深度

**失敗**：STE 版把 Q1 簡化成「這個程式是什麼」，丟掉了「同 cluster 或跨 cluster」「mesh 內或外」。使用者指出這些很重要。
**後果**：這些區分正是 tree 分岔的地方，丟掉就是把訪談變淺，不是變簡單。修法：每個會分岔的區分各自一句，或做成選項讓使用者勾。

## 7. 術語保留英文；不在句內解釋，每輪最後放 Terms 區

**失敗**：STE 版把 load balancer 翻成「負載平衡器」，使用者說反而難讀。同時，句內解釋術語會把問題本身埋掉。
**後果**：術語翻譯讓熟的人要反查、不熟的人也沒比較懂。Terms 區放最後：熟的人跳過，不熟的人不用猜。每個 Terms 附一個 hint（通常在哪裡、長什麼樣），例如「READY 是 2/2 通常就是有 sidecar」。

## 8. investigator 和 witness 分開

**失敗**：使用者問「拿去 grill reporter，是不是不能隨便讓他喊停」。原版結束條件之一是「使用者說證據夠了」，reporter 說夠了就會關掉調查。
**後果**：停止、決策、root cause 屬於 investigator；witness 只提供、確認、否認事實。witness 沉默不是結束，只是狀態不變。這條讓 primitive 可以被 `grill-reporter` 綁定不同的人，也是拆成 primitive + front door 的時機。

## 9. 帳本每條 Established 都要有來源；模型知識不進帳本

**失敗**：弱模型評估時預見的風險（尚未實際發生）：模型「知道」Istio 沒 route 會回 404 NR，直接寫進 Established。
**後果**：模型知識只能用來排序 hypothesis，不能當證據。沒有來源（使用者說的、tool 輸出、貼上的 log）的 Established 不算證據。

## 10. 沒跑過的指令標 untested，請 witness 連錯誤一起貼

**失敗**：弱模型評估時預見的風險：`istioctl` 的 flag、Loki/Kibana 的查詢語法版本不同，模型寫錯而使用者跑不動。
**後果**：一輪白費。標 untested 讓使用者有心理準備；連錯誤一起貼，下一輪可以修指令而不是重問。

## 11. grill-reporter：改完 description 要重讀，驗證分割線上方一字未動

**失敗**：弱模型評估時預見的風險：更新 description 時把 reporter 原文一起改掉，這是不可逆的破壞。
**後果**：reporter 的原文是最原始的證據，動了就沒了。read-modify-write 之後重讀比對，不一致就還原。

## 拆分的時機

一開始只有一個 skill。使用者問到 grill reporter 時，出現了第二個要用同一套機制、但角色綁定不同的情境，這時才拆成 `grill-evidence`（primitive）+ `grill-reporter`（front door）。這和 mattpocock 的 `grilling` / `grill-me` / `grill-with-docs` 是同一個模式。front door 第一行寫「Call the Skill tool with "grill-evidence"」而不是只提名字，因為 grilling 的文件說「提到另一個 skill 的名字不保證它被載入」是已知未修的問題。
