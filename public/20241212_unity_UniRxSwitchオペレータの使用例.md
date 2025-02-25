---
title: 【Unity】UniRx / Switchオペレータの使用例
tags:
  - Unity
  - UniRx
  - UniTask
private: false
updated_at: '2025-01-11T13:18:51+09:00'
id: 48bf9e523a0891e4430f
organization_url_name: null
slide: false
ignorePublish: false
---

## Switch について

参考文献では以下のように説明されています．

> Switch は購読の対象とする Observable を切り替えることができる Operator です．入力として複数の Observable を与えることができ、そのうちの最新の Observable についてのみ購読できるようになります。

> Switch は Observable が与えられたとき、すでに購読している Observable がある場合はそれをキャンセルし新しい Observable へと購読を切り替えます．そしてその Observable から発行されたメッセージをすべて後続へと素通しします．

> つまり Switch を用いることで、１つの Observable を維持したまま購読する対象を次へ次へと切り替えていくことができるようになります．

https://tatsu-zine.com/books/unirx-unitask

「マーブルダイアグラム」では以下のように表わされます．

<img width="400" alt="daiaglam.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/e67b95d0-47a6-3787-5f06-a66f3b5e5154.png">

出典: ReactiveX Documentation

https://reactivex.io/documentation/operators/switch.html

この `IObservable<IObservable<T>>`に対して<b>購読する対象を切り替えていく</b>ということに具体的なイメージを持てていなかったため、上記の文献で紹介されているものも含めて使用例をいくつか調べてみました．

## Switch の使用例

1. 選択中のキャラクターの攻撃力
2. 最後に接触したオブジェクトの追跡
3. 入力に応じたリソース読み込み (インクリメンタルサーチ)

## 1.選択中のキャラクターの攻撃力

`IObservable<IObservable<T>>` の一番分かりやすい例は入れ子になった `ReactiveProperty` かと思います．ここでは、攻撃力を持つ Actor と、選択中のアクターを管理する ActorSelectionController を考えます．

```.cs
public class Actor {
    public ReactiveProperty<int> AttackRP { get; } = new(0);
}

public class ActorSelectionController {
    public ReactiveProperty<Actor > CurrentActorRP { get; } = new(null);
}
```

以下のコードでは、現在選択されているアクターの AttackRP を監視し、値が更新された際にログを出力します。

```TestMono_01.cs
public class TestMono_01 : MonoBehaviour {

    private ActorSelectionController _actorSelectionController = new();

    private void Start() {
        _actorSelectionController.CurrentActorRP
            .Where(actor => actor != null)      // Null防止
            .Select(actor => actor.AttackRP)
            // 現在のActorのAttackRPだけを監視
            .Switch()
            .Subscribe(attack => Debug.Log($"Actor's Attack : {attack}"))
            .AddTo(this);

        // サンプルシナリオ
        SimulateActorSwitching();
    }

    private void SimulateActorSwitching() {
        // Actorの生成
        var actor1 = new Actor();
        var actor2 = new Actor();

        // Actor1を選択
        _actorSelectionController.CurrentActorRP.Value = actor1;

        // Actor1のAttackRPを変更
        actor1.AttackRP.Value = 1;
        actor1.AttackRP.Value = 2;
        actor1.AttackRP.Value = 3;

        // Actor2を選択
        _actorSelectionController.CurrentActorRP.Value = actor2;

        // Actor2のAttackRPを変更
        actor2.AttackRP.Value = 10;
        actor2.AttackRP.Value = 20;
        actor2.AttackRP.Value = 30;

        // 再びActor1を選択
        _actorSelectionController.CurrentActorRP.Value = actor1;

        // Actor1のAttackRPを変更
        actor1.AttackRP.Value = 5;
        actor1.AttackRP.Value = 6;
        actor1.AttackRP.Value = 7;
    }
}
```

【実行結果】

```
Actor's Attack : 0   ← 切り替え (nullからActor1)
Actor's Attack : 1
Actor's Attack : 2
Actor's Attack : 3
Actor's Attack : 0　← 切り替え (Actor1からActor2)
Actor's Attack : 10
Actor's Attack : 20
Actor's Attack : 30
Actor's Attack : 3　← 切り替え (Actor2からActor1)
Actor's Attack : 5
Actor's Attack : 6
Actor's Attack : 7
```

注意点として `AttackRP` への代入時に加えて `CurrentActorRP` の切り替え時にも発行されています．これは前述の「購読する対象を切り替えていく」タイミングで発火されたもののため，AttackRP の値に関係なく(※Actor1,2 の AttackRP が同じ値の場合でも)流れます．

---

<details> <summary>SelectManyの場合</summary>

ちなみに今回の例で `Select` + `Switch` の代わりに `SelectMany` を用いた場合，古い Observable がキャンセルされないため，Actor1 に再び切り替えた後は２回分の処理が実行されています．

```TestMono_01_kai.cs
_actorSelectionController.CurrentActorRP
            .Where(actor => actor != null)

            // 変更箇所
            .SelectMany(actor => actor.AttackRP)
            //.Select(actor => actor.AttackRP)
            //.Switch()

            .Subscribe(attack => Debug.Log($"Actor's Attack : {attack}"))
            .AddTo(this);
```

【実行結果】

