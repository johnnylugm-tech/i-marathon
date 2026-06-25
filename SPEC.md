# **馬拉松賽事與社群互動連結系統：需求規格與系統架構書**

## **1\. 專案概述 (Project Overview) & 核心價值**

在大型路跑與馬拉松賽事中，參賽者的數位體驗已成為賽事品牌經營的核心要素。然而，傳統馬拉松賽事的影像紀錄與分發流程，長期面臨效率低落與使用者體驗不佳的技術瓶頸。

就市場現況而言，目前全球已存在若干基於 AI 技術之賽事攝影服務（如 RaceTagger、Photohawk、Roboflow 等）。然而，現有方案多止於「AI 標記輔助攝影師整理」，均未實現將照片全自動推送至跑者個人社群帳號之最後一哩路。本系統之核心差異化定位在於：**從攝影到社群平台的最後一哩自動推播層**，此功能在全球及台灣市場中均屬缺口。過去數十年來，賽事攝影師多半依賴光學字元辨識（Optical Character Recognition, OCR）技術掃描跑者號碼布，或者純粹仰賴事後耗時的人工標記1。在真實的賽事環境中，號碼布經常因為跑者的肢體動作而彎折，或是被雨衣、外套遮蔽，甚至被汗水與泥漿覆蓋，這些現實環境干擾使得傳統 OCR 的辨識率大幅降低1。當辨識失敗時，成千上萬張未經整理的原始照片往往被直接上傳至雲端硬碟，迫使跑者必須在海量圖庫中進行漫長且令人沮喪的檢索，此現象被業界稱為「圖庫疲勞」（Gallery Fatigue）1。這不僅導致跑者錯失了在完賽當下即時分享成就的黃金時機，也讓賽事主辦單位流失了寶貴的社群曝光動能。  
本專案旨在開發名為「馬拉松賽事與社群互動連結系統」的創新數位服務，透過深度整合邊緣運算、雲端無伺服器架構（Serverless Architecture）、深度學習物件偵測與文字萃取模型，以及主流社群平台的應用程式介面（API），徹底翻轉傳統賽事影像的交付與傳播模式。系統的核心價值在於提供極致的「即時滿足感」（Instant Gratification），將分類與標記的沉重負擔從人類轉移至機器與演算法1。當跑者通過攝影點位後，系統將以全自動化的串流工作流程擷取影像、進行雲端人工智慧解析、動態套用賽事專屬視覺化數據（如配速、完賽時間、贊助商相框），並在極低的延遲內，將美化後的專屬照片推播至跑者事先授權的個人社群平台（包含 Instagram、Threads、Facebook 與 LINE）。  
此全自動化的數位轉型不僅能極大化跑者的參與感與專屬尊榮感，更能為賽事主辦單位與贊助商創造指數型的社群曝光度。透過跑者社群網絡的病毒式傳播，賽事品牌的數位足跡將以自動化推播的方式達到最大化。同時，系統於賽後自動生成的「個人專屬完賽報紙」，深度結合了無線射頻辨識（RFID）晶片計時數據、精選影像與社群互動指標，進一步將參賽的瞬間感動昇華為具備長久紀念價值的客製化數位資產。

## **2\. 系統架構與資料流 (System Architecture & Data Flow)**

為確保系統在數萬名跑者同時參賽、數十位專業攝影師密集上傳高畫質影像的極端負載下，仍能維持毫秒至秒級的低延遲與系統高可用性，本系統採用基於雲端原生（Cloud-Native）的事件驅動無伺服器架構（Event-Driven Serverless Architecture）2。整體的拓樸結構被精密劃分為四個主要層級：資料獲取與攝入層、非同步串流緩衝與處理層、人工智慧推論與渲染層，以及社群發布與展現層。  
### **2.1 資料獲取與攝入層**

在資料獲取層中，賽道沿線的專業攝影設備透過 5G 路由器或專線網路，將高解析度影像即時且持續地推送到雲端物件儲存空間（如 Amazon S3）3。本系統採用賽道定點攝影模式：賽前公告固定攝影點位，跑者預期在已知位置獲得拍攝服務。賽事主辦單位將提供攝影師完善硬體設施（5G 熱點、供電、遮陽/遮雨帳篷），確保上傳穩定性3。同步地，位於起終點與各個分段檢查點的 RFID 計時系統（例如 MyLaps 系統或搭載 UHF 被動式 RFID 天線的讀取器），會捕捉跑者鞋面或號碼布上的晶片數據4。本系統透過「外部計時系統整合抽象層」之 Adapter Pattern 接收資料（詳見 2.7 章節），以統一的內部資料模型隔離各系統 API 差異，無論計時系統供應商為 MyLaps、ChampionChip、RaceEntry 或其他，均能無差異地流入下游處理管線。  
### **2.2 非同步串流緩衝與流量調節層**

