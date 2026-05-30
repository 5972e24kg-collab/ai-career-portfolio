# Work064: VRMアニメーション制御とReact基盤最適化

Tag: [[React Three Fiber]], [[VRM]], [[AnimationMixer]], [[Mixamo]], [[Zustand]]

## 1. 目的

* ジェムちゃん（VRM）に対して、呼吸、瞬き、リップシンクといった生命感を付与する。
* 外部アニメーションデータ（`.vrma`, `.fbx`）を読み込み、`THREE.AnimationMixer` を用いて再生する。
* Desktop Mate用途を想定し、バストアップでのUI固定表示と、Reactコンポーネントのパフォーマンス最適化を行う。

## 2. システム環境

* **Node:** Client (開発用クライアントPC / IntelliJ IDEA)
* **Target:** Vite + React + R3F アプリケーション (`VrmGemchanAvatar.js`, `AvatarState.js` 等)

## 3. 作業ログ・解決プロセス

### 発生した課題と検証

1. **物理演算の暴走とモデルの消滅**

   * Tポーズ解除時にSpringBone（髪や服の揺れ）に地球の重力(-9.8)を設定した結果、演算が破綻しモデルが画面外へ消滅。重力をデフォルトに委ねることで解決した。
2. **R3F特有の `clock.getDelta()` トラップ**

   * `useFrame` 内で `clock.getDelta()` を呼び出すと常に0が返り、瞬きや物理演算が停止。`useFrame` の第2引数からシステム計算済みの `delta` を受け取ることで解決。
3. **カメラの「足元表示」トラップ**

   * `Camera Position` を調整しても足元が映る問題。カメラの「立ち位置（Position）」に加え、`OrbitControls` の「注視点（Target）」を顔の高さに設定する必要があることを体得した。
4. **Mixamo FBXの動的リターゲティングの限界**

   * Mixamoのモーション（`.fbx`）をブラウザ上で読み込み、スケール差（cmとm）の補正、およびRest Pose（初期姿勢）のクォータニオン相殺までは確認できた。
   * しかし、VRMとMixamo間の「ボーンのローカル軸（X/Y/Zの向き）の根本的な不一致」により関節のねじれが発生。JavaScriptによる動的補正は計算コストと調整難易度が高く、今回の用途では保守しづらいと判断したため、Unityを用いた事前変換（`.vrma` へのBake）への方針転換（Pivot）を決断した。
5. **Zustandの「再レンダリング地獄」**

   * Store全体を分割代入で取得していたため、無関係なState変化でも3Dアバター全体が再描画される状態だった。セレクタによる個別取得へ修正し、パフォーマンスを担保した。

### 解決策（方針転換）

* ブラウザ側でのリターゲティングは破棄し、`@pixiv/three-vrm-animation` で扱える `.vrma` 形式のみを扱う、安定運用しやすいコンポーネントとして完成度を高める方針へ移行。

## 4. 成果物

リファクタリングおよび `.vrma` 再生機能が統合された `VrmGemchanAvatar.js` のベースコード。

```javascript
import React, { useEffect, useState } from 'react';
import { useFrame } from '@react-three/fiber';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import { VRMLoaderPlugin } from '@pixiv/three-vrm';
import { VRMAnimationLoaderPlugin, createVRMAnimationClip } from '@pixiv/three-vrm-animation';
import * as THREE from 'three';
import { useAvatarState } from "./AvatarState";

export function VrmGemchanAvatar() {
    const characterVrm = '/vrm/GemChanV5.1.vrm';
    const idleLoopScene = '/vrm/scene/idle_loop.vrma';
    
    const [vrm, setVrm] = useState(null);
    const [mixer, setMixer] = useState(null);
    
    // 再レンダリング最適化（セレクタ取得）
    const emotion = useAvatarState((state) => state.emotion);
    const isSpeaking = useAvatarState((state) => state.isSpeaking);

    // 1. VRM本体のロード
    useEffect(() => {
        const loader = new GLTFLoader();
        loader.register((parser) => new VRMLoaderPlugin(parser));

        loader.load(
            characterVrm,
            (gltf) => {
                const vrmInstance = gltf.userData.vrm;
                vrmInstance.scene.rotation.y = Math.PI;
                setVrm(vrmInstance);
            }
        );
    }, []);

    // 2. VRMAのロードとAnimationMixerのセットアップ
    useEffect(() => {
        if (!vrm) return;

        const loader = new GLTFLoader();
        loader.register((parser) => new VRMAnimationLoaderPlugin(parser));

        loader.load(
            idleLoopScene,
            (gltf) => {
                const vrmAnimations = gltf.userData.vrmAnimations;
                if (vrmAnimations && vrmAnimations.length > 0) {
                    const clip = createVRMAnimationClip(vrmAnimations[0], vrm);
                    const newMixer = new THREE.AnimationMixer(vrm.scene);
                    const action = newMixer.clipAction(clip);
                    action.play();
                    
                    setMixer(newMixer);
                }
            }
        );
    }, [vrm]);

    // 3. 表情の更新
    useEffect(() => {
        if (!vrm || !vrm.expressionManager) return;
        ['neutral', 'happy', 'angry', 'sad', 'relaxed'].forEach(exp => {
            vrm.expressionManager.setValue(exp, 0);
        });
        const targetExpression = vrm.expressionManager.getExpression(emotion) ? emotion : 'neutral';
        vrm.expressionManager.setValue(targetExpression, 1.0);
    }, [vrm, emotion]);

    // 4. 瞬き用のRef
    const blinkState = React.useRef({ timer: 0, isBlinking: false, blinkValue: 0, nextBlinkTime: 3 });

    // 5. 毎フレームの更新
    useFrame(({ clock }, delta) => {
        if (!vrm) return;
        const time = clock.elapsedTime;

        if (mixer) mixer.update(delta);

        if (vrm.expressionManager) {
            // リップシンク
            if (isSpeaking) {
                const mouthOpen = (Math.sin(time * 15) + 1) / 2;
                vrm.expressionManager.setValue('aa', mouthOpen * 0.8);
            } else {
                vrm.expressionManager.setValue('aa', 0);
            }

            // 自動瞬き
            blinkState.current.timer += delta;
            if (!blinkState.current.isBlinking && blinkState.current.timer > blinkState.current.nextBlinkTime) {
                blinkState.current.isBlinking = true;
                blinkState.current.timer = 0;
                blinkState.current.nextBlinkTime = 3 + Math.random() * 3;
            }
            if (blinkState.current.isBlinking) {
                blinkState.current.blinkValue += delta * 15;
                const blinkAmount = Math.sin(blinkState.current.blinkValue);
                if (blinkAmount < 0) {
                    blinkState.current.isBlinking = false;
                    blinkState.current.blinkValue = 0;
                    vrm.expressionManager.setValue('blink', 0);
                } else {
                    vrm.expressionManager.setValue('blink', blinkAmount);
                }
            }
        }
        vrm.update(delta); // 物理演算へ正しいdeltaを渡す
    });

    return vrm ? <primitive object={vrm.scene} /> : null;
}
```
