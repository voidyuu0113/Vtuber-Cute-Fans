# Project Summary

這是一款以「直播聊天室互動視覺化」為核心的本地橋接程式。它把 YouTube / Twitch 的聊天室事件轉成前端 Overlay 畫面中的角色、對話泡泡、斗內效果、互動工具（拳頭、踢、戳、摸頭、椅子、抓取）與自訂素材映射，並提供一個功能很完整的 Settings 頁面，讓使用者在本機直接調整素材庫、動作映射、工具參數、表情映射、診斷與來源設定。

目前專案已採用 **Electron shell + 本地 server** 架構：App 啟動後由 Electron 啟動 bridge server，再載入 `http://127.0.0.1:41928/settings.html`；OBS 仍沿用 `http://127.0.0.1:41928/overlay.html`。打包方面已接上 **electron-builder**，目標是輸出 Windows 安裝包與可攜版，最終產物為可直接執行的 `.exe`（NSIS installer 與 portable）。

目前已完成的主要能力：

- 本地 WebSocket / HTTP 橋接，提供 Overlay 與 Settings 頁面同步設定與事件。
- YouTube Primary Source（API + gRPC / REST）與 YouTube Fallback Source（HTML / `ytInitialData` 解析）。
- Twitch IRC Source。
- Overlay 角色生成、移動、對話、互動工具、抓取拖曳、坐下、泡泡釘選、工具游標與特效。
- 完整的素材庫管理：上傳、刪除、重新命名、初始化翻轉、Spine / spritesheet / 單圖素材管理。
- 雙路徑資產系統：
  - default assets（隨 app 發佈、唯讀、不可刪）
  - user assets（位於 userData / AppData、可匯入與可刪改）
- 動作映射、會員覆寫、特殊名稱綁定、配件設定、表情映射群組化 UI。
- Sprite Animation Editor（格線切片、預覽、輸出設定）。
- YouTube 診斷摘要（REST / gRPC / clients gate / fallback）與 diagnostics JSONL。

下面是依檔案整理的職責說明。`server/public/user-assets/**` 已依需求排除；對大型檔案中的大量重複 UI helper，以下採「按功能族群」整理，而不是逐行列出每個重複 rerender 細節。`server/public/**` 中的 `assets/` 與 HTML 目前也是前端 build 產物，因此以下只描述其用途，不逐一展開每個 chunk。

---

## Root / 啟動與頁面

### [package.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/package.json)
- 前端工作區的套件定義。
- 定義 Vite / TypeScript / Pixi / Spine / Electron / electron-builder 相關依賴與 npm scripts。
- 目前主要 scripts 包含：
  - `build:web`: 將前端打包到 `server/public`
  - `build:server`: 編譯 server（含 `.proto` 複製到 `server/dist/yt/proto`）
  - `build:electron`: 編譯 Electron main process
  - `build`: 串接 web + server + electron 全部 build
  - `server:prod`: 用 `node server/dist/index.js` 啟 production server
  - `dist` / `dist:win`: 透過 electron-builder 產出 Windows `.exe`（portable / nsis）

### [AGENTS.md](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/AGENTS.md)
- 專案內部的開發備忘錄。
- 目前明確記錄了重要流程規則：
  - 只要修改會影響 `overlay.html`、`settings.html` 或前端資產載入的程式碼，就必須先執行 `npm run build`
  - 因為 41928 bridge server 直接對外 serve `server/public`

### [package-lock.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/package-lock.json)
- 前端依賴鎖定檔。
- 不含業務邏輯，作用是固定安裝版本。

### [tsconfig.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/tsconfig.json)
- 前端 TypeScript 編譯設定。

### [vite.config.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/vite.config.ts)
- Vite 開發與打包設定。
- 目前已設定：
  - `build.outDir = "server/public"`
  - `build.emptyOutDir = false`（避免清掉 `server/public/user-assets`）
  - multi-entry build，會輸出：
    - `index.html`
    - `overlay.html`
    - `settings.html`

### [README.md](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/README.md)
- 專案整體使用說明。

### [index.html](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/index.html)
- 前端入口 HTML，通常載入主要頁面 bundle。

### [overlay.html](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/overlay.html)
- Overlay 畫面入口 HTML。

### [settings.html](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/settings.html)
- Settings 頁入口 HTML。

### [public/vite.svg](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/public/vite.svg)
- 前端靜態資產（Vite 預設圖示）。

### [tmp-livechat.html](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/tmp-livechat.html)
- 開發期用的臨時 YouTube live chat HTML 樣本。
- 主要用於 fallback parser 驗證。

---

## Scripts / 開發工具腳本

### [scripts/build-update-manifest.mjs](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/scripts/build-update-manifest.mjs)
- 產生 app update manifest。
- 用於版本檢查與更新提示。

### [scripts/download-youtube-emojis.mjs](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/scripts/download-youtube-emojis.mjs)
- 批次下載 YouTube 表情素材。

### [scripts/inspect-youtube-livechat-sample.mjs](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/scripts/inspect-youtube-livechat-sample.mjs)
- 讀取 live chat 樣本並協助人工檢查 fallback 解析結構。

### [scripts/test-lmstudio-youtube-parser.mjs](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/scripts/test-lmstudio-youtube-parser.mjs)
- 測試某些 YouTube parser / 樣本解析流程。

### [scripts/youtube-emoji-list.txt](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/scripts/youtube-emoji-list.txt)
- YouTube 表情素材清單資料。

---

## Frontend / 主要入口

### [src/main.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/main.ts)
- 前端主入口。
- `boot()`: 啟動主頁 UI 或主流程，掛載必要的 DOM / 狀態初始化。

### [src/typescript.svg](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/typescript.svg)
- 前端靜態資產（TypeScript 圖示）。

### [src/overlay.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/overlay.ts)
- Overlay 主控制器，是整個畫面互動與角色運行的核心。
- 主要職責：
  - 建立 Pixi stage、載入 glow texture、建立角色與 renderer。
  - 管理工具選擇、滑鼠指標、自訂工具素材與工具特效。
  - 接收 WebSocket 訊息，把聊天事件轉成 actor。
  - 管理抓取、打擊、摸頭、戳、椅子與泡泡釘選互動。
  - 管理 1D floor / local spacing、泡泡避讓、生成節奏、回收。
- 主要函式：
  - `uid()`: 產生前端本地 ID。
  - `boot()`: 初始化整個 overlay。
  - `createGlowTexture()`: 建立角色發光效果貼圖。
- `boot()` 內的重要內部函式（依功能族群）：
  - 工具與游標：`setSelectedTool()`, `updateToolHud()`, `setToolCursorGlyph()`, `setToolCursorVisualMode()`
  - 指標與抓取：`updatePointerFromEvent()`, `releaseGrabbedActor()`, `updateManualGroundYFlag()`
  - 互動工具：`spawnToolFx()`, `applyHitTool()`, `applyHeadpatTool()`, `applyPokeTool()`, `applySeatTool()`, `breakSeatForInteractiveTool()`, `interactWithActor()`
  - 泡泡釘選：`togglePinnedBubble()`
  - 角色生成：`spawnNext()`, `schedulePendingSpawn()`, `fillSpawnSlots()`
  - 聊天事件：`enqueueChat()`
  - 排版：`applyLocalActorSpacing()`, `apply1dBubbleStacking()`
  - 設定同步：`applySettings()`
  - 視覺輔助：`pauseAndFlipDirection()`
  - 反應收尾：`finishTimedState()`（`hit_react`/`poke_react`/`headpat`/`talk` 轉回 `grab`/`sit`/`walk`）
- 補充：
  - `chair` 工具現在只負責把 actor 狀態切到 `seated`
  - 椅子本體只由 `BoxRenderer` 依 `model.seated` 渲染，不再另外生成第二張 overlay 椅子
  - `hit_react` 目前採「最短停留秒數 +（可判斷時）動畫播完」雙條件收尾；低 FPS 會依素材估算最短停留，不再只用固定秒數

