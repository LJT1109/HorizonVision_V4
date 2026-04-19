# HorizonVision V4

適用於 HoloLens 2 的混合實境應用程式，透過 QR Code 掃描將動態內容錨定於真實世界空間中顯示。

---

## 目錄

- [專案簡介](#專案簡介)
- [技術架構](#技術架構)
- [專案結構](#專案結構)
- [場景說明](#場景說明)
- [系統模組](#系統模組)
- [操作流程](#操作流程)
- [操作模式](#操作模式)
- [後端 API](#後端-api)
- [開發環境](#開發環境)

---

## 專案簡介

HorizonVision 是一套混合實境內容展示系統，使用者透過 HoloLens 2 掃描場地中的 QR Code，應用程式即會從後端伺服器下載對應的場景資料（文字、圖片、影片），並將內容疊加錨定於真實空間中顯示。系統支援「編輯模式」與「預覽模式」，可即時調整物件位置並同步回伺服器。

---

## 技術架構

| 項目 | 內容 |
|------|------|
| 引擎 | Unity 2022.3.18f1 |
| 目標平台 | HoloLens 2（Mixed Reality） |
| 空間追蹤 | Microsoft OpenXR + World Locking Tools |
| 輸入系統 | MRTK 2.8.2 手部追蹤、Pointer 系統 |
| UI 框架 | TextMesh Pro、Canvas UI + MRTK 按鈕 |
| 後端 | PHP API（horizonvision.ljthub.com） |
| 內容格式 | JSON 節點系統，支援 SHA256 驗證下載 |

---

## 專案結構

```
HorizonVision_Unity/
├── Assets/
│   ├── Scenes/                  # Unity 場景檔
│   ├── Scripts/                 # 核心腳本
│   │   └── QRCode/              # QR Code 追蹤相關腳本
│   ├── ContentDownloader/       # 動態內容下載系統
│   ├── Prefabs/                 # 可重用物件預製體
│   ├── MRTK/                    # Mixed Reality Toolkit 設定
│   ├── XR/                      # XR 平台設定
│   └── TextMesh Pro/            # 進階文字渲染系統
├── Packages/                    # Unity 套件依賴
└── ProjectSettings/             # 專案設定檔
```

---

## 場景說明

### 1. `main.unity`（主場景）

應用程式主要運作場景，包含：
- HoloLens 2 互動 UI 面板與按鈕
- QR Code 掃描啟動入口
- 動態內容顯示區域

**進入方式：** 應用程式啟動後預設載入此場景。

---

### 2. `toturial.unity`（教學場景）

新手引導場景，說明基本操作方式與系統功能。

---

### 3. `QRCodesSample.unity`（QR Code 測試場景）

QR Code 追蹤功能的獨立測試場景，可在此場景驗證：
- QR Code 偵測是否正常
- 空間錨定是否準確
- 追蹤事件（新增、更新、移除）

---

### 4. `ContentDownloader.unity`（下載測試場景）

用於測試後端 API 連線與內容下載功能的開發場景。

---

## 系統模組

### QR Code 追蹤系統

| 腳本 | 功能 |
|------|------|
| `QRCodesManager.cs` | QR Code 追蹤管理器（Singleton），管理掃描器生命週期與事件派發 |
| `QRCode.cs` | 單一 QR Code 物件，顯示 ID、資料、尺寸，偵測 URL 並觸發點擊事件 |
| `QRCodesVisualizer.cs` | 依事件佇列實例化／移除 QR Code 視覺物件 |
| `SpatialGraphNodeTracker.cs` | 將虛擬物件的 Transform 綁定至真實空間中 QR Code 的位置 |
| `QRCodesSetup.cs` | QR Code 系統初始化，可設定是否自動啟動 |
| `FollowQrcode.cs` | 讓指定物件跟隨帶有 "Root" Tag 的 QR Code 移動 |

**QR Code 事件流程：**

```
掃描環境 → QRCodeAdded 事件 → 實例化視覺物件
                            → SpatialGraphNodeTracker 更新座標
                            → 若為 URL → 觸發 onURLFound → 下載場景內容
```

---

### 動態內容系統

| 腳本 | 功能 |
|------|------|
| `Config.cs` | 全域設定管理器（Singleton），管理 Token、模式、下載進度、字型設定 |
| `ContentDownloader.cs` | 向後端發送 POST 請求，取得場景 JSON 資料 |
| `NodeCreator.cs` | 依 RawScene 資料建立場景節點層級（Singleton） |
| `Node.cs` | 單一內容節點，支援文字、圖片、影片，具前後節點鏈結 |
| `NodeSpawner.cs` | 實例化並定位文字、圖片、影片等視覺元素 |
| `RawScene.cs` | 後端 JSON 資料模型（RawScene、RawNode、TextSpace、MediaSpace） |

**內容節點結構：**

```
Scene
└── Groups（群組）
    ├── Head Node（進入點）
    └── Body Nodes（內容序列：文字 / 圖片 / 影片）
        → Node 之間以 prev/next 鏈結串聯
```

---

### 攝影機控制系統

| 腳本 | 功能 |
|------|------|
| `LazyFollow.cs` | 以 Lerp 平滑追蹤目標物件，保持固定距離 |
| `LockCameraHeight.cs` | 可鎖定單一或多個軸向的跟隨，適用於限制高度的場景 |

---

### 工具腳本

| 腳本 | 功能 |
|------|------|
| `SceneManager.cs` | Unity 場景切換的輕量封裝 |
| `PanelSize.cs` | 同步 RectTransform 尺寸至 BoxCollider，確保 UI 點擊判定準確 |
| `LogToTMP.cs` | 將 QR Code 追蹤狀態即時顯示於 TextMeshPro 介面，供開發除錯 |
| `Singleton.cs` | 泛型 Singleton 基底類別（DontDestroyOnLoad） |

---

## 操作流程

### 首次啟動

```
1. 佩戴 HoloLens 2，啟動 HorizonVision 應用程式
2. 系統初始化 QR Code 追蹤器
3. 授予「空間資料存取」與「QR Code 掃描」權限
4. 應用程式載入主場景
```

### 掃描 QR Code 並載入內容

```
1. 將視線或手部光線對準 QR Code
2. 系統自動偵測並顯示 QR Code 邊框（藍色 = 有效 URL）
3. 點擊 QR Code 或自動觸發 URL 解析
4. URL 格式：https://horizonvision.ljthub.com/token/mode/
   - token：後端認證碼
   - mode：edit（編輯）或 preview（預覽）
5. Config 系統解析 Token 與模式
6. ContentDownloader 向後端請求對應場景資料
7. NodeCreator 依資料建立 3D 內容節點
8. 內容錨定於 QR Code 對應的真實空間位置顯示
```

### 與內容互動

```
- 手部光線（遠距）：指向並點擊 UI 按鈕或 QR Code
- 手部直接接觸（近距）：直接按壓 HoloLens 2 實體按鈕
- 滑動手勢：捲動文字內容或切換節點
```

### 編輯模式（Edit Mode）下調整物件位置

```
1. QR Code URL 中 mode = edit 時進入編輯模式
2. 畫面顯示額外的編輯 UI 元素（Tag 為 "Edit" 的物件）
3. 使用手部抓取手勢移動內容節點
4. 放開後系統自動呼叫 updateTransform.php 同步位置至伺服器
```

---

## 操作模式

| 模式 | 說明 |
|------|------|
| `edit` | 編輯模式：顯示編輯 UI，可移動物件並即時同步回伺服器 |
| `preview` | 預覽模式：隱藏編輯 UI，純瀏覽互動，適合展示用途 |
| `none` | 停用狀態：觸發 `OnNone` 事件，系統進入閒置 |

---

## 後端 API

**Base URL：** `https://horizonvision.ljthub.com/bg/`

| 端點 | 方法 | 說明 |
|------|------|------|
| `getPageInfo.php` | POST | 以 Token 取得場景 JSON 資料（RawScene 格式） |
| `updateTransform.php` | POST | 提交節點的 Transform 更新（位置、旋轉、縮放） |

**回傳資料結構（RawScene）：**

```json
{
  "success": true,
  "groups": [...],
  "heads": [...],
  "bodys": [...]
}
```

---

## 開發環境

### 必要環境

- Unity 2022.3.18f1
- Visual Studio 2022 或 JetBrains Rider
- Windows 10/11（HoloLens 2 部署需要）
- HoloLens 2 裝置或模擬器

### 主要套件依賴

| 套件 | 版本 |
|------|------|
| Mixed Reality Toolkit Foundation | 2.8.2 |
| Mixed Reality OpenXR Plugin | 1.5.0 |
| World Locking Tools | 1.5.9 |
| XR Management | 4.4.0 |
| OpenXR Plugin | 1.9.1 |
| TextMesh Pro | 3.0.8 |

### 建置至 HoloLens 2

```
1. File → Build Settings → 選擇 Universal Windows Platform
2. Target Device：HoloLens
3. Architecture：ARM64
4. Build → 開啟 Visual Studio 方案
5. 部署至裝置或 HoloLens 模擬器
```
