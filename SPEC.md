# **馬拉松賽事與社群互動連結系統：需求規格與系統架構書**

## **0. 文件關係與版本資訊**

本文件為本專案之**唯一實作級規格來源（Single Source of Truth for Implementation）**，所有工程師、QA、後台管理員、第三方整合商於開發、測試、上線時應以本文件為準。

### **0.1 與其他文件的關係**

| 文件 | 角色 | 維護狀態 | 關係 |
|:----|:----|:----|:----|
| **`proposal.md`**（同 repo） | 原始提案與市場調研 | **reference-only, archived** | 本 SPEC 整合自 proposal.md 之市場分析、目標客群、技術選型建議章節；proposal.md 之內容已併入本 SPEC §1 / §2 / §5，不再單獨維護。如需查閱歷史脈絡，請參見 `git log -- proposal.md` |
| **本 SPEC.md** | 實作級規格 | **active, 持續修補** | 所有開發實作以此為準 |
| **未來：`docs/api/openapi.yaml`** | API 自動產出（規劃中） | planned | §3.5 OpenAPI 規範之自動產出 |

**重要：** 本 SPEC 內**無「詳見 proposal.md」之反向連結**，因 proposal.md 為歸檔狀態；如需引用具體提案數據（如「2024 全球馬拉松參賽人數 580 萬」），請查閱 git history `git log --all -- proposal.md` 或當前 SPEC 內已併入之段落。

### **0.2 版本歷史**

| 版本 | Commit | 日期 | 主要變更 |
|:----|:----|:----|:----|
| v0.9 | 6c94ca2 | 2026-06-XX | feat: incorporate proposal.md suggestions - Taiwan market, aesthetic scoring, IG story templates |
| v1.0 | 167d5bc | 2026-06-XX | A7 fixes: Admin API補足 GET/PUT/DELETE events、oauth/callback已存在確認 |
| v1.1 | d524858 | 2026-06-XX | fix: resolve audit findings - duplicate Section 2.8, add missing schemas and flows |
| v1.2 | 66ac7fb | 2026-06-XX | docs(SPEC): repair 34 real issues across 7 sections |
| v1.3 | 70a17ec | 2026-06-XX | feat(SPEC): P0~P2 deep-research 衍生優化 |
| v1.4 | (current) | 2026-06-27 | chore(SPEC): audit fixes — F-1/F-2/F-3/F-4 + S-1/S-2/S-3/S-4/S-5/S-6 (詳見報告) |

### **0.3 本次審計（P0/P1 Round 1）變更摘要**

詳見 commit message；高層級摘要：

- **F-1 (致命):** 統一 OCR 置信度閾值定義 — Cascade 內部 `stageThresholds` 與全域 `confidenceThreshold` 分離
- **F-2 (致命):** §4.8 區分 `ocrSuccessRate`（OCR 成功率）與 `runnerActivationRate`（跑者啟用率），避免混用
- **F-3 (致命):** Photo Processing Lambda Timeout 60s → 240s（與 Cascade `globalTimeoutMs` 一致）；Face Re-ID Lambda 45s → 120s
- **F-4 + S-1 (致命+嚴重):** §3.6 OAuth Token 完整化 — 補 DynamoDB `OAuthTokenRecord` schema、6 狀態狀態機、KMS envelope 流程、3 個 GSI 索引、跨章節關聯表
- **M-3 (中度):** §4.8 Lambda 成本算式重算（單位更正 per GB-second，賽事日總成本 $72.65 → $16.03）
- **S-2 (嚴重):** Threads 250 限流常數於 §2.8 / §3.9 統一引用說明（待 v1.5 抽為 config field）
- **S-3 (嚴重):** §4.2 延遲預算表加註排除項（SQS wait / X-Ray overhead）
- **S-4 (嚴重):** §3.5 Rate Limit 雙層優先序說明（API Gateway 為粗粒度先驗，Redis Token Bucket 為細粒度 AND 關係）
- **S-5 (嚴重):** pushMaxPerEvent 註解加（叢集 dedup 後）+ §4.8 計算公式對照
- **S-6 (嚴重):** 本 §0 文件關係章節（proposal.md reference-only 標記）

### **0.4 待決議問題（Open Questions，v1.5 新增）**

本節列出**目前尚未定案、需後續討論**之設計決策。每一項應於下次 Sprint Planning 前由 Product Owner + Tech Lead 共同決議；決議後應從本表移除並補入對應章節之正式規範。

| # | 問題 | 影響章節 | 建議方向 | 預計決議時間 |
|:----|:----|:----|:----|:----|
| OQ-1 | Threads 250 限流常數是否抽為 `EventFeatureConfig.publishing.threadsHardLimitPer24h` config field（v1.5 §2.8 L634 + §3.9 L2302 兩處引用）？ | §2.8 / §3.9 | 抽為 config field（建議） | v1.5 sprint 2 |
| OQ-2 | 是否引入 cold start 預算重算（M-9，§4.2 F-3 patch 後 cold start budget table 須重算）？ | §4.2 | 重算並更新 Provisioned Concurrency 配置 | v1.5 sprint 1 |
| OQ-3 | AWS Lambda 計價單位於不同 region 是否一致？§4.8 算式重算（M-3）以 `ap-northeast-1` 為基準，是否需補 `ap-southeast-1` 對照？ | §4.8 | 補 DR region 對照表 | v1.5 sprint 3 |
| OQ-4 | §5.4 短影音是否真要共用 `publish-queue`（M-8 結論）？或應該分離？需 spike 驗證 message payload size（影片 metadata 較大）對 SQS 256KB 限制的影響 | §5.4 | 待 spike | v2.0 前 |
| OQ-5 | Cluster Key 是否需強制 `bibNumber` 字元集限制為 `[A-Za-z0-9-]`（m-3）？由報名表單驗證還是後端 sanitize？ | §3.7 | 報名表單前端驗證 + 後端 Zod schema 雙重把關 | v1.5 sprint 1 |
| OQ-6 | §2.11 全目錄彙總（M-7）是否納入 release tooling 自動生成？目前手動維護易漏 | §2.11 | 從 `EventFeatureConfig` TypeScript 定義自動產出 markdown 表格 | v1.6 |

**Open Questions 流程規範：**
- 新增問題：於下次 audit 或 PR review 時主動加入
- 移除問題：決議後移除，並於對應章節補入正式規範
- 升級為 Finding：若發現影響實作正確性（例如 OQ-2 cold start 預算錯誤），升級為 spec audit finding

---

## **1\. 專案概述 (Project Overview) & 核心價值**

在大型路跑與馬拉松賽事中，參賽者的數位體驗已成為賽事品牌經營的核心要素。然而，傳統馬拉松賽事的影像紀錄與分發流程，長期面臨效率低落與使用者體驗不佳的技術瓶頸。

就市場現況而言，目前全球已存在若干基於 AI 技術之賽事攝影服務（如 RaceTagger、Photohawk、Roboflow 等）。然而，現有方案多止於「AI 標記輔助攝影師整理」，均未實現將照片全自動推送至跑者個人社群帳號之最後一哩路。本系統之核心差異化定位在於：**從攝影到社群平台的最後一哩自動推播層**，此功能在全球及台灣市場中均屬缺口。過去數十年來，賽事攝影師多半依賴光學字元辨識（Optical Character Recognition, OCR）技術掃描跑者號碼布，或者純粹仰賴事後耗時的人工標記1。在真實的賽事環境中，號碼布經常因為跑者的肢體動作而彎折，或是被雨衣、外套遮蔽，甚至被汗水與泥漿覆蓋，這些現實環境干擾使得傳統 OCR 的辨識率大幅降低1。當辨識失敗時，成千上萬張未經整理的原始照片往往被直接上傳至雲端硬碟，迫使跑者必須在海量圖庫中進行漫長且令人沮喪的檢索，此現象被業界稱為「圖庫疲勞」（Gallery Fatigue）1。這不僅導致跑者錯失了在完賽當下即時分享成就的黃金時機，也讓賽事主辦單位流失了寶貴的社群曝光動能。  
本專案旨在開發名為「馬拉松賽事與社群互動連結系統」的創新數位服務，透過深度整合邊緣運算、雲端無伺服器架構（Serverless Architecture）、深度學習物件偵測與文字萃取模型，以及主流社群平台的應用程式介面（API），徹底翻轉傳統賽事影像的交付與傳播模式。系統的核心價值在於提供極致的「即時滿足感」（Instant Gratification），將分類與標記的沉重負擔從人類轉移至機器與演算法1。當跑者通過攝影點位後，系統將以全自動化的串流工作流程擷取影像、進行雲端人工智慧解析、動態套用賽事專屬視覺化數據（如配速、完賽時間、贊助商相框），並在極低的延遲內，將美化後的專屬照片推播至跑者事先授權的個人社群平台（包含 Instagram、Threads、Facebook 與 LINE）。  
此全自動化的數位轉型不僅能極大化跑者的參與感與專屬尊榮感，更能為賽事主辦單位與贊助商創造指數型的社群曝光度。透過跑者社群網絡的病毒式傳播，賽事品牌的數位足跡將以自動化推播的方式達到最大化。同時，系統於賽後自動生成的「個人專屬完賽報紙」，深度結合了無線射頻辨識（RFID）晶片計時數據、精選影像與社群互動指標，進一步將參賽的瞬間感動昇華為具備長久紀念價值的客製化數位資產。

#### **台灣市場競爭格局**

台灣賽事攝影市場已有本地深耕逾 10 年的競爭者，以下為主要廠商與其定位差異：

| 服務商 | 規模 / 定位 | 強項 | 跑者痛點 |
| :---- | :---- | :---- | :---- |
| **AllSports（創星影像）** | 全台最大，單場臺北馬 68 萬張照片 | 全台最廣攝影師網絡 | 68 萬張中找一張宛如海底撈針；單張 NT$80–200；肖像權爭議 |
| **Sportag 運動標籤** | FB 34,749 粉絲，AI 人臉+號碼布雙軌 | 自拍照搜尋 | 找得到照片但無自動推播；仍須自行下載分享 |
| **RaceShot 運動拍檔** | AI 辨識，號碼布/服裝/GPX 多維搜尋 | 搜尋體驗佳 | 同樣無自動推播；付費下載 |
| **Phomi 瘋迷** | 7-11/全家/萊爾富機台列印明信片 | 實體紀念品價值 | 無自動推播 |
| **好拍 gooodshot** | 老牌賽事照片平台 | 長期配合攝影師 | 無自動推播 |

> **市場定位說明：** 台灣市場對「找到自己的賽事照片」已有成熟解決方案（AllSports、Sportag），但均止於「被動搜尋」而非「主動推播」。本系統的核心差異化是：**從號碼布 AI 辨識到個人社群帳號的零接縫自動推播**，以及結合「數位美照 + 實體紀念品」的完整體驗鏈。

## **2\. 系統架構與資料流 (System Architecture & Data Flow)**

為確保系統在數萬名跑者同時參賽、數十位專業攝影師密集上傳高畫質影像的極端負載下，仍能維持毫秒至秒級的低延遲與系統高可用性，本系統採用基於雲端原生（Cloud-Native）的事件驅動無伺服器架構（Event-Driven Serverless Architecture）2。整體的拓樸結構被精密劃分為四個主要層級：資料獲取與攝入層、非同步串流緩衝與處理層、人工智慧推論與渲染層，以及社群發布與展現層。  
### **2.1 資料獲取與攝入層**

在資料獲取層中，賽道沿線的專業攝影設備透過 5G 路由器或專線網路，將高解析度影像即時且持續地推送到雲端物件儲存空間（如 Amazon S3）3。本系統採用賽道定點攝影模式：賽前公告固定攝影點位，跑者預期在已知位置獲得拍攝服務。賽事主辦單位將提供攝影師完善硬體設施（5G 熱點、供電、遮陽/遮雨帳篷），確保上傳穩定性3。同步地，位於起終點與各個分段檢查點的 RFID 計時系統（例如 MyLaps 系統或搭載 UHF 被動式 RFID 天線的讀取器），會捕捉跑者鞋面或號碼布上的晶片數據4。本系統透過「外部計時系統整合抽象層」之 Adapter Pattern 接收資料（詳見 2.7 章節），以統一的內部資料模型隔離各系統 API 差異，無論計時系統供應商為 MyLaps、ChampionChip、RaceEntry 或其他，均能無差異地流入下游處理管線。  
### **2.2 非同步串流緩衝與流量調節層**

當影像檔案寫入 S3 儲存體時，會立即觸發 S3 事件通知機制，將該影像的處理任務物件推入 Amazon SQS（Simple Queue Service）訊息佇列中3。引入 SQS 作為非同步與佇列點對點處理（Queued point-to-point processing）的關鍵在於解耦與削峰填谷；它允許下游的無伺服器運算單元以自身的最佳節奏擷取任務，避免瞬間的巨量影像上傳壓垮後端運算資源與資料庫連線3。

#### **SQS 佇列類型與去重策略**

本系統所有任務佇列（`photo-processing-queue`、`face-reid-queue`、`publish-queue`）皆採用 **SQS Standard Queue**（非 FIFO），以取得最高 throughput 與最低成本；代價是 SQS 提供 **at-least-once delivery 語意**，可能因 Lambda 執行超時或 Visibility Timeout 期間容器被回收，導致同一訊息被重複消費。

**應用層去重機制：**

```typescript
// 應用層冪等性寫入範例(DynamoDB Conditional Check)
async function markPhotoProcessed(taskId: string, payload: PublishTask): Promise<void> {
  await dynamodb.send(new PutCommand({
    TableName: 'PhotoProcessingTask',
    Item: { taskId, ...payload, updatedAt: new Date().toISOString() },
    ConditionExpression: 'attribute_not_exists(taskId)',  // 已存在則拋 ConditionalCheckFailedException
  }));
}
```

- **`taskId` 作為隱含 Idempotency Key** — 由 S3 Object Key（`s3://{env}-imarathon-photos/{eventId}/station_{N}/{ts}_{seq}.jpg`）衍生,於 SQS Message Body 內攜帶。Lambda 寫入 DynamoDB Task 表時以 `attribute_not_exists(taskId)` Conditional Check 防重複。
- **Publish Lambda 去重** — DynamoDB `runnerId-platform-photoId` 複合鍵作為 Conditional Check key,避免同一照片重複推播至同一平台（與 §3.7 `GROUP_PHOTO_DEDUP` Feature Flag 配合）。
- **DLQ 寫入去重** — 失敗任務寫入 DLQ DynamoDB 表時亦以 `taskId` 為主鍵,3 次 SQS 重試期間若已寫入 DLQ 則不再重複。
- **副作用控管** — 推播至外部平台 API 為不可逆副作用,故 Publish Lambda 之 adapter 內部一律先 Conditional Write 標記 `publish_attempted=true` 後再呼叫平台 API;失敗時下次消費會看到已標記而跳過。

#### **SQS 佇列設定參考表**

| 設定項 | 建議值 | 說明 |
| :---- | :---- | :---- |
| Visibility Timeout | 6 × Lambda Timeout | AWS 最佳實踐,避免 Lambda 執行中超時被重複消費 |
| Message Retention Period | 4 天 (345,600 秒) | SQS 最大值;賽事後 DLQ 處理窗口足夠 |
| maxReceiveCount（redrive 至 DLQ 閾值） | 3 | 配合 §3.6 之指數退避重試策略 |
| Receive Request Wait Time | 20 秒 | Long polling,降低空 poll 成本 |
| DLQ Retention | 14 天 | DLQ 任務人工處理 SLA 上限（§3.7） |

### **2.3 AI 推論與影像渲染層**

接下來，SQS 將觸發具備高度彈性擴展能力的運算服務（如 AWS Lambda），啟動核心的影像處理與人工智慧管線。Lambda 函數會呼叫預先訓練好的邊緣或雲端物件偵測模型（如 YOLOv8 或 RF-DETR），精確進行跑者身軀與號碼布的邊界框（Bounding Box）定位，隨後交由 OCR 引擎提取字元2。辨識成功後，影像將傳遞至基於 Node.js 的 Sharp 影像處理模組，進行浮水印、濾鏡與賽事數據的動態疊加渲染10。

#### **競品技術對標與護城河設計（§2.3.X）**

基於 deep-research 對全球賽事照片服務商之 3-vote 對抗驗證(15 確認/10 駁回),Photohawk 為目前技術最接近之競品(OCR + Face Recognition + B2B SaaS),但商業模式卡在「銷售端」,未進入「消費者端自動推播」。本節定義對 Photohawk 等競品之差異化設計與護城河(moat)建構策略,供 §2.10 推論抽象層、§2.8 推播引擎、§3.8 後台定價之設計參考。

**對 Photohawk 的差異化：**

| 維度 | Photohawk | 本系統 (SPEC.md) |
|:----|:----|:----|
| **商業模式** | B2B SaaS(向賽事主辦方收月費+上傳費+銷售佣金 13.5%–25%) | B2C 為主(跑者自助授權)+ B2B 為輔(§3.8.X 三層定價) |
| **核心交付** | 照片搜尋+銷售平台(賽事主辦方自有網域) | 自動 OAuth 推播至跑者個人 IG/LINE/Threads 帳號 |
| **定價結構** | Pay As You Go $0/月 / Lite $16.99 / Pro $29.99 / Enterprise $44.99 | Free / Standard NT$50,000 / Premium NT$200,000+(見 §3.8.X) |
| **覆蓋規模** | 宣稱 29 國/450+ 會員/1.2 億張照片(後者為公司行銷數字,未經第三方稽核) | 台灣優先 + 日本 + 東南亞 |
| **Face Recognition** | 跑者上傳自拍後 AI 從賽事照片庫找出本人 | 同(§3.2 Face Re-ID Fallback)+ 自動推播至本人社群 |
| **與原圖所有者之互動** | 賽事方管理,跑者被動搜尋 | 跑者主動授權後,系統自動推播 |

**核心介面定義：**

```typescript
interface CompetitiveDifferentiation {
  eventId: string;

  // 對主要競品之差異化設計
  vsPhotohawk: {
    b2cVsB2b: 'b2c';                        // 跑者自助,非賽事方付費為主要商業模式
    oAuthAutoPublishing: true;              // Photohawk 無此功能 — 本系統核心護城河
    mobileNativeFlow: true;                 // LIFF SDK 在 LINE App 內一鍵授權 — Photohawk 無 LINE 整合
    jurisdictionAwareConsent: true;        // §3.1.X 多司法管轄區合規 — Photohawk 無此設計
    withdrawalCapability: true;            // §3.1.Y Post-Push Withdrawal — Photohawk 無撤回機制
  };

  // 對 Pic2Go / MYLAPS RunnerTag / Bhaago India 等已知有自動推播之服務
  // (deep-research 未深入驗證,假設其商業模式為 B2C;此處預留差異化欄位)
  vsOtherAutoPushServices: {
    multiJurisdictionCompliance: true;      // §3.1.X — 多司法管轄區合規
    dropInArchitecture: true;               // §2.6 多租戶隔離 + §2.8 社群平台抽象 — 第三方賽事可快速接入
    dualConsentWithdrawal: true;            // §3.1.Y — 雙重確認 + 撤回
  };

  // 必須在 MVP 前建立的護城河(先發優勢窗口)
  moat: {
    patentFiled?: boolean;                  // 「自動推播方法」專利申請(優先權 12 個月)
    patentApplicationId?: string;           // 申請案號
    firstMoverAdvantageExpiry: string;      // 預估 18 個月內會有模仿者
    dataNetworkEffect: boolean;             // 累積跑者授權資料後,新賽事進入門檻降低
    brandAssociation: 'marathon' | 'photo' | 'ai';  // 品牌定位關鍵字
  };

  // 觀察指標(供每季競品追蹤)
  competitorTracking: {
    photohawkFeatureReleasesMonitored: boolean;   // 訂閱 Photohawk Changelog
    bhaagoIndiaMonitored: boolean;
    pic2GoMonitored: boolean;
    runnerTagMonitored: boolean;
    quarterlyCompetitorReport: boolean;           // 每季產出競品報告
  };
}
```

**§2.10 推論抽象層之競品對標補強：**

§2.10 已定義之 `IInferenceAdapter` / `IFaceMatchAdapter` / `IAestheticScoringAdapter` 介面已涵蓋多引擎支援(Gemini / GPT-4o / Claude / YOLOv8 / RF-DETR / SnapSeek / Ollama),足以應對 Photohawk 之 OCR + Face Recognition 競爭。本節新增之差異化重點:

- **SnapSeek 為本系統已採用之 Face Re-ID 引擎**(§2.10 P0 預設);若 Photohawk 採用 SnapSeek 對手(如 AWS Rekognition、Clarifai、Face++),則本系統需評估引擎切換
- **§2.10 「多引擎級聯降級」(Cascade Fallback)** 設計使本系統在引擎失敗時自動升級至高精度引擎(Claude 3.5 Sonnet),優於 Photohawk 之單引擎依賴
- **§2.10 「本地部署模型的管理」**(Model Registry + 藍綠部署)使本系統可支援「隱私優先」策略之主辦方,優於 Photohawk 純雲端架構

**§2.8 推播引擎之競品對標補強：**

- Photohawk 無 OAuth 自動推播能力,本系統透過 §2.8「社群平台整合抽象層」與 §2.11 `PLATFORM_FALLBACK_CHAIN` 達成 LINE Primary → Meta Secondary 之降級鏈
- §2.8「Per-Runner Token Bucket 限流」(rate=10/min, burst=15)防止單一跑者 spam,優於 Photohawk 無個人限流之設計
- §2.8「小紅書分享預覽卡機制」覆蓋亞洲女性跑者滲透率高之平台,優於 Photohawk 之歐美中心設計

**先發優勢窗口評估：**

- deep-research 驗證之 5 家代表性服務商(Phomi / AllSports-Photo Create / MarathonFoto / FinisherPix / Photohawk)均無 OAuth 自動推播
- 預估先發優勢窗口為 **18 個月**(基於 Photohawk 之 OCR+Face Recognition B2B 商業模式已驗證技術可行性)
- 建議行動:
  1. MVP 上線前申請「自動推播方法」專利(優先權 12 個月)
  2. 鎖定台灣/日本/東南亞市場(Photohawk 尚未明確進入)
  3. 累積跑者授權資料形成 data network effect
  4. 與 LINE 官方帳號代理商建立策略夥伴關係(LINE 為 Photohawk 之明顯缺口)

### **2.4 社群發布與展現層**

最終的社群發布與展現層，系統會依據辨識出的跑者身分，查詢關聯的 OAuth 授權權杖（Access Tokens），並調用對應的平台 API（如 Meta Graph API、Threads API、LINE Messaging API），執行非同步的媒體上傳與貼文發布任務，完成端到端（End-to-End）的全自動化資料流12。

### **2.5 資料儲存層設計**

本系統採用混合儲存策略：關聯式資料庫儲存結構化業務資料，NoSQL 資料庫儲存高併發寫入之臨時性處理狀態，物件儲存服務持久化影像資產。

| 資料類型 | 儲存服務 | 說明 |
| :---- | :---- | :---- |
| **跑者註冊資料** | Amazon DynamoDB | 主要 Key 為賽事 ID (Partition Key) + 跑者號碼布編號 (Sort Key)，屬性包含姓名、OAuth Token 加密 blob、Face Re-ID 授權狀態、PDPA 同意時間戳 |
| **OAuth Token** | DynamoDB + AWS KMS | Token 狀態（有效/已撤銷/已過期）、加密後之 Access/Refresh Token、AES-256-GCM 加密 blob（由 KMS CMK 保護）、最近刷新時間 |
| **照片處理任務狀態** | DynamoDB (GSI) | 任務 ID、S3 URI、處理階段（pending/detected/ocr/fallback/published/failed）、跑者 ID、最終狀態更新時間 |
| **DLQ 任務** | DynamoDB | DLQ 任務 ID、所屬賽事、原始 S3 URI、失敗原因、retry 次數、人工確認狀態 |
| **賽事資料** | Amazon RDS (PostgreSQL) | 賽事名稱、日期、賽道點位座標、參賽人數上限、贊助商設定、PDF 模板 ID |
| **原始與處理後影像** | Amazon S3 | 原始上傳（原始大圖）、處理後（美化壓縮圖）、LINE 推播圖（縮圖）、PDF 成品 |

**KMS 金鑰管理政策：**每次賽事部署時產生新的 CMK（Customer Master Key），用於該賽事之 Token 加密；CMK 啟用日 + 90 天自動排程刪除，確保歷史 Token 無法被新金鑰解密而須重新授權。金鑰存取由 IAM Policy 控制，Lambda 執行角色具備以下最小必要 KMS 權限：

- `kms:Decrypt` — 解密已加密之 OAuth Token、自拍照 URL 等敏感資料
- `kms:GenerateDataKey` — 為 OAuth Token 寫入時產生 DEK（Data Encryption Key），以支援 AES-256-GCM envelope encryption 流程：KMS GenerateDataKey 取得明文 DEK → 本地 AES-256-GCM 加密 Token → 僅 ciphertext + encrypted DEK 寫入 DynamoDB
- `kms:DescribeKey` — 讀取 CMK 中繼資料（如 `KeyState`），供應用層驗證金鑰有效性後再執行解密

Lambda 執行角色**不具備**任何金鑰管理類權限：`kms:CreateKey`、`kms:ScheduleKeyDeletion`、`kms:DisableKey`、`kms:RotateKeyOnDemand` 等僅限 Infra / Terraform 部署角色持有，避免應用層誤刪除金鑰導致賽事資料永久無法解密。Resource 限制為 `arn:aws:kms:{primary_region}:{account}:key/{eventId}-cmk`，跨賽事資源存取以 IAM Condition 與 eventId 標籤雙重隔離。

**S3 儲存桶命名策略：**所有賽事共用同一主 bucket（`{env}-imarathon-photos`），以 `eventId/` 前綴隔離不同賽事資料。Face Re-ID 自拍照使用獨立隔離 bucket（`{env}-imarathon-biometric`），不得與一般照片bucket 混用。Bucket Policy 嚴禁跨前綴讀取；Lambda 執行角色僅有對應 eventId 前綴之 `s3:GetObject` / `s3:PutObject` 權限。

**DynamoDB 全域二級索引（GSI）設計：**

| 索引名稱 | GSI Key | 查詢場景 |
| :---- | :---- | :---- |
| `runner-platform-index` | PK: `runnerId`, SK: `platform` | 查詢特定跑者在各平台之 Token 狀態 |
| `dlq-status-index` | PK: `eventId`, SK: `status#createdAt` | 查詢特定賽事之 DLQ 任務（依狀態排序） |
| `task-status-index` | PK: `eventId`, SK: `status#updatedAt` | 查詢特定賽事之處理任務（依狀態+時間排序） |
| `runner-event-index` | PK: `eventId`, SK: `bibNumber` | 查詢特定賽事之跑者資料（by bibNumber） |
| `line-user-index` | PK: `lineUserId`, SK: `eventId` | 依 LINE User ID 查詢跑者綁定（§3.5 LINE Webhook follow 事件流程用） |

**GSI 命名規範（v1.5 補充）：** 本系統所有 DynamoDB GSI 統一採用 `<entity>-<attribute>-index` 三段 kebab-case 命名：
- **entity**: 索引主體（runner / dlq / task / line 等實體名）
- **attribute**: 索引鍵屬性（platform / status / event / user 等）
- **index**: 固定後綴

> **附註：** §3.6 OAuth Token 表新增之 3 個 GSI（`platform-user-index`、`status-expiry-index`、`event-status-index`，詳見 §3.6）遵循相同規範；Terraform/CDK 程式碼應以 `gsi-` 前綴作為 IaC 內部識別（如 `aws_dynamodb_table.runner.gsi["runner-platform-index"]`），但實際 GSI 物理名稱不帶前綴以維持 AWS Console 可讀性。

**GSI 投影（Projection）策略：** 所有 GSI 預設投影模式為 `KEYS_ONLY`（僅投影 PK + SK），降低儲存成本與寫入放大效應。`task-status-index` 例外：採 `INCLUDE` 模式並投影 `status`、`updatedAt`、`checkpointCode` 三個屬性，供後台 DLQ 處理面板（§3.8）直接查詢狀態而無須回主表 fetch。`platform-user-index`（§3.6 OAuth Token 表）採 `INCLUDE` 模式投影 `platformUserId` 與 `eventId`，供 §3.5 LINE Webhook follow 事件快速反查綁定關係。
**GSI 投影（Projection）策略：** 所有 GSI 預設投影模式為 `KEYS_ONLY`（僅投影 PK + SK），降低儲存成本與寫入放大效應。`task-status-index` 例外：採 `INCLUDE` 模式並投影 `status`、`updatedAt`、`checkpointCode` 三個屬性，供後台 DLQ 處理面板（§3.8）直接查詢狀態而無須回主表 fetch。

所有 GSI 寫入時維持 Strong Consistency，讀取支援最終一致性（應用於監控儀表板等非即時場景）。

**DynamoDB Capacity Mode（容量模式）：**

| Table | Capacity Mode | 理由 |
| :---- | :---- | :---- |
| `Runner` | **On-Demand** | 報名尖峰（賽事前一週）burst 寫入;平日低用量;成本最低 |
| `OAuthToken` | **On-Demand** | 寫入量低（每次 OAuth flow 一次）;讀取隨機;無預測需求 |
| `PhotoProcessingTask` | **On-Demand** | 賽事日 8 小時 burst 寫入 60K+;平日 0;Provisioned 浪費嚴重 |
| `DLQTask` | **On-Demand** | DLQ 寫入量低且不規則;無法預測 |
| `IdempotencyKey` | **On-Demand + TTL** | 24 小時自動清除;On-Demand 適合不規則高寫入 |
| `PendingLineBinding` | **On-Demand** | 跑者報名前的臨時綁定;低用量 |
| `EventConfig` | **Provisioned（5 RCU / 5 WCU）** | 後台設定讀寫;用量穩定可預測;Provisioned 更省成本 |
| `ModelRegistry` | **Provisioned（5 RCU / 5 WCU）** | AI 模型版本中繼資料;讀寫頻率低但穩定 |

**On-Demand 成本估算（以 20K 賽事為例）：** 總寫入約 100K WCU + 總讀取約 50K RCU,$1.25/M WCU + $0.25/M RCU = $0.125 + $0.0125 ≈ **$0.14/賽事日**,遠低於 Provisioned baseline $30/月 即使只算賽事當天 8 小時（$30 × 8/720 = $0.33）。On-Demand 為首選。

