# **馬拉松賽事與社群互動連結系統：需求規格與系統架構書**

## **1\. 專案概述 (Project Overview) & 核心價值**

在大型路跑與馬拉松賽事中，參賽者的數位體驗已成為賽事品牌經營的核心要素。然而，傳統馬拉松賽事的影像紀錄與分發流程，長期面臨效率低落與使用者體驗不佳的技術瓶頸。

就市場現況而言，目前全球已存在若干基於 AI 技術之賽事攝影服務（如 RaceTagger、Photohawk、Roboflow 等）。然而，現有方案多止於「AI 標記輔助攝影師整理」，均未實現將照片全自動推送至跑者個人社群帳號之最後一哩路。本系統之核心差異化定位在於：**從攝影到社群平台的最後一哩自動推播層**，此功能在全球及台灣市場中均屬缺口。過去數十年來，賽事攝影師多半依賴光學字元辨識（Optical Character Recognition, OCR）技術掃描跑者號碼布，或者純粹仰賴事後耗時的人工標記1。在真實的賽事環境中，號碼布經常因為跑者的肢體動作而彎折，或是被雨衣、外套遮蔽，甚至被汗水與泥漿覆蓋，這些現實環境干擾使得傳統 OCR 的辨識率大幅降低1。當辨識失敗時，成千上萬張未經整理的原始照片往往被直接上傳至雲端硬碟，迫使跑者必須在海量圖庫中進行漫長且令人沮喪的檢索，此現象被業界稱為「圖庫疲勞」（Gallery Fatigue）1。這不僅導致跑者錯失了在完賽當下即時分享成就的黃金時機，也讓賽事主辦單位流失了寶貴的社群曝光動能。  
本專案旨在開發名為「馬拉松賽事與社群互動連結系統」的創新數位服務，透過深度整合邊緣運算、雲端無伺服器架構（Serverless Architecture）、深度學習物件偵測與文字萃取模型，以及主流社群平台的應用程式介面（API），徹底翻轉傳統賽事影像的交付與傳播模式。系統的核心價值在於提供極致的「即時滿足感」（Instant Gratification），將分類與標記的沉重負擔從人類轉移至機器與演算法1。當跑者通過攝影點位後，系統將以全自動化的串流工作流程擷取影像、進行雲端人工智慧解析、動態套用賽事專屬視覺化數據（如配速、完賽時間、贊助商相框），並在極低的延遲內，將美化後的專屬照片推播至跑者事先授權的個人社群平台（包含 Instagram、Threads、Facebook 與 LINE）。  
此全自動化的數位轉型不僅能極大化跑者的參與感與專屬尊榮感，更能為賽事主辦單位與贊助商創造指數型的社群曝光度。透過跑者社群網絡的病毒式傳播，賽事品牌的數位足跡將以自動化推播的方式達到最大化。同時，系統於賽後自動生成的「個人專屬完賽報紙」，深度結合了無線射頻辨識（RFID）晶片計時數據、精選影像與社群互動指標，進一步將參賽的瞬間感動昇華為具備長久紀念價值的客製化數位資產。

## **2\. 系統架構與資料流 (System Architecture & Data Flow)**

