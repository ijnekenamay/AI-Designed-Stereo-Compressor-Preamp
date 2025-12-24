# せいせいAIにたすけてもらいながらさいきょうの Stereo Compressor / Preampをつくる

> 本書は、生成AIを使用して書いたV2164（SSM2164）を用いたステレオ・コンプレッサ／プリアンプの**詳細仕様書**である。
> 今のところ日本語のみで進めています。
> プロフェッショナル機材相当の機能と音質を実現する。
---

# SSI2164 VCA コンプレッサー 製作仕様書

---

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

---

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

---

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

---

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

---

## 3. サイドチェイン・ユニット（制御部）

本機は **フィードフォワード方式**のサイドチェインを採用する。  
入力信号のレベルを検波・整流し、  
生成された制御電圧（CV）によって SSI2164 のゲインを制御する。

※本仕様は **モノラル構成**を前提とする。  
将来的にステレオ化する場合は、L/R 信号を合成した CV を  
共通で VCA に供給する「ステレオリンク方式」へ拡張可能とする。

---

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

---

### B. Threshold（スレッショルド）設定

- Threshold Pot  
  10kΩ（B）

- 動作  
  - 整流電圧が Threshold を超えた分のみを有効成分とする  
  - Threshold 未満では圧縮動作を行わない（ユニティゲイン）

---

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

---

### D. Attack / Release（時定数回路）

- タイミングコンデンサ  
  10µF（電解またはタンタル）

- Attack  
  - Pot：50kΩ（A）  
  - 最小値制限抵抗：100Ω  
  - 時定数範囲：約 1ms ～ 500ms

- Release  
  - Pot：1MΩ（A）  
  - 時定数範囲：約 10ms ～ 10s

---

### E. CV 出力および VCA 駆動

- CV バッファ  
  オペアンプによるボルテージフォロワ

- CV 出力先  
  SSI2164 Pin 3（CV入力）

- CV Scaling  
  - 出力直列に 10kΩ 半固定抵抗を挿入  
  - SSI2164 の −33mV/dB 特性に合わせて
    実機でリダクション量を微調整可能とする

---

---

## 4. 全体信号フロー（俯瞰）

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

---

---

## 5. オーディオ・パス回路図

---

---

### Input Buffer
          IN
           │
           │
          1M
           │
           ▼
        ┌─────┐
        │  +  │
        │ OP1 │───────┬───────→ Dry
        │  -  │       │
        └──┬──┘       │
           │           │
           └───────────┘
                 GND
                 
- 構成：ボルテージフォロワ
- 出力は
-- VCA用
-- サイドチェイン用
-- Dry用へ分岐

---

---

### V/I Converter（SSI2164入力）

OP1 OUT
  │
  │   20k
  ├──/\/\/\───┬───────────→ SSI2164 Pin2
  │            │
  │           220
  │            │
  │          1200p
  │            │
  └────────────┴─────────── GND


- Stability Network：必須
- Pin1（Mode）：OPEN

---
---

### SSI2164 VCA Core

          CV (from sidechain)
                │
                ▼
           SSI2164
         ┌─────────┐
Pin2 ◄───┤ IN       │
Pin3 ◄───┤ CV       │
Pin4 ───►┤ OUT      │
         └─────────┘



- CV感度：−33mV/dB

---

---

### I/V Converter

SSI2164 Pin4
      │
      ▼
     20k
      │
      ▼
   ┌─────┐
   │  -  │
   │ OP2 │───────────┐
   │  +  │           │
   └──┬──┘           │
      │               │
      └──── GND       │
                      │
                    47p
                      │
                      └─────── OP2 OUT


- 位相反転（後段で戻す）

---

---

### Make-up Gain

OP2 OUT
  │
  │ 10k
  ├─/\/\/\──┐
  │         │
  │       ┌─▼───┐
  │       │  -  │
  │       │ OP3 │───────────→ Wet
  │       │  +  │
  │       └─┬───┘
  │         │
  │        GND
  │
  └─/\/\/\─/\/\/\─┘
    10k     100k(B)


- 最大 +20dB
- 位相が正相に戻る