**例外情境：** 若單一賽事 Runner 報名尖峰寫入 > 40K WCU/s（極少見,通常僅發生於售票式限量賽事開賣瞬間），自動改用 `AWS Auto Scaling` 動態 Provisioned（目標 70% 利用率）。

### **2.6 多租戶隔離策略**

本系統支援「同一套系統同時服務多場賽事」，以帳號資料隔離為核心設計原則：

- **網路層級：** 所有賽事共用同一組 Lambda 函數與 VPC，透過 AWS Resource Tag 與 IAM Condition 限制跨賽事資源存取。每次 Lambda 執行時以 eventId 參數確定資料存取範圍，避免每場賽事部署獨立 Lambda 造成過度的 IaC 管理負擔與冷啟動延遲。僅對有強制合規需求的賽事（如政府機關賽事要求資料完全隔離）提供可選的物理隔離部署模式。
- **資料庫層級：** DynamoDB Table 以賽事 ID 作為 Partition Key，確保查詢範圍永遠限於單一賽事；RDS 以 Schema 隔離（`event_{event_id}`），避免資料洩漏。
- **OAuth Token：** Token 嚴格绑定「賽事 ID + 跑者 ID」，跨賽事呼叫時 Token 比對會立即失敗，防止 A 賽事的照片被錯誤推播至 B 賽事跑者。

| 隔離層級 | 機制 | 失效情境 |
| :---- | :---- | :---- |
| 網路 | IAM Condition + Resource Tag + eventId 執行期參數隔離 | Lambda 函數未正確傳入 eventId 導致跨賽事讀取 |
| 資料庫 | DynamoDB Partition Key / RDS Schema | 查詢漏接 Partition Key 導致 Cross-event scan |
| OAuth | Token 附加 event_id 聲明（JWT Claims） | Token 被盜用且攻擊者成功更換綁定 event_id |

| 系統架構元件 | 採用技術與核心服務 | 核心職責與資料流向 |
| :---- | :---- | :---- |
| **資料獲取與攝入端** | 5G 路由器、FTP/API 腳本、UHF RFID 解碼器 | 攝影師高頻影像上傳；RFID 晶片通過時間點（JSON 陣列）推送，涵蓋 timingId 與 timestamp（參見 2.1）。 |
| **雲端儲存與事件觸發** | Amazon S3、S3 Event Notifications | 原始高畫質影像持久化儲存，自動觸發非同步的影像建立事件3。 |
| **訊息緩衝與流量調節** | Amazon SQS (Standard 佇列) | 影像處理任務非同步排程，內建重試機制與死信佇列（DLQ），防止大流量突發造成的處理遺漏3。 |
| **無伺服器運算與編排** | AWS Lambda (Node.js/Python 執行環境)、IAM Authenticated API | 執行核心業務邏輯、協調 AI 模型推論、進行資料庫 I/O 操作與處理社群 API 調用；資料庫連線使用 IAM Auth 而非長期帳號密碼15。 |
| **AI 視覺辨識引擎** | YOLOv8 / RF-DETR 搭配特徵金字塔網路 | 跑者身軀框選追蹤、號碼布精確定位與變形幾何校正；實際 OCR/LLM 推論引擎由 2.10 AI/LLM 抽象層決定2。 |
| **影像與文件動態渲染** | Sharp (Node.js via Lambda Layer), Puppeteer-core | 高效能記憶體內合成相框、疊加配速數據、修復 EXIF 翻轉；無頭瀏覽器生成專屬 PDF 完賽報紙18。 |
| **社群 API 整合模組** | OAuth 2.0, Meta Graph API, LINE Messaging API | 權杖生命週期管理與加密、多段式媒體容器上傳、API 頻率限制（Rate Limit）控制與退避策略13。 |
| **外部計時系統整合抽象層** | Adapter Pattern (工廠模式) | 統一封裝各賽事計時系統（MyLaps/ChampionChip/RaceEntry/TimingMatrix）之 API 差異，向下游提供一致的標準化資料模型。 |
| **社群平台整合抽象層** | Adapter Pattern (工廠模式) | 統一封裝各社群平台（Instagram/Threads/LINE/Facebook 等）之 API 差異，支援未來新平台熱拔插式擴充。 |
| **雲端基礎設施抽象層** | Provider Adapter Pattern + Terraform | 統一封裝 AWS/GCP/Azure 等雲端服務差異，支援多雲部署與雲端服務熱切換。 |
| **AI/LLM 推論抽象層** | Adapter Pattern (工廠模式) | 統一封裝各 OCR/LLM 引擎（雲端 API / 本地部署），支援成本導向與延遲導向之引擎動態切換。 |
| **功能開關與配置抽象層** | Feature Flag + Event Config | 每場賽事之功能開關（LINE/IG/Threads/臉部辨識）、推論引擎、AI 策略、PDF 生成等皆由集中式配置管理，支援熱切換與未來功能擴充。 |

### **2.7 外部計時與成績系統整合抽象層**

各賽事主辦單位所採用之晶片計時系統、成績系統與跑者資訊來源各異。本系統定義「外部整合抽象層」，以標準化的內部資料模型隔離所有外部系統差異，確保新增賽事支援時無需修改核心處理邏輯。

#### **標準化內部資料模型**

下游 AI 處理與排版系統僅依賴以下標準化資料模型，不直接耦合任何外部系統：

```typescript
// 標準晶片過站紀錄（Normalized Chip Passage Record）
interface NormalizedCheckpointRecord {
  eventId: string;
  chipId: string;           // 晶片唯一識別碼（與 bibNumber 不同）
  bibNumber: string;        // 號碼布編號（由 adapter 查表或 OCR 交叉比對取得）
  checkpointCode: string;    // 檢查點代碼（如 "START", "KM25", "FINISH"）
  timestamp: string;        // ISO 8601 UTC 時間戳
  gpsLat?: number;          // 選用：含 GPS 時提供
  gpsLng?: number;          // 選用：含 GPS 時提供
  sourceSystem: string;     // 來源系統名稱（如 "mylaps", "championchip"），僅供日誌用
}

// 標準成績紀錄（Normalized Official Result）
interface NormalizedOfficialResult {
  eventId: string;
  bibNumber: string;
  chipId: string;
  fullName: string;
  gender?: string;
  age?: number;
  category?: string;        // 參賽組別
  officialTime: string;     // 正式成績（HH:MM:SS）
  chipTime?: string;       // 晶片計時成績
  splits: {
    checkpointCode: string;
    splitTime: string;      // 區間時間
    cumulativeTime: string; // 累計時間
  }[];
  rankOverall?: number;     // 總排名
  rankCategory?: number;    // 組別排名
}
```

#### **Adapter 實作矩陣**

每個外部系統需實作對應之 Adapter，將其原生 API Output 轉換為上述標準模型。以下為規劃支援之主要系統與其差異說明：

| 外部系統 | 晶片類型 | API 形式 | 特殊欄位 | 預計 Adapter 優先順序 |
| :---- | :---- | :---- | :---- | :---- |
| **MyLaps** | UHF RFID（紙片式/鞋帶式） | REST API (XML/JSON) | `decoderId`, `passTime`, `BibTagID` | P0（最高） |
| **ChampionChip** | UHF RFID | FTP CSV 匯出 或 REST API | `chip_code`, `gun_time`, `net_time` | P0 |
| **RaceEntry** | UHF RFID | REST API (JSON) | `participant_id`, `split_times[]` | P1 |
| **TimingMatrix** | UHF RFID / 計時晶片 | Webhook POST | `reader_id`, `bib`, `time` | P1 |
| **IPICO** | UHF RFID | SOAP / REST API | `athleteID`, `splitID`, `matID` | P2 |
| **Active Network** | 晶片+號碼布整合 | REST API | `bibNumber`, `gunTime`, `chipTime` | P2 |
| **CPS（香港/澳門常用）** | UHF RFID | FTP CSV | `ChipID`, `CheckpointID`, `ReadTime` | P2 |

#### **Adapter 介面定義**

所有 Adapter 須實作以下統一介面（以 TypeScript 為例）：

```typescript
interface ITimingSystemAdapter {
  /** 系統類型識別碼（如 "mylaps", "championchip"） */
  readonly systemType: string;

  /**
   * 將外部系統原生晶片過站資料，轉換為標準化晶片過站紀錄。
   * @param rawPayload 外部系統 POST/GET 之原始 payload（型別依系統而異）
   * @param eventConfig 賽事特定設定（如 API Key、端點 URL、Mapping Table）
   */
  normalizeCheckpoint(rawPayload: unknown, eventConfig: EventTimingConfig): NormalizedCheckpointRecord[];

  /**
   * 將外部系統成績 API 結果，轉換為標準化成績紀錄。
   * @param officialResultRaw 外部成績系統原始輸出
   */
  normalizeOfficialResult(officialResultRaw: unknown, eventConfig: EventTimingConfig): NormalizedOfficialResult[];

  /**
   * 驗證 Adapter 與外部系統連線是否正常（健康檢查）。
   * 賽事啟用前、後台管理員可觸發此方法確認設定正確。
   */
  healthCheck(eventConfig: EventTimingConfig): Promise<{ ok: boolean; message: string }>;
}
```

#### **Adapter 工廠與動態路由**

系統於賽事初始化時，依據 `EventTimingConfig.adapterType` 欄位，由工廠函數（Factory Function）動態實例化對應 Adapter：

```typescript
function createTimingAdapter(
  adapterType: string,
  eventConfig: EventTimingConfig
): ITimingSystemAdapter {
  const adapterMap: Record<string, new (config: EventTimingConfig) => ITimingSystemAdapter> = {
    'mylaps': MyLapsAdapter,
    'championchip': ChampionChipAdapter,
    'raceentry': RaceEntryAdapter,
    'timingmatrix': TimingMatrixAdapter,
    'ipico': IPICOAdapter,
    'activenetwork': ActiveNetworkAdapter,
    'cps': CPSAdapter,
    // 未來擴充：直接在此新增映射
  };
  const AdapterClass = adapterMap[adapterType];
  if (!AdapterClass) throw new Error(`Unsupported timing adapter: ${adapterType}`);
  return new AdapterClass(eventConfig);
}
```

#### **晶片號碼布交叉查詢（Chip-to-Bib Mapping）**

部分系統以晶片 ID（chipId）為主要索引，而非 bibNumber。Adapter 須支援「晶片 ID → 號碼布」之查詢表，由賽事主辦單位於後台 CSV 匯入。此查詢表為 `chipId → bibNumber` Key-Value 對，儲存於 DynamoDB，Adapter 在 `normalizeCheckpoint` 執行過程中即時查表填入 `bibNumber`。

#### **新增賽事支援流程**

未來新增 Adapter 支援新計時系統時，開發流程為：

1. 於 `adapters/` 目錄下新增 `{systemName}.adapter.ts`，實作 `ITimingSystemAdapter` 介面
2. 向工廠函數 `createTimingAdapter` 新增 `adapterType` 映射
3. 於後台管理系統之「計時系統設定」下拉選單中新增該系統選項
4. 撰寫 Adapter 單元測試（Mock 外部系統 API 回應）與整合測試（實際呼叫外部測試環境）

此流程無需觸及核心 AI 處理、推播邏輯或資料庫 Schema，確保新增系統支援為水平擴充而非修改核心。

### **2.8 社群平台整合抽象層**

本系統透過「社群平台整合抽象層」將各社群平台之 OAuth 流程、API 規範、媒體格式限制與 Rate Limit 差異完全隔離。核心推播引擎僅依賴標準化的內部介面，新增平台無需修改推播引擎本身，僅需實作對應之 Adapter。

#### **標準化內部推播介面**

核心推播引擎僅依賴以下標準化介面：

```typescript
// 標準化推播任務（Normalized Publish Task）
interface NormalizedPublishTask {
  runnerId: string;
  platform: string;             // 標準化平台識別碼，如 "instagram", "threads", "line", "facebook"
  mediaType: 'image' | 'video' | 'carousel';
  primaryImageS3Uri: string;   // S3 URI of the processed/beautified image
  thumbnailS3Uri?: string;     // S3 URI of thumbnail (required for LINE)
  captionText: string;         // 平台無關之標準化文案，Adapter 依平台規範進行截斷/格式化
  hashtags?: string[];         // 標準化標籤列表，Adapter 依平台限制處理
  deepLinkUrl?: string;        // 點擊追蹤用 deep link
  metadata: {
    eventId: string;
    bibNumber: string;
    checkpointCode: string;
    publishedAt?: string;
  };
}

// 標準化推播結果（Normalized Publish Result）
interface NormalizedPublishResult {
  taskId: string;
  platform: string;
  status: 'published' | 'failed' | 'rate_limited' | 'token_revoked';
  externalPostId?: string;     // 平台返回之貼文 ID
  externalPostUrl?: string;    // 平台返回之貼文永久連結
  publishedAt: string;
  errorCode?: string;
  errorMessage?: string;
}
```

#### **Adapter 實作矩陣**

| 平台 | API 類型 | OAuth 流程 | 媒體限制 | 推播方式 | 支援狀態 |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **LINE** | LINE Messaging API | LINE Login | PNG/JPG, 必須同時提供原圖+縮圖 | Push Message (後續推播) | ✅ P0（標配） |
| **Instagram** | Instagram Graph API | Facebook OAuth 2.0 | 僅 JPEG, 8MB, 4:5–1.91:1 | Content Publishing API（兩階段） | ✅ P0（標配） |
| **Threads** | Threads API | Facebook OAuth 2.0 | PNG/JPG, 8MB, 9:16 或 4:5 | POST Media + Publish | ✅ P0（標配） |
| **Facebook** | Facebook Graph API | Facebook OAuth 2.0 | PNG/JPG/GIF, 8MB | Photo Upload API | ✅ P1（支援） |
| **X (Twitter)** | X API v2 | OAuth 2.0 (PKCE) | PNG/JPG/GIF, 5MB | Media Upload + Tweet | ✅ P1（支援） |
| **Bluesky** | AT Protocol | OAuth 2.0 with Bluesky | PNG/JPG, 1MB (未驗證帳號) / 5MB | Bluesky API (repo/createRecord) | ✅ P2（支援） |
| **TikTok** | TikTok API | OAuth 2.0 | PNG/JPG, ≤ 10MB | Content Posting API | ✅ P2（支援） |
| **Pinterest** | Pinterest API | OAuth 2.0 | PNG/JPG, ≤ 10MB | Pins API | ✅ P2（支援） |
| **Weibo** | Weibo API | OAuth 2.0 | PNG/JPG, ≤ 5MB | Statuses API | ✅ P2（未來擴充） |
| **小紅書** | 小紅書 API | OAuth 2.0 (企業號) | PNG/JPG, ≤ 10MB | Note API | ✅ P1（支援；對女性跑者滲透率高） |

#### **社群平台 Adapter 介面定義**

```typescript
interface ISocialPlatformAdapter {
  readonly platform: string;
  publish(task: NormalizedPublishTask, runnerConfig: RunnerPlatformConfig, credentials: PlatformCredentials): Promise<NormalizedPublishResult>;
  handleRateLimit(error: PlatformApiError, retryCount: number): Promise<Date>;
  handleAuthFailure(error: PlatformApiError): Promise<TokenStatus>;
  healthCheck(credentials: PlatformCredentials): Promise<{ ok: boolean; message: string }>;
  formatPostText(captionText: string, hashtags: string[]): string;
}
```

#### **推播引擎工作流程**

核心推播引擎完全不感知平台差異。AI 辨識 Lambda 與推播引擎 Lambda 之間以「**推播專用 SQS Queue（publish-queue）**」進行解耦，確保兩者可獨立擴縮容且互不阻塞：

1. AI 辨識 Lambda 完成影像處理與美化後，將 `NormalizedPublishTask` JSON 寫入推播專用 SQS Queue（`sqs://{eventId}-publish-queue`）
2. 推播引擎 Lambda 由推播 SQS 觸發，依 `task.platform` 呼叫 `SocialPlatformFactory.createAdapter(platform)` 動態實例化對應 Adapter
3. Adapter 查 Token Manager 取得 `credentials`（已解密之 Access Token）
4. Adapter 呼叫平台 API，處理 Rate Limit 與 Token 失效
5. 結果以 `NormalizedPublishResult` 回傳，引擎寫入 DynamoDB 任務狀態表
6. 若 `status === 'failed'`，引擎依失敗原因決定重試策略：
   - **Rate Limited（429）：** 依 Retry-After 標頭延遲後重新入隊至推播 SQS
   - **Token 失效（401/403）：** 寫入 DLQ「Token 失效」子類型，不重試
   - **平台暫時性錯誤（5xx）：** SQS 自動重試（最多 3 次），3 次失敗後寫入 DLQ

#### **Per-Runner 限流（防洗版機制）**

單一跑者於賽事中可能被多個攝影點拍攝（典型 3–8 張照片），若該跑者無意或惡意大量註冊多平台（如同時綁定 LINE + IG + Threads + Facebook），可能在短時間內觸發數十次推播，造成使用者體驗不佳甚至被平台視為 spam。為此，推播引擎內建 **Per-Runner Token Bucket 限流**：

- **後端實作** — Amazon ElastiCache for Redis（Single-AZ,cache.t3.micro 即可,Memory 1GB）
- **限流參數** — 預設每跑者每平台每分鐘 ≤ 10 則推播（`rate=10/min, burst=15`）;由 `EventFeatureConfig.publishing.perRunnerRateLimit`（預設 10）設定
- **Token Bucket Key** — `runner:{runnerId}:platform:{platform}` → `{tokens, lastRefillAt}`
- **超限行為** — 推播任務延遲 30 秒後重新入隊至 publish-queue（不寫 DLQ,給予自然洩洪）
- **跨平台獨立計算** — LINE 與 IG 限流桶各自獨立,跑者可同時收到 LINE 與 IG 推播但不會被任一平台單獨 spam
- **賽事結束後關閉** — `eventEndAt` 之後停用限流桶,賽後補發推播（如人工修正 DLQ 後重推）不受限

**Redis 故障降級** — 若 ElastiCache 連線失敗（網路瞬斷或維護），推播引擎採「fail-open」策略：直接放行所有推播,並於 CloudWatch 記錄 `metric: per_runner_rate_limit_fail_open_count` 供賽事後分析。Fail-open 避免 Redis 故障造成整體推播阻塞。

#### **新增平台支援流程**

未來新增社群平台支援時，開發流程為：

1. 於 `social-adapters/` 目錄下新增 `{platform}.adapter.ts`，實作 `ISocialPlatformAdapter` 介面
2. 向工廠函數 `SocialPlatformFactory.createAdapter` 新增 `platform` 映射
3. 向後台管理系統之「支援平台」設定頁面新增該平台開關（預設關閉）
4. 賽事主辦單位於後台填入新平台之 App Credentials（App ID / Secret / OAuth Redirect URI）
5. 撰寫 Adapter 單元測試（Mock 平台 API）與 E2E 測試（使用平台測試帳號）

此流程為純水平擴充，核心推播引擎、Lambda 函數與資料庫 Schema 完全不受影響。

#### **小紅書分享預覽卡機制（小紅書 P1 實作細節）**

小紅書不支援直接 API 發文（無公開 Content Publishing API），故採用「分享預覽卡」機制：
1. 系統產生 Open Graph 預覽卡（標題圖+賽事 Logo+完賽資訊），托管於 CloudFront CDN
2. 推播時，訊息文案包含「一鍵分享至小紅書」Deep Link，點擊後在 App 內開啟預覽卡，跑者可一鍵複製完整文案或截圖分享
3. 對於已授權 LINE 的港澳/東南亞女性跑者，可同時以 LINE 推播小紅書適用格式（emoji + 主題標籤 `#跑步``#完賽`+ 種草語氣）
4. Dcard 採用相同機制，以分享預覽卡形式提供一鍵分享至 Dcard 看板

#### **平台政策風險管理（§2.8.X）**

基於 deep-research 對 Meta Threads API 存取審查(被驗證者描述為「extremely difficult」與「opaque」)與 Threads 限流(滾動 24 小時 250 則貼文 + 1,000 則回覆)之觀察,本節定義平台政策風險之介面層級管理。本節為 §3.9「平台條款變動應變計畫」之介面層細節,提供 ISocialPlatformAdapter 之政策風險觀察點與降級信號。

**為何介面層需關注政策風險(而非僅 §3.9 治理層)：**

- Adapter Pattern 之優勢在於「平台差異抽象化」,但若無政策風險觀察點,Adapter 僅處理 API 語法層級差異,無法向上層(Publish 引擎、§3.9 應變計畫)回報「平台條款層級」之風險
- 本節新增之 `PlatformPolicyRisk` 介面由各 Adapter 實作時填報,提供 §3.9 監控機制所需之原始資料

**核心介面定義：**

```typescript
interface PlatformPolicyRisk {
  platform: string;                         // 'instagram' | 'threads' | 'facebook' | 'line'

  // Meta App Review 狀態(僅 Meta 系平台適用)
  metaAppReviewStatus: 'not_started' | 'in_review' | 'approved' | 'revoked';
  metaReviewExpiryAt?: string;              // Meta App Review 通常 12 個月效期
  metaTechProviderApplicationId?: string;   // Tech Provider 申請 ID(被驗證為「extremely difficult」)
  videoSubmissionUrl?: string;              // 審查用影片示範 URL

  // Threads 限流追蹤(僅 Threads 適用)
  threadsRolling24hPostCount: number;
  threadsRolling24hReplyCount: number;
  threadsRateLimitHitAt?: string;
  threadsRateLimitBuffer: number;           // 預設 0.2(保留 20% 緩衝)

  // 平台條款變更通知(Webhook 訂閱狀態)
  metaWebhookSubscriptionActive: boolean;
  threadsWebhookSubscriptionActive: boolean;
  lineWebhookSubscriptionActive: boolean;

  // 平台帳號健康度(由 Adapter 自動觀察)
  accountHealth: {
    lastSuccessfulPushAt?: string;
    consecutiveFailureCount: number;        // 連續失敗次數;> 5 觸發 P1 警告
    platformWarningReceived: boolean;        // 是否收到平台警告信(如 spam 警告)
    platformRestrictionType?: 'reduced_reach' | 'feature_limited' | 'temporary_block' | 'permanent_ban';
  };

  // 平台條款變動觀察(由 Adapter 透過 RSS/Changelog 監控填報)
  recentPolicyChanges: {
    detectedAt: string;
    changeSummary: string;
    impactAssessment: 'none' | 'low' | 'medium' | 'high';
    responseAction: string;
  }[];
}
```

**Meta Threads API 存取審查介面（重點細則）：**

```typescript
interface MetaTechProviderApplication {
  // 申請流程狀態追蹤
  applicationStatus: 'draft' | 'submitted' | 'in_review' | 'rejected' | 'approved';

  // 申請所需之 Metadata(對應 §2.8 自動產出之影片腳本)
  applicationMetadata: {
    appName: string;
    useCase: 'marathon_photo_auto_publishing';
    videoDemoUrl: string;                   // 5-10 分鐘影片,展示 instagram_content_publish 與 threads_content_publish 之使用情境
    testingInstructions: string;            // 詳細步驟供 Meta Reviewer 驗證
    dataHandlingDisclosure: string;         // 個資處理與隱私政策連結
    permissionJustification: {
      permission: string;                   // 'instagram_content_publish'
      whyNeeded: string;                    // '用於將賽事完賽照片自動推播至跑者個人 IG 帳號'
      alternativeConsidered: string;        // 'IG Personal Account 不支援 API,故需 Business Account'
    }[];
  };

  // 預估審查週期(基於 deep-research 觀察)
  expectedReviewDurationDays: number;       // 7-30 天(業界觀察值)
  reviewStartedAt?: string;
  reviewCompletedAt?: string;
  reviewerFeedbackLog: string[];
}
```

**平台條款變動觀察點（Adapter 實作契約）：**

各 ISocialPlatformAdapter 實作時,除原 `publish()` / `handleRateLimit()` / `handleAuthFailure()` / `healthCheck()` / `formatPostText()` 五方法外,**新增**以下觀察方法:

```typescript
interface ISocialPlatformAdapter {
  // ...(既有 5 方法保留)...

  // 新增:觀察平台條款變動(RSS / Changelog / Webhook)
  observePolicyChanges(): Promise<PlatformPolicyChange[]>;

  // 新增:取得當前 policy 風險評估(供 §3.9 監控使用)
  getPolicyRisk(): Promise<PlatformPolicyRisk>;

  // 新增:驗證當前批次是否符合平台當前條款(例:每日限流是否還有餘裕)
  validateBatchCompliance(task: NormalizedPublishTask[]): Promise<{
    compliant: boolean;
    blockedTasks: NormalizedPublishTask[];   // 被條款阻擋的任務
    reason?: string;
  }>;

  // 新增:撤回已推播貼文(§3.1.Y Post-Push Withdrawal 機制使用)
  deletePost(externalPostId: string, credentials: PlatformCredentials): Promise<{
    deleted: boolean;
    alreadyGone: boolean;                    // 平台端已不存在,視為撤回成功
    rateLimited: boolean;
  }>;
}
```

**Threads 限流緩衝實作（介面層細則）：**

> **常數引用說明（v1.4）：** Threads 24 小時 250 則貼文上限為**平台政策常數**，於本 SPEC 中於 §2.8.L634（介面層）、§3.9.L2302（治理層）兩處獨立陳述。為避免未來 Meta 調整限流時漏改一處，建議於下次修補（v1.5）將此常數抽為 `EventFeatureConfig.publishing.threadsHardLimitPer24h: number = 250`，並於本檔所有引用處改為 `${threadsHardLimitPer24h} × (1 - threadsRateLimitBuffer)`。

- Threads 對單一 profile 滾動 24 小時 250 則貼文 + 1,000 則回覆上限(1 個 carousel 算 1 則貼文)
- 推播引擎於 `validateBatchCompliance()` 內比對 `threadsRolling24hPostCount + 批次數量` 是否超過 `250 × (1 - threadsRateLimitBuffer)`
- 超限任務自動延後至下個 24 小時視窗(不寫 DLQ,因屬平台政策性限制而非系統故障)
- `threadsRolling24hPostCount` 由 Adapter 於每次 `publish()` 成功後 +1,並於 Threads API 回 200 時自動同步至 DynamoDB `PlatformUsageMetrics` 表

**Meta 政策變動 → §3.9 應變計畫觸發鏈：**

```
Meta App Review 過期前 30 天
  → §2.8.X MetaAppReviewChecklist.metaReviewExpiryAt 倒數
  → §2.11 Feature Flag META_APP_REVIEW_STATUS 觸發 P2 警告
  → §3.9 監控機制記錄事件
  → Super Admin 後台顯示「請提前送審續期」提醒

Meta 撤回 threads_content_publish scope
  → Adapter publish() 回 401/403 with scope_error
  → §2.8.X handleAuthFailure() 標記 token_revoked
  → §3.9 P0_immediate 觸發
  → 自動啟用 Open Graph 預覽卡 + LINE 全量推播降級
```

### **2.9 雲端基礎設施抽象層**

本系統並非僅部署於 AWS，賽事主辦單位可能已有其他雲端供應商之既有資源，或因資料落地（Data Residency）法規要求須使用特定區域之雲端服務。本系統定義「雲端基礎設施抽象層」，將核心業務邏輯與底層雲端服務解耦，支援在 AWS、GCP、Azure 之間無縫遷移或同時使用多雲架構。

#### **核心抽象元件**

本系統之業務邏輯依賴以下幾類雲端服務，抽象層為每一類定義標準化介面：

| 服務類別 | AWS 預設 | GCP 對應 | Azure 對應 |
| :---- | :---- | :---- | :---- |
| **物件儲存** | Amazon S3 | Cloud Storage (GCS) | Azure Blob Storage |
| **無伺服器運算** | AWS Lambda | Cloud Functions / Cloud Run | Azure Functions |
| **訊息佇列** | Amazon SQS | Cloud Tasks / Pub/Sub | Azure Queue Storage |
| ** NoSQL 資料庫** | Amazon DynamoDB | Firestore / Cloud Datastore | Azure Cosmos DB |
| **關聯式資料庫** | Amazon RDS (PostgreSQL) | Cloud SQL | Azure Database for PostgreSQL |
| **金鑰管理** | AWS KMS | Cloud KMS | Azure Key Vault |
| **CDN / 邊緣快取** | Amazon CloudFront | Cloud CDN | Azure CDN |
| **AI/ML 推論** | AWS SageMaker / Bedrock | Vertex AI | Azure AI Studio |
| **容器登錄** | Amazon ECR | Artifact Registry | Azure Container Registry |
| **基礎設施即程式** | AWS CDK / Terraform | Terraform / Deployment Manager | Terraform / Bicep |

#### **標準化基礎設施介面**

```typescript
// 標準化儲存抽象（IStorageAdapter）
interface IStorageAdapter {
  readonly provider: 'aws' | 'gcp' | 'azure';
  upload(key: string, body: Buffer, options?: UploadOptions): Promise<string>; // 回傳公開存取 URL 或 S3 URI
  download(key: string): Promise<Buffer>;
  delete(key: string): Promise<void>;
  generatePresignedUrl(key: string, expiresInSeconds: number): Promise<string>;
}

// 標準化無伺服器函數抽象（IComputeAdapter）
interface IComputeAdapter {
  readonly provider: 'aws' | 'gcp' | 'azure';
  invoke(functionName: string, payload: unknown): Promise<unknown>;
  setEventSource(functionName: string, triggerConfig: TriggerConfig): void;
  setConcurrency(functionName: string, concurrency: number): void;
}

// 標準化訊息佇列抽象（IQueueAdapter）
interface IQueueAdapter {
  readonly provider: 'aws' | 'gcp' | 'azure';
  enqueue(queueName: string, message: unknown): Promise<void>;
  dequeue(queueName: string, batchSize: number): Promise<QueueMessage[]>;
  acknowledge(queueName: string, receiptHandle: string): Promise<void>;
  deadLetter(queueName: string, message: unknown, errorReason: string): Promise<void>;
}

// 標準化 NoSQL 資料庫抽象（IDatabaseAdapter）
interface IDatabaseAdapter {
  readonly provider: 'aws' | 'gcp' | 'azure';
  put(table: string, item: Record<string, unknown>): Promise<void>;
  get(table: string, key: Record<string, string>): Promise<Record<string, unknown> | null>;
  query(table: string, indexName: string, keyCondition: Record<string, unknown>): Promise<Record<string, unknown>[]>;
  update(table: string, key: Record<string, string>, updates: Record<string, unknown>): Promise<void>;
}
```