為確保系統在數萬名跑者同時參賽、數十位專業攝影師密集上傳高畫質影像的極端負載下，仍能維持毫秒至秒級的低延遲與系統高可用性，本系統採用基於雲端原生（Cloud-Native）的事件驅動無伺服器架構（Event-Driven Serverless Architecture）2。整體的拓樸結構被精密劃分為四個主要層級：資料獲取與攝入層、非同步串流緩衝與處理層、人工智慧推論與渲染層，以及社群發布與展現層。  
在資料獲取層中，賽道沿線的專業攝影設備透過 5G 路由器或專線網路，將高解析度影像即時且持續地推送到雲端物件儲存空間（如 Amazon S3）3。本系統採用賽道定點攝影模式：賽前公告固定攝影點位，跑者預期在已知位置獲得拍攝服務。賽事主辦單位將提供攝影師完善硬體設施（5G 熱點、供電、遮陽/遮雨帳篷），確保上傳穩定性3。同步地，位於起終點與各個分段檢查點的 RFID 計時系統（例如 MyLaps 系統或搭載 UHF 被動式 RFID 天線的讀取器），會捕捉跑者鞋面或號碼布上的晶片數據4。這些硬體設備或其對應的中介軟體（Middleware）會將包含讀取器 ID、晶片 ID、GPS 經緯度以及世界協調時間（UTC）時間戳記等欄位的 JSON 陣列，透過 HTTPS POST 請求傳送至本系統的計時 API 閘道器7。  
當影像檔案寫入 S3 儲存體時，會立即觸發 S3 事件通知機制，將該影像的處理任務物件推入 Amazon SQS（Simple Queue Service）訊息佇列中3。引入 SQS 作為非同步與佇列點對點處理（Queued point-to-point processing）的關鍵在於解耦與削峰填谷；它允許下游的無伺服器運算單元以自身的最佳節奏擷取任務，避免瞬間的巨量影像上傳壓垮後端運算資源與資料庫連線3。  
接下來，SQS 將觸發具備高度彈性擴展能力的運算服務（如 AWS Lambda），啟動核心的影像處理與人工智慧管線。Lambda 函數會呼叫預先訓練好的邊緣或雲端物件偵測模型（如 YOLOv8 或 RF-DETR），精確進行跑者身軀與號碼布的邊界框（Bounding Box）定位，隨後交由 OCR 引擎提取字元2。辨識成功後，影像將傳遞至基於 Node.js 的 Sharp 影像處理模組，進行浮水印、濾鏡與賽事數據的動態疊加渲染10。  
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

### **2.6 多租戶隔離策略**

本系統支援「同一套系統同時服務多場賽事」，以帳號資料隔離為核心設計原則：

- **網路層級：** 每場賽事之 Lambda 函數、VPC、Subnets 以賽事 ID 作為前綴，透過 AWS Resource Tag 與 IAM Condition 限制跨賽事資源存取。
- **資料庫層級：** DynamoDB Table 以賽事 ID 作為 Partition Key，確保查詢範圍永遠限於單一賽事；RDS 以 Schema 隔離（`event_{event_id}`），避免資料洩漏。
- **OAuth Token：** Token 嚴格绑定「賽事 ID + 跑者 ID」，跨賽事呼叫時 Token 比對會立即失敗，防止 A 賽事的照片被錯誤推播至 B 賽事跑者。

| 隔離層級 | 機制 | 失效情境 |
| :---- | :---- | :---- |
| 網路 | IAM Condition + Resource Tag | Lambda 角色設定錯誤導致跨 partition 讀取 |
| 資料庫 | DynamoDB Partition Key / RDS Schema | 查詢漏接 Partition Key 導致 Cross-event scan |
| OAuth | Token 附加 event_id 聲明（JWT Claims） | Token 被盜用且攻擊者成功更換綁定 event_id |

