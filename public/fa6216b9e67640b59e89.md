---
title: 【Unity】IEventSystemHandlerの実行タイミング
tags:
  - Unity
  - uGUI
private: false
updated_at: "2024-07-23T13:09:05+09:00"
id: fa6216b9e67640b59e89
organization_url_name: null
slide: false
ignorePublish: false
---

## 概要

UI (Image) に IDragHandler を持つコンポーネントが複数アタッチされており、それらの実行順番を制御したかった．

## 結論

GameObject に IEventSystemHandler を実装したコンポーネントが複数アタッチされている場合、それらの実行順はアタッチした順番（gameObject.GetComponents()で取得される並び）で決定される．

※自分は勘違いしていたが ExecutionOrder は関係ない．

## OnDrag の実行タイミング

そもそも OnDrag()等がどのタイミングで実行されるのか把握していなかったため、
以下のログ出力を行うスクリプトで確認．

```.cs
public class TestComponent : MonoBehaviour, IBeginDragHandler, IDragHandler, IEndDragHandler {

    public RectTransform rect;

    //
    public void Update() => Debug.Log($"{Time.frameCount} Update");
    public void FixedUpdate() => Debug.Log($"{Time.frameCount} FixedUpdate");

    //
    public void OnBeginDrag(PointerEventData eventData) => Debug_.Log($"{Time.frameCount} OnBeginDrag", Colors.Red);
    public void OnEndDrag(PointerEventData eventData) => Debug_.Log($"{Time.frameCount} OnEndDrag", Colors.Red);

    public void OnDrag(PointerEventData eventData) {
        Debug_.Log($"{Time.frameCount} OnDrag", Colors.Orange);

        {   // マウス位置に追従させる
            Vector2 localPoint;
            RectTransformUtility.ScreenPointToLocalPointInRectangle(rect.parent as RectTransform, eventData.position, eventData.pressEventCamera, out localPoint);
            rect.localPosition = localPoint;
        }
    }
}
```

【結果】
FixedUpdate() > OnBegineDrag() > OnDrag() > Update() の順に実行されている．
（おそらく MonoBehaviour の OnMouseXXX と同じタイミングで実行されている）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/367835c3-113e-c5ca-fd69-1eadcfe7bb90.png)

## 複数アタッチした時の実行順

#### DefaultExecutionOrder 設定による変化

コンポーネント A, B の DefaultExecutionOrder の値を入れ替えても実行順番に変化は無かった．

```.cs
[DefaultExecutionOrder(-100)]
public class ExecutionOrderTestA{}

[DefaultExecutionOrder(100)]
public class ExecutionOrderTestB{}
```

#### コンポーネントのアタッチ順による変化

アタッチされている順番を入れ替えることで OnBeginDrag()/OnDrag()/OnEndDrag()の実行順番も逆になった．

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1596227/c50cb5c9-5ea5-cfd4-c91f-91e92edad3e2.png)

調べてみると UnityEngine.EventSystems.ExecuteEvents が処理する際に、
普通に gameObject.GetComponets()で取得した順に処理しているようだった．

```ExecuteEvents.cs
        /// <summary>
        /// Bubble the specified event on the game object, figuring out which object will actually receive the event.
        /// </summary>
        public static GameObject GetEventHandler<T>(GameObject root) where T : IEventSystemHandler
        {
            if (root == null)
                return null;

            Transform t = root.transform;
            while (t != null)
            {
                if (CanHandleEvent<T>(t.gameObject))
                    return t.gameObject;
                t = t.parent;
            }
            return null;
        }

        /// <summary>
        /// Whether the specified game object will be able to handle the specified event.
        /// </summary>
        public static bool CanHandleEvent<T>(GameObject go) where T : IEventSystemHandler{
            var internalHandlers = s_HandlerListPool.Get();
            GetEventList<T>(go, internalHandlers);
            var handlerCount = internalHandlers.Count;
            s_HandlerListPool.Release(internalHandlers);
            return handlerCount != 0;
        }

        /// <summary>
        /// Get the specified object's event event.
        /// </summary>
        private static void GetEventList<T>(GameObject go, IList<IEventSystemHandler> results)
        where T : IEventSystemHandler{
            // Debug.LogWarning("GetEventList<" + typeof(T).Name + ">");
            if (results == null)
                throw new ArgumentException("Results array is null", "results");

            if (go == null || !go.activeInHierarchy)
                return;

            var components = ListPool<Component>.Get();
            go.GetComponents(components);
            for (var i = 0; i < components.Count; i++){
                if (!ShouldSendToComponent<T>(components[i]))
                    continue;

                // Debug.Log(string.Format("{2} found! On {0}.{1}", go, s_GetComponentsScratch[i].GetType(), typeof(T)));
                results.Add(components[i] as IEventSystemHandler);
            }
            ListPool<Component>.Release(components);
            // Debug.LogWarning("end GetEventList<" + typeof(T).Name + ">");
        }

```

【参考】

https://github.com/Unity-Technologies/uGUI/blob/135661e057d2a45f5b5b73ea48fb67a445816f61/UnityEngine.UI/EventSystem/ExecuteEvents.cs

https://hacchi-man.hatenablog.com/entry/2021/01/14/220000
