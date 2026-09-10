# grill-evidence 完整範例

症狀：「我在 client log 看到 404。」使用者同時是 investigator 和 witness。
這是一次模擬，第一輪的答案是假設的。用途：讓模型看到每一輪該長什麼樣。

---

## 第一輪：找出 request 走的路

只問 path。沒有假設，沒有「How to get it」。

```
❓ Q1 - 看到 404 的 client 是什麼？
   1a. 它是哪一種？選一個：瀏覽器 / 手機 App / CLI / 另一個 service / cron job。
   1b. 如果是 service：它和目標 server 在同一個 cluster 嗎？
       它有 Istio sidecar 嗎？
       目標 server 有 Istio sidecar 嗎？
   1c. 這個 client 的程式碼是你們的，還是 vendor 的？

➡️ 為什麼要問：
   1a 決定 client 的 log 怎麼拿。
   1b 決定 request 有沒有走 mesh。跨 cluster 會多經過 east-west gateway。
       有 sidecar，Envoy 找不到 route 時會自己回 404。
   1c 是 vendor 的話，我們只能看 server 這一邊的證據。
```

```
❓ Q2 - request 從 client 到目標 server，中間經過哪些東西？
   照順序列出來。例如：client → load balancer → ingress gateway → sidecar → app。
   列出每一個你確定的。

🔍 不確定的部分不用猜。給我這三個東西，我自己查：
   - client 呼叫的 hostname 或完整 URL
   - 目標 server 所在的 cluster 和 namespace
   - 你的 kubectl context 名稱
   我會用 kubectl 查 Gateway、VirtualService、Service，把缺的補上。
   想自己查的話，跑這兩行，把輸出貼給我：
     kubectl get gateway,virtualservice -A | grep <hostname>
     dig +short <hostname>

➡️ 為什麼要問：中間每一個東西都可能是回 404 的那一個。
   路徑少一段，就少一個可能的原因。
```

```
❓ Q3 - 路上每一個東西的 log 放在哪裡？你能看哪些？
   一個一行。例如：
     client log → 在 Loki
     ingress gateway log → 在 Kibana
     app log → 在 Datadog
     load balancer log → 要問 infra team

➡️ 為什麼要問：之後每個查詢指令都要指到你看得到的地方。
   要向別的 team 拿的，先發出去，因為等最久。
```

```
❓ Q4 - 目標 server 的程式碼在哪個 repo？部署記錄在哪裡看？
   例如：GitHub URL，加上 Argo CD 或 GitHub Actions 的連結。

➡️ 為什麼要問：這題是給我用的。
   有 repo，我自己查 route 設定和最近的變動，不用問你。
```

### 證據帳本

```
## 已確認
- 某個 client 的 log 出現 404。來源：你說的。時間：還不知道。

## 已排除
（無）

## 還沒確定
- request 走的路還沒畫出來。畫出來以後，我再列可能的原因。
```

### Terms

- **client**：發出 request 的那一邊。看到 404 的程式。
- **cluster**：一組 Kubernetes 主機。`kubectl config get-contexts` 可以看到你有哪些。
- **namespace**：cluster 裡面的分區。`kubectl get ns` 可以列出來。
- **sidecar**：Istio 放在每個 Pod 旁邊的 Envoy proxy。`kubectl get pod -n <ns>` 看到 READY 是 `2/2`，通常就是有 sidecar。
- **load balancer**：cloud 提供的入口，把流量分給後面的機器。通常在 ingress gateway 前面。
- **ingress gateway**：Istio 在 cluster 邊界收外部流量的 Envoy。
- **east-west gateway**：Istio 專門處理 cluster 之間流量的 gateway。跨 cluster 才會經過。
- **VirtualService**：Istio 的 route 規則。決定某個 hostname 的 request 送去哪裡。

---

## 假設的第一輪答案

> **Q1** 1a: 另一個 service，`order-service` 呼叫 `payment-service`。1b: 不同 cluster，`c1` → `c2`，兩邊都有 sidecar。1c: 都是我們自己的程式碼。
> **Q2** `order-service` → sidecar → ？ → `payment-service` sidecar → app。hostname 是 `payment.payments.svc.cluster.local`。中間有沒有 east-west gateway 我不確定。kubectl context 是 `c1` 和 `c2`。
> **Q3** 兩邊 sidecar 的 access log 在 Kibana。app log 和 client log 在 Loki。
> **Q4** `github.com/example/payment-service`，部署用 Argo CD。