#### **Provider 工廠與動態路由**

所有 Adapter 均透過工廠函數依據部署時的 `CLOUD_PROVIDER` 環境變數（`aws` / `gcp` / `azure`）動態實例化：

```typescript
function createStorageAdapter(provider: string): IStorageAdapter {
  const map = {
    'aws': () => new S3StorageAdapter(),
    'gcp': () => new GCSStorageAdapter(),
    'azure': () => new AzureBlobStorageAdapter(),
  };
  return (map[provider] ?? map['aws'])();
}

function createQueueAdapter(provider: string): IQueueAdapter {
  const map = {
    'aws': () => new SQSQueueAdapter(),
    'gcp': () => new CloudTasksQueueAdapter(),
    'azure': () => new AzureQueueAdapter(),
  };
  return (map[provider] ?? map['aws'])();
}
```

#### **基礎設施即程式（IaC）跨雲支援**

基礎設施所有資源皆以 Terraform 宣告式定義，確保跨雲部署可重現。模組結構如下：

```
infrastructure/
├── modules/
│   ├── storage/           # S3 / GCS / Azure Blob 通用介面模組
│   ├── compute/           # Lambda / Cloud Functions / Azure Functions 通用介面模組
│   ├── queue/             # SQS / Cloud Tasks / Azure Queue 通用介面模組
│   ├── database/          # DynamoDB / Firestore / Cosmos DB 通用介面模組
│   └── iam/              # IAM / Cloud IAM / Azure AD 角色對應模組
├── environments/
│   ├── aws/              # AWS 特定 provider + region 設定
│   ├── gcp/              # GCP 特定 provider + project 設定
│   └── azure/            # Azure 特定 provider + subscription 設定
└── main.tf               # 根層級 Module 組合
```

每個 `modules/` 子目錄接收 Provider 無關之參數，內部依 `var.provider` 條件式呼叫對應雲端資源，確保同一份 Terraform 程式碼可部署至三個雲端。

#### **AI 模型部署抽象**

AI 推論（YOLOv8 / RF-DETR / SnapSeek）之模型檔案存放於物件儲存，模型執行環境之差異由 Adapter 隔離：

```typescript
interface IAIModelAdapter {
  readonly provider: 'aws' | 'gcp' | 'azure';
  loadModel(modelKey: string): Promise<void>;     // 載入模型至記憶體
  infer(input: Buffer): Promise<ModelOutput>;     // 執行推論
  getModelMetadata(modelKey: string): ModelMetadata;
}
```

#### **跨雲遷移流程**

當賽事主辦單位需要將系統從 AWS 遷移至 GCP 或 Azure 時，流程如下：

1. 於新雲端供應商之帳號下執行 `terraform apply`
2. 將 S3 上的模型檔案與照片資料遷移至新雲端之物件儲存
3. 將 DynamoDB 資料以 CSV 格式匯出，寫入新 NoSQL 資料庫
4. 變更 `CLOUD_PROVIDER` 環境變數，切換至新雲端 Adapter
5. 執行 Smoke Test 驗證所有核心功能正常運作
6. 確認無誤後，關閉舊雲端帳號下所有資源

此過程中，業務邏輯（Lambda 函數程式碼、AI 模型參數、Adapter 實作）完全不需要修改。

### **2.10 AI/LLM 推論抽象層**

本系統之 AI 視覺辨識管線（跑者身軀偵測、號碼布定位、OCR 字元萃取、Face Re-ID）並非寫死特定模型或供應商，而是透過「AI/LLM 推論抽象層」動態選擇最適合當下場景的推論引擎。此設計讓賽事主辦單位能依據成本、延遲、準確率與資料隱私需求，在不同推論引擎之間切換；同時支援本地部署開源模型（如 Ollama/LLaMA、LLaVA），以降低雲端 API 費用。

#### **推論引擎矩陣**

| 推論引擎 | 類型 | 部署方式 | 適用場景 | 成本模型 | 優先順序 |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Gemini API** (Google) | 雲端 API | SaaS（遠端呼叫） | 高精度號碼布 OCR、多語言支援 | 按 Token/按圖收費 | P0（預設） |
| **GPT-4o Vision** (OpenAI) | 雲端 API | SaaS（遠端呼叫） | 高精度號碼布 OCR、多語言支援 | 按 Token/按圖收費 | P0 |
| **Claude 3.5 Sonnet** (Anthropic) | 雲端 API | SaaS（遠端呼叫） | 號碼布 OCR，含推理過渡解釋 | 按 Token/按圖收費 | P1 |
| **AWS Bedrock** (Claude/Llava) | 雲端 API（託管） | AWS 原生 | AWS 既有帳單整合，適合已用 AWS 的主辦方 | 按用量收費（AWS 帳單） | P1 |
| **Azure OpenAI** (GPT-4V) | 雲端 API | Azure 原生 | 已用 Azure 的企業主辦方 | Azure 帳單整合 | P1 |
| **YOLOv8 + Tesseract OCR** | 本地模型 | 自架伺服器 / Lambda Layer | 追求低延遲、無 API 費用、高隱私（資料不離地） | 一次性模型授權 + GPU 硬體 | P1 |
| **RF-DETR** | 本地模型 | 自架伺服器 / Lambda Layer | 高精度物件偵測，替代 YOLOv8 | 開源模型（Apache 2.0） | P2 |
| **Ollama + LLaVA / LLaMA-OCR** | 本地 LLM | 自架 GPU 伺服器 或 高記憶體 Lambda | 極度成本敏感、須資料落地（不允許影像上雲）之賽事 | 硬體電力成本，無 API 費用 | P2 |
| **vLLM + Qwen2-VL** | 本地 VLM | 自架 GPU 伺服器 | 大規模部署、對延遲和吞吐量有嚴格要求 | 開源模型，GPU 成本 | P2 |
| **SnapSeek 人臉比對引擎** | 雲端 API（專用 Face Re-ID） | SaaS（遠端呼叫） | Face Re-ID Fallback，搭配 OCR 失敗情境（詳見 §3.2） | 按 API 呼叫次數收費 | P0（Face Re-ID 預設） |

> **Face Re-ID 引擎與 OCR 引擎正交：** 上表最後一列 SnapSeek 為獨立的人臉比對引擎（實作 `IFaceMatchAdapter` 介面，與 `IInferenceAdapter` 解耦），與上方 OCR / 物件偵測引擎不共用同一推論管線。OCR 階段失敗後由 Face Re-ID 階段接手，兩者可採用不同推論策略（例：OCR 用 Gemini Flash，Face Re-ID 用 SnapSeek）。

> **美學評分引擎：** GPT-4o Vision、Claude 3.5 Sonnet、Gemini Flash 亦可作為 `IAestheticScoringAdapter` 之引擎，透過 VLM prompt 評分維度（sharpness、composition、lighting、faceClarity）；成本較傳統 OCR 引擎高，建議搭配 `AESTHETIC_SCORING_ENABLED` Flag 使用。

#### **標準化推論介面**

```typescript
// OCR 推論結果（Provider 無關）
interface OCRResult {
  bibNumber: string | null;      // 萃取出的號碼布數字，無法辨識時為 null
  confidence: number;             // 0.0–1.0 置信度
  boundingBox: BoundingBox;       // 號碼布在原圖中的邊界框座標
  processingTimeMs: number;       // 推論耗時（毫秒），用於成本分析
  provider: string;               // 實際使用之推論引擎（如 "gemini-1.5-flash"），僅供日誌
}

// 人臉比對結果（Provider 無關）
interface FaceMatchResult {
  matched: boolean;
  confidence: number;             // 0.0–1.0
  processingTimeMs: number;
  provider: string;
}

// 號碼布偵測結果
interface BibDetectionResult {
  detections: {
    bibCrop: Buffer;              // 裁切後之號碼布圖片區塊（傳給 OCR 使用）
    boundingBox: BoundingBox;
    confidence: number;
  }[];
  processingTimeMs: number;
  provider: string;
}

// AI/LLM 推論抽象層核心介面
interface IInferenceAdapter {
  /** 引擎識別碼 */
  readonly engine: string;

  /**
   * 執行號碼布偵測（物件偵測階段）。
   * 適用於傳統 pipeline（YOLOv8 → Tesseract）之第一階段。
   * @param imageBuffer 原始圖片 Buffer（JPEG/PNG）
   */
  detectBib(imageBuffer: Buffer): Promise<BibDetectionResult>;

  /**
   * 執行 OCR（字元萃取階段）。
   * 適用於傳統 pipeline 之第二階段。
   * @param bibCrop 裁切後之號碼布圖片區塊
   */
  extractBibNumber(bibCrop: Buffer): Promise<OCRResult>;

  /**
   * 一次性端到端辨識（偵測 + OCR 合併執行）。
   * 適用於 LLM-based 引擎（如 Gemini、GPT-4o），一次 API 呼叫即完成
   * 號碼布定位與字元萃取，避免拆分為兩次呼叫造成額外成本與延遲。
   * 傳統 pipeline adapter 可於此方法中內部依序呼叫 detectBib + extractBibNumber。
   * @param imageBuffer 原始圖片 Buffer（JPEG/PNG）
   */
  detectAndExtractBib(imageBuffer: Buffer): Promise<BibDetectionResult & OCRResult>;

  /**
   * 健康檢查：驗證 API 連線 / 本地模型是否正常。
   */
  healthCheck(): Promise<{ ok: boolean; latencyMs: number; message: string }>;

  /**
   * 取得目前推論引擎的估算單張處理成本（USD）。
   * 用於成本監控與引擎選擇參考。
   */
  estimateCostPerImage(): number;
}

// 人臉比對獨立介面（與 OCR/物件偵測引擎正交）
// 人臉比對為 Fallback 階段之獨立能力，非所有推論引擎皆具備此功能，
// 故從 IInferenceAdapter 拆出為獨立介面，由專門的 Face Re-ID 引擎實作。
interface IFaceMatchAdapter {
  /** 引擎識別碼（如 "snapseek", "aws-rekognition"） */
  readonly engine: string;

  /**
   * 執行人臉比對（Fallback 階段）。
   * @param faceCrop 裁切後之人臉圖片區塊
   * @param registeredSelfie 跑者註冊時上傳之清晰自拍照
   */
  matchFace(faceCrop: Buffer, registeredSelfie: Buffer): Promise<FaceMatchResult>;

  /**
   * 健康檢查：驗證人臉比對服務是否正常。
   */
  healthCheck(): Promise<{ ok: boolean; latencyMs: number; message: string }>;
}

// 美學評分獨立介面（與 OCR/Bib 偵測引擎正交）
// 目的：在叢集內選出一張「最美的瞬間」用於推播，而非僅以 OCR 置信度決定。
// 評分維度為客觀可測量指標：銳利度、構圖平衡、光線充足程度、表情清晰度。
interface IAestheticScoringAdapter {
  /** 引擎識別碼（如 "gpt4o-vision", "claude-sonnet", "gemini-flash"） */
  readonly engine: string;

  /**
   * 對照片進行美學評分。
   * @param imageBuffer 候選照片之 Buffer（JPEG/PNG）
   * @returns 美學評分結果，含各維度分數與總分（0–1）
   */
  scoreImage(imageBuffer: Buffer): Promise<AestheticScoreResult>;

  /**
   * 健康檢查。
   */
  healthCheck(): Promise<{ ok: boolean; latencyMs: number; message: string }>;

  /**
   * 估算單張處理成本（USD）。
   */
  estimateCostPerImage(): number;
}

// 美學評分結果
interface AestheticScoreResult {
  overallScore: number;        // 0.0–1.0，綜合評分
  dimensions: {
    sharpness: number;        // 銳利度（解析度/邊緣梯度能量）
    composition: number;       // 構圖平衡（三分法、對角線、主體位置）
    lighting: number;         // 光線充足與均勻程度
    faceClarity: number;     // 人臉表情清晰度（含否為笑容）
    motionClarity: number;    // 肢體動感清晰度（對跑步姿態有意義）
  };
  processingTimeMs: number;
  provider: string;           // 實際使用之推論引擎
}
```

#### **工廠與引擎選擇策略**

```typescript
function createInferenceAdapter(
  strategy: 'cost_optimized' | 'latency_optimized' | 'privacy_first' | 'accuracy_first' | string,
  eventConfig: EventConfig
): IInferenceAdapter {
  const engineMap: Record<string, IInferenceAdapter> = {
    // 成本優先：本地 Tesseract OCR，無 API 費用
    'cost_optimized':       new LocalYoloTesseractAdapter(),
    // 隱私優先：嚴格資料落地，完全不呼叫外部 API
    'privacy_first':        new OllamaLLaVAAdapter(),
    // 延遲優先：雲端低延遲 API
    'latency_optimized':    new GeminiFlashAdapter(),
    // 精度優先：最大最強模型
    'accuracy_first':       new ClaudeSonnetAdapter(),
  };

  // 可在 eventConfig 中指定明確引擎覆寫（後台可設定）
  if (eventConfig.inferenceEngineOverride) {
    return new ExplicitEngineAdapter(eventConfig.inferenceEngineOverride);
  }

  return engineMap[strategy] ?? engineMap['cost_optimized'];
}
```

#### **多引擎級聯降級（Cascade Fallback）**

為同時滿足 SLA 延遲目標與成本控制，系統支援「多引擎級聯降級」策略。Cascade 各階段採用**獨立階段閾值（stageThresholds）**，與全域 `confidenceThreshold` (legacy, 單引擎模式) **完全分離**：

1. **第一階段（≤ 30 秒）：** 嘗試本地 YOLOv8 + Tesseract OCR（免費、低延遲）。**採用門檻 = `stageThresholds[0]`**（預設 0.5，較寬鬆以保留低品質本地辨識結果）
2. **第一階段失敗（`confidence < stageThresholds[0]`）或超時：** 切換至 Gemini Flash API（低成本雲端 OCR）
3. **第二階段仍失敗（`confidence < stageThresholds[1]`）：** 升級至 Claude 3.5 Sonnet（高精度推理）。**採用門檻 = `stageThresholds[1]`**（預設 0.7，較嚴格以僅接受高品質雲端結果）
4. **最終失敗：** 寫入 DLQ「AI 推論耗盡」，通知人工處理

每個階段的 `maxRetries`、`timeoutMs`、`stageThresholds[i]` 皆由 `eventConfig.inferenceStrategy.cascadeFallback` 參數化控制。**重要：`stageThresholds[i]` 各階段獨立設定，不應假設與全域 `confidenceThreshold` 相同**——前者控制 Cascade 內部降級時機，後者僅在非 Cascade 模式（單引擎）下生效。

```typescript
interface CascadeConfig {
  enabled: boolean;                        // 是否啟用 Cascade 多引擎模式（預設 true）
  stages: CascadeStage[];                  // 各階段依序執行
  globalTimeoutMs: number;                 // 所有階段合計的全局 timeout(預設 240000 = 4 分鐘)

  // 階段閾值陣列(向後相容語法;等同於 stages[i].fallbackOnConfidence)
  // 預設 [0.5, 0.7]，第一階段寬鬆、第二階段嚴格
  stageThresholds?: number[];
}

interface CascadeStage {
  engineType: string;                      // 該階段使用之引擎(如 "yolov8-tesseract", "gemini-flash", "claude-sonnet")
  maxRetries: number;                      // 該階段內部重試次數
  timeoutMs: number;                       // 該階段單次 timeout(毫秒)
  fallbackOnConfidence: number;            // 該階段 confidence 低於此值時降級至下一階段
}
```

#### **本地部署模型的管理**

若賽事主辦單位選擇「隱私優先」或「成本優先」策略，需在機房或本地 GPU 伺服器部署 Ollama 或 vLLM。模型管理由以下機制支撐：

- **模型註冊表（Model Registry）：** DynamoDB 儲存 `{modelKey, modelVersion, sha256, deployedAt, status}`，每次部署前核對 SHA-256 防篡改
- **藍綠部署（Blue/Green）：** 新模型驗證通過後，透過 Feature Flag 逐步切換流量，避免一次性全量上線
- **模型監控：** 每個推理請求寫入 CloudWatch，記錄 `engine`、`latencyMs`、`confidence`、`costEstimate`，供成本分析與 SLA 報告使用

#### **新增推論引擎流程**

未來新增推論引擎時，開發流程為：

1. 於 `inference-adapters/` 目錄下新增 `{engineName}.adapter.ts`，實作 `IInferenceAdapter` 介面
2. 向工廠函數 `createInferenceAdapter` 新增策略映射
3. 於後台管理系統之「AI 推論引擎設定」頁面新增該引擎選項（可設定為賽事專用）
4. 撰寫單元測試（Mock API 回應）與整合測試（實際呼叫引擎端點）

此流程屬純水平擴充，核心 AI 管線（YOLOv8 → OCR → Face Re-ID）完全不需修改。

### **2.11 功能開關與配置抽象層**

每場賽事的需求、預算與技術成熟度各異。賽事主辦單位應能自由啟用或關閉特定功能，而無需重新部署程式碼。本系統定義「功能開關與配置抽象層」，以集中式配置管理所有功能的啟用狀態、參數調控與未來擴充點。

#### **配置模型定義**

```typescript
// 賽事功能配置（EventFeatureConfig）
interface EventFeatureConfig {
  eventId: string;

  // --- 社群平台推播開關 ---
  platforms: {
    line:    PlatformConfig;    // LINE 推播啟用/停用
    instagram: PlatformConfig;  // Instagram 推播啟用/停用
    threads: PlatformConfig;    // Threads 推播啟用/停用
    facebook: PlatformConfig;   // Facebook 推播啟用/停用（預設關閉）
    x:        PlatformConfig;   // X (Twitter) 推播啟用/停用（預設關閉）
    bluesky:  PlatformConfig;  // Bluesky 推播啟用/停用（預設關閉）
  };

  // --- AI 辨識功能開關 ---
  ai: {
    ocrEnabled: boolean;                    // 號碼布 OCR 啟用
    faceReIdEnabled: boolean;              // Face Re-ID Fallback 啟用（須用戶另行同意）
    inferenceEngine: string;               // 推論引擎策略（見 2.10）
    inferenceStrategy: InferenceStrategy;   // cost_optimized / latency_optimized / privacy_first / accuracy_first
    // 單引擎 OCR 結果採用門檻(legacy threshold,僅供單一引擎模式下參考)
    // 註:Cascade 多引擎模式之下,各階段採用 cascadeFallback.stageThresholds 陣列控制
    //   此欄位保留以維持向後相容,新部署應使用 stageThresholds
    confidenceThreshold: number;           // 單引擎 OCR 置信度閾值(預設 0.7),僅單一引擎時生效
    faceReIdConfidenceThreshold: number;   // Face Re-ID 置信度閾值(預設 0.75)
    cascadeFallback: CascadeConfig;        // 級聯降級設定(多引擎模式時以 stageThresholds 為準)
  };

  // --- 照片美化與渲染 ---
  rendering: {
    watermarkEnabled: boolean;             // 浮水印啟用
    sponsorFrameEnabled: boolean;         // 贊助商相框啟用
    paceOverlayEnabled: boolean;          // 配速文字疊加啟用
    bibNumberOverlayEnabled: boolean;    // 號碼布數字疊加啟用
    filterStyle: 'none' | 'vintage' | 'cinematic' | 'custom'; // 相片濾鏡風格
  };

  // --- 完賽報紙 ---
  newspaper: {
    enabled: boolean;                     // 完賽報紙生成啟用
    templateId: string;                  // PDF 模板 ID
    includeSplitsChart: boolean;        // 包含分段配速圖表
    includeSocialMetrics: boolean;      // 包含社群互動指標
    deliveryDelayMinutes: number;       // 賽後多少分鐘後開始生成（預設 30）
  };

  // --- DLQ 處理 ---
  dlq: {
    enabled: boolean;                    // DLQ 功能啟用
    autoEmailNotification: boolean;     // DLQ 累積超標時自動通知主辦單位
    autoEmailThreshold: number;         // 觸發通知的 DLQ 數量閾值（預設 100）
  };

  // --- 延遲 SLA ---
  sla: {
    targetP95Minutes: number;           // 目標 P95 延遲（分鐘），預設 5
    hardCapMinutes: number;            // 困難場景最大容忍時間（分鐘），預設 15
  };

  // --- 擴充區段（未来功能在此新增）---
  extensions: Record<string, unknown>;
}
```

#### **Feature Flag 管理機制**

系統採用 AWS AppConfig（或同類型 Feature Flag 服務）作為 Feature Flag 的集中管理後端，支援：

- **即時熱切換（Real-time）：**  flag 變更後，Lambda 函數在 30 秒內感知，無需重新部署
- **分段放量（Gradual Rollout）：**  新功能可先對 1% 流量啟用，逐步擴大至 100%
- **Targeting Rules：** 可依據跑者區域、參賽組別、號碼布範圍等條件，針對特定族群啟用功能
- **Audit Log：** 所有 flag 變更皆寫入 CloudTrail，具備時間戳、操作者、變更前後值

#### **預設功能目錄**

以下為目前已實作之功能開關，未來新增功能皆依相同模式擴充：

| 功能代碼 | 功能名稱 | 預設狀態 | 說明 |
| :---- | :---- | :---- | :---- |
| `PLATFORM_LINE` | LINE 推播 | ✅ 開啟 | 標配平台 |
| `PLATFORM_INSTAGRAM` | Instagram 推播 | ✅ 開啟 | 標配平台 |
| `PLATFORM_THREADS` | Threads 推播 | ✅ 開啟 | 標配平台 |
| `PLATFORM_FACEBOOK` | Facebook 推播 | ❌ 關閉 | 須主動啟用 |
| `PLATFORM_X` | X (Twitter) 推播 | ❌ 關閉 | 須主動啟用 |
| `AI_OCR` | 號碼布 OCR 辨識 | ✅ 開啟 | 無法關閉（為核心功能） |
| `AI_FACE_REID` | Face Re-ID Fallback | ✅ 開啟 | 須用戶另行同意（PDPA） |
| `RENDER_WATERMARK` | 浮水印疊加 | ✅ 開啟 | 可上傳自訂浮水印 |
| `RENDER_SPONSOR_FRAME` | 贊助商相框 | ✅ 開啟 | |
| `RENDER_PACE_OVERLAY` | 配速文字疊加 | ✅ 開啟 | |
| `RENDER_FILTER` | 相片濾鏡風格 | ❌ 關閉 | 須主動啟用 |
| `NEWSPAPER_ENABLED` | 完賽報紙生成 | ✅ 開啟 | |
| `NEWSPAPER_SPLITS_CHART` | 完賽報紙含分段圖表 | ✅ 開啟 | |
| `NEWSPAPER_SOCIAL_METRICS` | 完賽報紙含社群指標 | ❌ 關閉 | 須主動啟用 |
| `DLQ_ENABLED` | DLQ 人工處理機制 | ✅ 開啟 | |
| `PHOTO_BURST_DEDUP` | 連拍叢集去重（取最高置信度一張推播） | ✅ 開啟 | 主辦單位可關閉以對所有連拍照片均執行推播 |
| `GROUP_PHOTO_DEDUP` | 集團通過去重（防止同一照片重複推播） | ✅ 開啟 | |
| `DLQ_AUTO_EMAIL` | DLQ 超標自動通知 | ✅ 開啟 | |
| `INSTAGRAM_STORY_TEMPLATE` | IG 限動自動模板包（12 種女性向設計） | ❌ 關閉 | 主辦單位啟用後，跑者於報名時可選 2–3 個偏好模板，推播時直接生成 ready-to-share 限動圖 |
| `AESTHETIC_SCORING_ENABLED` | 美學評分模型（最美一張決策） | ❌ 關閉 | 結合 OCR 置信度與美學客觀分數加權選圖；需搭配 `IAestheticScoringAdapter` |
| `EVENT_BADGE_SYSTEM` | 賽事挑戰賽徽章系統 | ❌ 關閉 | 偵測姊妹群組集體完賽後自動頒發可分享徽章圖，附加於推播訊息中 |
| `SISTER_GROUP_NOTIFY` | 姊妹群組推播通知 | ❌ 關閉 | 推播時同步通知跑者預填之 1–3 位朋友帳號；需朋友同意外部通知（PDPA 第三人通知合規流程另訂） |

#### **擴充點：未來功能接入流程**

未來任何新功能或新平台支援，均須依以下流程接入 Feature Flag 系統，確保可被主辦單位動態控制：

1. **定義 Feature Flag 代碼：** 於 `feature-flags.ts` 枚舉中新增 `{FEATURE_NAME}`，並填入 `defaultValue`（預設狀態）
2. **實作功能開關護衛（Guard）：** 在 Lambda 函數中，以 `AppConfig.evaluate(ruleId, flagKey)` 包圍功能邏輯；flag 為 `false` 時跳過該功能
3. **註冊至 Feature Flag 管理後台：** 後台管理系統之「功能開關」頁面自動讀取所有已註冊 flag，以 Toggle Switch 形式呈現，並附功能說明
4. **寫入監控指標：** 每個 flag 變更時，寫入 CloudWatch 自訂指標 `feature_flag_active_{flagKey}`，供 SLA 報告追蹤各功能的實際使用率
5. **列入 Release Notes：** 每次新增 flag 須同步更新後台 Release Notes 頁面，說明新功能用途與預設狀態

#### **後台配置 API**

| 端點 | 方法 | 說明 |
| :---- | :---- | :---- |
| `/admin/v1/events/{eventId}/config` | GET | 取得該賽事完整功能配置 |
| `/admin/v1/events/{eventId}/config` | PATCH | 部分更新功能配置（僅變更傳入的欄位，支援熱切換） |
| `/admin/v1/events/{eventId}/config/features` | GET | 列出所有可用功能開關與目前狀態 |
| `/admin/v1/feature-flags` | GET | 列出系統全部 Feature Flag（含已停用之歷史 flag） |

#### **§2.11 Feature Flag 全目錄彙總（v1.5 新增）**

本章節列出之 flag 為「系統定義目錄」（canonical set）。各章節新增之 flag 應於本表補登，避免散落於各章節。

| # | 功能代碼 | 章節 | 預設狀態 | 用途 |
|:----|:----|:----|:----|:----|
| 1 | `PLATFORM_LINE` | §2.11 | ✅ 開啟 | LINE 推播標配 |
| 2 | `PLATFORM_INSTAGRAM` | §2.11 | ✅ 開啟 | Instagram 推播標配 |
| 3 | `PLATFORM_THREADS` | §2.11 | ✅ 開啟 | Threads 推播標配 |
| 4 | `PLATFORM_FACEBOOK` | §2.11 | ❌ 關閉 | Facebook 推播 |
| 5 | `PLATFORM_X` | §2.11 | ❌ 關閉 | X (Twitter) 推播 |
| 6 | `AI_OCR` | §2.11 | ✅ 開啟（核心，無法關閉） | 號碼布 OCR |
| 7 | `AI_FACE_REID` | §2.11 | ✅ 開啟（須用戶同意） | Face Re-ID Fallback |
| 8 | `RENDER_WATERMARK` | §2.11 | ✅ 開啟 | 浮水印 |
| 9 | `RENDER_SPONSOR_FRAME` | §2.11 | ✅ 開啟 | 贊助商相框 |
| 10 | `RENDER_PACE_OVERLAY` | §2.11 | ✅ 開啟 | 配速文字疊加 |
| 11 | `RENDER_FILTER` | §2.11 | ❌ 關閉 | 相片濾鏡 |
| 12 | `NEWSPAPER_ENABLED` | §2.11 | ✅ 開啟 | 完賽報紙生成 |
| 13 | `NEWSPAPER_SPLITS_CHART` | §2.11 | ✅ 開啟 | 報紙含分段圖 |
| 14 | `NEWSPAPER_SOCIAL_METRICS` | §2.11 | ❌ 關閉 | 報紙含社群指標 |
| 15 | `DLQ_ENABLED` | §2.11 | ✅ 開啟 | DLQ 機制 |
| 16 | `PHOTO_BURST_DEDUP` | §2.11 | ✅ 開啟 | 連拍叢集去重 |
| 17 | `GROUP_PHOTO_DEDUP` | §2.11 | ✅ 開啟 | 集團通過去重 |
| 18 | `DLQ_AUTO_EMAIL` | §2.11 | ✅ 開啟 | DLQ 超標通知 |
| 19 | `INSTAGRAM_STORY_TEMPLATE` | §2.11 / §3.3 | ❌ 關閉 | IG 限動模板包 |
| 20 | `AESTHETIC_SCORING_ENABLED` | §2.11 / §3.7 | ❌ 關閉 | 美學評分模型 |
| 21 | `EVENT_BADGE_SYSTEM` | §2.11 | ❌ 關閉 | 賽事挑戰徽章 |
| 22 | `SISTER_GROUP_NOTIFY` | §2.11 | ❌ 關閉 | 姊妹群組通知 |
| 23 | `POST_PUSH_WITHDRAWAL_WINDOW` | §3.1.Y | ✅ 開啟 | 推播後撤回視窗 |
| 24 | `PUSH_PREVIEW_DUAL_CONSENT` | §3.1.Y | ✅ 開啟 | 推播雙重確認 |
| 25 | `PLATFORM_FALLBACK_CHAIN` | §3.9 | ✅ 開啟 | 平台降級鏈 |
| 26 | `META_APP_REVIEW_STATUS` | §3.9 | ✅ 開啟 | Meta App Review 監控 |
| 27 | `THREADS_RATE_LIMIT_BUFFER` | §3.9 | ✅ 開啟 | Threads 限流緩衝 |
| 28 | `MULTI_ACCOUNT_PUBLISH` | §3.9 | ❌ 關閉 | 多帳號分散推播 |
| 29 | `GALLERY_FALLBACK_ALWAYS` | §3.9 | ✅ 開啟 | Gallery 永久降級 |

**Flag 變更流程（v1.5 補充）：** 未來新增 flag 時，請同步更新本表與對應章節之定義表格，避免出現「章節有但本表無」或反之的不一致狀態。命名規範：`SCREAMING_SNAKE_CASE`，前綴分類為 `PLATFORM_*` / `AI_*` / `RENDER_*` / `NEWSPAPER_*` / `DLQ_*`。

### **2.12 共用型別定義（Shared Type Definitions）**