### [src/settings.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/settings.ts)
- Settings 頁面主檔，屬於整個專案最複雜的前端管理台。
- 主要職責：
  - 載入素材庫與伺服器狀態。
  - 提供 Overlay / 工具 / 資產 / 動作映射 / 配件 / 表情映射等 UI。
  - 提供 Sprite Animation Editor。
  - 提供 YouTube 診斷摘要與 fallback 摘要卡片。
  - 將修改即時送回 overlay 與 server。
- 核心函式族群：
  - 國際化與字串：
    - `getUiLocale()`
    - `t()`
    - `localizeActionLabel()`
    - `localizeActionKind()`
    - `getLocalizedToolLabel()`
  - 通用 UI 建立：
    - `el()`
    - `styleButtonChrome()`
    - `styleFieldChrome()`
    - `labeledCheckbox()`
    - `labeledRange()`
    - `labeledNumber()`
    - `labeledText()`
  - 設定與素材同步：
    - `applyAssetLibrary()`
    - `pickPersistedOverlaySettings()`
    - `syncEditorsFromState()`
    - `sendSettings()`
    - `schedulePersistOverlaySettings()`
    - `fetchLibrary()`
    - `fetchAssetImages()`
    - `fetchBridgeRuntime()`
    - `persistOverlaySettings()`
    - `saveFallbackLiveUrl()`
    - `reconnectChatSource()`
  - YouTube 診斷：
    - `normalizeYoutubeDiagnosticsPayload()`
    - `normalizeYoutubeFallbackSummaryPayload()`
    - `buildFetchErrorPayload()`
    - `buildFallbackFetchErrorSummary()`
    - `formatDiagDuration()`
    - `formatDiagDate()`
    - `formatDiagTimestamp()`
    - `formatDiagAgeExpiry()`
    - `formatDiagRecord()`
    - `formatRetryCountdown()`
    - `collectFrontendYoutubeHints()`
    - `appendDiagField()`
    - `createDiagSection()`
    - `createDiagAlert()`
    - `renderYouTubeDiagnostics()`
    - `copyTextWithFallback()`
    - `copyYouTubeDiagnosticsJson()`
    - `fetchYouTubeDiagnostics()`
    - `startYouTubeDiagnosticsPolling()`
    - `stopYouTubeDiagnosticsPolling()`
  - 資產庫操作：
    - `clearAssetPreviewTimers()`
    - `createAssetPreview()`
    - `rerenderAssetList()`
    - `importFilesToLibrary()`
    - `importFiles()`
    - `toImportPayload()`
    - `setDroppedRelativePath()`
    - `collectDroppedFiles()`
  - APNG / Sprite 動畫：
    - `isSpriteAnimCandidateImage()`
    - `shouldAutoConvertImportedApng()`
    - `normalizeApngBaseName()`
    - `formatSpriteAnimSourceLabel()`
    - `rerenderSpriteAnimSourceOptions()`
    - `saveSpriteAnimToServer()`
    - `saveSpriteAnimAssetInternal()`
    - `convertApngToSpriteAnim()`
    - `resolveAssetUrl()`
    - `getAssetOrigin()`
    - `isAssetPathname()`
    - `refreshSpriteAnimEditor()`
    - `rebuildSpriteAnimPreview()`
    - `createSpriteAnimPreview()`
  - 動作映射與配件：
    - `saveActionMappings()`
    - `persistMappings()`
    - `scheduleMappingsPersist()`
    - `clipSupportsFlip()`
    - `clipSupportsScale()`
    - `clipSupportsFps()`
    - `getClipScaleValue()`
    - `getDefaultClipFps()`
    - `createBindingSection()`
    - `rerenderBindingForm()`
    - `rerenderActionOptions()`
    - `buildClipOptionsForAction()`
    - `clipToOptionValue()`
    - `optionValueToClip()`
    - `dedupeSelectOptions()`
    - `dedupeAssetsById()`
    - `buildAccessoryClipOptions()`
    - `accessoryOptionToClip()`
    - `rerenderAccessoryForm()`
  - 會員 / 特殊名稱覆寫：
    - `getNameOverrides()`
    - `describeNameOverride()`
    - `getNameOverrideLabel()`
    - `updateNamedOverride()`
    - `rerenderOverrideForm()`
    - `rerenderSpecialOverrideForm()`
  - 表情映射：
    - `saveEmojiMappingsOnly()`
    - `persistEmojiMappings()`
    - `importEmojiPack()`
    - `uploadEmojiFile()`
    - `loadEmojiUiGroupsState()`
    - `saveEmojiUiGroupsState()`
    - `createEmojiUiGroupId()`
    - `getDefaultEmojiGroupLabel()`
    - `ensureEmojiUiGroupsForPlatform()`
    - `updateEmojiUiGroupsForPlatform()`
    - `addEmojiUiGroup()`
    - `renameEmojiUiGroup()`
    - `deleteEmojiUiGroup()`
    - `appendEmojiMappingEntry()`
    - `toggleEmojiUiGroupCollapsed()`
    - `rerenderEmojiMappingTable()`
  - 頁面啟動：
    - `fetchUpdateStatus()`
    - `normalizeBridgeUrl()`
    - `boot()`
- 補充：
  - 目前 `importFiles()` 會自動辨識多幀 APNG，直接轉成 `spriteAnim`
  - APNG 轉換後會自動建立 spritesheet、載入圖片、套用 grid，並直接儲存成 `spriteAnim` 資產
  - Sprite Animation Editor 的 PNG 來源清單會排除 `/user-assets/tools/`，避免把互動工具素材誤當作動畫圖來源
  - Settings 頁面目前已包含教學頁、YouTube diagnostics + fallback diagnostics 同卡片顯示、以及多個資產 URL 診斷 log
  - 動作映射區的 `Fine tune / 微調` 展開狀態會在重繪時保留（不再一操作就自動收合）
  - `Fine tune` 的 `scale/fps` 拉桿改為 `input` 只更新標題、`change` 才送設定，避免拖曳時被即時重繪中斷

### [src/styles/overlay.css](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/styles/overlay.css)
- Overlay 的樣式表。

---

## Frontend / Shared

### [src/shared/bridgeUrl.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/bridgeUrl.ts)
- 處理前後端 bridge URL 組裝。
- 主要函式：
  - `getBridgeHttpUrl()`
  - `getBridgeWsUrl()`
  - `normalizeBridgeAssetUrl()`
- 補充：
  - 在 dev (`5173`) 下，資產路徑如 `/user-assets/`、`/imports/`、`/assets/`、`/library/` 會自動改寫到 bridge server `http://127.0.0.1:41928`
  - 用來避免 Vite dev server 誤回傳 HTML 或造成跨來源資產載入錯誤

### [src/shared/assetMapping.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/assetMapping.ts)
- 定義素材、動作、綁定、配件、表情映射的共享型別與預設值。
- 主要函式：
  - `resolveBindingClip()`
  - `resolveAccessories()`
  - `findEmojiMapping()`
- 也是前後端共享的核心配置模型來源。

### [src/shared/channel.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/channel.ts)
- 定義 `OverlaySettings` 與 `DEFAULT_SETTINGS`。
- 是 Overlay / Settings 同步設定的基準。

### [src/shared/ws.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/ws.ts)
- 管理前端 WebSocket 單例連線。
- 主要函式：
  - `connectWsSingleton()`: 建立或重用 WebSocket 連線，接收 server 事件。

---

## Frontend / Core 與 Simulation

### [src/core/debug.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/core/debug.ts)
- Debug HUD / 開發除錯輔助。

### [src/core/stage.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/core/stage.ts)
- 建立與封裝 Pixi stage / world。
- 主要函式 / 類別：
  - `createStage()`: 建立 Pixi 應用與場景容器。

### [src/sim/renderer.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/sim/renderer.ts)
- 定義 renderer 介面（例如 actor renderer 需要提供哪些方法）。

### [src/sim/actionCatalog.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/sim/actionCatalog.ts)
- 定義角色可用動作名稱與對應規格。

