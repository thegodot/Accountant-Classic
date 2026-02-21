```mermaid
flowchart TD
  %% Trader's Tender (2032) Lifecycle

  subgraph LOGIN_PRIME[登录后基线检查]
    LP1[PrimeDiscoveredCurrenciesOnLogin] --> LP2{2032?}
    LP2 -->|否| LP3[继续下一个]
    LP2 -->|是| LP4[读取 liveAmt 和 Total]
    LP4 --> LP5{liveAmt 和 Total 差异?}
    LP5 -->|是| LP6[ApplyTotalOnlyBaselineDelta]
    LP6 --> LP7[更新 lastCurrencyAmounts]
    LP5 -->|否| LP8[InitializeCurrencyData]
    LP8 --> LP7
    LP7 --> LP9[primedCurrencies = true]
  end

  subgraph FIRST_EVENT[首次 CURRENCY_DISPLAY_UPDATE]
    FE1[收到事件参数] --> FE2{2032?}
    FE2 -->|否| FE3[走通用流程]
    FE2 -->|是| FE4[HandleZeroChangeCurrency]
    FE4 --> FE5{已 prime?}
    FE5 -->|是| SE1
    FE5 -->|否| FE6{quantityChange 非零?}
    FE6 -->|是| FE7[pre = new - quantityChange]
    FE7 --> FE8[base = Total 或 0]
    FE8 --> FE9[ApplyTotalOnlyBaselineDelta]
    FE9 --> FE10[TrackCurrencyChange quantityChange]
    FE6 -->|否| FE11[base = Total 或 0]
    FE11 --> FE12[ApplyTotalOnlyBaselineDelta]
    FE12 --> FE13[更新快照]
    FE10 --> FE13
    FE13 --> FE14[primedCurrencies = true]
  end

  subgraph SUBSEQUENT_EVENT[后续 CURRENCY_DISPLAY_UPDATE]
    SE1[HandleZeroChangeCurrency 已 prime] --> SE2{quantityChange == 0?}
    SE2 -->|是| SE3[delta = new - last]
    SE3 --> SE4{delta == 0?}
    SE4 -->|是| SE5[仅更新快照]
    SE4 -->|否| SE6[记录元数据]
    SE6 --> SE7[TrackCurrencyChange delta]
    SE7 --> SE8[更新快照]
    SE2 -->|否| SE9[delta = new - last]
    SE9 --> SE10[记录元数据]
    SE10 --> SE11[TrackCurrencyChange delta]
    SE11 --> SE8
  end

  LP9 --> FE1
  FE14 --> SE1