本節集中定義規格書各章節所引用之共用型別，確保實作時無歧義。

#### **配置型別層級關係**

三個核心配置型別之關係如下：

- `EventConfig`：賽事頂層配置，聚合所有子配置
- `EventTimingConfig`：計時系統相關配置，為 `EventConfig` 的子物件
- `EventFeatureConfig`：功能開關配置（已於 §2.11 定義），為 `EventConfig` 的子物件

```typescript
// 賽事頂層配置
interface EventConfig {
  eventId: string;
  eventName: string;
  eventDate: string;              // ISO 8601
  eventType: 'marathon' | 'half_marathon' | 'triathlon' | 'cycling' | 'custom';
  maxParticipants: number;
  timezone: string;               // IANA timezone (e.g., "Asia/Taipei")
  timing: EventTimingConfig;      // 計時系統配置
  features: EventFeatureConfig;   // 功能開關配置（§2.11）
  inferenceEngineOverride?: string; // 明確覆寫推論引擎（覆蓋 strategy）
  inferenceStrategy: 'cost_optimized' | 'latency_optimized' | 'privacy_first' | 'accuracy_first';
}

// 計時系統配置
interface EventTimingConfig {
  adapterType: string;            // Adapter 類型識別碼（如 "mylaps", "championchip"）
  apiEndpoint?: string;           // 外部計時系統 API 端點 URL
  apiKey?: string;                // 外部系統 API 金鑰（加密存儲）
  ftpHost?: string;               // FTP 模式之主機位址
  ftpCredentials?: string;        // FTP 認證資訊（加密存儲）
  chipToBibMappingTable: string;  // DynamoDB 中 chipId → bibNumber 查詢表名稱
  checkpointCodes: string[];      // 該賽事定義之所有檢查點代碼
}
```

#### **平台相關型別**

```typescript
// 單一平台啟用配置
interface PlatformConfig {
  enabled: boolean;               // 該平台是否啟用
  appId?: string;                 // 平台 App ID（如 Meta App ID）
  appSecret?: string;             // 平台 App Secret（加密存儲）
  redirectUri?: string;           // OAuth Redirect URI
  additionalScopes?: string[];    // 額外需要的 OAuth scope
}

// 跑者於特定平台的設定
interface RunnerPlatformConfig {
  runnerId: string;
  platform: string;
  platformUserId: string;         // 平台用戶 ID（如 LINE User ID、IG Business Account ID）
  platformUsername?: string;      // 平台用戶名稱（供顯示用）
  linkedPageId?: string;          // Instagram 需連結之 Facebook 粉絲專頁 ID
}

// 平台 OAuth 憑證（由 Token Manager 解密後提供）
interface PlatformCredentials {
  accessToken: string;
  refreshToken?: string;
  expiresAt: string;              // ISO 8601 過期時間
  tokenType: string;              // 通常為 "Bearer"
  scopes: string[];               // 已授權之 scope 列表
}

// 平台 API 錯誤
interface PlatformApiError {
  statusCode: number;             // HTTP 狀態碼（401, 403, 429, 5xx）
  errorCode?: string;             // 平台自訂錯誤碼
  errorMessage: string;
  retryAfter?: number;            // 429 時的 Retry-After 秒數
  platform: string;
}

// Token 狀態
type TokenStatus = 'active' | 'needs_reauth' | 'revoked' | 'expired';
```

#### **AI 推論相關型別**

```typescript
// 邊界框座標
interface BoundingBox {
  x: number;        // 左上角 x 座標（像素）
  y: number;        // 左上角 y 座標（像素）
  width: number;    // 寬度（像素）
  height: number;   // 高度（像素）
  confidence: number; // 偵測置信度 0.0–1.0
}

// 推論策略配置
interface InferenceStrategy {
  primary: string;                // 主推論引擎名稱
  fallbackChain: string[];        // 降級引擎鏈（依序嘗試）
  globalTimeoutMs: number;        // 全局 timeout budget（所有階段合計）
}

// 級聯降級配置
interface CascadeConfig {
  stages: {
    engine: string;               // 推論引擎名稱
    maxRetries: number;           // 該階段最大重試次數
    timeoutMs: number;            // 該階段 timeout（毫秒）
    fallbackOnConfidence: number; // 置信度低於此值時觸發降級
  }[];
  globalTimeoutMs: number;        // 所有階段合計的全局 timeout（毫秒），預設 240000（4 分鐘）
}

// 模型輸出（Provider 無關）
interface ModelOutput {
  predictions: {
    label: string;                // 類別標籤（如 "bib", "runner", "face"）
    boundingBox: BoundingBox;
    confidence: number;
  }[];
  processingTimeMs: number;
}

// 模型元資料
interface ModelMetadata {
  modelKey: string;
  modelVersion: string;
  sha256: string;                 // 模型檔案 SHA-256 校驗碼
  deployedAt: string;             // ISO 8601
  inputResolution: string;        // 如 "640x640"
  framework: string;              // 如 "onnx", "pytorch", "tensorflow"
}
```

#### **雲端基礎設施相關型別**

```typescript
// 上傳選項
interface UploadOptions {
  contentType?: string;           // MIME type
  acl?: 'private' | 'public-read';
  metadata?: Record<string, string>;
  cacheControl?: string;
}

// 事件觸發配置
interface TriggerConfig {
  type: 'queue' | 'storage_event' | 'http' | 'schedule';
  sourceArn?: string;             // 事件來源 ARN/URI
  batchSize?: number;
  maxConcurrency?: number;
}

// 佇列訊息
interface QueueMessage {
  messageId: string;
  body: unknown;
  receiptHandle: string;          // 用於 acknowledge
  sentTimestamp: string;
  approximateReceiveCount: number;
}
```

#### **Secrets Manager 設計（敏感憑證集中管理）**

所有第三方平台憑證統一存於 AWS Secrets Manager，Lambda 透過 SecretsManagerClient 於啟動時讀取並 in-memory 快取（單次執行週期）。嚴禁將 Secret 寫入環境變數、S3、DynamoDB 或 CloudWatch Logs。

**Secrets Path 命名規範：**

```
{env}/events/{eventId}/secrets/{secretName}
```

範例：
- `prod/events/event-2026-taipei-marathon/secrets/line-channel-secret`
- `prod/events/event-2026-taipei-marathon/secrets/meta-app-secret`
- `prod/events/event-2026-taipei-marathon/secrets/smtp-password`
- `prod/events/event-2026-taipei-marathon/secrets/snapseek-api-key`
- `dev/events/event-test-run/secrets/line-channel-secret`

**Rotation 政策：**

| Secret | Rotation 週期 | Rotation 機制 |
| :---- | :---- | :---- |
| `line-channel-secret` | 90 天 | LINE 後台手動產生新 Secret → Terraform 更新 Secrets Manager |
| `meta-app-secret` | 180 天 | Meta App Dashboard 重新產生 |
| `smtp-password` | 90 天 | SES SMTP Credential v2 重新產生 |
| `snapseek-api-key` | 365 天 | SnapSeek 後台手動 |

**IAM Policy（最小權限）：**

```json
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue"],
  "Resource": "arn:aws:secretsmanager:{primary_region}:{account}:secret:{env}/events/*/secrets/*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/DeploymentTopology": "${var.deployment_topology}"
    }
  }
}
```

Lambda 不得具備 `secretsmanager:PutSecretValue`、`secretsmanager:DeleteSecret`、`secretsmanager:RotateSecret` 等寫入權限（僅 Infra/Terraform 角色具備）。

**KMS 加密：** 每個 Secret 以獨立的 CMK 加密（共用 §2.5 之 `eventId-cmk`）,Resource Policy 限制僅該 eventId 之 Lambda 可解密。

## **3\. 功能性需求 (Functional Requirements)**

本系統的業務邏輯高度複雜且具備多樣性，涵蓋從賽前的合規註冊與授權、賽中極高頻率的 AI 運算，到賽果客製化媒體產出的完整生命週期。以下針對五大核心功能進行深度的架構與實作需求解析。

### **3.1 賽前註冊系統：動態 API 模組設計與嚴謹合規授權**

賽前註冊系統是建立跑者數位身分與後續自動推播功能的基石。跑者必須能夠透過響應式網頁介面（RWD）輸入其專屬的參賽號碼布編號，並進行主流社群帳號的深度綁定。

#### **前端 Hosting 規格**

| 前端應用 | Hosting 方案 | 技術棧 | Custom Domain | CI/CD |
| :---- | :---- | :---- | :---- | :---- |
| **跑者註冊頁**（公開） | AWS Amplify Hosting | React 18 + Next.js 14（SSR）+ LIFF SDK（LINE Front-end Framework） | `register.{eventId}.imarathon.app` | GitHub Actions → Amplify auto-deploy |
| **跑者 Gallery 頁**（公開） | S3 + CloudFront（靜態 SPA） | React 18 + Vite | `gallery.{eventId}.imarathon.app` | GitHub Actions → `aws s3 sync` + CloudFront invalidation |
| **完賽報紙下載頁**（公開） | S3 + CloudFront（純 HTML + JS） | 純 HTML + Alpine.js（最小化） | `newspaper.{eventId}.imarathon.app` | 同上 |
| **後台 Admin Dashboard**（內部） | S3 + CloudFront（靜態 SPA）+ Cognito 認證 | React 18 + Vite + Ant Design | `admin.{eventId}.imarathon.app` | GitHub Actions + manual approval |

**選擇理由：**
- **Amplify** 提供 SSR、Custom Domain、TLS、CI/CD 整合;適合需要 SSR 處理 OAuth Callback 與 SEO 之註冊頁
- **S3 + CloudFront** 為純靜態資產,成本最低、CDN 全球 edge cache;適合 Gallery / 報紙頁 / 後台
- **後台採 S3 + CloudFront 而非 Amplify**:後台無 SSR 需求,但需透過 CloudFront + Lambda@Edge 加上 Cognito 認證 Header 才可存取

**TLS 憑證：** 全部使用 AWS Certificate Manager (ACM) 簽發之免費 DV (Domain Validation) 憑證,自動 renewal。`*.{eventId}.imarathon.app` 之 wildcard 憑證由 Super Admin 預先申請。

**LIFF（LINE Front-end Framework）整合：** 註冊頁透過 LIFF SDK 於 LINE App 內開啟時,自動取得 LINE User ID（免使用者手動輸入）,直接送入 `POST /api/v1/races/{eventId}/register` 完成綁定。

註冊階段亦須同時揭露「服務限制與不可抗力因素告知」。本系統雖以全自動化為目標，惟仍存在以下可能導致照片無法順利交付之情境，須於賽前註冊頁面以顯著方式告知跑者：天候因素（暴雨、濃霧、高溫導致設備或跑者受限）、人群過度密集導致號碼布遭遮蔽、號碼布污損或字樣模糊（泥濘、汗漬、摺皺）、號碼布遭外套或背心遮蓋、拍攝角度過度傾斜（超過 75°）、突發性硬體故障或 5G 網路瞬斷，以及其他非可控之環境因素。若因上述因素導致 OCR 辨識失敗，系統將自動降級至圖庫模式，跑者仍可透過傳統關鍵字搜尋機制於線上圖庫中自行查找照片，但不再享有自動化社群推播服務。為應對未來不斷演進的社群平台生態與 API 變更，系統架構必須採用模組化的轉接器設計模式（Adapter Pattern）與標準的開放授權（OAuth 2.0）框架，將各平台的驗證流程予以抽象化。  
在社群 API 模組設計上，必須針對各平台的特殊要求進行定製化與嚴密的防錯處理。以 Instagram 為例，其 API 整合的門檻極高。根據 Meta Graph API 的規範，內容發布功能（Content Publishing）僅支援 Instagram 的「商業帳號（Business）」或「創作者帳號（Creator）」，完全排除了標準的個人帳號13。此外，該專業帳號必須明確連結至一個 Facebook 粉絲專頁，否則無法透過 API 進行發布13。因此，系統在註冊階段的 UX/UI 設計中，必須主動偵測使用者的帳號屬性；若為一般個人帳號，系統需提供清晰且友善的圖文引導流程，協助跑者在 Instagram 設定中切換帳號屬性並完成粉絲專頁連結，方能取得 instagram\_business\_content\_publish 與 instagram\_business\_basic 權限13。  
在 LINE 平台的整合上，系統將利用 Messaging API 的推播訊息（Push Message）功能。跑者在註冊時需加入賽事專屬的 LINE 官方帳號為好友，系統藉此取得該跑者的唯一使用者 ID（User ID）22。此 ID 將作為後續點對點推播影像訊息（Image Message）或客製化圖文選單（Imagemap Message）的目標位址14。  
此外，遵循台灣《個人資料保護法》（PDPA），賽前註冊系統必須具備極為嚴謹的隱私權同意與告知機制。系統收集之資料包含跑者的真實姓名、精確定位資料、影像，甚至潛在的生理特徵（如後續啟用的人臉辨識），這些均屬於受高度管制的個人資料甚至特種個人資料範圍24。系統不得採用預設勾選或將同意條款包裹於一般賽事報名條款中（Bundled Consent）27。必須以獨立的隱私聲明頁面，明確告知蒐集目的、資料利用範圍、傳輸對象以及跑者依法可行使之存取、更正與刪除權利25。為了負起法定的舉證責任（Burden of Proof），系統資料庫需精確記錄每一次同意操作的時間戳記、IP 位址與同意條款版本，確保日後受主管機關稽核時，具備無懈可擊的數位證據力27。

#### **多司法管轄區合規設計（§3.1.X）**

本系統部署範圍橫跨台灣、日本、歐盟、中國大陸等司法管轄區，各區域對臉部辨識（生物特徵資料）之處理、跨境傳輸、自動化決策皆有不同強制規範。本節定義各區域之差異化合規設計，使系統於單一 codebase 內同時滿足多重法域要求。

**適用法域與強制規範對照：**

| 法域 | 主要法規 | 對本系統之強制要求 | SPEC 對應設計 |
|:----|:----|:----|:----|
| **台灣 (TW)** | 《個人資料保護法》(PDPA) | 特種個人資料(§6)須取得當事人書面同意;跨境傳輸(§21)原則禁止,例外須經明確同意 | §4.3 已涵蓋 PDPA §21 跨境同意;§4.7 Face Re-ID 自拍照 24hr 自動刪除 |
| **日本 (JP)** | 《個人情報保護法》(APPI) 2026 修法(內閣 4/7 通過、提交國會) | 新增「Specified Biometric Personal Information」類別;臉部特徵資料**禁止以 opt-out 機制移轉第三方** | 本節 §3.1.X 新增 `biometricConsent.thirdPartyTransferOptIn` 強制 opt-in |
| **歐盟 (EU)** | GDPR Art. 9(2)(a) + Art. 6(1)(a) | 生物辨識資料處理須「明確同意」(explicit consent),嚴格度高於一般同意 | 本節 §3.1.X 新增 `biometricConsent.explicitConsentRequired` |
| **中國大陸 (CN)** | 《個人信息保護法》(PIPL) | 敏感個人生物特徵資訊須取得「單獨同意」;不得超過最小必要原則 | 本節 §3.1.X 新增 `consentDesign.cn.bundled=false` + 最小化蒐集原則 |

**標準化合規資料模型：**

```typescript
// 司法管轄區合規設定（JurisdictionCompliance）
interface JurisdictionCompliance {
  eventId: string;
  applicableRegions: ('tw' | 'jp' | 'eu' | 'cn' | 'global')[];
  defaultRegion: 'tw' | 'jp' | 'eu' | 'cn';   // 賽事主辦單位所在地

  // 依區域分層之同意設計
  consentDesign: {
    tw: ConsentConfig;      // PDPA — 獨立同意 + 跨境傳輸同意
    jp: ConsentConfig;      // APPI — opt-in(非 opt-out)+ 揭露移轉對象
    eu: ConsentConfig;      // GDPR — 須 explicit consent,不得包裹於一般條款
    cn: ConsentConfig;      // PIPL — 須單獨同意 + 最小化蒐集原則
  };

  // 臉部辨識特別條款(僅於 faceReIdConsent=true 時適用)
  biometricConsent: {
    explicitConsentRequired: boolean;        // GDPR Art. 9(2)(a) — EU 市場必須 true
    thirdPartyTransferOptIn: boolean;        // 日本 APPI 修法 — JP 市場必須 true
    dataRetentionDays: number;               // 預設 30 天自動刪除自拍照
    deletionProofRecord: boolean;            // 刪除時記錄 IP + 時間戳(舉證責任)
    conditionalUseOnly: boolean;             // 僅得用於明確告知之目的(不得流用)
  };

  // 跨境傳輸同意書(獨立欄位)
  crossBorderTransferConsent: {
    destinationCountries: string[];          // ['US'(Meta 資料中心), 'SG'(LINE)]
    transferMechanism: 'SCC' | 'adequacy' | 'BCR' | 'PDPA_§21_consent';
    userAcknowledged: boolean;
    acknowledgedAt?: string;                 // ISO 8601
    ipAddress?: string;
  };
}

interface ConsentConfig {
  version: string;                           // e.g. 'pdpa-2026-v1'
  isBundled: false;                          // 強制 false — 不可與報名條款包裹
  consentTimestamp: string;
  ipAddress: string;
  withdrawalEndpoint: string;                // '/api/v1/races/{eventId}/runner/{bib}/revoke'
  withdrawalEffect: 'immediate_purge' | 'race_end_purge';
  policyVersion: string;                     // 隱私政策版本,供日後版本對照
}
```

**日本市場進入條件（Feature Flag 鎖定）：**

當 `EventConfig.region === 'jp'` 時,系統於初始化時強制設定以下約束（主辦方不得覆寫）:

```typescript
// 日本市場 Feature Flag 鎖定 — 不可由主辦方覆寫
const JP_REGION_LOCK: Partial<EventFeatureConfig> = {
  platforms: {
    // 第三方推播平台需用戶另行 opt-in(因 APPI 禁止 opt-out 移轉生物辨識資料)
    line: { enabled: true, requiresExplicitConsent: true },
    instagram: { enabled: true, requiresExplicitConsent: true },
    threads: { enabled: true, requiresExplicitConsent: true },
  },
  ai: {
    faceReIdEnabled: true,
    // 第三方移轉需 opt-in;不得於 Face Re-ID 同意書內以 opt-out 機制處理
    faceReIdThirdPartyTransferRequiresOptIn: true,
  },
};
```

**歐洲市場進入條件：**

當 `EventConfig.region === 'eu'` 時:

- `biometricConsent.explicitConsentRequired` 強制為 `true`
- Face Re-ID Fallback 流程須於「獨立於 OCR 同意的勾選頁」取得 explicit consent
- 禁止任何「預設勾選」、「包裹同意」(bundled consent) UX
- 臉部特徵向量(feature embedding)與自拍照原始檔分開儲存;向量僅得用於當次賽事比對,不得用於跨賽事訓練或留存

**台灣 PDPA 補強要求（基於 §3.1 既有機制擴充）：**

- 「服務限制告知」頁面須新增「臉部辨識拒絕之替代方案」段落,明確告知跑者:拒絕 Face Re-ID 後,系統降級為 OCR-only 模式,仍享有 LINE/IG/Threads 推播
- §4.3 「PDPA §21 跨境傳輸合規」機制對歐盟與日本跑者亦適用(縱使賽事主辦單位位於台灣,跑者於歐盟/日本完成註冊仍須取得獨立跨境同意)

**§4.3 既有 PDPA 表之擴充項目（新增列）：**

| PDPA 法規核心要求 | 系統實作與合規對策 |
|:----|:----|
| **（既有四項保留）** | （詳見 §4.3 原表） |
| **§3.1.X 跨國跑者之 Face Re-ID 同意** | 當跑者 IP 位址或註冊時段判定為歐盟/日本/中國大陸時,Face Re-ID 同意流程須切換至該區域之強制合規模式(見 `JurisdictionCompliance.biometricConsent`) |
| **§3.1.X 賽事主辦方區域感知** | `EventConfig.region` 欄位決定預設同意書語言與法域設計;若賽事為國際賽,系統依跑者 IP 與註冊來源自動選擇對應同意書 |

#### **推播預覽與撤回機制（§3.1.Y）**

基於 deep-research 對論壇接受度與 Photo Create X 帳號「購買前禁止分享」規定的間接推論,跑者對「照片被自動貼到我 IG」之心理門檻高於「自動搜尋」。本節定義雙重確認機制與推播後撤回窗口,以建立 UX 信任。

**雙重確認機制（Dual Consent Flow）：**

```typescript
interface PushPreviewConsent {
  eventId: string;
  runnerId: string;

  // 雙重同意設計 — 兩階段勾選
  dualConsentFlow: {
    // 第一階段:報名時勾選(§3.1 既有機制)
    registrationConsent: {
      granted: boolean;
      grantedAt: string;
      ipAddress: string;
    };

    // 第二階段:首次推播前再行確認
    firstPushPreviewConsent: {
      enabled: boolean;                       // 預設 true
      previewSentAt?: string;
      previewChannel: 'line_push';           // 透過 LINE 推送預覽圖
      previewMessage: string;                 // '將於賽事當天自動將您的完賽照推播至 LINE/IG,是否同意?'
      responseDeadline: string;               // ISO 8601,預設 push 前 60 分鐘
      userResponse?: 'confirmed' | 'declined';
      respondedAt?: string;
    };
  };

  // 推播後撤回窗口
  postPushWithdrawal: {
    enabled: boolean;                       // 預設 true
    windowMinutes: number;                  // 預設 5 分鐘內可撤回
    channel: 'line_reply_keyword' | 'app_button' | 'gallery_link';
    keywordTrigger?: string;                // e.g. '撤回' / '刪除'
    autoDeleteFromPlatform: boolean;        // 是否呼叫 Meta/LINE API 刪除已推播貼文
  };

  // 推播偏好設定(細粒度控制)
  pushPreference: {
    pushAtCheckpoint: boolean;              // 賽道中段是否推播(預設 false,僅終點推播)
    aestheticFilterOnly: boolean;           // 僅推播經美學評分 > 0.7 的照片(預設 true)
    pushMaxPerEvent: number;                // 單場賽事最多推播張數(預設 5)
    quietHoursStart?: string;               // e.g. '23:00'(Asia/Taipei) — 深夜不推播
    quietHoursEnd?: string;
  };
}
```

**撤回流程（Post-Push Withdrawal）：**

1. 推播成功後,系統於 LINE Push Message 內附「撤回」按鈕或關鍵字觸發指令
2. 跑者於 5 分鐘視窗內觸發撤回 → Publish Lambda 接收撤回訊息
3. Publish Lambda 呼叫 `ISocialPlatformAdapter.deletePost(externalPostId)` 從原平台刪除已推播貼文
4. DynamoDB `NormalizedPublishResult.status` 更新為 `withdrawn`,標記撤回時間與操作者
5. CloudWatch 記錄 `metric: push_withdrawal_count` 供賽事後分析 UX 信任度
6. 若撤回時 Meta/LINE API 回 404(已被平台刪除),視為撤回成功,不重試

**§2.11 Feature Flag 新增項目（由 §3.1.Y 觸發）：**

| 功能代碼 | 功能名稱 | 預設狀態 | 說明 |
|:----|:----|:----|:----|
| `PUSH_PREVIEW_DUAL_CONSENT` | 推播雙重確認機制 | ✅ 開啟 | 報名同意後,首次推播前再行確認 |
| `POST_PUSH_WITHDRAWAL_WINDOW` | 推播後撤回視窗 | ✅ 開啟 | 預設 5 分鐘,可由 `windowMinutes` 調整 |
| `PUSH_MAX_PER_EVENT` | 單場推播張數上限 | ✅ 開啟 | 預設 5 張,防止跑者被 spam 推播 |
| `JURISDICTION_LOCK` | 區域合規鎖定 | ❌ 關閉 | 由 `EventConfig.region` 自動觸發;主辦方不得覆寫 |

### **3.2 即時影像處理與辨識流：邊緣到雲端的 AI 協同與極速渲染**

賽事期間的影像處理要求極致的低延遲與極高的準確率。當攝影師按下快門後，影像將透過內建 FTP 或 API 上傳功能的相機，搭配 5G 網路直接推送到指定的雲端 S3 儲存貯列。此舉將觸發 SQS 佇列訊息，進而喚醒 AWS Lambda 函數進行非同步的初步處理8。  
AI 辨識流採用多階段的深度學習管線，並透過 AI/LLM 推論抽象層（見 2.10 章節）動態選擇推論引擎2。本系統首先調用 YOLO 架構（如 YOLOv8 或 RF-DETR）的物件偵測模型進行號碼布定位2。該模型經過包含多種光影條件、跑者姿態與複雜背景（In-the-wild benchmark datasets）的專屬賽事資料集進行微調（Fine-tuning），能迅速在百萬像素的影像中框選出「跑者身軀」，接著再進一步於身軀範圍內定位出「號碼布」的精確邊界框（Bounding Box）29。定位完成後，系統會對號碼布區域進行幾何校正與形態學操作（Morphological Operations），將傾斜或皺褶的區域拉平，最後再將處理後的高對比區塊送入 OCR 引擎提取精確的數字序列2。此種結合特徵金字塔網路（Feature Pyramid Network）與卷積注意力機制（Convolutional Block Attention Module）的方法，能有效排除環境雜訊，將號碼布辨識準確率提升至高達 91.6% 以上的商業標準2。

當 OCR 引擎回傳之置信度分數（Confidence Score）低於預設閾值時（例如低於 0.7），系統將自動觸發**號碼布 OCR 失敗的容錯回退機制（Fallback Mechanism）**：以 YOLOv8/RF-DETR 偵測並裁切跑者臉部區塊，呼叫 SnapSeek 架構之人臉比對引擎，與跑者於註冊時自願上傳之清晰自拍照進行空間幾何特徵比對（需跑者於註冊時另行授權），以多模態融合（Multi-modal）方式提升惡劣環境下的身分對接成功率。若 Face Re-ID 置信度同樣低於閾值，該照片標記為「待人工確認」，寫入 DLQ 由賽事方進行後續處理。
辨識成功並與資料庫比對無誤後，系統隨即進入「動態美化處理」階段。此階段同樣於無伺服器環境中執行，主要利用建置於 Lambda Layer 上的高效能 Node.js 影像處理模組 Sharp 來完成10。數位相機拍攝的原始圖片通常包含 EXIF 旋轉元數據，若不加以處理，合成後的影像可能會呈現倒置；因此，Sharp 會首先呼叫 .rotate() 方法進行 EXIF 自動翻轉校正18。接著，系統會根據賽事主辦方的設定，利用 .composite() 方法動態疊加帶有透明度（Alpha Channel）的 PNG 賽事專屬相框與活動贊助商浮水印18。此外，系統可即時查詢該跑者通過該點位的 RFID 晶片配速與經過時間，並利用文字渲染功能將其實時繪製於相片角落。Sharp 函式庫底層運用 libvips，能在極低的記憶體消耗與毫秒級的時間內完成高品質的影像合成，確保 Lambda 函數不會因超時（Timeout）或記憶體不足而中斷10。

#### **照片處理 SQS 訊息格式（Photo Processing Lambda 輸入）**

當 S3 收到新照片上傳時，S3 Event Notification 將以下訊息寫入 `photo-processing-queue`（`sqs://{eventId}-photo-processing-queue`），隨後由 Photo Processing Lambda 消費：

```json
// SQS Message Body（photo-processing-queue）
{
  "photoId": "uuid-xxxx",
  "s3Uri": "s3://{env}-imarathon-photos/{eventId}/station_01/2026-06-26_05-30-00_001.jpg",
  "eventId": "event-2026-macroformat",
  "stationId": "station_01",
  "capturedAt": "2026-06-26T05:30:00.000Z",
  "photographerId": "PHT-001",
  "checkpointCode": "KM25",
  "traceId": "abc123-def456"
}
```

#### **Bib → Runner 解析流程（OCR 成功後）**

當 OCR 階段成功萃取出 `bibNumber` 後，Photo Processing Lambda 執行以下查詢鏈：

1. **查 DynamoDB `runner-event-index`**（PK: `eventId`, SK: `bibNumber`），取得 `runnerId`。
2. **查 `runner-platform-index`**（PK: `runnerId`），取得該跑者所有已授權的平台與對應的 `platformUserId`（如 LINE User ID、Instagram Business Account ID）。
3. **若任何查詢結果為空**（跑者未報名、或該 bibNumber 無對應跑者），寫入 DLQ 子類型 `runner_not_found`，不執行推播。

此查詢結果同時用於：美化階段疊加跑者姓名（從 runner 資料取得）、寫入 `NormalizedPublishTask.metadata`。

#### **Face Re-ID Lambda 訊息格式（Fallback 觸發後）**

當 OCR 置信度低於閾值時，Photo Processing Lambda 將 Face Re-ID 任務寫入專用 SQS Queue（`sqs://{eventId}-face-reid-queue`）：

```json
// SQS Message Body（face-reid-queue）
{
  "taskId": "uuid-xxxx",
  "photoId": "uuid-yyyy",
  "s3Uri": "s3://{env}-imarathon-photos/{eventId}/station_01/2026-06-26_05-30-00_001.jpg",
  "eventId": "event-2026-macroformat",
  "faceCropS3Uri": "s3://{env}-imarathon-biometric/{eventId}/face-crops/uuid-yyyy.jpg",
  "runnerId": "uuid-zzzz",
  "registeredSelfieS3Uri": "s3://{env}-imarathon-biometric/{eventId}/selfies/uuid-zzzz.jpg",
  "ocrConfidence": 0.45,
  "triggerReason": "ocr_confidence_below_threshold",
  "traceId": "abc123-def456"
}
```

Face Re-ID Lambda 由 `face-reid-queue` 觸發，執行 `SnapSeek.matchFace(faceCrop, registeredSelfie)`，輸出 `{ matched: boolean, confidence: number, processingTimeMs: number }`。若置信度 ≥ `faceReIdConfidenceThreshold`（預設 0.75），則視同識別成功，繼續美化與推播流程。

#### **Face Re-ID 失敗之 Graceful Degradation**

當 Face Re-ID Lambda 仍無法達到置信度閾值時，系統依以下優先順序降級處理，避免單張照片直接進入 DLQ 後人工處理成本過高：