### [src/sim/actorLogic.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/sim/actorLogic.ts)
- 純邏輯層的 actor 狀態機。
- 主要函式：
  - `getActionForState()`: 把狀態轉成動作 ID。
  - `updateActor()`: 每幀更新 actor 的移動、重力、反應與狀態轉移。

### [src/sim/spamGuard.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/sim/spamGuard.ts)
- 聊天防洗版機制。
- 主要類別：
  - `SpamGuard`
- 主要方法：
  - `shouldBlock()`: 判定訊息是否應被阻擋。

### [src/sim/spatial/SpatialHashGrid.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/sim/spatial/SpatialHashGrid.ts)
- 空間哈希網格，用於碰撞 / 鄰近查詢。
- 主要類別：
  - `SpatialHashGrid`
- 主要方法：
  - `clear()`
  - `insert()`
  - `query()`

### [src/sim/assets/SpineAssetLoader.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/sim/assets/SpineAssetLoader.ts)
- 管理 Spine 資產載入、快取與初始化。
- 主要類別：
  - `SpineAssetLoader`
- 主要方法：
  - `setAssetLibrary()`
  - `load()`
  - `dispose()`

### [src/sim/renderers/boxRenderer.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/sim/renderers/boxRenderer.ts)
- 單一 actor 的主要 renderer。
- 主要職責：
  - 顯示角色本體（圖片 / Spine / 動畫）。
  - 顯示座椅、泡泡、名稱、配件、光效。
  - 處理表情映射的 inline emoji。
  - 套用坐下椅子素材、泡泡釘選樣式、縮放與朝向。
- 主要類別：
  - `BoxRenderer`
- 主要方法：
  - 生命週期：`mount()`, `unmount()`, `syncFromModel()`, `update()`
  - 外觀控制：`setScale()`, `setSeatPropScale()`, `setNameScale()`, `setBubbleLift()`, `setShowUserName()`, `setGlowEnabled()`
  - 素材設定：`setAssetLibrary()`, `setAssetMappings()`, `setEmojiMappings()`, `setSpineLoader()`
  - 動作控制：`play()`, `setPreviewAction()`
  - 播放完成判定：`canAwaitCurrentActionPlaybackCompletion()`, `isCurrentActionPlaybackComplete()`, `getCurrentActionMinimumHoldSec()`
  - 高亮與互動：`setHighlight()`, `onPointerTap()`
  - 版面資訊：`getBubbleWorldAabb()`, `getBubbleHeightWorld()`, `setBubbleAvoidOffset()`, `setBubbleAvoidOffsetTarget()`
  - 內部繪製：`renderBubbleContent()`, `createInlineEmojiSprite()`, `redrawSeatProp()`, `redrawFallbackBody()`, `applyHorizontalFacing()`
  - 視覺 clip 解析：`resolveVisualClipForAction()`（針對 `hitReact/pokeReact/headpat/chat/enter/exit` 強制 one-shot）

### [src/settings/spriteAnim/gridDetect.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/settings/spriteAnim/gridDetect.ts)
- Sprite sheet 自動偵測格線模組。
- 主要職責：
  - 以 alpha mask、投影訊號、autocorrelation、offset/spacing 估計與 integral image coverage scoring 偵測較可靠的 grid。
- 主要函式：
  - `detectGridFromSpriteSheet()`
  - `smooth1D()`
  - `normalize1D()`
  - `autocorrPeaks()`
  - `estimateOffset()`
  - `estimateSpacing()`
  - `buildIntegralImage()`
  - `rectSum()`
  - `scoreCandidate()`

### [src/settings/spriteAnim/gridSlicer.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/settings/spriteAnim/gridSlicer.ts)
- Sprite sheet 切格工具。
- 主要函式：
  - `buildFrameRects()`: 根據格線設定產生每個 frame 的矩形範圍。

---

## Server / 啟動與設定

### [server/package.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/package.json)
- 後端 Node/TypeScript 工作區的套件定義。

### [server/package-lock.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/package-lock.json)
- 後端依賴鎖定檔。

### [server/tsconfig.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/tsconfig.json)
- 後端 TypeScript 編譯設定。

### [server/README.md](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/README.md)
- 後端使用與開發說明。

### [server/ws-server.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/ws-server.ts)
- 舊版或獨立啟動用的 WS server 入口。

### [server/src/index.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/index.ts)
- 後端主入口，負責啟動 Express、WsHub、來源 orchestrator、資產 API、設定 API、診斷 API。
- 主要職責：
  - 建立 `AssetStore`、`WsHub`、OAuth、primary/fallback/twitch source。
  - 管理 overlay client 數量驅動的來源啟停。
  - 提供 `/status`、`/config`、`/api/*`、`/auth/*` 等路由。
  - 對外 serve `server/public`，讓 OBS 可直接載入 `http://127.0.0.1:41928/overlay.html`
  - 提供 `/imports`、`/user-assets` 靜態檔與對應 CORS / CORP header
  - 提供 YouTube 診斷與 fallback 診斷摘要。
- 主要函式：
  - `main()`: 啟動整個 server。
  - `getImageRoots()`: 決定可掃描的圖片根目錄。
  - `getServerDataRoot()`: 定位 data 根目錄。
  - `readJsonFileSafe()`: 安全讀 JSON。
  - `readLastJsonlObject()`: 從 JSONL 取最後一筆。
  - `parseLastJsonlLine()`: 解析最後一行 JSONL。
  - `prefixForDiag()`: 取診斷字串前綴。
  - `sanitizeDiagText()`: 截短診斷文字。
  - `readStringField()`
  - `readNumberField()`
  - `clampDiagQueryNumber()`
  - `scanPublicPngs()`: 掃描可供前端選擇的圖片素材。
- 補充：
  - 目前會在啟動時印出 `importsDir` 與 `publicDir`
  - 目前包含明確路由：
    - `/overlay.html`
    - `/settings.html`
    - `/api/diagnostics/youtube/summary`
    - `/api/diagnostics/youtube-fallback/summary`

### [server/src/config.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/config.ts)
- 管理 app config、runtime config、overlay settings schema。
- 主要類別：
  - `AppConfigStore`
- 主要方法：
  - `getRuntime()`
  - `setRuntime()`
  - `getOverlaySettings()`
  - `setOverlaySettings()`
- 也定義 Zod schema 與預設值。

### [server/src/logger.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/logger.ts)
- 統一 logger 封裝。

---

## Server / WebSocket 與型別

### [server/src/ws/hub.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/ws/hub.ts)
- 管理所有 WebSocket client 與廣播。
- 主要類別：
  - `WsHub`
- 主要方法：
  - `onOverlayClientCountChanged()`
  - `getOverlayClientCount()`
  - `broadcastEvent()`
  - `broadcastSystem()`
  - `broadcastRawForApi()`
  - `toLegacy()`: 將新事件轉成舊格式。
  - `detectType()`: 偵測 client 類型。
  - `handleControlMessage()`: 處理控制訊息。
  - `unregisterClient()`
  - `emitOverlayCountChanged()`
  - `broadcastRaw()`

### [server/src/ws/types.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/ws/types.ts)
- WebSocket 事件型別定義。
- 包含 `ChatEvent` 與 legacy event 型別。

---

## Server / OAuth 與 Token

### [server/src/storage/tokenStore.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/storage/tokenStore.ts)
- 檔案式 token 儲存。
- 主要類別：
  - `TokenStore`
- 主要方法：
  - `load()`
  - `save()`
  - `clear()`

### [server/src/yt/oauth.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/yt/oauth.ts)
- YouTube / Google OAuth 管理。
- 主要類別：
  - `GoogleOAuthService`
- 主要方法：
  - `hasToken()`
  - `getAuthUrl()`
  - `handleCallback()`
  - `getAuthenticatedClient()`
  - `clearToken()`

### [server/src/twitch/oauth.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/twitch/oauth.ts)
- Twitch OAuth 管理。
- 主要類別：
  - `TwitchOAuthService`
