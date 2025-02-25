---
title: 【Unity】ProjectSettingsの iOS (Player設定) に関するメモ
tags:
  - iOS
  - Unity
  - ProjectSettings
private: false
updated_at: '2025-01-11T13:18:50+09:00'
id: 4f88ba65af32b330f207
organization_url_name: null
slide: false
ignorePublish: false
---

## 概要

iOS 向けの設定でいろいろとハマることがあった（主に画面端の入力関係）ため、備忘録として残します．

環境：Unity2021.2.1

https://docs.unity3d.com/ja/2023.2/Manual/class-PlayerSettingsiOS.html

設定項目は以下のセクションに分かれている．

- Icon (アイコン)
- Resolution and Presentation (解像度と表示)
- Splash Image (スプラッシュ画像)
- Debugging and crash reporting (デバッグとクラッシュのレポート)
- Other Settings (その他の設定)

## Resolution and Presentation

#### Orientation

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/8eb58fff-d5b1-0fb1-cba6-b7403327e0bf.png)

画面の回転方向に関する設定．※この項目は`Android`, `iOS`, `UWP`で共有．

#### Allowed Orientations for Auto Rotation

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/6c1a0c98-2228-bba5-bde7-52599a042964.png)

`Default Orientaion`が`Auto Rotaion`の時に表示される．各方向への自動回転を許可するか否かを設定できる．

また`Auto Rotation`の場合、`Orientation`の方に自動回転時のアニメーションを有効にするかの項目も追加される．

#### Multitasking Support

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/7c7e0b13-7ba2-01aa-e0dc-49365c3ed01e.png)

マルチタスク機能を使用するかの設定．使用しない場合は`RequiresFullScreen`にチェックを入れる．有効にすると、`StatusBar`中央に表示される`・・・`から画面分割を行える．

https://support.apple.com/ja-jp/102576

また有効にする場合、横画面と縦画面の両方に対応している必要があるらしい．
(私の環境では XCode のビルドは通るが、`・・・`が表示されず無効な状態になっていた)

https://kan-kikuchi.hatenablog.com/entry/Requires_full_screen#google_vignette

#### Status Bar

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/37a6de4f-5650-a2aa-0918-8bba976fa660.png)

iOS アプリのステータスバー（画面上部に表示される時間、バッテリー状態、先ほどの`・・・`などを含む領域）に関連する設定．

- Status Bar Hidden：　表示 / 非表示
- Status Bar Style：　表示色（Auto / 黒 / 白）

## Other Settings

#### Configuration

- Camera Usage Description
- Microphone Usage Description
- Location Usage Desctiption

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/ab00a505-7837-594b-e9bb-374bcab94859.png)
iOS アプリでカメラやマイク、位置情報を使用する場合は、何らかの文字列を入力する必要がある．ここに設定した文字列は許可ダイアログ内の説明文で使用されるらしい．

https://qiita.com/ohbashunsuke/items/c56da7461ff6f3bd6910

<details><summary>説明文の多言語化</summary>
余談だが、UnityLocalizationで多言語化対応している場合、これらの説明文は`ProjectSettings > Localization > Metadata`から`LocalizedString`を設定できる．

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/2d641424-f28b-1460-946c-d2ed6c88008d.png)

</details>

- Target Device：　アプリが対象とするデバイス
- Target minimum iOS Support：　アプリが動作する iOS の最小バージョン

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/a852be2f-91e7-879c-e880-e2d3b8a43d5e.png)

- Defer system gester on edges：　画面端でのシステムジェスチャの遅延
  ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/5c51a38b-b92d-c834-1d23-22b878dea855.png)

画面端からのスワイプでシステムメニューが開くが、それに必要なスワイプが２回になる（１回目は選択状態になり、２回目で通常）．
<b>画面右端にある`Slider` (uGUI) の反応が遅れるという問題が発生していた</b>が、この`Right Edge`にチェックを入れることで解決した．Slider は IDragHandler で値の更新を行っているため、それとシステムジェスチャの判定が競合していたものと思われる．

一応マニュアルでは下記のように説明されており、実際に上下のジェスチャは使用頻度が高いため有効化する場合は注意が必要．

> iOS ヒューマン インターフェイスのガイドラインでは、この動作を有効にするとユーザーが混乱する可能性があるため、有効にすることはお勧めしません。

https://docs.unity3d.com/6000.0/Documentation/ScriptReference/PlayerSettings.iOS-deferSystemGesturesMode.html
