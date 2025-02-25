---
title: 【Unity】uGUIへのデータバインディング関連のメモ
tags:
  - Unity
  - uGUI
  - UniRx
private: false
updated_at: '2025-01-11T13:18:51+09:00'
id: bc2d4d2d41385c11f700
organization_url_name: null
slide: false
ignorePublish: false
---

## 概要

UniRx を学習中であり，モデルと UI を連携する処理を書くことが度々あるため、備忘録としてまとめていきます．

**使用環境**

- Unity UI: 2.0.0
- UniRx: 7.1.0

**対象 UI**

- [Text](#text)
- [Slider](#slider)
- [Toggle](#toggle)
- [InputField](#inputfield)
- [Dropdown](#dropdown)

## Text

単方向 (Model → View)

```.cs
// [MEMO] SubscribeWithStateはクロージャが生成されない分、Subscribeより効率が良いらしい

public static IDisposable SubscribeToText(this IObservable<string> source, TextMeshProUGUI text)
{
    return source.SubscribeWithState(text, (x, t) => t.text = x);
}

public static IDisposable SubscribeToText<T>(this IObservable<T> source, TextMeshProUGUI text)
{
    return source.SubscribeWithState(text, (x, t) => t.text = x.ToString());
}

public static IDisposable SubscribeToText<T>(this IObservable<T> source, TextMeshProUGUI text, Func<T, string> selector)
{
    return source.SubscribeWithState2(text, selector, (x, t, s) => t.text = s(x));
}
```

## Slider

双方向 (Model → Veiw / View → Model)

```.cs
public static void BindToSlider(this IReactiveProperty<float> property, Slider slider, ICollection<IDisposable> disposables)
{
    // Model → View
    property.SubscribeWithState(slider, (x, s) => s.value = x).AddTo(disposables);

    // View → Model
    slider.OnValueChangedAsObservable()
          .SubscribeWithState(property, (x, p) => p.Value = x).AddTo(disposables);
}
```

## Toggle

双方向 (Model → Veiw / View → Model)

```.cs
public static void BindToToggle(this IReactiveProperty<bool> property, Toggle toggle, ICollection<IDisposable> disposables)
{
    // Model → View
    property.SubscribeWithState(toggle, (x, t) => t.isOn = x).AddTo(disposables);

    // View → Model
    toggle.OnValueChangedAsObservable()
          .SubscribeWithState(property, (x, p) => p.Value = x).AddTo(disposables);
}
```

## InputField

単方向 (View → Model)

```.cs
/// <summary> Observe onEndEdit (Submit) event.</summary>
public static IObservable<string> OnEndEditAsObservable(this TMP_InputField inputField)
{
    return inputField.onEndEdit.AsObservable();
}

/// <summary> Observe onValueChanged with current `text` value on subscribe. </summary>
public static IObservable<string> OnValueChangedAsObservable(this TMP_InputField inputField)
{
    return Observable.CreateWithState<string, TMP_InputField>(inputField, (i, observer) => {
        observer.OnNext(i.text);
        return i.onValueChanged.AsObservable().Subscribe(observer);
    });
}
```

双方向 (Model → Veiw / View → Model)

```.cs
// String型
public static void BindToInputField(this IReactiveProperty<string> property, TMP_InputField inputField, ICollection<IDisposable> disposables)
{
    // Model → View
    property.SubscribeWithState(inputField, (x, i) => i.text = x).AddTo(disposables);

    // View → Model
    inputField.OnEndEditAsObservable()
              .SubscribeWithState(property, (x, p) => p.Value = x).AddTo(disposables);
}

// Generic型
public static void BindToInputField<T>(this IReactiveProperty<T> property, TMP_InputField inputField, ICollection<IDisposable> disposables,
                                       Func<string, T> parseFunc, Func<T, string> formatFunc)
{
    // Model → View
    property.SubscribeWithState2(inputField, formatFunc, (x, i, f) => i.text = f(x)).AddTo(disposables);

    // View → Model
    inputField.OnEndEditAsObservable()
              .Subscribe(value => {
                  try
                  {
                      property.Value = parseFunc(value);
                  }
                  catch
                  {
                      // 変換失敗時に入力フィールドをリセット
                      inputField.text = formatFunc(property.Value);
                  }
              })
              .AddTo(disposables);
}

// Int型
public static void BindToInputField(this IReactiveProperty<int> property, TMP_InputField inputField, ICollection<IDisposable> disposables)
{
    property.BindToInputField(inputField, disposables,
                              value => int.Parse(value),
                              value => value.ToString());
}

// Float型
public static void BindToInputFieldFloat(this IReactiveProperty<float> property, TMP_InputField inputField, ICollection<IDisposable> disposables)
{
    property.BindToInputField(inputField, disposables,
                              value => float.Parse(value),
                              value => value.ToString("F2"));
}
```

## Dropdown

単方向 (View → Model)

```.cs
public static IObservable<int> OnValueChangedAsObservable(this TMP_Dropdown dropdown)
{
    return Observable.CreateWithState<int, TMP_Dropdown>(dropdown, (d, observer) => {
        observer.OnNext(d.value);
        return d.onValueChanged.AsObservable().Subscribe(observer);
    });
}
```

双方向 (Model → Veiw / View → Model)

```.cs
public static void BindToDropdown(this IReactiveProperty<int> property, TMP_Dropdown dropdown, ICollection<IDisposable> disposables)
{
    // Model → View
    property.SubscribeWithState(dropdown, (x, d) => d.value = x).AddTo(disposables);

    // View → Model
    dropdown.OnValueChangedAsObservable()
            .SubscribeWithState(property, (x, p) => p.Value = x).AddTo(disposables);
}
```

※オプションを列挙型を対応させるコンポーネント

```DropdownEnumOptions.cs
[DisallowMultipleComponent]
[RequireComponent(typeof(TMP_Dropdown))]
public abstract class DropdownEnumOptions<TEnum> : MonoBehaviour where TEnum : Enum {
    private TMP_Dropdown _dropdown;
    private readonly ReactiveProperty<TEnum> _currentRP = new();

    /// <summary>
    /// 値の変化を通知するObservable
    /// </summary>
    public IObservable<TEnum> OnValueChanged => _currentRP;

    private static readonly TEnum[] enumValues = (TEnum[])Enum.GetValues(typeof(TEnum));

    private static int GetEnumIndex(TEnum type){
        return Array.IndexOf(enumValues, type);
    }

    // LifeCycle Events

    protected void Start(){
        Setup();

        // Viewの更新
        _dropdown.onValueChanged.AsObservable()
                 .Subscribe(index => {
                     if (index.IsInRange(enumValues))
                         _currentRP.Value = enumValues[index];
                     else
                         Setup();
                 })
                 .AddTo(this);

        // RPの更新
        _currentRP
            .Subscribe(type => {
                _dropdown.value = GetEnumIndex(type);
                _dropdown.RefreshShownValue();
            }).AddTo(this);
    }

    private void OnDestroy() {
        _currentRP?.Dispose();
    }

    private void OnValidate() {
        if (_dropdown == null)
            _dropdown = GetComponent<TMP_Dropdown>();
    }

    // Private Method

    /// <summary> ゲッタ </summary>
    public TEnum GetValue() {
        return _currentRP.Value;
    }

    /// <summary> セッタ </summary>
    public void SetValue(TEnum type) {
        _currentRP.Value = type;
    }

    private void Setup() {
        OnValidate();

        // Enumの名前リストを取得してDropdownのオプションに設定
        _dropdown.options.Clear();
        _dropdown.options.AddRange(enumValues.Select(name => new TMP_Dropdown.OptionData(name.ToString())));

        // 初期選択を最初の項目に設定
        _dropdown.value = GetEnumIndex(_currentRP.Value);
        _dropdown.RefreshShownValue();
    }
}


// -----
// 使用例
public enum MyType { ModeA, ModeB, ModeC, ModeD,}
public sealed class TestDropdownOptions : DropdownEnumOptions<MyType> {}
```