- 主要方法：
  - `hasToken()`
  - `getAuthUrl()`
  - `handleCallback()`
  - `getAuthenticatedToken()`
  - `clearToken()`
- 內部方法：
  - `loadToken()`
  - `saveToken()`
  - `refreshToken()`
  - `hydrateToken()`
  - `validateToken()`

---

## Server / Chat Sources

### [server/src/sources/chatSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/chatSource.ts)
- 多來源協調器。
- 補充：
  - `autoConnect` 流程下，Twitch live 時只要 Twitch 狀態為 `starting` 或 `running` 都會先停掉 YouTube primary/fallback，避免 YouTube 先於 Twitch 啟動
- 主要類別：
  - `ChatSourceOrchestrator`
- 主要方法：
  - `start()`
  - `stop()`
  - `getStatus()`
  - `notifyConfigChanged()`
  - `forceReconnect()`
  - `reconcile()`: 決定 primary / fallback / twitch 的切換。
  - `emitSystem()`
- 輔助函式：
  - `isTwitchRuntime()`

### [server/src/sources/youtubeApiSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/youtubeApiSource.ts)
- YouTube primary source。
- 主要職責：
  - 用 OAuth、REST、gRPC 抓 live chat。
  - 管理 REST quota 與 gRPC exhausted 冷卻。
  - 對 gRPC `FAILED_PRECONDITION (code=9)` 視為聊天室已關閉，進入 lookup 暫停冷卻（避免錯誤後持續重打 REST/gRPC）。
  - 管理 liveChat cache、reconnect、state persist。
  - 輸出 diagnostics JSONL。
- 主要類別：
  - `YouTubeApiSource`
- 主要方法：
  - `start()`
  - `stop()`
  - `getStatus()`
  - `forceReconnect()`
  - `canUseFallback()`
  - `runLoop()`
  - `lookupLiveChat()`
  - `openStream()`
  - `markFailure()`
  - `bumpReconnectCount()`
  - `enterRestCooldown()`
  - `enterGrpcCooldown()`
  - `ensurePersistedStateLoaded()`
  - `persistCooldownState()`
  - `ensureLiveChatCacheLoaded()`
  - `persistLiveChatCache()`
  - `getCachedLiveChat()`
  - `maybeFlushLookupMetrics()`
  - `logLoopSummary()`
  - `isRecentlyRecoveredFromGrpcExhausted()`
  - `shouldRunPostExhaustedSlowStart()`
- 主要輔助函式：
  - `formatSourceError()`
  - `isDailyQuotaExceededError()`
  - `isRestRateLimitError()`
  - `isGrpcResourceExhaustedError()`
  - `classifyExhaustedKind()`
  - `randomizeDelay()`
  - `shouldInvalidateLiveChatCache()`
  - `getRestCooldownBaseMs()`
  - `getRestCooldownMaxMs()`
  - `getGrpcCooldownBaseMs()`
  - `getNextStrike()`
  - `resetExpiredStrike()`
  - `readRestCooldownReason()`
  - `normalizeRestQuotaState()`
  - `normalizeGrpcCooldownState()`
  - `readErrorEndpoint()`
  - `sanitizePersistedCooldownState()`
  - `sanitizeRestQuotaForSave()`
  - `sanitizeGrpcCooldownForSave()`
  - `sanitizeNumber()`
  - `summarizeCooldownRecord()`
  - `analyzeSanitizedCooldownState()`

### [server/src/sources/youtubeFallbackSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/youtubeFallbackSource.ts)
- YouTube fallback source。
- 主要職責：
  - 直接抓取 live chat HTML / `ytInitialData`。
  - 多策略抽取結構化 payload。
  - 解析 chat / super chat。
  - 做本地 dedupe。
  - 記錄 unknown renderer、membership、paid evidence、missing continuation 診斷。
  - 提供 runtime summary 給 Settings 顯示。
- 主要類別：
  - `YouTubeFallbackSource`
- 主要方法：
  - `start()`
  - `stop()`
  - `getStatus()`
  - `getSummary()`
  - `notifyConfigChanged()`
  - `resetReconnectLock()`
  - `getTargetUrl()`
  - `shouldStopAfterWatchProbe()`
  - `runLoop()`
  - `flushBatchDiagnostics()`
  - `rememberDedupe()`
  - `bumpUnknownRendererCounts()`
  - `maybeNotifyAboutAvailableUpdate()`
- 主要輔助函式：
  - `extractFallbackBatch()`
  - `inferWatchPageLiveState()`
  - `normalizeWatchUrl()`
  - `isRecord()`
  - `extractStructuredChatBatch()`
  - `parseStructuredPayload()`
  - `collectStructuredChat()`
  - `walkStructured()`
  - `toFallbackChatEvent()`
  - `toFallbackSuperChatEvent()`
  - `resolveFallbackTimestamp()`
  - `readPublishedTimestamp()`
  - `parseAnyTimestamp()`
  - `extractContinuationFromRecord()`
  - `readRendererText()`
  - `readFirstRendererText()`
  - `parseDisplayAmount()`
  - `clampPollMs()`
  - `shouldIgnoreFallbackText()`
  - `resolveContinuationUrl()`
  - `normalizeFallbackUrl()`
  - `formatLogUrl()`
  - `extractAssignedJson()`
  - `extractExplicitInitialData()`
  - `extractInitialDataBySearch()`
  - `sliceBalancedJson()`
  - `createSafeDiagSnapshot()`
  - `toSafeDiagValue()`
  - `limitDiagPayloadSize()`
  - `collectPaidEvidence()`
  - `formatColorValue()`
  - `sanitizeFallbackInlineText()`
  - `sanitizeFallbackName()`
  - `sanitizeFallbackMessage()`
  - `sanitizeFallbackAmount()`
  - `stripHtmlTags()`
  - `pruneDedupeMap()`
  - `shortError()`
  - `summarizeTargetUrl()`
  - `isYoutubeHost()`

### [server/src/sources/twitchIrcSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/twitchIrcSource.ts)
- Twitch IRC source。
- 主要類別：
  - `TwitchIrcSource`
- 主要方法：
  - `start()`
  - `stop()`
  - `getStatus()`
  - `notifyConfigChanged()`
  - `forceReconnect()`
  - `connectOrWait()`
  - `handleLine()`
  - `handleUserNotice()`
  - `scheduleReconnect()`
  - `resolveCredentials()`
- 主要輔助函式：
  - `normalizeTwitchOauthToken()`
  - `parseTwitchChannel()`
  - `sanitizeDisplayName()`
  - `buildSubNoticeText()`
  - `displaySubAmount()`
  - `decodeTagValue()`
  - `parseIrcLine()`

---

## Server / YouTube Utilities

### [server/src/yt/restCall.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/yt/restCall.ts)
- YouTube REST 呼叫封裝。
- 主要函式：
  - `callYoutube()`
  - `describeResult()`
  - `readErrorDetails()`
  - `readHeader()`
  - `fingerprint()`
  - `stableStringify()`

### [server/src/yt/broadcasts.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/yt/broadcasts.ts)
- 查找直播對應的 `liveChatId`。
- 主要函式：
  - `findLiveChatByVideoUrl()`
  - `findActiveLiveChat()`
  - `extractYouTubeVideoId()`
  - `maybeFlushQuotaTotals()`
  - `getClientIdSuffix()`

### [server/src/yt/streamList.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/yt/streamList.ts)
- gRPC stream list 相關處理。
- 主要職責是用 proto/stream 描述抓取 live chat segment。

### [server/src/yt/emoji.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/yt/emoji.ts)
- YouTube 表情文字 / shortcode 正規化。
- 主要函式：
  - `normalizeYoutubeEmojiShortcodes()`

### [server/src/yt/byokStore.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/yt/byokStore.ts)
- 儲存與讀取 YouTube BYOK（clientId/clientSecret）設定的資料存取層。
- 供 `yt/oauth.ts` 與 `/auth/youtube/credentials` 相關 API 使用。