| 系統架構元件 | 採用技術與核心服務 | 核心職責與資料流向 |
| :---- | :---- | :---- |
| **資料獲取與攝入端** | 5G 路由器、FTP/API 腳本、UHF RFID 解碼器 | 攝影師高頻影像上傳；RFID 晶片通過時間點（JSON 陣列）推送，涵蓋 timingId 與 \`timestamp6。 |
| **雲端儲存與事件觸發** | Amazon S3、S3 Event Notifications | 原始高畫質影像持久化儲存，自動觸發非同步的影像建立事件3。 |
| **訊息緩衝與流量調節** | Amazon SQS (Standard 佇列) | 影像處理任務非同步排程，內建重試機制與死信佇列（DLQ），防止大流量突發造成的處理遺漏3。 |
| **無伺服器運算與編排** | AWS Lambda (Node.js/Python 執行環境) | 執行核心業務邏輯、協調 AI 模型推論、進行資料庫 I/O 操作與處理社群 API 調用15。 |
| **AI 視覺辨識引擎** | YOLOv8 / RF-DETR 搭配特徵金字塔網路, Gemini API | 跑者身軀框選追蹤、號碼布精確定位與變形幾何校正、高精準度光學字元提取2。 |
| **影像與文件動態渲染** | Sharp (Node.js via Lambda Layer), Puppeteer-core | 高效能記憶體內合成相框、疊加配速數據、修復 EXIF 翻轉；無頭瀏覽器生成專屬 PDF 完賽報紙18。 |
| **社群 API 整合模組** | OAuth 2.0, Meta Graph API, LINE Messaging API | 權杖生命週期管理與加密、多段式媒體容器上傳、API 頻率限制（Rate Limit）控制與退避策略13。 |

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
AI 辨識流採用多階段的深度學習管線。傳統單純依賴 OCR 的作法在賽道複雜背景、隨意變換角度與惡劣氣候中極易崩潰1。因此，本系統首先調用基於 YOLO 架構（如 YOLOv8 或專為伺服器端優化的 RF-DETR）的先進物件偵測模型2。該模型經過包含多種光影條件、跑者姿態與複雜背景（In-the-wild benchmark datasets）的專屬賽事資料集進行微調（Fine-tuning），能迅速在百萬像素的影像中框選出「跑者身軀」，接著再進一步於身軀範圍內定位出「號碼布」的精確邊界框（Bounding Box）29。定位完成後，系統會對號碼布區域進行幾何校正與形態學操作（Morphological Operations），將傾斜或皺褶的區域拉平，最後再將處理後的高對比區塊送入 OCR 引擎提取精確的數字序列2。此種結合特徵金字塔網路（Feature Pyramid Network）與卷積注意力機制（Convolutional Block Attention Module）的方法，能有效排除環境雜訊，將號碼布辨識準確率提升至高達 91.6% 以上的商業標準2。

當 OCR 引擎回傳之置信度分數（Confidence Score）低於预设阈值時（例如低於 0.7），系統將自動觸發**號碼布 OCR 失敗的容錯回退機制（Fallback Mechanism）**：以 YOLOv8/RF-DETR 偵測並裁切跑者臉部區塊，调用 SnapSeek 架構之人脸比对引擎，與跑者於註冊時自願上傳之清晰自拍照进行空间几何特征比对（需跑者於註冊時另行授權），以多模態融合（Multi-modal）方式提升恶劣环境下的身份对接成功率。若 Face Re-ID 置信度同樣低於阈值，該照片標記為「待人工確認」，寫入 DLQ 由賽事方進行後續處理。  
辨識成功並與資料庫比對無誤後，系統隨即進入「動態美化處理」階段。此階段同樣於無伺服器環境中執行，主要利用建置於 Lambda Layer 上的高效能 Node.js 影像處理模組 Sharp 來完成10。數位相機拍攝的原始圖片通常包含 EXIF 旋轉元數據，若不加以處理，合成後的影像可能會呈現倒置；因此，Sharp 會首先呼叫 .rotate() 方法進行 EXIF 自動翻轉校正18。接著，系統會根據賽事主辦方的設定，利用 .composite() 方法動態疊加帶有透明度（Alpha Channel）的 PNG 賽事專屬相框與活動贊助商浮水印18。此外，系統可即時查詢該跑者通過該點位的 RFID 晶片配速與經過時間，並利用文字渲染功能將其實時繪製於相片角落。Sharp 函式庫底層運用 libvips，能在極低的記憶體消耗與毫秒級的時間內完成高品質的影像合成，確保 Lambda 函數不會因超時（Timeout）或記憶體不足而中斷10。

### **3.3 自動社群推播：跨平台的動態適配與流量控制機制**

當跑者的專屬美化照片生成後，系統必須將其即時推送至跑者授權的社群平台。然而，各家社群平台的 API 規範與媒體限制差異甚大，系統的推播模組必須具備高度的動態適配與嚴格的流量控制能力。  
針對 **Threads 平台**，系統可利用 Threads API 發布最多包含 10 個媒體項目的輪播（Carousel）貼文，單張圖片大小上限為 8MB12。Threads 平台非常適合行動裝置瀏覽，官方強烈建議的最佳行動端長寬比為垂直的 9:16（1080 × 1920px）或 4:5（1080 × 1350px），且允許在輪播中混合不同的長寬比而不強制裁切33。Threads 允許高達 500 字元的主文字段落，且支援多達 10,000 字元的純文字附件，系統可自動帶入豐富的跑者賽事表現文字、即時感動與活動專屬 Hashtag33。  
針對 **Instagram 平台**，發布流程更為繁瑣，採用「媒體容器（Media Container）」的兩步驟發布模式13。系統首先需呼叫 POST 請求創建一個裝載圖片的容器（若是影片則需建立可續傳的 rupload 容器），待 Meta 伺服器回報容器備妥後，再呼叫發布端點完成貼文13。Instagram API 嚴格限制僅支援 JPEG 格式，不支援 PNG 或 WebP，單張圖片大小上限為 8MB，且長寬比必須介於 4:5 至 1.91:1 之間12。若處理後的圖片不在規範內，API 將直接拒絕請求。此外，單一商業帳號在 24 小時內僅能透過 API 發布 25 篇貼文（輪播算作一篇）13。雖然此限制對單一企業帳號大量發布較為不利，但由於本系統是透過跑者「個人」授權的 Token 代表跑者發布，因此配額是計算在跑者的帳戶上，系統只需控管好每個獨立 Token 的使用頻率即可。  
針對 **LINE 平台**，系統需透過 Messaging API 向跑者發送影像訊息。其規範要求提供原始圖片（Original Image）與預覽圖片（Preview Image）兩組安全的 HTTPS URL14。因此，系統需在 S3 儲存桶中將處理好的大圖與壓縮後的縮圖分別存放，並產生對應的公開網址或預先簽名網址（Pre-signed URL），再將此 JSON 負載傳送給 LINE API，讓跑者能在對話視窗中無縫點擊放大，獲得最即時的完賽肯定14。

| 社群平台 | API 核心發布限制 | 影像格式與尺寸要求 | 最佳實踐與文字規範 |
| :---- | :---- | :---- | :---- |
| **Threads** | 輪播最多 10 個媒體項目（部分用戶達 20 個），圖片上限 8MB33。 | 支援 JPG, PNG。建議 9:16 (1080x1920) 或 4:5 (1080x1350)33。 | 500 字元主文字限制，不強制裁切輪播圖片比例33。 |
| **Instagram** | 25 篇 API 貼文 / 24小時。採兩階段容器發布模式13。 | 僅限 JPEG，最高 8MB。長寬比需介於 4:5 與 1.91:1 之間12。 | 標題最多 2,200 字元，最多 30 個 Hashtags，不支援 PNG13。 |
| **LINE** | 頻率限制視官方帳號方案而定，支援 Push Message37。 | 需同時提供 HTTPS 的原始圖 URL 與預覽小圖 URL14。 | 可整合 Imagemap 或圖文選單，實現更高互動性的完賽推送14。 |

### **3.4 完賽個人專屬報紙（客製化）：無伺服器架構下的動態 PDF 生成**

賽事結束後，系統會彙整每位跑者的多維度資料，產生一份極具紀念價值且可供列印的客製化「個人專屬報紙」。這份數位報紙的資料來源涵蓋廣泛：包含自計時系統 API 傳回的 JSON 數據（包含各檢查點的 timingId、chipId、timestamp 與最終完賽晶片時間）4；系統在賽事中捕捉到並辨識成功的精選衝線照片；以及跑者在社群平台上互動所產生的數據反饋（如點讚數、特定活動 Hashtag 的擴散程度等）。  
在輸出格式與呈現邏輯上，此專屬報紙被定義為一份高解析度、相容於 A4 實體列印的 PDF 文件。其版面設計模擬傳統頭版新聞，主視覺為跑者的高畫質大圖，並搭配動態生成的標題（例如：「破風前行！\[跑者姓名\] 以 \[完賽時間\] 征服 \[賽事名稱\]」）。版面側邊以視覺化的圖表（如折線圖）呈現各分段區間的配速曲線，底部區塊則拼貼社群互動亮點與贊助商致敬欄位。  
為實現此高度動態且排版複雜的文件生成，系統採用無伺服器環境中的 Puppeteer-core 搭配無頭瀏覽器（Headless Chromium）解決方案，捨棄了在排版彈性上較差的 PDFKit20。系統後端首先將匯集的 JSON 數據注入預先設計好的 HTML5/CSS3 樣式模板中（使用如 Pug 或 Jinja2 模板引擎）39。隨後，AWS Lambda 函數會啟動專為 Serverless 環境壓縮的輕量化 Chromium 執行檔（如 @sparticuz/chromium-min 或利用 AWS Lambda Layer 載入的 chrome-aws-lambda）20。  
在記憶體中渲染該 HTML 頁面時，Chromium 會確保所有特殊字體（如粗體報紙標題字型）、複雜的 CSS 網格排版與外部高解析度圖片皆完美載入。為達到最佳列印效果，開發團隊將在樣式表中宣告 @media print { @page { size: A4 portrait; margin: 0; } }，確保產出的 PDF 不留白邊41。生成的 PDF 檔案將匯出為 Buffer 並自動寫入 Amazon S3 存放，隨後透過電子郵件或 LINE 訊息將預先簽名的下載連結遞送給跑者，完成賽事體驗的最後一哩路20。

### **3.5 系統 API 設計**

本系統之 API 分為「外部公開 API」與「內部系統 API」兩類，所有外部端點均須通過 API Gateway 並啟用 Rate Limiting 與 IAM 授權。

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
  "chipId": "MYLAPS-ABCD1234",
  "timestamp": "2026-06-26T04:30:00.000Z",
  "gpsLat": 25.0330,
  "gpsLng": 121.5654,
  "checkpointCode": "START"
}
```

