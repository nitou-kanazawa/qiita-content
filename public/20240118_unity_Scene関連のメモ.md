---
title: 【Unity】Scene関連のメモ
tags:
  - C#
  - Unity
  - SceneManager
private: false
updated_at: "2024-09-21T21:08:34+09:00"
id: ffa7055f3c51646540cb
organization_url_name: null
slide: false
ignorePublish: false
---

## 目次

- [SceneManager の機能拡張](https://qiita.com/nitto/items/ffa7055f3c51646540cb#scenemanager%E3%81%AE%E6%A9%9F%E8%83%BD%E6%8B%A1%E5%BC%B5)
- [Scene に対する処理](https://qiita.com/nitto/items/ffa7055f3c51646540cb#scene%E3%81%AB%E5%AF%BE%E3%81%99%E3%82%8B%E5%87%A6%E7%90%86)
- [Scene アセットをインスペクタで参照する](https://qiita.com/nitto/items/ffa7055f3c51646540cb#scene%E3%82%A2%E3%82%BB%E3%83%83%E3%83%88%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%9A%E3%82%AF%E3%82%BF%E3%81%A7%E5%8F%82%E7%85%A7%E3%81%99%E3%82%8B)
- [Scene のエディタ操作関連](https://qiita.com/nitto/items/ffa7055f3c51646540cb#scene%E3%81%AE%E3%82%A8%E3%83%87%E3%82%A3%E3%82%BF%E6%93%8D%E4%BD%9C%E9%96%A2%E9%80%A3)

## SceneManager の機能拡張

```SceneNavigator.cs
using System.Linq;
using UnityEngine.SceneManagement;
using Cysharp.Threading.Tasks;

// SceneManagerに機能を追加した静的クラス
public static class SceneNavigator {

    /// <summary>
    /// 現在読み込まれている全シーンを取得する
    /// </summary>
    public static IEnumerable<Scene> GetAllScenes() {
        var sceneCount = SceneManager.sceneCount;
        for (var i = 0; i < sceneCount; i++) {
            yield return SceneManager.GetSceneAt(i);
        }
    }

    /// <summary>
    /// シーンが読み込まれているか確認する
    /// </summary>
    public static bool IsLoaded(string sceneName) =>
        GetAllScenes().Any(x => x.name == sceneName && x.isLoaded);

    /// <summary>
    /// シーンを取得する．存在しない場合は読み込んで取得する
    /// </summary>
    public static async UniTask<Scene> GetOrLoadSceneAsync(string sceneName) {
        if (!IsLoaded(sceneName)) {
            await SceneManager.LoadSceneAsync(sceneName, LoadSceneMode.Additive);
        }
        return SceneManager.GetSceneByName(sceneName);
    }
}
```

↓ 参考

https://baba-s.hatenablog.com/entry/2022/11/28/162103

## Scene に対する処理

```SceneExtensions.cs
using UnityEngine;
using UnityEngine.SceneManagement;

public static class SceneExtensions {

    /// <summary>
    /// 指定シーン内のルートからコンポーネントを取得する
    /// </summary>
    public static bool TryGetComponentInSceneRoot<T>(this Scene scene, out T result) {
        if (!scene.IsValid()) {
            throw new System.ArgumentException("Scene is invalid.", nameof(scene));
        }

        // シーン内のルートオブジェクトを順にチェックする
        foreach (GameObject rootObj in scene.GetRootGameObjects()) {
            if(rootObj.TryGetComponent(out result)) {
                return true;
            }
        }

        result = default;
        return false;
    }
```

## Scene アセットをインスペクタで参照する

```SceneObject.cs
using System;
using UnityEngine;
#if UNITY_EDITOR
using UnityEditor;
#endif

[Serializable]
public class SceneObject {

    [SerializeField] private string _sceneName;

    public static implicit operator string(SceneObject sceneObject) {
        return sceneObject._sceneName;
    }

    public static implicit operator SceneObject(string sceneName) {
        return new SceneObject() { _sceneName = sceneName };
    }
}


/// ----------------------------------------------------------------------------
#if UNITY_EDITOR
[CustomPropertyDrawer(typeof(SceneObject))]
public class SceneObjectEditor : PropertyDrawer {

    protected SceneAsset GetSceneObject(string sceneObjectName) {
        if (string.IsNullOrEmpty(sceneObjectName)) return null;

        for (int i = 0; i < EditorBuildSettings.scenes.Length; i++) {
            EditorBuildSettingsScene scene = EditorBuildSettings.scenes[i];
            if (scene.path.IndexOf(sceneObjectName) != -1) {
                return AssetDatabase.LoadAssetAtPath(scene.path, typeof(SceneAsset)) as SceneAsset;
            }
        }

        Debug.Log("Scene [" + sceneObjectName + "] cannot be used. Add this scene to the 'Scenes in the Build' in the build settings.");
        return null;
    }

    public override void OnGUI(Rect position, SerializedProperty property, GUIContent label) {
        var sceneObj = GetSceneObject(property.FindPropertyRelative("_sceneName").stringValue);
        var newScene = EditorGUI.ObjectField(position, label, sceneObj, typeof(SceneAsset), false);
        if (newScene == null) {
            var prop = property.FindPropertyRelative("_sceneName");
            prop.stringValue = "";
        } else {
            if (newScene.name != property.FindPropertyRelative("_sceneName").stringValue) {
                var scnObj = GetSceneObject(newScene.name);
                if (scnObj == null) {
                    Debug.LogWarning("The scene " + newScene.name + " cannot be used. To use this scene add it to the build settings for the project.");
                } else {
                    var prop = property.FindPropertyRelative("_sceneName");
                    prop.stringValue = newScene.name;
                }
            }
        }
    }
}
#endif
```

↓ 参考

https://baba-s.hatenablog.com/entry/2017/11/14/110000

## Scene イベントの購読

```ObservableSceneEvent.cs
using System;
using UnityEngine.Events;
using UnityEngine.SceneManagement;
using UniRx;

namespace nitou.SceneSystem {

    // SceneManagerのイベントをObserbableに変換する静的クラス
    public static class ObservableSceneEvent {

        // "activeSceneChanged"イベントをObservableに変換する
        public static IObservable<Tuple<Scene, Scene>> ActiveSceneChangedAsObservable() {
            return Observable.FromEvent<UnityAction<Scene, Scene>, Tuple<Scene, Scene>>(
                h => (x, y) => h(Tuple.Create(x, y)),
                h => SceneManager.activeSceneChanged += h,
                h => SceneManager.activeSceneChanged -= h);
        }

        // "sceneLoaded"イベントをObservableに変換する
        public static IObservable<Tuple<Scene, LoadSceneMode>> SceneLoadedAsObservable() {
            return Observable.FromEvent<UnityAction<Scene, LoadSceneMode>, Tuple<Scene, LoadSceneMode>>(
                h => (x, y) => h(Tuple.Create(x, y)),
                h => SceneManager.sceneLoaded += h,
                h => SceneManager.sceneLoaded -= h);
        }

        // "sceneUnloaded"イベントをObservableに変換する
        public static IObservable<Scene> SceneUnloadedAsObservable() {
            return Observable.FromEvent<UnityAction<Scene>, Scene>(
                h => h.Invoke,
                h => SceneManager.sceneUnloaded += h,
                h => SceneManager.sceneUnloaded -= h);
        }
    }
}
```

## Scene のエディタ操作関連

// Editor window

https://tyfkda.github.io/blog/2021/07/15/unity-scene-switcher.html

https://learning.unity3d.jp/1429/

// Toobar
※以下を参考に ToolBar にシーン切り替えを追加できそう

https://qiita.com/marv_nakazawaka/items/7d50320c70d9560bc5d9

https://github.com/BennyKok/unity-toolbar-buttons
