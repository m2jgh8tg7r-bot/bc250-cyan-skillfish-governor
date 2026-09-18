# BC-250 SMU 周波数遷移の安全化

## 概要

AMD BC-250 / Cyan Skillfish で `cyan-skillfish-governor-smu` を使用した際、
GPU高負荷中のダウンクロックでシステム全体がクラッシュする問題を調査した。

調査の結果、低クロック・低電圧設定そのものではなく、
高クロック状態から低クロック状態へ遷移する際の
周波数と電圧の変更順序、および変更間隔が重要であることが分かった。

今回の修正版では次の順序を使用する。

アップクロック:

~~~text
VIDを先に上げる
    ↓
周波数を上げる
~~~

ダウンクロック:

~~~text
周波数を先に下げる
    ↓
2 ms待機
    ↓
VIDを下げる
~~~

さらに、SMUから毎回取得する瞬間的な周波数値ではなく、
最後に正常に設定した周波数 `last_freq` を追跡して
アップクロックかダウンクロックかを判定する。

## 基準バージョン

上流:

- cyan-skillfish-governor v0.4.12
- upstream commit:
  `be9537fc36f24b17570088cafa8c79365f80fee8`

BC-250用 known-good コード:

- commit:
  `90a9d44b2c65c632e4e4e27f38f443e33e809437`
- tag:
  `bc250-safeorder-2ms-known-good-20260918`

検証バイナリ:

`cyan-skillfish-governor-smu v0.4.12-safeorder-2ms-track-debug3`

## 発生していた問題

FurMarkでGPUに高負荷を掛けた状態から周波数を下げると、
システム全体がクラッシュする場合があった。

特に次の遷移で再現しやすかった。

~~~text
1600 MHz → 1300 MHz
1600 MHz → 1200 MHz
~~~

一方、最初から

~~~text
1200 MHz / 700 mV
~~~

に固定した状態ではFurMarkを正常に実行できた。

このため、低クロック・低電圧の動作点そのものではなく、
そこへ移動する途中の遷移に問題がある可能性が高いと判断した。

## 元のガバナーの動作

上流版では周波数変更時に常に次の順序だった。

~~~rust
self.smu.force_gfx_vid(vol)?;
self.smu.force_gfx_freq(freq)?;
~~~

つまり、

~~~text
VID変更
   ↓
周波数変更
~~~

である。

アップクロックでは合理的だが、ダウンクロックでも同じため、
高い周波数のまま先に電圧を下げる瞬間が発生する可能性がある。

## 待機時間の実験

ダウンクロックを

~~~text
FREQを下げる
   ↓
待機
   ↓
VIDを下げる
~~~

という順序に変更してFurMark下で検証した。

| 待機時間 | 結果 |
|---|---|
| 0 ms | クラッシュ |
| 0.4 ms | クラッシュ |
| 0.6 ms | 2サイクル通過後、3サイクル目でクラッシュ |
| 0.65 ms | 10サイクル / 20回の下降遷移を完走 |
| 0.70 ms | 5サイクル完走 |
| 0.75 ms | 10サイクル完走 |
| 1.5 ms | 完走 |
| 3 ms | 完走 |
| 6.25 ms | 完走 |
| 12.5 ms | 完走 |
| 25 ms | 完走 |
| 50 ms | 完走 |

0.6～0.7 ms付近に安定性が変化する領域が見られた。

これはBC-250内部の正確なハードウェア仕様を証明するものではないが、
周波数低下直後にVIDを下げず、短い待機時間を置く必要があることを
強く示唆している。

常用向けには余裕を取って2 msを採用した。

## 実装

主要な変更は次の通り。

~~~rust
if freq < current_freq {
    self.smu.force_gfx_freq(freq)?;
    std::thread::sleep(std::time::Duration::from_millis(2));
    self.smu.force_gfx_vid(vol)?;
} else {
    self.smu.force_gfx_vid(vol)?;
    self.smu.force_gfx_freq(freq)?;
}
~~~

