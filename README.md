# せいせいAIにたすけてもらいながらさいきょうの Stereo Compressor / Preampをつくる

> 本書は、生成AIを使用して書いたV2164（SSI2164）を用いたステレオ・コンプレッサ／プリアンプの**詳細仕様書**である。
> 今のところ日本語のみで進めています。
> プロフェッショナル機材相当の機能と音質を実現する。


[VCA IC購入先](https://ears-modular.com/products/sound-semiconductor-fatkeys%E2%84%A2-ssi2164-vca)

---

# SSI2164 VCA コンプレッサー 製作仕様書

## 1. 全体構成図（オーディオ・パス）

本機はステレオ構成の VCA コンプレッサーであり、各ブロックは以下の順序で信号処理を行う。最終出力は正相（In-Phase）とする。
当面の目標はkiCADによるデータ作成と基板発注。基板は両面ベタグラウンドを想定しており、以下の回路設計に関してもそれを想定するものとする。

Input Buffer  
→ V/I Converter  
→ VCA Core（SSI2164）  
→ I/V Converter  
→ Make-up Gain  
→ Blend（Parallel Mix）  
→ Output Buffer  

## 2. メイン基板：オーディオ・パス詳細
- A. 入力および VCA コア部
- B. 出力およびゲイン
- C. Blend（Parallel Mix）回路
## 3. サイドチェイン・ユニット（制御部）
本機は **フィードフォワード方式**のサイドチェインを採用する。  
入力信号のレベルを検波・整流し、生成された制御電圧（CV）によってSSI2164のゲインを制御する。
※本仕様は **ステレオ構成**を前提とする。 ステレオ化は、L/R 信号を合成したCVを共通でVCAに供給する「ステレオリンク方式」とする。
- A. 検波部（Precision Rectifier）
- B. Threshold（スレッショルド）設定
- C. Ratio（圧縮率）制御
- D. Attack / Release（時定数回路）
- E. CV 出力および VCA 駆動
## 4. 全体信号フロー（俯瞰）
```text
[ INPUT L/R ]
             │
             ▼
      [[ Input Buffer ]] (OPA1644等) ───┐
             │                          │
             ├─ (A) [Buffered Bypass] ──┼──────────────────────┐
             │    (4PDT Switch: DRY)    │                      │
             │                          │                      │
             ├─ (B) [SC Pick-up]        │                      │
             │       │                  │                      │
             │       ▼                  │                      │
             │  [ SC-HPF SW ] <Deep Punch>                     │
             │       │                  │                      │
             │       ▼                  │                      │
             │  [[ Sidechain Section ]] │                      │
             │   (Summing/Rect/A/R)     │                      │
             │       │                  │                      │
             │       ▼                  │                      │
             │  [[ Stereo Link / CV ]] ─┼─── [ GR METER ]      │
             │   (CV Buffer + Trim)     │                      │
             │       │                  │                      │
             ▼       │                  │                      │
      [[ V/I Conv (R_in) ]]             │                      │
             │       │ (Gain Control)   │                      │
             ▼       ▼                  │                      │
      [[ SSI2164 VCA Core ]]            │                      │
             │                          │                      │
             ▼                          │                      │
      [[ I/V Conv / Make-up ]]          │                      │
             │                          │                      │
             ▼                          │                      │
      [[ Blend Pot ]] <WET 100%> ◄──────┘                      │
             │        (Parallel Compression)                   │
             ▼                                                 │
      [[ Output Buffer ]] (OPA1644等)                          │
             │                                                 │
             ├─ [ Saturation SW ] <Warmth & Punch>             │
             │                                                 │
             ▼                                                 │
      [ 4PDT Switch ] (WET / DRY 切り替え) ◄────────────────────┘
             │
             ▼
       [ OUTPUT L/R ]

```

## 5. オーディオ・パス回路図
### Input BufferとV/I Converter（SSI2164入力）
```mermaid
flowchart TD
    IN_JACK["Input Jack L"] --> C1["C1<br/>10uF (Non-polar)"]
    C1 --> BUF_IN["+ Input (U1A)"]
    BUF_IN -->|Bias| R1["R1<br/>1MΩ"]
    R1 --> GND["GND"]
    subgraph U1A["U1A<br/>TL074 / OPA1644<br/>(Voltage Follower)"]
        BUF_IN --> OP_OUT["Output (Pin1)"]
        OP_OUT --> OP_MINUS["- Input (Pin2)"]
    end
    OP_OUT --> BLEND["BLEND_DRY_L"]
    OP_OUT --> SC["SIDECHAIN_L"]
    OP_OUT --> R2["R2<br/>20kΩ (1%)"]
    R2 --> VCA_NODE["VCA_IN_L"]
    subgraph STAB["Stability Network<br/>(SSI2164 Pin2 near)"]
        VCA_NODE --> R3["R3<br/>220Ω"]
        R3 --> C2["C2<br/>1200pF"]
        C2 --> GND
    end
```
- 構成：ボルテージフォロワ
- 出力は
  - VCA用
  - サイドチェイン用
  - Dry用へ分岐
- Stability Network (220Ω + 1200pF): SSI2164の入力端子が発振するのを防ぐための必須パーツです。SSI2164の2番ピン（および7番ピン）のできるだけ近く（5mm以内）に配置してください。
  - 初号機は 680pF / 820pF も差し替えテスト推奨
- R_in (20kΩ 1%): SSI2164は「電流入力」のICであるため、この抵抗によって電圧を電流に変換します。ステレオの音量差をなくすため、必ず精度の高い1%抵抗を使用してください。
- C_in (10uF 無極性): 入力信号に含まれる余計な直流（DC）をカットします。音質に直結するため、オーディオ用の無極性電解コンデンサ（MUSE ES等）を推奨します。
- 出力が途切れているように見えますが、後述の追加機能（HPF）のスイッチ回路のためです。


### SSI2164 VCA Core
![](vca_core.png)
```mermaid
flowchart LR
    %% ===== Power =====
    VPLUS["+15V"] --> C5["C5 0.1uF"]
    C5 --> U2_PIN16["U2 Pin16 (+V)"]
    C5 --> GND1["GND"]

    VMINUS["-15V"] --> C6["C6 0.1uF"]
    C6 --> U2_PIN9["U2 Pin9 (-V)"]
    C6 --> GND2["GND"]

    %% ===== SSI2164 Core =====
    subgraph SSI2164["U2 : SSI2164"]
        U2_PIN2["Pin2 LIN1"]
        U2_PIN3["Pin3 VC1"]
        U2_PIN4["Pin4 LOUT1"]

        U2_PIN7["Pin7 LIN2"]
        U2_PIN6["Pin6 VC2"]
        U2_PIN5["Pin5 LOUT2"]

        U2_PIN10["Pin10 LIN3"]
        U2_PIN11["Pin11 VC3"]

        U2_PIN15["Pin15 LIN4"]
        U2_PIN14["Pin14 VC4"]

        U2_PIN1["Pin1 MODE"]
        U2_PIN8["Pin8 GND"]
    end

    %% ===== Audio Inputs =====
    VCA_IN_L["VCA_IN_L"] --> U2_PIN2
    CV_IN_L["CV_IN_L"] --> U2_PIN3

    VCA_IN_R["VCA_IN_R"] --> U2_PIN7
    CV_IN_R["CV_IN_R"] --> U2_PIN6

    %% ===== Audio Outputs =====
    U2_PIN4 --> VCA_OUT_L["VCA_OUT_L_TO_IV"]
    U2_PIN5 --> VCA_OUT_R["VCA_OUT_R_TO_IV"]

    %% ===== Unused Channels Termination =====
    U2_PIN10 --> R8["R8 20k"]
    R8 --> GND3["GND"]

    U2_PIN15 --> R7["R7 20k"]
    R7 --> GND4["GND"]

    U2_PIN11 --> GND5["GND"]
    U2_PIN14 --> GND6["GND"]

    %% ===== Mode & Ground =====
    U2_PIN1 -. OPEN .- NC1["NC"]
    U2_PIN8 --> GND7["GND"]

```
- CV感度：−33mV/dB
- CV入力 (Pin 3 & 6): ステレオ・リンクさせるため、CV Buffer (OP_D) から来た同じ1本の線を、3番ピンと6番ピンの両方に接続します。
- 電源 (Pin 16 & 9):
  - 16番ピンには +15V を接続。
  - 9番ピンには -15V を接続。
  - それぞれのピンのすぐそばに、0.1µFのセラミックコンデンサを配置してGNDへ落としてください（これが無いと「キーン」という発振の原因になります）。
- 未使用チャンネル (Pin 10〜15): SSI2164は4ch入りですが、今回は2chしか使いません。
  - 入力 (10, 15番): 浮かせておくとノイズを拾うので、20kΩ程度の抵抗を介してGNDへ。
  - CV (11, 14番): 直接GNDへ。
  - 出力 (12, 13番): 何も繋がず放置（OPEN）でOKです。

```text
Pin 10 ──/\/\/\── GND (20k)
Pin 11 ────────── GND
Pin 12 ────────── (OPEN)
Pin 13 ────────── (OPEN)
Pin 14 ────────── GND
Pin 15 ──/\/\/\── GND (20k)
```

### I/V Converter & Make-up Gain (Left)
![](ivconverter_and_makeupgain.png)
```mermaid
flowchart LR
    %% ========= I/V Converter =========
    VCA_OUT[VCA_OUT_L_TO_IV] -->|R9 20k| N1
    N1 -->|C7 47pF| N2
    N2 -->|Feedback| U1B_OUT

    U1B_MINUS[U1B (-)] --> N1
    GND1[GND] --> U1B_PLUS[U1B (+)]
    U1B_PLUS --> U1B
    U1B_MINUS --> U1B
    U1B -->|Pin7| U1B_OUT[U1B OUT]

    %% ========= Make-up Gain =========
    U1B_OUT -->|R10 10k| SUM_NODE

    SUM_NODE -->|to (-)| U1C_MINUS
    GND2[GND] --> U1C_PLUS[U1C (+)]

    U1C_PLUS --> U1C
    U1C_MINUS --> U1C
    U1C -->|Pin8| U1C_OUT[MAKEUP_OUT_L]

    %% ========= Gain Control Feedback =========
    U1C_OUT -->|R11 10k| POT_R[MakeUpGain Pot 100kB (3)]
    POT_R --> POT_W[Pot Wiper (2)]
    POT_W --> U1C_MINUS

    POT_L[Pot (1) OPEN]
```

- この回路の上段の出力[IV_OUT_L]は**位相が反転し逆相となります。**
- その後後段の回路で出力[MAKEUP_OUT_L]は**再度位相が反転し正相となります。**
- Pot 1番端子はオープン（未接続）
-  入力側：R10 (10k)
  - 片方： [IV_OUT_L] (IC1 Pin 7) へ
  - もう片方： IC1 Pin 9 (−) へ※これが入力信号をオペアンプへ送る道です。
- 帰還側：R11 (10k) ＋ Pot (100k)
  - IC1 Pin 8 (出力) から出発し、R11 を通って Potの3番 へ。
  - Potの2番（ワイパー） から IC1 Pin 9 (−) へ戻る。※これで「出力→入力」のループ（帰還）が完成し、ゲイン制御が機能します。
- オペアンプ IC1 Pin 9 (−) に集まるもの（2点のみ）
  - R10 の片足（入力からの信号）
  - Pot 2番 からの線（帰還からの信号）

### Blend & Output (Left)
この構成は、Blendポットを回した際のインピーダンス変化を抑え、音量変化を最小限にするプロ仕様の構成です。
![](blend_and_output.png)
```mermaid
flowchart LR

%% ==== Inputs ====
BLEND_DRY_L[BLEND_DRY_L]
MAKEUP_OUT_L[MAKEUP_OUT_L]

%% ==== Resistors before Blend Pot ====
R15[R15 20k]
R16[R16 20k]

BLEND_DRY_L --> R15
MAKEUP_OUT_L --> R16

%% ==== Blend Pot ====
subgraph BlendPot["Blend Pot 50kB"]
    P1["Pin1"]
    P2["Pin2 (Wiper)"]
    P3["Pin3"]
end

R15 --> P1
R16 --> P3

%% ==== OpAmp U1D ====
subgraph U1D["U1D (TL074 / OPA1644)"]
    INP["+ (Pin12)"]
    INN["- (Pin13)"]
    OUT["Out (Pin14)"]
end

P2 --> INP
OUT --> INN

%% ==== Output Stage ====
R17[R17 100Ω]
OUT_JACK_L[Output Jack L]

OUT --> R17 --> OUT_JACK_L
```
- 最大 +20dB
- **位相が正相に戻る**
- （※注1）定数の決定
  - R_dry / R_wet (20kΩ): ポットの両端に保護抵抗を入れることで、ポットを回しきった時に前段のオペアンプ同士が直接ぶつかるのを防ぎ、ミキシングを安定させます。
  - Blend Pot (10kΩ〜50kΩ Bカーブ): Bカーブ（線形）を使用することで、真ん中で 50:50 のミックスになります。18k〜22kで微調整が必要かも。
  - R_out (100Ω): ケーブルを繋いだ際の静電容量による発振を防ぐための「お守り」です。
- ポット周辺の接続:
  - 1番端子: R_dry を介して原音（Input Buffer直後）と接続。
  - 3番端子: R_wet を介してメイクアップゲイン出力（エフェクト音）と接続。
  - 2番端子 (ワイパー): ここから混合された信号を取り出し、IC1の12番ピン (+) へ送ります。
- OP4 (バッファ段) の接続:
  - IC1 Pin 12 (+): ポットの2番端子（ワイパー）から信号が入ります。
  - IC1 Pin 13 (-): 自身の出力である 14番ピン と直結します（ボルテージ・フォロア）。
  - IC1 Pin 14 (Out): ここから最終的な音が出力されます。
- 最終出力:
  - 出力保護抵抗 R_out (100Ω) を通り、最終的なアウトプットジャックへ繋がります。
  
## 6. サイドチェイン回路図

### [SC-1] Summing & Rectifier (精密整流・サミング)
LとRの信号を混ぜて、検波（プラスのDCに変換）する段階です。
![](sc1.png)
```mermaid
flowchart LR

%% Inputs
SIDECHAIN_L["SIDECHAIN_L"]
SIDECHAIN_R["SIDECHAIN_R"]

%% Resistors for summing
R21["R21 47k"]
R22["R22 47k"]

%% Summing node
SUM_NODE((Summing Node))

%% Feedback resistor
R23["R23 47k"]

%% OpAmp U4A
U4A["U4A TL074\n(Summing Amp)"]

%% Rectifier stage
R24["R24 10k"]
U4B["U4B TL074\n(Precision Rectifier)"]

D1["D1 1N4148"]
D2["D2 1N4148"]
R25["R25 10k"]

%% Output
RECTIFIED_DC["RECTIFIED_DC"]

%% Connections
SIDECHAIN_L --> R21 --> SUM_NODE
SIDECHAIN_R --> R22 --> SUM_NODE

SUM_NODE -->|(-)| U4A
U4A -->|OUT| R24 --> U4B

U4A --> R23 --> SUM_NODE
U4A --> GND["GND"]

%% Rectifier loop
U4B -->|OUT| D1 --> RECTIFIED_DC
U4B -->|OUT| D2 --> R25 --> RECTIFIED_DC
U4B --> GND
```

- 1. サミングアンプ (OP_A)合流地点 (Pin 2):
  - LとRの47kΩ抵抗がここで初めて出会います。ここは「仮想接地」と呼ばれ、信号が混ざる重要なポイントです。
  - 帰還抵抗 (R_fb 47k):
    - 出力(Pin 1)から合流地点(Pin 2)に戻るこの抵抗により、ゲインが1倍（$47k/47k$）に固定され、L+Rが正確にミックスされます。
- 2. 精密整流回路 (OP_B)半波整流:
  - AC信号（音声）のマイナス成分をカットし、プラス成分だけを抽出します。
  - ダイオードの向き: * D1: 出力から順方向で外に出る（プラスを逃がす）。
    - D2: 出力から反転入力へ戻る（クランプ用）。
  - 出力 [RECTIFIED_DC]: ここで、音声信号がコンプレッサーを制御するための「電圧（DC）」へと変換されます。
  
### [SC-2] Threshold & Ratio (圧縮開始点と比率)
ここで「どれくらいの音量から、どれくらい潰すか」を決めます。
![](sc2.png)
```mermaid
graph LR
    subgraph Input_Stage [入力セクション]
        J1[Input_Jack_L] --> U1A[Buffer L: TL074]
        J2[Input_Jack_R] --> U3A[Buffer R: TL074]
        U1A --> BLEND_DRY_L
        U1A --> SIDECHAIN_L
        U3A --> BLEND_DRY_R
        U3A --> SIDECHAIN_R
    end

    subgraph Sidechain_Process [サイドチェイン制御系]
        SIDECHAIN_L & SIDECHAIN_R --> SW2[Deep Punch Switch: HPF]
        SW2 --> U4AB[SC-1: Summing & Rectifier]
        U4AB -->|RECTIFIED_DC| U4C[SC-2: Threshold & Ratio]
        U4C -->|ST_LINK_TIMING| ATK_REL[SC-3: Attack/Release Timing]
        ATK_REL -->|Point_X| U4D[CV Output Trim]
        U4D --> CV_IN_L & CV_IN_R
    end

    subgraph Metering [メーター表示]
        Point_X --> U5A[Voltage Follower]
        U5A --> U6[LM3914 LED Driver]
        U6 --> LEDs[Gain Reduction Meter]
    end

    subgraph Gain_Cell [VCA & ゲイン・セクション]
        VCA_IN_L[VCA_IN_L] --> U2[VCA: SSI2164]
        VCA_IN_R[VCA_IN_R] --> U2
        CV_IN_L & CV_IN_R --> U2
        U2 --> U1B_U3B[I/V Converter]
        U1B_U3B --> U1C_U3C[Makeup Gain]
        U1C_U3C --> MAKEUP_OUT_L & MAKEUP_OUT_R
    end

    subgraph Output_Stage [出力 & サチュレーション]
        MAKEUP_OUT_L & MAKEUP_OUT_R --> SW1[Diode Saturator Switch]
        SW1 --> BLEND_L_R[Blend Pot: Dry/Wet]
        BLEND_DRY_L & BLEND_DRY_R --> BLEND_L_R
        BLEND_L_R --> U1D_U3D[Output Buffer]
        U1D_U3D --> J3[Output_Jack_L]
        U1D_U3D --> J4[Output_Jack_R]
    end

    %% スタイル設定
    style U2 fill:#f9f,stroke:#333,stroke-width:2px
    style U6 fill:#bbf,stroke:#333
    style ATK_REL fill:#dfd,stroke:#333
```

- Threshold Pot
  - 1番 → +V（+15V）
  - 3番 → GND
  - 2番 → 10kΩの抵抗（R_th）を介して OP_C の −入力 へ接続
- OP_C の基本動作
  - +入力：RECTIFIED_DC（検波後のプラス信号）を入力
  - −入力：Threshold Pot からの入力抵抗（R_th） + Ratio Pot（帰還）
  - Output：ダイオード (D_gate: 1N4148) のアノードへ接続　→　D_gate のカソード（出力点）→　Ratio Pot を通して OP_C の −入力 へ戻します（負帰還）。
- Ratio Pot
  - 1番 → OP_C 出力
  - 3番 → OP_C の−入力
  - 2番 → OP_C の−入力（可変帰還抵抗）

### [SC-3] Stereo Link, Timing & CV Output (反転バッファ統合)
![](sc3.png)
```mermaid
graph TD
    subgraph Input_Section [Input & Buffer]
        InL[Input L] --> BufL[Input Buffer L]
        InR[Input R] --> BufR[Input Buffer R]
        BufL --> VCA_InL[VCA Input L]
        BufR --> VCA_InR[VCA Input R]
        BufL --> SC_Sum[SC Summing Input]
        BufR --> SC_Sum[SC Summing Input]
        BufL --> DryL[Dry Path L]
        BufR --> DryR[Dry Path R]
    end

    subgraph Sidechain_Path [Sidechain Control SC-1 to SC-3]
        SC_Sum --> HPF{Deep Punch SW}
        HPF --> SC1[SC-1: Summing & Rectifier]
        SC1 --> SC2[SC-2: Threshold & Ratio]
        
        subgraph Controls [SC Controls]
            Thresh[Threshold Pot] --> SC2
            Ratio[Ratio Pot] --> SC2
            Atk[Attack Pot] --> SC3
            Rel[Release Pot] --> SC3
        end

        SC2 --> SC3[SC-3: Timing C_time]
        SC3 --> PointX((Point X))
        PointX --> CVBuf[OP_D: CV Buffer / Inverter]
        CVBuf --> CVTrim[CV Trim L/R]
    end

    subgraph Metering [GR Meter]
        PointX --> MetBuf[OP_E: Meter Buffer]
        MetBuf --> LM3914[LM3914 Driver]
        LM3914 --> LED[LED Bar Graph]
    end

    subgraph VCA_Core [Audio Path: VCA & Gain]
        VCA_InL --> VCA[SSI2164 VCA]
        VCA_InR --> VCA
        CVTrim -->|Negative CV| VCA
        VCA --> IV[I/V Converter]
        IV --> Makeup[Makeup Gain Pot]
    end

    subgraph Output_Mix [Output & Blend]
        Makeup --> Saturate{Saturation SW}
        Saturate --> WetPath[Wet Path]
        DryL & DryR --> Blend[Blend Pot: Dry/Wet]
        WetPath --> Blend
        Blend --> OutBuf[Output Buffer]
        OutBuf --> Bypass{4PDT Bypass SW}
        Bypass --> OutL[Output L]
        Bypass --> OutR[Output R]
    end

    %% スタイル定義
    style VCA fill:#f9f,stroke:#333,stroke-width:2px
    style PointX fill:#fff4dd,stroke:#d4a017,stroke-width:2px
    style Sidechain_Path fill:#e1f5fe,stroke:#01579b
    style Metering fill:#f3e5f5,stroke:#4a148c
```
- ステレオリンク: 左右の検波信号が OP_A でサミングされ、1つの C_time（タイミング用コンデンサ）を共有して1つの OP_D（CVバッファ）から左右のVCAへ送られるため、完璧に同期します。
- D_atk(1N4148)
  - アノード：入力側
  - Attack Pot 側
- 2連ポット端子番号接続先備考
  - Attack3番（CCW）D_atk カソード充電経路の入力
  - Attack2番（Wiper）Point XC_timeへ
  - Attack1番（CW）OPEN未接続
  - Release3番（CCW）Point XC_timeから
  - Release2番（Wiper）GND放電経路の出口
  - Release1番（CW）OPEN未接続
- OP_Dの接続
  - 非反転入力（＋）：GND
  - 反転入力（−）：
    - Point X（C_time ノード）からR_inを通して-入力に入れる
    - OP_Dの出力はR_fb (10k)を通して-入力に入れる
    - これにより、**仮想接地**が形成される
  - OP_D 出力
    - それぞれ精密校正のため、LR用に2つに分岐し10kトリムを通してVCA IC（CV IN）へ
- 要テスト。Point X（C_timeの+側）:
  - 無信号時 = 0V
  - 圧縮発生時 = +0.2V〜+1.5V（プラス電圧）
- VCAへのCV供給
  - SSI2164 Pin 3 (L ch CV)
  - SSI2164 Pin 6 (R ch CV)
## 7. GRメーター回路
### 構成概要
- IC: LM3914 (Linear Scale) Mode: Bar Mode (Dot modeの場合はPin9をオープン)─（既存）CV Trim → SSI2164
###GR Meter 入力回路（非反転バッファ）
GRメーター表示用に、Point X（C_timeノード）から信号を分岐します。Point Xは正電圧（0V〜+1.5V）であり、LM3914の入力仕様に適合するため、負荷分離のためのボルテージフォロワのみを挿入します。
![](grmeter1.png)
- **Point X** (C_timeコンデンサの+側)
  - 電圧範囲: 0V〜+1.5V (圧縮量に比例)
  - 分岐先1: OP_D (CV Buffer) → VCA制御用（負電圧生成）
  - 分岐先2: OP_M (Meter Buffer) → LM3914表示用（正電圧維持）
  - 信号側のノイズ遮断用にOP_M 出力に 100Ω + 1µF追加
- 動作
  - 無音時: Point X = 0V → OP_M出力 = 0V → LED全消灯
  - 圧縮時: Point X = +0.2V〜+1.5V → OP_M出力 = +0.2V〜+1.5V → LED点灯数増加
### LM3914 基本配線
![](grmeter2.png)
※ LEDには直列抵抗不要（LM3914内蔵）
- 電源・基本設定:
  - Pin 2 (V-): GNDに接続。
  - Pin 3 (V+): +15Vに接続（C11 0.1µF でパスコンが入っているのもGoodです）。
  - Pin 4 (RLO): GNDに接続。
  - Pin 9 (MODE):
  - Pin 3 (+15V) に直結。 → バー表示モード。
- 信号入力:
  - Pin 5 (SIG): LM3914_Pin5（バッファ出力）を接続。
- 基準電圧・スケール調整:
  - Pin 7 (REFOUT): Scale_Adjust_Trim の 1番端子に接続。
  - Pin 8 (REFADJ): R32（2.2kΩ）を介してGNDに接続。
  - Pin 6 (RHI): Scale_Adjust_Trim の 2番端子（ワイパー）に接続。
- 表示部 (LED1〜10):
  - Pin 1, 18, 17, 16, 15, 14, 13, 12, 11, 10: 各LEDのカソード (K) を接続。
  - 各LEDのアノード (A): すべてまとめて+15Vに接続
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
 - +15V側: 100µF (電解) || 0.1µF (セラミック) → GND
 - -15V側: 100µF (電解) || 0.1µF (セラミック) → GND
 - 2.2Ωの抵抗は、ノイズを「熱」に変えて遮断するダムのような役割を果たします。
- 電解コンデンサ（C3）の向きに注意！
 - +15V側（C1）: コンデンサの「＋」をラインに、「ー」をGNDに繋ぎます。
 - -15V側（C3）: コンデンサの**「ー」をラインに、「＋」をGNDに繋ぎます。**
![](filter_for_psu.png)
## 9. 4PDT バッファード・バイパス
- ※この段はシンプル化のため削除しました。電源が投入されればONになります。

## 10. 【追加機能】
- ステレオの実装:
  - SC-HPF: L/Rのサミング前にそれぞれコンデンサを入れます。2極のスイッチ（DPDT）を使えば、一つのスイッチでL/R同時にフィルターのON/OFFが可能です。
  - Saturation: これもL/RそれぞれのOP4にダイオードペアを配置します。同じく2極のスイッチ（DPDT）で一括管理します。
### スイッチ配線図
- サイドチェイン・ハイパスフィルター(Deep Punch)用配線図
キックの低域に反応させない「Deep Punch」モードを切り替えるスイッチです。
- [SC-1] Summing & Rectifier の直前、または [TO_SIDECHAIN_L/R] のラインに挿入します。（L/R2回路必要）
- 仕組み: スイッチが 開いている（OFF） とき、信号は 0.1µF のコンデンサを通ります。後段の抵抗（47kΩ）と組み合わさり、約 34Hz 以下 をカットするハイパスフィルターになります。
- 定数の変更: もしもっと低域を逃がしたい（キックの重低音をより活かしたい）場合は、コンデンサを 0.047µF（約72Hzカット） に変更してください。
- 効果: スイッチを「コンデンサ経由」側に倒すと、低域にコンプが過剰反応しなくなり、重心の低いパンチのある音になります。
![](dps.png)
- サチュレーター (Saturation) 用配線図
「Warmth & Punch」を加える、ダイオード・クリッピングのON/OFFスイッチです。
最終段の [Output Buffer (OP4)] の帰還（フィードバック）ループ内に挿入します。（L/R2回路必要）
![](saturator_switch.png)

※1 [Diode Section] の詳細: 2つのダイオードを逆向きに並列接続します（対称クリッピング）。
```text
        ┌─────►|─────┐
  ──────┤ (1N34A等)  ├──────
        └─────|<─────┘
```
- 仕組み: スイッチを ON にすると、出力電圧がダイオードの順方向電圧（ゲルマニウムなら約0.3V）を超えた瞬間に波形がソフトに潰れます。
- 推奨ダイオード:
  - 1N34A (ゲルマニウム): 非常にウォームでヴィンテージな質感。
  - BAT41 (ショットキー): 1N34Aより少し派手で、パンチのあるサチュレーション。
- 効果: メイクアップ・ゲインを上げた際に、デジタルのような嫌な割れ方ではなく、音楽的な「粘り」と「太さ」が加わります。
