---
title: 【Unity】Screen.width / heightはOnEnable内で使わない方がよさそう
tags:
  - Unity
private: false
updated_at: "2024-08-06T09:17:16+09:00"
id: 3e82c85399df69b0002f
organization_url_name: null
slide: false
ignorePublish: false
---

## 概要

ある MonoBehaviour の OnEnable() / OnDisable()内で<b>Screen.width / height</b>を使用した際に、予期しない値が返されていた．Unity2021.2.10f1 を使用．

公式ドキュメント：https://docs.unity3d.com/ScriptReference/Screen.html

## 結論

<b>Screen.width</b>や<b>Screen.height</b>を OnEnable()内で使用すると、正しくない値が返されることがあるらしい．

- [Issue Tracker 1](https://issuetracker.unity3d.com/issues/screen-dot-width-and-screen-dot-height-values-in-onenable-function-are-incorrect)
- [Issue Tracker 2](https://issuetracker.unity3d.com/issues/screen-dot-width-slash-screen-dot-height-in-onenable-shows-inspector-window-size-when-the-component-is-enabled-by-a-toggle-in-inspector-window)

（※自分はまだ上記バージョンでしか確認作業を行ってない．）

## 詳細

#### 【確認のため使用したコード】

以下のコードで MonoBehaviour の Update、OnEnable、OnDisable 時の値を記録してインスペクタ上に表示した．また、Editor クラスの OnInspectorGUI での値も同時に確認した．

<details> <summary>ScreenSizeDebugger.cs</summary>

```csharp
using System.Collections.Generic;
using UnityEngine;
using System;
#if UNITY_EDITOR
using UnityEditor;
#endif

//[ExecuteAlways]  // ※ExecuteAlwaysでもほぼ同様の結果だった
public class ScreenSizeDebugger: MonoBehaviour{

    // サイズの記録
    public Vector2 size_Update;
    public Vector2 size_OnEnable;
    public Vector2 size_OnDisable;

    //
    public DateTime lastEnableTime = DateTime.MinValue;
    public DateTime lastDisableTime = DateTime.MinValue;

    private void Update() {
        size_Update =  new Vector2(Screen.width, Screen.height);
    }

    private void OnEnable() {
        size_OnEnable =  new Vector2(Screen.width, Screen.height);
        lastEnableTime = DateTime.Now;
    }

    private void OnDisable() {
        size_OnDisable =  new Vector2(Screen.width, Screen.height);
        lastDisableTime = DateTime.Now;
    }
}

#if UNITY_EDITOR
[CustomEditor(typeof(ScreenSizeDebugger))]
public class ScreenSizeDebuggerEditor : Editor {

    public override void OnInspectorGUI() {
        var instance = target as ScreenSizeDebugger;

        // MonoBehaviour
        var modeText = Application.isPlaying ? "Play mode" : "Edit mode";
        DrawScreenSizeInfo($"MonoBehaviour.<b>Update</b> ({modeText})", instance.size_Update.x, instance.size_Update.y);

        DrawScreenSizeInfo($"MonoBehaviour.<b>OnEnable</b> ({modeText})", instance.size_OnEnable.x, instance.size_OnEnable.y);
        EditorGUILayout.LabelField($"elaposed time {(DateTime.Now - instance.lastEnableTime).ToString("ss")} [s]");

        DrawScreenSizeInfo($"MonoBehaviour.<b>OnDisable</b> ({modeText})", instance.size_OnDisable.x, instance.size_OnDisable.y);
        EditorGUILayout.LabelField($"elaposed time {(DateTime.Now - instance.lastDisableTime).ToString("ss")} [s]");


        // Editor
        DrawScreenSizeInfo($"Editor.OnInspector", Screen.width, Screen.height);

        EditorUtil.GUI.HorizontalLine();
        // -------------------

        // インスペクタが表示されている画面の解像度
        EditorGUILayout.LabelField("Current Resolution", $"{Screen.currentResolution.width} x {Screen.currentResolution.height} @ {Screen.currentResolution.refreshRate}Hz");

        // 登録されている全ての解像度
        EditorGUILayout.LabelField("Supported Resolutions:");
        using (new EditorGUI.IndentLevelScope()) {
            foreach (var resolution in Screen.resolutions) {
                EditorGUILayout.LabelField($"{resolution.width} x {resolution.height} @ {resolution.refreshRate}Hz");
            }
        }
    }

    private void DrawScreenSizeInfo(string labelText, float width, float height) {
        EditorUtil.GUI.HorizontalLine();

        // SCREEN SIZE
        EditorGUILayout.LabelField(labelText, Styles.label);
        using (new EditorGUILayout.VerticalScope(GUI.skin.box))
        using (new EditorGUI.IndentLevelScope()) {
            EditorGUILayout.LabelField("Screen Width", width.ToString(), Styles.label);
            EditorGUILayout.LabelField("Screen Height", height.ToString(), Styles.label);
        }
    }

    private static class Styles {
        public static GUIStyle label;
        static Styles() {
            label = new GUIStyle(GUI.skin.label) {
                richText = true,
            };
        }
    }
}
#endif
```

</details>

<details> <summary>Util.cs</summary>

```util.cs
public static partial class EditorUtil {
    public static partial class GUI {

        /// <summary>
        /// 仕切り線を表示する
        /// </summary>
        public static void HorizontalLine(Color color, int thickness = 1, int padding = 10) {

            using (new EditorGUILayout.HorizontalScope()) {
                var splitterRect = EditorGUILayout.GetControlRect(false, GUILayout.Height(thickness + padding));
                splitterRect = EditorGUI.IndentedRect(splitterRect);
                splitterRect.height = thickness;
                splitterRect.y += padding / 2;

                EditorGUI.DrawRect(splitterRect, color);
            }
        }
    }
}
```

</details>

#### 【結果】

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/2e01c30d-234b-790e-f01a-670a2d9b61ed.png)

Update では設定した Game 画面の解像度の値、Editor.OnInspectorGUI()では<u>インスペクタウインドウのサイズ</u>が返される．

一方、OnEnable/OnDisable は基本的に Editor.OnInspectorGUI()と同様の値が入っていた．
(※PlayMode に入るときの最初の呼び出し時のみ、Update と同じ値だった．)
