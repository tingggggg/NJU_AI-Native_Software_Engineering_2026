# 07 需求和架构 (2)

- 影片:[需求和架构 (2) [07-Raw/26生成式软件工程/NJU] (1:39:42)](https://www.bilibili.com/video/BV1JGap6LE74)
- 講義:[lect7](https://jyywiki.cn/GSE/2026/lect7.md) · [slides7.html](https://jyywiki.cn/GSE/2026/slides7.html)
- Lab:[Lab2: Deeper Research](https://jyywiki.cn/GSE/2026/lab2.md)

---

## 核心論點

- 軟體工程的本質是駕馭複雜性:essential complexity 來自問題本身,accidental complexity 來自實作方式。
- 需求工程直面 essential complexity;架構負責分解它,並減少 accidental complexity。
- 好的架構不是選出一個解,而是設計解空間:固定協作規則,保留局部選擇。
- 終點:把「狀態一致性問題」變成「計算正確性問題」。

## 1. Intent → Spec → Impl

> 📷 圖:一個意圖對應多種規格,每種規格又對應多種實作(見 [slides7](https://jyywiki.cn/GSE/2026/slides7.html) 第 1 張)

- 看似明確的需求其實很開放:一個意圖有多種規格,每種規格又有多種實作。未確認的假設會讓目標漂移。
- 需求越完備歧義越小,但每補一條約束,就多一條要持續核對的承諾(今天的代價是 token 與模型亂發揮)。
- 判斷架構好壞的實用問題:需求改一處,要跟著改多少處?

## 2. 讀經典,建立品味

- **Dijkstra(Structured Programming, 1972)**:人一次只能精確思考少量元素,所以要靠抽象分層。每項設計決策像項鍊上的一顆珠子,越晚做表示決策,越多層能獨立於它。
- **Steele(Growing a Language, 1998)**:比起設計一個東西,更好也更難的是設計一個 pattern。保留 slots,讓後來的人選擇與演化;哪些選擇留給後人,本身就是設計。
- 讀的時候拿自己的系統追問:作者把哪些判斷交還給了使用者?這些選擇需要什麼共同規則才能彼此相容?

## 3. 設計解空間:三個 Systems 案例

> 📷 圖:開放的需求經由架構規則,對應一族可演化的解(見 [slides7](https://jyywiki.cn/GSE/2026/slides7.html) 第 12 張)

| 案例 | 固定的共同規則 | 保留下來的自由 |
| --- | --- | --- |
| UNIX | 文字流是通用介面;POSIX 約定參數語法、stdin / stdout / stderr、退出狀態 | 使用者自由組合程式、隨時檢查中間結果,組出設計者沒想到的用途(jq、fzf) |
| 關係資料庫 | 邏輯關係與資料獨立性:分開「對資料的理解」與「資料在機器中的組織」 | 資料頁、索引、存取路徑都可以換;同一份資料回答不同問題 |
| Internet | 分層,只做好「資料傳輸」一件事 | 局部協定持續更換(Telnet → ssh、HTTP → HTTPS),整體照常運轉 |

> 📷 圖:現實世界、邏輯關係、物理組織三層:查詢關心事實,儲存可以改變(見 [slides7](https://jyywiki.cn/GSE/2026/slides7.html) 第 18 張)

- 這三個都是 Systems:為通用應用搭的抽象層。每一輪浪潮都有新抽象(System/360、UNIX、Smalltalk、Web、Android、Kubernetes)。

## 4. 面向客戶需求的軟體:Web 架構的演進

- **每個頁面一個 SQL transaction**:最直接,但規則被複製到每個頁面,改一次規則要找遍所有入口。
- **JSP**:在 HTML 裡嵌 Java,業務邏輯很容易順手寫進頁面;換成標籤也不會自動得到 MVC。
- **MVC**:讓不同的變化各有歸屬。Model 封裝資料、狀態與業務規則(不只是幾張表);View 負責展示;Controller 接收輸入並協調。課表從列表換成週曆主要改 View,選課限制變了主要改 Model。

> 📷 圖:Web MVC 的一次選課請求:Controller 組織請求,Model 執行業務規則,View 呈現結果(見 [slides7](https://jyywiki.cn/GSE/2026/slides7.html) 第 26 張)

- **把 ViewState 也當成 Model**:View = render(Model, ViewState)。篩選詞、展開的選單不進業務資料庫,但一樣值得建模。是否持久化,與是否值得建模,是兩件事。
- **MVVM(Vue)與 React**:都由狀態決定界面。Vue 用綁定宣告輸入與狀態的連接;React 每次 render 是狀態快照,在事件裡顯式更新。衍生資料都不另存。

> 📷 圖:同一個課程篩選框:Vue 的綁定與 React 的狀態快照(見 [slides7](https://jyywiki.cn/GSE/2026/slides7.html) 第 30 張)

## 5. MVC / CRUD 的極限與 Event Sourcing

- MVC / CRUD 適合業務物件穩定、一次請求只做一小段局部操作的系統(部落格、社群、課程資訊)。把複雜性塞進 Model,並沒有解決 essential complexity:長流程(下單到對帳)、強規則(畢業審核)照樣難。
- 記帳為什麼麻煩:「當前狀態」同時編碼了過去(合約已簽)、現在(等待打款)、未來(交貨、潛在壞帳)。
- Event Sourcing 的分解:

|  | 是什麼 | 性質 |
| --- | --- | --- |
| 過去 | 事實 | 不可更改 |
| 現在 | 視圖 | 由事實經 dependency 計算得到 |
| 未來 | 預期 | 不是事實;世界沒照預期走,就追加新的事實來矯正 |

- 顯式建模 data dependency:選課紀錄與成績 → 已修課程與學分 → 是否滿足畢業條件。整個過程可查證、可溯源,解決「反悔要改很多處,漏一處就崩」。
- 記帳的例子:原始憑證(事件)→ 分錄 → 狀態變化,由 LLM 把憑證編譯成可執行的分錄。借貸平衡可以檢查,但不代表業務真實。
- SQL 的缺憾:跨資料的任意計算關係沒有成為 first-class citizen。MVVM、React、Dataflow、Event Sourcing 都是在把 derived state 與 dependency 顯式化。
- 結論:保存足以重建其餘資訊的 Source of Truth,其他狀態盡量作為計算結果。
  - 計算可以重跑、測試、審計;規則錯了,修正後從某個時點起用新規則重算。
  - 重現當時的結果,需要當時的規則與外部輸入(哪一年的培養方案、哪一天的匯率)。
  - 重放可以重算餘額,不能再次打款:外部副作用要隔離。

延伸閱讀:Brooks, No Silver Bullet;Dahl / Dijkstra / Hoare, Structured Programming (1972);Steele, Growing a Language (OOPSLA '98);Codd (1970);RFC 1958