---

## 第二輪

### 路徑

```
order-service ─▶ sidecar(c1) ─▶ [跨 cluster，直連或經 east-west gateway] ─▶ sidecar(c2) ─▶ payment-service
```

中間那一段有空格。先試著自己查。

### 我自己查的

```
A1 - c1 到 c2 之間有沒有 east-west gateway；order-service 的 sidecar 有沒有 payment 的 endpoint
     嘗試：kubectl --context c2 get svc -n istio-system
     結果：失敗。這個 session 沒有 c2 的 kubectl context，只有 c1。
     → 改由你查。帳本記下：agent 缺 c2 的 kubectl 存取。

A2 - payment-service 最近的變動、Argo CD 部署時間
     嘗試：git log、Argo CD API
     結果：失敗。沒有 Argo CD 的 token；repo 不在本機。
     → 改由你查。帳本記下：agent 缺 repo clone 和 Argo CD 讀取權限。
```

> **注意**：A1 是 path 上的空格。依規則 3，正確做法是**單獨一輪只問 A1**，等答案回來再畫 tree。
> 下面為了示範完整流程，假設 A1 已經回來、答案是「same network，沒有 east-west gateway，直連」。

### 假設樹

| # | 假設 | 誰回的 404 | 能確認或排除它的證據 |
|---|---|---|---|
| H1 | c1 的 sidecar 找不到 `payment` 的 route 或 endpoint，自己回 404 | client sidecar | c1 sidecar log 的 `response_flags` 是 `NR`，`upstream_host` 是 `-` |
| H2 | request 到了 c2，但 c2 的 sidecar 找不到 route | server sidecar | c2 sidecar log 有這個 request，`response_flags` 是 `NR` |
| H3 | request 到了 app，app 回 404 | payment-service | c2 sidecar log 的 `upstream_host` 是 app 的 IP；app log 有這個 request id |
| H4 | client 打的 path 或 hostname 不對 | 任何一層 | log 裡的 URL 對照 payment-service 的 route table |
| H5 | request 去了別的 cluster 或別的 namespace 的同名 service | 錯的目標 | c2 sidecar log 沒有這個 request，但 c1 sidecar 有轉送出去 |

排序理由：跨 cluster 的 404 最常見是 H1，因為 endpoint discovery 出問題時 Envoy 就是回 404 NR。這是模型知識，只用來排序，不進帳本。

### 第二輪問題

```
❓ Q5 - order-service 那一行 404 log 的原文

🔍 在 Loki 查：
     {app="order-service"} |= "404" |= "payment"
   把那一行整段貼上。要有：method、完整 URL、時間（含時區）、x-request-id。
   如果 log 沒有印 x-request-id，告訴我。

➡️ 決定什麼：x-request-id 是把三段 log 串起來的鑰匙。
   沒有它，Q6 和 Q7 就要用時間和 path 去對，比較不準。
```

```
❓ Q6 - c1 sidecar 對這個 request 的 access log

🔍 在 Kibana 查，index 是 c1 的 istio-proxy：
     x_request_id:"<Q5 的 request id>" AND response_code:404
   貼出這四個欄位：response_flags、upstream_host、upstream_cluster、route_name。
   欄位名稱是 Envoy 預設值。你們的 index 可能改過名字，找不到就貼整筆。

➡️ 決定什麼：
   response_flags 有 NR，upstream_host 是 "-" → H1 成立，H2、H3 排除
   upstream_host 是一個 c2 的 Pod IP → request 有送出去，看 Q7
   完全找不到這筆 → request 沒有經過 sidecar，看 Q5 的 URL 是不是打錯 host（H4）
```

```
❓ Q7 - c2 sidecar 對同一個 request 的 access log

🔍 在 Kibana 查，index 是 c2 的 istio-proxy：
     x_request_id:"<Q5 的 request id>"
   貼出：response_code、response_flags、upstream_host。找不到就說找不到。

➡️ 決定什麼：
   找不到 → request 沒到 c2。配合 Q6，是 H1 或 H5
   有，response_flags 是 NR → H2
   有，upstream_host 是 app 的 IP，response_code 404 → H3，第三輪去看 app log
```

