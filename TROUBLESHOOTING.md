# 常見問題排除

## 通訊類

### Q1：ESPHome log 一直顯示 "Response timeout"

冷氣完全沒回應 ESP 的 polling。

**可能原因 1：TX/RX 接反**
- 用三用電表重新確認 CN3 pin 3 / pin 4
- 或最快的測試：把 YAML 裡的 `tx_pin` 跟 `rx_pin` 對調試試（GPIO20 ↔ GPIO21）

**可能原因 2：ESP32-C3 的 UART loopback bug**
- 症狀：log 中看到自己發出去的 frame 又被讀回來，後接 timeout
- 解法 1：升級 ESPHome 到最新版（2024 年後大多修好了）
- 解法 2：在 `uart:` 區塊加上 `invert: false`（雖然預設就是 false，但顯式寫有幫助）

**可能原因 3：BSS138 的 LV reference 沒接**
- 確認 BSS138 的 LV 腳有接到 ESP 的 **3V3**（不是 5V，不是 GND）
- 沒接的話 level shifter 不知道要轉成 3.3V，訊號出不來

**可能原因 4：baud rate 不對**
- 標準 Midea 是 9600 bps；YAML 已設好
- 極少數老機種可能是 4800 或 19200，可以試著改 baud_rate 看看

---

### Q2：模組上線但 Home Assistant 沒有 climate entity

- 確認 YAML 裡有 `autoconf: true`
- 看 log 是不是有 "Initial query failed" 或 "Unable to determine capabilities"
- 通常還是 UART 不通的問題，回頭看 Q1
- 真的不行可以把 `autoconf` 關掉手動指定 capabilities（參考 ESPHome midea 文件）

---

### Q3：HA 上能看到 entity 但開關、溫度調整沒反應

**確認 1：beeper 有沒有響**
- YAML 裡 `beeper: true` 開著的話，每次下命令冷氣應該嗶一聲
- 沒嗶 = 命令沒傳到，回頭查 UART
- 有嗶但冷氣沒動作 = 命令傳到了但解析有問題

**確認 2：log 看 Midea 元件狀態**
- 應該每秒一次 polling，看到「Got response」之類訊息
- 看到 "Bad checksum" 表示有干擾，需要加屏蔽線或縮短線距

---

## 電源類

### Q4：ESP 三天兩頭重開

**症狀**：log 看到 `brownout detector triggered` 或啟動 log 重複出現

**解法**：
1. 確認 470µF 電容有焊好、極性沒反（白條那側 = 負極接 GND）
2. 不夠的話再並聯一顆 470µF（兩顆並聯 = 940µF）
3. 還是不行的話，CN3 的 5V 不夠力，改從外部 USB 5V 變壓器供電——但 **GND 必須跟冷氣共地**，否則 UART 訊號會浮動

---

### Q5：ESP 完全沒反應（連 USB 都點不亮）

USB 接電腦時的問題：
- 換條線、換 port、換電腦試
- ESP32-C3 SuperMini 有些版本要按住 BOOT 鍵才會進燒錄模式

連 CN3 時的問題：
- 萬用電表量 CN3 pin 1 對機殼地是不是 5V
- 如果 0V，可能該機型需要短接主板上的 J9 跳線電阻才會輸出 5V（先別動，拍照到 HA 社群問）
- 或者 pin 1 不是 +5V（再用電表量過一次）

---

## 安裝類

### Q6：WiFi 訊號很弱、常斷線

- 模組是不是被金屬背板擋住了？換位置，盡量靠塑膠面板
- 路由器訊號離冷氣太遠？加 mesh 或 repeater
- ESP32-C3 SuperMini 的 PCB 天線是板邊那條金屬蛇形，不要被金屬或太厚的塑膠蓋住
- log 看 `WiFi signal strength` 低於 -75dBm 就算弱了

---

### Q7：紅外線遙控失效（裝了 ESP 之後）

某些 OEM 板 IR 接收器跟 UART TX 共用線路，會用 0Ω 跳線電阻（J3）短接：
- 拆下這顆 0Ω 電阻通常可以解決
- 但這需要進階 SMD 焊接技巧
- 拍主板照片到 Home Assistant 社群或 Github issue 找 Midea 線路圖比對

如果原本沒接 ESP 時遙控就失效，那是別的問題（電池、IR 接收器壞了之類）。

---

## 安全類

### Q8：把 ESP 連 USB 到電腦，同時冷氣也通著電，有什麼風險？

**極高風險，絕對不要這樣做。**

冷氣 5V 軌的地線參考 220V 火線。正常運作時跟大地接近等電位，但開關電源萬一絕緣擊穿，地線會直接帶 220V。這時候 USB 線就是 220V → 電腦主機板的直通路徑——輕則炸主板，重則人觸電。

**所有韌體更新走 OTA**（已在 YAML 設好）。debug 時要嘛斷冷氣總電源、要嘛拔掉 CN3 線，二選一。

---

### Q9：拆冷氣面板時要注意什麼？

- **拔總電源**（斷路器），不是只用遙控器關
- 拔下後等 5 分鐘讓內部電容放電
- 不要碰主板上的大電容、繼電器、壓縮機線
- CN3 是低壓側相對安全，但其他區域不要亂摸
- 動工前手摸機殼地放靜電，免得燒主板上的 IC

---

### Q10：可以接電容到別的位置嗎？例如直接在 CN3 pin 1/2 上加？

可以，**而且有時候更好**——直接在 CN3 接頭旁邊並聯電容，能更早攔截電源跌落。

實作方式：找 ESP 5V 跟 GND 之間焊一顆，或直接把電容兩腳跨在 JST XH 線的紅黑線（5V 跟 GND）之間。注意極性。

---

## Debug 技巧

### 看 ESPHome log

```
esphome logs maxe-aircon.yaml
```

或在 ESPHome dashboard 點該裝置的 **LOGS** 按鈕。重點看：
- `[uart] Setting up UART...`：UART 是否正常初始化
- `[midea] Setting up...`：Midea 元件啟動
- 每秒一次的 `Sending: AA...` / `Got: AA...`：polling 在運作
- `brownout detector triggered`：電源不穩
- `Response timeout`：冷氣沒回應

### 三用電表 + 示波器（進階）

如果有示波器，可以直接量 CN3 pin 3（冷氣 TX）的波形：
- 9600 bps 一個 bit 寬度 = 104µs
- Idle 時應該是 high（5V），有資料時看到一連串 0/1 變化
- 完全沒訊號 = 冷氣沒在 polling，可能是 CN3 沒啟用，需要查 J9 或其他 jumper

### 抓 UART 對話（更進階）

USB-to-TTL 模組（FT232 / CH340）+ 3.3V 邏輯分析儀（Saleae 或副廠），可以直接 sniff CN3 上的封包，比對 Midea 協定文件，這樣連協定層的問題都能查。
