# CLAUDE.md — where-what-how-strategy-studio

「Where-What-How 商業策略工作室」——單檔前端工具，把一個商業議題拆解成三個構面：**Where**（要鎖定什麼戰場）／**What**（該建構什麼致勝模式）／**How**（該如何具體落地）。每個構面產生分析時，內部借用**六頂思考帽的六種視角**（事實／風險／機會／創意／情感／整合結論）交叉檢視同一個問題，最終呈現 5 張角度小卡＋1 張視覺上更突出的「整合結論」卡——**六頂思考帽在這裡是分析引擎/視角來源，不是獨立的六帽 UI 模式**，這是使用者本次明確要求的設計決策。三構面各自產生後，第 4 個分頁「整合建議」會把三段整合結論拼接（或選用 AI 潤飾）成一段完整的策略陳述。

## 架構

單一 `index.html`：CSS/JS 全內嵌、無外部資源、無建置步驟。`data.js`（`window.WWH_DATA`）是規則引擎的唯一真實來源。視覺主題是橄欖草綠色系「戰場／地圖」風格（`--bg #12140d` + 橄欖綠 `--accent #9caf3d`），與姊妹專案（藍/靛藍/珊瑚玫瑰/琥珀/紫/翠綠薄荷/teal/粉/橘/咖啡棕）區隔。

- **表單欄位**：4 個共用基本資料（`issueName`商業議題名稱／`industry`產業或情境，驅動`classifyCategory`／`background`現況背景／`goal`期望目標）＋三構面各自 2 個選填補充欄位（Where: `whereCandidates`/`whereConstraint`；What: `whatAdvantage`/`whatDifferentiation`；How: `howTimeline`/`howResource`）。補充欄位**只餵給 AI prompt 當脈絡，不進規則模板佔位符**，確保規則式引擎永遠可用、不會因選填欄位空白而出現破洞。
- **規則式引擎（免 API 金鑰，同輸入必同輸出）**：`data.js` 的 `classifyCategory(industry, background)` 沿用六桶產業分類（healthTech/foodBeverage/saasSoftware/homeAppliance/eduContent/general），`hashString()`（djb2變體）+ `pickTpl()` 確定性選模板。`WHERE_TEMPLATES`/`WHAT_TEMPLATES`/`HOW_TEMPLATES` 各為 `{桶名: {角度鍵: [模板字串]}}`，角度鍵固定 `fact/risk/opportunity/creative/emotional/conclusion`，`conclusion` 模板必須是綜合前五角度的收斂句、不是又一則獨立觀點。`index.html` 的 `generateRuleFacet(facetKey, s)` 依構面挑對應模板庫套值。
- **AI 優化路徑（選用，BYOK）**：`callLLM()`／`extractJsonObject()` 與 `new-product-strategy-studio` 同一套實作（Claude 需 `anthropic-dangerous-direct-browser-access` header；429/500/503/529 重試3次；180秒逾時）。`buildFacetPrompt(facetKey, s)` 是**通用**的 prompt builder（不像 NPS 四模組各自一組），依 `D.FACET_LABELS`/`D.FACET_QUESTIONS` 動態組出六角度提問，AI 回傳 JSON 需含六個角度鍵；`validateFacetAi()` 逐角度驗證，缺漏或無效的角度個別退回規則式結果（`missing[]`＋`textSource:'ai'|'mixed'|'rule'`），不整批放棄。整合建議分頁另有 `buildIntegrationPrompt()`/`validateIntegrationAi()`，把三構面的 `angles.conclusion` 送給 AI 潤飾成一段陳述，失敗退回規則式拼接版 `generateRuleIntegration()`。`runGeneration()` 是四個產生流程（where/what/how/integration）共用的封裝：沒填金鑰直接用規則式、有填金鑰才呼叫 AI 並在失敗時自動退回規則式。四個產生流程**共用同一組** provider/model/apiKey 設定面板。

### localStorage

`wwhState`（`{issueName,industry,background,goal,whereCandidates,whereConstraint,whatAdvantage,whatDifferentiation,howTimeline,howResource,results:{where,what,how,integration}}`，切換分頁不遺失其他分頁已產生的內容）、`wwhApiConfig`（`{provider,model,apiKey}`）、`wwhMarquee`（跑馬燈快取）。**不使用**序號授權相關的 key。

## 與姊妹專案的差異

