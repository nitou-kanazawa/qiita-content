---
title: 【Unity】Shader Graphサンプルを利用して動く2D背景を作る
tags:
  - C#
  - Unity
private: false
updated_at: '2024-06-29T19:54:16+09:00'
id: 2b3a1f616fe4ccb5ad7e
organization_url_name: null
slide: false
ignorePublish: false
---
## 初めに
この記事はUnity初学者が学習の備忘録として書いたものです．誤っている点もあるかと思いますので，その際はご指摘いただけると幸いです．


## 完成イメージ
下図のように2Dゲームの背景をそれっぽく動かすことを目指しました．

環境：Unity6 Preview (6000.0.4f1)

![メディア1.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/24e0cfbd-07f7-7b54-1578-17c01e6e0cd4.gif)

#### 概要
全体の流れとしては以下の通りです．
1. ShaderGraphサンプルを利用してシェーダー作成
2. マテリアルをSpriteRenderに適用
3. プロパティ操作用コンポーネントを作成
4. アセット"DOTween"でアニメーションさせる


## Shader Graphの公式サンプル
ShaderGraphには、 Procedural Patternsというサンプルデータが提供されています．
詳しくは以下の記事を参照してください．

https://zenn.dev/r_ngtm/books/shadergraph-cookbook/viewer/tips-samples

今回はサンプルの"Stripes"ノードを使用して，各種パラメータを設定できるUnlitシェーダーを作成しました．"Stripes"ノードは入力として "Offset" を受け付けているため，ここに "Time"ノードの出力を入れることで時間経過によるスクロールが可能です．

※"Time"ノードとUVオフセットを使用すれば容易に動きのあるシェーダーを作成できます．


![画像1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/6ddf44ba-9dbb-a86a-dc77-a11d9af7408c.png)

下表のプロパティを作成しました．プロパティはGraph Editorのブラックボードで作成できます．
※このあとC#スクリプトからアクセスする際には，下表の [Reference名] でプロパティにアクセスします．

|Name|Reference|型|役割|
|:---:|:---:|:--:|:--:|
|Frequency|_Frequency|Float|周期(線の本数)|
|Thickness|_Thickness|Float|線の間隔|
|ScrollX|_ScrollX|Float|X方向のオフセット速度|
|ScrollY|_ScrollY|Float|Y方向のオフセット速度|
|Rotation|_Rotation|Float|線の傾き|
|Color1|_Color1|Color|色|
|Color2|_Color2|Color|色|


## 背景オブジェクトの準備
背景はSprite Rendererで作成することにします．
Create GameObject Menuの "2D Object > Sprites > Square" でオブジェクトを作成して，Material項目に作成したマテリアルを適用します．

![タイトルなし.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/75f698ed-c542-292a-5e06-1320ff73df1f.png)


## マテリアルの操作に関して

##### プロパティの設定

マテリアルのプロパティは "SetFloat", "SetVector" 等のセッターで設定できます．引数に [プロパティ名]，もしくは[プロパティID] を渡すことで，任意のプロパティを指定できます．

``` Test.cs
Material _material = GetComponent<Renderer>().material;

// 名前指定で_Colorプロパティを設定
_material.SetColor("_Color", Color.white);

// ID指定で_Colorプロパティを設定
int colorPropID = Shader.PropertyToID("_Color");
_material.SetColor(colorPropID, Color.white);
```

https://kan-kikuchi.hatenablog.com/entry/Material

<br>

↓ Shaderがどのようなプロパティを持っているかはインスペクタの"Properities"項目で確認することができます．

![qiita_fig01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/879d9c88-9fba-e81e-7626-2b2ac43dea74.png)

##### マテリアルの破棄

またマテリアルのプロパティにアクセスすると，自動でコピーが生成されます（プロパティの変更はコピーに対して行われる）．そのためコピーされたオブジェクトは自分で破棄しないとメモリリークを引き起こすようです．

``` TestMonoBehaviour.cs
Material _material;

private void Awake() {   
    _material = GetComponent<Renderer>().material;
    _material.color = Color;
}

private void OnDestroy() {
    if (_material != null) {
        Destroy(_material);     // 自分で破棄する
        _material = null;
    }
}
```
Resources.UnloadUnusedAssets() でも使われてないマテリアルを削除することができます．

https://www.create-forever.games/unity-material-memory-leak/

https://light11.hatenadiary.com/entry/2019/11/03/223241


## 適当な操作コンポーネントを作成
必須ではありませんが，materal.GetVelue() / materal.SetVelue()の操作を簡略化するため下コードのようにアクセス用のプロパティを用意しておきます．

``` test.cs
    [RequireComponent(typeof(SpriteRenderer))]
    public class Test : MonoBehaviour {

        private Material _material;

        // ※簡易操作用のProperity
        public float Frequency {
            get => _material.GetFloat("_Frequency");
            set => _material.SetFloat("_Frequency", value);
        }

        private void Awake() {
            _material = gameObject.GetComponent<SpriteRenderer>().material;
        }

        private void OnDestroy() {
            Destroy(_material);
        }
    }
``` 

