# Home Assistant 防霉乾燥自動化

冷氣關掉後室內機蒸發器仍濕 → 發霉 → 酸味。此 package 在 Home Assistant 端加兩個功能：

1. **睡眠定時**：設定 N 小時 → 到時關機。
2. **自動防霉**（核心）：每次從冷氣/除濕關機（含 IR 遙控器、HA、HomeKit、app），自動先吹送風 N 分鐘吹乾蒸發器再真正關。

ESPHome 端不需任何改動，沿用現有 `climate id: ac` 暴露的 climate 實體。

---

## 為何放在 HA 而非 ESPHome

倒數用 HA 的 `timer` helper（`restore: true`）：撐得過 ESP 重開（brown-out，見 `../TROUBLESHOOTING.md`）與 HA 重啟。ESPHome 的 `delay` 只存 RAM，半夜重開會丟失整晚倒數 → 冷氣整晚不關。

## 為何 IR 遙控器關機也會觸發防霉

ESPHome midea 元件 `period: 1s` 持續輪詢冷氣**主機板實際狀態**。IR 遙控器與 UART 控制同一塊主機板 → IR 關掉冷氣，主機板狀態變 off，ESP ≤1s 內讀到 → HA climate 變 `off` → 觸發防霉。

> 副作用：用 IR「定時關」時冷氣不會準時真關，會延後乾燥分鐘數才關。預期內。不想被干涉就把該台「自動防霉」開關關掉。

---

## 部署步驟

### 1. 找出真實 climate entity_id（必做）

中文 friendly_name 產生的 entity_id 無法事先猜。到 **HA → 開發者工具 → 狀態**，搜尋 `climate.`，記下兩台的真實 id（可能是 `climate.maxe_aircon_living` 之類，**不是** `climate.客廳冷氣`）。

順便確認 hvac 模式字串是 `cool` / `dry` / `fan_only` / `off`（Midea 標準應為這些）。

### 2. 改 entity_id

編輯 `packages/aircon_antimold.yaml`，把所有 `climate.living` / `climate.bedroom` 換成步驟 1 找到的真實 id（檔內標了 `← 換成真實 entity_id`）。

### 3. 放進 HAOS

用 **File editor / Samba share / Studio Code Server** add-on 任一，把 `aircon_antimold.yaml` 複製到：

```
/config/packages/aircon_antimold.yaml
```

### 4. 啟用 packages（若尚未）

`/config/configuration.yaml` 加上（已有 `homeassistant:` 區塊就只加 `packages:` 那行）：

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 5. 檢查 + 重啟

**開發者工具 → YAML → 檢查設定（Check config）**確認無誤 → **重啟 Home Assistant**（新 helper / timer 需重啟才生效）。

### 6.（選配）暴露到 Apple 家庭 App

把 `input_number.*_sleep_hours`、`input_number.fan_dry_minutes`、`input_boolean.*_antimold_enabled`、`input_button.*_sleep_*` 經 HA HomeKit Bridge 暴露，手機就能設睡眠時數、開關自動防霉。

---

## 使用

| 想做的事 | 操作 |
|---|---|
| 開自動防霉 | 開 `input_boolean.<台>_antimold_enabled`（建議常開） |
| 睡前定時關 | 設 `input_number.<台>_sleep_hours` → 按 `input_button.<台>_sleep_start` |
| 取消睡眠定時 | 按 `input_button.<台>_sleep_cancel`（只停倒數，不強制關機） |
| 調乾燥時長 | 改 `input_number.fan_dry_minutes`（兩台共用，預設 60 分） |

> ⚠️ **睡眠定時要吹乾，自動防霉開關必須是 on。** 睡眠定時倒數完只下「關機」，乾燥由自動防霉接手。防霉 off 時睡眠 = 純延遲關機（不吹乾）。

### 預設行為

- DRY（除濕）關機**也**觸發防霉（除濕同樣弄濕盤管）。
- HEAT_COOL（自動）關機**不**觸發（暖氣不弄濕盤管；自動模式無法判斷當下在冷或暖）。
- IR 遙控器/IR 定時關**會**觸發防霉（延後乾燥分鐘數才真關）。

---

## 驗證

1. **載入**：重啟後開發者工具 → 狀態，確認 helpers / timers 都在。
2. **自動防霉**：開 `<台>_antimold_enabled`，冷氣設 cool → 手動關機 → 應自動轉 `fan_only` 且乾燥倒數啟動 → 手動 `timer.finish` `timer.<台>_fan_dry`（或把 `fan_dry_minutes` 設小值）→ 確認關機。
3. **睡眠定時串接**：開發者工具 → 動作，手動 `timer.start` `timer.<台>_sleep` duration `00:00:10`，冷氣維持 cool → 10 秒後應 關機 → 防霉接手轉 `fan_only` → 乾燥倒數 → 關。整串走通。
4. **防霉 off 對照**：關 `<台>_antimold_enabled`，重複步驟 3 → 應直接關、不吹乾。
5. **IR 觸發**：防霉 on，用實體遙控器關冷氣 → ≤1s 後 HA 應轉 `fan_only` 開始吹乾。
6. **迴圈確認**：開該 automation 的 trace，確認 `fan_only→off` 與 `cool→fan_only` 都沒再觸發防霉。

---

## 運作原理（迴圈為何不會無限循環）

自動防霉的 trigger 是 `from: [cool, dry]  to: off`：

- 防霉設 `fan_only` 時 → `cool→fan_only`，`to` 非 `off` → 不觸發。
- 乾燥結束關機 → `fan_only→off`，`from` 非 `cool/dry` → 不觸發。
- 睡眠定時的關機 `cool→off` → **會**觸發防霉（這正是方案 B 要的），只跑一輪（`mode: single`）。

所以不需額外的 global 旗標即可避免無限循環。
