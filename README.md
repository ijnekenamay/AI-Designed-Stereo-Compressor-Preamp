# せいせいAIにたすけてもらいながらさいきょうの Stereo Compressor / Preampをつくる

> 本書は、生成AIを使用して書いたV2164（SSI2164）を用いたステレオ・コンプレッサ／プリアンプの**詳細仕様書**である。
> 今のところ日本語のみで進めています。
> プロフェッショナル機材相当の機能と音質を実現する。


[VCA IC購入先](https://ears-modular.com/products/sound-semiconductor-fatkeys%E2%84%A2-ssi2164-vca)

---

# SSI2164 VCA コンプレッサー 製作仕様書

## 1. 全体構成図（オーディオ・パス）

本機はモノラル構成の VCA コンプレッサーであり、  
各ブロックは以下の順序で信号処理を行う。  
最終出力は正相（In-Phase）とする。

Input Buffer  
→ V/I Converter  
→ VCA Core（SSI2164）  
→ I/V Converter  
→ Make-up Gain  
→ Blend（Parallel Mix）  
→ Output Buffer  


## 2. メイン基板：オーディオ・パス詳細

### A. 入力および VCA コア部

- Input Buffer  
  ボルテージフォロワ構成によるインピーダンス変換

- Input Resistor（Rin）  
  20kΩ（1%）  
  SSI2164 Pin 2 へ直列接続

- Stability Network（必須）  
  SSI2164 Pin 2 と GND 間に以下を直列接続  
  - 220Ω  
  - 1200pF  

- Mode 設定（Pin 1）  
  OPEN（未接続）  
  Class AB 動作モード  
  低ノイズかつ楽器用途向きの設定

- 電源デカップリング  
  - Pin 16（V+）– GND：0.1µF  
  - Pin 9（V−）– GND：0.1µF  
  ※最短距離で配置すること


### B. 出力およびゲイン調整部

- I/V 変換段  
  SSI2164 Pin 4 をオペアンプ反転入力へ接続  
  - 帰還抵抗：20kΩ（1%）  
  - 帰還コンデンサ：47pF（発振防止）

  ※本段で位相反転が発生する

- Make-up Gain 段  
  反転増幅回路構成  
  - 入力抵抗：10kΩ  
  - 帰還固定抵抗：10kΩ  
  - 帰還可変抵抗：100kΩ（B）

  最大ゲイン：  
  (10k + 100k) / 10k = 11 倍（約 +20dB）

  ※本段で位相は正相に戻る


### C. Blend（Parallel Mix）回路

- Dry 信号：Input Buffer 出力より分岐
- Wet 信号：Make-up Gain 出力より分岐

- Mix Pot  
  - 25kΩ（B）または 50kΩ（MN）
  - 端子1：Dry  
  - 端子3：Wet  
  - 端子2（ワイパー）：出力バッファへ

- 出力バッファ  
  オペアンプによるボルテージフォロワ  
  出力保護抵抗：100Ω（直列）


## 3. サイドチェイン・ユニット（制御部）

本機は **フィードフォワード方式**のサイドチェインを採用する。  
入力信号のレベルを検波・整流し、生成された制御電圧（CV）によってSSI2164のゲインを制御する。

※本仕様は **ステレオ構成**を前提とする。  
ステレオ化は、L/R 信号を合成したCVを共通でVCAに供給する「ステレオリンク方式」とする。


### A. 検波部（Precision Rectifier）

- 入力信号  
  Input Buffer 出力より分岐（オーディオ・パス非影響）

- サミング  
  47kΩ 抵抗を介してオペアンプ入力へ接続

- 整流方式  
  1N4148 ダイオードを用いた全波精密整流回路

- 設計意図  
  - ダイオード順方向電圧による誤差を補償  
  - 小信号レベルでも正確な制御電圧を生成

- 想定電圧レンジ  
  - 整流後 DC：0 ～ 約 0.9V（入力レベルに比例）


### B. Threshold（スレッショルド）設定

- Threshold Pot  
  10kΩ（B）

- 動作  
  - 整流電圧が Threshold を超えた分のみを有効成分とする  
  - Threshold 未満では圧縮動作を行わない（ユニティゲイン）


### C. Ratio（圧縮率）制御

- Ratio Pot  
  50kΩ（B）

- 動作原理  
  - Threshold を超えた信号成分のみを増幅  
  - 制御電圧の増加量を調整することで圧縮率を決定

- 対応圧縮率（目安）
  - 1:1（圧縮なし）
  - 2:1 ～ 4:1（ナチュラル）
  - 10:1（強い圧縮）
  - ∞:1（リミッター動作）

- 備考  
  SSI2164 の制御感度は **−33mV/dB**  
  Ratio は連続可変とする


### D. Attack / Release（時定数回路）

- タイミングコンデンサ  
  10µF（電解またはタンタル）

- Attack  
  - Pot：50kΩ（A）  
  - 最小値制限抵抗：100Ω  
  - 時定数範囲：約 1ms ～ 500ms

- Release  
  - Pot：250kΩ（A）  
  - 時定数範囲：約 10ms ～ 10s


### E. CV 出力および VCA 駆動

- CV バッファ  
  オペアンプによるボルテージフォロワ

- CV 出力先  
  SSI2164 Pin 3（CV入力）

- CV Scaling  
  - 出力直列に 10kΩ 半固定抵抗を挿入  
  - SSI2164 の −33mV/dB 特性に合わせて
    実機でリダクション量を微調整可能とする


## 4. 全体信号フロー（俯瞰）

```text
INPUT
 │
 ▼
[Input Buffer]
 │
 ├───────────────┐
 │               │
 ▼               ▼
[V/I Conv]   [Sidechain Pick]
 │               │
 ▼               ▼
[SSI2164 VCA] [Rectifier → Thr → Ratio → A/R]
 │                               │
 ▼                               ▼
[I/V Conv]                [CV Buffer + Trim]
 │                               │
 ▼                               │
[Make-up Gain] <─────────────────┘
 │
 ├─── Dry ─────┐
 │             │
 ▼             ▼
[Blend Pot]  (Dry)
 │
 ▼
[Output Buffer]
 │
 OUTPUT

```

## 5. オーディオ・パス回路図
### Input BufferとV/I Converter（SSI2164入力）
```text
(Input Jack)
    │
  [AUDIO_IN_L]
    │
    │   C_in (10uF - 22uF)
    │   Bipolar Electrolytic
    ├───┤ | ───┐
    │          │
    │          │             (+12V/15V)
    │          │                 ▲
    │          │                 │
    │          │               ┌─┴─┐
    │    R_pd  │               │ + │
    │    1MΩ   ├───┬───────────┤OP1│◄── TL074 / OPA1678
    │          │   │           │ - │
    │         ┌┴┐  │           └─┬─┘
    │         GND  │             │
    │              └─────────────┤
    │                            │
    └────────────────────────────)───┬┴───────────────────────────→ [TO_SIDECHAIN_L]
                                     │   │
                                     │   │    R_in (Precision)
                                     │   │      20k (1%)
                                     │   └───/\/\/\/\/──┬─────────────→ [TO_VCA_IN_L] (Pin 2)
                                     │                  │
                                     │             Stability
                                     │              Network
                                     │             ┌────┴────┐
                                     │             │         │
                                     │            220        │
                                     │             │         │
                                     │           1200p       │
                                     │             │         │
                                     │            ┌┴┐       ┌┴┐
                                     │            GND       GND
                                     │
                                     └────────────────────────────────→ [TO_BLEND_DRY_L]
```
- 構成：ボルテージフォロワ
- 出力は
-- VCA用
-- サイドチェイン用
-- Dry用へ分岐
- Stability Network：必須
- Pin1（Mode）：OPEN

### SSI2164 VCA Core
```
[TO_CV_BUFFER_OUT] (From Sidechain)
                           │
                           │ (Stereo Link)
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
      (Pin 3: CV1)                    (Pin 6: CV2)
 _______________________________________________________
|                                                       |
|                  SSI2164 / V2164                      |
|                                                       |
|  [CH 1 : LEFT]                      [CH 2 : RIGHT]    |
|                                                       |
|  Pin 2: IN <──[TO_VCA_IN_L]         Pin 7: IN <──[TO_VCA_IN_R]
|                                                       |
|  Pin 4: OUT ──>[TO_IV_CONV_L]       Pin 5: OUT ──>[TO_IV_CONV_R]
|_______________________________________________________|
       │      │      │             │      │      │
       │Pin 1 │Pin 16│Pin 8        │Pin 9 │Pin 10-15
       │ MODE │  V+  │ GND         │  V-  │ (Unused)
       └──┬───┴──┬───┴──┬──────────┴──┬───┴───┬─────┐
          │      │      │             │       │     │
        (OPEN) (+15V)  ┌┴┐          (-15V)   20k   ┌┴┐
        Class          GND                    │    GND
          AB                                 ┌┴┐
                                             GND
```
- CV感度：−33mV/dB
- 未使用CH (3 & 4): Pin 10, 15 (IN) は20kΩ経由でGNDへ。Pin 11, 14 (CV) は直接GNDへ。Pin 12, 13 (OUT) はオープン。（以下の通り）
```text
Pin 10 ──/\/\/\── GND (20k)
Pin 11 ────────── GND
Pin 12 ────────── (OPEN)
Pin 13 ────────── (OPEN)
Pin 14 ────────── GND
Pin 15 ──/\/\/\── GND (20k)
```

### I/V Converter & Make-up Gain (Left)
```text
[TO_IV_CONV_L] (Pin 4)
      │
      │     R_fb (20k 1%)    C_fb (47p)
      ├───/\/\/\/\/───────┤|├───┐
      │                         │
      │       ┌───┐             │
      │       │ - │             │
      └───────┤OP2│◄── I/V Amp  │
              │ + │             │
          ┌───┤   ├─────────────┴─┬──────────→ (Inverted Audio)
          │   └───┘               │
         ┌┴┐                      │
         GND                      │
                                  │
      ┌───────────────────────────┘
      │
      │     R_mk_in (10k)
      ├───/\/\/\/\/───┐
      │               │
      │           Make-up Gain Pot
      │            100k (Lin/B)
      │          ┌──/\/\/\──┐
      │          │    ▲     │
      │   R_fix  │    │     │
      │    10k   │    │     │
      ├──/\/\/\──┴────┘     │
      │                     │
      │       ┌───┐         │
      │       │ - │         │
      └───────┤OP3│◄── Gain Amp
              │ + │             
          ┌───┤   ├──────────────┬───────────→ [TO_BLEND_WET_L]
          │   └───┘              │
         ┌┴┐                     │
         GND               (Phase Corrected)
```
- ここで位相反転している

### Blend & Output (Left)

```text
[TO_BLEND_DRY_L] ──────────────┐
                               │
                          [1]  │
                        ┌──────┴──────┐
                        │             │
        Blend Pot       │  /\/\/\/\/  │
      25k(B) or 50k(MN) │      ▲      │
                        │      │      │
                        └──────┬──────┘
                          [3]  │ [2: Wiper]
                               │
[TO_BLEND_WET_L] ──────────────┘
                               │
                               │
                               ▼
                            ┌─────┐
                            │  +  │
                            │ OP4 │◄── Output Buffer
                      ┌─────┤  -  │
                      │     └─────┘
                      │        │
                      └────────┼────/\/\/\────→ [OUTPUT_JACK_L]
                               │      100R
                               │
                              To Metering? (Optional)
```
- 最大 +20dB
- 位相が正相に戻る

## 6. サイドチェイン回路図

### [SC-1] Summing & Rectifier (精密整流・サミング)
LとRの信号を混ぜて、検波（プラスのDCに変換）する段階です。
```text
[TO_SIDECHAIN_L] ──/\/\/\──┐ 47k
                           │
[TO_SIDECHAIN_R] ──/\/\/\──┼──────┐
                     47k   │      │
                           │    ┌─▼─┐
                           │    │ - │
                           └────┤OP_A│ (Summing Amp)
                                │ + │
                           ┌────┴─┬─┘
                          ┌┴┐     │
                          GND     │
          ┌───────────────────────┘
          │
          │   10k            D1(1N4148)
          ├──/\/\/\──┬───────►|───────┐
          │          │                │
          │          │     D2(1N4148) │
          │        ┌─┴─┐     ┌────────│
          │        │ - │     │        │
          └────────┤OP_B├────┘        │
                   │ + │              │
                ┌──┴─┬─┘              │
               ┌┴┐   │                │
               GND   │      10k       │
                     └────/\/\/\──────┤
                                      │
                                      ▼
                                [RECTIFIED_DC] (0V ~ 約 +0.9V)
```
### [SC-2] Threshold & Ratio (圧縮開始点と比率)
ここで「どれくらいの音量から、どれくらい潰すか」を決めます。
```text
[RECTIFIED_DC]
      │
      │     Threshold Pot (10k B)
      │        ┌──/\/\/\──┐
      │        │    ▲     │
      │       +V    │     │
      │             │    ┌┴┐
      │   100k      │    GND
      ├──/\/\/\─────┤
      │             │
      │           ┌─▼─┐                Ratio Pot (50k B)
      │           │ + │              ┌──/\/\/\──┐
      └───────────┤OP_C│    ┌────────┤    ▲     │
                  │ - │     │        │    │     │
                  └─┬─┘     │        └───┬──────┘
                    │       │            │
                    └───────┴────────────┘
                                         │
                                         ▼
                                  [TO_ST_LINK_TIMING]
```
### [SC-3] Stereo Link, Timing & CV Output (反転バッファ統合)
```text
[TO_ST_LINK_TIMING] (From Ratio Out)
      │
      │   [[ Stereo Link & Timing Section ]]
      │
      ├───────►|───────┬───/\/\/\────┐  (D_atk & Attack Pot)
      │     D_atk      │   Atk_Pot   │
      │                │  (2連/Dual) │
      │                └────────────┤
      │                             │  [Point X]
      ├───────/\/\/\────────────────┤  (Release Pot 2連/Dual)
      │       Rel_Pot               │
      │                             ├──────┬──────→ [TO_METER_IN]
      │                             │      │        (0V ~ プラス電圧)
      │                           ──┴──    │
      │                           ──┬──    │
      │                           C_time   │
      │                            10uF    │
      │                             │      │
      │                            GND     │
      │                                    │
      └────────────────────────────────────┘
                                           │
      [[ CV Buffer Section: Inverting Amp ]]
      (SSI2164を駆動するためにプラスをマイナスに反転)
                                           │
                 R_fb (10k)                │
             ┌──/\/\/\/\──┐                │
             │            │          R_in  │
             │   ┌────┐   │        ┌─/\/\/\┘
             └───┤ -  │   │        │
                 │OP_D├─┬─┴──┬─────┴─ [CV_Trim_L (10k)] ──► [VCA Pin 3]
      GND ───────┤ +  │ │    │
                 └────┘ │    └─────── [CV_Trim_R (10k)] ──► [VCA Pin 6]
                        │
                        └─ (Stereo Link Sync Point)
```
- ステレオリンク: 左右の検波信号が OP_A でサミングされ、1つの C_time（タイミング用コンデンサ）を共有して1つの OP_D（CVバッファ）から左右のVCAへ送られるため、完璧に同期します。

## 7. GRメーター回路
### 構成概要
- IC: LM3914 (Linear Scale) Mode: Bar Mode (Dot modeの場合はPin9をオープン)─（既存）CV Trim → SSI2164

### LM3914 基本配線
```text
[TO_METER_IN] (From SC Buffer / CV 0V ~ -1.5V程度)
      │
      │ ※CVが負電圧の場合は反転アンプを通すか、
      │   SC Bufferの段階で正負を反転させておく必要があります。
      │
      └───────────────────────┐
                              │
                              ▼
                        (Pin 5: SIG)
                ┌───────────────────────────┐
 (+12V/15V) ────┤ Pin 3: V+                 │
                │                           │
                │         LM3914            │
                │                           │
   ┌────────────┤ Pin 9: MODE (Bar/Dot)     │
   │            │                           │
  1k8           │ Pin 1: LED1 (Green)  ──►|─┐│
   │            │ Pin 18: LED2 (Green) ──►|─┤│
   ▼            │  ...                      ││
 (Pin 7) ───────┤ Pin 10: LED10 (Red)  ──►|─┤│
 Ref Out        │                           ││
   │            └───────────────────────────┘│
   │                                         │
   ├────────────┐                            │
   │            │                            │
   │      Scale Adjust Trim                  │
   │       (10k Trimmer)                     │
   │      ┌────────────┐                     │
   │      │    [1]     │                     │
   └──────┤      ┌─────┴────── (Pin 6: R_HI) │
          │      ▲ [2: Wiper]                │
          │    [3]     │                     │
          └──────┬─────┘                     │
                 │             (Pin 4: R_LO) │
                 ├───────────────────────────┤
                 │                           │
                ┌┴┐                         ┌┴┐
                GND                         GND
```
※ LEDには直列抵抗不要（LM3914内蔵）

#### 表示ロジック

SSI2164 の特性 [−33mV = 1dB GR]

| LED  | 表示GR   | 色 | カソード (－) の接続先 | アノード (＋) の接続先 |
| ---- | ------ | - | ------ | ---------- |
| LED1 | −3 dB  | 緑 | Pin 1 | 電源 V+ (15V) |
| LED2 | −6 dB  | 緑 | Pin 18 | 電源 V+ (15V) |
| LED3 | −9 dB  | 黄 | Pin 17 | 電源 V+ (15V) |
| LED4 | −12 dB | 黄 | Pin 16 | 電源 V+ (15V) |
| LED5 | −15 dB | 赤 | Pin 15 | 電源 V+ (15V) |
| LED6 | −18 dB | 赤 | Pin 14 | 電源 V+ (15V) |
LM3914（およびLM3915/3916）というICに限っては、アノードを15Vに直接つないで大丈夫。理由は、LM3914が**「定電流吸い込み（Constant Current Sink）」**という特殊な出力形式を採用しているから。

👉赤が点いた瞬間＝「かなり潰してる」

#### 表示レベルの校正

1. 1kHz 正弦波入力
2. Threshold を下げる
3. GR = 6dB に設定
4. LED2 が点灯するよう → LM3914の Scale Adj Trim (Ref電圧) を微調整
（厳密にやるなら SIG IN に 10kトリマーを入れる）

## 8. 電源部

- モジュール: DD1718PA (DC-DC + 7815/7915)
- 入力: DC 9V アダプタ (1A以上推奨)
- 後段フィルター (L/R共通):
 -- +15V側: 100µF (電解) || 0.1µF (セラミック) → GND
 -- -15V側: 100µF (電解) || 0.1µF (セラミック) → GND
 -- 2.2Ωの抵抗は、ノイズを「熱」に変えて遮断するダムのような役割を果たします。
- 電解コンデンサ（C3）の向きに注意！
 -- +15V側（C1）: コンデンサの「＋」をラインに、「ー」をGNDに繋ぎます。
 -- -15V側（C3）: コンデンサの**「ー」をラインに、「＋」をGNDに繋ぎます。**


```text
M315D OUT (+15V)
              │
             2.2Ω (1/4W)
              │
          ┌───┴───┐
      C1 100µF    │ C2 0.1µF (Ceramic)
        (Elect.)  │
          │       │
         GND     GND
          │       │
      +15V SYSTEM (VCA/OPAMP Pin)

------------------------------------------

       M315D OUT (-15V)
              │
             2.2Ω (1/4W)
              │
          ┌───┴───┐
      C3 100µF    │ C4 0.1µF (Ceramic)
        (Elect.)  │
          │       │
         GND     GND
          │       │
      -15V SYSTEM (VCA/OPAMP Pin)
```