### [server/src/yt/types.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/yt/types.ts)
- YouTube 相關型別定義。

### [server/src/yt/proto/stream_list.proto](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/yt/proto/stream_list.proto)
- gRPC stream list 的 protobuf 定義。

---

## Server / Packs

### [server/src/packs/packRegistry.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/packs/packRegistry.ts)
- 已安裝角色包（friend pack）索引與啟用狀態管理。
- 提供 `/api/packs/list` 需要的清單輸出。

### [server/src/packs/packPaths.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/packs/packPaths.ts)
- pack 安裝路徑與目標資料夾路徑組裝。
- 提供 pack 匯入/刪除時的標準路徑定位。

### [server/src/packs/unzipSafe.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/packs/unzipSafe.ts)
- 以安全策略解析 zip 並寫入 pack 內容（避免路徑跳脫）。
- 主要函式：
  - `parseStoredZip()`
  - `writePackEntries()`

### [server/src/packs/validatePack.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/packs/validatePack.ts)
- 驗證 pack 結構與 manifest/子集設定是否合法。
- 主要函式：
  - `validatePackEntries()`

---

## Server / Assets

### [server/src/assets/types.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/assets/types.ts)
- 資產庫型別定義。

### [server/src/assets/store.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/assets/store.ts)
- 資產庫核心。
- 主要職責：
  - 初始化 user asset 結構。
  - 匯入單圖、序列圖、Spine。
  - 重建 index。
  - 儲存 bindings / accessories / sprite anim / emoji mappings。
  - 處理 emoji 包與單檔上傳。
  - 清理舊版匯入命名與修正舊 `spriteAnim` 路徑。
- 主要類別：
  - `AssetStore`
- 主要方法：
  - `ensureInitialized()`
  - `getLibraryPayload()`
  - `importFiles()`
  - `renameAsset()`
  - `setAssetInitialFlipX()`
  - `deleteAsset()`
  - `rebuildIndex()`
  - `saveBindings()`
  - `saveAccessories()`
  - `saveSpriteAnim()`
  - `saveSpineManifest()`
  - `saveEmojiMappings()`
  - `importEmojiPack()`
  - `saveEmojiFile()`
  - `migrateSpriteAnimAssetPaths()`
  - `normalizeSpriteAnimAssetPaths()`
  - `removeAssetReferences()`
  - `ensureInitializedBase()`
  - `writeJson()`
  - `looksLikeSpineFolder()`
  - `looksLikeSequence()`
  - `writeGroupedImport()`
  - `ensureSpineManifest()`
  - `buildMinimalSpineManifest()`
  - `inspectSpineDirectory()`
  - `extractSpineJsonManifestInfo()`
  - `writeSingleImport()`
  - `makeAssetId()`
  - `isRootAssetCandidate()`
  - `walk()`
  - `listRelativeFiles()`
- 主要輔助函式：
  - `mergeDefaultActions()`
  - `mergeDefaultBindings()`
  - `normalizeBindingClip()`
  - `normalizeRelativePath()`
  - `toPosix()`
  - `defaultDisplayName()`
  - `inferSequenceDisplayName()`
  - `slugify()`
  - `findPreviewPath()`
  - `dedupeAssets()`
  - `normalizeEmojiMappings()`
  - `dedupeEmojiEntries()`
  - `normalizeEmojiKey()`
  - `detectEmojiPlatform()`
  - `inferEmojiKey()`
  - `upsertEmojiMapping()`
- 補充：
  - 目前匯入命名策略已改成優先保留乾淨名稱，不再新增隨機序列號
  - 同名資料夾匯入時會覆蓋舊資料夾，而不是自動在名稱尾端加碼
  - server 啟動時會嘗試把舊版 `sprite-anims.json` 中帶亂碼前後綴的 `imageUrl/path` 清洗為現行乾淨檔名

---

## Server / Utilities

### [server/src/util/sanitize.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/sanitize.ts)
- 純文字清洗與 hash。
- 主要函式：
  - `sanitizePlainText()`
  - `escapeHtml()`
  - `hashText()`

### [server/src/util/diagLog.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/diagLog.ts)
- 診斷 JSONL append / rotate / health。
- 主要函式：
  - `appendJsonl()`
  - `getProcessBootId()`
  - `getDiagnosticsHealth()`
- 內部函式：
  - `isRecord()`
  - `rotateIfNeeded()`
  - `pruneOldRotatedFiles()`
  - `formatTimestampForFile()`
  - `nextRotateSuffix()`
  - `parseEnvInt()`
  - `recordDiagFailure()`

### [server/src/util/dedupe.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/dedupe.ts)
- 訊息去重快取。
- 主要類別：
  - `MessageDedupeCache`
- 主要方法：
  - `has()`
  - `mark()`
  - `remember()`
  - `prune()`

### [server/src/util/backoff.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/backoff.ts)
- Backoff 與 sleep 工具。
- 主要類別 / 函式：
  - `Backoff`
  - `sleep()`

### [server/src/util/stateStore.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/stateStore.ts)
- 同步式狀態檔存取工具。
- 主要函式：
  - `loadState()`
  - `saveState()`
  - `getStatePath()`

### [server/src/util/sourceStateStore.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/sourceStateStore.ts)
- 非同步來源狀態持久化工具。
- 主要類別：
  - `SourceStateStore`
- 主要方法：
  - `load()`
  - `scheduleSave()`
  - `getPath()`

### [server/src/util/youtubeDiagSummary.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/youtubeDiagSummary.ts)
- 組合 YouTube 診斷摘要（給 Settings 卡片）。
- 主要函式：
  - `readTailBytes()`
  - `parseJsonlLines()`
  - `buildYoutubeSummary()`
- 內部還包含多個 normalize / summarize / hints builder，用來把 state 與 diagnostics 整理成可讀摘要。

### [server/src/util/youtubeDiagSummary.testHarness.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/youtubeDiagSummary.testHarness.ts)
- 測試用 harness。
- 主要函式：
  - `runHarness()`

---

## Server / Update

### [server/src/update/checker.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/update/checker.ts)
- 檢查更新 manifest。
- 主要類別：
  - `AppUpdateChecker`
- 主要方法：
  - `checkForUpdate()`
  - `fetchManifest()`
- 輔助函式：
  - `compareVersions()`
  - `normalizeVersion()`

---

## Runtime / Generated Data（非 user-assets）

### [server/public/index.html](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/public/index.html)
- 前端 build 後輸出的主頁 HTML。

### [server/public/overlay.html](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/public/overlay.html)
- 前端 build 後輸出的 Overlay HTML。
- OBS 目前可直接載入這份檔案（由 41928 bridge server 對外提供）。

### [server/public/settings.html](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/public/settings.html)
- 前端 build 後輸出的 Settings HTML。

### [server/public/vite.svg](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/public/vite.svg)
- build 後輸出的靜態圖示資產。

### [server/public/assets/*](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/public/assets/)
- Vite build 產生的 JS / CSS chunk。
- 實際檔名會因 build 而改變，故不逐檔列出。

### [server/public/app-update.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/public/app-update.json)
- 更新 manifest 靜態檔。

### [server/data/state/youtube_livechat_cache.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/state/youtube_livechat_cache.json)
- YouTube liveChat cache 狀態。

### [server/data/state/youtube_cooldowns.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/state/youtube_cooldowns.json)
- YouTube REST / gRPC 冷卻狀態。

### [server/data/state/youtube_channel_id.json](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/state/youtube_channel_id.json)
- 已解析的 YouTube 自身 channelId 快取（降低重複查詢頻率）。

### [server/data/diagnostics/youtube_rest.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_rest.jsonl)
- REST 診斷事件流。

### [server/data/diagnostics/youtube_grpc.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_grpc.jsonl)
- gRPC 診斷事件流。

### [server/data/diagnostics/youtube_lookup.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_lookup.jsonl)
- liveChat lookup 診斷事件流。

### [server/data/diagnostics/youtube_clients_gate.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_clients_gate.jsonl)
- overlay client gate / source 啟停診斷。