1. **跨幀同跑者投票（Cross-frame Consensus）** — 將此照片的臉部 crop 與同一 `runnerId` 在前 5 分鐘內已成功識別的其他照片進行比對,若 ≥ 2 張照片同意,則採用「群體投票」結果視為匹配（用於跑者服裝一致但單張臉部角度不佳之情境）。
2. **服裝顏色 + 配速 fallback** — 若跨幀投票仍失敗,系統以「號碼布附近的衣服主色（HSV 空間 dominant color）」+「RFID 配速時序」作為輔助識別;當單一跑者於該攝影點前後 30 秒內僅有一名候選人時,自動匹配。
3. **人工確認寫入 DLQ** — 上述均失敗時,將照片標記 `status: pending_manual_review` 並寫入 DLQ;後台 DLQ 處理面板（§3.8）顯示照片縮圖、失敗原因、拍攝時間、攝影師資訊,人工可「一指確認」快速填入 bibNumber。

**跳過 Face Re-ID 之情境：**

- `AI_FACE_REID` Feature Flag 關閉 → Photo Processing Lambda 直接跳過 Face Re-ID Fallback,置信度低於閾值之照片直接寫入 DLQ。
- 跑者於註冊時未上傳自拍照（`runner.faceReIdConsent = false`） → 同上跳過。
- SnapSeek API 回 5xx 錯誤連續 3 次 → 自動降級為「僅 OCR」模式,Face Re-ID Lambda 跳過,直接寫入 DLQ。

#### **美化完成後：推播 SQS 訊息格式（Publish Lambda 輸入）**

Photo Processing Lambda 完成美化處理後，將 `NormalizedPublishTask` JSON 寫入 `publish-queue`（`sqs://{eventId}-publish-queue`），由推播引擎 Lambda 消費。訊息格式見 §2.8 `NormalizedPublishTask` 介面定義（line 221）。

### **3.3 自動社群推播：跨平台的動態適配與流量控制機制**

當跑者的專屬美化照片生成後，系統必須將其即時推送至跑者授權的社群平台。本系統透過 2.8 節所定義之「社群平台整合抽象層」處理所有平台差異，核心推播引擎完全不感知平台專屬邏輯。

各平台之 API 規範、OAuth 流程、媒體格式限制、Rate Limit 與推播方式，請參見 §2.8「社群平台整合抽象層」之 Adapter 實作矩陣與 ISocialPlatformAdapter 介面定義。

Token 生命週期管理與刷新失敗處理，請參見 §3.6。

**§3.3 IG 限動模板 vs §3.7 Cluster Dedup 執行順序（v1.5 補充）：**

兩者看似目標衝突（IG 模板要「多樣化」，Cluster dedup 要「單一最佳」），實際上是**不同階段的不同決策**，由以下順序串接：

1. **§3.7 Cluster Dedup 先執行**：對同一跑者同一攝影點的多張候選照片，依 `PHOTO_BURST_DEDUP` + `AESTHETIC_SCORING_ENABLED` Feature Flag 選出 1 張入推播佇列；其餘照片僅入 Gallery。
2. **§3.3 IG 限動模板後執行**：被選出的那 1 張照片，於推播時依跑者報名時選定的 2-3 個偏好模板，自動生成對應的 2-3 個 ready-to-share 限動版本推播至 IG。

> **關鍵差異：** §3.7 是「從多張候選中選 1 張」，§3.3 是「對選中的 1 張做 N 種格式變化」。N ≤ 12（INSTAGRAM_STORY_TEMPLATE 啟用時），預設 N = 1（未啟用）。

**實作注意事項：** 跑者於註冊頁選定的「2-3 個偏好模板」為**主推播模板**；INSTAGRAM_STORY_TEMPLATE 啟用後，系統會額外生成其他模板作為 Gallery 預覽（非主動推播），供跑者手動分享。

#### **IG 限動自動模板包（Feature Flag: INSTAGRAM_STORY_TEMPLATE）**

當 `INSTAGRAM_STORY_TEMPLATE` 啟用時，系統於推播前額外產生 Ready-to-Share 限動圖片。跑者於報名時從 12 種女性向設計模板中選擇 2–3 個偏好（櫻花漸層、馬卡龍、雜誌封面風、晨曦暖色、賽道速度感等）。系統依跑者選擇自動套用對應模板，生成 9:16 垂直格式圖片，自動裁切為 Instagram 限動尺寸（1080×1920），並疊加配速、完賽時間、賽事 Logo 與跑者姓名。同一張原始照片針對不同平台輸出最佳構圖版本（ Threads 4:5、 LINE 16:9 等），由推播引擎依平台自動選擇適合的預生成模板圖。

#### **姊妹群組推播（Feature Flag: SISTER_GROUP_NOTIFY）**

當 `SISTER_GROUP_NOTIFY` 啟用時，跑者於報名時可填寫 1–3 位「姊妹」的 LINE User ID 或 Instagram 帳號。推播時，系統除向跑者本人推播外，同步向其姊妹帳號發送通知訊息（LINE Push Message / IG DM），內容為：「[姊妹姓名] 剛完賽了！她的完賽照在這裡：[連結]」。此功能為社群病毒傳播的觸發器，由 Feature Flag 控制且**預設關閉**，须待 PDPA 第三人通知合規流程確認後方可啟用。

### **3.4 完賽個人專屬報紙（客製化）：無伺服器架構下的動態 PDF 生成**

賽事結束後，系統會彙整每位跑者的多維度資料，產生一份極具紀念價值且可供列印的客製化「個人專屬報紙」。這份數位報紙的資料來源涵蓋廣泛：包含自計時系統 API 傳回的 JSON 數據（包含各檢查點的 timingId、chipId、timestamp 與最終完賽晶片時間）4；系統在賽事中捕捉到並辨識成功的精選衝線照片；以及跑者在社群平台上互動所產生的數據反饋（如點讚數、特定活動 Hashtag 的擴散程度等）。  
在輸出格式與呈現邏輯上，此專屬報紙被定義為一份高解析度、相容於 A4 實體列印的 PDF 文件。其版面設計模擬傳統頭版新聞，主視覺為跑者的高畫質大圖，並搭配動態生成的標題（例如：「破風前行！\[跑者姓名\] 以 \[完賽時間\] 征服 \[賽事名稱\]」）。版面側邊以視覺化的圖表（如折線圖）呈現各分段區間的配速曲線，底部區塊則拼貼社群互動亮點與贊助商致敬欄位。  
為實現此高度動態且排版複雜的文件生成，系統採用無伺服器環境中的 Puppeteer-core 搭配無頭瀏覽器（Headless Chromium）解決方案，捨棄了在排版彈性上較差的 PDFKit20。系統後端首先將匯集的 JSON 數據注入預先設計好的 HTML5/CSS3 樣式模板中（使用如 Pug 或 Jinja2 模板引擎）39。隨後，AWS Lambda 函數會啟動專為 Serverless 環境壓縮的輕量化 Chromium 執行檔（如 @sparticuz/chromium-min 或利用 AWS Lambda Layer 載入的 chrome-aws-lambda）20。  
在記憶體中渲染該 HTML 頁面時，Chromium 會確保所有特殊字體（如粗體報紙標題字型）、複雜的 CSS 網格排版與外部高解析度圖片皆完美載入。為達到最佳列印效果，開發團隊將在樣式表中宣告 @media print { @page { size: A4 portrait; margin: 0; } }，確保產出的 PDF 不留白邊41。生成的 PDF 檔案將匯出為 Buffer 並自動寫入 Amazon S3 存放，隨後透過 LINE 訊息（主要 delivery 方式）將預先簽名的下載連結遞送給跑者，完成賽事體驗的最後一哩路。若主辦單位已啟用 email 通知（Feature Flag `NEWSPAPER_EMAIL`），則同時以 SMTP 寄送下載連結至跑者 email；email 基礎設施（Sender Policy、MTA、SES）須由主辦單位另行提供20。

#### **Email 寄送設定（SPF / DKIM / DMARC）**

為避免完賽報紙下載連結之 email 落入垃圾郵件夾,賽事主辦單位須於其網域 DNS 設定以下記錄後,系統方能透過 Amazon SES 大量寄送：

| 記錄類型 | 主機 / 名稱 | 值範例 | 說明 |
| :---- | :---- | :---- | :---- |
| **SPF（TXT）** | `marathon.example.com` | `"v=spf1 include:amazonses.com -all"` | 授權 SES 為合法寄件者;`-all` 表示硬性拒絕未授權寄件 |
| **DKIM（CNAME）** | `<ses-token>._domainkey.marathon.example.com` | `<ses-token>.dkim.amazonses.com` | SES 自動產生 3 組 CNAME;於 SES Console 取得 |
| **DMARC（TXT）** | `_dmarc.marathon.example.com` | `"v=DMARC1; p=quarantine; rua=mailto:dmarc@marathon.example.com; pct=100"` | 監控報表收件人;`p=quarantine` 表示未通過 DMARC 之郵件進入垃圾夾 |

**系統端 SES 設定：**
- **Verified Identity** — 賽事主辦單位須於 SES Console 驗證其網域（Domain Identity,非 Email Address Identity）
- **Production Access** — SES 預設為 Sandbox Mode（每日 200 封、僅可寄至已驗證地址）;主辦單位須提交「Production Access Request」,由 AWS Support 審查後解除上限
- **Bounce / Complaint 處理** — 啟用 SES Configuration Set 之 Event Destination (SNS Topic),系統監聽 `Bounce` / `Complaint` 事件並自動將該 email 從 DynamoDB `Runner` 表標記為 `emailInvalid=true`,未來停止寄送
- **Suppression List** — 自動加入 SES 全域 Suppression List,避免重複寄送至已知退件地址

**系統提供檢查 API：** `GET /admin/v1/events/{eventId}/email-health-check` 回傳 SPF / DKIM / DMARC 三項 DNS 記錄之驗證狀態,賽事前 7 天自動排程呼叫並將結果寄給 Super Admin。

### **3.5 系統 API 設計**

本系統之 API 分為「外部公開 API」、「內部系統 API」與「管理 API」三類。外部公開 API 以 Amazon Cognito User Pool 簽發之 JWT Bearer Token 進行身分驗證，經 API Gateway 啟用 Rate Limiting；內部系統 API 以 API Key + IP 白名單進行認證；管理 API 以 Cognito User Pool + RBAC 角色驗證。

#### **Rate Limit 雙層設計（API Gateway + Lambda）**

為避免單一維度限流誤判或粒度不足,本系統採雙層限流機制：

| 層級 | 機制 | 粒度 | 用途 |
| :---- | :---- | :---- | :---- |
| **粗粒度（API Gateway）** | Usage Plan + API Key（Per IP / Per User） | 預設「每 IP 60 req/min」、「每跑者 30 req/min」 | 防止 DDoS 與暴力枚舉;由 API Gateway 原生支援,零 Lambda 開銷 |
| **細粒度（Lambda 內 Redis Token Bucket）** | §2.8 Per-Runner Token Bucket | 「每跑者每平台 10 req/min」 | 防止單一跑者內部濫用（推播 spam）;細粒度需業務邏輯判斷,API Gateway 無法處理 |

**為何需要兩層：**
- 若僅 API Gateway 限流,無法區分「跑者 A 透過同一 IP 連續呼叫」與「10 個不同跑者同時透過同一 NAT IP 呼叫」(後者為合法賽事尖峰)
- 若僅 Lambda 限流,DDoS 攻擊會直接打到 Lambda 啟動成本,單一攻擊事件可耗盡 account concurrency
- 雙層組合:API Gateway 擋下大量異常 IP,Lambda Redis Token Bucket 處理「已通過身分驗證但行為異常」之細粒度限流

**雙層優先序（v1.4 補充）：** 兩個限流層為 **AND 關係（兩者皆須通過才放行）**，而非「先 API Gateway 後 Redis」：

| 層級 | 判定時機 | 阻擋後行為 | 是否計入另一層？ |
|:----|:----|:----|:----|
| **API Gateway（粗粒度）** | 請求進入 API Gateway 時（Lambda 啟動前） | 回 HTTP 429 + `RATE_LIMIT_EXCEEDED`；Lambda 不會被觸發 | ✗（請求未進入 Lambda，Redis Token Bucket 不計數） |
| **Lambda Redis Token Bucket（細粒度）** | 請求通過 API Gateway 後，於 Lambda handler 內執行 | 推播任務延遲 30s 重入 publish-queue（見 §2.8）；下次重試若仍超限則繼續延遲 | ✓（每次成功 publish 都消耗 1 個 token） |

**重要：** API Gateway 阻擋之請求**不會**被 Redis Token Bucket 計入；反之 Lambda 內 Redis 阻擋**仍會計入** API Gateway 之 Per-User / Per-IP 配額（因請求已通過 API Gateway）。**因此同一跑者若遭 API Gateway 限流，不會同步扣減其 Redis Token Bucket 額度**——兩個桶獨立運作、各自 quota。

**設定檔位置：**
- API Gateway Usage Plan — Terraform `infrastructure/modules/api-gateway/usage-plans.tf`
- Lambda Token Bucket — `EventFeatureConfig.publishing.perRunnerRateLimit`（§2.8）

#### **Idempotency-Key 機制（寫入端點專用）**

所有 POST 寫入端點（`/register`、`/register/face-consent`、DLQ `assign` / `resolve`、`/admin/v1/events`、`/admin/v1/events/{eventId}/publish` 等）皆接受 HTTP Header `Idempotency-Key: <uuid-v4>` 供客戶端指定冪等性識別：

- **Key TTL 24 小時** — Key 與其對應 Response 快取於 DynamoDB 獨立的 `IdempotencyKey` 表（PK: `idempotencyKey`，SK: `createdAt`，TTL 屬性於 24 小時後自動清除）。
- **相同 Key 重送規則** — 第二次以後請求（含不同 body）回傳第一次的 Response（200 / 201），不重複執行副作用；若第二次 body 與首次不同則回傳 HTTP 409 `IDEMPOTENCY_KEY_CONFLICT`。
- **內部 SQS 消費者隱含 Idempotency** — Photo Processing Lambda 與 Publish Lambda 以 `taskId` / `photoId` 作為隱含 Idempotency Key，搭配 DynamoDB Conditional Write（`attribute_not_exists(taskId)`）確保 at-least-once 重複消費僅生效一次。
- **未帶 Key 之 POST** — 系統仍允許請求（不強制），但會在 Response Header 加 `Idempotency-Key: <auto-generated-uuid>` 供客戶端後續 retry 使用。

```typescript
// Idempotency Key 快取紀錄
interface IdempotencyRecord {
  idempotencyKey: string;       // UUID v4（client 提供或 server 自動產生）
  requestHash: string;          // SHA-256 of canonicalized request body
  responseStatus: number;       // 第一次回應的 HTTP status
  responseBody: unknown;        // 第一次回應的 body
  expiresAt: number;            // Unix timestamp; DynamoDB TTL 自動清除（now + 24h）
}
```

#### **外部公開 API**

| 端點 | 方法 | 說明 | 頻率限制 |
| :---- | :---- | :---- | :---- |
| `/api/v1/races/{eventId}/register` | POST | 跑者報名並提交號碼布編號與社群帳號綁定 | 每 IP 10 req/min |
| `/api/v1/races/{eventId}/register/face-consent` | POST | Face Re-ID 自願授權上傳清晰自拍照 | 每跑者 1 req/event |
| `/api/v1/races/{eventId}/runner/{bibNumber}/status` | GET | 跑者查詢自身照片處理狀態 | 每跑者 30 req/min |
| `/api/v1/races/{eventId}/gallery?bib={bibNumber}` | GET | 圖庫搜尋：依號碼布查詢照片（降級 Fallback 模式） | 每 IP 60 req/min |

**POST /api/v1/races/{eventId}/register — Request Body：**
```json
{
  "bibNumber": "A1234",
  "fullName": "王小明",
  "email": "runner@example.com",
  "lineUserId": "Uxxxxxxxxxxxx",
  "instagramUsername": "runner_ig",
  "threadsUsername": "runner_threads",
  "facebookUid": "fb_123456",
  "consent": {
    "socialPush": true,
    "faceRecognition": false,
    "privacyPolicyVersion": "2026-v1",
    "timestamp": "2026-06-26T04:00:00Z",
    "ipAddress": "203.0.113.42"
  }
}
```

**Response（成功，201）：**
```json
{ "runnerId": "uuid-xxxx", "status": "registered", "message": "報名成功，請加入 LINE 官方帳號完成驗證" }
```

#### **內部系統 API（RFID 計時系統推送）**

| 端點 | 方法 | 說明 |
| :---- | :---- | :---- |
| `/internal/v1/timing/{eventId}/checkpoint` | POST | RFID 計時系統推送晶片通過時間點 JSON 陣列 |
| `/internal/v1/races/{eventId}/results/official` | POST | 官方成績公告觸發 PDF 生成任務 |

**POST /internal/v1/timing/{eventId}/checkpoint — Request Body：**
```json
{
  "readerId": "RDR-START-01",
  "records": [
    {
      "chipId": "MYLAPS-ABCD1234",
      "timestamp": "2026-06-26T04:30:00.000Z",
      "gpsLat": 25.0330,
      "gpsLng": 121.5654,
      "checkpointCode": "START"
    }
  ]
}
```

> **Note：** 為減少 API 呼叫次數，系統接受批次陣列模式。外部計時系統 adapter 之 `normalizeCheckpoint()` 會將每筆外部紀錄轉換為 `NormalizedCheckpointRecord[]`，再由本系統寫入資料庫。

#### **DLQ 人工處理 API**

| 端點 | 方法 | 說明 |
| :---- | :---- | :---- |
| `/internal/v1/dlq/{taskId}/assign` | POST | 將 DLQ 任務指派給特定人工處理員 |
| `/internal/v1/dlq/{taskId}/resolve` | POST | 人工確認跑者身分後補充填入 bibNumber |

#### **OAuth 授權回調與跑者自助 API**

**URL 命名規範（v1.5 統一）：** 本系統所有對外 endpoint URL 一律使用 `*.{eventId}.imarathon.app` 子域名格式（與 §3.1 前端 Hosting 規格一致），**不再使用 `api.example.com` placeholder**。OAuth Callback 端點因此為 `https://api.{eventId}.imarathon.app/api/v1/oauth/callback/{platform}`，其中 `{eventId}` 與 `register.{eventId}.imarathon.app` 等前端子域名共享同一賽事命名空間。

| 端點 | 方法 | 說明 | 頻率限制 |
| :---- | :---- | :---- | :---- |
| `/api/v1/oauth/callback/{platform}` | GET | OAuth 授權回調端點，接收 authorization code 並換取 Token | N/A（由平台觸發） |

**GET /api/v1/oauth/callback/{platform} — OAuth 2.0 Callback 流程：**

所有 OAuth 授權流程（含 LINE Login、Meta Graph API、Threads）均使用 `state` 參數防止 CSRF 攻擊，並攜帶 runner 識別資訊。Callback URL 格式如下：

```
https://api.{eventId}.imarathon.app/api/v1/oauth/callback/{platform}?code={authorization_code}&state={base64url_json}
```

其中 `state` 參數為 Base64URL 編碼的 JSON，格式如下：

```json
{
  "runnerId": "uuid-xxxx",
  "eventId": "event-2026-taipei-marathon",
  "platform": "instagram",
  "nonce": "random-csrf-token"
}
```

Callback Lambda 收到 `code` 與 `state` 後，解碼並解密 `state`，驗證 `nonce` 後依序執行：
1. 拿 `code` 向平台換取 `access_token` 與 `refresh_token`
2. 將 Token 以 AES-256-GCM（KMS CMK）加密後寫入 DynamoDB，狀態設為 `active`
3. 更新 `runner-platform-index`，建立 `{ runnerId, platform }` 複合鍵
4. 重新導向至註冊完成頁面（前端）
| `/api/v1/races/{eventId}/runner/{bibNumber}/revoke` | DELETE | 跑者撤銷社群推播授權（PDPA 刪除權），系統銷毀對應 Token 並停止推播 | 每跑者 5 req/hour |
| `/api/v1/races/{eventId}/runner/{bibNumber}/photos` | GET | 取得跑者所有已處理照片列表（含 Pre-signed Download URL） | 每跑者 30 req/min |
| `/api/v1/races/{eventId}/runner/{bibNumber}/newspaper` | GET | 取得完賽報紙 PDF 下載連結（Pre-signed URL，有效期 24 小時） | 每跑者 10 req/min |

#### **統一 Error Response 規範**

所有 4xx / 5xx 回應統一採用以下 JSON Schema，前端 / SDK 可依此格式解析錯誤：

```typescript
interface ApiErrorResponse {
  error: {
    code: string;             // 機器可讀錯誤代碼,UPPER_SNAKE_CASE,例 "RUNNER_NOT_FOUND"
    message: string;          // 人類可讀訊息,繁體中文（國際賽事可 i18n 切換）
    details?: unknown;        // 補充資訊（例: 欄位驗證錯誤陣列、retry-after 秒數）
    traceId: string;          // AWS X-Ray Trace ID,供 debug 與客服追蹤
    timestamp: string;        // ISO 8601 UTC,伺服器時間
  };
}
```

**HTTP 狀態碼對應規範：**

| HTTP Status | 語意 | 典型 `code` 範例 |
| :---- | :---- | :---- |
| 400 Bad Request | 請求格式錯誤（JSON parse 失敗、欄位缺失） | `INVALID_REQUEST_BODY`、`MISSING_REQUIRED_FIELD` |
| 401 Unauthorized | 未帶 JWT 或 JWT 過期 | `UNAUTHENTICATED`、`TOKEN_EXPIRED` |
| 403 Forbidden | 權限不足（RBAC 拒絕、Cognito Group 不符） | `FORBIDDEN`、`INSUFFICIENT_ROLE` |
| 404 Not Found | 資源不存在 | `RUNNER_NOT_FOUND`、`EVENT_NOT_FOUND`、`PHOTO_NOT_FOUND` |
| 409 Conflict | 狀態衝突（Idempotency Key 重複但 body 不同、報名已存在） | `IDEMPOTENCY_KEY_CONFLICT`、`RUNNER_ALREADY_REGISTERED` |
| 422 Unprocessable Entity | 語意錯誤（格式正確但業務邏輯拒絕） | `BIB_NUMBER_INVALID`、`OAUTH_CODE_EXPIRED` |
| 429 Too Many Requests | 超過 Rate Limit | `RATE_LIMIT_EXCEEDED`（`details.retryAfterSeconds` 提供退避時間） |
| 500 Internal Server Error | 伺服器內部錯誤（未預期例外） | `INTERNAL_ERROR`（前端不應重試,僅顯示錯誤頁） |
| 503 Service Unavailable | 服務暫時不可用（DB / S3 連線失敗） | `SERVICE_UNAVAILABLE`（前端可指數退避重試） |

#### **Webhook 接收端點**

| 端點 | 方法 | 說明 |
| :---- | :---- | :---- |
| `/webhook/v1/line` | POST | LINE Webhook 接收端點，處理好友新增（follow）事件以綁定 User ID |
| `/webhook/v1/meta` | POST | Meta Webhook 接收端點，處理 Instagram/Facebook 授權變更通知 |

**POST /webhook/v1/line — LINE Platform Webhook Payload：**
```json
{
  "events": [
    {
      "type": "follow",
      "replyToken": "nHqRhkJP0H...",
      "source": {
        "userId": "Uxxxxxxxxxxxx",
        "type": "user"
      },
      "timestamp": 1749293400000
    }
  ]
}
```

**LINE Webhook Runner 識別流程：**

當系統收到 `follow` 事件時，LINE User ID（`source.userId`）已知，但系統尚不知道這個 LINE User ID 對應哪一位跑者。綁定邏輯如下：

1. **正向查詢**：系統以 `lineUserId` 為索引（GSI: `line-user-index`，PK: `lineUserId`）查詢 DynamoDB `Runner` 表，取得對應的 `runnerId`。
2. **已報名跑者**：若 `runnerId` 存在且該跑者已於本賽事報名（`eventId` 存在於其報名記錄中），則完成 `lineUserId → runnerId` 綁定寫入，後續可直接以 LINE User ID 查詢 runner。
3. **未報名跑者**（`runnerId` 尚不存在）：系統暫存 `lineUserId` 至 `PendingLineBinding` 表（PK: `lineUserId`, SK: `eventId`），等待跑者完成報名後於 `POST /api/v1/races/{eventId}/register` 中填入 `lineUserId`，再合併綁定。
4. **OAuth 授權綁定**（LINE Login）：若跑者透過 LINE Login 進行 OAuth 授權，callback URL 中的 `state` 參數（Base64URL encoded JSON：`{ runnerId, eventId, platform: "line" }`）於授權成功後傳回系統，直接完成三元組 `{ runnerId, eventId, platform }` 的綁定，邏輯更為簡潔且安全性更高。

#### **Webhook 簽章驗證（HMAC-SHA256）**

所有外部 Webhook（LINE 與 Meta）於 Lambda 處理事件前必須驗證 HMAC 簽章，杜絕偽造請求與重放攻擊。Webhook Lambda 內部介面：

```typescript
interface IWebhookSignatureVerifier {
  readonly platform: 'line' | 'meta';
  /**
   * 驗證 Webhook 請求之 HMAC 簽章。回 false 時 Lambda 必須立即回 401 並終止處理。
   * @param rawBody 請求原始 Body(必須為字串,不可預先 JSON.parse,否則字節序會破壞 HMAC)
   * @param signatureHeader HTTP Header 之簽章(LINE: X-Line-Signature; Meta: X-Hub-Signature-256)
   * @param timestampHeader HTTP Header 之時間戳(LINE: 無;Meta: 無;事件 body 內含 timestamp)
   */
  verify(rawBody: string, signatureHeader: string, timestampHeader?: string): boolean;
}
```

**LINE Messaging API 簽章驗證流程：**

1. Channel Secret 存於 AWS Secrets Manager,路徑 `secrets/{env}/events/{eventId}/line-channel-secret`（Rotation: 90 天）。
2. 收到 POST `/webhook/v1/line` 後,從 Header 取 `X-Line-Signature` 值。
3. 以 Node.js `crypto.createHmac('sha256', channelSecret).update(rawBody).digest('base64')` 計算預期簽章。
4. 以 `crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signatureHeader))` **常數時間比對**（防止 timing attack）。
5. 簽章不符 → Lambda 回 HTTP 401 + `ApiErrorResponse{ code: "INVALID_SIGNATURE" }`,不進入業務邏輯。
6. 額外 replay 防護：LINE Event body 內含 `timestamp`（毫秒）,若與伺服器時間差 > 5 分鐘視為重放攻擊丟棄。

**Meta Webhook（Instagram / Facebook）簽章驗證流程：**

1. App Secret 存於 Secrets Manager,路徑 `secrets/{env}/events/{eventId}/meta-app-secret`。
2. 收到 POST `/webhook/v1/meta` 後,從 Header 取 `X-Hub-Signature-256` 值（格式：`sha256=<hex>`）。
3. 移除 `sha256=` prefix,以 `crypto.createHmac('sha256', appSecret).update(rawBody).digest('hex')` 計算預期 hex digest。
4. 同樣以 `timingSafeEqual` 常數時間比對。
5. Meta Webhook 之 GET `/webhook/v1/meta`（Verification Challenge）依 Meta 規範回傳 `hub.challenge` 查詢參數,無需 HMAC 驗證但須驗證 `hub.verify_token` 環境變數匹配。

**Lambda 程式碼範例（LINE）：**

```typescript
import crypto from 'crypto';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

export const handler = async (event: { body: string; headers: Record<string, string> }) => {
  const channelSecret = await getSecret(`events/${process.env.EVENT_ID}/line-channel-secret`);
  const expected = crypto.createHmac('sha256', channelSecret).update(event.body).digest('base64');
  const received = event.headers['x-line-signature'] ?? '';
  if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received))) {
    return { statusCode: 401, body: JSON.stringify({ error: { code: 'INVALID_SIGNATURE', traceId: '', timestamp: new Date().toISOString(), message: '簽章驗證失敗' } }) };
  }
  // ... 後續事件處理 ...
};
```

#### **Gallery API Response 格式**

**GET /api/v1/races/{eventId}/gallery?bib={bibNumber} — Response（成功，200）：**
```json
{
  "data": [
    {
      "photoId": "uuid-xxxx",
      "checkpointCode": "KM25",
      "capturedAt": "2026-06-26T05:30:00Z",
      "thumbnailUrl": "https://cdn.example.com/thumb/uuid-xxxx.jpg",
      "downloadUrl": "https://s3.amazonaws.com/...(pre-signed)...",
      "publishedPlatforms": ["line", "instagram"],
      "ocrConfidence": 0.92
    }
  ],
  "nextCursor": "eyJldmVudElkIjoiZXZlbnQtMjAyNi10YWlwZWktbWFyYXRob24iLCJwaG90b0lkIjoiNjY0Zi4uLiJ9",
  "hasMore": true
}
```

#### **分頁規範（Cursor-based Pagination）**

所有 List 端點（`/gallery`、`/admin/v1/events`、`/admin/v1/events/{eventId}/dlq`、`/admin/v1/events/{eventId}/photos` 等）統一採用 **Cursor-based pagination**，不使用 offset-based pagination：

- **Request 參數** — `?cursor=<opaque-base64-token>&limit=20`（`limit` 預設 20，上限 100）
- **Response 結構** — `{ data: T[], nextCursor: string | null, hasMore: boolean }`
- **後端實作** — DynamoDB Query 結果之 `LastEvaluatedKey` 經 HMAC-SHA256 簽章後 base64 編碼為 opaque cursor；下一頁請求帶回，後端解密後續查
- **為何不用 offset** — DynamoDB 沒有原生 skip，offset-based 在大表（單賽事 60,000 張照片）會觸發大量廢棄掃描，成本過高且 P99 延遲不可預測
- **Gallery API 強制分頁** — 預期單賽事 Gallery 累積 60K 張照片；未分頁 Response 將觸發 Lambda 6 MB payload 上限錯誤或 CloudFront 緩存失效

```typescript
interface PaginatedResponse<T> {
  data: T[];
  nextCursor: string | null;   // opaque base64-encoded HMAC-signed token; null 表示最後一頁
  hasMore: boolean;            // nextCursor !== null 的冗餘旗標,供前端快速判斷
}
```

#### **OpenAPI / Swagger 文件規範**

所有 API 端點須以 **OpenAPI 3.1** 規格產出可互動文件,作為前端工程師、QA、第三方整合商之 Single Source of Truth。

