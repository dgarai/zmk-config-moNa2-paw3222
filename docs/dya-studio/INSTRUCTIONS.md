# 作業指示書: moNa2 (PAW3222 版) の DYA Studio レベル2 対応

対象リポジトリ: `dgarai/zmk-config-moNa2-paw3222`（ローカル: `/Volumes/SANZANG/Develop/zmk-for-moNa2-paw3222`）

**着手前に [HANDOFF.md](HANDOFF.md) を全部読むこと。** 背景・調査結果・落とし穴はそちらにある。

---

## 完了条件

- [ ] 作業ブランチ `feat/dya-studio-level2` で、GitHub Actions の 3 ファーム（右・左・settings_reset）のビルドがすべて成功している
- [ ] `main` 向けの PR が作られている（**マージはしない**。ユーザーが実機確認してからマージする）
- [ ] トラックボールの向き・スクロールの向きと量・スナップ・オートマウスが、旧構成と等価になるように設定されている（根拠を PR に書く）
- [ ] `config/mona2.keymap` を変更していない
- [ ] ユーザーへの報告に、実機確認の手順（`settings_reset` → 新ファーム → 確認項目 → DYA Studio 接続）が含まれている
- [ ] `wsdev release` で引き継ぎメモを残している

## やってはいけないこと

- ローカルで west / Docker ビルドをしない（ディスクが満杯になる。HANDOFF §4-1）
- `config/mona2.keymap` を編集しない（v2 から自動同期で上書きされる。HANDOFF §4-4）
- `main` に直接 push・マージしない
- PAW3222 用に独自 RPC ドライバを書かない（レベル3。今回のスコープ外）
- `CONFIG_ZMK_KSCAN_DEBOUNCE_PRESS_MS=0` など、DYA Studio と無関係な既存設定を変えない（気づいた点は報告だけ）

---

## 手順

### 0. 準備

```sh
cd /Volumes/SANZANG/Develop
wsdev who
wsdev claim zmk-for-moNa2-paw3222 "DYA Studio レベル2 対応"
cd zmk-for-moNa2-paw3222
git status            # 未コミットの変更がないこと（docs/dya-studio/ は未追跡でよい）
git pull --ff-only    # ローカルは origin より 40 コミット以上遅れている
git switch -c feat/dya-studio-level2
```

先行事例の差分を読む:

```sh
git -C /Volumes/SANZANG/Develop/zmk-config-moNa2-v2 show cceb722
```

### 1. pull 後の再確認

HANDOFF §3 は 2026-11-18 時点の `origin/main` で調べた内容。pull 後にもう一度確認する:

- `config/west.yml` のモジュール一覧
- `boards/shields/mona2/mona2_r.overlay` の listener / `scroller` / `zip_threshold_temp_layer` の値
- `git grep -n -i -E "prospector|keybind|status_adv"` で未使用のものを確認する

HANDOFF と違っていたら、実際のファイルを正として進め、違いを報告に書く。

### 2. ユーザーに確認する（実装前）

HANDOFF §3.3 のオートマウスについて、AskUserQuestion で確認する:

- **A（推奨）**: gohanda11 の「100 カウント動いてからオートマウス」を残す。DYA Studio からオートマウスは調整できない
- **B**: DYA Studio のオートマウス（移動量しきい値なし）に一本化する

### 3. 実装

v2 の `cceb722` を雛形に、以下を変更する。

#### 3.1 `config/west.yml`

v2 の `config/west.yml` をベースにし、次の点だけ変える:

- **追加**: `zmk-driver-paw3222`（remote: `sekigon-gonnoc`、revision: `main`）
- **追加（A のときだけ）**: `zmk-input-processor-threshold-temp-layer`（remote: `gohanda11`、revision: `main`）
- **追加しない**: `zmk-driver-pmw3610-with-custom-studio-rpc`（PMW3610 用。PAW3222 には使えない）
- **削除**: `prospector-zmk-module`、`zmk-input-processor-keybind`、`zmk-scroll-snap`（手順 1 で未使用を確認したもの）
- 不要になった remote（`prospector`、`zettaface`、`kot149`、`sayu-hub` など）も消す

#### 3.2 `build.yaml` / `.github/workflows/build.yml` / `mona2.zmk.yml`

v2 と同じ:

- `build.yaml`: `seeeduino_xiao_ble` → `xiao_ble//zmk`
- `.github/workflows/build.yml`: `@v0.3.0` → `@main`
- `mona2.zmk.yml`: `requires: [seeed_xiao]`

#### 3.3 `boards/shields/mona2/Kconfig.defconfig` / `mona2.dtsi`

- `config NFCT_PINS_AS_GPIOS` のブロックを削除する
- `mona2.dtsi` の末尾に追加する:
  ```dts
  &uicr {
      nfct-pins-as-gpios;
  };
  ```
- `mona2.dtsi` の listener から `input-processors` を削除する（overlay で設定するため。もともと overlay の値で上書きされて効いていない）

#### 3.4 `boards/shields/mona2/mona2_r.overlay`

- センサーノード（`compatible = "pixart,paw3222"`、`res-cpi = <950>`）は **そのまま**
- `#include <scroll-snap.dtsi>` を削除し、次を追加する:
  ```dts
  #include <input/processors/runtime-input-processor.dtsi>
  #include <dt-bindings/zmk/runtime_input_processor.h>
  ```
- `&zip_scroll_snap { ... }` のブロックを削除する
- listener を次のように書き換える。**どちらも旧構成と等価になるよう計算済み**（HANDOFF §3.2）

**A（gohanda11 のしきい値を残す）の場合**: `zip_threshold_temp_layer` の定義ノードは残す。SCROLL レイヤーでは override でスクロール処理だけを通す（旧構成と同じく、オートマウスは通らない）。

