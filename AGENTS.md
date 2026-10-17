# AI 交接說明：basketball-eco-gamev2（環境保衛戰・垃圾分類投籃）

> 給接手的 AI（Claude、ChatGPT／Codex、Gemini、Copilot、Cursor…）看的交接檔。
> 先讀完這份再動手；改完程式請同步更新本檔的「目前狀態」與 README 的版本紀錄。
> 最後更新：2026-10-17（v2.2）

## 1. 這是什麼

垃圾分類投籃遊戲：8×8 LED 矩陣顯示物品圖案並語音念出名稱，玩家把它「投」進正確的分類籃框（♻️ 資源回收／🍎 廚餘／🗑️ 一般垃圾／⚠️ 有害垃圾）。限時 30 秒，答對 +100、答錯 −50（最低 0）。

作者是做 Arduino 互動裝置與教學的老師；網頁版是實體裝置（LED 矩陣、WS2812 燈環、紅外對射進球偵測、語音模組）的模擬與展示用。

- 線上網址：https://jackeytw886.github.io/basketball-eco-gamev2/ （2D）、`/3d.html`（3D）
- 部署：GitHub Pages，推到 `main` 約 1～2 分鐘後生效

## 2. 檔案

| 檔案 | 內容 | 注意 |
|---|---|---|
| `index.html` | 2D 原版（約 680 行，純 HTML/CSS/JS） | 這個檔由作者用 **EZPage** 工具更新（commit 訊息「更新 index.html via EZPage」），可能被整檔覆蓋；除非作者要求，盡量不要改它 |
| `3d.html` | 3D 模擬版＋攝影機投籃（約 1780 行，單檔） | 主要開發都在這裡 |
| `README.md` | 給人看的說明：玩法、題庫、版本紀錄 | 改功能時同步更新 |
| `AGENTS.md`／`CLAUDE.md` | 本交接檔／Claude Code 入口 | |

## 3. `3d.html` 架構

單一 `<script type="module">`，用 importmap 從 CDN 載入：

- `three@0.160.0`（`cdn.jsdelivr.net/npm/three@0.160.0/…`，含 OrbitControls、EffectComposer、UnrealBloomPass、OutputPass）
- `@mediapipe/tasks-vision@0.10.21`（按下攝影機才動態 `import()`），模型 `pose_landmarker_lite.task`（`storage.googleapis.com/mediapipe-models/…`）

程式依註解編號分段（搜尋 `// 1.`～`// 12.` 即可找到）：

| 段落 | 重點 |
|---|---|
| 1. 圖案庫 | `ART`（8×8 陣列）、`NUMBERS`（3×5 數字字型）、`LED_COLORS`、`CATEGORIES`（順序＝鍵盤 1～4）、`ITEMS`（題庫：name／art／color／category） |
| 2. 場景 | renderer、相機、OrbitControls、bloom（threshold 0.95；發光物件用「顏色倍率 >1」的 MeshBasicMaterial 才會有光暈） |
| 3. LED 矩陣 | 64 顆 `InstancedMesh`；`setDot(i, color)`、`drawMatrix(art, color)`、`clearMatrix()` |
| 4. 看板 | 計分板 `drawScoreboard()`、名稱看板 `setName(text)`，都是 canvas 貼圖 |
| 5. 籃框 | 籃板、籃框、籃網（LineSegments）、每框 56 顆燈環；`hoops[]`、`hoopById`、`hitBoxes`（點擊判定）；燈環效果 `ringFx(hoop, 'good'|'bad'|'hint')`，`updateRing()` 每幀更新（攝影機瞄準中的框會整圈呼吸亮） |
| 6. 版面 | `LAYOUTS.land`（橫向一排四框）／`LAYOUTS.port`（直向 2×2）；`fitCamera()` 依內容邊界取景，攝影機面板開啟時會讓出空間 |
| 7. 籃球 | `launchBall(hoop, srcX)` 回傳 Promise，球穿過籃網那一刻 resolve；`updateBalls(dt)` 拋物線＋落地彈跳 |
| 8. 音效 | Web Audio 合成音（right／wrong 與 2D 版相同、swish、bounce、throw）；`speak()` 用 speechSynthesis（zh-TW） |
| 9. 遊戲流程 | `startGame()`、`nextItem()`、`throwBall(catId, srcX)`、`flashNumberSequence()`；**倒數只在 `isAnimating === false` 時進行**（每球播報期間暫停，這是刻意設計） |
| 10. 操作 | 下方四顆按鈕、鍵盤 1～4、Enter/空白鍵開始、點 3D 籃框（拖曳 <10px 才算點擊） |
| 11. 攝影機投籃 | 見下節 |
| 12. 主迴圈 | `renderer.setAnimationLoop`：計時、`detectPose`、球、籃網、燈環、矩陣更新、`composer.render()` |