當影像檔案寫入 S3 儲存體時，會立即觸發 S3 事件通知機制，將該影像的處理任務物件推入 Amazon SQS（Simple Queue Service）訊息佇列中3。引入 SQS 作為非同步與佇列點對點處理（Queued point-to-point processing）的關鍵在於解耦與削峰填谷；它允許下游的無伺服器運算單元以自身的最佳節奏擷取任務，避免瞬間的巨量影像上傳壓垮後端運算資源與資料庫連線3。  
### **2.3 AI 推論與影像渲染層**

接下來，SQS 將觸發具備高度彈性擴展能力的運算服務（如 AWS Lambda），啟動核心的影像處理與人工智慧管線。Lambda 函數會呼叫預先訓練好的邊緣或雲端物件偵測模型（如 YOLOv8 或 RF-DETR），精確進行跑者身軀與號碼布的邊界框（Bounding Box）定位，隨後交由 OCR 引擎提取字元2。辨識成功後，影像將傳遞至基於 Node.js 的 Sharp 影像處理模組，進行浮水印、濾鏡與賽事數據的動態疊加渲染10。  
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

**KMS 金鑰管理政策：**每次賽事部署時產生新的 CMK（Customer Master Key），用於該賽事之 Token 加密；CMK 啟用日 + 90 天自動排程刪除，確保歷史 Token 無法被新金鑰解密而須重新授權。金鑰存取由 IAM Policy 控制，Lambda 執行角色僅有 `kms:Decrypt` 權限，無 `kms:GenerateDataKey` 以外之金鑰管理權限。

**S3 儲存桶命名策略：**所有賽事共用同一主 bucket（`{env}-imarathon-photos`），以 `eventId/` 前綴隔離不同賽事資料。Face Re-ID 自拍照使用獨立隔離 bucket（`{env}-imarathon-biometric`），不得與一般照片bucket 混用。Bucket Policy 嚴禁跨前綴讀取；Lambda 執行角色僅有對應 eventId 前綴之 `s3:GetObject` / `s3:PutObject` 權限。

**DynamoDB 全域二級索引（GSI）設計：**

| 索引名稱 | GSI Key | 查詢場景 |
| :---- | :---- | :---- |
| `runner-platform-index` | PK: `runnerId`, SK: `platform` | 查詢特定跑者在各平台之 Token 狀態 |
| `dlq-status-index` | PK: `eventId`, SK: `status#createdAt` | 查詢特定賽事之 DLQ 任務（依狀態排序） |
| `task-status-index` | PK: `eventId`, SK: `status#updatedAt` | 查詢特定賽事之處理任務（依狀態+時間排序） |
| `runner-event-index` | PK: `eventId`, SK: `bibNumber` | 查詢特定賽事之跑者資料（by bibNumber） |

所有 GSI 寫入時維持 Strong Consistency，讀取支援最終一致性（應用於監控儀表板等非即時場景）。

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
| **小紅書** | 小紅書 API | OAuth 2.0 (企業號) | PNG/JPG, ≤ 10MB | Note API | ✅ P2（未來擴充） |

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

#### **新增平台支援流程**

未來新增社群平台支援時，開發流程為：

1. 於 `social-adapters/` 目錄下新增 `{platform}.adapter.ts`，實作 `ISocialPlatformAdapter` 介面
2. 向工廠函數 `SocialPlatformFactory.createAdapter` 新增 `platform` 映射
3. 向後台管理系統之「支援平台」設定頁面新增該平台開關（預設關閉）
4. 賽事主辦單位於後台填入新平台之 App Credentials（App ID / Secret / OAuth Redirect URI）
5. 撰寫 Adapter 單元測試（Mock 平台 API）與 E2E 測試（使用平台測試帳號）

此流程為純水平擴充，核心推播引擎、Lambda 函數與資料庫 Schema 完全不受影響。

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

為同時滿足 SLA 延遲目標與成本控制，系統支援「多引擎級聯降級」策略：

1. **第一階段（≤ 30 秒）：** 嘗試本地 YOLOv8 + Tesseract OCR（免費、低延遲）
2. **第一階段失敗（置信度 < 0.5）或超時：** 切換至 Gemini Flash API（低成本雲端 OCR）
3. **第二階段仍失敗（置信度 < 0.7）：** 升級至 Claude 3.5 Sonnet（高精度推理）
4. **最終失敗：** 寫入 DLQ「AI 推論耗盡」，通知人工處理