---

---

### Blend（Parallel Mix）

Dry ──┐
      │
     ┌┴─────┐
     │  B   │ 25k / 50k(MN)
     │ POT  │
     └┬─────┘
      │
Wet ──┘
      │
      ▼
   ┌─────┐
   │ OP4 │  Voltage Follower
   └──┬──┘
      │
     100
      │
    OUTPUT

- 最大 +20dB
- 位相が正相に戻る

---

---

## 6. サイドチェイン回路図

---

---

### サイドチェイン入力 & 整流

Input Buffer OUT
      │
      │ 47k
      ├─/\/\/\──┐
      │          │
      │        ┌─▼───┐
      │        │ OP5 │
      │        └─┬───┘
      │          │
      └──────────┘
                 │
        Precision Full-Wave
           Rectifier
        (1N4148 x2)
                 │
                 ▼
              Rectified DC

---

---

### Threshold & Ratio

Rectified DC
     │
     ├───────────────┐
     │               │
    Thr            Ratio
   10k(B)         50k(B)
     │               │
     └─────┬─────────┘
           ▼
        Gain Stage
      (Variable k)

---

---

### Attack / Release

Gain Stage OUT
      │
      ├─►|──┐  Attack
      │      │
      │     50k(A)
      │      │
      │     100Ω
      │      │
      │     ┌─▼─┐
      │     │10µ│
      │     └─┬─┘
      │       │
      └─|◄────┘  Release
            │
           1M(A)
            │
           GND

---

---

### CV Buffer & Scaling

A/R OUT
   │
   ▼
┌─────┐
│ OP6 │  Voltage Follower
└──┬──┘
   │
  10k (Trim)
   │
   ▼
SSI2164 Pin3

---

---

## 7. GRメーター回路

---

---

### Gain Reduction Meter（圧縮量表示）

---

---

#### 構成概要

CV Buffer OUT
     │
     ├─── 100k ───► LM3915 SIG IN
     │
     └───（既存）CV Trim → SSI2164

---
---

#### 回路への組み込み位置

           ┌─────────────┐
           │ Attack/Rel. │
           └─────┬───────┘
                 │
            ┌────▼─────┐
            │ CV Buffer│
            └────┬─────┘
                 │
         ┌───────┴────────┐
         │                │
   ┌─────▼─────┐    ┌─────▼─────┐
   │ CV Trim   │    │ LM3915     │
   │ 10k       │    │ GR Meter   │
   └─────┬─────┘    └───────────┘
         │
 SSI2164 Pin3


---
---

#### LM3915 基本配線

                +V (例: +12V)
                  │
                 1k8
                  │
                  ▼
                REF OUT (Pin7)
                  │
                 10k
                  │
                 GND

CV Buffer OUT
      │
     100k
      │
      ▼
SIG IN (Pin5)

MODE (Pin9) ── GND     ← Dot mode
           or +V       ← Bar mode

LED1 (Pin10) ── Green
LED2 (Pin11) ── Green
LED3 (Pin12) ── Yellow
LED4 (Pin13) ── Yellow
LED5 (Pin14) ── Red
LED6 (Pin15) ── Red
(LED7/8 optional)

GND (Pin2) ─── GND

※ LEDには直列抵抗不要（LM3915内蔵）

---

---

#### 表示ロジック

SSI2164 の特性 [−33mV = 1dB GR]

| LED  | 表示GR   | 色 |
| ---- | ------ | - |
| LED1 | −3 dB  | 緑 |
| LED2 | −6 dB  | 緑 |
| LED3 | −9 dB  | 黄 |
| LED4 | −12 dB | 黄 |
| LED5 | −15 dB | 赤 |
| LED6 | −18 dB | 赤 |

👉赤が点いた瞬間＝「かなり潰してる」

---

---

#### 表示レベルの校正

1. 1kHz 正弦波入力
2. Threshold を下げる
3. GR = 6dB に設定
4. LED2 が点灯するよう → LM3915入力側の100k or REF側を微調整
（厳密にやるなら SIG IN に 10kトリマーを入れる）

---