### 攝影機投籃判定（段落 11）

- 肩寬 `S` 當長度單位；`rel = (肩膀 y − 手腕 y) / S`（手越高越大）
- **預備**：手腕 `rel < 0.25`（肩膀以下）就記下時間 `armedAt`
- **投籃**：在 `SENS[...].win` 秒內，手腕推過「投籃線」`rel > 鼻子高度 + SENS[...].over`，且手肘 `rel > 0.2`
- `SENS`：高 `{win: .75, over: .25}`、中 `{win: .5, over: .45}`、低 `{win: .35, over: .65}`
- 投完要手放低才會重新預備（`cool` 旗標），兩球最少間隔 0.8 秒
- **瞄準**：鏡像畫面分四欄＝四個籃框；「站的位置」看雙肩中心、「手的位置」看較高那隻手腕；有平滑與遲滯（離欄中心 >0.15 才換欄）
- **舉雙手 1 秒**（雙手腕都高於鼻子 0.2S）＝開始遊戲／再玩一次
- 影像只在瀏覽器處理；攝影機需要 HTTPS（Pages 可以）或 localhost

## 4. 測試方式

`3d.html` 內有測試掛勾（不影響遊戲）：

```js
window.__eco.state()          // 分數、倒數、目前題目、版面…
window.__eco.setTime(秒)       // 直接改剩餘時間
window.__eco.rings()          // 四個燈環目前的模式
window.__eco.simPose(lm, ms)  // 餵一幀假骨架（33 個 MediaPipe landmark），測投籃判定
window.__eco.camState()       // 攝影機狀態與提示文字
```

先前用 Playwright＋Chromium（`--use-gl=swiftshader`）跑過：桌機 1280×760、手機 390×760 的完整流程（開始→答對→答錯→時間到）、假攝影機（`--use-fake-device-for-media-stream`）權限流程、模擬骨架的投籃判定（快推會投、慢舉不會、舉著不連發、兩種瞄準、舉雙手開始）。

若執行環境連不到 CDN：用 `npm i three@0.160.0 @mediapipe/tasks-vision@0.10.21`，再用 Playwright 的 `page.route()` 把 jsdelivr 的網址導到本機 `node_modules`。注意 swiftshader 很慢，遊戲倒數是實際時間，長測試前先 `__eco.setTime(999)`。

## 5. 目前狀態與待辦

- ✅ v2.2：3D 版＋攝影機投籃已上線
- ⚠️ **攝影機對真人的偵測還沒實測**（測試環境下載不到姿勢模型，只用模擬骨架驗證過）。若作者回報太敏感／不靈敏，調 `SENS` 與預備門檻 0.25
- ⚠️ `index.html` 的「查看遊戲說明」連到 `instructions.html`，但這個檔不存在
- ⚠️ 版本標籤 v2.0（`97342a7`）、v2.1（`04cafb3`）、v2.2（`c473b65`）在 GitHub 上還沒建立（之前的工作環境不能推 tag），可在 Releases 頁建立
- 💡 可能的下一步：WebSerial 連 Arduino，讓實體紅外對射偵測到進球時觸發 `throwBall()`；作者另有規劃書提過雙籃框（可回收／不可回收）＋CF9510M 語音出題的實體版

## 6. 和作者合作的慣例

- 對話、介面文字、程式註解都用**繁體中文**（例外：天問 ASRPRO 程式碼裡的中文字串用簡體）
- 偏好**單檔 HTML**（硬體專案是 `.ino`＋HTML 一組）；給檔案時直接給原始檔，**不要打包成壓縮檔**
- 偏好小步快跑：先做出能用的版本，再依回饋微調，少長篇討論
- 所有專案集中放在 GitHub 管理（作者稱「ghub」）；改完要 commit＋push，commit 訊息用繁體中文
- 題庫分類以台灣垃圾分類為準；「有害垃圾」是原版遊戲自訂的類別，新增題目時避免有爭議的品項