さほどメリットを感じませんが一応マテリアルを意識せずにパラメータ設定が行えます．
``` test.cs
    [RequireComponent(typeof(SpriteRenderer))]
    public class Test : MonoBehaviour {

        // インスペクタでここを操作すると，変動することを確認できる
        [Range(0, 30)] public float frequency;

        private void Update() {
            Frequency = frequency;
        }
    }
``` 

<br>
またアセット"DOTween"には任意のプロパティをアニメーションさせる DOTween.To() というメソッドがあります．こちらを用いると簡単にアニメーション処理を実装できます．

``` test.cs
[RequireComponent(typeof(SpriteRenderer))]
    public class Test : MonoBehaviour {

        // "Frequency"を指定した値までアニメーションさせる関数
        public Tweener DOFloat_Frequency(float endValue, float duration) {
            return DOTween.To(
                () => Frequency,
                x => Frequency = x,
                endValue,
                duration
                ).SetLink(gameObject);
        }

        IEnumerator Start() {
            while (true) {
                if (Input.GetKeyDown(KeyCode.Return)) {
                    // アニメーション開始
                    DOFloat_Frequency(endValue: 100, duration: 1);
                    break;
                }
                yield return null;
            }
        }
    }
``` 
DOTweenの文法については以下の記事を参照ください．

https://zenn.dev/ohbashunsuke/books/20200924-dotween-complete/viewer/dotween-05


## 最終的なコード


<details><summary> Material Handler 　(マテリアル操作を簡略化するためのラッパー)</summary>
※マテリアルは動的に生成する形に変更しています．

``` MaterialHandler.cs
using System;
using UnityEngine;

namespace nitou {

    /// <summary>
    /// マテリアルのプロパティ操作用ラッパークラス
    /// </summary>
    public abstract class MaterialHandler : IDisposable {

        protected readonly Shader _shader = null;
        protected Material _material = null;

        /// ----------------------------------------------------------------------------
        // Public Method 

        public MaterialHandler(Shader shader) {
            if (shader == null) throw new ArgumentNullException(nameof(shader));

            // マテリアル生成
            _shader = shader;
            _material = new Material(_shader);
            DefinePropertyID();
        }

        public void Dispose() {
            if (_material == null) return;
            GameObject.Destroy(_material);
            _material = null;
        }

        /// <summary>
        /// レンダラーにマテリアルを適用する
        /// </summary>
        public void ApplayMaterial(Renderer renderer) {
            if (renderer == null) throw new ArgumentNullException(nameof(renderer));
            renderer.sharedMaterial = _material;
        }


        /// ----------------------------------------------------------------------------
        // Protected Method 

        /// <summary>
        /// マテリアルのプロパティID定義
        /// </summary>
        protected abstract void DefinePropertyID();
    }


    public static partial class RendererExtensions {

        /// <summary>
        /// レンダラーにマテリアルを適用する拡張メソッド
        /// </summary>
        public static void SetSharedMaterial(this Renderer self, MaterialHandler handler) {
            handler.ApplayMaterial(self);
        }
    }
}
```

↓今回作ったシェーダー用のクラス．
``` StripeMaterial.cs
using UnityEngine;

namespace nitou {

    public class StripeMaterial : MaterialHandler {

        // ID
        protected int _frequencyID;
        protected int _thicknessID;
        protected int _rotationID;
        protected int _scrollXID;
        protected int _scrollYID;
        protected int _color1Id;
        protected int _color2Id;

        /// ----------------------------------------------------------------------------
        // Properity

        public float Frequency {
            get => _material.GetFloat(_frequencyID);
            set => _material.SetFloat(_frequencyID, value);
        }

        public float Thickness {
            get => _material.GetFloat(_thicknessID);
            set => _material.SetFloat(_thicknessID, value);
        }

        public float Rotation {
            get => _material.GetFloat(_rotationID);
            set => _material.SetFloat(_rotationID, value);
        }

        public float ScrollX {
            get => _material.GetFloat(_scrollXID);
            set => _material.SetFloat(_scrollXID, value);
        }

        public float ScrollY {
            get => _material.GetFloat(_scrollYID);
            set => _material.SetFloat(_scrollYID, value);
        }

        public Color Color1 {
            get => _material.GetColor(_color1Id);
            set => _material.SetColor(_color1Id, value);
        }

        public Color Color2 {
            get => _material.GetColor(_color2Id);
            set => _material.SetColor(_color2Id, value);
        }


        /// ----------------------------------------------------------------------------
        // Public Method 

        public StripeMaterial(Shader shader) : base(shader) {}


        /// ----------------------------------------------------------------------------
        // Protected Method 

        /// <summary>
        /// マテリアルのプロパティID定義
        /// </summary>
        protected override void DefinePropertyID() {
            _frequencyID = Shader.PropertyToID("_Frequency");
            _thicknessID = Shader.PropertyToID("_Thickness");
            _rotationID = Shader.PropertyToID("_Rotation");
            _scrollXID = Shader.PropertyToID("_ScrollX");
            _scrollYID = Shader.PropertyToID("_ScrollY");
            _color1Id = Shader.PropertyToID("_Color1");
            _color2Id = Shader.PropertyToID("_Color2");
        }
    }
}

```
</details>

