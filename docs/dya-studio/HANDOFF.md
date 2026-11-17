# 作業引継書: moNa2 (PAW3222 版) を DYA Studio レベル2 に対応させる

作成: 2026-11-18 / 作成元: zmk-config-moNa2-v2（PMW3610 版）で同じ対応を完了したセッション

作業手順と完了条件は [INSTRUCTIONS.md](INSTRUCTIONS.md) にある。この文書はその前提となる背景・調査結果・落とし穴をまとめる。

---

## 1. ゴール

[DYA Studio](https://studio.dya.cormoran.works)（cormoran 氏の Web 設定ツール。ZMK Studio 互換 + 独自 RPC）で、このキーボードのキーマップ・マクロ・コンボ・トラックボール・BLE 接続・設定・診断を変更できるようにする。

DYA Studio 開発者ガイドの区分で **レベル2**（cormoran の ZMK fork + 既存モジュールの組み合わせ）。レベル3（独自 RPC モジュールの自作）はやらない。

## 2. 先行事例: PMW3610 版での実施内容（完了・実機確認済み）

- リポジトリ: `dgarai/zmk-config-moNa2-v2`（ローカル: `/Volumes/SANZANG/Develop/zmk-config-moNa2-v2`）
- PR: https://github.com/dgarai/zmk-config-moNa2-v2/pull/2 （マージ済み、`9903757`）
- 実装コミット: `cceb722`。**このコミットの差分がそのまま雛形になる。** まず `git -C /Volumes/SANZANG/Develop/zmk-config-moNa2-v2 show cceb722` を読むこと
- ユーザーが実機で確認済み（「バッチリでした」）。キー、エンコーダ、左右接続、トラックボール、DYA Studio の各タブまで問題なし

### やったこと（PMW3610 版）

| ファイル | 変更 |
|---|---|
| `config/west.yml` | ZMK を `cormoran/zmk` `main+dya` に、Zephyr を `cormoran/zephyr` `v4.1.0+zmk-fixes+nrf-half-duplex-uart` に差し替えた。DYA Studio 用モジュール 15 個を追加し、rgbled-widget を `main` に変えた。badjeff PMW3610 / kot149 scroll-snap / 未使用の keybind は削除 |
| `build.yaml` | `seeeduino_xiao_ble` → `xiao_ble//zmk` |
| `.github/workflows/build.yml` | `build-user-config.yml@v0.3.0` → `@main` |
| `boards/shields/mona2/Kconfig.defconfig` | `config NFCT_PINS_AS_GPIOS` を削除（Zephyr 4 で廃止） |
| `boards/shields/mona2/mona2.dtsi` | `&uicr { nfct-pins-as-gpios; };` を追加。listener の `input-processors` を削除（overlay 側で設定） |
| `boards/shields/mona2/mona2_r.overlay` | ドライバを `cormoran,pmw3610` に替えた。listener を runtime input processor（`&scroll_runtime_input_processor &mouse_runtime_input_processor`）に移した。DYA Studio 表示用の `cormoran,physical-layout-trackball` ノードを追加 |
| `boards/shields/mona2/mona2.zmk.yml` | `requires: [seeed_xiao]` |
| `config/mona2_r.conf` | 各モジュールの `CONFIG_*` と `*_STUDIO_RPC`、推奨のスタック・バッファ値 |
| `config/mona2_l.conf` | 周辺側に必要な最小限（relay / custom-settings / settings-rpc / `CONFIG_CONSOLE=n`） |
| `config/mona2.keymap` | **変更なし**（後述） |

CI の結果は右 FLASH 50% / RAM 67%、左 FLASH 27% / RAM 30%。

## 3. PAW3222 版との違い（調査済み）

`origin/main` の時点（2026-11-18）で確認した内容。**ローカルの作業コピーは origin より 40 コミット以上遅れているので、着手前に必ず pull すること。**

### 3.1 センサードライバ

- 現在は `sekigon-gonnoc/zmk-driver-paw3222` の `main`（compatible `pixart,paw3222`、`res-cpi = <950>`）を使っている
- **PAW3222 用の DYA Studio 対応ドライバ（Custom Studio RPC 付き）は存在しない。** cormoran 氏のリポジトリ一覧で確認した。cormoran 製は PMW3610 版だけ
- 結論: **ドライバは sekigon 版をそのまま使う。** DYA Studio の Trackball タブのうち、次のように分かれる
  - 使える: runtime input processor の部分（感度・回転・反転・スナップ・スクロール・オートマウス・有効レイヤー）
  - 使えない: センサー固有の部分（CPI、省電力など）は「未対応」表示になる。CPI は従来どおり overlay の `res-cpi` で決める
- sekigon 版の `main` は ZMK `main` 向け。README の west.yml 例が `zmk main` で、2025-12 に「Zephyr の sign_extend と衝突するため改名」というコミットがある。ZMK v0.3 向けは `v0.2` ブランチ。そのため Zephyr 4.1 でもビルドできる見込みだが、**未検証**

### 3.2 トラックボールの入力処理（現状を正確に読んだ結果）

`mona2.dtsi` の listener には `<&zip_xy_transform INPUT_TRANSFORM_Y_INVERT>, <&zip_temp_layer 4 500>` が書かれている。しかし `mona2_r.overlay` が `input-processors` を **上書き** しているため、実際に効いているのは overlay 側だけ。

| 用途 | 現状（overlay の実効値） |
|---|---|
| ポインター | ドライバ出力そのまま（反転なし）+ `zip_threshold_temp_layer 4 500`（gohanda11 製、`activation-threshold = <100>`：100 カウント動かしてから MOUSE レイヤー(4)へ、500ms で解除） |
| スクロール（`scroller`、layers = 5） | `zip_xy_to_scroll_mapper` → `zip_scroll_transform INPUT_TRANSFORM_Y_INVERT` → `zip_scroll_snap`（x/y-threshold 1/3）→ `zip_scroll_scaler 1 10` |

runtime input processor に移すと、次の値が旧構成と等価になる（v2 と同じ考え方。runtime processor の xy→scroll はコードの付け替えだけで、符号は変えないことをソースで確認済み）。

- mouse: 反転なし
- scroll: `x-codes = <INPUT_REL_X>; y-codes = <INPUT_REL_Y>; xy-to-scroll-enabled; y-invert; active-layers = <(1 << 5)>; scale 1/10; axis-snap-mode = Y`

### 3.3 要判断: オートマウスの「移動量しきい値」

**PAW3222 版だけ** gohanda11 の threshold-temp-layer で「100 カウント動くまでオートマウスに入らない」ようにしている（センサーの微小な揺れで MOUSE レイヤーに入るのを防ぐ目的と思われる）。

cormoran の runtime input processor の temp-layer には **移動量しきい値がない**。あるのは「キー入力から N ms 以内は発動しない」（`temp-layer-activation-delay-ms`、既定 100）だけ。`feat/input-threshold` ブランチに `small-input-threshold`（小さい入力を捨てる）があるが未マージで、目的も違う。

選択肢:

- **A（推奨）: gohanda11 の threshold-temp-layer を残す**
  - オートマウスは gohanda11、感度・スクロール・スナップは runtime processor が担当する
  - mouse runtime processor の `temp-layer-enabled` は付けない
  - 旧来の挙動を保てる。ただし DYA Studio のオートマウス設定は効かない
  - gohanda11 モジュールが Zephyr 4.1 でビルドできるかは **未検証**
- **B: runtime processor の temp-layer に一本化する**
  - DYA Studio からオートマウスを調整できる。しかし移動量しきい値がなくなり、微小な揺れでオートマウスに入るかもしれない

A でビルドが通らなければ B に切り替える。どちらにするかは実装前にユーザーに確認すること（INSTRUCTIONS.md の手順 2）。

### 3.4 その他の差分

| 項目 | PAW3222 版の現状 | 対応 |
|---|---|---|
| `prospector-zmk-module`（t-ogura, v2.0.0） | west.yml にあるが、`CONFIG_ZMK_STATUS_ADVERTISEMENT` などは全部コメントアウト。未使用 | 削除してよい（Zephyr 4.1 対応も不明なため）。消す前に `git grep -i prospector` で参照がないことを再確認 |
| `zmk-input-processor-keybind`（zettaface） | 参照なし。未使用 | 削除 |
| `zmk-scroll-snap`（kot149） | 使用中 | runtime processor の axis-snap に置き換えて削除 |
| `Kconfig.defconfig` | `INPUT` / `ZMK_POINTING` / `SPI` / `PAW3222` を **左右両方** で `default y` にしている。`NFCT_PINS_AS_GPIOS` あり | `NFCT_PINS_AS_GPIOS` を削除する（必須）。他はビルドが通ればそのままでよい |
| `CONFIG_ZMK_KEYBOARD_NAME` | `"moNa2-paw3222"`（13 バイト） | そのまま（BLE 名は 16 バイト以内） |
| `CONFIG_ZMK_KSCAN_DEBOUNCE_PRESS_MS=0` | v2 ではチャタリング疑いで削除済み | 今回のスコープ外。触らない（ユーザーに報告だけ） |
| 物理配置・キー数 | v2 と同じ 42 キー | トラックボール表示ノードは v2 と同じ座標でよい |

## 4. 落とし穴（v2 で実際に踏んだもの）

1. **ローカルビルドをしない。** Mac mini の起動ディスクは空きが少なく（約 2.7GB）、Docker + west の Zephyr 一式でディスクが満杯になり、セッション全体のコマンドが止まった。ビルド確認は **GitHub Actions** で行う
2. **push は SSH で。** HTTPS 側の認証情報には `workflow` スコープがなく、`.github/workflows/build.yml` を変える push が拒否される。`git push git@github.com:dgarai/zmk-config-moNa2-paw3222.git <branch>` を使う
3. **リポジトリ名が変わっている。** GitHub 上は `dgarai/zmk-config-moNa2-paw3222`。ローカルのディレクトリと origin の URL は旧名の `zmk-for-moNa2-paw3222` のまま（GitHub がリダイレクトする）
4. **`config/mona2.keymap` は触らない。** v2 リポジトリの `sync-keymap.yml` が、v2 の keymap をこのリポジトリの `main` に自動で上書き push する（KEYMAP-SYNC.md 参照）。プロセッサ設定などを keymap に書くと次の同期で消える。トラックボール関係はすべて `mona2_r.overlay` に書く
5. **ビルド警告は無視してよい。** cormoran 製モジュール由来の `implicit declaration of function 'zmk_split_central_send_relay_event'` や `-Warray-bounds` などが出るが、v2 では実機で問題がなかった
6. **既存ファームからの書き換えは `settings_reset` を先に書く。** BLE ペアリングが消えるので、ホストとの接続はやり直しになる。ユーザー向けの手順に必ず含める
7. `CONFIG_ZMK_STUDIO_LOCKING=n` は既存の設定のまま残す（`&studio_unlock` は keymap にないため）

## 5. 参考資料

- DYA Studio 開発者ガイド: https://studio.dya.cormoran.works/developer-guide （レベル2 とモジュール別ページ）
- サンプル: https://github.com/cormoran/zmk-config-dya-studio-sample （機能ごとに PR が分かれている）
- DYA2（分割キーボードの参照実装）: https://github.com/cormoran/zmk-keyboard-dya2 （`config/west-dependency.yml`、`boards/shields/dya2/dya2_right.conf`、`dya2_left.conf`、`dya2.dtsi`）
- runtime input processor: https://github.com/cormoran/zmk-module-runtime-input-processor （binding: `dts/bindings/input_processors/zmk,input-processor-runtime.yaml`）
- PAW3222 ドライバ: https://github.com/sekigon-gonnoc/zmk-driver-paw3222
- threshold temp layer: https://github.com/gohanda11/zmk-input-processor-threshold-temp-layer