```dts
&trackball_central_listener {
    status = "okay";
    device = <&trackball_central>;
    // ポインター: 100 カウント動いたら MOUSE(4) へ、500ms で解除 → 感度などは DYA Studio で調整
    input-processors = <&zip_threshold_temp_layer 4 500>, <&mouse_runtime_input_processor>;

    scroller {
        layers = <5>;   // 5 = SCROLL
        input-processors = <&scroll_runtime_input_processor>;
    };
};

&scroll_runtime_input_processor {
    x-codes = <INPUT_REL_X>;
    y-codes = <INPUT_REL_Y>;
    xy-to-scroll-enabled;
    y-invert;                                   // 旧 zip_scroll_transform Y_INVERT
    active-layers = <(1 << 5)>;
    scale-multiplier = <1>;
    scale-divisor = <10>;                       // 旧 zip_scroll_scaler 1 10
    axis-snap-mode = <AXIS_SNAP_MODE_Y>;        // 旧 zip_scroll_snap の代わり
    axis-snap-threshold = <600>;
};
```

**B（runtime processor に一本化）の場合**: `zip_threshold_temp_layer` の定義ノードと `scroller` を削除する。v2 と同じ形にする。

```dts
&trackball_central_listener {
    status = "okay";
    device = <&trackball_central>;
    input-processors = <&scroll_runtime_input_processor &mouse_runtime_input_processor>;
};

&mouse_runtime_input_processor {
    temp-layer-enabled;
    temp-layer = <4>;
    temp-layer-deactivation-delay-ms = <500>;
};

&scroll_runtime_input_processor { /* A と同じ内容 */ };
```

A・B 共通で、DYA Studio の表示用ノードを追加する（v2 と同じ）:

```dts
/ {
    trackball_layout: trackball_layout {
        compatible = "cormoran,physical-layout-trackball";
        display-name = "Trackball";
        size = <180>;
        x = <960>;
        y = <300>;
        linked-device-identifiers = "mouse", "scroll";
        linked-subsystems = "cormoran_rip", "cormoran_rip";
    };
};
```

#### 3.5 `config/mona2_r.conf` / `config/mona2_l.conf`

- v2 の `config/mona2_r.conf` 末尾の「DYA Studio レベル2」ブロックを右手側に追加する。ただし次の 2 行は **入れない**（PMW3610 専用）:
  ```
  CONFIG_ZMK_PMW3610_STUDIO_RPC=y
  CONFIG_ZMK_PMW3610_CUSTOM_SETTINGS=y
  ```
- v2 の `config/mona2_l.conf` 末尾のブロックを、そのまま左手側に追加する
- 既存の `CONFIG_PAW3222_*`（コメントアウト）や debounce などは触らない

#### 3.6 README

v2 の README の「DYA Studio（レベル2）」節を参考に、PAW3222 版向けに書く。次の点を明記する:

- Trackball タブのセンサー設定（CPI など）は未対応。CPI は `res-cpi` で決める
- A を選んだ場合、オートマウスは DYA Studio ではなく overlay の `activation-threshold` で決める

### 4. コミット・push・CI

```sh
git add -A    # docs/dya-studio/ も一緒にコミットしてよい
git commit    # メッセージは v2 の cceb722 に倣う。末尾にセッションの attribution 行を付ける
git push -u git@github.com:dgarai/zmk-config-moNa2-paw3222.git feat/dya-studio-level2
git branch -u origin/feat/dya-studio-level2   # SSH URL で push すると追跡先が変になるため直す
gh run list -R dgarai/zmk-config-moNa2-paw3222 -b feat/dya-studio-level2 -L 1
```

- CI が完了するまで待つ（`gh run view <id> --json status,conclusion` を background の until ループで待つ）
- 失敗したら `gh api repos/dgarai/zmk-config-moNa2-paw3222/actions/jobs/<job_id>/logs` でログを取り、直して push し直す
- **A でビルドが通らず、原因が gohanda11 モジュールの Zephyr 4.1 非対応だった場合**: 自分で fork して直さない。ユーザーに報告して B に切り替えるか確認する
- 成功したら FLASH / RAM の使用率をログから拾う（`Memory region` の行）

### 5. PR 作成

```sh
gh pr create -R dgarai/zmk-config-moNa2-paw3222 --base main --head feat/dya-studio-level2 ...
```

PR 本文に書くこと（v2 の PR #2 を参考に）:

- 変更点
- 旧構成との等価性の根拠（反転・スケール・スナップ・オートマウス）
- 選んだオートマウス方式（A / B）
- 既知の差分（スナップ方式の変化。B ならしきい値がなくなること）
- 実機確認のチェックリスト

### 6. 終了処理

```sh
wsdev release zmk-for-moNa2-paw3222 --note "<未検証事項 / 選んだ案と却下した案と理由>"
```

Agent Memory Vault の `10_Sessions/2026/<日付>/` にセッション要約を書く。

### 7. ユーザーへの報告

次を含める:

- PR の URL と CI の結果（FLASH / RAM）
- 実機確認の手順
  1. PR の Actions から firmware の zip をダウンロードする
  2. 左右とも `settings_reset` を先に書き込む（BLE ペアリングが消える）
  3. 左右に新しいファームを書き込む
  4. キー・エンコーダ・左右接続・トラックボールの向き・オートマウス・J+K+L での SCROLL（向き・量・スナップ）を確認する
  5. 右手側を USB でつなぎ、Chrome で https://studio.dya.cormoran.works を開いて「USB」で接続し、各タブを確認する。Trackball タブのセンサー設定は「未対応」表示で正常
- 戻し方: `main` のファームを書き込めば元に戻る