每個階段的 `maxRetries`、`timeoutMs`、`fallbackOnConfidence` 皆由 `eventConfig.inferenceStrategy` 參數化控制。

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
    confidenceThreshold: number;           // OCR 置信度閾值（預設 0.7）
    faceReIdConfidenceThreshold: number;   // Face Re-ID 置信度閾值（預設 0.75）
    cascadeFallback: CascadeConfig;        // 級聯降級設定
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

## **3\. 功能性需求 (Functional Requirements)**

本系統的業務邏輯高度複雜且具備多樣性，涵蓋從賽前的合規註冊與授權、賽中極高頻率的 AI 運算，到賽果客製化媒體產出的完整生命週期。以下針對五大核心功能進行深度的架構與實作需求解析。

### **3.1 賽前註冊系統：動態 API 模組設計與嚴謹合規授權**

賽前註冊系統是建立跑者數位身分與後續自動推播功能的基石。跑者必須能夠透過響應式網頁介面（RWD）輸入其專屬的參賽號碼布編號，並進行主流社群帳號的深度綁定。

註冊階段亦須同時揭露「服務限制與不可抗力因素告知」。本系統雖以全自動化為目標，惟仍存在以下可能導致照片無法順利交付之情境，須於賽前註冊頁面以顯著方式告知跑者：天候因素（暴雨、濃霧、高溫導致設備或跑者受限）、人群過度密集導致號碼布遭遮蔽、號碼布污損或字樣模糊（泥濘、汗漬、摺皺）、號碼布遭外套或背心遮蓋、拍攝角度過度傾斜（超過 75°）、突發性硬體故障或 5G 網路瞬斷，以及其他非可控之環境因素。若因上述因素導致 OCR 辨識失敗，系統將自動降級至圖庫模式，跑者仍可透過傳統關鍵字搜尋機制於線上圖庫中自行查找照片，但不再享有自動化社群推播服務。為應對未來不斷演進的社群平台生態與 API 變更，系統架構必須採用模組化的轉接器設計模式（Adapter Pattern）與標準的開放授權（OAuth 2.0）框架，將各平台的驗證流程予以抽象化。  
在社群 API 模組設計上，必須針對各平台的特殊要求進行定製化與嚴密的防錯處理。以 Instagram 為例，其 API 整合的門檻極高。根據 Meta Graph API 的規範，內容發布功能（Content Publishing）僅支援 Instagram 的「商業帳號（Business）」或「創作者帳號（Creator）」，完全排除了標準的個人帳號13。此外，該專業帳號必須明確連結至一個 Facebook 粉絲專頁，否則無法透過 API 進行發布13。因此，系統在註冊階段的 UX/UI 設計中，必須主動偵測使用者的帳號屬性；若為一般個人帳號，系統需提供清晰且友善的圖文引導流程，協助跑者在 Instagram 設定中切換帳號屬性並完成粉絲專頁連結，方能取得 instagram\_business\_content\_publish 與 instagram\_business\_basic 權限13。  
在 LINE 平台的整合上，系統將利用 Messaging API 的推播訊息（Push Message）功能。跑者在註冊時需加入賽事專屬的 LINE 官方帳號為好友，系統藉此取得該跑者的唯一使用者 ID（User ID）22。此 ID 將作為後續點對點推播影像訊息（Image Message）或客製化圖文選單（Imagemap Message）的目標位址14。  
此外，遵循台灣《個人資料保護法》（PDPA），賽前註冊系統必須具備極為嚴謹的隱私權同意與告知機制。系統收集之資料包含跑者的真實姓名、精確定位資料、影像，甚至潛在的生理特徵（如後續啟用的人臉辨識），這些均屬於受高度管制的個人資料甚至特種個人資料範圍24。系統不得採用預設勾選或將同意條款包裹於一般賽事報名條款中（Bundled Consent）27。必須以獨立的隱私聲明頁面，明確告知蒐集目的、資料利用範圍、傳輸對象以及跑者依法可行使之存取、更正與刪除權利25。為了負起法定的舉證責任（Burden of Proof），系統資料庫需精確記錄每一次同意操作的時間戳記、IP 位址與同意條款版本，確保日後受主管機關稽核時，具備無懈可擊的數位證據力27。

### **3.2 即時影像處理與辨識流：邊緣到雲端的 AI 協同與極速渲染**