**自動產出流程：**
1. TypeScript 型別定義（§2.12 + §3.5 各介面）作為 Single Source of Truth
2. 使用 `zod-to-openapi` 或 `@asteasolutions/zod-to-openapi` 從 zod schema 自動產生 OpenAPI 3.1 YAML
3. PR 合併時由 GitHub Actions 自動產生並部署至 `https://api.{eventId}.imarathon.app/api/docs`
4. CI 驗證：若 PR 修改 `src/**/*.ts` 但未更新對應 OpenAPI schema,CI 失敗（防止 spec 漂移）

**存取控制：**
- 公開文件 — 僅顯示 GET 端點列表與摘要（無 request/response body）
- 完整 schema — 需 Super Admin 登入後檢視（防範 API 設計洩漏）

**SDK 自動產生：**
- TypeScript SDK — `openapi-typescript-codegen` → 發佈至內部 NPM Registry
- Python SDK — `openapi-python-client` → 發佈至內部 PyPI
- 第三方整合商可下載 SDK 並直接使用,降低整合摩擦

**版本管理：**
- OpenAPI 文件 URL 含 major 版本：`/api/v1/docs`、`/api/v2/docs`
- 舊版於 deprecation 後保留 6 個月
- Breaking change 須於 Release Notes 公告至少 30 天後實施

### **3.6 OAuth Token 生命週期與刷新失敗處理**

OAuth Token（Access Token + Refresh Token）之生命週期管理為系統穩定性的關鍵環節。本節定義 Token 完整生命週期、DynamoDB 儲存結構、加密 envelope、與平台撤銷流程的整合。

#### **Token 生命週期狀態機**

每張 OAuth Token 在 DynamoDB 中以下列狀態存在：

```
[active] ──expires_in < 10min──> [expiring_soon]
   │                                 │
   │                                 ├──refresh_success──> [active] (新 expiresAt)
   │                                 ├──refresh_401/403──> [revoked]
   │                                 ├──refresh_410/expired──> [needs_reauth]
   │                                 └──refresh_5xx (3 retries fail)──> [refresh_failed] (寫入 DLQ)
   │
   ├──user_revoke (PDPA DELETE /revoke)──> [revoked]
   ├──admin_force_revoke (§3.8 Super Admin)──> [revoked]
   ├──eventEndAt + 30d──> [purged] (S3 + DynamoDB 雙刪除;依 §4.3)
   └──user_unbind (OAuth callback 解綁)──> [revoked]
```

| 狀態 | 意義 | 推播可用？ | 加密儲存？ |
|:----|:----|:----:|:----:|
| `active` | Token 有效，距過期 > 10 分鐘 | ✓ | ✓ |
| `expiring_soon` | 距過期 ≤ 10 分鐘，下次推播前需 refresh | ✓ (refresh 後) | ✓ |
| `revoked` | 用戶撤銷或平台 401/403 拒絕 | ✗ | ✓ (audit 留 30 天) |
| `needs_reauth` | Refresh Token 過期，需用戶重新 OAuth | ✗ | ✗ (已清除) |
| `refresh_failed` | 5xx 重試耗盡，進入 DLQ 人工處理 | ✗ | ✓ |
| `purged` | 賽事結束 30 天後或 PDPA 撤回，實體刪除 | ✗ | ✗ |

#### **DynamoDB OAuthToken Schema**

```typescript
// 主表:OAuthToken (PK: runnerId+platform, SK: eventId)
interface OAuthTokenRecord {
  // 複合主鍵
  runnerId: string;                        // PK part 1 — 跑者唯一 ID (UUID v4)
  eventId: string;                         // PK part 2 — 賽事 ID (對應 §2.6 多租戶隔離)
  platform: 'line' | 'instagram' | 'threads' | 'facebook' | 'x'; // 平台識別

  // Token 本體(皆以 AES-256-GCM + KMS CMK 加密,僅 Lambda 解密後使用)
  accessTokenEncrypted: Buffer;            // KMS-encrypted access token
  accessTokenNonce: Buffer;                // GCM nonce (12 bytes)
  refreshTokenEncrypted?: Buffer;          // KMS-encrypted refresh token (可選,某些平台無)
  refreshTokenNonce?: Buffer;

  // 生命週期狀態
  status: 'active' | 'expiring_soon' | 'revoked' | 'needs_reauth' | 'refresh_failed' | 'purged';
  expiresAt: string;                       // ISO 8601 UTC
  issuedAt: string;                        // ISO 8601 UTC
  lastRefreshedAt?: string;

  // OAuth 平台專屬資訊
  scope: string[];                         // 授權範圍,例 ['instagram_basic', 'instagram_content_publish']
  platformUserId: string;                  // 平台 User ID (LINE User ID, IG Business Account ID 等)
  linkedPageId?: string;                   // 僅 Instagram — Facebook 粉絲專頁 ID

  // 加密中繼資料
  kmsKeyId: string;                        // ARN of KMS CMK used
  kmsKeyVersion: number;                   // KMS key version (for rotation tracking)
  encryptionAlgorithm: 'AES-256-GCM';      // 明文標示,供解密路徑判斷

  // PDPA / 稽核
  consentId: string;                       // FK → Runner.consent.crossBorderTransferConsent (見 §3.1.X)
  crossBorderTransferGranted: boolean;     // 是否取得跨境同意
  createdAt: string;
  updatedAt: string;
  revokedAt?: string;                      // 撤銷時間(若有)
  revokedReason?: 'user_revoke' | 'platform_401' | 'admin_force' | 'refresh_expired' | 'event_end_purge';

  // TTL (DynamoDB 自動清除,僅為輔助;真正的清除走 §4.3 PDPA 流程)
  ttl: number;                             // Unix timestamp, 預設 status='purged' 後 90 天
}

// GSI 索引(支援 §3.5 §3.5.2 LINE Webhook 流程與 §3.8 後台管理)
interface OAuthTokenGSI {
  // GSI1: 依平台 User ID 查詢(用於 LINE follow / Meta callback)
  'platform-user-index': {
    PK: `platform#${platform}#${platformUserId}`; // e.g. "platform#line#Uxxxxxxxxxxxx"
    SK: `eventId`;
  };

  // GSI2: 依狀態 + 過期時間查詢(用於背景 refresh job 批次掃描 expiring_soon)
  'status-expiry-index': {
    PK: `status#${status}`;                // e.g. "status#expiring_soon"
    SK: `expiresAt`;                       // 排序用
  };

  // GSI3: 依賽事 + 狀態查詢(用於 §3.8 後台 DLQ 顯示、§4.3 跨境副本刪除)
  'event-status-index': {
    PK: `eventId`;
    SK: `status#updatedAt`;
  };
}
```

#### **Token 加密 Envelope 流程**

```
[1] OAuth Callback Lambda 收到 authorization code
    ↓
[2] 拿 code 換 access_token + refresh_token (向平台 API)
    ↓
[3] AWS KMS GenerateDataKey(CMK) → 取得 plaintext DEK + encrypted DEK
    ↓
[4] AES-256-GCM(plaintext DEK, access_token) → accessTokenEncrypted + nonce
    ↓
[5] 銷毀 plaintext DEK (memory only); 僅保留 encrypted DEK
    ↓
[6] 寫入 DynamoDB OAuthToken(accessTokenEncrypted, accessTokenNonce, kmsKeyId, kmsKeyVersion)
    ↓
[7] Lambda process exit; plaintext access_token 從記憶體清除
```

**KMS CMK Rotation 政策：**
- 每場賽事建立獨立 CMK（Customer Master Key）
- CMK 啟用 90 天後自動排程刪除（見 §2.5 L170）
- Rotation 時 `kmsKeyVersion` 自動遞增；舊 Token 仍可用舊 version 解密，新 Token 強制使用新 version
- 詳見 §4.3 PDPA Token 銷毀政策

#### **Token 刷新邏輯**

每張 OAuth Token 均攜帶 `expiresAt` 時間戳。Lambda 函數在執行推播前，會先檢查是否已過期或距離過期不足 10 分鐘；若符合條件則先呼叫平台 Refresh Endpoint 取得新 Token，再執行推播。刷新成功後新 Token 加密後寫回 DynamoDB，並更新 `expiresAt` 與 `kmsKeyVersion`。

**背景刷新 Job：**
- 觸發：EventBridge Schedule 每 5 分鐘掃描 `status-expiry-index` 中 `status='expiring_soon'` 的 Token
- 預期 batch size：~50-200 個 / 5 min（單賽事 12,000 啟用跑者 × 3 平台 ÷ 90 天 × 5 min）
- Concurrency：透過 SQS `token-refresh-queue` 觸發 Token Refresh Lambda（Memory 512MB, Timeout 30s）
- Idempotency：以 `runnerId + platform + eventId` 為隱含 Idempotency Key，Conditional Write 防止 race condition

#### **刷新失敗時之降級流程**

| 失敗原因 | HTTP Code | 系統行為 | 後續狀態 |
| :---- | :---- | :---- | :---- |
| 用戶主動撤銷授權 | 401 Unauthorized | 立即更新 Token 狀態為 `revoked`，**不重試**，寫入事件日誌；如跑者有備援平台（LINE），自動切換至備援推播；無備援則標記 `needs_reauth` 並於完賽報紙中通知 | `revoked` |
| Refresh Token 過期（一般為 30–60 天） | 400 invalid_grant | 視同撤銷處理；觸發重新授權通知（LINE Push Message） | `needs_reauth` |
| 平台 API 暫時性錯誤 | 5xx | 指數退避重試（最多 3 次，間隔 30s/60s/120s），3 次失敗後寫入 DLQ「Token 刷新重試失敗」 | `refresh_failed` (DLQ) |
| 網路瞬斷 | timeout / ECONNRESET | 重試 1 次，失敗即寫入 DLQ | `refresh_failed` (DLQ) |

#### **重新授權通知**

當 Token 狀態變更為 `needs_reauth` 時，系統透過 LINE Push Message 主動通知跑者：「您的社群授權已過期，請於 48 小時內重新授權以確保收到完賽報紙」。48 小時內未重新授權 → 完賽報紙中以「授權過期」區塊說明，照片仍寫入 Gallery 供事後手動下載（符合 §4.3 資料最小化原則）。

#### **Token 撤銷對應之 PDPA 流程**

當用戶呼叫 `DELETE /api/v1/races/{eventId}/runner/{bibNumber}/revoke`（§3.5）或後台 Super Admin 強制撤銷時：
1. DynamoDB Transactional Write：`OAuthToken.status` → `revoked`、`revokedAt` = now、`revokedReason` = `user_revoke` 或 `admin_force`
2. S3 Cross-Region Replication 刪除標記（針對海外節點副本，§4.3 PDPA §21）
3. DynamoDB Global Tables 條件刪除（僅刪除 status='revoked' 之後的更新版本，保留 30 天 audit log）
4. CloudTrail 寫入撤銷事件（保留 7 年，§4.3 稽核軌跡）
5. 推播引擎下次嘗試時偵測 status='revoked' → 跳過該平台，推播其他備援平台或 fallback 至 Gallery

#### **跨章節關聯**

| 本節定義 | 引用之章節 |
|:----|:----|
| OAuthTokenRecord schema | §2.5 DynamoDB 表設計（已預留 `runner-platform-index` GSI） |
| KMS CMK 加密 envelope | §4.3 PDPA §21 跨境傳輸合規 |
| Token 狀態機 | §3.1.X 跨國跑者同意、§3.8 Super Admin 強制撤銷 |
| 重新授權通知 | §3.1.Y 推播撤回機制（撤回後狀態變更需要通知） |
| 48hr 重新授權窗口 | §4.3 完賽報紙 PDF 30 天保留政策 |

### **3.7 照片歸戶邏輯（Corner Cases）**

**同一攝影點之連拍叢集（Photo Burst）處理：**當多位跑者在極短時間內相繼通過同一攝影點時，單一 Lambda 實例可能對同一號碼布產生多張照片。系統以「**號碼布 + 拍攝時間窗口**」作為叢集鍵（Cluster Key）。時間窗口預設為「**拍攝時間 ±3 秒**」，可由 `EventFeatureConfig.rendering.burstClusterWindowMs`（預設 3000ms，範圍 1000–10000ms）依賽事特性動態調整：

- **預設 3000ms（±3 秒）** — 適用於一般晴朗賽事
- **雨天 / 夜間調整為 5000ms（±5 秒）** — 跑者通過速度變慢、攝影師快門間隔拉長
- **衝線場景 1000ms（±1 秒）** — 起終點瞬間集團通過,縮短以避免不同跑者被誤合併
- **拍立得場景 8000–10000ms（±8–10 秒）** — 拍立得輸出慢,需拉長等待

叢集鍵組成：`{eventId}#{bibNumber}#{stationId}#{floor(capturedAt / burstClusterWindowMs)}`，由 Photo Processing Lambda 計算後內存比對（無需 DynamoDB round-trip）。

**叢集鍵分隔符安全性說明（v1.5 補充）：** 叢集鍵使用 `#` 作為分隔符，需注意以下 DynamoDB 約束：
- DynamoDB Sort Key 長度上限 **2048 bytes**；本叢集鍵最壞情況長度估算：`eventId(64) + bibNumber(16) + stationId(32) + timestampFloor(13) + 3個'#'` = **125 bytes**，遠低於上限。
- `#` 在 DynamoDB Sort Key 中為合法字元（ASCII 0x23），無特殊語意（不像 DynamoDB 預留的 `#` 在某些 query 操作中）。
- **衝突風險：** 若 `bibNumber` 或 `stationId` 本身可包含 `#` 字元（如某些賽事允許客製化號碼布含 `#`），會導致叢集鍵解析錯誤。**規範：** `bibNumber` 與 `stationId` 應限制為 `[A-Za-z0-9-]` 字元集；若無法保證，叢集鍵應改用 length-prefixed 編碼（如 `JSON.stringify(parts).length#JSON.stringify(parts)`）或 Base64URL 編碼後再 hash。

**單一叢集去重複合鍵：** `photoId + bibNumber` 用於防止同一照片重複推播給同一跑者（§3.7 集團通過場景），採 `#` 分隔；與叢集鍵共用相同字元限制。

叢集內選圖策略（由 Feature Flag 控制）：
- **`PHOTO_BURST_DEDUP` 關閉：** 所有照片均執行推播（不退而求其次）。
- **`PHOTO_BURST_DEDUP` 開啟 + `AESTHETIC_SCORING_ENABLED` 關閉：** 選擇「OCR 置信度最高」之一張執行推播，其餘寫入圖庫。
- **`PHOTO_BURST_DEDUP` 開啟 + `AESTHETIC_SCORING_ENABLED` 開啟：** 執行加權評分決策，選擇總分最高之一張：

```
score_final = α × ocr_confidence + β × aesthetic_overall_score
其中 α + β = 1，α, β 由主辦單位透過 Feature Flag 參數調整（建議 α=0.4, β=0.6）
美學分數維度：銳利度(sharpness)、構圖平衡(composition)、光線充足(lighting)、表情清晰(faceClarity)
```

此加權設計旨在「OCR 可識別」與「照片好看」之間取得平衡；對於女性跑者群體，β 權重通常較高。

**同一照片含多位跑者（集團通過）：**系統對照片內所有偵測到之候選邊界框（所有 Confidence ≥ 0.5 之 Bib Bounding Box）逐一執行 OCR 與推播；同一張照片可同時歸戶至多位跑者。為避免重複推播，系統以 `photoId + bibNumber` 複合鍵做去重檢查，已成功推播之組合不再重複發布。此去重邏輯由 Feature Flag `GROUP_PHOTO_DEDUP` 控制（預設開啟）。

**DLQ 照片之人工處理 SLA：**進入 DLQ 之照片須於賽事結束後 48 小時內完成人工確認，否則於完賽報紙中以「遺珠之憾」區塊說明該照片存在但未能及時確認。

### **3.8 後台管理系統：賽事管理與權限控制**

賽事主辦單位須透過專屬後台系統完成賽事注册、系統配置與即時監控。本系統提供基於瀏覽器的回應式後台（Admin Dashboard），所有操作皆經 API Gateway 呼叫後端 Lambda，無傳統意義的常駐伺服器。

#### **帳號與權限模型**

本系統採用 **RBAC（Role-Based Access Control）** 模型，共定義四種角色：

| 角色 | 權限範圍 |
| :---- | :---- |
| **系統管理員（Super Admin）** | 創建/刪除賽事主辦單位帳號、跨所有賽事之全局設定、稽核日誌檢視、KMS 金鑰管理 |
| **賽事主辦單位（Event Organizer）** | 完整管理其所屬賽事：建立/編輯賽事、上傳參賽者名單、配置攝影點位、管理 LINE 官方帳號、檢視 DLQ、匯出完賽報告 |
| **攝影師（Photographer）** | 檢視所屬賽事之攝影點位指示、上傳照片進度儀表板、無法存取 DLQ 或修改賽事設定 |
| **人工處理員（DLQ Operator）** | 僅可檢視並處理 DLQ 任務、填入 bibNumber，無法修改其他賽事資料 |

#### **賽事建立與配置流程**

賽事主辦單位登入後，依序完成以下配置：

1. **基本資訊設定**：賽事名稱、日期時間、賽事地點（GPS 座標）、預估參賽人數、賽事類型（馬拉松/半馬/三鐵/自行車等）
2. **攝影點位配置**：於地圖介面新增/編輯攝影點位，每點位包含座標、點位名稱（如「終點線」、「折返點」）、上傳方式（S3 Direct Upload 或 FTP）、該點位之照片預設浮水印模板
3. **參賽者名單匯入**：支援 CSV 匯入（欄位：`bibNumber, fullName, email, chipId, gender, category`，共 6 欄，與 §3.8 API 規格一致），系統自動與計時系統晶片資料比對；名單確認後系統鎖定，後續跑者變更須由主辦單位授權後放行。CSV header 順序固定，缺少任一欄位或欄位順序錯誤時視為上傳失敗，並回傳錯誤明細（詳見 §3.5 `POST /admin/v1/events/{eventId}/participant-csv`）
4. **LINE 官方帳號設定**：填入 LINE 官方帳號之 Channel Access Token，系統驗證 Messaging API 連線；每位跑者完成報到後，系統自動發送 LINE 好友邀請連結
5. **贊助商與相框模板**：上傳 PNG 相框模板（Alpha Channel）、設定浮水印強度、填入贊助商名稱與 Logo
6. **PDF 完賽報紙模板選擇**：從預設版型庫中選擇，或上傳客製化 HTML/CSS 模板（須通過安全審查）
7. **OAuth 平台設定**：填入 Meta App ID/Secret、Threads App Token，系統進行 API 連線測試後啟用

#### **第三方平台帳號註冊責任聲明**

LINE 官方帳號、Meta Business Account、Threads Developer Account、SnapSeek 等第三方平台之帳號註冊、App 審查、API 權限申請，**均由賽事主辦單位負責**，本系統僅提供技術整合層。具體責任劃分如下：

| 項目 | 主辦單位責任 | 系統提供 |
| :---- | :---- | :---- |
| LINE 官方帳號註冊 | 申請 LINE Official Account、通過 Messaging API 審查、取得 Channel Access Token + Channel Secret | 後台輸入 Secret 至 Secrets Manager（§2.9） |
| Meta Business Account | 註冊 Meta Business Suite、建立 Facebook 粉絲專頁、申請 Instagram Business / Creator 帳號、申請 Meta App、通過 App Review（含 `instagram_business_content_publish` 等權限） | 後台輸入 App ID/Secret;OAuth Callback URL 自動產生（§3.5） |
| Threads Developer Account | 註冊 Threads API、產生 App Token | 後台輸入 Token |
| SnapSeek API Key | 註冊 SnapSeek 帳號、付費方案、取得 API Key | 後台輸入至 Secrets Manager |
| LINE Login OAuth | 註冊 LINE Login Channel、取得 Channel ID/Secret | OAuth Callback URL 自動配置 |

**後台「OAuth 連線測試」按鈕：** 後台設定頁提供一鍵測試按鈕，呼叫 `GET /admin/v1/oauth/test-connection?platform={platform}` 端點,該端點以最小 API 呼叫驗證 Token 有效（例：LINE 呼叫 `/v2/bot/info`,Meta 呼叫 `/{ig-user-id}?fields=id`）。回傳結果分為 `ok` / `expired` / `insufficient_scope` / `network_error` 四種,並附「如何修復」說明連結。

**App Review 注意事項：** Meta App 審查通常需 5-10 個工作天,須提交「影片示範」說明 App 如何使用 `instagram_business_content_publish` 權限。系統提供 Admin Dashboard 內建「App Review Submission Helper」,自動產生示範影片腳本與截圖。

#### **即時監控儀表板**

賽事進行期間，後台提供以下即時資訊：

| 監控面板 | 內容 |
| :---- | :---- |
| **照片處理儀表板** | 已上傳照片總數、處理中數、成功推播數、DLQ 待處理數；各 Lambda 函數錯誤率 |
| **跑者互動儀表板** | 當前完成報到之跑者人數、各平台授權比例（LINE/IG/Threads）、預計覆蓋率（拍到照之完賽跑者比例） |
| **DLQ 處理面板** | DLQ 任務清單（照片縮圖、拍攝時間、失敗原因）、一指「確認身分」快速填入 bibNumber |
| **網路監控面板** | 各 5G 攝影站之上傳頻寬、即時照片上傳速率、連線中斷報警 |

#### **後台安全性要求**

- 所有後台操作均寫入 **CloudTrail** 稽核日誌（誰、何時、哪個 IP、進行何操作）
- 敏感操作（如刪除賽事、修改 KMS 金鑰）須為 **Super Admin** 角色，並觸發 **MFA（Multi-Factor Authentication）** 驗證
- 賽事主辦單位密碼須符合 AWS Cognito 密碼政策（最少 12 字元，包含大小寫、數字、特殊符號）
- 後台 Session Token 有效期為 8 小時，賽事當日操作人員須每日重新登入
- 資料庫連線使用 IAM Authenticated API，不使用長期資料庫帳號密碼

#### **管理 API（內部使用）**

| 端點 | 方法 | 說明 |
| :---- | :---- | :---- |
| `/admin/v1/events` | POST | 建立新賽事（僅 Super Admin） |
| `/admin/v1/events` | GET | 列出所有賽事（支援分頁） |
| `/admin/v1/events/{eventId}` | GET | 取得單一賽事詳情（含狀態） |
| `/admin/v1/events/{eventId}` | PUT | 完整更新賽事設定（不含敏感資訊） |
| `/admin/v1/events/{eventId}` | DELETE | 軟刪除賽事（僅 Super Admin，保留資料） |
| `/admin/v1/events/{eventId}/photographer-station` | POST | 新增攝影點位 |
| `/admin/v1/events/{eventId}/participant-csv` | POST | 匯入參賽者 CSV |
| `/admin/v1/events/{eventId}/publish` | POST | 發布賽事（鎖定名單、啟用 LINE 推播） |
| `/admin/v1/dlq/{taskId}/assign` | POST | 指派 DLQ 任務 |
| `/admin/v1/dlq/{taskId}/resolve` | POST | 人工確認並標記 DLQ 完成 |

#### **Admin API Request / Response 格式**

**POST /admin/v1/events — Request Body：**
```json
{
  "eventName": "2026 臺北市民馬拉松",
  "eventDate": "2026-12-05",
  "eventType": "marathon",
  "maxParticipants": 20000,
  "timezone": "Asia/Taipei",
  "location": {
    "city": "臺北市",
    "startPoint": { "lat": 25.0330, "lng": 121.5654 }
  },
  "organizerId": "org-uuid-xxxx"
}
```

**Response（成功，201）：**
```json
{
  "eventId": "event-2026-taipei-marathon",
  "status": "draft",
  "createdAt": "2026-06-26T08:00:00Z",
  "message": "賽事建立成功，請前往賽事設定完成配置"
}
```

**POST /admin/v1/events/{eventId}/participant-csv — Request：**`Content-Type: multipart/form-data`，CSV 欄位：`bibNumber, fullName, email, chipId, gender, category`。**

**Response（成功，200）：**
```json
{
  "imported": 19850,
  "duplicates": 12,
  "errors": [
    { "row": 1042, "bibNumber": "A9999", "error": "chipId 格式不符" }
  ]
}
```

**POST /internal/v1/races/{eventId}/results/official — Request Body：**
```json
{
  "officialResultsUrl": "https://official-timer.example.com/results/2026-taipei-marathon.json",
  "announcedAt": "2026-12-05T14:30:00Z"
}
```

收到此請求後，系統依 `officialResultsUrl` 取得官方成績 JSON，匯入 DynamoDB，並觸發該 eventId 所有 DLQ 狀態為 `resolved` 或 `photo_matched` 之跑者的完賽報紙 PDF 生成任務（見 §3.4）。

#### **賽事方定價與 ROI 計算（§3.8.X）**

基於 deep-research 對 FinisherPix 兩種 B2B 模式（固定費率+自留銷售收入 / 收入分成+賽事佣金）的觀察,賽事方對「照片服務」有明確成本結構期望。本節定義三層 Freemium 定價與賽事方 ROI 計算面板,以突破 AllSports 銷售平台的單次付費慣性。

**三層定價結構（對應 FinisherPix 模式延伸）：**

```typescript
interface EventPricingTier {
  eventId: string;
  selectedTier: 'free' | 'standard' | 'premium';

  tiers: {
    free: {
      photosPerRunner: number;              // 預設 3 張
      pushPlatforms: ('line' | 'instagram' | 'threads')[];  // 預設僅 LINE
      newspaperEnabled: boolean;            // 預設 false
      faceReIdEnabled: boolean;             // 預設 false(降低 AI 成本)
      customWatermark: boolean;             // 預設 false
      pricePerEvent: number;                // 0(免費)
      targetUseCase: string;                // '小型賽事(< 500 人) / 試營運'
    };
    standard: {
      photosPerRunner: number;              // 預設 8 張
      pushPlatforms: ('line' | 'instagram' | 'threads' | 'facebook')[];
      newspaperEnabled: boolean;            // 預設 true
      faceReIdEnabled: boolean;             // 預設 true
      customWatermark: boolean;             // 預設 true
      pricePerEvent: number;                // 預估 NT$50,000
      targetUseCase: string;                // '中型賽事(500-5,000 人) / 標準方案'
    };
    premium: {
      photosPerRunner: 'unlimited';
      pushPlatforms: ('line' | 'instagram' | 'threads' | 'facebook' | 'x')[];
      newspaperEnabled: boolean;            // 預設 true
      faceReIdEnabled: boolean;             // 預設 true
      customWatermark: boolean;             // 預設 true
      customNewspaperTemplate: boolean;     // 預設 true(客製 PDF 模板)
      aestheticScoringEnabled: boolean;     // 預設 true(美學評分選圖)
      pricePerEvent: number;                // 預估 NT$200,000+
      targetUseCase: string;                // '大型賽事(> 5,000 人) / 國際賽 / 城市形象賽事'
    };
  };

  // 升級觸發條件(自動推薦下一層)
  upgradeTriggers: {
    runnerCountThreshold: number;           // 預設 500 → 推薦 standard;5000 → 推薦 premium
    photoCountThreshold: number;            // 預設 5,000 張 → 推薦 premium
    socialImpressionsThreshold: number;     // 預設 100,000 → 推薦 premium(基於預估)
  };
}
```

**賽事方 ROI 計算面板（後台 Admin Dashboard）：**

```typescript
interface EventRoiMetrics {
  eventId: string;

  // 觸及指標(Reach Metrics)
  reach: {
    totalPushDelivered: number;             // 成功推播至至少一個平台的總張數
    uniqueRunnersReached: number;           // 收到推播之獨立跑者數
    totalSocialImpressions: number;         // 推播貼文之累積曝光(由各平台 API 回傳)
    totalSocialEngagements: number;         // 讚 + 留言 + 分享 累積數
    hashtagExposure: number;                // 賽事 hashtag 之擴散觸及人數
  };

  // 成本指標(Cost Metrics)
  cost: {
    eventPricingTierCost: number;           // 該場賽事所選方案費用
    aiApiCost: number;                      // AI 推論 API 費用(可由 §2.10 策略影響)
    linePushCost: number;                   // LINE 官方帳號 Push Message 費用
    storageCost: number;                    // S3 儲存費用
    totalInfrastructureCost: number;        // 上述加總
  };

  // 價值指標(Value Metrics)
  value: {
    costPerImpression: number;              // totalInfrastructureCost / totalSocialImpressions
    costPerRunner: number;                  // totalInfrastructureCost / uniqueRunnersReached
    brandAwarenessLift: number;             // 賽事 hashtag 與品牌字詞之搜尋量提升(可選整合 Google Trends API)
    mediaValueEquivalent: number;           // 等價媒體曝光價值(假設 NT$50/CPM)
  };

  // 主辦方報告匯出
  exportableReport: {
    format: 'pdf' | 'xlsx';
    recipients: string[];                   // 主辦方決策者 email
    autoEmailAfterDays: number;             // 賽事結束後 N 天自動寄送
  };
}
```

**§3.8 後台新增面板項目：**

| 面板 | 內容 | 對應介面 |
|:----|:----|:----|
| **既有四項（保留）** | 照片處理 / 跑者互動 / DLQ 處理 / 網路監控 | §3.8 原表 |
| **賽事方 ROI 儀表板** | 觸及/成本/價值三大指標 + 自動報告匯出 | `EventRoiMetrics` |
| **定價方案升級推薦** | 依 `upgradeTriggers` 自動通知主辦方升級方案 | `EventPricingTier.upgradeTriggers` |

### **3.9 平台條款變動應變計畫**

基於 deep-research 對 Meta Threads API 存取審查(被驗證者描述為「extremely difficult」與「opaque」)與 Threads 限流(滾動 24 小時 250 則貼文 + 1,000 則回覆)之觀察,Meta/Instagram/Threads 政策變動頻率高且無預警。本章定義平台條款變動之監控、分級應對與依賴度降低策略。

**為何需要獨立章節：**

- 2018 Cambridge Analytica(FB 政策急轉彎)、2020 iOS 14 ATT(影響所有 IDFA 追蹤)、2024 Threads 上線(新平台新規則)、2026 Threads 限流新規(每日上限) — 顯示社群平台政策變動週期已縮短至 12–24 個月
- SPEC §2.8「社群平台整合抽象層」解決「日常 API 差異」,但「政策變動」屬不同層級風險,需獨立治理
- 平台依賴度(Platform Dependency)是 SaaS 商業模式的致命弱點 — 需明確降級路徑