### [server/data/diagnostics/youtube_concurrency.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_concurrency.jsonl)
- 來源併發與門檻診斷。

### [server/data/diagnostics/youtube_cooldown.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_cooldown.jsonl)
- 冷卻進入 / 恢復診斷。

### [server/data/diagnostics/youtube_fallback_dedupe.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_fallback_dedupe.jsonl)
- fallback dedupe hit/miss 診斷。

### [server/data/diagnostics/youtube_fallback_unknown.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_fallback_unknown.jsonl)
- fallback 解析遇到未知 renderer 類型時的診斷紀錄。

### [server/data/diagnostics/youtube_fallback_membership_seen.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/youtube_fallback_membership_seen.jsonl)
- fallback 偵測到 membership 相關訊號的診斷紀錄。

### [server/data/diagnostics/gRPC_status_log.jsonl](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/data/diagnostics/gRPC_status_log.jsonl)
- lookup / gRPC 視窗統計與狀態摘要日誌（例如 per-window metrics）。

---

## 總結

這個專案已經不只是單純的聊天室 Overlay，而是一個完整的「直播互動渲染工作台」：

- 後端負責來源整合、節流、診斷、資產 API 與設定持久化。
- 前端 Overlay 負責即時角色演出與互動工具。
- Settings 頁面則是高度產品化的內容編輯器與診斷控制台。

如果之後要繼續維護，最重要的三個核心檔案是：

1. [src/overlay.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/overlay.ts)
2. [src/settings.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/settings.ts)
3. [server/src/index.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/index.ts)

而 YouTube 行為核心則集中在：

1. [server/src/sources/youtubeApiSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/youtubeApiSource.ts)
2. [server/src/sources/youtubeFallbackSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/youtubeFallbackSource.ts)
3. [server/src/util/youtubeDiagSummary.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/youtubeDiagSummary.ts)

---

## Core Data Structures

本節整理未來開發最常碰到的核心資料模型。型別來源主要為：
- [src/shared/assetMapping.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/assetMapping.ts)
- [server/src/assets/types.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/assets/types.ts)
- [src/shared/channel.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/channel.ts)

### 1) `emoji-mappings.json`

TypeScript:

```ts
type EmojiMappingEntry = {
  key: string;
  file: string;
  scale?: number;
  aliases?: string[];
};

type EmojiMappingsConfig = {
  schema: 1;
  platforms: Record<"youtube" | "twitch" | "7tv", EmojiMappingEntry[]>;
};
```

JSON 範例:

```json
{
  "schema": 1,
  "platforms": {
    "youtube": [
      { "key": "hand-pink-waving", "file": "/default-assets/emoji/youtube/global/yt-hand-pink-waving.png" }
    ],
    "twitch": [
      { "key": "Kappa", "file": "/default-assets/emoji/twitch/global/twitch-25-kappa.png" }
    ],
    "7tv": [
      { "key": "OMEGALUL", "file": "/user-assets/emojis/7tv/channel-123456/7tv-abc-omegalul.webp", "aliases": ["OMEGA"] }
    ]
  }
}
```

欄位用途:
- `schema`: 版本標記，目前固定 `1`
- `platforms`: 各平台 emote 清單
- `key`: 聊天訊息中的比對 token
- `file`: 可被前端載入的資產路徑（支援 `/default-assets/...`、`/user-assets/...`、`/packs/...` 或 URL）
- `aliases`: 同義 token（可選）
- `scale`: 局部縮放（可選）

維護方式:
- 可被使用者修改：可（Settings JSON editor / API）
- 程式自動生成：會（匯入 emoji pack、單檔上傳、Twitch/7TV 匯入）

### 2) Library Payload（`GET /api/assets`、多數寫入 API 的回傳）

TypeScript:

```ts
type AssetLibraryPayload = {
  assets: AssetRecord[];
  actions: ActionSpec[];
  bindings: BindingConfig;
  accessories: AccessoriesConfig;
  emojiMappings: EmojiMappingsConfig;
};
```

JSON 範例:

```json
{
  "assets": [
    {
      "assetId": "image:user:abc123",
      "source": "user",
      "type": "image",
      "displayName": "hero_idle",
      "path": "imports/hero_idle.png",
      "preview": "imports/hero_idle.png"
    }
  ],
  "actions": [{ "id": "idle", "kind": "loop", "label": "Idle" }],
  "bindings": { "schema": 1, "targetCharacterId": "hero", "actionSetId": "default", "base": { "idle": { "type": "none" } }, "overrides": [] },
  "accessories": { "schema": 1, "targetCharacterId": "hero", "base": [], "overrides": [] },
  "emojiMappings": { "schema": 1, "platforms": { "youtube": [], "twitch": [], "7tv": [] } }
}
```

維護方式:
- 可被使用者修改：間接（透過 API）
- 程式自動生成：是（`AssetStore.getLibraryPayload()`）

### 3) Asset Index（`user-assets/index.json`）

TypeScript: `AssetRecord[]`

```ts
type AssetRecord = {
  assetId: string;
  source?: "default" | "user" | "pack";
  type: "image" | "apng" | "spritesheetSeq" | "spine" | "spriteAnim";
  displayName: string;
  path: string;
  preview?: string;
  frames?: string[];
  imageUrl?: string;
  grid?: SpriteAnimGrid;
  playback?: SpriteAnimPlayback;
  manifest?: { ... };
};
```

JSON 範例:

```json
[
  {
    "assetId": "spritesheetSeq:user:86c7...",
    "source": "user",
    "type": "spritesheetSeq",
    "displayName": "walk",
    "path": "imports/walk",
    "preview": "imports/walk/001.png",
    "frames": ["imports/walk/001.png", "imports/walk/002.png"]
  }
]
```

維護方式:
- 可被使用者直接修改：不建議
- 程式自動生成：是（`rebuildIndex()`）

### 4) `bindings.json`

TypeScript: `BindingConfig`

JSON 範例:

```json
{
  "schema": 1,
  "targetCharacterId": "hero",
  "actionSetId": "default",
  "base": {
    "idle": { "type": "spineAnim", "assetId": "spine:user:...", "anim": "Idle", "loop": true }
  },
  "overrides": [
    {
      "when": { "isMember": true, "groupLabel": "VIP" },
      "set": { "chat": { "type": "spineAnim", "assetId": "spine:user:...", "anim": "Talk", "loop": false } }
    }
  ]
}
```

維護方式:
- 可被使用者修改：可（Settings）
- 程式自動生成：部分（預設檔、clip normalize、刪素材時清理引用）

### 5) `accessories.json`

TypeScript: `AccessoriesConfig`

JSON 範例:

```json
{
  "schema": 1,
  "targetCharacterId": "hero",
  "base": [
    {
      "id": "base-accessory",
      "clip": { "type": "image", "file": "imports/hat.png", "scale": 1 },
      "anchor": "actor",
      "offset": { "x": 0, "y": -120 },
      "scale": 1,
      "zIndex": 50
    }
  ],
  "overrides": []
}
```

維護方式:
- 可被使用者修改：可
- 程式自動生成：部分（預設檔、刪素材時清理引用）

### 6) `sprite-anims.json`

TypeScript: `AssetRecord[]`（僅 `type === "spriteAnim"`）

JSON 範例:

```json
[
  {
    "assetId": "spriteAnim:user:3f0f...",
    "type": "spriteAnim",
    "displayName": "run",
    "path": "imports/run-sheet.png",
    "imageUrl": "imports/run-sheet.png",
    "preview": "imports/run-sheet.png",
    "grid": {
      "cellW": 128,
      "cellH": 128,
      "offsetX": 0,
      "offsetY": 0,
      "spacingX": 0,
      "spacingY": 0,
      "startIndex": 0,
      "frameCount": 8,
      "order": "rowMajor"
    },
    "playback": { "fps": 12, "loop": true },
    "anchor": { "x": 0.5, "y": 1 }
  }
]
```