賽事期間的影像處理要求極致的低延遲與極高的準確率。當攝影師按下快門後，影像將透過內建 FTP 或 API 上傳功能的相機，搭配 5G 網路直接推送到指定的雲端 S3 儲存貯列。此舉將觸發 SQS 佇列訊息，進而喚醒 AWS Lambda 函數進行非同步的初步處理8。  
AI 辨識流採用多階段的深度學習管線，並透過 AI/LLM 推論抽象層（見 2.10 章節）動態選擇推論引擎2。本系統首先調用 YOLO 架構（如 YOLOv8 或 RF-DETR）的物件偵測模型進行號碼布定位2。該模型經過包含多種光影條件、跑者姿態與複雜背景（In-the-wild benchmark datasets）的專屬賽事資料集進行微調（Fine-tuning），能迅速在百萬像素的影像中框選出「跑者身軀」，接著再進一步於身軀範圍內定位出「號碼布」的精確邊界框（Bounding Box）29。定位完成後，系統會對號碼布區域進行幾何校正與形態學操作（Morphological Operations），將傾斜或皺褶的區域拉平，最後再將處理後的高對比區塊送入 OCR 引擎提取精確的數字序列2。此種結合特徵金字塔網路（Feature Pyramid Network）與卷積注意力機制（Convolutional Block Attention Module）的方法，能有效排除環境雜訊，將號碼布辨識準確率提升至高達 91.6% 以上的商業標準2。

當 OCR 引擎回傳之置信度分數（Confidence Score）低於预设阈值時（例如低於 0.7），系統將自動觸發**號碼布 OCR 失敗的容錯回退機制（Fallback Mechanism）**：以 YOLOv8/RF-DETR 偵測並裁切跑者臉部區塊，调用 SnapSeek 架構之人脸比对引擎，與跑者於註冊時自願上傳之清晰自拍照进行空间几何特征比对（需跑者於註冊時另行授權），以多模態融合（Multi-modal）方式提升恶劣环境下的身份对接成功率。若 Face Re-ID 置信度同樣低於阈值，該照片標記為「待人工確認」，寫入 DLQ 由賽事方進行後續處理。  
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

#### **美化完成後：推播 SQS 訊息格式（Publish Lambda 輸入）**

Photo Processing Lambda 完成美化處理後，將 `NormalizedPublishTask` JSON 寫入 `publish-queue`（`sqs://{eventId}-publish-queue`），由推播引擎 Lambda 消費。訊息格式見 §2.8 `NormalizedPublishTask` 介面定義（line 221）。

### **3.3 自動社群推播：跨平台的動態適配與流量控制機制**

當跑者的專屬美化照片生成後，系統必須將其即時推送至跑者授權的社群平台。本系統透過 2.8 節所定義之「社群平台整合抽象層」處理所有平台差異，核心推播引擎完全不感知平台專屬邏輯。

各平台之 API 規範、OAuth 流程、媒體格式限制、Rate Limit 與推播方式，請參見 §2.8「社群平台整合抽象層」之 Adapter 實作矩陣與 ISocialPlatformAdapter 介面定義。

Token 生命週期管理與刷新失敗處理，請參見 §3.6。

### **3.4 完賽個人專屬報紙（客製化）：無伺服器架構下的動態 PDF 生成**

賽事結束後，系統會彙整每位跑者的多維度資料，產生一份極具紀念價值且可供列印的客製化「個人專屬報紙」。這份數位報紙的資料來源涵蓋廣泛：包含自計時系統 API 傳回的 JSON 數據（包含各檢查點的 timingId、chipId、timestamp 與最終完賽晶片時間）4；系統在賽事中捕捉到並辨識成功的精選衝線照片；以及跑者在社群平台上互動所產生的數據反饋（如點讚數、特定活動 Hashtag 的擴散程度等）。  
在輸出格式與呈現邏輯上，此專屬報紙被定義為一份高解析度、相容於 A4 實體列印的 PDF 文件。其版面設計模擬傳統頭版新聞，主視覺為跑者的高畫質大圖，並搭配動態生成的標題（例如：「破風前行！\[跑者姓名\] 以 \[完賽時間\] 征服 \[賽事名稱\]」）。版面側邊以視覺化的圖表（如折線圖）呈現各分段區間的配速曲線，底部區塊則拼貼社群互動亮點與贊助商致敬欄位。  
為實現此高度動態且排版複雜的文件生成，系統採用無伺服器環境中的 Puppeteer-core 搭配無頭瀏覽器（Headless Chromium）解決方案，捨棄了在排版彈性上較差的 PDFKit20。系統後端首先將匯集的 JSON 數據注入預先設計好的 HTML5/CSS3 樣式模板中（使用如 Pug 或 Jinja2 模板引擎）39。隨後，AWS Lambda 函數會啟動專為 Serverless 環境壓縮的輕量化 Chromium 執行檔（如 @sparticuz/chromium-min 或利用 AWS Lambda Layer 載入的 chrome-aws-lambda）20。  
在記憶體中渲染該 HTML 頁面時，Chromium 會確保所有特殊字體（如粗體報紙標題字型）、複雜的 CSS 網格排版與外部高解析度圖片皆完美載入。為達到最佳列印效果，開發團隊將在樣式表中宣告 @media print { @page { size: A4 portrait; margin: 0; } }，確保產出的 PDF 不留白邊41。生成的 PDF 檔案將匯出為 Buffer 並自動寫入 Amazon S3 存放，隨後透過 LINE 訊息（主要 delivery 方式）將預先簽名的下載連結遞送給跑者，完成賽事體驗的最後一哩路。若主辦單位已啟用 email 通知（Feature Flag `NEWSPAPER_EMAIL`），則同時以 SMTP 寄送下載連結至跑者 email；email 基礎設施（Sender Policy、MTA、SES）須由主辦單位另行提供20。

