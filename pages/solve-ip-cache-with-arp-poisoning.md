# 從 L7 到 L2 的降維打擊：利用 ARP 欺騙解決 IP 殘留問題

## 背景

最近處理了一個有趣的網路問題。我們的產品是 IoT 裝置，上頭運行著 Web 服務，客戶提出了一個嚴格的要求：**當裝置變更 IP 之後，必須確保 Client 端無法再透過舊 IP 連上裝置。**

這聽起來很直覺，但在實作上卻遇到了一個經典的網路底層挑戰。

## 原理探討

### ARP (Address Resolution Protocol)

在 TCP/IP 網路中，IP 位址是用來進行邏輯定址的，但實際在 Data Link Layer (L2) 傳輸時，Switch 和網卡認的是 **MAC address**。

當一台電腦想傳送資料給某個 IP 時，流程如下：

1. 檢查本機的 **ARP Table** 是否有該 IP 對應的 MAC。

2. 若無，發送 **ARP Request** 廣播：「誰是 192.168.1.10？請告訴我你的 MAC」。

3. 目標裝置回覆 **ARP Reply**：「我是 192.168.1.10，我的 MAC 是 AA:BB:CC...」。

4. 發送端將這組 Mapping 存入 ARP Table，並設定一個 TTL (Time To Live)。


重點就在這個 **ARP Table**。為了效能，作業系統不會每次都發 ARP，而是會快取一段時間（Windows 可能數分鐘，Linux 視設定而定）。

### Gratuitous ARP (GARP)

當裝置設定新 IP 或介面起來時，通常會主動廣播一個 **Gratuitous ARP**。它的功能主要有二：

1. **IP 衝突偵測 (DAD):** 確認網段上有沒有人已經用了這個 IP。

2. **更新鄰居快取:** 告訴網段上的其他裝置：「嘿，這個 IP 現在對應到我的 MAC 喔，請更新你們的 Table。」


## 問題描述

問題發生在裝置「換 IP」的瞬間。

當裝置取得新 IP 時，它發送 GARP 宣告了「新 IP = 我的 MAC」。**但是，它並沒有義務（也沒有標準機制）去告訴大家「舊 IP 已經不是我了」。**

這導致 Client 端的 OS 裡，**舊 IP 的 ARP 快取依然存在**且指向裝置的 MAC。因此，Client 仍然會嘗試把封包送到裝置的 MAC，而裝置如果沒有嚴格的 Ingress Filtering，或者 Web Server 綁定的是 `0.0.0.0`，就可能發生「明明換了 IP，舊 IP 卻還能通」的幽靈現象。

這完全不可控，因為清除 ARP cache 是 Client OS 的行為。

## 解法思路

### 笨方法：應用層攔截

這個 ticket 不知道輾轉了幾手，最後來到 Backend Team 的我的手上，我被要求從 Web Server (L7) 下手，技術上是做得到，例如：

- Middleware 檢查進來的 Request 是透過哪個 IP 進來的。

- 如果是舊 IP，就回傳 403 Forbidden 或直接斷開連線。


**看似能通過驗收標準，但是缺點是：**

- **行為詭異：** Client 明明在 L2/L3 成功連上了，卻在 L7 被踢掉。

- **效能浪費：** 封包都已經經過網卡、進入 Kernel、爬到 Application Layer 了才被丟棄，浪費裝置資源。

- **無法滿足「不允許連上」的嚴格定義：** TCP Handshake 其實已經完成了。


更要命的是，如果事後對方的工程師發現這種詭異的行為，可能會覺得我們很蠢。

### 完美解方：鏈路層欺騙

我傾向從根源解決問題。既然問題出在 Client 的 ARP Table 不更新，那我們就**強迫它更新**。

這個解法需要 L2 的操作權限，於是我找了 Platform Team 分享我的妙招並進行 Pair Programming。策略是利用 **Gratuitous ARP** 進行一次「善意的 ARP 欺騙 (ARP Poisoning)」。

**實作邏輯：**

1. 裝置取得新 IP。

2. 裝置發送標準 GARP 廣播新 IP。

3. **（關鍵步驟）裝置發送一個偽造的 GARP：**

    - **Sender IP:** 舊 IP

    - **Sender MAC:** 一個隨機生成的、不存在的 MAC (例如 `DE:AD:BE:EF:00:00`)


```
sequenceDiagram
    participant Client
    participant Device

    Note over Client: ARP Table:<br/>Old_IP -> Device_MAC

    Device->>Client: GARP (IP: Old_IP, MAC: FAKE_MAC)

    Note over Client: ARP Table 更新:<br/>Old_IP -> FAKE_MAC

    Client->>Network: Request to Old_IP
    Note right of Network: Switch 轉送給 FAKE_MAC<br/>(黑洞/丟棄)

    Client--xDevice: Connection Timeout


```

## 結果

這個做法的效果是立竿見影的：

1. 網段上的所有裝置收到這個 GARP，誤以為舊 IP 換網卡了。

2. 它們更新 ARP Table，將舊 IP 對應到那個不存在的 MAC。

3. 當 Client 再次嘗試連線舊 IP，封包被 Switch 送往黑洞（或者因為找不到該 MAC 而被丟棄）。

4. Client 端會出現 **Connection Timeout**，徹底斷絕了與裝置的實體連線。


這個萬年 Issue 最終在 Backend 與 Platform Team 的合作下，透過對協定的理解完美解決！有時候後端工程師不能只看 HTTP，往下鑽研 TCP/IP 協定堆疊，往往能找到更優雅的解法。

### 風險提示

雖然這招很有效，但在實作時需注意以下風險：

**Switch 安全機制 (DAI):** 在企業級環境中，如果 Switch 開啟了 Dynamic ARP Inspection，這種 IP 與 MAC 不符（或是來自錯誤 Port）的 ARP 封包可能會被視為攻擊而被攔截，甚至導致 Port 被封鎖。