#### **DLQ 人工處理 API**

| 端點 | 方法 | 說明 |
| :---- | :---- | :---- |
| `/internal/v1/dlq/{taskId}/assign` | POST | 將 DLQ 任務指派給特定人工處理員 |
| `/internal/v1/dlq/{taskId}/resolve` | POST | 人工確認跑者身分後補充填入 bibNumber |

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

**同一攝影點之連拍叢集（Photo Burst）處理：**當多位跑者在極短時間內相繼通過同一攝影點時，單一 Lambda 實例可能對同一號碼布產生多張照片。系統以「**號碼布 + 拍攝時間窗口（±3 秒）**」作為叢集鍵（Cluster Key），同一叢集內僅選擇「OCR 置信度最高」之一張照片執行推播，其餘寫入跑者個人圖庫供自行下載。

**同一照片含多位跑者（集團通過）：**系統對照片內所有偵測到之候選邊界框（所有 Confidence ≥ 0.5 之 Bib Bounding Box）逐一執行 OCR 與推播；同一張照片可同時歸戶至多位跑者。為避免重複推播，系統以 `photoId + bibNumber` 複合鍵做去重檢查，已成功推播之組合不再重複發布。

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
| `/admin/v1/events/{eventId}/photographer-station` | POST | 新增攝影點位 |
| `/admin/v1/events/{eventId}/participant-csv` | POST | 匯入參賽者 CSV |
| `/admin/v1/events/{eventId}/publish` | POST | 發布賽事（鎖定名單、啟用 LINE 推播） |
| `/admin/v1/dlq/{taskId}/assign` | POST | 指派 DLQ 任務 |
| `/admin/v1/dlq/{taskId}/resolve` | POST | 人工確認並標記 DLQ 完成 |