**核心應變介面：**

```typescript
interface PlatformChangeResponsePlan {
  // 監控機制
  monitoring: {
    metaChangelogWebhook: string;           // 訂閱 Meta Developer Changelog RSS
    threadsChangelogWebhook: string;
    metaAppReviewExpiryTracking: boolean;   // Meta App Review 通常 12 個月效期,提前 30 天警告
    quarterlyPolicyReviewMeeting: boolean;  // 每季由 CTO/Product Owner 召開政策檢視會議
  };

  // 風險分級與應對(對應 §4.4 監控 P0/P1/P2 三級)
  riskLevels: {
    P0_immediate: {
      // 例:Meta 撤回 threads_content_publish scope(完全無法推播)
      // 例:Threads 平台停用(如同 2025 年 Twitter → X 之政策急轉)
      trigger: 'meta_revokes_publish_scope' | 'platform_shutdown' | 'rate_limit_zero';
      responseTime: '4 hours';
      onCallEscalation: 'Super Admin + CTO';
      actions: [
        '啟用 Open Graph 預覽卡機制(§2.8)',
        'LINE 全量推播作為 primary',
        'Email 通知所有賽事方',
        '主動聯絡 Meta Partner Manager',
      ];
    };
    P1_warning: {
      // 例:Threads 限流從 250/24h 降為 50/24h
      // 例:Meta App Review 審查週期延長(由 7 天 → 30 天)
      trigger: 'platform_announces_rate_limit_reduction' | 'app_review_delay';
      responseTime: '48 hours';
      actions: [
        '調整 EventFeatureConfig.publishing.rateLimitBuffer',
        '啟用多帳號分散推播(multi-account fallback)',
        '提前 30 天提示主辦方預留備援推播窗口',
      ];
    };
    P2_advisory: {
      // 例:Threads 新增 Stories 推播 API
      // 例:Meta 開放 Threads 商業帳號自動化發文(政策放寬)
      trigger: 'platform_announces_new_feature' | 'platform_announces_relaxation';
      responseTime: '14 days';
      actions: [
        '評估技術可行性',
        '新增 Feature Flag(§2.11)',
        '排入下個 Sprint 優先級',
      ];
    };
  };

  // 平台依賴度降低策略(Decoupling Strategy)
  decouplingStrategy: {
    lineAsPrimary: boolean;                 // LINE 為第一推播目標(日本/台灣滲透率高,API 較穩定)
    fallbackToLine: boolean;                // Meta 失敗時自動降級至 LINE
    emailAsLastResort: boolean;             // Email 報紙連結作為最終交付手段
    galleryFallbackAlways: boolean;         // 無論推播成功與否,所有照片皆寫入 Gallery(跑者主動查詢)

    // 平台依賴度評分(供每季政策檢視會議評估)
    platformDependencyScore: {
      line: number;                         // 0-10,越低越獨立
      instagram: number;
      threads: number;
      facebook: number;
    };
  };
}

interface MetaAppReviewChecklist {
  useCase: 'marathon_photo_auto_publishing';
  requiredPermissions: [
    'instagram_basic',
    'instagram_content_publish',      // 須 Business Account + 連結粉絲專頁
    'pages_show_list',
    'threads_content_publish',         // 須 Meta App Review 通過(極困難)
    'threads_delete'                   // 撤回推播能力,因應跑者撤銷授權
  ];
  businessVerificationCompleted: boolean;
  metaAppReviewStatus: 'not_started' | 'in_review' | 'approved' | 'revoked';
  metaReviewExpiryAt?: string;             // Meta App Review 通常 12 個月效期
  metaTechProviderApplicationId?: string;
  videoSubmissionUrl?: string;             // Meta 審查用影片示範 URL
}
```

**§2.11 Feature Flag 新增項目（由 §3.9 觸發）：**

| 功能代碼 | 功能名稱 | 預設狀態 | 說明 |
|:----|:----|:----|:----|
| `PLATFORM_FALLBACK_CHAIN` | 平台降級鏈 | ✅ 開啟 | 預設順序 `['line', 'instagram', 'threads', 'facebook', 'email']`;主辦方得調整順序 |
| `META_APP_REVIEW_STATUS` | Meta App Review 狀態監控 | ✅ 開啟 | 預設阻擋推播直至審查通過;若 `metaReviewExpiryAt` 過期自動觸發 P1 警告 |
| `THREADS_RATE_LIMIT_BUFFER` | Threads 限流緩衝 | ✅ 開啟 | 預設保留 20% 配額(250 × 0.8 = 200/24h);可由 `EventFeatureConfig.publishing.threadsRateLimitBuffer` 調整 |
| `MULTI_ACCOUNT_PUBLISH` | 多帳號分散推播 | ❌ 關閉 | 當單一 Threads 帳號限流時自動切換至備援帳號;預設關閉以避免帳號管理複雜度 |
| `GALLERY_FALLBACK_ALWAYS` | Gallery 永久降級 | ✅ 開啟 | 無論推播成功與否,所有照片皆寫入 Gallery(跑者主動查詢) — 永遠的最後手段 |

**Threads 限流緩衝實作（§3.9 衍生細則）：**

- Threads 對單一 profile 滾動 24 小時 250 則貼文 + 1,000 則回覆上限(1 個 carousel 算 1 則貼文)
- 系統於推播引擎內維護 Redis Token Bucket:
  - `key: threads:rate_limit:{threads_profile_id}:window_24h`
  - `limit: 200(預設保留 20% 緩衝)`
  - 觸發限流時:`EventFeatureConfig.publishing.threadsRateLimitBuffer` 自動減量,並將超額任務延後至下個 24 小時視窗
- 超限任務不寫 DLQ,而是延遲重試(不同於一般 5xx 錯誤)

**Meta App Review 流程支援：**

- §3.8 後台既有「App Review Submission Helper」自動產生示範影片腳本與截圖
- 本章節新增:`MetaAppReviewChecklist` 介面供 Super Admin 追蹤審查狀態與效期
- 當 `metaReviewExpiryAt` 倒數 30 天,系統自動發送提醒給 Super Admin,要求提前送審續期

**監控指標（新增至 §4.4 業務關鍵指標）：**

| 指標 | 說明 | 警示閾值 |
|:----|:----|:----|
| `meta_push_success_rate` | Meta(IG/FB/Threads)推播成功率 | < 95% 觸發 P1 警告 |
| `threads_push_success_rate` | Threads 推播成功率(獨立監控因 Threads 政策最不穩定) | < 90% 觸發 P1 警告 |
| `meta_app_review_days_to_expiry` | Meta App Review 距到期天數 | < 30 天觸發 P2 警告 |
| `push_withdrawal_count` | 跑者撤回推播次數(§3.1.Y) | > 5% 推播總數時觸發 P2 UX 信任度警告 |

## **4\. 非功能性需求 (Non-Functional Requirements)**

非功能性需求定義了系統在極端壓力下的表現與基礎設施的穩健程度。對於一項需要即時處理巨量多媒體數據與個人隱私的賽事系統而言，高併發處理能力、超低延遲的運算架構與堅若磐石的資安防護是不可或缺的三大支柱。

### **4.1 高併發照片上傳與智慧流量塑形 (Traffic Shaping)**

在大型馬拉松賽事中，選手通常呈現集團式移動。當大批跑者同時通過熱門攝影點（如起終點或知名地標）時，多台攝影設備將在極短的時間內上傳數千張高解析度檔案至雲端。這種典型的「突發性流量（Bursty Traffic）」若以同步（Synchronous）方式直接交由後端伺服器處理，極易導致運算資源瞬間耗盡、資料庫連線池崩潰，甚至引發雪崩效應3。  
為應對此挑戰，本系統高度依賴 Amazon SQS 作為核心的流量塑形與緩衝機制。當 S3 接收到新影像時，僅將輕量化的物件元資料（Metadata）非同步發送至 SQS 佇列中3。AWS Lambda 被配置為 SQS 的消費者，利用其強大的擴展性進行影像消化。雖然 AWS Lambda 理論上可以每分鐘增加 60 個實例，瞬間擴展至高達 1,000 個以上的併發執行環境，但這種無節制的擴充將帶來毀滅性的後果——瞬間發出的大量請求會觸發下游社群平台 API 之速率限制（Meta Graph API 以 Instagram Business Account 維度計算，24 小時內最多 25 篇貼文）與關聯式資料庫的速率限制（Rate Limits）16。  
因此，系統架構師必須在 SQS 與 Lambda 之間的事件源映射（Event Source Mapping）中設定「最大併發數量（Maximum Concurrency）」16。藉由嚴格控制同時啟動的 Lambda 實例上限（例如限制在 50 個併發），系統能確保影像處理的吞吐量穩定維持在下游 API 可承受的安全閾值內。未及時處理的影像則安穩地保留在 SQS 佇列中等待消化。此外，搭配 DLQ 設計，即使遇到損毀的影像或 OCR 演算法意外崩潰，該問題任務也會被隔離至 DLQ 中進行後續的人工除錯或重新驅動，確保絕不遺漏任何一張跑者的珍貴影像8。

### **4.2 AI 辨識延遲度與無伺服器冷啟動 (Cold Start) 優化**

提升即時推播體驗的核心關鍵，在於將從「影像上傳」到「社群發布」的端到端時間差壓縮至最短。AI 影像辨識與無頭瀏覽器渲染作為整個管線中最消耗運算與記憶體資源的環節，其延遲度的控制至關重要。

**延遲目標（SLA）：**從攝影師按下快門、影像上傳至 S3 起，至跑者收到 LINE 推播通知為止，目標延遲為 **5 分鐘以內**（P95）。困難場景（OCR 置信度低、需觸發 Face Re-ID fallback）可延長至 15 分鐘。完賽報紙 PDF 生成則於賽事正式成績公告後 30 分鐘內完成。

**可用性目標（Availability SLA）：**

| 元件 | 可用性目標 | 說明 |
| :---- | :---- | :---- |
| **系統整體（照片攝入到推播）** | ≥ 99.5%（賽事日 8 小時） | 含 S3、Lambda、SQS、DynamoDB 全部串聯 |
| **跑者圖庫查詢 API（Gallery）** | ≥ 99.9% | 讀取導向，CDN 快取可遮蔽 |
| **後台管理系統** | ≥ 99%（平日 24 小時） | 非即時服務，備援容許較低 |
| **LINE 推播（最終抵達）** | ≥ 99% | 受 LINE 平台可用性約束，系統端僅承担 API 呼叫成功責任 |

> **Note：** 可用性目標以賽事日（event day）8 小時為計算窗口。平日（非賽事日）可用性目標可相應下調至 99%。  

**端到端延遲預算表（End-to-End Latency Budget）：**

為驗證 SLA「P95 ≤ 5 分鐘」之可達成性，系統對端到端處理鏈各階段定義明確時間預算：

| 階段 | 預算時間 | 說明 |
| :---- | :---- | :---- |
| Cold Start（Provisioned Concurrency 暖機） | ≤ 1 秒 | 暖機實例跳過初始化，可立即處理 SQS 訊息 |
| Cold Start（未暖機，首次） | 5–15 秒 | 透過 Provisioned Concurrency（§4.2）規避；此情境僅發生於賽事前 5 分鐘冷啟動 |
| Photo Processing（含 AI 推論） | ≤ 240 秒 | 由 Inference Cascade（§2.10）控制；級聯降級總時長上限 240 秒（見 `globalTimeoutMs`，§2.10） |
| 美化渲染（Sharp） | ≤ 5 秒 | 記憶體內 libvips 合成 |
| Publish Lambda（含社群 API 呼叫） | ≤ 30 秒 | 含 LINE Push / IG Content Publishing 兩階段上傳 |
| **P95 總和（暖機路徑）** | **≤ 276 秒** | 1 + 240 + 5 + 30 = 276 秒，遠低於 5 分鐘 SLA（300 秒） |
| **P95 總和（困難場景，走完整 Fallback 鏈）** | **≤ 15 分鐘** | 滿足 `hardCapMinutes` 之 hard cap |

> **排除項說明：** 以上預算**未計入**：
> - **SQS Long Polling Wait Time** — §2.2 Receive Request Wait Time 設為 20 秒；理論上 P95 可能命中該 wait time
> - **X-Ray SDK overhead** — 6 個 segment 建立累計 ~100-150ms
> - **API Gateway cold start** — 首次 cold start ~300-500ms
>
> 完整 P95 仍穩定低於 5 分鐘 SLA，具 ~24 秒緩衝空間。

**備註：** Lambda Timeout 已從 60 秒調升至 **240 秒**（詳見 §4.2 配置表），與 `globalTimeoutMs` 一致，以避免 Cascade Fallback 在第一階段尚未完成時被 AWS Lambda 硬性 timeout 中斷。Face Re-ID Lambda Timeout 從 45 秒調升至 120 秒（兩階段 Cascade）。

**Lambda 函數配置表（依賽事規模 20,000 人基準）：**

| 函數 | 觸發來源 | Memory | Timeout | Reserved Concurrency | Provisioned Concurrency | 冷啟動影響 |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **Photo Processing Lambda** | SQS `photo-processing-queue` | 1024 MB | **240 秒** | 100 | 20 | 高（Cold Start 5-10s） |
| **Face Re-ID Lambda** | SQS `face-reid-queue` | 1024 MB | 120 秒 | 15 | 5 | 中（Cold Start 3-5s） |
| **Publish Lambda** | SQS `publish-queue` | 512 MB | 30 秒 | 30 | 5 | 低（Cold Start 1-2s） |
| **PDF Generation Lambda** | SQS `pdf-generation-queue` | 2048 MB | 120 秒 | 20 | 10 | 極高（Cold Start 8-15s，Headless Chromium 啟動） |
| **Callback Lambda**（OAuth） | API Gateway | 512 MB | 15 秒 | 10 | 0 | 低 |
| **Webhook Lambda**（LINE/Meta） | API Gateway | 512 MB | 10 秒 | 10 | 0 | 低 |
| **Register API Lambda** | API Gateway | 512 MB | 10 秒 | 20 | 0 | 低 |
| **DLQ Resolve Lambda** | API Gateway | 512 MB | 10 秒 | 5 | 0 | 低 |
| **DLQ Auto Email Lambda** | EventBridge Schedule | 256 MB | 30 秒 | 1 | 0 | 極低 |

**Reserved Concurrency 預算說明：**
- 單 AWS account Lambda 並發上限預設 1,000。本系統合計 Reserved = 211（賽事日尖峰期間）,低於 account 上限,保留 ~789 並發空間供其他系統或 burst 緩衝。
- 建議於 Production AWS Account 透過 `aws lambda put-account-concurrency` 預留 500 並發給本系統,避免其他服務意外佔用影響賽事。

**Provisioned Concurrency 成本估算：**
- 20K 賽事單日 8 小時：Photo Proc 20 × $0.0000041/GB-s × 1GB × 28800s = ~$2.36;PDF 10 × $0.0000041 × 2GB × 28800s = ~$2.36;其餘忽略。合計 < $10/賽事日。
- 平日時段（無賽事）以 EventBridge Schedule 自動將 Provisioned Concurrency 降為 0,僅保留 Reserved Concurrency 上限避免 cold start。

**Cold Start 全局聲明 vs 個別函數對照（v1.5 補充）：**

| 項目 | 全局聲明（§4.2 開頭） | 個別函數（上方表格） | 差異原因 |
|:----|:----|:----|:----|
| 未暖機 Cold Start 範圍 | 5–15 秒 | Photo Proc 5-10s / Face Re-ID 3-5s / Publish 1-2s / **PDF 8-15s** | 全局 5-15s 為「多數 AI/業務 Lambda」之代表值；PDF Generation Lambda 因包含 Headless Chromium 啟動（~50 MB binary 解壓縮），實際冷啟動更長 |
| 影響層級 | 整體 SLA 計算用 | Provisioned Concurrency 配置決策用 | 全局數字納入延遲預算表（§4.2）；個別數字決定 Provisioned Concurrency 應配置多少（冷啟動越長，PC 越多） |

**配置策略：** PDF Generation Lambda 因冷啟動 8-15s 對 P95 SLA（5 分鐘）影響最大，故配置最高 Provisioned Concurrency = 10（詳見上表）；Photo Processing Lambda 雖冷啟動 5-10s 較短，但尖峰 QPS 高，故 PC = 20；其餘 Lambda 配置 0-5 PC 即可，因冷啟動對其影響在容許範圍。

在雲端無伺服器架構中，最棘手的問題是「冷啟動（Cold Start）」延遲。當系統需要載入龐大的 AI 模型參數，或解壓縮高達數十 MB 的 Headless Chromium 執行檔與 Node.js 依賴庫時，Lambda 函數的初始化動輒需要耗費 5 到 15 秒39。為徹底克服此痛點，系統將針對負責 AI 辨識與 PDF 生成的關鍵 Lambda 函數啟用「預先配置的併發（Provisioned Concurrency）」17。此設定能確保賽事期間始終有一批運算實例維持在「暖機（Warm）」的待命狀態，一旦 SQS 分配任務，函數即可在數毫秒內啟動執行緒進行處理。具體 Provisioned Concurrency 數量與各函數 Memory / Timeout 設定見 §4.2 配置表。

此外，在 Lambda 的組態設定上，針對運行 Puppeteer 的函數，必須分配至少 2048 MB 的記憶體，以確保瀏覽器引擎有足夠的資源進行 HTML 與高解析度圖片的渲染，同時將逾時（Timeout）設定延長至 120 秒（PDF 生成場景）以防超時中斷20。在程式碼實作層面，系統將採用單例模式（Singleton Pattern），在 Lambda 的全域執行環境中持久化加載 AI 模型與瀏覽器實例（Browser Instance）；使得同一個運算容器在處理連續不斷的照片流時，能重複利用已啟動的資源，從根本上分攤掉初始化的龐大時間開銷20。

### **4.3 資安與隱私權（合規性與社群授權）**

社群授權代幣與跑者影像涉及極高的資安與隱私風險。從基礎架構層面觀之，所有存放在資料庫中的 OAuth Access Token 與 Refresh Token 必須採用應用程式層級的高強度加密（如 AES-256-GCM 演算法），並搭配 AWS Key Management Service (KMS) 進行金鑰輪替與安全存取管控。這確保即使資料庫層面遭到惡意攻破，攻擊者也無法解密並濫用被盜取的 Token 侵入跑者的私人社群帳號。在資料傳輸層，無論是 S3 上傳、SQS 訊息傳遞，或是與第三方社群 API 的通訊，全程強制使用 TLS 1.2 以上版本的加密協定14。  
在法律合規性方面，本專案嚴格遵循台灣《個人資料保護法》（PDPA）的規範。對於系統涉及的影像收集與自動發布行為，隱私權保護機制涵蓋以下落實面向：

| PDPA 法規核心要求 | 系統實作與合規對策 |
| :---- | :---- |
| **特定目的明確性與告知** | 註冊時必須明確宣告收集照片的專屬特定目的為「賽事成績發布與社群留念」，嚴禁將蒐集之照片流用於未經告知的商業演算法訓練或無關的第三方廣告行銷25。 |
| **獨立與明確同意 (Separate Consent)** | 社群自動推播之授權機制不得使用預設打勾，且須與賽事一般報名條款完全分離（Separate Declaration），確保跑者在充分知情下擁有真實的選擇權27。 |
| **特種個人資料之處理** | 若未來擴充使用人臉特徵值比對（Biometric Data），依法將被視為高度敏感的特種個人資料，系統必須依法取得書面同意或具備同等效力的強烈電子同意憑證，並建立嚴格的存取控制24。 |
| **資料最小化與永久抹除** | 照片：賽事正式成績公告後 **30 天**自動刪除原始未處理圖片；DLQ 中人工未確認之照片於賽事結束後 **48 小時**強制刪除。處理後已推播之壓縮圖片：賽事結束後 **90 天**自動刪除。完賽報紙 PDF：跑者以下載連結取得後 **30 天** 或 賽事結束後 **180 天**（兩者取其早）自動刪除。OAuth Token：賽事結束後 **30 天** 或 跑者主動撤銷後 **7 天**銷毀（兩者取其早）。Face Re-ID 自拍照（生物特徵資料）：於 Face Re-ID 流程完成後 **24 小時**內自動刪除，不得留存。依據隱私政策，所有資料皆須以 Secure Erasure 方式（即資料覆寫或實體銷毀）處理，不可僅做邏輯刪除28。 |

#### **PDPA §21 跨境傳輸合規**

依台灣《個人資料保護法》第 21 條，個資原則上不得跨境傳輸至第三地（非台灣）。本系統因 Primary Region 預設為 `ap-northeast-1`（東京），屬「第三地」，須以下列機制合規：

| 機制 | 實作 |
| :---- | :---- |
| **Separate Consent for Cross-Border Transfer** | 註冊流程中，於「隱私聲明」獨立頁面以顯著 checkbox 取得跑者對「個資將傳輸至日本 AWS 東京機房」之明確同意，不得預設勾選 |
| **同意紀錄持久化** | DynamoDB `Runner` 表新增 `consent.crossBorderTransferGranted: boolean`、`consent.crossBorderTransferTimestamp: string`、`consent.privacyPolicyVersion: string` 三欄位,供稽核時舉證 |
| **撤銷機制** | 跑者可隨時呼叫 `DELETE /api/v1/races/{eventId}/runner/{bibNumber}/revoke` 撤銷跨境同意,系統於 7 日內刪除海外節點之副本（透過 S3 Cross-Region Replication 刪除標記 + DynamoDB Global Tables 條件刪除） |
| **特種個資加強同意** | Face Re-ID 自拍照屬「特種個人資料」（生物特徵），跨境同意須額外以「顯著電子同意」流程取得（雙重確認 + OTP 驗證 + 同意書 PDF 留存於 S3,加密儲存） |
| **稽核軌跡** | 所有跨境同意之建立 / 撤銷事件寫入 CloudTrail,保留 7 年 |

**部署拓樸對跨境同意之影響：**

| 拓樸代碼（見 §4.4） | 是否需跨境同意 | 適用情境 |
| :---- | :---- | :---- |
| `aws-tokyo` | 須取得（見上表機制） | 預設；最低延遲 |
| `aws-taipei-local` | 無需（Primary Region 預留於 AWS 台北區,假設 AWS 後續啟用） | 對 §21 有疑慮之主辦單位 |
| `on-premise` | 無需（資料全程於賽事主辦單位自有機房） | 政府機關賽事 / 強制資料落地 |

### **4.4 監控、警示與災難復原**

本系統於賽事當日須具備完整的可觀測性（Observability），以確保任何環節異常能在第一時間被发现並處置。

**業務關鍵指標（Business KPIs）與監控儀表板：**

| 指標 | 說明 | 警示閾值 |
| :---- | :---- | :---- |
| OCR 辨識成功率 | 成功取得有效 bibNumber 的照片比例 | < 70% 時觸發 P2 警示 |
| 推播抵達率 | 成功推播至至少一個平台 / 總處理照片數 | < 80% 時觸發 P1 警示 |
| P95 端到端延遲 | 從 S3 上傳到 LINE 推送通知的 P95 時間 | > 5 分鐘時觸發 P2 警示 |
| Lambda 錯誤率 | 各 Lambda 函數之錯誤率（5xx / Invocation） | > 1% 時觸發 P1 警示 |
| SQS 佇列深度 | 等待處理之訊息數量 | > 5000 時觸發 P2 警示 |
| Token 刷新失敗率 | 刷新請求中失敗的比例 | > 5% 時觸發 P2 警示 |

**CloudWatch 警示與 On-Call 政策：**賽事進行期間，on-call 工程師須於 alert 觸發後 15 分鐘內確認並開始處理。P1 alert（如 Lambda 錯誤率飆升、SQS 堆積）須於 5 分鐘內口頭回報。

**Primary AWS Region 與部署拓樸：**

本系統定義三種部署拓樸選項，由賽事主辦單位於 `EventConfig.deploymentTopology` 欄位選擇（預設 `aws-tokyo`）。所有拓樸共用同一份核心業務邏輯、Lambda 程式碼與 Terraform module，僅 `infrastructure/environments/{topology}/` 之 provider 與 region 設定不同；CI/CD 矩陣（GitHub Actions）依此值展開部署管線。

| 拓樸代碼 | Primary Region | DR Region | 適用情境 | PDPA 跨境同意 |
| :---- | :---- | :---- | :---- | :---- |
| **`aws-tokyo`**（預設） | `ap-northeast-1`（東京） | `ap-southeast-1`（新加坡） | 最低延遲、已熟悉 AWS 東京區之主辦單位 | 須取得跑者明確同意（詳見 §4.3） |
| **`aws-taipei-local`** | 預留（AWS 若於 2026 後啟用台北區） | `ap-northeast-1`（東京） | 對 PDPA §21 跨境傳輸有疑慮之主辦單位 | 無需跨境同意 |
| **`on-premise`** | 主辦單位自有資料中心（K8s / EKS Anywhere） | 同機房異機架 | 政府機關賽事 / 強制資料落地需求 | 無需跨境同意 |

Terraform 透過三個變數管理：`var.deployment_topology`、`var.primary_region`、`var.dr_region`。所有 AWS 資源以 Resource Tag `DeploymentTopology={value}` 與 IAM Condition `aws:ResourceTag/DeploymentTopology = ${var.deployment_topology}` 雙重標註，便於成本分帳與資源盤點。

**災難復原（DR）策略：**

| 情境 | 復原方式 | RTO | RPO |
| :---- | :---- | :---- | :---- |
| 單一 Lambda 函數崩潰 | SQS 自動重試（最多 3 次）+ DLQ 寫入 | < 5 分鐘 | 照片可能延遲但不遺失 |
| DynamoDB 吞吐量限流（Throughput Throttling） | DynamoDB Auto Scaling 自動擴增 WCU/RCU + 指數退避 | < 1 分鐘 | 0（無資料遺失，僅延遲） |
| DynamoDB 資料損毀（Data Corruption / 誤刪） | Point-in-time Recovery（PITR）+ 人工 restore 至過去 35 天內任一時刻 | < 15 分鐘（人工觸發後） | 可恢復至過去 35 天內任一時刻（連續備份，精度 1 秒） |
| DynamoDB Global Tables 跨區同步失敗 | Conflict Resolution + Last-Writer-Wins（預設） | < 5 分鐘（自動） | 可能丟失最近 1-5 秒寫入（依 RTT） |
| S3 儲存桶意外刪除 | S3 Versioning 開啟 + Life Cycle Rule 保存版本 | < 1 小時 | 最近刪除版本可復原 |
| Primary Region（預設 `ap-northeast-1`）全斷 | 跨區域備援：照片以 S3 Cross-Region Replication 同步至 DR Region；DynamoDB Global Tables 雙寫；推播 SQS 自動切換至 DR Region 之 queue | < 4 小時 | 最近 5 分鐘資料 |
| 賽事當日網路瞬斷（5G） | 每站攝影機本機 SSD 暫存，連線恢復後自動續傳（Resumable Upload） | 0（本地暫存） | 0（本地備份） |

**On-Call Runbook：**每次賽事須備有 `oncall-runbook-{eventId}.md`，內容包含：各 Lambda 函數 ARN、錯誤率儀表板連結、DLQ 人工處理頁面 URL、AWS Support 聯絡方式、備援推播（純文字 LINE 通知）觸發條件。

**分散式追蹤（Distributed Tracing）：** 端到端管線跨越多個 Lambda 函數與 SQS，傳統 CloudWatch Logs 難以關聯單一照片之完整生命週期。系統採用 AWS X-Ray（預設）或相容 OpenTelemetry SDK，以 Trace ID 串接以下環節：

| 環節 | X-Ray 子segment |
| :---- | :---- |
| S3 上傳觸發 | `s3.put` |
| SQS 進入 | `sqs.receive` |
| Lambda 偵測 | `lambda.invoke` → AI detection |
| Lambda OCR | `lambda.invoke` → OCR |
| Lambda 美化渲染 | `lambda.invoke` → Sharp |
| 推播 SQS 寫入 | `sqs.put` |
| 推播 Lambda | `lambda.invoke` → social adapter |
| LINE API 呼叫 | `http.post` |
| DynamoDB 寫入 | `dynamodb.putItem` |

Trace ID 會寫入 DynamoDB 任務狀態記錄，供除錯時直接以 Trace ID 查詢完整 call chain。取證時以 CloudWatch Contributor Insights 匯出高延遲 traces。

### **4.5 測試策略**

本系統之測試分為四個層級，各層級有不同的通過標準與工具選擇：

| 測試層級 | 測試標的 | 工具 | 通過標準 |
| :---- | :---- | :---- | :---- |
| **單元測試（Unit）** | 各 Lambda 函數業務邏輯、Adapter Pattern 各平台模組、Token 刷新邏輯、照片歸戶決策函數 | Jest / Pytest | 分支覆蓋率 ≥ 80% |
| **整合測試（Integration）** | SQS → Lambda → DynamoDB 寫入流程、RFID API → Lambda → S3 觸發、Sharp 影像合成輸出驗證 | LocalStack + Jest/Pytest | 所有路徑 100% 通過 |
| **AI 模型驗收測試** | OCR 辨識準確率（以獨立測試集驗證）、Face Re-ID 置信度分佈 | 獨立測試 Dataset（含雨天/夜間/號碼布遮蔽/集團通過等 Edge Case） | OCR ≥ 91.6% 準確率（依規格書目標）；Face Re-ID ≥ 85% Top-1 準確率 |
| **端到端測試（E2E）** | 從 Mock 攝影機上傳 → S3 → SQS → Lambda → LINE Push Message 送達 | staging 環境 + LINE Debug 帳號，模擬 1000 張照片批次處理 | SLA P95 ≤ 5 分鐘達成，無錯誤 |
| **負載/壓力測試（Load & Stress）** | 模擬 5000 張照片/小時持續湧入（集團通過高峰），驗證 SQS 佇列深度、Lambda 並發上限、OCR 成功率不掉速 | k6（HTTP 層）+ AWS Lambda concurrent invocations 監控 + 實際照片流 | 5000 張/小時 持續 30 分鐘：P95 延遲不超過 8 分鐘（加計 Line平台容許延遲），Lambda 錯誤率 < 1%，SQS 無訊息堆積 |
| **破壞性測試（Chaos）** | 人為關閉 Lambda、網路瞬斷、S3 故意回傳 500，驗證 DLQ 寫入、SQS retry、DLQ 通知觸發正確 | AWS Fault Injection Simulator ( FIS ) | DLQ 任務正確寫入，賽事日第一天 end-to-end P95 ≤ 10 分鐘 |
| **PDPA 撤銷端到端測試**（v1.5 新增） | 跑者呼叫 `DELETE /revoke` → DynamoDB `OAuthToken.status='revoked'` → S3 Cross-Region Replication 刪除標記 → DynamoDB Global Tables 條件刪除 → CloudTrail 事件寫入 → 推播引擎下次嘗試跳過該平台 → Gallery fallback 啟用 | staging + 多 Region mock + CloudTrail 查詢 | 全鏈路 < 5 分鐘完成；CloudTrail 100% 寫入；§4.3 §21 海外節點副本確認刪除 |

