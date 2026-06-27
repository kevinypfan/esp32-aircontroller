# MAXE RA-50SH32 冷氣 ESPHome 控制器

把 MAXE（萬士益）RA-50SH32 冷氣接到 Home Assistant，繞過原廠雲端、繞過官方 WiFi 模組，本地直接控制。

---

## 為什麼這樣做

| | 官方 MRP-0508 路線 | DIY 路線（本專案） |
|---|---|---|
| 花費 | 模組買全套，未知價格 | 幾個零件，便宜得多 |
| 雲端依賴 | 走原廠雲端 | 本地直連 HA |
| 自動化 | App 內建有限 | HA 全部自動化都能用 |
| 整合度 | 自家 App | Apple Home / Google / Alexa 任選 |

## 為什麼用 Midea 協定而非 TaiSEIA

RA-50SH32 是美的（Midea）OEM 機種——主板印著 `KFR-35G/E` 是鐵證。

官方 MRP-0508 模組做的事就是「Midea UART ↔ TaiSEIA 雲端」的翻譯。既然冷氣 MCU 母語是 Midea 協定，我們直接講 Midea 反而簡單，社群（ESPHome `midea` platform）成熟度也高得多——這個元件是 ESPHome 官方內建、dudanov 維護的，數百人實測過。

---

## 硬體

- **冷氣**：MAXE RA-50SH32（KFR-35G 主板）
- **接頭**：CN3 WIFI，JST XH 2.54mm，4-pin
- **微控制器**：ESP32-C3 SuperMini
- **電平轉換**：BSS138 4-channel level shifter（CN3 是 5V TTL，ESP32-C3 是 3.3V）
- **緩衝電容**：470µF 16V（接 ESP 5V 輸入，防壓縮機啟動 brown-out）

詳細 BOM、接線、Pinout：👉 [HARDWARE.md](HARDWARE.md)

## 軟體

- ESPHome（內建 `midea` climate platform，arduino framework）
- Home Assistant

設定檔採「共用 + 各台入口」結構:

- [maxe-aircon-common.yaml](maxe-aircon-common.yaml) — 共用設定(不直接燒,由各台 `!include`)
- [aircon-living.yaml](aircon-living.yaml) — 客廳那台的燒入入口
- [aircon-bedroom.yaml](aircon-bedroom.yaml) — 臥室那台的燒入入口

Home Assistant 端的自動化（自動防霉吹乾 / 睡眠定時 / beeper 半夜靜音）放在 [homeassistant/](homeassistant/)（含 package YAML 與說明）。

之後若要加其他設備,會以同樣方式新增各自的入口檔。

