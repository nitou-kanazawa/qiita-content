---
title: 【Unity】Debug 関連のメモ
tags:
  - Unity
private: false
updated_at: '2025-01-11T13:18:51+09:00'
id: 8836e6eabe63e122adb1
organization_url_name: null
slide: false
ignorePublish: false
---

## Debug.Log

<details><summary>UnityEngine.Debugのラッパー</summary>

```Debug_.cs

    /// <summary>
    /// Debugのラッパークラス
    /// </summary>
    public static partial class Debug_ {

        /// ----------------------------------------------------------------------------
        #region Public Method (基本ログ)

        /// <summary>
        /// UnityEditor上でのみ実行されるLogメソッド
        /// </summary>
        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void Log(object o) => Debug.Log(FormatObject(o));

        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void Log(object o, Color color) => Debug.Log(FormatObject(o).WithColorTag(color));

        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void LogWarning(object o) => Debug.LogWarning(FormatObject(o));

        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void LogWarning(object o, Color color) => Debug.LogWarning(FormatObject(o).WithColorTag(color));

        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void LogError(object o) => Debug.LogError(o);

        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void LogError(object o, Color color) => Debug.LogError(FormatObject(o).WithColorTag(color));

        #endregion


        /// ----------------------------------------------------------------------------
        #region Public Method (コレクション)

        private static readonly int MAX_ROW_NUM = 100;

        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void ListLog<T>(IReadOnlyList<T> list) => Log(list.Convert<T>());

        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void ListLog<T>(IReadOnlyList<T> list, Color color) => Log(list.Convert<T>(), color);


        [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
        public static void DictLog<TKey, TValue>(IReadOnlyDictionary<TKey, TValue> dict) => Log(dict.Convert<TKey, TValue>());

        #endregion


        /// ----------------------------------------------------------------------------
        #region Private Method (変換メソッド)

        /// <summary>
        /// 文字列への変換（※null，空文字が判別できる形式）
        /// </summary>
        private static string FormatObject(object o) {
            if (o is null) {
                return "(null)";
            }
            if (o as string == string.Empty) {
                return "(empty)";
            }
            return o.ToString();
        }

        /// <summary>
        /// リスト要素を文字列に変換する
        /// </summary>
        private static string Convert<T>(this IReadOnlyList<T> list) {
            if (list == null) return "(null)";

            var sb = new StringBuilder();
            sb.Append($"(The total number of elements is {list.Count})\n");

            // 文字列へ変換
            for (int index = 0; index < list.Count; index++) {
                // 最大行数を超えた場合，
                if (index >= MAX_ROW_NUM) {
                    sb.Append($"(+{list.Count - MAX_ROW_NUM} items has been omitted)");
                    break;
                }
                // 要素追加
                sb.Append($"[ {index} ] = {list[index]} \n".WithIndentTag());
            }
            return sb.ToString();
        }

        /// <summary>
        /// ディクショナリ要素を文字列に変換する
        /// </summary>
        private static string Convert<TKey, TValue>(this IReadOnlyDictionary<TKey, TValue> dict) {
            if (dict == null) return "(null)";

            var sb = new StringBuilder();
            sb.Append($"(The total number of elements is {dict.Count})\n");

            // 文字列へ変換
            int index = 0;
            foreach ((var key, var value) in dict) {
                // 最大行数を超えた場合，
                if (index >= MAX_ROW_NUM) {
                    sb.Append($"(+{dict.Count - MAX_ROW_NUM} items has been omitted)");
                    break;
                }
                // 要素の追加
                sb.Append($"[ {key} ] = {value} \n".WithIndentTag());
                index++;
            }
            return sb.ToString();
        }
        #endregion
    }

```

```RichTextUtil.cs
/// <summary>
    /// 文字列をリッチテキストへ変換する拡張メソッド集
    /// </summary>
    public static class RichTextUtil {

        /// <summary>
        /// カラータグを挿入する
        /// </summary>
        public static string WithColorTag(this string @this, Color color) {
            return string.Format("<color=#{0:X2}{1:X2}{2:X2}>{3}</color>",
                (byte)(color.r * 255f),
                (byte)(color.g * 255f),
                (byte)(color.b * 255f),
                @this
            );
        }

        /// <summary>
        /// 太字タグを挿入する
        /// </summary>
        public static string WithBoldTag(this string @this) {
            return $"<b>{@this}</b>";
        }

        /// <summary>
        /// 斜体タグを挿入する
        /// </summary>
        public static string WithItalicTag(this string @this) {
            return $"<i>{@this}</i>";
        }

        /// <summary>
        /// サイズタグを挿入する
        /// </summary>
        public static string WithSizeTag(this string @this, int size) {
            return $"<size={size}>{@this}</size>";
        }

        /// <summary>
        /// インデントタグを挿入する
        /// </summary>
        public static string WithIndentTag(this string @this, int charNum = 1) {
            return $"<indent={Mathf.Clamp(charNum, 1, 20)}em>{@this}</indent>";
        }
    }
```

</details>

参考

https://qiita.com/toRisouP/items/d856d65dcc44916c487d

https://zenn.dev/happy_elements/articles/38be21755773e0

https://nmxi.hateblo.jp/entry/2019/02/24/235216

https://docs.unity3d.com/ja/2022.3/Manual/UIE-supported-tags.html

https://kan-kikuchi.hatenablog.com/entry/DEVELOPMENT_BUILD_DEBUG

## Null チェック

```Error.cs
public static class Error {
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public static void ArgumentNullException<T>(T value) {
            if (value == null) throw new ArgumentNullException();
        }
    }
```

参考

https://baba-s.hatenablog.com/entry/2021/11/16/090000#google_vignette

https://qiita.com/msm2001/items/b57f6b92371026307c0b

https://sat-box.hatenablog.jp/entry/2022/05/20/133607