**AI 模型訓練資料來源：**採用 Hugging Face `race-numbers-detection-and-ocr` 資料集作為基準訓練集，並於每場賽事後以實測失敗案例擴充訓練集，持續微調模型權重。模型版本以 Semantic Versioning 管理，並於 `/models/{version}/` 目錄存放每次上線前之模型 checkpoint。

### **4.6 CI/CD 部署流程**

本系統所有程式碼與基礎設施皆以 Git 為單一事實來源（Single Source of Truth），部署流程分為四個環境：

| 環境 | 觸發條件 | 部署內容 | 審核要求 |
| :---- | :---- | :---- | :---- |
| **dev** | 任何分支 push | 自動部署至開發 AWS 帳號，供開發團隊內測 | 自動，無人工審核 |
| **staging** | PR merge 至 `develop` | 部署至 staging 環境，觸發 E2E + Load Test | 自動，測試通過後通知 QA |
| **preview** | 每次新 PR 開啟 | 部署 Preview 環境（隔離，獨立資源），附上 Lambda ARN 供 reviewer 測試 | 自動，PR 作者通知 |
| **production** | PR merge 至 `main` | 部署至 production AWS 帳號，觸發 smoke test | 需要 Super Admin 人工核准 |

**部署管線（使用 GitHub Actions 或同類 CI）：**
```
1. Lint (ESLint / Prettier / tsc --noEmit)
2. Unit Tests (Jest --coverage)
3. Docker Build (for Lambda custom runtimes)
4. Terraform Plan (aws plan, 差異輸出至 PR comment)
5. Deploy to target environment
6. Smoke Test (/health, /api/v1/races/{testEventId}/status)
7. (staging only) E2E + Load Test
8. (production only) Approval Gate → Deploy
```

**Infrastructure as Code：** 所有 AWS 資源由 Terraform 管理，儲存於 `infrastructure/` 目錄。每一個 Terraform state 鎖定於對應環境（dev/staging/prod），不跨環境共享。

**回滾（Rollback）流程：** production 部署失敗時，GitHub Actions 自動觸發 `terraform apply -target=<previous_version>`，並執行 `/admin/v1/health/rollback` 內部 API 確認服務上線。

### **4.7 Face Re-ID 自拍照儲存生命週期**

Face Re-ID 為選配功能，跑者於註冊時自願上傳清晰自拍照作為日後比對之用。該照片屬於《個人資料保護法》特種個人資料（生物特徵），儲存與處理須遵守以下規定：

| 階段 | 儲存位置 | 加密方式 | 保留期限 | 銷毀方式 |
| :---- | :---- | :---- | :---- | :---- |
| **上傳時** | Amazon S3（隔離 bucket） | AES-256-GCM 客戶自管金鑰（Customer-Managed CMK） | — | — |
| **比對完成後** | — | — | 拍照後 **24 小時** | S3 立即刪除 + CMK 輪替（使舊資料無法解密） |
| **系統錯誤（比對失敗）** | S3 | AES-256-GCM CMK | 最長 **48 小時** | 自動刪除 |
| **DLQ 人工處理中** | S3 | AES-256-GCM CMK | **48 小時** | 人工確認完成後立即刪除自拍檔，僅保留比對結果（分數，不含原始圖） |

> **嚴禁事項：** 自拍照不得寫入一般物件儲存桶（與跑者照片共用）；不得與號碼布照片寫入相同前綴；不得留存於 Lambda 容器磁碟（/tmp 或任何可持久化位置）；不得寫入 CloudWatch Logs。

### **4.8 成本模型**

本節定義系統運行之估算成本，供賽事主辦單位評估費用。成本以 20,000 人賽事為基準（假設 60% 照片可識別推播，每位跑者平均 3 張照片）。

**估算假設（重要）：** 以下成本數字之計算基於以下假設;若實際部署調整,須重新計算：

- **Lambda Memory 配置** — Photo Processing 1024 MB、Face Re-ID 1024 MB、Publish 512 MB、PDF Generation 2048 MB、其餘 512 MB（詳見 §4.2 配置表）
- **Lambda 計價單位** — AWS Lambda $0.0000166667/GB-second（含免費 100 萬次/月 + 400,000 GB-second）
- **AWS Region** — `ap-northeast-1`（東京）單價;若採 `on-premise` 拓樸則 AWS 費用為 0,僅計硬體折舊與電力
- **DynamoDB 模式** — On-Demand 為主（§2.5）;若改採 Provisioned 須另計 baseline
- **賽事日 8 小時** — 平日（非賽事日）Provisioned Concurrency 自動降為 0,僅 Reserved Concurrency 上限
- **S3 儲存週期** — 原始 30 天 + 處理圖 90 天;若調整須重算儲存費用
- **AI API 費用** — 採「延遲優先」策略（Gemini Flash）之估算;若採「成本優先」本地模型則 AI 費用為 0,僅計 GPU 硬體折舊

**兩類啟用率假設（嚴格區分，不可混用）：**

| 變數 | 預設值 | 定義 | 影響之費用項目 |
|:----|:----:|:----|:----|
| `ocrSuccessRate`（OCR 成功率） | 60% | 進站照片中可被 AI 識別出有效 `bibNumber` 之比例 | AI API 費用（每張被處理之照片皆需呼叫一次 AI） |
| `runnerActivationRate`（跑者啟用率） | 60% | 已報名跑者中完成 OAuth 授權並成功綁定至少一個社群平台之比例 | LINE/IG/Threads 推播費用（每位啟用跑者最多 `pushMaxPerEvent` 張推播） |

**重要：此兩變數各自獨立，數值可能不同。** 典型情境：OCR 成功率 70% 但跑者啟用率僅 35%（許多跑者未完成 OAuth）。下表所有 LINE/IG/Threads 推播費用計算以 `runnerActivationRate` 為基準；AI API 費用以 `ocrSuccessRate` 為基準。

#### **LINE 官方帳號費用（台灣賽事必備）**

| 方案 | 費用 | Push Message 上限 | 適用場景 |
| :---- | :---- | :---- | :---- |
| 輕用量（Light） | NT$0 / 月 | 200 則/月 | 測試環境、小型賽事（< 200 推播） |
| 中用量（Standard） | NT$800 / 月 | 3,000 則/月 | 5,000 人賽事（假設 35% 啟用率 × 3 張 = 5,250 則） |
| 高用量（Pro） | NT$1,200 / 月 | 6,000 則/月 | 20,000 人賽事（假設 25% 啟用率 × 3 張 = 15,000 則，不足；須加購） |
| 無限制（Premium） | NT$15,000 / 月 | 無上限 | 大型賽事或多場次賽事主辦方；高用量加購每則 NT$0.2 起 |

> **Note：** 上述方案容量對照假設：20,000 人賽事若 `runnerActivationRate=60%`（12,000 啟用跑者，詳見 §4.8 兩類啟用率假設表），每人 `pushMaxPerEvent=5` 張 = 60,000 則 Push 上限 — 此為**推播上限**，實際推播張數由 §3.1.Y 推播偏好設定與 §3.7 Cluster dedup 後的最終張數決定。中用量（3,000）與高用量（6,000）方案均明顯不足，建議直接採用無限制方案（NT$15,000/月）或高用量搭配加購訊息。費用以 LINE 官方公告為準，本表於 2026 上半年之計價版本彙整。

**§4.8 推播張數 vs §3.1.Y pushMaxPerEvent 對照：**
- `pushMaxPerEvent=5`（預設）= 叢集 dedup **後** 對單一跑者的最終推播張數上限（詳見 §3.7）
- LINE 方案容量計算 = `eventParticipants × runnerActivationRate × pushMaxPerEvent`，**非** `eventParticipants × ocrSuccessRate`
- 跑者啟用率與 OCR 成功率為不同概念，詳見 §4.8 假設表；不得混用

#### **AWS 運行成本（20,000 人賽事，單日 8 小時）**

> **算式說明：** AWS Lambda 計價單位為 **$0.0000166667 per GB-second**（即 $16.67 per 1M GB-s）。計算公式：
> `cost = calls × avgDurationSec × memoryGB × $0.0000166667`
> 以下數字依此公式重新計算，**先前的 $30/$6/$30 為單位誤判（per 100ms 而非 per second），已更正**。

| 元件 | 估算用量 | 單價 | 估算費用（USD） | 算式 |
| :---- | :---- | :---- | :----: | :---- |
| **Lambda（照片處理）** | 60,000 次調用 × 3s avg × 1 GB（1024 MB） | $0.0000166667 / GB-s | **~$3.00** | 60,000 × 3 × 1 × 0.0000166667 |
| **Lambda（推播引擎）** | 36,000 次調用 × 1s avg × 0.5 GB（512 MB） | $0.0000166667 / GB-s | **~$0.30** | 36,000 × 1 × 0.5 × 0.0000166667 |
| **Lambda（PDF 生成）** | 12,000 次 × 15s avg × 2 GB（2048 MB） | $0.0000166667 / GB-s | **~$6.00** | 12,000 × 15 × 2 × 0.0000166667 |
| **S3 儲存** | 原始圖 200 GB + 處理圖 50 GB | $0.023 / GB | ~$5.75 | (200 + 50) × 0.023 |
| **SQS** | 60,000 訊息 | $0.40 / 百萬訊息 | ~$0.024 | 60,000 × 0.40 / 1,000,000 |
| **DynamoDB** | 80,000 Write + 20,000 Read（On-Demand 模式） | $1.25 / 百萬 WCU + $0.25 / 百萬 RCU | ~$0.105 | 80,000 × 1.25 / 1M + 20,000 × 0.25 / 1M |
| **CloudFront CDN** | 100 GB 流出 | $0.0085 / GB | ~$0.85 | 100 × 0.0085 |
| **合計（不含 AI API）** | | | **~$16.03 / 賽事日** | |

**修正說明：** 原文件估算 `~$72.65`，係將單價單位誤判為「per 100ms」（即 $0.0000166667/0.1s = $0.000166667/GB-s）所致，相差 10 倍。更正後 Lambda 總額從 $66 → $9.30，賽事日總成本從 $72.65 → $16.03。AI API 費用（§4.8 末段表格）不受此影響，仍維持原估算。

#### **AI API 成本（最大變異項）**

| 策略 | 引擎 | 估算費用（20,000 人賽事） |
| :---- | :---- | ---: |
| **成本優先** | YOLOv8 本地（Tesseract OCR） | ~$0（無 API 費用，GPU 硬體另計） |
| **隱私優先** | Ollama + LLaVA（本地） | ~$0（無 API 費用） |
| **延遲優先** | Gemini Flash（$0.0008/圖） | ~$48 |
| **精度優先** | Claude 3.5 Sonnet（$0.015/圖） | ~$900 |

> **建議：** 平日使用成本優先策略（本地 OCR），大型賽事切換至延遲優先策略（Gemini Flash）兼顧速度與成本。

## **5\. 未來擴充性考量 (Scalability)**

在架構設計初期，本系統便被賦予高度的鬆耦合、模組化與無狀態（Stateless）特性，這為未來的跨領域功能擴充與技術升級提供了極大的彈性與想像空間。  
### **5.1 新興社群平台的無縫整合**

由於系統的核心推播邏輯已將社群 API 的複雜性抽離為獨立的轉接器模組，當未來如 Bluesky（使用 AT Protocol，圖片上限 1MB）、TikTok（支援 Content Posting API）或 Pinterest 等新興平台崛起時44，開發團隊只需撰寫對應的 API 介接層，即可快速將新平台納入跑者的綁定選項中，而無須大幅重構核心的影像處理或排程系統。

### **5.2 AI 辨識能力的演進：超越號碼布的身分確認技術**

當前系統以號碼布 OCR 為主要身分匹配方式（見 §3.2 Face Re-ID Fallback）。然而，冬季賽事中跑者常穿著外套遮擋號碼布，或號碼布因汗漬/泥濘嚴重污損，導致 OCR 完全失效。未來系統可擴充以下尚未實作之身分確認技術：

- **Person Re-ID（身軀重識別）：** 以深度神經網路比對跑者身形特徵（如體型、步態、穿著顏色），不依賴號碼布或人臉。適用於號碼布遮蔽且用戶未上傳自拍照之情境。需額外收集 training dataset 涵蓋多種氣候與姿態。
- **Gait Recognition（步態辨識）：** 透過影片分析跑者獨特步態特徵，進一步提升集團通過時的身分確認準確率。
- **多模態融合（Multi-modal Fusion）升級：** 將 OCR 置信度、人臉比對分數、Person Re-ID 分數以加权评分模型（Weighted Score Fusion）進行綜合排序，而非現行之串聯式降級（Cascade）判斷。
- **自監督學習（Self-supervised Learning）：** 以賽事內大量未標記照片自動學習跑者視覺特徵 Embedding，減少對大規模標註資料集的依賴。

### **5.3 多運動類別擴充：三鐵、自行車與游泳賽事支援**

本系統架構之 OCR 與物件偵測管線可擴充至三鐵（鐵人三項）、自行車賽與游泳賽事。需注意差異化訓練資料：自行車賽中參賽者之號碼布常置於頭盔或車身而非胸前，OCR 目標區域與姿態各異；游泳賽事中參賽者於出發與轉換區拍攝，衣物與號碼布形式亦不同於路跑。模型微調須收集中、長距離鐵人三項賽事之公開照片集（如 Ironman、Challenge Family 系列賽）與 UCI 自行車賽影像，建立獨立之領域適配模型權重，再於推論時依據賽事類型動態切換對應模型。

### **5.4 從靜態影像邁向動態短影音（Reels / Shorts）的全自動生成**

- **Instagram Reels API：** 單支 Reels 最長 90 秒至 3 分鐘（依 IG 帳號類型而異，Business / Creator 帳號上限較高）；檔案上限約 1 GB；解析度 1080×1920（9:16）為最佳。
- **Threads Video API：** 影片最長 5 分鐘；檔案上限 1 GB（單次上傳）；解析度 1080×1920（9:16）為標準。

預設影片生成長度為 15–30 秒（對應「15 秒短影音」之設計目標），由 `EventFeatureConfig.rendering.videoMaxDurationSec` 設定控制（預設 30，可由後台調整至 90）。當實際影片超出平台上限時，系統自動將影片截斷為多段 Clip 並依序發布，避免單支影片推播失敗。所有影片推播均遵守 §4.3 之 PDPA 跨境傳輸規範。這項基於短影音的自動化升級，將成為進一步提升賽事曝光量與網路互動率的強大新引擎。

**§5.4 與既有 photo pipeline 相容性策略（v1.5 補充）：**

短影音擴充採「**水平擴充 + 共用 publish-queue**」設計，**不新增獨立 Lambda**，避免推播引擎分裂：

| 元件 | 既有 photo pipeline | 短影音擴充（未來） | 共用程度 |
|:----|:----|:----|:----|
| **攝入端** | 攝影師高頻上傳 → S3 Event | 同樣由 S3 Event 觸發，依檔案副檔名分流（`.jpg/.png` → photo; `.mp4/.mov` → video） | 共用 S3 + SQS Event Source |
| **AI 推論** | Photo Processing Lambda (YOLOv8 + OCR + 美學評分) | 新增 Video Processing Lambda (FFmpeg Lambda Layer + FFmpeg frame extraction + 美學評分) | 兩者輸出相同 `NormalizedPublishTask` 介面，**獨立 Lambda 各自處理** |
| **推播佇列** | `sqs://{eventId}-publish-queue` | **共用同一 queue**，由 message 內 `mediaType: 'image' \| 'video'` 欄位分流 | 完全共用，避免雙維護 |
| **推播引擎** | Publish Lambda + Social Adapter | 同樣 Publish Lambda 處理；Adapter 介面不變（已支援 video，詳見 §2.8） | 完全共用 |
| **S3 儲存** | `{env}-imarathon-photos/{eventId}/` | 同前綴，僅多 video 副檔名檔案；影片 metadata 寫入 S3 Object Metadata | 共用 bucket，僅靠 metadata 區分 |

**關鍵決策：** 不新增 `video-publish-queue` 也不新增 `Video Publish Lambda` —— 推播引擎層完全共用。AI 推論層**獨立**（Photo vs Video 兩 Lambda），因為計算特性差異大（影片需 FFmpeg 抽幀 + 多幀合併評分）。此設計維持 §2.8 Adapter Pattern 與 §2.10 Inference Adapter 之抽象層一致性。

**功能開關：** §2.11 須新增 `VIDEO_REELS_AUTO_GENERATE` Feature Flag（預設關閉），控制是否啟用短影音自動生成與推播；啟用時自動啟用相關之上傳分流邏輯。

**實作階段：**
- v2.0：§5.4 完整實作（影片生成 + 推播）
- v2.1：Person Re-ID / Gait Recognition（§5.2）
- v2.2：多運動類別（§5.3）

綜上所述，本需求規格與系統架構書定義了一個具備高度前瞻性與技術可行性的數位藍圖。透過嚴謹的無伺服器雲端基礎建設、智慧化的邊緣與雲端協同人工智慧推論，以及緊扣社群脈動的 API 深度整合，此系統不僅將徹底解決傳統賽事攝影的痛點，更將為未來的運動科技服務立下全新的數位轉型標竿。

#### **引用的著作**

**引文編號使用說明（v1.5 補充）：** 本 SPEC 之引文編號（如「1」「13」「27」）對應至下方清單編號，於正文內以**阿拉伯數字**標示於相關論述後（例如「…傳統 OCR 辨識率大幅降低1」）。§3.1 等長段落引文編號較密集，可能連續出現多個編號（如「…詳閱 PDPA 規範24」），此屬刻意保留之引用密度，便於讀者追溯原文出處。**不**重新編排為 footnote 形式（如 `[24]`），以維持 markdown 純文字可讀性與 git diff 友善性。

1. How to Deliver Marathon Photos With Facial Recognition App \- SnapSeek, [https://snapseek.app/blog/marathon-face-recognition-photos](https://snapseek.app/blog/marathon-face-recognition-photos)  
2. Race AI : A Deep Learning Approach to Marathon Bib Detection and Recognition | Request PDF \- ResearchGate, [https://www.researchgate.net/publication/395041941\_Race\_AI\_A\_Deep\_Learning\_Approach\_to\_Marathon\_Bib\_Detection\_and\_Recognition](https://www.researchgate.net/publication/395041941_Race_AI_A_Deep_Learning_Approach_to_Marathon_Bib_Detection_and_Recognition)  
3. Get Started with Amazon S3 Event Driven Design Patterns | AWS Architecture Blog, [https://aws.amazon.com/blogs/architecture/get-started-with-amazon-s3-event-driven-design-patterns/](https://aws.amazon.com/blogs/architecture/get-started-with-amazon-s3-event-driven-design-patterns/)  
4. (PDF) RFID data processing in a real-time monitoring system for marathon \- ResearchGate, [https://www.researchgate.net/publication/307816312\_RFID\_data\_processing\_in\_a\_real-time\_monitoring\_system\_for\_marathon](https://www.researchgate.net/publication/307816312_RFID_data_processing_in_a_real-time_monitoring_system_for_marathon)  
5. Timing System for Running | Chip Timing \- MYLAPS, [https://mylaps.com/active-sports/sports/running/](https://mylaps.com/active-sports/sports/running/)  
6. Designing Race Timing Applications Using UHF RFID Technology \- Jadak, [https://www.jadaktech.com/wp-content/uploads/2022/08/Designing-Race-Timing-Applications-Using-UHF-RFID-Technology-1.pdf](https://www.jadaktech.com/wp-content/uploads/2022/08/Designing-Race-Timing-Applications-Using-UHF-RFID-Technology-1.pdf)  
7. Other timing system | RACEMAP \- Getting started, [https://docs.racemap.com/prediction/timekeeping/other-timingsystem](https://docs.racemap.com/prediction/timekeeping/other-timingsystem)  
8. Designing a Cost-Efficient Parallel Data Pipeline on AWS Using Lambda and SQS, [https://dev.to/aws-builders/designing-a-cost-efficient-parallel-data-pipeline-on-aws-using-lambda-and-sqs-2bpo](https://dev.to/aws-builders/designing-a-cost-efficient-parallel-data-pipeline-on-aws-using-lambda-and-sqs-2bpo)  
9. Automate Marathon Bib Number Recognition with Computer Vision \- Roboflow Blog, [https://blog.roboflow.com/automated-marathon-bib-recognition/](https://blog.roboflow.com/automated-marathon-bib-recognition/)  
10. nodejs-sharp-lambda-layer \- AWS Serverless Application Repository, [https://serverlessrepo.aws.amazon.com/applications/us-east-1/987481058235/nodejs-sharp-lambda-layer](https://serverlessrepo.aws.amazon.com/applications/us-east-1/987481058235/nodejs-sharp-lambda-layer)  
11. AWS Knowledge Series: Serverless Image Processing With S3 and AWS Lambda | by Sanjay Dandekar, [https://sanjay-dandekar.medium.com/aws-knowledge-series-serverless-image-processing-with-s3-and-aws-lambda-ad6ad08f829b](https://sanjay-dandekar.medium.com/aws-knowledge-series-serverless-image-processing-with-s3-and-aws-lambda-ad6ad08f829b)  
12. Threads Publishing \- Emplifi Documentation Center, [https://docs.emplifi.io/platform/latest/home/threads-publishing](https://docs.emplifi.io/platform/latest/home/threads-publishing)  
13. Post to Instagram via API: Guide (2026) \- Postproxy, [https://postproxy.dev/blog/post-to-instagram-via-api/](https://postproxy.dev/blog/post-to-instagram-via-api/)  
14. Message types | LINE Developers, [https://developers.line.biz/en/docs/messaging-api/message-types/](https://developers.line.biz/en/docs/messaging-api/message-types/)  
15. Building a Production-Ready Serverless Image Processing Pipeline on AWS \- Medium, [https://medium.com/@ikbenezer/building-a-production-ready-serverless-image-processing-pipeline-on-aws-6288c01adc48](https://medium.com/@ikbenezer/building-a-production-ready-serverless-image-processing-pipeline-on-aws-6288c01adc48)  
16. Introducing maximum concurrency of AWS Lambda functions when using Amazon SQS as an event source, [https://aws.amazon.com/blogs/compute/introducing-maximum-concurrency-of-aws-lambda-functions-when-using-amazon-sqs-as-an-event-source/](https://aws.amazon.com/blogs/compute/introducing-maximum-concurrency-of-aws-lambda-functions-when-using-amazon-sqs-as-an-event-source/)  
17. Configuring scaling behavior for SQS event source mappings \- AWS Lambda, [https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-scaling.html](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-scaling.html)  
18. How to Build an Image Processing Pipeline on AWS with Lambda \- OneUptime, [https://oneuptime.com/blog/post/2026-02-12-build-image-processing-pipeline-on-aws-with-lambda/view](https://oneuptime.com/blog/post/2026-02-12-build-image-processing-pipeline-on-aws-with-lambda/view)  
19. Resizing Images and Adding Watermarks with AWS Lambda \- Wojciech Lepczyński, [https://lepczynski.it/en/aws\_en/resizing-images-and-adding-watermarks-with-aws-lambda/](https://lepczynski.it/en/aws_en/resizing-images-and-adding-watermarks-with-aws-lambda/)  
20. How to Use Lambda for PDF Generation \- OneUptime, [https://oneuptime.com/blog/post/2026-02-12-use-lambda-for-pdf-generation/view](https://oneuptime.com/blog/post/2026-02-12-use-lambda-for-pdf-generation/view)  
21. Instagram Graph API: Overview, Content Publishing, Limitations, and Re \- Datkira's Blog, [https://datkira.hashnode.dev/instagram-graph-api-overview-content-publishing-limitations-and-references-to-do-quickly](https://datkira.hashnode.dev/instagram-graph-api-overview-content-publishing-limitations-and-references-to-do-quickly)  
22. Send messages | LINE Developers, [https://developers.line.biz/en/docs/messaging-api/sending-messages/](https://developers.line.biz/en/docs/messaging-api/sending-messages/)  
23. Line Message API : push message & reply | n8n workflow template, [https://n8n.io/workflows/2733-line-message-api-push-message-and-reply/](https://n8n.io/workflows/2733-line-message-api-push-message-and-reply/)  
24. Data protection laws in Taiwan, [https://www.dlapiperdataprotection.com/?t=law\&c=TW](https://www.dlapiperdataprotection.com/?t=law&c=TW)  
25. Personal Data Protection \- Veolia Taiwan, [https://www.veolia.tw/en/personal-data-protection](https://www.veolia.tw/en/personal-data-protection)  
26. Personal Data Protection Act \- Article Content \- Laws & Regulations Database of The Republic of China (Taiwan), [https://ntnurec.ntnu.edu.tw/xhr/archive/download?file=66b2da2406ee4bf2ed03e540](https://ntnurec.ntnu.edu.tw/xhr/archive/download?file=66b2da2406ee4bf2ed03e540)  
27. Taiwan: Data Protection & Cybersecurity – Country Comparative Guides \- Legal 500, [https://www.legal500.com/guides/chapter/taiwan-data-protection-cybersecurity/](https://www.legal500.com/guides/chapter/taiwan-data-protection-cybersecurity/)  
28. Preparatory Office of the Personal Data Protection Commission-Personal Data Protection Act and Interpretations, [https://www.pdpc.gov.tw/en/News\_Html/165/](https://www.pdpc.gov.tw/en/News_Html/165/)  
29. Marathon Bib Number Recognition using Deep Learning | Semantic Scholar, [https://www.semanticscholar.org/paper/Marathon-Bib-Number-Recognition-using-Deep-Learning-Apap-Seychell/ff06ad86139b8056e34440f276e5d1dbf9f9daaa](https://www.semanticscholar.org/paper/Marathon-Bib-Number-Recognition-using-Deep-Learning-Apap-Seychell/ff06ad86139b8056e34440f276e5d1dbf9f9daaa)  
30. UniqueData/race-numbers-detection-and-ocr · Datasets at Hugging Face, [https://huggingface.co/datasets/UniqueData/race-numbers-detection-and-ocr](https://huggingface.co/datasets/UniqueData/race-numbers-detection-and-ocr)  
31. Marathon Detection :: Portfolio, [https://cv.faraphel.fr/en/posts/university-projects-marathon/](https://cv.faraphel.fr/en/posts/university-projects-marathon/)  
32. Error using .overlayWith() on AWS Lambda in node.js sharp \- Stack Overflow, [https://stackoverflow.com/questions/52111405/error-using-overlaywith-on-aws-lambda-in-node-js-sharp](https://stackoverflow.com/questions/52111405/error-using-overlaywith-on-aws-lambda-in-node-js-sharp)  
33. Threads Post Size: 1080×1350 (4:5) \[Updated January 2026\] \- PostFast, [https://postfa.st/sizes/threads/posts](https://postfa.st/sizes/threads/posts)  
34. Threads Photo and Video format requirements \- Help Center · Swat.io, [https://help.swat.io/en/articles/11994597-threads-photo-and-video-format-requirements](https://help.swat.io/en/articles/11994597-threads-photo-and-video-format-requirements)  
35. Threads Post, Image & Video Sizes: Complete Guide \- PostEverywhere's AI, [https://posteverywhere.ai/blog/threads-image-sizes](https://posteverywhere.ai/blog/threads-image-sizes)  
36. Instagram Graph API 2026: Dev Questions Meta's Docs Leave Open \- Zernio, [https://zernio.com/blog/instagram-graph-api](https://zernio.com/blog/instagram-graph-api)  
37. Messaging API reference \- LINE Developers, [https://developers.line.biz/en/reference/messaging-api/](https://developers.line.biz/en/reference/messaging-api/)  
38. 活用Messaging API 打造客製化的官方帳號（應用篇）, [https://tw.linebiz.com/e-learning/Messaging-API-application/](https://tw.linebiz.com/e-learning/Messaging-API-application/)  
39. How To Generate PDFs Serverlessly With AWS Lambda and Headless Chromium, [https://apitemplate.io/blog/html-to-pdf-aws-lambda-serverless-guide/](https://apitemplate.io/blog/html-to-pdf-aws-lambda-serverless-guide/)  
40. Generate a PDF in AWS Lambda with NodeJS and Puppeteer \- DEV Community, [https://dev.to/akirautio/generate-a-pdf-in-aws-lambda-with-nodejs-and-puppeteer-2b93](https://dev.to/akirautio/generate-a-pdf-in-aws-lambda-with-nodejs-and-puppeteer-2b93)  
41. Generate HTML as PDF using Next.js & Puppeteer running on Serverless (Vercel/AWS Lambda) | by Martin Danielson | Medium, [https://medium.com/@martin\_danielson/generate-html-as-pdf-using-next-js-puppeteer-running-on-serverless-vercel-aws-lambda-ed3464f7a9b7](https://medium.com/@martin_danielson/generate-html-as-pdf-using-next-js-puppeteer-running-on-serverless-vercel-aws-lambda-ed3464f7a9b7)  
42. How to Scale HTML to PDF with Serverless and Puppeteer \- pdf noodle, [https://pdfnoodle.com/blog/how-to-scale-html-to-pdf-with-serverless-and-puppeteer](https://pdfnoodle.com/blog/how-to-scale-html-to-pdf-with-serverless-and-puppeteer)  
43. Instagram Graph API: Complete Developer Guide for 2026 \- Elfsight, [https://elfsight.com/blog/instagram-graph-api-complete-developer-guide-for-2026/](https://elfsight.com/blog/instagram-graph-api-complete-developer-guide-for-2026/)  
44. Video Upload Limits | YouTube, TikTok, Instagram, Facebook, LinkedIn & More \- Postly, [https://postly.ai/resources/media-upload-limits](https://postly.ai/resources/media-upload-limits)