```
❓ Q8 - 404 的範圍和時間

🔍 在 Kibana 查 c1 istio-proxy：
     response_code:404 AND upstream_cluster:*payment*
   時間拉到最近 7 天。貼出：
     - 第一次出現的時間
     - 一天大約幾筆
     - 只有 order-service 在打，還是別的 service 也有
   再查一次 c2 內部的 caller 有沒有 404：同樣的查詢，index 換 c2。

➡️ 決定什麼：
   只有跨 cluster 的 caller 有 404，c2 內部沒有 → H1 或 H5，問題在 cluster 之間
   c2 內部也有 → H2、H3、H4，問題在 c2 這一邊
   某個時間點突然開始 → 對照 Q10 的部署時間
```

```
❓ Q9 - c1 的 sidecar 有沒有 payment 的 route 和 endpoint
   （原本我要自己查，但我沒有 c1 的 istioctl 執行環境。）

🔍 在你的機器上跑這三行，把輸出貼上。指令沒有跑過，跑不動就連錯誤一起貼：
     POD=$(kubectl --context c1 -n orders get pod -l app=order-service -o name | head -1)
     istioctl --context c1 -n orders proxy-config routes $POD | grep payment
     istioctl --context c1 -n orders proxy-config endpoints $POD | grep payment

➡️ 決定什麼：
   routes 沒有 payment → H1 確認，原因是 c1 不知道這個 service
   routes 有，endpoints 是空的 → H1 確認，原因是 endpoint discovery 沒有拿到 c2 的 Pod
   兩個都有 → H1 排除
```

```
❓ Q10 - payment-service 最近的變動
   （原本我要自己查，但我沒有 repo 和 Argo CD 的存取。）

🔍 貼出這兩個：
     git -C <payment-service repo> log --since="14 days ago" --oneline -- '*route*' '*handler*' '*virtualservice*' '*.yaml'
     Argo CD 上 payment-service 最近三次 sync 的時間

➡️ 決定什麼：Q8 的「第一次出現」時間如果貼近某次 sync，直接看那次 diff。
```

### 證據帳本

```
## 已確認
- order-service（c1）呼叫 payment.payments.svc.cluster.local（c2）時看到 404。來源：你說的。
- 兩邊都有 Istio sidecar。來源：你說的。
- c1 到 c2 是 same network 直連，沒有 east-west gateway。來源：你說的（A1 那一輪）。
- sidecar log 在 Kibana，app log 在 Loki。來源：你說的。

## 已排除
（無）

## 還沒確定
- H1 c1 sidecar 找不到 route 或 endpoint。等 Q6、Q9
- H2 c2 sidecar 找不到 route。等 Q7
- H3 app 回 404。等 Q7
- H4 client 打錯 URL。等 Q5
- H5 request 去了錯的目標。等 Q6、Q7、Q8

## agent 查不到的（下次補權限可以省一輪）
- c2 的 kubectl context
- payment-service 的 repo clone
- Argo CD 讀取 token
```

### Terms

- **x-request-id**：Istio 給每個 request 的編號。同一個 request 在每一層 log 都是同一個值。
- **response_flags**：Envoy 在 access log 裡寫的失敗原因代碼。`NR` 是 No Route：找不到可以送的地方。
- **upstream_host**：Envoy 把 request 送去的目標 IP。`-` 表示沒有送出去。
- **istioctl proxy-config**：直接看某個 Pod 的 sidecar 現在有哪些 route 和 endpoint。這是 sidecar 腦子裡的實際內容，比看 YAML 準。
- **endpoint discovery**：Istio 讓 c1 的 sidecar 知道 c2 有哪些 Pod 的機制。跨 cluster 404 最常出在這裡。

---

## 這個範例展示的規則

| 規則 | 在哪裡看到 |
|---|---|
| 先畫 path | 第一輪只有 Q1–Q4，沒有假設 |
| path 不接受「不知道」 | Q2 的 🔍 把「不確定」轉成三個給 agent 的輸入，或兩行指令 |
| 查不到的空格單獨一輪 | A1 之後的「注意」 |
| 第一桶先試、失敗掉第二桶、不猜 | A1、A2 → Q9、Q10；帳本的「agent 查不到的」區 |
| STE 句型、深度保留 | Q1 的 1a/1b/1c |
| 術語英文、Terms 區 | 每輪最後 |
| ➡️ 是預測不是建議答案 | 每題的「看到 X → H1，看到 Y → H2」 |
| 模型知識只排序不進帳本 | 假設樹下方的「排序理由」 |
| 指令沒跑過要標 | Q9 的 🔍 |