維護方式:
- 可被使用者修改：可（透過 Sprite Animation Editor）
- 程式自動生成：會（儲存時 normalize 路徑，且會併入 index）

---

## Asset System Architecture

核心來源分三層：`default`、`user`、`pack`，最後合併到同一份 `library payload`。

| 類型 | 主要路徑 | 可寫 | 可刪 | rebuildIndex | 出現在 library payload |
|---|---|---|---|---|---|
| default assets | `server/default-assets/*`（packaged 時 `resources/server/default-assets/*`） | 否 | 否 | 不走 user rebuild，但會被 `readDefaultAssets()` 掃描 | 會（`source: "default"`） |
| user assets | `APP_USER_DATA_DIR/user-assets/*` | 是 | 是 | 需要（匯入/刪除後） | 會（`source: "user"`） |
| pack assets | `APP_USER_DATA_DIR/packs/<packId>/...` | 安裝/刪除流程寫入 | 可刪整包 | 不走一般 `rebuildIndex`，由 `PackRegistry.reload()` | 會（`source: "pack"`） |

路徑示例:

```text
default imports:      /default-assets/imports/hero_idle.png
default emoji:        /default-assets/emoji/twitch/global/twitch-25-kappa.png
user imports:         /user-assets/imports/custom/hero_walk.png
user emoji:           /user-assets/emojis/7tv/channel-12345/7tv-abc-omegalul.webp
pack asset (served):  /packs/com.example.pack/assets/imports/npc_idle.png
```

補充規則:
- `default` 資產邏輯上唯讀，server 端 `rename/delete/flip` 會拒絕。
- `user` 端才是匯入、刪除、客製化 mapping 的落點。
- `GET /api/assets` 回傳時，`assets` 已是 default + user（pack 由 settings/overlay 另外合併 active packs）。
- `/user-assets` 路由含一層向 `default-assets` 的 fallback（兼容舊路徑），但新資料應用明確路由：`/default-assets`、`/user-assets`、`/packs`。

---

## Emoji / Emote Mapping System

主要實作位於 [server/src/assets/store.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/assets/store.ts) 與 [src/shared/assetMapping.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/assetMapping.ts)。

### 核心流程

1. 讀取 default + user 的 mappings
2. `normalizeEmojiMappings()`：各平台去重、標準化
3. `mergeEmojiMappings()`：user 覆蓋/補入 default
4. 前端用 `findEmojiMapping()` 查 token -> file

### 關鍵函式行為

- `normalizeEmojiMappings(input)`
  - 確保 schema=1，平台齊全：`youtube/twitch/7tv`
  - 對每個平台跑 `dedupeEmojiEntries`

- `dedupeEmojiEntries(entries, platform)`
  - 以 `normalizeEmojiKey(platform, entry.key)` 作唯一鍵去重
  - 清理無效 key/file，排序輸出
  - aliases 也會 normalize

- `upsertEmojiMapping(target, entry, platform)`
  - 以「平台內 normalized key」為 upsert 主鍵
  - 同 key 新值會覆蓋舊值

- `saveEmojiMappings(mappings)`
  - 寫入 `user-assets/emoji-mappings.json`
  - default mapping 不在這裡寫（default mapping 走 default-assets 下的 JSON）

### mapping 唯一鍵與衝突策略

- 唯一鍵不是全域檔名，而是「**平台 + normalized key**」
- `key` 仍是主要比對 token（因此同平台同 key 會互蓋）
- 檔案落地命名已保留 `platform + emoteId + key`，減少重名覆蓋風險
- 目前 `findEmojiMapping()` 在 `platform=twitch` 時，找不到會 fallback 查 `platform=7tv`

### alias 與 asset path

- `aliases` 已支援，查詢時會同樣 normalize 後比對
- `file` 可是：
  - relative (`emojis/...`)
  - absolute asset path (`/default-assets/...`, `/user-assets/...`, `/packs/...`)
  - 外部 URL

---

## Settings UI Architecture

主檔：[src/settings.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/settings.ts)

### 主要 UI 區塊與核心 state

| 區塊 | 主要 state |
|---|---|
| Status/Auth | `hasGoogleLogin`, `hasTwitchLogin`, `fallbackLiveUrl`, `fallbackAutoConnect` |
| Overlay 調整 | `settings`（`OverlaySettings`） |
| Asset Library | `assetIndex`, `assetImages`, `installedPacks` |
| Action/Accessory Mapping | `bindings`, `accessories`, `actionSpecs`, `selectedNameOverrideIndex` |
| Emoji Mapping | `emojiMappings`, `emojiUiGroups` |
| SpriteAnim Editor | `spriteAnimEditor`, `spriteAnimLoadedImage` |
| Diagnostics | `youtubeDiagnosticsSummary`, `youtubeFallbackSummary` 等 |

### 重要按鈕 -> API 對照

| UI 操作 | API |
|---|---|
| 匯入素材檔/資料夾 | `POST /api/assets/import` |
| 重建索引 | `POST /api/assets/rebuild-index` |
| 刪除素材 | `POST /api/assets/delete` |
| 重命名素材 | `POST /api/assets/rename` |
| 設定初始翻轉 | `POST /api/assets/flip` |
| 儲存 Spine manifest | `POST /api/assets/spine/manifest` |
| 儲存 spriteAnim | `POST /api/assets/sprite-anim` |
| 儲存動作/配件/emoji mappings | `POST /api/mappings` |
| 匯入 emoji pack | `POST /api/assets/import-emoji-pack` |
| 上傳單一 emoji | `POST /api/assets/import-emoji-file` |
| Twitch 區塊「匯入7TV」 | `POST /api/emotes/7tv/import`（先 `GET /auth/twitch/status` 檢查） |
| Twitch 登入 | `POST /auth/twitch/device/start` + `POST /auth/twitch/device/poll` |
| Twitch 登出 | `POST /auth/twitch/logout` |
| YouTube BYOK 設定 | `GET/POST/DELETE /auth/youtube/credentials` |
| 重新連線聊天來源 | `POST /api/reconnect-chat` |
| 儲存 runtime URL/autoConnect | `POST /config` |
| 讀 bridge 狀態 | `GET /status` |

---

## Server API Overview

主檔：[server/src/index.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/index.ts)

### Auth / Runtime

| Method | Path | 作用 | 主要模組 |
|---|---|---|---|
| GET | `/status` | runtime + source + auth + overlay client 數 | `ChatSourceOrchestrator`, `AppConfigStore` |
| POST | `/config` | 更新 runtime（videoUrl/fallbackUrl/autoConnect） | `AppConfigStore.updateRuntime` |
| POST | `/api/reconnect-chat` | 強制重連來源 | `ChatSourceOrchestrator.forceReconnect` |
| GET | `/auth/youtube/credentials` | 讀 BYOK 狀態 | `GoogleOAuthService` |
| POST | `/auth/youtube/credentials` | 寫 BYOK | `GoogleOAuthService` |
| DELETE | `/auth/youtube/credentials` | 清 BYOK | `GoogleOAuthService` |
| POST | `/auth/twitch/device/start` | Twitch device flow 開始 | `TwitchOAuthService.startDeviceAuth` |
| POST | `/auth/twitch/device/poll` | Twitch device flow 輪詢 | `TwitchOAuthService.pollDeviceAuth` |
| GET | `/auth/twitch/status` | Twitch token/user 狀態 | `TwitchOAuthService.getDeviceAuthStatus` |
| POST | `/auth/twitch/logout` | Twitch 登出 | `TwitchOAuthService.clearToken` |

### Assets / Mappings / Emotes