```
Actor's Attack : 0   ← 切り替え (nullからActor1)
Actor's Attack : 1
Actor's Attack : 2
Actor's Attack : 3
Actor's Attack : 0　← 切り替え (Actor1からActor2)
Actor's Attack : 10
Actor's Attack : 20
Actor's Attack : 30
Actor's Attack : 3　← 切り替え (Actor2からActor1)
Actor's Attack : 5　←  ※以下は２回づつ処理が走っている
Actor's Attack : 5　←
Actor's Attack : 6  ←
Actor's Attack : 6　←
Actor's Attack : 7　←
Actor's Attack : 7　←
```

</details>

## 2. 最後に接触した対象の追跡

こちらは `Observable<T1>` の T1 から新しい `Observable<T2>` を作っていくパターンです．
以下では `IObservable<Collider>` に対して Collider から対象の座標を毎フレーム通知する `IObservable<Vector3>` を生成しています．これに <b>`Switch`オペレータ</b> を適用して最新のもののみ監視を行います．

```TestMono_02.cs
public class TestMono_02 : MonoBehaviour {

    private void Start() {
        var targetObservable =
            // IObservable<Collider>型
            this.OnCollisionEnterAsObservable()
            // IObservable<IObservable<Vector3>>型
            // （※ColliderからIObservable<Vector3>を作っている）
            .Select(collider => CreatePositionObservable(collider.gameObject));

        targetObservable
            // 最後に接触したColliderのみを監視
            .Switch()
            .Subscribe(targetPos => {
                    // 対象を追跡
                    var newPosition = Vector3.Lerp(transform.position, targetPos, Time.deltaTime);
                transform.position = newPosition;
            })
            .AddTo(this);
    }

    //
    private IObservable<Vector3> CreatePositionObservable(GameObject target) {
        return target.UpdateAsObservable()  // 毎フレーム監視
            .Select(_ => target.transform.position);
    }
}
```

## 3. 入力に応じたリソース読み込み (インクリメンタルサーチ)

先ほどはある型 `T` から `Observable` を直接作成していましたが，こちらは `T` を入力として非同期処理を実行してそれを `Observable` に変換するパターンです．

例では`InputField.OnValueChangedAsObservable` で入力値（リソースキー）の変化を検知して，非同期でスプライトを読み込みます．

```.cs
// スプライト読み込み
private async UniTask<Sprite> LoadSpriteAsync(string resourcePath, CancellationToken token = default) {
    var sprite = await Resources.LoadAsync<Sprite>(resourcePath) as Sprite;
    // 2秒程かかると仮定
    await UniTask.WaitForSeconds(2f, cancellationToken: token);
    return sprite;
}
```

### 3-1. UniTask.ToObservable()

UniTask には `Observable` へ変換するための拡張メソッドが用意されています．

```ObservableSwitchTest.cs
    public class ObservableSwitchTest : MonoBehaviour {
        [SerializeField] private TextMeshProUGUI _logText = null;
        [SerializeField] private TMP_InputField _inputField = null;
        [SerializeField] private Image _image = null;

        private void Start() {
            _inputField.OnValueChangedAsObservable()
                // 値が変化するたびに読み込み処理を非同期実行
                .Select(path => LoadSpriteAsync(path).ToObservable())
                // 最新のIObservableに切り替える
                .Switch()
                .Subscribe(sprite => _image.sprite = sprite)
                .AddTo(this);
        }
    }
```

【実行結果】
![SwitchTest_01](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/56ecf21e-05ea-9fa7-810a-53bdd0b67847.gif)

`Switch` オペレータによって `IObservable<IObservable<T>>` の最新のものに対してのみ処理が実行できています (古い `Observable` は Dispose されている)． しかし、実行した非同期処理 (UniTask) は裏で走り続けてしまっています．

`UniTask.ToObservable()` では Cancellation を渡せないため，そのままでは停止させることができません．
また Switch オペレータに限らず，UniRx で<b>非同期コールバックをどのように扱うか</b>は頻出する問題のようです．余談ですがこの辺の Rx と async/await の連携が R3 では改善されているようです．

https://developer.aiming-inc.com/csharp/unity-csharp-async-callback-patterns/
https://developer.aiming-inc.com/csharp/post-10773/

### 3-2. ObservableConverter.FromAsync()を実装して用いる

とりあえず，この問題へ対処するために以下の記事を参考にさせていただきました．記事ではパフォーマンスを考慮して「UniRx パッケージにメソッドを追加する方法」も示されていますが，少しハードルが高いため簡単な「static メソッドを実装する方法」の方を採用しました．

https://qiita.com/toRisouP/items/8ec18d73d9e8c5169587

以下は実装した `ObservableConverter.FromUniTask()` を用いた場合の実行結果です．

```ObservableSwitchTest_kai.cs
private void Start() {
            // ObservableConverter.FromUniTaskを使用する形に変更

            _inputField.OnValueChangedAsObservable()
                // 非同期読み込み処理を実行 (※CancellationDisposableを返す)
                .Select(path => ObservableConverter.FromUniTask(ct => LoadSpriteAsync(path, ct)))
                //.Select(path => LoadSpriteAsync(path).ToObservable())

                // 最新のIObservableに切り替える
                .Switch()
                .Subscribe(sprite => _image.sprite = sprite)
                .AddTo(this);
        }
```

【実行結果】
![SwitchTest_02](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/51ee3a47-f82f-f67a-a1f1-bbe919476598.gif)

最新の `Observable` の非同期処理のみ実行されていることが確認できました．

---

【参考】

https://nobollel-tech.hatenablog.com/entry/2016/09/27/185223

https://shitakami.hateblo.jp/entry/2021/08/22/204549