## **4\. 非功能性需求 (Non-Functional Requirements)**

非功能性需求定義了系統在極端壓力下的表現與基礎設施的穩健程度。對於一項需要即時處理巨量多媒體數據與個人隱私的賽事系統而言，高併發處理能力、超低延遲的運算架構與堅若磐石的資安防護是不可或缺的三大支柱。

### **4.1 高併發照片上傳與智慧流量塑形 (Traffic Shaping)**

在大型馬拉松賽事中，選手通常呈現集團式移動。當大批跑者同時通過熱門攝影點（如起終點或知名地標）時，多台攝影設備將在極短的時間內上傳數千張高解析度檔案至雲端。這種典型的「突發性流量（Bursty Traffic）」若以同步（Synchronous）方式直接交由後端伺服器處理，極易導致運算資源瞬間耗盡、資料庫連線池崩潰，甚至引發雪崩效應3。  
為應對此挑戰，本系統高度依賴 Amazon SQS 作為核心的流量塑形與緩衝機制。當 S3 接收到新影像時，僅將輕量化的物件元資料（Metadata）非同步發送至 SQS 佇列中3。AWS Lambda 被配置為 SQS 的消費者，利用其強大的擴展性進行影像消化。雖然 AWS Lambda 理論上可以每分鐘增加 60 個實例，瞬間擴展至高達 1,000 個以上的併發執行環境，但這種無節制的擴充將帶來毀滅性的後果——瞬間發出的大量請求會觸發下游社群平台 API（如 Instagram 的 200 次/小時限制）與關聯式資料庫的速率限制（Rate Limits）16。  
因此，系統架構師必須在 SQS 與 Lambda 之間的事件源映射（Event Source Mapping）中設定「最大併發數量（Maximum Concurrency）」16。藉由嚴格控制同時啟動的 Lambda 實例上限（例如限制在 50 個併發），系統能確保影像處理的吞吐量穩定維持在下游 API 可承受的安全閾值內。未及時處理的影像則安穩地保留在 SQS 佇列中等待消化。此外，搭配死信佇列（Dead-Letter Queue, DLQ）設計，即使遇到損毀的影像或 OCR 演算法意外崩潰，該問題任務也會被隔離至 DLQ 中進行後續的人工除錯或重新驅動，確保絕不遺漏任何一張跑者的珍貴影像8。

