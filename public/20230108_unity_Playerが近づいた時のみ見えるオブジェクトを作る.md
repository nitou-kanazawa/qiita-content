---
title: 【Unity】 Playerが近づいた時のみ見えるオブジェクトを作る
tags:
  - C#
  - Unity
  - 初心者
private: false
updated_at: '2025-01-11T13:34:18+09:00'
id: 8433c865e2be6c748f5e
organization_url_name: null
slide: false
ignorePublish: false
---

## 初めに

この記事は Unity 初学者が学習の備忘録として書いたものです．誤っている点もあるかと思いますので，ご指摘いただけると幸いです．

(Unity2020.3 を使用)

## 完成イメージ

Unity 公式チュートリアル"**Roll-a-ball**"に追加機能として実装しました．
https://learn.unity.com/project/roll-a-ball

![ezgif.com-gif-maker.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/5575feb7-c91f-e0b0-7aa8-ef06c0eb8b0d.gif)

## 利用した機能

プレーヤーの接近検知には**OnTrigger**・**OnCollision**メソッドを利用しました．

https://www.sejuku.net/blog/83742

### 接触イベント

上記の記事で説明されているように，**Collider コンポーネント**が取り付けられた GameObject では他の Collider との接触時に実行したい処理を，**OnTrigger・OnCollision**メソッドで指定しておくことができます．
※少なくともどちらかの GameObject に**Rigidbody コンポーネント**も必要です．

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/15069441-a766-6821-84a9-ab57d88dff6f.png)

### OnTrigger・OnCollision の違い

これらは接触イベントの対象となる Collider の Trigger 状態(※すり抜ける否か)によって使い分けます．

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/1e5d4441-d06b-26d1-ba8b-e715d43c7d73.png)

今回はプレイヤー側に Trigger あり(すり抜ける)の**Spher Cllider**を取り付け，これを探索範囲として Trigger なし(すり抜けない)の壁への近接判定を行います．よって上図の対応から**OnTrigger**に処理を記述します．

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/fbd3c125-2a71-50e4-86cc-0ea296c5857a.png)

### 3 種類の OnTrigger

また，OnTrigger には呼び出されるタイミングが異なる 3 種類が用意されています．これによって flag や state などのあるタイミングで状態を切り変える処理や，接触が起きている間に繰り返し呼び出したい処理をそれぞれ設定することができます．

| メソッド         | 機能                                                 |
| ---------------- | ---------------------------------------------------- |
| OnTriggerEnter() | Trigger オブジェクトに侵入した瞬間に呼び出される     |
| OnTriggerStay()  | Trigger オブジェクトに侵入している間呼び出され続ける |
| OnTriggerExit()  | Trigger オブジェクトから脱出した瞬間に呼び出される   |

# 実装したコンポーネント(スクリプト)

今回は「プレイヤーが一定の距離まで近づいたら，距離に応じた透明度を設定する」という実装にしたいため OnTriggerStay()を使用しました．※透明度は Mesh コンポーネントの material.color の alpha 値で設定

また，接触相手の GameObject が透明な壁かどうかは透明度の制御スクリプト（InvisibleObject コンポーネント）がアタッチされているかで判定しています．

```PlayerSearchArea.cs
[RequireComponent(typeof(SphereCollider))]
public class PlayerSearchArea : MonoBehaviour {

    private SphereCollider area;   // 探索エリアとして扱うClollider
    private float areaRadius = 5;  // SphereColliderの半径

    void Start() {
        area = GetComponent<SphereCollider>();
        area.radius = areaRadius;
    }

    // ↓ 他のColliderが探索エリア内にいる限り呼ばれ続ける
    private void OnTriggerStay(Collider other) {
        // 相手が不可視オブジェクトの場合，
        if (other.gameObject.TryGetComponent<InvisibleObject>(out var obj)) {

            // 距離に応じてalpha値を設定
            var distance = (transform.position - other.transform.position).magnitude;
            var aipha = Mathf.Clamp01(1 - distance / areaRadius);
            obj.SetVisibility(true, aipha);     // InvisibleObjectクラスの透明度設定メソッド
        }
    }

```

こちらが相手側にアタッチしておくスクリプトです．Mesh マテリアルの透明度を設定するメソッドを public で用意しておき，初期化時に見えないようにしておきます．
※今回の機能には不要なプロパティ(IsVisible)が含まれています．

```InvisibleObject.cs
[RequireComponent(typeof(MeshRenderer))]
public class InvisibleObject : MonoBehaviour{

    // 制御対象
    private MeshRenderer mesh;

    // 透明かどうか
    public bool IsVisible { get; private set; } = false;

    //
    void Start() {
        mesh = GetComponent<MeshRenderer>();
        SetVisibility(false);           // 可視性に対応したalpha値を適用
    }

    // 可視性を設定し，見える場合はalpah値を指定可能
    public void SetVisibility(bool visibility, float alpha=1f) {
        IsVisible = visibility;
        SetMeshAlpha(IsVisible ? alpha : 0f);
    }
}
```