| Method | Path | 作用 | 主要模組 |
|---|---|---|---|
| GET | `/api/assets` | 回傳 library payload | `AssetStore.getLibraryPayload` |
| GET | `/api/assets/images` | 列出可用圖片 | `scanPublicPngs` |
| POST | `/api/assets/import` | 匯入一般素材 | `AssetStore.importFiles` |
| POST | `/api/assets/rebuild-index` | 重建 user index | `AssetStore.rebuildIndex` |
| POST | `/api/assets/rename` | 改名 | `AssetStore.renameAsset` |
| POST | `/api/assets/delete` | 刪素材 | `AssetStore.deleteAsset` |
| POST | `/api/assets/flip` | 設初始翻轉 | `AssetStore.setAssetInitialFlipX` |
| POST | `/api/assets/sprite-anim` | 儲存 spriteAnim | `AssetStore.saveSpriteAnim` |
| POST | `/api/assets/spine/manifest` | 儲存 spine manifest | `AssetStore.saveSpineManifest` |
| POST | `/api/assets/import-emoji-pack` | 匯入 emoji pack | `AssetStore.importEmojiPack` |
| POST | `/api/assets/import-emoji-file` | 匯入單檔 emoji | `AssetStore.saveEmojiFile` |
| POST | `/api/emotes/7tv/import` | 用 Twitch user 匯入 7TV emotes | `import7tvChannelEmotes` |
| POST | `/api/mappings` | 儲存 bindings/accessories/emojiMappings | `AssetStore.save*` |

### Packs / Diagnostics / Update

| Method | Path | 作用 | 主要模組 |
|---|---|---|---|
| GET | `/api/packs/list` | 已安裝 pack 清單 | `PackRegistry.list` |
| POST | `/api/packs/import` | 安裝 pack zip | `parseStoredZip`, `validatePackEntries`, `PackRegistry.reload` |
| POST | `/api/packs/delete` | 刪 pack | `PackRegistry.reload` |
| GET | `/api/diagnostics/youtube/summary` | YouTube summary | `buildYoutubeSummary` |
| GET | `/api/diagnostics/youtube-fallback/summary` | fallback summary | `YouTubeFallbackSource.getSummary` |
| GET | `/api/diagnostics/health` | diagnostics health | `getDiagnosticsHealth` |
| GET | `/api/update-status` | 更新檢查 | `AppUpdateChecker` |

---

## Event Flow

### a) 匯入素材流程（一般素材）

```text
Settings UI (drag/drop or import button)
  -> POST /api/assets/import
  -> AssetStore.importFiles()
  -> AssetStore.rebuildIndex()
  -> AssetStore.getLibraryPayload()
  -> WS broadcast SETTINGS_UPDATE + HTTP response payload
  -> settings/overlay apply latest library
```

### b) emoji mapping 儲存流程

```text
Settings UI editor/table
  -> POST /api/mappings { emojiMappings }
  -> AssetStore.saveEmojiMappings()
  -> AssetStore.getLibraryPayload() (default+user merge)
  -> broadcast SETTINGS_UPDATE
```

### c) Twitch OAuth 登入流程（device flow）

```text
Settings Twitch Login
  -> POST /auth/twitch/device/start
  -> user opens verification URL and enters code
  -> Settings interval poll: POST /auth/twitch/device/poll
  -> TwitchOAuthService.saveToken()
  -> refreshSourcesForOverlayDemand("notify")
  -> orchestrator reconcile
```

### d) Chat message -> overlay render 流程

```text
YouTubeApiSource / YouTubeFallbackSource / TwitchIrcSource
  -> emit ChatEvent to WsHub.broadcastEvent()
  -> overlay websocket receives CHAT_MESSAGE/SUPER_CHAT
  -> overlay enqueueChat()
  -> actor model created/updated
  -> BoxRenderer resolves asset mapping + emoji mapping
  -> frame update renders actor + bubble + emoji
```

---

## Invariants / System Rules

- 所有可持久化資產路徑必須是可預期路徑字串，並以 POSIX 風格處理（`toPosix` / normalize）。
- `default assets` 視為唯讀：server 端 rename/delete/flip 會拒絕。
- 一般素材新增/刪除後必須 `rebuildIndex()`，否則 `index.json` 與實體檔案會不一致。
- `library payload` 是前端 settings/overlay 同步的唯一資料來源。
- emoji key 寫入前一定會 normalize（平台敏感）。
- emoji mapping 寫入採 upsert（平台內 key 唯一），不是 append-only。
- `twitch` chat platform 與 `7tv` emoji platform 是不同層：
  - WebSocket chat event 平台目前仍是 `youtube | twitch`
  - `7tv` 只在 emoji mapping 層使用（並可被 twitch token fallback 查找）
- runtime 可寫資料不能寫回安裝目錄；Electron 模式下應寫入 `app.getPath("userData")`。

---

## External Integrations

### YouTube
- API:
  - YouTube Data API（broadcast/liveChat lookup）
  - gRPC stream（`server/src/yt/streamList.ts`）
  - HTML fallback parsing（`YouTubeFallbackSource`）
- channelId/userId 來源:
  - OAuth token + `findMyChannelId`
  - 或由 live URL 解析
- 內部轉換:
  - 統一轉成 `ChatEvent`（`CHAT_MESSAGE` / `SUPER_CHAT`）

### Twitch
- API/Protocol:
  - OAuth Device Code flow
  - Helix users/streams（live status）
  - IRC over WebSocket（聊天室）
- userId 來源:
  - token validate / users endpoint 填 `user_id`
- 內部轉換:
  - Twitch IRC `PRIVMSG/USERNOTICE` -> `ChatEvent`

### 7TV
- API:
  - `GET https://7tv.io/v3/users/twitch/{twitchUserId}`
- userId 來源:
  - 由 `TwitchOAuthService` 取目前登入 `user_id`
- 內部轉換:
  - 7TV set emotes -> `RemoteEmojiInput[]`
  - 寫入 `user-assets/emojis/7tv/...`
  - upsert 到 `emoji-mappings.json` 的 `platforms["7tv"]`

### BTTV / FFZ（現況）
- 目前尚未接入正式匯入流程。
- 建議插入點:
  - Server: 比照 `server/src/twitch/emotes.ts` 增加 provider client 與 import API
  - Assets: 重用 `AssetStore.importRemoteEmotes()`
  - Settings: 在 Twitch emoji 區塊新增對應 import 按鈕與 API 呼叫

---

## File Ownership Map

| 檔案 | 責任 |
|---|---|
| [server/src/index.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/index.ts) | Server bootstrap、路由註冊、source/orchestrator 組裝、WS 廣播、API 協調 |
| [server/src/assets/store.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/assets/store.ts) | 資產生命週期（匯入、索引、刪除、mapping 儲存、default/user merge） |
| [server/src/twitch/oauth.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/twitch/oauth.ts) | Twitch OAuth token/device flow/live status |
| [server/src/twitch/emotes.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/twitch/emotes.ts) | Twitch global emote 初始化、7TV 匯入 |
| [server/src/sources/chatSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/chatSource.ts) | 多來源協調（YouTube primary/fallback/Twitch 切換） |
| [server/src/sources/youtubeApiSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/youtubeApiSource.ts) | YouTube 主來源、cooldown/cache/diagnostics |
| [server/src/sources/youtubeFallbackSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/youtubeFallbackSource.ts) | YouTube fallback HTML parser |
| [server/src/sources/twitchIrcSource.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/sources/twitchIrcSource.ts) | Twitch IRC 連線與事件轉換 |
| [server/src/util/appPaths.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/util/appPaths.ts) | 開發/打包模式的路徑決策（public/default/userData/state/diag/packs） |
| [server/src/ws/hub.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/server/src/ws/hub.ts) | WS client 管理、event/legacy event 廣播、overlay client gate |
| [src/settings.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/settings.ts) | Settings UI、狀態管理、所有管理 API 呼叫入口 |
| [src/overlay.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/overlay.ts) | Overlay runtime、actor/tool interaction、消息渲染 |
| [src/shared/assetMapping.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/assetMapping.ts) | 前後端共用 mapping/binding/accessory/emoji 型別與解析規則 |
| [src/shared/channel.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/src/shared/channel.ts) | `OverlaySettings` 與跨頁事件格式 |
| [electron/main.ts](/mnt/c/WebDevelope/CuChat/fan-chat-overlay/electron/main.ts) | Electron 主程序，啟 server 子程序 + 載入 settings URL |