### **4.2 AI 辨識延遲度與無伺服器冷啟動 (Cold Start) 優化**

提升即時推播體驗的核心關鍵，在於將從「影像上傳」到「社群發布」的端到端時間差壓縮至最短。AI 影像辨識與無頭瀏覽器渲染作為整個管線中最消耗運算與記憶體資源的環節，其延遲度的控制至關重要。

**延遲目標（SLA）：**從攝影師按下快門、影像上傳至 S3 起，至跑者收到 LINE 推播通知為止，目標延遲為 **5 分鐘以內**（P95）。困難場景（OCR 置信度低、需觸發 Face Re-ID fallback）可延長至 15 分鐘。完賽報紙 PDF 生成則於賽事正式成績公告後 30 分鐘內完成。  
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
| **資料最小化與永久抹除** | 賽事結束且專屬報紙遞送完畢後，系統應制定自動銷毀排程。依據隱私政策，必須將暫存的授權 Token 與未比對成功的原始相片從系統中永久且安全地抹除（Secure Erasure）28。 |

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

### **4.5 測試策略**

本系統之測試分為四個層級，各層級有不同的通過標準與工具選擇：

| 測試層級 | 測試標的 | 工具 | 通過標準 |
| :---- | :---- | :---- | :---- |
| **單元測試（Unit）** | 各 Lambda 函數業務邏輯、Adapter Pattern 各平台模組、Token 刷新邏輯、照片歸戶決策函數 | Jest / Pytest | 分支覆蓋率 ≥ 80% |
| **整合測試（Integration）** | SQS → Lambda → DynamoDB 寫入流程、RFID API → Lambda → S3 觸發、Sharp 影像合成輸出驗證 | LocalStack + Jest/Pytest | 所有路徑 100% 通過 |
| **AI 模型驗收測試** | OCR 辨識準確率（以獨立測試集驗證）、Face Re-ID 置信度分佈 | 獨立測試 Dataset（含雨天/夜間/號碼布遮蔽/集團通過等 Edge Case） | OCR ≥ 91.6% 準確率（依規格書目標）；Face Re-ID ≥ 85% Top-1 準確率 |
| **端到端測試（E2E）** | 從 Mock 攝影機上傳 → S3 → SQS → Lambda → LINE Push Message 送達 | staging 環境 + LINE Debug 帳號，模擬 1000 張照片批次處理 | SLA P95 ≤ 5 分鐘達成，無錯誤 |

**AI 模型訓練資料來源：**採用 Hugging Face `race-numbers-detection-and-ocr` 資料集作為基準訓練集，並於每場賽事後以實測失敗案例擴充訓練集，持續微調模型權重。模型版本以 Semantic Versioning 管理，並於 `/models/{version}/` 目錄存放每次上線前之模型 checkpoint。

## **5\. 未來擴充性考量 (Scalability)**

在架構設計初期，本系統便被賦予高度的鬆耦合、模組化與無狀態（Stateless）特性，這為未來的跨領域功能擴充與技術升級提供了極大的彈性與想像空間。  
**5.1 新興社群平台的無縫整合** 由於系統的核心推播邏輯已將社群 API 的複雜性抽離為獨立的轉接器模組，當未來如 Bluesky（使用 AT Protocol，圖片上限 1MB）、TikTok（支援 Content Posting API）或 Pinterest 等新興平台崛起時44，開發團隊只需撰寫對應的 API 介接層，即可快速將新平台納入跑者的綁定選項中，而無須大幅重構核心的影像處理或排程系統。  
**5.2 AI 辨識能力的演進：生物特徵與多模態融合 (Multi-modal Integration)** 當前系統主要依賴號碼布與 OCR 進行身分匹配。然而，在冬季賽事或極端氣候中，跑者穿著防風外套遮擋號碼布，或是號碼布嚴重扭曲的情況屢見不鮮，導致 OCR 徹底失效1。未來的系統架構可擴充整合高精度的臉部辨識（Facial Recognition）或身軀重識別（Person Re-ID）演算法作為容錯回退機制（Fallback Mechanism）1。跑者在註冊階段可選擇性地上傳一張清晰的自拍照；當 OCR 的置信度分數（Confidence Score）過低時，系統可將照片輸入如 SnapSeek 架構的生物特徵比對引擎中，利用空間幾何特徵與多模態技術（Multi-modal technique，融合文字與生物特徵）大幅提高惡劣環境下的身分對接成功率1。  
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