# 08 Agent Native 开发

- 影片:[Agent-native 开发 [08-Raw/26生成式软件工程/NJU] (1:37:10)](https://www.bilibili.com/video/BV11LWw6VEHW)
- 講義:[lect8](https://jyywiki.cn/GSE/2026/lect8.md) · [slides8.html](https://jyywiki.cn/GSE/2026/slides8.html)

---

## 核心論點

- 總要在某個時刻開始做原型:符合 spec 不代表 spec 裡的假設成立,原型讓隱含假設變成可觀察的結果。
- Workspace 是狀態,Agent 是狀態遷移。只要每次遷移都檢查,就能讓任務比任何一次對話都長。
- 開發者是不可信的,agent 更是。把「好」寫成可執行的條件,由 CI 攔下、agent 修正。
- 不用寫程式了,但環境、約束、回饋和驗證機制還是要有人設計。

## 1. Ralph Loop:狀態機觀點

- 做法:同一份 prompt 反覆餵給 agent,每輪重新開始對話,進展靠工作區裡的檔案留給下一輪。Prompt 不變,但世界在變。
- 迴圈本身不會判斷「無法再改進」,何時停止要另外約定。
- Everything is a state machine:需求、文件、部分實作、測試、日誌、TODO 都是狀態;每次開發是一次狀態遷移。
- 版本控制讓狀態可以分叉、可以並行,但並行任務之間的依賴仍要協調。
- Long-horizon task 的前提:下一輪能從留下的狀態繼續。

## 2. 每次遷移都檢查,等於發明了 CI/CD

- 我們希望每個狀態都是「好」的,所以在每次 transition 做檢查:能構建、舊功能沒壞、符合專案約定。沒問題還可以直接發布。
- 三個名詞:

|  | 意思 |
| --- | --- |
| Continuous Integration | 頻繁整合進主幹,每次都自動構建與驗證 |
| Continuous Delivery | 保持隨時可發布,何時上線由人決定 |
| Continuous Deployment | 通過檢查的變更自動上線 |

- 機制很簡單:為事件(commit、push、PR)配置動作。難的是瑣碎的業務規則:合併前要驗證最新組合、偶發失敗誰來修、緊急修復誰能 override、怎麼避免舊版覆蓋新版。
- 檢查要放在有約束力的位置:本地 pre-commit 可以被 `--no-verify` 跳過;遠端檢查加上 required checks 與分支保護,失敗才真的擋得住合併。
- 現在 agent 可以寫配置,但「失敗怎麼辦、誰有權例外」仍然是需求的一部分,要寫進 spec。
- 驗收方式:故意製造一次 lint 錯誤、一次測試失敗,確認真的合不了、發不出。

## 3. CI 在 agent 時代的價值

- Agent 有自己的偏執:一寫就是 20 KB 文件、事無巨細堆出冗餘、在細枝末節瘋狂出力、寫出人讀不懂的程式碼。只要專案還要人共同維護,就必須擋下來。
- 類比豐田的 jidoka:發現異常就停線,把品質檢查放進生產過程。
- 底線:負責改程式碼的 agent,不能單方面降低過關標準。

> 📷 圖:把 Agent 的偏執變成可執行的約束:常見問題、自動檢查、人工判斷(見 [slides8](https://jyywiki.cn/GSE/2026/slides8.html) 第 13 張)

## 4. 業界怎麼做

| 來源 | 做法 | 學到什麼 |
| --- | --- | --- |
| OpenAI(Harness engineering) | 從空倉庫開始。`AGENTS.md` 當目錄,設計、計畫、決策都放進倉庫;每份工作區能獨立啟動應用,agent 自己操作頁面、查日誌、重現 bug | 人腦中的工程知識要外部化成 agent 找得到的版本化狀態;讓 agent 直接觀察程式行為 |
| OpenAI(同上) | 依賴方向用 linter 檢查,報錯訊息直接附修復指引;後台 agent 定期掃描偏離規則的程式碼,提小型重構 PR | 把架構品味做成可檢查的機制;技術債持續小規模清理 |
| Stripe Minions | 工作流混合兩類節點:實作與修 CI 交給 agent,跑 linter、推分支用普通程式碼。完整 CI 最多跑兩輪,之後交回人類審查 | 固定工序不必用模型;重試的收益會遞減 |
| Anthropic(C 編譯器實驗) | 16 個 agent 編譯 Linux 時全撞上同一個 bug。改成大部分檔案用 GCC,只把不同子集交給新編譯器 | 並行需要不同的任務;驗證方式會改變任務的可分性 |
| StrongDM | 端到端場景放在程式碼庫之外,像留出的考題;為 Slack、Okta 等外部服務造行為替身 | 測試全綠可能只是 agent 在「改考卷」(return true、改測試遷就程式碼);驗收環境也要造出來 |

## 5. 經驗無法憑空想出

- 真實的負載、依賴和異常,才會暴露原型裡隱藏的設計假設。失敗讓你理解架構該往哪裡修。
- 持續清技術債不夠:可能一直在維護一個已經失效的設計假設。
- Friendster:工程師看出要按用戶分片,但組織推不動,同一套設計後來在 Flickr 落地。及時重構既要看清架構,也要組織改得動。
- Webvan(倉配規模與訂單密度失配)、Quibi(需求假設不成立):好的想法也會敗在沒驗證的假設上,而且重資產讓每一輪試錯更貴。
- 推論:好的設計要做很多次、見很多個才學得會。沒有 token 就沒有多次嘗試的機會,實踐機會的差距會累積成判斷力的差距。

延伸閱讀:Huntley, Ralph Wiggum loop;Humble / Farley, Continuous Delivery;OpenAI, Harness engineering;Toyota Production System(jidoka)