### **3.5 系統 API 設計**

本系統之 API 分為「外部公開 API」、「內部系統 API」與「管理 API」三類。外部公開 API 以 Amazon Cognito User Pool 簽發之 JWT Bearer Token 進行身分驗證，經 API Gateway 啟用 Rate Limiting；內部系統 API 以 API Key + IP 白名單進行認證；管理 API 以 Cognito User Pool + RBAC 角色驗證。

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

| 端點 | 方法 | 說明 | 頻率限制 |
| :---- | :---- | :---- | :---- |
| `/api/v1/oauth/callback/{platform}` | GET | OAuth 授權回調端點，接收 authorization code 並換取 Token | N/A（由平台觸發） |

**GET /api/v1/oauth/callback/{platform} — OAuth 2.0 Callback 流程：**

所有 OAuth 授權流程（含 LINE Login、Meta Graph API、Threads）均使用 `state` 參數防止 CSRF 攻擊，並攜帶 runner 識別資訊。Callback URL 格式如下：

```
https://api.example.com/api/v1/oauth/callback/{platform}?code={authorization_code}&state={base64url_json}
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

#### **Gallery API Response 格式**

**GET /api/v1/races/{eventId}/gallery?bib={bibNumber} — Response（成功，200）：**
```json
{
  "total": 5,
  "photos": [
    {
      "photoId": "uuid-xxxx",
      "checkpointCode": "KM25",
      "capturedAt": "2026-06-26T05:30:00Z",
      "thumbnailUrl": "https://cdn.example.com/thumb/uuid-xxxx.jpg",
      "downloadUrl": "https://s3.amazonaws.com/...(pre-signed)...",
      "publishedPlatforms": ["line", "instagram"],
      "ocrConfidence": 0.92
    }
  ]
}
```

### **3.6 OAuth Token 生命週期與刷新失敗處理**

OAuth Token（Access Token + Refresh Token）之生命週期管理為系統穩定性的關鍵環節：

**Token 刷新邏輯：**每張 OAuth Token 均攜帶 `expiresAt` 時間戳。Lambda 函數在執行推播前，會先檢查是否已過期或距離過期不足 10 分鐘；若符合條件則先呼叫平台 Refresh Endpoint 取得新 Token，再執行推播。刷新成功後新 Token 寫回 DynamoDB，並更新 `expiresAt`。

**刷新失敗時之降級流程：**

| 失敗原因 | 系統行為 |
| :---- | :---- |
| 用戶主動撤銷授權（401 Unauthorized from platform） | 立即更新 Token 狀態為 `revoked`，**不重試**，寫入事件日誌；如跑者有備援平台（LINE），自動切換至備援推播；無備援則標記「需重新授權」並於完賽報紙中通知 |
| Refresh Token 過期（一般為 30–60 天） | 視同撤銷處理，同上 |
| 平台 API 暫時性錯誤（5xx） | 指數退避重試（最多 3 次，間隔 30s/60s/120s），3 次失敗後寫入 DLQ「Token 刷新重試失敗」 |
| 網路瞬断 | 重試 1 次，失敗即寫入 DLQ |

**重新授權通知：**當 Token 狀態變更為 `needs_reauth` 時，系統透過 LINE Push Message 主動通知跑者：「您的社群授權已過期，請於 48 小時內重新授權以確保收到完賽報紙」。

### **3.7 照片歸戶邏輯（Corner Cases）**

**同一攝影點之連拍叢集（Photo Burst）處理：**當多位跑者在極短時間內相繼通過同一攝影點時，單一 Lambda 實例可能對同一號碼布產生多張照片。系統以「**號碼布 + 拍攝時間窗口（±3 秒）**」作為叢集鍵（Cluster Key），同一叢集內僅選擇「OCR 置信度最高」之一張照片執行推播，其餘寫入跑者個人圖庫供自行下載。此行為由 Feature Flag `PHOTO_BURST_DEDUP` 控制（預設開啟），主辦單位可關閉此功能以對所有照片均執行推播。

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
3. **參賽者名單匯入**：支援 CSV 匯入（欄位：`bibNumber, fullName, email, chipId`），系統自動與計時系統晶片資料比對；名單確認後系統鎖定，後續跑者變更須由主辦單位授權後放行
4. **LINE 官方帳號設定**：填入 LINE 官方帳號之 Channel Access Token，系統驗證 Messaging API 連線；每位跑者完成報到後，系統自動發送 LINE 好友邀請連結
5. **贊助商與相框模板**：上傳 PNG 相框模板（Alpha Channel）、設定浮水印強度、填入贊助商名稱與 Logo
6. **PDF 完賽報紙模板選擇**：從預設版型庫中選擇，或上傳客製化 HTML/CSS 模板（須通過安全審查）
7. **OAuth 平台設定**：填入 Meta App ID/Secret、Threads App Token，系統進行 API 連線測試後啟用

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

## **4\. 非功能性需求 (Non-Functional Requirements)**

非功能性需求定義了系統在極端壓力下的表現與基礎設施的穩健程度。對於一項需要即時處理巨量多媒體數據與個人隱私的賽事系統而言，高併發處理能力、超低延遲的運算架構與堅若磐石的資安防護是不可或缺的三大支柱。

### **4.1 高併發照片上傳與智慧流量塑形 (Traffic Shaping)**

在大型馬拉松賽事中，選手通常呈現集團式移動。當大批跑者同時通過熱門攝影點（如起終點或知名地標）時，多台攝影設備將在極短的時間內上傳數千張高解析度檔案至雲端。這種典型的「突發性流量（Bursty Traffic）」若以同步（Synchronous）方式直接交由後端伺服器處理，極易導致運算資源瞬間耗盡、資料庫連線池崩潰，甚至引發雪崩效應3。  
為應對此挑戰，本系統高度依賴 Amazon SQS 作為核心的流量塑形與緩衝機制。當 S3 接收到新影像時，僅將輕量化的物件元資料（Metadata）非同步發送至 SQS 佇列中3。AWS Lambda 被配置為 SQS 的消費者，利用其強大的擴展性進行影像消化。雖然 AWS Lambda 理論上可以每分鐘增加 60 個實例，瞬間擴展至高達 1,000 個以上的併發執行環境，但這種無節制的擴充將帶來毀滅性的後果——瞬間發出的大量請求會觸發下游社群平台 API 之速率限制（Meta Graph API 以 Instagram Business Account 維度計算，24 小時內最多 25 篇貼文）與關聯式資料庫的速率限制（Rate Limits）16。  
因此，系統架構師必須在 SQS 與 Lambda 之間的事件源映射（Event Source Mapping）中設定「最大併發數量（Maximum Concurrency）」16。藉由嚴格控制同時啟動的 Lambda 實例上限（例如限制在 50 個併發），系統能確保影像處理的吞吐量穩定維持在下游 API 可承受的安全閾值內。未及時處理的影像則安穩地保留在 SQS 佇列中等待消化。此外，搭配死信佇列（Dead-Letter Queue, DLQ）設計，即使遇到損毀的影像或 OCR 演算法意外崩潰，該問題任務也會被隔離至 DLQ 中進行後續的人工除錯或重新驅動，確保絕不遺漏任何一張跑者的珍貴影像8。

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
在雲端無伺服器架構中，最棘手的問題是「冷啟動（Cold Start）」延遲。當系統需要載入龐大的 AI 模型參數，或解壓縮高達數十 MB 的 Headless Chromium 執行檔與 Node.js 依賴庫時，Lambda 函數的初始化動輒需要耗費 5 到 15 秒39。為徹底克服此痛點，系統將針對負責 AI 辨識與 PDF 生成的關鍵 Lambda 函數啟用「預先配置的併發（Provisioned Concurrency）」17。此設定能確保賽事期間始終有一批運算實例維持在「暖機（Warm）」的待命狀態，一旦 SQS 分配任務，函數即可在數毫秒內啟動執行緒進行處理。  
此外，在 Lambda 的組態設定上，針對運行 Puppeteer 的函數，必須分配至少 2048 MB 的記憶體，以確保瀏覽器引擎有足夠的資源進行 HTML 與高解析度圖片的渲染，同時將逾時（Timeout）設定延長至 60 秒以防超時中斷20。在程式碼實作層面，系統將採用單例模式（Singleton Pattern），在 Lambda 的全域執行環境中持久化加載 AI 模型與瀏覽器實例（Browser Instance）；使得同一個運算容器在處理連續不斷的照片流時，能重複利用已啟動的資源，從根本上分攤掉初始化的龐大時間開銷20。

### **4.3 資安與隱私權（合規性與社群授權）**

社群授權代幣與跑者影像涉及極高的資安與隱私風險。從基礎架構層面觀之，所有存放在資料庫中的 OAuth Access Token 與 Refresh Token 必須採用應用程式層級的高強度加密（如 AES-256-GCM 演算法），並搭配 AWS Key Management Service (KMS) 進行金鑰輪替與安全存取管控。這確保即使資料庫層面遭到惡意攻破，攻擊者也無法解密並濫用被盜取的 Token 侵入跑者的私人社群帳號。在資料傳輸層，無論是 S3 上傳、SQS 訊息傳遞，或是與第三方社群 API 的通訊，全程強制使用 TLS 1.2 以上版本的加密協定14。  
在法律合規性方面，本專案嚴格遵循台灣《個人資料保護法》（PDPA）的規範。對於系統涉及的影像收集與自動發布行為，隱私權保護機制涵蓋以下落實面向：

| PDPA 法規核心要求 | 系統實作與合規對策 |
| :---- | :---- |
| **特定目的明確性與告知** | 註冊時必須明確宣告收集照片的專屬特定目的為「賽事成績發布與社群留念」，嚴禁將蒐集之照片流用於未經告知的商業演算法訓練或無關的第三方廣告行銷25。 |
| **獨立與明確同意 (Separate Consent)** | 社群自動推播之授權機制不得使用預設打勾，且須與賽事一般報名條款完全分離（Separate Declaration），確保跑者在充分知情下擁有真實的選擇權27。 |
| **特種個人資料之處理** | 若未來擴充使用人臉特徵值比對（Biometric Data），依法將被視為高度敏感的特種個人資料，系統必須依法取得書面同意或具備同等效力的強烈電子同意憑證，並建立嚴格的存取控制24。 |
| **資料最小化與永久抹除** | 照片：賽事正式成績公告後 **30 天**自動刪除原始未處理圖片；DLQ 中人工未確認之照片於賽事結束後 **48 小時**強制刪除。處理後已推播之壓縮圖片：賽事結束後 **90 天**自動刪除。完賽報紙 PDF：跑者以下載連結取得後 **30 天** 或 賽事結束後 **180 天**（兩者取其早）自動刪除。OAuth Token：賽事結束後 **30 天** 或 跑者主動撤銷後 **7 天**銷毀（兩者取其早）。Face Re-ID 自拍照（生物特徵資料）：於 Face Re-ID 流程完成後 **24 小時**內自動刪除，不得留存。依據隱私政策，所有資料皆須以 Secure Erasure 方式（即資料覆寫或實體銷毀）處理，不可僅做邏輯刪除28。 |

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

**災難復原（DR）策略：**

| 情境 | 復原方式 | RTO | RPO |
| :---- | :---- | :---- | :---- |
| 單一 Lambda 函數崩潰 | SQS 自動重試（最多 3 次）+ DLQ 寫入 | < 5 分鐘 | 照片可能延遲但不遺失 |
| DynamoDB 資料庫故障 | DynamoDB Auto Scaling + Point-in-time Recovery（PITR） | < 15 分鐘 | 最近 35 天內任意秒 |
| S3 儲存桶意外刪除 | S3 Versioning 開啟 + Life Cycle Rule 保存版本 | < 1 小時 | 最近刪除版本可復原 |
| AWS 主要區域（例：us-east-1）全斷 | 跨區域備份：照片預設同步至 `ap-northeast-1`（東京），RDS 讀寫副本置於 `ap-southeast-1`（新加坡） | < 4 小時 | 最近 5 分鐘資料 |
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

#### **LINE 官方帳號費用（台灣賽事必備）**

| 方案 | 費用 | Push Message 上限 | 適用場景 |
| :---- | :---- | :---- | :---- |
| 輕用量（基本） | NT$0 / 月 | 500 則/月 | 測試環境、小型賽事（< 500 推播） |
| 進用量 | NT$800 / 月 | 5,000 則/月 | 5,000 人賽事（假設 25% 啟用） |
| 高用量 | NT$2,500 / 月 | 25,000 則/月 | 20,000 人賽事（假設 60% 啟用） |
| 無限制 | NT$12,000 / 月 | 無上限 | 大型賽事或多場次賽事主辦方 |

> **Note：** 20,000 人賽事若 60% 啟用（12,000 跑者），每人 3 張照片 = 36,000 則 Push，進用量（5,000）與高用量（25,000）均不足，須使用無限制方案（NT$12,000/月）。

#### **AWS 運行成本（20,000 人賽事，單日 8 小時）**

| 元件 | 估算用量 | 單價 | 估算費用（USD） |
| :---- | :---- | :---- | :----: |
| **Lambda（照片處理）** | 60,000 次調用 × 3s avg | $0.0000166667 / 100ms | ~$30 |
| **Lambda（推播引擎）** | 36,000 次調用 × 1s avg | $0.0000166667 / 100ms | ~$6 |
| **Lambda（PDF 生成）** | 12,000 次 × 15s avg | $0.0000166667 / 100ms | ~$30 |
| **S3 儲存** | 原始圖 200 GB + 處理圖 50 GB | $0.023 / GB | ~$5.75 |
| **SQS** | 60,000 訊息 | $0.40 / 百萬訊息 | ~$0.024 |
| **DynamoDB** | 80,000 Write + 20,000 Read | $1.25 / 100 Write + $0.25 / 100 Read | ~$1.05 |
| **CloudFront CDN** | 100 GB 流出 | $0.0085 / GB | ~$0.85 |
| **合計（不含 AI API）** | | | **~$73.7 / 賽事日** |

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
**5.1 新興社群平台的無縫整合** 由於系統的核心推播邏輯已將社群 API 的複雜性抽離為獨立的轉接器模組，當未來如 Bluesky（使用 AT Protocol，圖片上限 1MB）、TikTok（支援 Content Posting API）或 Pinterest 等新興平台崛起時44，開發團隊只需撰寫對應的 API 介接層，即可快速將新平台納入跑者的綁定選項中，而無須大幅重構核心的影像處理或排程系統。  
**5.2 AI 辨識能力的演進：超越號碼布的身分確認技術** 當前系統以號碼布 OCR 為主要身分匹配方式（見 §3.2 Face Re-ID Fallback）。然而，冬季賽事中跑者常穿著外套遮擋號碼布，或號碼布因汗漬/泥濘嚴重污損，導致 OCR 完全失效。未來系統可擴充以下尚未實作之身分確認技術：

- **Person Re-ID（身軀重識別）：** 以深度神經網路比對跑者身形特徵（如體型、步態、穿著顏色），不依賴號碼布或人臉。適用於號碼布遮蔽且用戶未上傳自拍照之情境。需額外收集 training dataset 涵蓋多種氣候與姿態。
- **Gait Recognition（步態辨識）：** 透過影片分析跑者獨特步態特徵，進一步提升集團通過時的身分確認準確率。
- **多模態融合（Multi-modal Fusion）升級：** 將 OCR 置信度、人臉比對分數、Person Re-ID 分數以加权评分模型（Weighted Score Fusion）進行綜合排序，而非現行之串聯式降級（Cascade）判斷。
- **自監督學習（Self-supervised Learning）：** 以賽事內大量未標記照片自動學習跑者視覺特徵 Embedding，減少對大規模標註資料集的依賴。  
**5.3 多運動類別擴充：三鐵、自行車與游泳賽事支援** 本系統架構之 OCR 與物件偵測管線可擴充至三鐵（鐵人三項）、自行車賽與游泳賽事。需注意差異化訓練資料：自行車賽中參賽者之號碼布常置於頭盔或車身而非胸前，OCR 目標區域與姿態各異；游泳賽事中參賽者於出發與轉換區拍攝，衣物與號碼布形式亦不同於路跑。模型微調須收集中、長距離鐵人三項賽事之公開照片集（如 Ironman、Challenge Family 系列賽）與 UCI 自行車賽影像，建立獨立之領域適配模型權重，再於推論時依據賽事類型動態切換對應模型。

**5.4 從靜態影像邁向動態短影音（Reels / Shorts）的全自動生成** 隨著全球社群媒體的演算法全面向短影音（Short-form Video）傾斜，單純的靜態照片推播已不足以壟斷社群流量與注意力。未來的擴充藍圖中，無伺服器處理層可引入基於 FFmpeg 的 Lambda Layer 或整合雲端影片轉碼服務（如 AWS Elemental MediaConvert）。當系統偵測到特定點位（如終點線）有拍攝跑者的連拍圖或短片時，可自動將跑者的配速數據、動態賽事 Logo 與背景音樂合成為一支 15 秒的垂直格式（9:16）短影片。這些影片可透過 Instagram Reels API（支援最高 15 分鐘長度）或 Threads Video API（支援長達 5 分鐘、大小達 500MB 的影片）自動發布13。這項基於短影音的自動化升級，將成為進一步提升賽事曝光量與網路互動率的強大新引擎。  
綜上所述，本需求規格與系統架構書定義了一個具備高度前瞻性與技術可行性的數位藍圖。透過嚴謹的無伺服器雲端基礎建設、智慧化的邊緣與雲端協同人工智慧推論，以及緊扣社群脈動的 API 深度整合，此系統不僅將徹底解決傳統賽事攝影的痛點，更將為未來的運動科技服務立下全新的數位轉型標竿。

#### **引用的著作**

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