## SMU周波数readbackの異常

ストレス試験中、`get_gfx_frequency()` が一度だけ

~~~text
6 MHz
~~~

という不自然な値を返した。

実際の動作状態は1300～1600 MHz付近だったため、
この瞬間値だけで遷移方向を判定すると誤判定する可能性がある。

例えば実際には1600 MHzなのに6 MHzと読まれた場合、
1200 MHzへの変更を本来のダウンクロックではなく
アップクロックとして扱う危険がある。

## last_freqによる追跡

そこで `SmuFreqStrategy` に

~~~rust
last_freq: u32
~~~

を追加した。

遷移方向は、

~~~rust
let current_freq = self.last_freq;
~~~

で判定する。

周波数・電圧変更が正常終了した後だけ、

~~~rust
self.last_freq = freq;
~~~

として更新する。

## 起動時readbackの検証

起動時だけはSMUから初期周波数を読む必要がある。

そこで、

~~~text
350 ～ 2100 MHz
~~~

の範囲のみを妥当な値として採用する。

異常値の場合は2 ms待って再取得し、
最大10回試行する。

妥当な値を取得できなければ、
不正な状態で起動せず初期化を失敗させる。

## 検証結果

### Rust TestMode

FurMark実行中に20サイクルを実施し、
合計40回の高負荷ダウンクロックを完走した。

### 通常自動ガバナー

TestModeを使わず、
通常のbusy-flag負荷判定を使用した。

FurMarkを `SIGSTOP` / `SIGCONT` で停止・再開し、
GPU負荷のON/OFFを繰り返した。

上限1600 MHzの試験設定で、
1600 MHzから350 MHzまでの段階的下降と、
350 MHzから1600 MHzへの復帰を繰り返し完走した。

### systemd・通常設定

修正版を

`/usr/local/libexec/cyan-skillfish-governor-smu-safeorder`

に配置し、systemd drop-inから使用した。

標準の

`/usr/bin/cyan-skillfish-governor-smu`

は上書きしていない。

通常設定の上限1500 MHzで
FurMark負荷ON/OFFを10サイクル実施した。

各サイクルでは、

~~~text
1500
 ↓
1400
 ↓
1300
 ↓
1200
 ↓
1100
 ↓
1000
 ↓
900
 ↓
800
 ↓
700
 ↓
600
 ↓
500
 ↓
400
 ↓
350 MHz
~~~

と下降した。

10サイクルで合計120回の自動ダウンクロックが記録され、
すべて `FREQ → 2 ms → VID` の順序で完走した。

PC再起動後もsystemdから修正版ガバナーが正常に自動起動した。

## 現在の構成

修正版:

`/usr/local/libexec/cyan-skillfish-governor-smu-safeorder`

設定:

`/etc/cyan-skillfish-governor-smu/config.toml`

systemd drop-in:

`/etc/systemd/system/cyan-skillfish-governor-smu.service.d/20-safeorder.conf`

標準版:

`/usr/bin/cyan-skillfish-governor-smu`

## 標準版へ戻す方法

~~~bash
sudo systemctl stop cyan-skillfish-governor-smu.service

sudo rm \
  /etc/systemd/system/cyan-skillfish-governor-smu.service.d/20-safeorder.conf

sudo systemctl daemon-reload
sudo systemctl start cyan-skillfish-governor-smu.service
~~~

標準バイナリを上書きしていないため、
drop-inを削除するだけで元に戻せる。

## 注意点

最終試験時の通常設定では、
1300 MHz付近の補間電圧は約710 mVだった。

以前の直接安定性試験では1300 MHz / 730 mV付近も使用していた。

これは今回の周波数・電圧遷移順序とは別件として検証する。

また、2 msはBC-250の公式仕様ではなく、
今回の実測結果から安全側に選択した経験的な待機時間である。
