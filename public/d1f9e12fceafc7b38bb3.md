---
title: 【Unity】Vetor3用のInputFieldを作る
tags:
  - Unity
  - uGUI
private: false
updated_at: '2024-11-14T09:10:47+09:00'
id: d1f9e12fceafc7b38bb3
organization_url_name: null
slide: false
ignorePublish: false
---
## 概要
Vector3や配列等の複数の要素を持つ型に使用できるInputFieldが必要だったため、汎用的に使用できそうなものを作成した．カスタムInputFieldは、入力値を内部でReactiveProperty<T>に変換して管理し、外部へはその変換後の値を通知する．また、Modelとの連携を容易にするためのインタフェースを実装しておく．

## 実装の詳細

**インタフェース定義**
ゲッタ/セッタと通知用Observableを持つインタフェース．

``` IDataHolder.cs
namespace nitou {

    /// <summary>
    /// <see cref="T"/>型のデータを保持できるオブジェクト．
    /// </summary>
    public interface IDataHolder<T> {
        public IObservable<T> OnValueChanged { get; }
        // ※必要に応じてバリデーションを差し込む
        public T GetValue();
        public void SetValue(T value);
    }

    public static class DataHolderExtensions {

        public static void BindTo<T>(this IReactiveProperty<T> property, IDataHolder<T> target, ICollection<IDisposable> disposables) {

            // Property → Target
            property.SubscribeWithState(target, (x, t) => t.SetValue(x)).AddTo(disposables);

            // Targer → Property
            target.OnValueChanged.SubscribeWithState(property, (x, p) => p.Value = x).AddTo(disposables);
        }
    }
}

```


**基本のInputFieldビュークラス**


``` InputFieldView.cs
using System;
using UniRx;
using UnityEngine;

namespace nitou.UI{
    public abstract class InputFieldView<T> : MonoBehaviour, IDataHolder<T>{

        protected readonly ReactiveProperty<T> _valueRP = new();
        public IObservable<T> OnValueChanged => _valueRP;

        // LifeCycle Events
        protected virtual void OnDestroy() {
            _valueRP.Dispose();
        }

        // Public Method
        public virtual T GetValue() => _valueRP.Value;
        public virtual void SetValue(T newValue) => _valueRP.Value = newValue;
        public virtual void ResetValue() => _valueRP.Value = default(T);
    }
}

```

**Vector3用のカスタムInputField**

※入力文字列のバリデーションは別途InputFieldインスペクタで設定する想定．

``` Vector3InputFieldView.cs
using System;
using System.Globalization;
using UniRx;
using UnityEngine;
using TMPro;

namespace nitou.UI {

    public sealed class Vector3InputFieldView : InputFieldView<Vector3> {

        [Header("View")]
        [SerializeField] private TMP_InputField _xInputField;
        [SerializeField] private TMP_InputField _yInputField;
        [SerializeField] private TMP_InputField _zInputField;

        [Header("Settings")]
        [Range(0, 6)]
        [SerializeField] private int _decimalPlaces = 2;


        /// ----------------------------------------------------------------------------
        // LifeCycle Events

        private void Awake() {
            // ViewModelの監視
            _valueRP.Subscribe(v => ApplyValue(v)).AddTo(this);

            // Viewの監視
            Observable.Merge(
                _xInputField.onEndEdit.AsObservable().AsUnitObservable(),
                _yInputField.onEndEdit.AsObservable().AsUnitObservable(),
                _zInputField.onEndEdit.AsObservable().AsUnitObservable()
            )
            .Subscribe(_ => UpdateValue())
            .AddTo(this);
        }


        /// ----------------------------------------------------------------------------
        // Private Method

        private void UpdateValue() {
            Vector3 currentValue = _valueRP.Value;
            Vector3? parsedValue = TryParseValue();

            // パースが成功した場合のみReactivePropertyに反映
            if (parsedValue.HasValue) {
                _valueRP.Value = parsedValue.Value;
            } else {
                // ※パースに失敗した場合は元の値を再適用
                ApplyValue(currentValue);
            }
        }

        // Viewから値を読み取る．
        private Vector3? TryParseValue() {

            // parse values
            bool xParsed = float.TryParse(_xInputField.text, NumberStyles.Float, CultureInfo.InvariantCulture, out var x);
            bool yParsed = float.TryParse(_yInputField.text, NumberStyles.Float, CultureInfo.InvariantCulture, out var y);
            bool zParsed = float.TryParse(_zInputField.text, NumberStyles.Float, CultureInfo.InvariantCulture, out var z);

            // 全てのパースが成功した場合にのみ値を返す
            if (xParsed && yParsed && zParsed) {
                return new Vector3(x, y, z);
            }

            // いずれかのパースに失敗した場合はnullを返す
            return null;
        }

        // Viewに値を適用する．
        private void ApplyValue(Vector3 value) {
            _xInputField.text = value.x.ToFloatText(_decimalPlaces);
            _yInputField.text = value.y.ToFloatText(_decimalPlaces);
            _zInputField.text = value.z.ToFloatText(_decimalPlaces);
        }
    }
}

```