**第三台：日立除濕機 RD-18FJ**（✅ 2026-06-27 實機通訊驗證：讀到 `HITACHI / RD-18FJ`、室內濕度即時回報）——它**沒有紅外線遙控**，但跟冷氣一樣有廠商服務埠（官方 WiFi 模組 RC-W04XE 插的孔，接頭 Hitachi JST PAP-04-V，5V 邏輯），走 **TaiSEIA 101 協定**。改用社群 ESPHome 外部元件 [tsunglung/taixia](https://github.com/tsunglung/taixia) 控制（本地、非雲端），架構與冷氣相同：ESP32-C3 + BSS138 → 服務埠 → ESPHome → HA。除濕機在 taixia 不是 climate，而是 switch（電源／防霉／嗶聲）＋ number（目標濕度 40~70%／風速／除濕等級）＋ sensor／binary_sensor（室內濕度／水箱滿水）的組合。燒入入口檔：[dehumidifier-rd18f.yaml](dehumidifier-rd18f.yaml)。

---

## 進度追蹤

> ✅ **已完成並上線運作中**：兩台冷氣（客廳＋臥室）實接 ESP，ESPHome + Home Assistant 控制中；防霉吹乾 / 睡眠定時 / beeper 半夜靜音都已部署驗證。下面是當初的施工清單，留作紀錄。

### 規劃階段
- [x] 確認冷氣是 Midea OEM 機種（KFR-35G）
- [x] 確認 CN3 接頭規格（JST XH 2.54mm 4-pin）
- [x] 確認 pin 1 = +5V（從 PCB 絲印）
- [x] 決定走 Midea 協定（非 TaiSEIA）
- [x] 決定 MCU：ESP32-C3 SuperMini
- [x] 決定電平轉換：BSS138

### 採購階段
- [ ] ESP32-C3 SuperMini（已有 / 待購）
- [ ] BSS138 4-channel 模組
- [ ] JST XH2.54 4P 帶線母頭 20cm
- [ ] 470µF 16V 電容 × 2~3
- [ ] 杜邦線母對母 20cm

### 量測階段
- [ ] 三用電表確認 CN3 pin 2/3/4（GND / TX / RX 順序）

### 焊接階段
- [ ] 電容焊到 ESP32-C3 的 5V 跟 GND 腳之間
- [ ] BSS138 排針焊好
- [ ] JST XH 線裸線端焊到 BSS138 HV 側

### 燒韌體階段
- [ ] 先複製 `secrets.yaml.example` 為 `secrets.yaml` 並填入真實值
- [ ] ESPHome 編譯/燒入 `aircon-living.yaml`(或 `aircon-bedroom.yaml`)
- [ ] USB 接電腦燒入（**先不要接冷氣**）
- [ ] 確認 ESP 上線到 Home Assistant
- [ ] 拔 USB

### 接機測試階段
- [ ] 確認冷氣已斷電（總斷路器）
- [ ] BSS138 → ESP 杜邦線接好
- [ ] JST 線插入 CN3
- [ ] 冷氣通電
- [ ] 看 ESPHome log 是否正常 polling
- [ ] HA 上測試開關、調溫、改模式
- [ ] 從遙控器改設定，看 HA 是否同步更新

### 收尾階段
- [ ] OTA 更新測試（之後韌體更新走這條）
- [ ] 找個塑膠盒把 ESP+BSS138 包起來
- [ ] 塗三防漆（選配，防潮）

### 第三台：日立除濕機 RD-18FJ（TaiSEIA / taixia）
- [x] 確認 RD-18FJ 無 IR、有 TaiSEIA 服務埠（RC-W04XE 同孔）
- [x] 確認協定走 taixia（type: dehumidifier, SA ID 4），ESPHome `config` 驗證通過
- [x] 撰寫入口檔 `dehumidifier-rd18f.yaml`（沿用 C3 + BSS138）
- [x] USB 燒入 C3、上線（這塊板需 `output_power: 8.5` 才連得上 ASUS）
- [x] 接服務埠、TaiSEIA 通訊驗證：讀到 `HITACHI / RD-18FJ`（SA ID 0004）、室內濕度即時回報
- [ ] HA 採納、測開關／目標濕度／模式／水箱滿水
- [ ] （Phase 2）接 HA beeper 半夜靜音 + 防霉/排程 + Lovelace 卡片
- 旁支：另有 ESP8266 D1 mini 實驗版 `dehumidifier-rd18f-d1mini.yaml`（編譯通過、暫未實接）

---

## ⚠️ 重要安全提醒

1. **接到冷氣後絕對不要再把 ESP 的 USB 接電腦**。萬一冷氣開關電源絕緣擊穿，5V 地線會帶 220V，會把電腦炸掉（甚至觸電）。所有韌體更新走 OTA。
2. **第一次通電前用三用電表確認 CN3 pin 順序**，不要憑想當然爾。
3. **拆冷氣面板前先斷總電源**，等 5 分鐘讓內部電容放電。
4. **電容極性**：長腳 / 不在白條那側 = 正極，接 +5V。接反會爆。

---

## 文件導覽

| 檔案 | 用途 |
|---|---|
| [README.md](README.md) | 你現在看的，總覽 + 進度 |
| [HARDWARE.md](HARDWARE.md) | BOM、CN3 pinout、完整接線表 + 接線圖 |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | 常見問題排除 |
| [maxe-aircon-common.yaml](maxe-aircon-common.yaml) | ESPHome 共用設定(各台 `!include`,不直接燒) |
| [aircon-living.yaml](aircon-living.yaml) | 客廳那台燒入入口 |
| [aircon-bedroom.yaml](aircon-bedroom.yaml) | 臥室那台燒入入口 |
| [dehumidifier-rd18f.yaml](dehumidifier-rd18f.yaml) | 日立除濕機 RD-18FJ 燒入入口（TaiSEIA / taixia，單檔自帶設定） |
| [homeassistant/](homeassistant/) | HA 端自動化（防霉吹乾 / 睡眠定時 / beeper 靜音）package 與說明 |
| [secrets.yaml.example](secrets.yaml.example) | 密碼/金鑰範例（複製成 secrets.yaml 後填寫） |

## 參考資源

- [ESPHome midea component](https://esphome.io/components/climate/midea/)
- [dudanov/MideaUART](https://github.com/dudanov/MideaUART)
- [Midea AC ESPHome HA Community thread](https://community.home-assistant.io/t/midea-branded-ac-s-with-esphome-no-cloud/265236)
- [tsunglung/taixia](https://github.com/tsunglung/taixia)（TaiSEIA 協定備案，本專案沒用）
