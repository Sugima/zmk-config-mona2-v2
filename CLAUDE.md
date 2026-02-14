# mona2-v2 ZMK Config

mona2 キーボード（右手トラックボール付き分割キーボード）の ZMK ファームウェア設定リポジトリ。

## ビルド

GitHub Actions で自動ビルド。`build.yaml` に定義された3ターゲットがビルドされる:
- `mona2_r` (右手・トラックボール側)
- `mona2_l` (左手)
- `settings_reset`

ボードは `seeeduino_xiao_ble`。

## ファイル構成

```
config/
  mona2.keymap       # キーマップ本体（レイヤー、ジェスチャー、コンボ、マクロ）
  mona2_r.conf       # 右手側 Kconfig（エンコーダ、バッテリー、LED、Studio）
  mona2_l.conf       # 左手側 Kconfig
  mona2.json         # ZMK Studio / keymap editor 用レイアウト定義
  west.yml           # West マニフェスト（ZMK本体 + 外部モジュール）
boards/shields/mona2/
  mona2.dtsi         # シールド共通定義（トラックボールリスナー、エンコーダ、マトリクス）
  mona2_r.overlay    # 右手オーバーレイ（SPI、トラックボールドライバ、スクロール設定）
  mona2_l.overlay    # 左手オーバーレイ
```

## 外部モジュール (west.yml)

| モジュール | リモート | 用途 |
|---|---|---|
| `zmk` v0.3.0 | zmkfirmware | ZMK 本体 |
| `zmk-pmw3610-driver` | badjeff | トラックボールセンサー (PMW3610) ドライバ |
| `zmk-rgbled-widget` | caksoylar | RGB LED ウィジェット |
| `zmk-input-processor-keybind` | zettaface | トラックボールジェスチャー → キー発火 |

## レイヤー構成

| # | 名前 | 用途 |
|---|---|---|
| 0 | default | QWERTY |
| 1 | NAVI | ナビゲーション、記号 |
| 2 | NUM | テンキー、記号 |
| 3 | FUNCTION | F1-F12 |
| 4 | MISC | メディア、Bluetooth、スクショ |
| 5 | MOUSE | マウスボタン（トラックボール移動で自動有効化） |
| 6 | SCROLL | トラックボールスクロールモード |
| 7 | GESTURE | トラックボールジェスチャーモード |

## トラックボール入力処理の仕組み

トラックボールの入力は以下の階層で処理される:

### 1. ハードウェアドライバ (`mona2_r.overlay`)
PMW3610 センサーで `invert-x` / `invert-y` が有効。センサーレベルで軸を反転。

### 2. Input Listener ベースプロセッサ (`mona2.dtsi`)
```
trackball_central_listener {
    input-processors =
        <&zip_xy_transform INPUT_TRANSFORM_Y_INVERT>,  // Y軸反転
        <&zip_temp_layer 5 500>;                        // 自動マウスレイヤー(500ms)
};
```
ハードウェアの `invert-y` と合わせて二重反転になり、意図した方向になる。

### 3. Child Node（レイヤー固有プロセッサ）

ZMK の input listener child node は**デフォルトで base processors を置き換える**。
`process-next` を指定すると base processors も実行される。
参考: https://zmk.dev/docs/keymaps/input-processors/usage

#### スクロール (`mona2_r.overlay`, layer 6)
```
scroller {
    layers = <6>;
    input-processors = <&zip_xy_transform INPUT_TRANSFORM_X_INVERT>,
        <&zip_xy_to_scroll_mapper>, <&zip_scroll_transform INPUT_TRANSFORM_X_INVERT>,
        <&zip_scroll_scaler 1 5>;
};
```

#### ジェスチャー (`config/mona2.keymap`, layer 7)
```
gesture {
    layers = <GESTURE>;
    process-next;
    input-processors = <&zip_keybind_gesture>;
};
```
`process-next` により base processors (Y_INVERT, zip_temp_layer) も実行される。
これがないと Y_INVERT がスキップされ、ジェスチャー中のカーソルY方向が反転する。

### ジェスチャープロセッサ (`zip_keybind_gesture`)

`zmk-input-processor-keybind` モジュール (https://github.com/zettaface/zmk-input-processor-keybind) を使用。
トラックボールの移動を4方向のキー入力に変換する。

```
zip_keybind_gesture {
    compatible = "zmk,input-processor-keybind";
    bindings = <右>, <左>, <下>, <上>;   // この順序で4方向バインド
    tick = <30>;          // 発火しきい値
    tap-ms = <10>;        // タップ時間
    mode = <1>;           // 4方向モード
    track_remainders;     // 端数を保持
};
```

現在のバインド:
- 右: `Cmd+]` (ブラウザ進む)
- 左: `Cmd+[` (ブラウザ戻る)
- 下: `Cmd+W` (タブ閉じる)
- 上: `Cmd+T` (新規タブ)

**注意:** このプロセッサは `ZMK_INPUT_PROC_STOP` を返すため、gesture child 内の後続プロセッサは実行されない。

## keymap editor との互換性

`config/mona2.json` がレイアウト定義。keymap editor でキー変更後、DTS コメント (`/* ... */`) の位置が変わりビルドエラーになることがある。コメントがバインディング行から分離されると DTS パースエラーになるため、ビルドエラー時はコメントの位置を確認すること。