*** 

<details><summary>Material Controller 　(Rendererを持つGameObjectにアタッチするコンポーネント)</summary>

``` MaterialController.cs
using UnityEngine;

namespace nitou {

    /// <summary>
    /// マテリアルの操作を行うコンポーネント
    /// </summary>
    public abstract class MaterialController<T> : MonoBehaviour
        where T : MaterialHandler {

        [SerializeField] Renderer _renderer = null;
        [SerializeField] Shader _shader = null;

        protected T _handler = null;

        /// ----------------------------------------------------------------------------
        // MonoBehaviour Method 

        private void Awake() {
            _handler = CreateHandler(_shader);
            _renderer.SetSharedMaterial(_handler);
        }

        private void OnDestroy() {
            _handler?.Dispose();
        }

        /// ----------------------------------------------------------------------------
        // Protected Method 

        /// <summary>
        /// シェーダーからマテリアルとハンドラを生成する
        /// </summary>
        protected abstract T CreateHandler(Shader shader);



#if UNITY_EDITOR
        private void OnValidate() {
            if (_renderer == null) _renderer = gameObject.GetComponent<Renderer>();
        }
#endif
    }

}
```
今回作ったシェーダーのマテリアルを操作するコンポーネント
``` StripeMaterialController.cs
using UnityEngine;
using DG.Tweening;

namespace nitou {

    public class StripeMaterialController : MaterialController<StripeMaterial> {

        /// ----------------------------------------------------------------------------
        // Protected Method 

        protected override StripeMaterial CreateHandler(Shader shader) {
            return new StripeMaterial(shader);
        }

        /// ----------------------------------------------------------------------------
        // Public Method 

        public Tweener Tween_Frequency(float endValue, float duration) {
            return DOTween.To(
                () => _handler.Frequency,
                x => _handler.Frequency = x,
                endValue,
                duration
                ).SetLink(gameObject);
        }

        public Tweener Tween_Thickness(float endValue, float duration) {
            return DOTween.To(
                () => _handler.Thickness,
                x => _handler.Thickness = x,
                endValue,
                duration
                ).SetLink(gameObject);
        }

        public Tweener Tween_Rotation(float endValue, float duration) {
            return DOTween.To(
                () => _handler.Rotation,
                x => _handler.Rotation = x,
                endValue,
                duration
                ).SetLink(gameObject);
        }

        public Tweener Tween_ScrollX(float endValue, float duration) {
            return DOTween.To(
                () => _handler.ScrollX,
                x => _handler.ScrollX = x,
                endValue,
                duration
                ).SetLink(gameObject);
        }

        public Tweener Tween_ScrollY(float endValue, float duration) {
            return DOTween.To(
                () => _handler.ScrollY,
                x => _handler.ScrollY = x,
                endValue,
                duration
                ).SetLink(gameObject);
        }
    }

}
```
</details>


***
以下のクラスで簡単な動作確認をした結果です．（今回作ったシェーダーではアスペクト比の変更に対応していないので，変形させると斜線が崩れます．）

↓ 実行結果
![メディア2.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/b588c3c0-ba18-687f-84f4-132e0a3f1e24.gif)

``` testMain.cs
    // 動作確認用
    public class testMain : MonoBehaviour {
        [SerializeField] StripeMaterialController _controller;

        private Tween _tween;

        void Update() {
            if (Input.GetKeyDown(KeyCode.J)) {
                _tween?.Kill();
                _tween = TweenA(1);

            } else if (Input.GetKeyDown(KeyCode.K)) {
                _tween?.Kill();
                _tween = TweenB(1);
            }
        }

        public Tween TweenA(float duration) {
            return DOTween.Sequence()
                .Join(_controller.Tween_Frequency(20f, duration))
                .Join(_controller.Tween_Thickness(-0.1f, duration))
                .Join(_controller.Tween_Rotation(-45f, duration))

                // スクロール速度・方向
                .Join(_controller.Tween_ScrollX(0, 0.2f))
                .Join(_controller.Tween_ScrollY(0, 0.2f));
        }

        public Tween TweenB(float duration) {
            return DOTween.Sequence()
                .Join(_controller.Tween_Frequency(10f, duration))
                .Join(_controller.Tween_Thickness(0.5f, duration))
                .Join(_controller.Tween_Rotation(45f, duration))

                // スクロール速度・方向
                .Join(_controller.Tween_ScrollX(0.5f, 0.2f))
                .Join(_controller.Tween_ScrollY(0.5f, 0.2f));
        }
``` 

## 終わりに
 Procedural Patternsには様々なサンプルデータが含まれるため，色々なパターンを試せそうです．
 
![d2j4qpvlmkwelo7n50j8xmn4wobg.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/e3efad62-4c65-3cb9-d564-2405fda8268c.png)