- **不加序號授權機制、不打包 exe**——比照 `mandala-thinking`/`scamper-thinking-generator`/`coffee-ig-planner`/`social-post-grader` 的先例，這是「思考框架練習工具」類別，使用者本次明確要求不要鎖工具。
- **六頂思考帽當作分析引擎，不是獨立模式**——這點與 `six-thinking-hats-generator`（六頂帽子本身就是產出、依序或單一模式切換）明顯不同：本專案的六角度是內嵌在 Where/What/How 三構面分析裡的固定結構，使用者看不到「切換帽子」的操作。
- **prompt builder 是通用函式**（`buildFacetPrompt(facetKey, s)` 依構面動態組提示詞），不像 `new-product-strategy-studio` 四模組各自寫一組 `buildXxxPrompt`——因為六角度結構在三構面間完全一致，只有問題描述（`FACET_QUESTIONS`）不同。
- **整合建議分頁**是本專案獨有的第 4 個構面，把三構面的「整合結論（藍帽）」拼接成一段完整策略陳述，這是 `new-product-strategy-studio`／`scamper-thinking-generator` 都沒有的跨構面統整機制。

## 共用模組（直接複製既有程式碼）

- **跑馬燈**：`#marqueeBar` 逐字複製 `new-product-strategy-studio/index.html` 的 IIFE，共用同一個 Google Apps Script 端點與 Sheet（`localStorage` key 改為 `wwhMarquee`）。
- **PWA**：`manifest.json`／`service-worker.js`／`#installBtn` 邏輯逐字複製既有專案（network-first + 同源快取備援，iOS/macOS Safari 安裝 fallback 文案）。圖示（`icons/`）用 Python PIL 現畫，橄欖綠底＋指北針星芒圖示（呼應「戰場定位」的隱喻），一次性腳本用完即刪、未進版控。
- **PDF匯出浮水印**：`#pdfWatermark`（`<img id="wmImg">`），base64 內容用 Python 腳本直接從 `new-product-strategy-studio/index.html` 的 `WATERMARK_DATA_URI` 常數抽出字串複製過來（未經對話視窗顯示），與姊妹專案共用同一張「馬克老師 AI・工具・學習・成長」品牌圖片。
- **訪客計數器**：`visitor-badge.laobi.icu`，`page_id=m255525.where-what-how-strategy-studio`。
- **創作者資訊／使用警語**：`manual.html`／`index.html` footer 逐字複製既有共用內容，第一點警語措辭改成貼合 Where/What/How 策略分析的情境。

## Port

**8807**（工作區 8765-8806 已全數被其他專案占用，8806 是最新的 `scamper-thinking-generator`）。已在 `.claude/launch.json` 新增對應設定，供 Preview MCP 使用。

## 指令

無建置/測試指令。修改 `index.html` 後直接用瀏覽器開啟驗證，或暫起 `python -m http.server 8807` 測完關閉。修改內嵌 `<script>` 後可用以下方式快速檢查語法：

```bash
python -c "
import re
html = open('index.html', encoding='utf-8').read()
open('_check.js','w',encoding='utf-8').write('\n\n'.join(re.findall(r'<script>(.*?)</script>', html, re.S)))
"
node --check _check.js
node --check data.js
```

驗證 `data.js` 模板庫完整性（每個桶×構面×角度都有內容）：

```bash
node -e "
global.window = {};
require('./data.js');
var D = window.WWH_DATA;
['WHERE_TEMPLATES','WHAT_TEMPLATES','HOW_TEMPLATES'].forEach(function(f){
  Object.keys(D.CATEGORY_LABELS).forEach(function(b){
    D.ANGLE_KEYS.forEach(function(a){
      if(!D[f][b] || !D[f][b][a] || !D[f][b][a].length) console.log('MISSING', f, b, a);
    });
  });
});
console.log('checked');
"
```

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假的 provider 回應格式，確認 `callLLM → extractJsonObject → validateFacetAi → renderFacet` 整條管線正確（含單一角度驗證失敗時的單點 fallback），測完記得還原 `window.fetch`。

## 部署

已推公開 GitHub repo `M255525/where-what-how-strategy-studio`，用 `.github/workflows/deploy-pages.yml`（比照 `scamper-thinking-generator`／`mandala-thinking` 逐字複製）以 Actions workflow 部署 GitHub Pages（非 legacy branch-source）。

## 本次未做（後續視需要再處理）

- 桌面版 exe 打包
- 序號授權（使用者本次明確要求不要鎖工具；若之後要鎖，比照 `ai-prompt-generator` 的「鎖整個工具 12 個月」模式加回）
