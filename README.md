
##  Pixel Yahtzee v0.7

一個基於像素風格、支援遠端連線 (P2P) 的經典 Yahtzee 骰子遊戲。

###  版本 v0.7 更新亮點

* **動態計分系統**：表格下方新增實時總分 (Total) 與排名 (Rank) 計算。
* **結算畫面優化**：遊戲結束時，計分板會保留在背景，並以半透明覆蓋層顯示最終排名，方便玩家對帳。
* **行動裝置適配**：針對手機螢幕進行佈局優化，採上下分欄設計並縮放骰子尺寸。
* **加寬記分板**：側邊欄空間擴大，提升閱讀性與操作精確度。

###  技術棧

* **Frontend**: HTML5, CSS3 (Flexbox/Grid), Vanilla JavaScript.
* **Font**: Press Start 2P (Google Fonts).
* **Networking**: PeerJS (WebRTC) - 提供瀏覽器對瀏覽器的點對點連線。
* **Icons**: Pure CSS/Text-based Pixel Art.

###  如何開始

1. **開啟遊戲**：直接以瀏覽器開啟 `index.html`。
2. **本地模式**：點擊 "LOCAL MULTI"，可手動新增多個玩家進行同台裝置遊玩。
3. **線上模式**：
* **Host (房主)**：點擊 "ONLINE P2P"，將生成的 **ROOM ID** 分享給朋友。
* **Join (加入者)**：輸入房主的 ID 並點擊 "JOIN"。


4. **遊戲規則**：每回合最多擲骰 3 次，點擊骰子可保留 (Keep)，最後在左側記分板選擇一個類別填入分數。

---

# 📡 P2P 技術實作深度解析

本遊戲的連線邏輯是基於 **WebRTC (Web Real-Time Communication)** 技術，並透過 **PeerJS** 簡化信令交換 (Signaling) 流程。

### 1. 網路架構：星狀拓樸 (Star Topology)

雖然是 P2P，但為了保持數據一致性，我們採用了 **"Host-Client"** 的邏輯模型：

* **Host (房主)**：作為運算核心。負責驗證玩家動作、計算隨機骰子數值、維護全域 `players` 陣列狀態。
* **Client (玩家)**：作為終端。將操作 (如點擊骰子、選擇分數) 發送給 Host，並接收來自 Host 的全域狀態更新。

### 2. 連線建立流程

1. **信令階段**：透過 PeerJS 伺服器交換對等端的 IP 與 Port (ICE Candidates)。
2. **NAT 穿透**：代碼中配置了 `stun:stun.l.google.com:19302`，幫助位元在防火牆後的玩家建立直連通道。
3. **握手**：
* Client 發送 `{type: 'JOIN', name: '...'}`。
* Host 接收後，將其加入玩家列表並廣播全域同步訊息。



### 3. 數據同步機制 (Sync Logic)

遊戲中的所有狀態變更都封裝在一個 JSON 對象中進行傳輸：

```javascript
// 核心同步物件範例
{
  type: 'GAME_UPDATE',
  players: [...],   // 包含所有玩家的分數、Bonus、名稱
  curIdx: 0,        // 當前輪到誰
  rolls: 2,         // 剩餘擲骰次數
  dice: [6, 2, 4, 4, 1],
  kept: [false, false, true, true, false]
}

```

### 4. 關鍵衝突處理

* **防呆機制**：在 `applyScore` 或 `handleRoll` 函數中，會先判斷 `(mode === 'online' && myIdx !== curIdx)`。如果不是輪到該玩家，封包根本不會發送，且 UI 按鈕會被禁用。
* **最終權限**：Host 擁有最後的寫入權，所有 Client 的 UI 渲染皆是以 Host 發出的同步訊息為準，這能有效防止因延遲導致的邏輯分歧。

### 5. v0.7 中的資料計算邏輯

在 `updateGameUI` 中，我們不只是渲染 HTML，還即時執行了排序算法：

```javascript
// 即時排名計算邏輯
let sortedData = [...scoresData].sort((a, b) => b.total - a.total);
const rank = sortedData.findIndex(d => d.idx === pIdx) + 1;

```

這確保了無論是 Host 還是 Client，看到的排名始終是根據當前全域狀態計算出的最新結果。

---

> **💡 開發筆記**：
> 在 WebRTC 的 DataChannel 中，數據傳輸是非常快速的（幾乎等於 ping 值）。v0.7 為了優化手機體驗，在 CSS 中加入了大量的 `max-width` 限制與 `vh` 單位，確保在不同解析度下，PeerJS 的連線狀態指示燈始終清晰可見。
