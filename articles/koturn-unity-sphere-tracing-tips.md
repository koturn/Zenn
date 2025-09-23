---
title: "UnityのBRPにおけるスフィアトレーシングを用いたレイマーチングのTips"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "BRP", "Shader", "Raymarching", "SphereTracing"]
published: false
---

この記事はUnityでスフィアトレーシングによるレイマーチングシェーダーを書くときのTipsを紹介する．
対象読者としては[VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】](https://butadiene.booth.pm/items/1829988 "VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】")の内容を予め理解しているものとする．

---

## ベースとなるシェーダー

サンプルコード: [Assets/koturn/SphereTracingTips/01_Basic/Shaders/Basic.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/01_Basic/Shaders/Basic.shader "Assets/koturn/SphereTracingTips/01_Basic/Shaders/Basic.shader")

本記事では最低限の機能を実装した下記のシェーダーをベースとして変更・解説を行っていく．

### 関数名・変数名

#### 頂点シェーダーの出力・フラグメントシェーダーの入力構造体

`SV_POSITION` に対応するメンバの名前は `pos` にしてある．


### ワールド空間/オブジェクト空間は切り替え可能

オブジェクト空間かワールド空間のどちらでレイマーチングを行うかどうかは，シェーダーキーワード `_CALCSPACE_OBJECT` または `_CALCSPACE_WORLD` によって切り替え可能とした．

また，オブジェクト空間におけるレイの視点であるカメラ座標は，頂点シェーダーで事前に計算（ワールド空間におけるカメラ座標をオブジェクト空間座標に変換する行列演算）することにより，多少の処理負荷軽減を試みている．
もっと最適化するならレイの非正規化方向ベクトルを求めるところまで頂点シェーダーでできるが，フラグメントシェーダーでたかだが1回の減算命令を削減するだけの結果としかならないため，対応のモチベーション低い．

ラスタライズでは3頂点の情報の線形補間が行われるが，正規化処理は非線形変換なので，下記2つは異なる結果となる．

- 正規化方向ベクトルを線形補間した結果
- 非正規化方向ベクトルを線形補間した結果を正規化したベクトル

そのため，頂点シェーダーで正規化方向ベクトルの算出はできない．
すなわち，フラグメントシェーダーから方向ベクトルの正規化処理を除去することはできない．

### スケーリング

x軸，y軸，z軸それぞれの方向への拡大・縮小倍率のベクトル `_Scales` を設けている．

```shaderlab
_Scales ("Scale vector", Vector) = (1.0, 1.0, 1.0, 1.0)
```

```hlsl
//! Scale vector.
uniform float3 _Scales;
```

空間自体の伸縮であるため， `map()` への引数自体に `_Scales` の逆数を乗算している．

しかし，それだけでは不十分で，レイの進行長に対しての拡大・縮小倍率の控除が必要となる．
レイの方向ベクトル `rayDir` に対して `Scales` の逆数を乗算したものの長さの逆数を，1回のステップのレイの進行長 `d` に乗算することでこの控除が可能となる．

長さの逆数をそのままコードに落とし込むと `1.0 / length(rayDir * rcpScales)` となるのだが，これは [`sqrt` 命令](thttps://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/sqrt--sm4---asm- "sqrt (sm4 - asm) - Win32 apps | Microsoft Learn")と [`div` 命令](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/div--sm4---asm- "div (sm4 - asm) - Win32 apps | Microsoft Learn")になってしまう．
世の中には[高速に逆平方根を計算するアルゴリズム](https://lipoyang.hatenablog.com/entry/2021/02/06/194619 "高速逆平方根 (fast inverse square root) のアルゴリズム解説 - 滴了庵日録")があり，DirectX11的にも [`rsq`](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/rsq--sm4---asm- "rsq (sm4 - asm) - Win32 apps | Microsoft Learn") という単一の命令がある．
ハードウェア的にもサポートされていると考えられるので，逆平方根の算出には組み込み関数 [`rsqrt()`](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-rsqrt "rsqrt - Win32 apps | Microsoft Learn") を用いる方がよい．

これらを踏まえると，マーチングループ部分のコードは下記のようになる．

```hlsl
const float3 rcpScales = rcp(_Scales);
const float marchingFactor = _MarchingFactor * rsqrt(dot(rayDir * rcpScales, rayDir * rcpScales));

float rayLength = 0.0;
float d = asfloat(0x7f800000);  // +inf
for (int rayStep = 0; d >= _MinRayLength && rayLength < _MaxRayLength && rayStep < _MaxLoop; rayStep++) {
    d = map((rayOrigin + rayDir * rayLength) * rcpScales);
    rayLength += d * marchingFactor;
}

if (d >= _MinRayLength) {
    discard;
}
```

### Tetrahedron techniqueによる法線算出

法線の導出は[Tetrahedron technique](https://iquilezles.org/articles/normalsSDF/ "Inigo Quilez :: computer graphics, mathematics, shaders, fractals, demoscene and more")を用いて，距離関数の評価が4回で済むようにした．
元記事の導出過程はやや飛ばし気味だが，もう少し丁寧に書くと下記のようになる（折り畳み部分）．

:::details Tetrahedron techniqueの導出過程
まず，陰関数 $\boldsymbol{f}$ （ $f: \mathbb{R}^3 \mapsto \mathbb{R}$ ）の点 $\boldsymbol{p}$ における正規化法線ベクトル $\boldsymbol{n}$ は下記のとおり．

$$
\boldsymbol{n} = normalize(\nabla f( \boldsymbol{p} ))
= normalize \begin{pmatrix}
\dfrac{\partial f(\boldsymbol{p})}{\partial x}
\\
\dfrac{\partial f(\boldsymbol{p})}{\partial y}
\\
\dfrac{\partial f(\boldsymbol{p})}{\partial z}
\end{pmatrix}  \\
$$

ここで下記の4つのベクトルを設ける．

$$
\begin{cases}
\boldsymbol{k}_0 & = & \begin{pmatrix}1 & -1 & -1\end{pmatrix}^T  \\
\boldsymbol{k}_1 & = & \begin{pmatrix}-1 & -1 & 1\end{pmatrix}^T  \\
\boldsymbol{k}_2 & = & \begin{pmatrix}-1 & 1 & -1\end{pmatrix}^T  \\
\boldsymbol{k}_3 & = & \begin{pmatrix}1 & 1 & 1\end{pmatrix}^T
\end{cases}
$$

この4つのベクトルを用いると下記のように打ち消しが発生し，ゼロベクトルとなる．

$$
\sum_i \boldsymbol{k}_i f(\boldsymbol{p}) =
\begin{pmatrix}
f(\boldsymbol{p})_x  \\
-f(\boldsymbol{p})_y  \\
-f(\boldsymbol{p})_z
\end{pmatrix} +
\begin{pmatrix}
-f(\boldsymbol{p})_x  \\
f(\boldsymbol{p})_y  \\
-f(\boldsymbol{p})_z
\end{pmatrix} +
\begin{pmatrix}
-f(\boldsymbol{p})_x  \\
-f(\boldsymbol{p})_y  \\
f(\boldsymbol{p})_z
\end{pmatrix} +
\begin{pmatrix}
f(\boldsymbol{p})_x  \\
f(\boldsymbol{p})_y  \\
f(\boldsymbol{p})_z
\end{pmatrix}
= \boldsymbol{0}
$$

従って， $h$ を微小な値とすると，

$$
\begin{align}
\boldsymbol{m} &= \sum_i \boldsymbol{k}_i \dfrac{f(\boldsymbol{p} + h \boldsymbol{k}_i)}{h}  \\
&= \sum_i \boldsymbol{k}_i \dfrac{f(\boldsymbol{p} + h \boldsymbol{k}_i)}{h} - \boldsymbol{0} \\
&= \sum_i \boldsymbol{k}_i \dfrac{f(\boldsymbol{p} + h \boldsymbol{k}_i)}{h} - \sum_i \boldsymbol{k}_i \dfrac{f(\boldsymbol{p})}{h} \\
&= \sum_i \left( \boldsymbol{k}_i \dfrac{f(\boldsymbol{p} + h \boldsymbol{k}_i)}{h} - \boldsymbol{k}_i \dfrac{f(\boldsymbol{p})}{h} \right) \\
&= \sum_i \boldsymbol{k}_i \dfrac{f(\boldsymbol{p} + h \boldsymbol{k}_i) - f(\boldsymbol{p})}{h}  \\
&= \sum_i \boldsymbol{k}_i \nabla_{\boldsymbol{k}_i} f(\boldsymbol{p})  \\
&= \sum_i \boldsymbol{k}_i (\boldsymbol{k}_i \cdot \nabla f(\boldsymbol{p}))
\end{align}
$$

となる．

ここで， $\boldsymbol{m}$ の成分 $x$ だけに注目すると，以下を得る．

$$
\begin{align}
m_x &= \sum_i k_{ix} (\boldsymbol{k}_i \cdot \nabla f(\boldsymbol{p}))  \\
&= \sum_i \left( (k_{ix} \boldsymbol{k}_i) \cdot \nabla f(\boldsymbol{p}) \right)  \\
&= \nabla f(\boldsymbol{p}) \cdot \sum_i k_{ix} \boldsymbol{k}_i  \\
&= \nabla f(\boldsymbol{p}) \cdot \begin{pmatrix} 4 & 0 & 0 \end{pmatrix}^T  \\
&= 4 \dfrac{\partial f(\boldsymbol{p})}{\partial x}
\end{align}
$$

$y$, $z$ に関しても同様であるため，最終的に $\boldsymbol{m}$ は下記のようになる．

$$
\begin{align}
\boldsymbol{m} &= \begin{pmatrix}
m_x & m_y & m_z
\end{pmatrix}^T  \\
&= \begin{pmatrix}
4 \dfrac{\partial f(\boldsymbol{p})}{\partial x} & 4 \dfrac{\partial f(\boldsymbol{p})}{\partial y} & 4 \dfrac{\partial f(\boldsymbol{p})}{\partial z}
\end{pmatrix}^T  \\
&= 4 \begin{pmatrix}
\dfrac{\partial f(\boldsymbol{p})}{\partial x} & \dfrac{\partial f(\boldsymbol{p})}{\partial y} & \dfrac{\partial f(\boldsymbol{p})}{\partial z}
\end{pmatrix}^T  \\
&= 4 \nabla f(\boldsymbol{p})
\end{align}
$$

正規化を行うなら定数倍は無視できるので，正規化法線ベクトル $\boldsymbol{n}$ は

$$
\begin{align}
\boldsymbol{n}
&= normalize \left( \nabla f(\boldsymbol{p}) \right)  \\
&= normalize \left( \dfrac{1}{4} \boldsymbol{m} \right)  \\
&= normalize \left( \sum_i \boldsymbol{k}_i \dfrac{f(\boldsymbol{p} + h \boldsymbol{k}_i)}{4h} \right) \\
&= normalize \left( \sum_i \boldsymbol{k}_i f(\boldsymbol{p} + h \boldsymbol{k}_i) \right)
\end{align}
$$

となる．
:::

プログラムに落とし込むと下記のようになる．

```hlsl
/*!
 * @brief Calculate normal of the objects with tetrahedron technique.
 * @param [in] p  Position of the tip of the ray.
 * @return Normal of the objects.
 * @see https://iquilezles.org/articles/normalsSDF/
 */
float3 calcNormal(float3 p)
{
    static const float2 k = float2(1.0, -1.0);
    static const float3 ks[] = {k.xyy, k.yxy, k.yyx, k.xxx};
    static const float h = 0.0001;

    const float3 rcpScales = rcp(_Scales);
    float3 normal = float3(0.0, 0.0, 0.0);

    for (int i = 0; i < 4; i++) {
        normal += ks[i] * map((p + ks[i] * h) * rcpScales);
    }

    return normalize(normal);
}
```

多くの例だとループを用いずに書かれているが，あえてループの形としたのは下記2点の理由による．

- ループが行われるコードの方がコードサイズは小さく，コンパイル時間も短い
- 手動でアンループされているコードからループするコードへのコンパイルはできないが，ループを用いたコードからアンロールしたコードへのコンパイルは `[unroll]` の指定により容易にできる

アンロールに関して，例えば下記2つのどちらも同じコード生成がされる．

```hlsl
/*!
 * @brief Calculate normal of the objects with tetrahedron technique.
 * @param [in] p  Position of the tip of the ray.
 * @return Normal of the objects.
 * @see https://iquilezles.org/articles/normalsSDF/
 */
float3 calcNormal(float3 p)
{
    static const float h = 0.0001;  // replace by an appropriate value
    static const float2 k = float2(1.0, -1.0);
    static const float2 ks = h * k;

    const float3 rcpScales = rcp(_Scales);

    return normalize(
        k.xyy * map((p + ks.xyy) * rcpScales)
            + k.yxy * map((p + ks.yxy) * rcpScales)
            + k.yyx * map((p + ks.yyx) * rcpScales)
            + map((p + ks.xxx) * rcpScales).xxx);
}
```

```hlsl
/*!
 * @brief Calculate normal of the objects with tetrahedron technique.
 * @param [in] p  Position of the tip of the ray.
 * @return Normal of the objects.
 * @see https://iquilezles.org/articles/normalsSDF/
 */
float3 calcNormal(float3 p)
{
    static const float2 k = float2(1.0, -1.0);
    static const float3 ks[] = {k.xyy, k.yxy, k.yyx, k.xxx};
    static const float h = 0.0001;

    const float3 rcpScales = rcp(_Scales);
    float3 normal = float3(0.0, 0.0, 0.0);

    [unroll]  // <- unroll lopp!!
    for (int i = 0; i < 4; i++) {
        normal += ks[i] * map((p + ks[i] * h) * rcpScales);
    }

    return normalize(normal);
}
```

アンロールの指定がない場合，ループ内のコードが十分に小さければアンロールされる．
例えば， `map()` が単純な球の距離関数であれば，無指定のfor文であってもアンロールされた結果となる．

### pragma target 3.0 と環境光

[VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】](https://butadiene.booth.pm/items/1829988 "VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】")ではpragma targetの指定はなかったが，本記事のすべてのコードで `#pragma target 3.0` を指定している．
これは[指定がなければ `#pragma target 2.5` が指定されているのと同様になる](https://docs.unity3d.com/2022.3/Documentation/Manual/SL-ShaderCompileTargets.html "Unity - Manual: Targeting shader models and GPU features in HLSL")が，これだと環境光を頂点単位で行う設定であるため， `ShaderSHPerPixel()` が何も行わなくなり，レイマーチングでのライティングに不都合なためである．

下記はHalf-LambertとBlinn-Phongによるライティング関数だが， `ShadeSHPerPixel()` が機能しなければ描画結果が大きく異なる．

```hlsl
/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLighting(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    const float3 worldViewDir = normalize(_WorldSpaceCameraPos - worldPos);
#if defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    const float3 worldLightDir = UnityWorldSpaceLightDir(worldPos);
#else
    const float3 worldLightDir = normalize(UnityWorldSpaceLightDir(worldPos));
#endif  // defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    const fixed3 lightCol = _LightColor0.rgb * atten;

    // Lambertian reflectance.
    const float nDotL = dot(worldNormal, worldLightDir);
    const half3 diffuse = lightCol * pow(nDotL * 0.5 + 0.5, 2.0);  // will be mul instruction.

    // Specular reflection.
    const float nDotH = dot(worldNormal, normalize(worldLightDir + worldViewDir));
    const half3 specular = pow(max(0.0, nDotH), _SpecPower) * _SpecColor.rgb * lightCol;

    // Ambient color.
#if UNITY_SHOULD_SAMPLE_SH
    ambient = ShadeSHPerPixel(worldNormal, ambient, worldPos);
#endif  // UNITY_SHOULD_SAMPLE_SH

    const half4 outColor = half4((diffuse + ambient) * color.rgb + specular, color.a);

    return outColor;
}
```

#### Directional Lightあり状況下での環境光の有無比較

環境光の処理があれば，全体的に明るめになる．

| 環境光あり | 環境光なし |
|:-:|:-:|
| [![環境光あり](/images/koturn-unity-sphere-tracing-tips/AmbientWithDirectionalLight.png)](/images/koturn-unity-sphere-tracing-tips/AmbientWithDirectionalLight.png "環境光あり") | [![環境光なし](/images/koturn-unity-sphere-tracing-tips/NoAmbientWithDirectionalLight.png)](/images/koturn-unity-sphere-tracing-tips/AmbientWithDirectionalLight.png "環境光なし") |

#### Directional Lightなし状況下での環境光の有無比較

環境光の処理があれば，Directional Lightが存在しない状況下でも明るさを確保できる．
一方で，環境光の処理がなければ真っ黒な描画結果となってしまう．

VRChatにはDirectional Lightが存在しないワールドもあるため，Directional Lightが存在しない状況は考慮すべきである．

| 環境光あり | 環境光なし |
|:-:|:-:|
| [![環境光あり](/images/koturn-unity-sphere-tracing-tips/AmbientWithoutDirectionalLight.png)](/images/koturn-unity-sphere-tracing-tips/AmbientWithoutDirectionalLight.png "環境光あり") | [![環境光なし](/images/koturn-unity-sphere-tracing-tips/NoAmbientWithoutDirectionalLight.png)](/images/koturn-unity-sphere-tracing-tips/AmbientWithoutDirectionalLight.png "環境光なし") |

### SPS-I

シェーダーのSPS-Iへの対応は [lilxyzw/Shader-MEMO](https://github.com/lilxyzw/Shader-MEMO/blob/main/Assets/SPSITest.shader "Shader-MEMO/Assets/SPSITest.shader at main · lilxyzw/Shader-MEMO") を参考に行った．
端的には下記の部分の構造体へのメンバ追加と転送，初期化処理が該当し，これがあるだけでSPS-Iに対応したことになる．
詳細はそれぞれのマクロの定義を参照すること．

```hlsl
        struct appdata
        {
            /* ---------- 略 ---------- */

            // Instance ID for single pass instanced rendering, `uint instanceID : SV_InstanceID`.
            UNITY_VERTEX_INPUT_INSTANCE_ID
        };

        /* ---------- 略 ---------- */

        struct v2f
        {
            /* ---------- 略 ---------- */

            // Instance ID for single pass instanced rendering, `uint instanceID : SV_InstanceID`.
            UNITY_VERTEX_INPUT_INSTANCE_ID
            // Stereo target eye index for single pass instanced rendering, `stereoTargetEyeIndex` and `stereoTargetEyeIndexSV`.
            UNITY_VERTEX_OUTPUT_STEREO
        }

        v2f vert(appdata v)
        {
            /* ---------- 略 ---------- */

            UNITY_SETUP_INSTANCE_ID(v);
            UNITY_TRANSFER_INSTANCE_ID(v, o);
            UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);

            /* ---------- 略 ---------- */
        }

        /* ---------- 略 ---------- */

        fout frag(v2f fi)
        {
            UNITY_SETUP_INSTANCE_ID(fi);
            UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(fi);

            /* ---------- 略 ---------- */
        }
```


### Tagsの設定

Tagsは下記の指定を行っている．それぞれの解説を行う．

```shaderlab
Tags
{
    "Queue" = "AlphaTest"
    // "RenderType" = "Transparent"
    "DisableBatching" = "True"
    "IgnoreProjector" = "True"
    "VRCFallback" = "Hidden"
}
```

#### Queue

本記事では通常のスフィアトレーシングによる不透明オブジェクトの描画を行うことを前提としている．
`discard` を行うため， `"Queue" = "AlphaTest"` (2450) の指定が妥当と考えている．

#### RenderType

Post Processing の Ambient Occlusion に Scalable Ambient Obscurance (SAO) を用いている場合，[下敷きにしているメッシュに沿って誤った影が描画](https://tips.hecomi.com/entry/2019/01/27/233137#%E3%83%95%E3%82%A9%E3%83%AF%E3%83%BC%E3%83%89%E3%81%A7-DepthNormalTexture-%E3%82%92%E4%BD%BF%E3%81%86%E5%A0%B4%E5%90%88 "Unity で距離関数の記述だけでレイマーチングができる uRaymarching を Forward / XR 対応した - 凹みTips")される．
Multi Scale Volumetric Obscurance (MSVO) であれば問題は発生しないのだが，VRChatのようなプラットフォームにおけるアバターにレイマーチングのオブジェクトを仕込む場合，SAOが採用されているワールドに遭遇する可能性がある．

この問題を回避するには，下記2つのどちらかの手段を取るとよい．

- RenderTypeタグを記述しない
- `"RenderType" = "Transparent"`

#### DisableBatching

同じレイマーチングシェーダーのマテリアルを持つオブジェクトが2つ以上ある場合，バッチングが行われ，ローカル座標が取得できなくなる現象が発生する．
これを防ぐために， `"DisableBatching" = "True"` を指定する．

ただし，GPU Instancingに対応させている場合，この指定は不要である．
SPS-I対応を行っていることにより，GPUインスタンシングにも対応したコードとなっているので， `#pragma multi_compile_instancing` を記載し，インスペクタで有効にすれば，バッチングの問題は解消できる．

#### IgnoreProjector

プロジェクタでの投影はマテリアルの差し替えによって行われる．
しかし，レイマーチングは基本的にメッシュを無視して描画するものであるため，対策なしだと下敷きにしているメッシュの形状に沿って投影が行われてしまう．
プロジェクタの投影を無効にするために `"IgnoreProjector" = "True"` を指定することをオススメする．

#### VRCFallback

レイマーチングは基本的にメッシュに沿わない描画を行うものなので，フォールバック可能なシェーダーはない．
そのため， `"VRCFallback" = "Hidden"` を指定し，シェーダーブロックされている場合は非表示となるように設定しておくことをオススメする．

### フラグメントシェーダーでの深度出力

`SV_Depth` セマンティクスによりフラグメントシェーダーでの深度出力が可能となるが，これはシェーダーキーワードにより切り替え可能とした．

```shaderlab
[Toggle(_SVDEPTH_ON)]
_SVDepth ("SV_Depth ouput", Int) = 1
```

```hlsl
#pragma shader_feature_local_fragment _ _SVDEPTH_ON
```

また，[VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】](https://butadiene.booth.pm/items/1829988 "VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】")のコード中には記載がないが，OpenGL系だとクリッピング座標の深度値がDirectX系と異なっており， `SV_Depth` として出力する深度値として 0.0～1.0 の範囲へ変換が必要となる．

| 種別 | Near | Far |
|-|-|-|
| DirectX | 1.0 | 0.0 |
| OpenGL | -1.0 | 1.0 |

コード中の下記の関数が該当する．

```hlsl
/*!
 * @brief Get depth from clip space position.
 * @param [in] clipPos  Clip space position.
 * @return Depth value.
 */
float getDepth(float4 clipPos)
{
    const float depth = clipPos.z / clipPos.w;
    UNITY_REVERSE_Z
#if defined(SHADER_API_GLCORE) \
    || defined(SHADER_API_OPENGL) \
    || defined(SHADER_API_GLES) \
    || defined(SHADER_API_GLES3)
    // [-1.0, 1.0] -> [0.0, 1.0]
    // Near: -1.0
    // Far: 1.0
    return depth * 0.5 + 0.5;
#else
    // [0.0, 1.0] -> [0.0, 1.0] (No conversion)
    // Near: 1.0
    // Far: 0.0
    return depth;
#endif
}
```

この判定には `HLSLSuport.cginc` で定義されている `UNITY_REVERSED_Z` を用いてもよい．

```hlsl
/*!
 * @brief Get depth from clip space position.
 * @param [in] clipPos  Clip space position.
 * @return Depth value.
 */
float getDepth(float4 clipPos)
{
    const float depth = clipPos.z / clipPos.w;
#if defined(UNITY_REVERSED_Z)
    // [0.0, 1.0] -> [0.0, 1.0] (No conversion)
    // Near: 1.0
    // Far: 0.0
    return depth;
#else
    // [-1.0, 1.0] -> [0.0, 1.0]
    // Near: -1.0
    // Far: -1.0
    return depth * 0.5 + 0.5;
#endif  // defined(UNITY_REVERSED_Z)
}
```

---

## 投影先オブジェクト

### Quad

UnityのデフォルトのQuadでも問題はない．
ただし，UV，法線，接平面は必要ないため，それらを含まないメッシュを用意してもよいかもしれない．

### Cube

UnityのデフォルトキューブはUVや法線を考慮して24頂点12ポリゴンとなっているが，レイマーチングの投影を行うCubeとしてはUV，法線，接平面は必要ではないため8頂点12ポリゴンのキューブで十分であるし，頂点情報にUV，法線，接平面を含めなくてよい．
（デフォルトキューブはそれぞれの面の法線を一様にするために24頂点必要となっている）

スフィアトレーシングでレイの進行の反復回数が多くなるのは，レイがある程度描画オブジェクトに漸近しつつも結局は外れている場合である．
Cubeの内側にのみオブジェクトを描画するという前提があるなら，描画オブジェクトを完全に被覆する必要最小限のサイズのCube（直方体でもよい）を用意すると多少の処理負荷の軽減につながると思われる．

この程度のメッシュであれば[Unityのスクリプトで作成することが可能](https://github.com/koturn/SphereTracingTips/tree/main/Assets/koturn/SphereTracingTips/Editor "Assets/koturn/SphereTracingTips/Editor")である．

---

## ForwardBase以外のレンダリングパスへの対応

サンプルコード: [Assets/koturn/SphereTracingTips/02_Pass/Shaders/BaseAddShadow.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/02_Pass/Shaders/BaseAddShadow.shader "Assets/koturn/SphereTracingTips/02_Pass/Shaders/BaseAddShadow.shader")

BRPにおけるレンダリングパスにはForward Base以外にもForward AddとShadow Casterがある．
本章ではこの2つのレンダリングパスの実装について述べる．

### ForwardAdd Pass

Forward Addパスの実装は簡単で，Forward Baseの `vert()` と `frag()` を使い回す形でよい．
Tagsに `"LightMode" = "ForwardAdd"` を指定するのを忘れないこと．
また，各種キーワードへの対応は

- `#pragma multi_compile_fwdadd`
- `#pragma multi_compile_fwdadd_fullshadows`

のどちらかを指定する必要があるが，機能の多い後者を指定することにした．

```hlsl
Pass
{
    Name "FORWARD_ADD"
    Tags
    {
        "LightMode" = "ForwardAdd"
    }

    Blend [_SrcBlend] One
    ZWrite Off

    CGPROGRAM
    // #pragma multi_compile_fwdadd
    #pragma multi_compile_fwdadd_fullshadows
    #pragma multi_compile_fog

    #pragma vertex vert
    #pragma fragment frag
    ENDCG
}
```

ForwardAddパスでは `UNITY_PASS_FORWARDADD` マクロが定義されるが， `UNITY_SHOULD_SAMPLE_SH` の定義が下記の通りであるため，環境光がForwardBase分も含めて二重に計上されるわけではない．

```hlsl
#define UNITY_SHOULD_SAMPLE_SH (defined(LIGHTPROBE_SH) && !defined(UNITY_PASS_FORWARDADD) && !defined(UNITY_PASS_PREPASSBASE) && !defined(UNITY_PASS_SHADOWCASTER) && !defined(UNITY_PASS_META))
```

### ShadowCaster Pass

ShadowCasterパスについては[hecomiさんの記事](https://tips.hecomi.com/entry/2019/01/27/233137#DepthTexture-%E3%81%AE%E7%94%9F%E6%88%90 "Unity で距離関数の記述だけでレイマーチングができる uRaymarching を Forward / XR 対応した - 凹みTips")の内容そのままである．
ただ，レイの正規化前の方向ベクトルについては頂点シェーダーで計算可能なため，フラグメントシェーダーでは計算しないようにしている（条件分岐がある，また，オブジェクト空間でのレイマーチングだと行列演算も加わり，ややコスト高でもあるため）．

```hlsl
Pass
{
    Name "SHADOW_CASTER"
    Tags
    {
        "LightMode" = "ShadowCaster"
    }

    Blend Off
    ZWrite On

    CGPROGRAM
    #pragma multi_compile_shadowcaster

    #pragma vertex vertShadowCaster
    #pragma fragment fragShadowCaster


    /*!
     * @brief Input of the vertex shader, vertShadowCaster().
     * @see vertShadowCaster
     */
    struct appdata_shadowcaster
    {
        //! Object space position of the vertex.
        float4 vertex : POSITION;
        // instanceID for single pass instanced rendering.
        UNITY_VERTEX_INPUT_INSTANCE_ID
    };

    /*!
     * @brief Output of the vertex shader, vertShadowCaster()
     * and input of fragment shader, fragShadowCaster().
     * @see vertShadowCaster
     * @see fragShadowCaster
     */
    struct v2f_shadowcaster
    {
        // V2F_SHADOW_CASTER;
        // `float3 vec : TEXCOORD0;` is unnecessary even if `!defined(SHADOWS_CUBE) || defined(SHADOWS_CUBE_IN_DEPTH_TEX)`
        // because calculate `vec` in fragment shader.

        //! Clip space position of the vertex.
        float4 pos : SV_POSITION;
        //! Ray origin in object/world space
        float3 rayOrigin : TEXCOORD0;
        //! Unnormalized ray direction in object/world space.
        float3 rayDirVec : TEXCOORD1;
        // instanceID for single pass instanced rendering.
        UNITY_VERTEX_INPUT_INSTANCE_ID
        // stereoTargetEyeIndex for single pass instanced rendering.
        UNITY_VERTEX_OUTPUT_STEREO
    };

    #if defined(SHADER_API_GLCORE) || defined(SHADER_API_GLES) || defined(SHADER_API_D3D9)
    typedef fixed face_t;
    #    define FACE_SEMANTICS VFACE
    #else
    typedef bool face_t;
    #    define FACE_SEMANTICS SV_IsFrontFace
    #endif  // defined(SHADER_API_GLCORE) || defined(SHADER_API_GLES) || defined(SHADER_API_D3D9)


    float3 getCameraDirVec(float4 screenPos);
    bool isFacing(face_t facing);


    /*!
     * @brief Vertex shader function for ShadowCaster pass.
     * @param [in] v  Input data.
     * @return Interpolation source data for fragment shader function, fragShadowCaster().
     * @see fragShadowCaster
     */
    v2f_shadowcaster vertShadowCaster(appdata_shadowcaster v)
    {
        v2f_shadowcaster o;
        UNITY_INITIALIZE_OUTPUT(v2f_shadowcaster, o);

        UNITY_SETUP_INSTANCE_ID(v);
        UNITY_TRANSFER_INSTANCE_ID(v, o);
        UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);

        //
        // TRANSFER_SHADOW_CASTER(o)
        //
        o.pos = UnityObjectToClipPos(v.vertex);
    #if !defined(SHADOWS_CUBE) || defined(SHADOWS_CUBE_IN_DEPTH_TEX)
        o.pos = UnityApplyLinearShadowBias(o.pos);
    #endif  // !defined(SHADOWS_CUBE) || defined(SHADOWS_CUBE_IN_DEPTH_TEX)

        float4 screenPos = ComputeNonStereoScreenPos(o.pos);
        COMPUTE_EYEDEPTH(screenPos.z);

    #if defined(_CALCSPACE_WORLD)
        o.rayOrigin = mul(unity_ObjectToWorld, v.vertex).xyz;
    #    if defined(SHADOWS_CUBE) && !defined(SHADOWS_CUBE_IN_DEPTH_TEX)
        o.rayDirVec = getCameraDirVec(screenPos);
    #    else
        if (UNITY_MATRIX_P[3][3] == 1.0) {
            // For directional light.
            o.rayDirVec = -UNITY_MATRIX_V[2].xyz;
        } else if (abs(unity_LightShadowBias.x) < 1.0e-5) {
            // For depth output of camera.
            o.rayDirVec = o.rayOrigin - _WorldSpaceCameraPos.xyz;
        } else {
            // For spot light.
            o.rayDirVec = getCameraDirVec(screenPos);
        }
    #    endif
    #else
        o.rayOrigin = v.vertex.xyz;
    #    if defined(SHADOWS_CUBE) && !defined(SHADOWS_CUBE_IN_DEPTH_TEX)
        o.rayDirVec = mul((float3x3)unity_WorldToObject, getCameraDirVec(screenPos));
    #    else
        if (UNITY_MATRIX_P[3][3] == 1.0) {
            // For directional light.
            o.rayDirVec = mul((float3x3)unity_WorldToObject, -UNITY_MATRIX_V[2].xyz);
        } else if (abs(unity_LightShadowBias.x) < 1.0e-5) {
            // For depth output of camera.
            o.rayDirVec = o.rayOrigin - mul(unity_WorldToObject, float4(_WorldSpaceCameraPos, 1.0)).xyz;
        } else {
            // For spot light.
            o.rayDirVec = mul((float3x3)unity_WorldToObject, getCameraDirVec(screenPos));
        }
    #    endif
    #endif  // defined(_CALCSPACE_WORLD)

        return o;
    }

    /*!
     * @brief Fragment shader function for ShadowCaster pass.
     * @param [in] fi  Input data from vertex shader.
     * @param [in] facing  Facing parameter.
     * @return Depth of fragment.
     */
    fout fragShadowCaster(v2f_shadowcaster fi, face_t facing : FACE_SEMANTICS)
    {
        UNITY_SETUP_INSTANCE_ID(fi);
        UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(fi);

        const float3 rayOrigin = fi.rayOrigin;
        const float3 rayDir = normalize(isFacing(facing) ? fi.rayDirVec : -fi.rayDirVec);

        const float3 rcpScales = rcp(_Scales);
        const float marchingFactor = _MarchingFactor * rsqrt(dot(rayDir * rcpScales, rayDir * rcpScales));

        float rayLength = 0.0;
        float d = asfloat(0x7f800000);  // +inf
        for (int rayStep = 0; d >= _MinRayLength && rayLength < _MaxRayLength && rayStep < _MaxLoopShadowCaster; rayStep++) {
            d = map((rayOrigin + rayDir * rayLength) * rcpScales);
            rayLength += d * marchingFactor;
        }

        if (d >= _MinRayLength) {
            discard;
        }

    #if defined(_CALCSPACE_WORLD)
        const float3 worldFinalPos = rayOrigin + rayDir * rayLength;
    #else
        const float3 localFinalPos = rayOrigin + rayDir * rayLength;
        const float3 worldFinalPos = mul(unity_ObjectToWorld, float4(localFinalPos, 1.0)).xyz;
    #endif  // defined(_CALCSPACE_WORLD)

    #if defined(SHADOWS_CUBE) && !defined(SHADOWS_CUBE_IN_DEPTH_TEX)
        //
        // TRANSFER_SHADOW_CASTER_NORMALOFFSET
        //
        const float3 vec = worldFinalPos - _LightPositionRange.xyz;

        //
        // SHADOW_CASTER_FRAGMENT
        //
        fout fo;
        fo.color = UnityEncodeCubeShadowDepth((length(vec) + unity_LightShadowBias.x) * _LightPositionRange.w);
        return fo;
    #else
        //
        // TRANSFER_SHADOW_CASTER_NORMALOFFSET
        //
        float3 worldPos = worldFinalPos;
        if (unity_LightShadowBias.z != 0.0) {
    #    if defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
            const float3 worldLightDir = UnityWorldSpaceLightDir(worldPos);
    #    else
            const float3 worldLightDir = normalize(UnityWorldSpaceLightDir(worldPos));
    #    endif  // defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    #    if defined(_CALCSPACE_WORLD)
            const float3 worldNormal = calcNormal(worldFinalPos);
    #    else
            const float3 worldNormal = UnityObjectToWorldNormal(calcNormal(localFinalPos));
    #    endif  // defined(_CALCSPACE_WORLD)
            const float shadowCos = dot(worldNormal, worldLightDir);
            const float shadowSine = sqrt(1.0 - shadowCos * shadowCos);
            const float normalBias = unity_LightShadowBias.z * shadowSine;
            worldPos.xyz -= worldNormal * normalBias;
        }
        const float4 clipPos = UnityApplyLinearShadowBias(UnityWorldToClipPos(worldPos));

        //
        // SHADOW_CASTER_FRAGMENT
        //
        fout fo;
        fo.color = float4(0.0, 0.0, 0.0, 0.0);
    #    if defined(_SVDEPTH_ON)
        fo.depth = getDepth(clipPos);
    #    endif  // defined(_SVDEPTH_ON)
        return fo;
    #endif  // defined(SHADOWS_CUBE) && !defined(SHADOWS_CUBE_IN_DEPTH_TEX)
    }

    /*!
     * @brief Get unnormalized camera direction vector from screen space position.
     * @param [in] Screen space position.
     * @return Camera direction in world space.
     */
    float3 getCameraDirVec(float4 screenPos)
    {
        float2 sp = (screenPos.xy / screenPos.w) * 2.0 - 1.0;

        // Following code is equivalent to: sp.x *= _ScreenParams.x / _ScreenParams.y;
        sp.x *= _ScreenParams.x * _ScreenParams.w - _ScreenParams.x;

        return UNITY_MATRIX_V[0].xyz * sp.x
            + UNITY_MATRIX_V[1].xyz * sp.y
            + -UNITY_MATRIX_V[2].xyz * abs(UNITY_MATRIX_P[1][1]);
    }

    /*!
     * @brief Identify whether surface is facing the camera or facing away from the camera.
     * @param [in] facing  Facing variable (fixed or bool).
     * @return True if surface facing the camera, otherwise false.
     */
    bool isFacing(face_t facing)
    {
    #if defined(SHADER_API_GLCORE) || defined(SHADER_API_GLES) || defined(SHADER_API_D3D9)
        return facing >= 0.0;
    #else
        return facing;
    #endif  // defined(SHADER_API_GLCORE) || defined(SHADER_API_GLES) || defined(SHADER_API_D3D9)
    }
    ENDCG
}
```

メッシュの裏表によってレイの方向を反転させなければ， `Cull Front` （表面のポリゴンをカリング）を指定したとき，下記表左の画像のように影がうまく描画されない（画像中では小さな点3つとなっている）．

| レイ方向反転なし | レイ方向反転あり |
|-|-|
| [![ShadowCaster Pass，Cull Front指定でレイを反転させない場合](/images/koturn-unity-sphere-tracing-tips/ShadowCasterCullFrontNonInvertRayDirection.png)](/images/koturn-unity-sphere-tracing-tips/ShadowCasterCullFrontNonInvertRayDirection.png "ShadowCaster Pass，Cull Front指定でレイを反転させない場合") | [![ShadowCaster Pass，Cull Front指定でレイを反転させる場合](/images/koturn-unity-sphere-tracing-tips/ShadowCasterCullFrontInvertRayDirection.png)](/images/koturn-unity-sphere-tracing-tips/ShadowCasterCullFrontInvertRayDirection.png "ShadowCaster Pass，Cull Front指定でレイを反転させる場合") |

また，レイを反転させたとしても `Cull Front` だと，下記表右の画像のようにポリゴンメッシュとの接触部分の影の描画に少し問題がある．

| `Cull Back` | `Cull Front` |
|-|-|
| [![ShadowCaster Pass，Cull Backでポリゴンメッシュと接触](/images/koturn-unity-sphere-tracing-tips/ShadowCasterCullBackIntersect.png)](/images/koturn-unity-sphere-tracing-tips/ShadowCasterCullBackIntersect.png "ShadowCaster Pass，Cull Backでポリゴンメッシュと接触") | [![ShadowCaster Pass，Cull Frontでポリゴンメッシュと接触](/images/koturn-unity-sphere-tracing-tips/ShadowCasterCullFrontIntersect.png)](/images/koturn-unity-sphere-tracing-tips/ShadowCasterCullFrontIntersect.png "ShadowCaster Pass，Cull Frontでポリゴンメッシュと接触") |


---

## Builtin Render Pipelineの標準ライブラリを用いたライティング

本記事でベースにするシェーダーはLambert反射モデルによる拡散反射とBlinn-Phong反射モデルによる鏡面反射のライティングを自前実装していた．
しかし，ライティングの実装は面倒であり，何とかして楽をしたいものである．
この章ではBRPの標準ライブラリで利用可能なライティング関数を用いて，ライティング実装の手間を削減する方法について述べる．

ここで紹介するのはUnityの標準ライブラリで提供されている4つのライティング関数

- Lambert
- Blinn-Phong
- Standard
- Standard Specular

を用いた例である．
下記の画像は左から順にLambert, Blinn-Phong, Standard, Standard Specularの例となっている．

[![UnityLighting.png](/images/koturn-unity-sphere-tracing-tips/UnityLighting.png)](/images/koturn-unity-sphere-tracing-tips/UnityLighting.png "UnityLighting.png")

### Lambert

サンプルコード: [Assets/koturn/SphereTracingTips/03_Lighting/Shaders/Lambert.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/03_Lighting/Shaders/Lambert.shader "Assets/koturn/SphereTracingTips/03_Lighting/Shaders/Lambert.shader")

Lambert程度であれば自前実装してもよいのだが，ライブラリとして提供されているため，使用する例を示す．

```hlsl
/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLighting(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    SurfaceOutput so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutput, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = fixed3(0.0, 0.0, 0.0);
    // so.Specular = 0.0;
    // so.Gloss = 0.0;
    so.Alpha = color.a;

    UnityGI gi = getGI(worldPos, atten);
    const float3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
#if defined(UNITY_PASS_FORWARDBASE)
    const float4 lmap = float4(0.0, 0.0, 0.0, 0.0);
    UnityGIInput giInput = getGIInput(gi.light, worldPos, worldNormal, worldViewDir, atten, lmap, ambient);
    LightingLambert_GI(so, giInput, /* inout */ gi);
#endif  // defined(UNITY_PASS_FORWARDBASE)

    half4 col = LightingLambert(so, gi);
#if defined(UNITY_PASS_FORWARDBASE)
    col.rgb += so.Emission;
#endif  // defined(UNITY_PASS_FORWARDBASE)

    return col;
}
```

また， `getGI()` と `getGIInput()` の実装は下記のとおり．
これは他の例でも用いる．

```hlsl
/*!
 * @brief Get initial instance of UnityGI.
 * @param [in] worldPos  World coordinate.
 * @param [in] atten  Light attenuation.
 * @return Initial instance of UnityGI.
 */
UnityGI getGI(float3 worldPos, half atten)
{
    UnityGI gi;
    UNITY_INITIALIZE_OUTPUT(UnityGI, gi);

#if defined(UNITY_PASS_FORWARDBASE)
    gi.light.color = _LightColor0.rgb;
#elif defined(UNITY_PASS_DEFERRED)
    gi.light.color = half3(0.0, 0.0, 0.0);
#else
    gi.light.color = _LightColor0.rgb * atten;
#endif  // defined(UNITY_PASS_FORWARDBASE)
#if defined(UNITY_PASS_DEFERRED)
    gi.light.dir = half3(0.0, 1.0, 0.0);
#elif defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    gi.light.dir = UnityWorldSpaceLightDir(worldPos);
#else
    gi.light.dir = normalize(UnityWorldSpaceLightDir(worldPos));
#endif  // defined(UNITY_PASS_DEFERRED)
    // gi.indirect.diffuse = half3(0.0, 0.0, 0.0);
    // gi.indirect.specular = half3(0.0, 0.0, 0.0);

    return gi;
}


/*!
 * @brief Get initial instance of UnityGIInput.
 * @param [in] light  The lighting parameter which contains color and direction of the light.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] worldViewDir  View direction in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] lmap  Light map parameters.
 * @param [in] ambient  Ambient light.
 * @return Initial instance of UnityGIInput.
 */
UnityGIInput getGIInput(UnityLight light, float3 worldPos, float3 worldNormal, float3 worldViewDir, half atten, float4 lmap, half3 ambient)
{
    UnityGIInput giInput;
    UNITY_INITIALIZE_OUTPUT(UnityGIInput, giInput);
    giInput.light = light;
    giInput.worldPos = worldPos;
    giInput.worldViewDir = worldViewDir;
    giInput.atten = atten;

#if defined(LIGHTMAP_ON) || defined(DYNAMICLIGHTMAP_ON)
    giInput.lightmapUV = lmap;
#else
    giInput.lightmapUV = float4(0.0, 0.0, 0.0, 0.0);
#endif  // defined(LIGHTMAP_ON) || defined(DYNAMICLIGHTMAP_ON)

#if UNITY_SHOULD_SAMPLE_SH
    giInput.ambient = ambient;
#else
    giInput.ambient = half3(0.0, 0.0, 0.0);
#endif  // UNITY_SHOULD_SAMPLE_SH

    giInput.probeHDR[0] = unity_SpecCube0_HDR;
    giInput.probeHDR[1] = unity_SpecCube1_HDR;
#if defined(UNITY_SPECCUBE_BLENDING) || defined(UNITY_SPECCUBE_BOX_PROJECTION)
    giInput.boxMin[0] = unity_SpecCube0_BoxMin;
#endif  // defined(UNITY_SPECCUBE_BLENDING) || defined(UNITY_SPECCUBE_BOX_PROJECTION)
#if defined(UNITY_SPECCUBE_BOX_PROJECTION)
    giInput.boxMax[0] = unity_SpecCube0_BoxMax;
    giInput.probePosition[0] = unity_SpecCube0_ProbePosition;
    giInput.boxMax[1] = unity_SpecCube1_BoxMax;
    giInput.boxMin[1] = unity_SpecCube1_BoxMin;
    giInput.probePosition[1] = unity_SpecCube1_ProbePosition;
#endif  // defined(UNITY_SPECCUBE_BOX_PROJECTION)

    return giInput;
}
```

### Blinn Phong

サンプルコード: [Assets/koturn/SphereTracingTips/03_Lighting/Shaders/BlinnPhong.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/03_Lighting/Shaders/BlinnPhong.shader "Assets/koturn/SphereTracingTips/03_Lighting/Shaders/BlinnPhong.shader")

Lambertと同様に `SurfaceOutput` 構造体を用いるが，使用する関数が `LightingBlinnPhong_GI()` と `LightingBlinnPhong()` である点が異なっている．

```hlsl
/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLighting(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    SurfaceOutput so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutput, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = fixed3(0.0, 0.0, 0.0);
    so.Specular = _SpecPower / 128.0;
    so.Gloss = _Glossiness;
    so.Alpha = color.a;

    UnityGI gi = getGI(worldPos, atten);
    const float3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
#if defined(UNITY_PASS_FORWARDBASE)
    const float4 lmap = float4(0.0, 0.0, 0.0, 0.0);
    UnityGIInput giInput = getGIInput(gi.light, worldPos, worldNormal, worldViewDir, atten, lmap, ambient);
    LightingBlinnPhong_GI(so, giInput, /* inout */ gi);
#endif  // defined(UNITY_PASS_FORWARDBASE)

    half4 col = LightingBlinnPhong(so, worldViewDir, gi);
#if defined(UNITY_PASS_FORWARDBASE)
    col.rgb += so.Emission;
#endif  // defined(UNITY_PASS_FORWARDBASE)

    return col;
}
```

### Standard

サンプルコード: [Assets/koturn/SphereTracingTips/03_Lighting/Shaders/Standard.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/03_Lighting/Shaders/Standard.shader "Assets/koturn/SphereTracingTips/03_Lighting/Shaders/Standard.shader")

Standardシェーダー相当のライティングを行うには，標準ライブラリが提供している `LightingStandard_GI()` と `LightingStandard()` を用いるとよい．
そのため `calcLighting()` を下記のように変更する．

```hlsl
/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLighting(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    SurfaceOutputStandard so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutputStandard, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = half3(0.0, 0.0, 0.0);
    so.Metallic = _Metallic;
    so.Smoothness = _Glossiness;
    so.Occlusion = 1.0;
    so.Alpha = color.a;

    UnityGI gi = getGI(worldPos, atten);
    const float3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
#if defined(UNITY_PASS_FORWARDBASE)
    const float4 lmap = float4(0.0, 0.0, 0.0, 0.0);
    UnityGIInput giInput = getGIInput(gi.light, worldPos, worldNormal, worldViewDir, atten, lmap, ambient);
    LightingStandard_GI(so, giInput, /* inout */ gi);
#endif  // defined(UNITY_PASS_FORWARDBASE)

    half4 col = LightingStandard(so, worldViewDir, gi);
#if defined(UNITY_PASS_FORWARDBASE)
    col.rgb += so.Emission;
#endif  // defined(UNITY_PASS_FORWARDBASE)

    return col;
}
```

必要となるプロパティは `_Glossiness`, `_Metallic` の2つである．

`Properties` に下記を，

```shaderlab
_Glossiness ("Smoothness", Range(0.0, 1.0)) = 0.5

[Gamma]
_Metallic ("Metallic", Range(0.0, 1.0)) = 0.0
```

シェーダーコード部分に下記を記述しておく．

```hlsl
//! Value of smoothness.
uniform half _Glossiness;
//! Value of Metallic.
uniform half _Metallic;
```

なお， `_GLOSSYREFLECTIONS_OFF` マクロが定義されている場合，リフレクションプローブの反映を行わない，
`_SPECULARHIGHLIGHTS_OFF` マクロが定義されている場合，鏡面反射を行わないようになっているので，下記の宣言があるとよりStandardシェーダーの機能を具備した形となる．

```shaderlab
[ToggleOff(_SPECULARHIGHLIGHTS_OFF)]
_SpecularHighlights ("Specular Highlights", Int) = 1

[ToggleOff(_GLOSSYREFLECTIONS_OFF)]
_GlossyReflections ("Glossy Reflections", Int) = 1
```

```hlsl
#pragma shader_feature_local_fragment _ _SPECULARHIGHLIGHTS_OFF
#pragma shader_feature_local_fragment _ _GLOSSYREFLECTIONS_OFF
```

### Standard Specular

サンプルコード: [Assets/koturn/SphereTracingTips/03_Lighting/Shaders/StandardSpecular.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/03_Lighting/Shaders/StandardSpecular.shader "Assets/koturn/SphereTracingTips/03_Lighting/Shaders/StandardSpecular.shader")

Standardシェーダー相当のライティングを行うには，標準ライブラリが提供している `LightingStandardSpecular_GI()` と `LightingStandardSpecular()` を用いるとよい．
そのため `calcLighting()` を下記のように変更する．

```hlsl
/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLighting(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    SurfaceOutputStandardSpecular so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutputStandardSpecular, so);
    so.Albedo = color.rgb;
    so.Specular = _SpecColor.rgb;
    so.Normal = worldNormal;
    so.Emission = half3(0.0, 0.0, 0.0);
    so.Smoothness = _Glossiness;
    so.Occlusion = 1.0;
    so.Alpha = color.a;

    UnityGI gi = getGI(worldPos, atten);
    const float3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
#if defined(UNITY_PASS_FORWARDBASE)
    const float4 lmap = float4(0.0, 0.0, 0.0, 0.0);
    UnityGIInput giInput = getGIInput(gi.light, worldPos, worldNormal, worldViewDir, atten, lmap, ambient);
    LightingStandardSpecular_GI(so, giInput, /* inout */ gi);
#endif  // defined(UNITY_PASS_FORWARDBASE)

    half4 col = LightingStandardSpecular(so, worldViewDir, gi);
#if defined(UNITY_PASS_FORWARDBASE)
    col.rgb += so.Emission;
#endif  // defined(UNITY_PASS_FORWARDBASE)

    return col;
}
```

必要となるプロパティは `_SpecColor` と `_Glossiness` の2つである．
Standardと異なり， `_Metallic` は不要である．

`Properties` に下記を，

```shaderlab
_SpecColor ("Specular Color", Color) = (0.5, 0.5, 0.5, 1.0)

_Glossiness ("Smoothness", Range(0.0, 1.0)) = 0.5
```

シェーダーコード部分に下記を記述しておく．
`_SpecColor` は UnityLightingCommon.cginc で宣言されているため不要である．

```hlsl
//! Value of smoothness.
uniform half _Glossiness;
```

### 全部入り

サンプルコード: [Assets/koturn/SphereTracingTips/03_Lighting/Shaders/AllInOne.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/03_Lighting/Shaders/AllInOne.shader "Assets/koturn/SphereTracingTips/03_Lighting/Shaders/AllInOne.shader")

本章で紹介した4つ

- Lambert
- Blinn-Phong
- Standard
- Standard Specular

のいずれもUnityのBRPのシェーダー標準ライブラリが提供するライティング関数のインターフェースはほぼ同じであるため，マクロ定義で差を吸収しやすい．
これら4つのライティングと，記事冒頭のベースにしているシェーダーのHalf-Lambert + Blinn-Phongのライティング，それに加えてUnlitをシェーダーキーワードで切り替え可能にした例を抜粋して示す．

キーワードによってライティングモデルを切り替えたい場合は下記のようになる．
`calcLighting()` をさらに `calcLightingUnity()` と `calcLightingCustom()` に細分化した．

```hlsl
/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLighting(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
#if defined(_LIGHTING_CUSTOM)
    return calcLightingCustom(color, worldPos, worldNormal, atten, ambient);
#elif defined(_LIGHTING_UNITY_LAMBERT) \
    || defined(_LIGHTING_UNITY_BLINN_PHONG) \
    || defined(_LIGHTING_UNITY_STANDARD) \
    || defined(_LIGHTING_UNITY_STANDARD_SPECULAR)
    return calcLightingUnity(color, worldPos, worldNormal, atten, ambient);
#else
    // assume _LIGHTING_UNLIT
    return color;
#endif  // defined(_LIGHTING_CUSTOM)
}

/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLightingUnity(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    // Uniform variable requirements:
    //
    // Variant                             | `_Glossiness` | `_Metallic` | `_SpecColor` | `_SpecPower`
    // ------------------------------------|---------------|-------------|--------------|-------------
    // `_LIGHTING_UNITY_LAMBERT`           |               |             |              |
    // `_LIGHTING_UNITY_BLINN_PHONG`       | o             |             | o            | o
    // `_LIGHTING_UNITY_STANDARD`          | o             | o           |              |
    // `_LIGHTING_UNITY_STANDARD_SPECULAR` | o             |             | o            |

#if defined(_LIGHTING_UNITY_STANDARD)
#    define LightingUnity_GI(so, giInput, gi) LightingStandard_GI(so, giInput, gi)
#    define LightingUnity(so, worldViewDir, gi) LightingStandard(so, worldViewDir, gi)
    SurfaceOutputStandard so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutputStandard, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = half3(0.0, 0.0, 0.0);
    so.Metallic = _Metallic;
    so.Smoothness = _Glossiness;
    so.Occlusion = 1.0;
    so.Alpha = color.a;
#elif defined(_LIGHTING_UNITY_STANDARD_SPECULAR)
#    define LightingUnity_GI(so, giInput, gi) LightingStandardSpecular_GI(so, giInput, gi)
#    define LightingUnity(so, worldViewDir, gi) LightingStandardSpecular(so, worldViewDir, gi)
    SurfaceOutputStandardSpecular so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutputStandardSpecular, so);
    so.Albedo = color.rgb;
    so.Specular = _SpecColor.rgb;
    so.Normal = worldNormal;
    so.Emission = half3(0.0, 0.0, 0.0);
    so.Smoothness = _Glossiness;
    so.Occlusion = 1.0;
    so.Alpha = color.a;
#else
    SurfaceOutput so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutput, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = fixed3(0.0, 0.0, 0.0);
#    if defined(_LIGHTING_UNITY_BLINN_PHONG)
#        define LightingUnity_GI(so, giInput, gi) LightingBlinnPhong_GI(so, giInput, gi)
#        define LightingUnity(so, worldViewDir, gi) LightingBlinnPhong(so, worldViewDir, gi)
    so.Specular = _SpecPower / 128.0;
    so.Gloss = _Glossiness;
    // NOTE: _SpecColor is used in UnityBlinnPhongLight() used in LightingBlinnPhong().
#    else
#        define LightingUnity_GI(so, giInput, gi) LightingLambert_GI(so, giInput, gi)
#        define LightingUnity(so, worldViewDir, gi) LightingLambert(so, gi)
#    endif  // defined(_LIGHTING_UNITY_BLINN_PHONG)
    so.Alpha = color.a;
#endif  // defined(_LIGHTING_UNITY_STANDARD)

    UnityGI gi = getGI(worldPos, atten);
    const float3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
#if defined(UNITY_PASS_FORWARDBASE)
    const float4 lmap = float4(0.0, 0.0, 0.0, 0.0);
    UnityGIInput giInput = getGIInput(gi.light, worldPos, worldNormal, worldViewDir, atten, lmap, ambient);
    LightingUnity_GI(so, giInput, /* inout */ gi);
#endif  // defined(UNITY_PASS_FORWARDBASE)

    half4 col = LightingUnity(so, worldViewDir, gi);
#if defined(UNITY_PASS_FORWARDBASE)
    col.rgb += so.Emission;
#endif  // defined(UNITY_PASS_FORWARDBASE)

    return col;

#undef LightingUnity_GI
#undef LightingUnity
}

/*!
 * Calculate lighting with custom method.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLightingCustom(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    const float3 worldViewDir = normalize(_WorldSpaceCameraPos - worldPos);
#if defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    const float3 worldLightDir = UnityWorldSpaceLightDir(worldPos);
#else
    const float3 worldLightDir = normalize(UnityWorldSpaceLightDir(worldPos));
#endif  // defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    const fixed3 lightCol = _LightColor0.rgb * atten;

    // Lambertian reflectance.
    const float nDotL = dot(worldNormal, worldLightDir);
    const half3 diffuse = lightCol * pow(nDotL * 0.5 + 0.5, 2.0);  // will be mul instruction.

    // Specular reflection.
    const float nDotH = dot(worldNormal, normalize(worldLightDir + worldViewDir));
    const half3 specular = pow(max(0.0, nDotH), _SpecPower) * _SpecColor.rgb * lightCol;

    // Ambient color.
#if UNITY_SHOULD_SAMPLE_SH
    ambient = ShadeSHPerPixel(worldNormal, ambient, worldPos);
#endif  // UNITY_SHOULD_SAMPLE_SH

    const half4 outColor = half4((diffuse + ambient) * color.rgb + specular, color.a);

    return outColor;
}
```

Propertiesの宣言には下記が必要．

```shaderlab
[KeywordEnum(Unity Lambert, Unity Blinn Phong, Unity Standard, Unity Standard Specular, Unlit, Custom)]
_Lighting ("Lighting method", Int) = 2
```

また，シェーダーキーワードのプラグマは下記のとおり．

```hlsl
#pragma shader_feature_local_fragment _LIGHTING_UNITY_LAMBERT _LIGHTING_UNITY_BLINN_PHONG _LIGHTING_UNITY_STANDARD _LIGHTING_UNITY_STANDARD_SPECULAR _LIGHTING_UNLIT _LIGHTING_CUSTOM
```

ライティングモデルにより必要とするプロパティが異なるが，表にまとめると下記のとおりである．

| ライティング名称 | キーワード | `_Glossiness` | `_Metallic` | `_SpecColor` | `_SpecPower` |
|-|-|:-:|:-:|:-:|:-:|
| Lambert | `_LIGHTING_UNITY_LAMBERT` | | | | |
| Blinn-Phong | `_LIGHTING_UNITY_BLINN_PHONG` | 〇 | | 〇 | 〇 |
| Standard | `_LIGHTING_UNITY_STANDARD` | 〇 | 〇 | | |
| Standard Specular | `_LIGHTING_UNITY_STANDARD_SPECULAR` | 〇 | | 〇 | |
| Unlit | `_LIGHTING_UNLIT` | | | | |
| Custom | `_LIGHTING_CUSTOM` | | | 〇 | 〇 |

---

## VRChat特有のライティング

ライティングの話題の続きであるため，本章は前章の[全部入りのシェーダー](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/03_Lighting/Shaders/AllInOne.shader "Assets/koturn/SphereTracingTips/03_Lighting/Shaders/AllInOne.shader")をベースに改造することとする．

### VRC Light Volumes

サンプルコード: [Assets/koturn/SphereTracingTips/04_LightingSpecial/Shaders/VRCLV.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/04_LightingSpecial/Shaders/VRCLV.shader "Assets/koturn/SphereTracingTips/04_LightingSpecial/Shaders/VRCLV.shader")

[VRC Light Volumes](https://github.com/REDSIM/VRCLightVolumes "REDSIM/VRCLightVolumes")に対応するには[公式のドキュメントに記載があるように](https://github.com/REDSIM/VRCLightVolumes/blob/v.2.0.1/Documentation/ForShaderDevelopers.md "REDSIM/VRCLightVolumes Documentation/ForShaderDevelopers.md")，下記の対応を行うとよい．

1. `LightVolumeSH()` で球面調和関数の係数を計算
2. `ShaderSH9()` あるいは `ShadeSHPerPixel()` の代わりに `LightVolumeEvaluate()` を用いてdiffuseを計算
3. `LightVolumeSpecular()` または `LightVolumeSpecularDominant()` を用いてspecularを計算（必須ではない）
4. 出力カラーにdiffuseとspecularを反映

公式に[Surfaceシェーダーのサンプルコード](https://github.com/REDSIM/VRCLightVolumes/blob/v.2.0.1/Packages/red.sim.lightvolumes/Shaders/ASE%20Shaders/Light%20Volume%20PBR.shader "REDSIM/VRCLightVolumes Packages/red.sim.lightvolumes/Shaders/ASE Shaders/Light Volume PBR.shader")があるが，[Amplified Shader Editor](https://assetstore.unity.com/packages/tools/visual-scripting/amplify-shader-editor-68570 "Amplify Shader Editor | Visual Scripting | Unity Asset Store")製であるため，人間に優しくないコードとなっている．
[人間向けに優しくしたコードをgistに置いた](https://gist.github.com/koturn/2626d9e33d3fc2b0ea2237c5afe6d8a3 "https://gist.github.com/koturn/2626d9e33d3fc2b0ea2237c5afe6d8a3")ので，コードとして参照したい場合はこちらを見ることを推奨する．

シェーダーコードとしては，まず有効・無効を切り替えるためのプロパティとシェーダーキーワードを設けた．
Additiveに関してはライトマップを使用する静的なオブジェクトに対して使用するものなので，レイマーチングシェーダーでの用途はないが，VRC Light Volumesの関数部分だけ他のシェーダーにそのまま転用できるようにしたかったので一応残している．

```shaderlab
[KeywordEnum(Off, On, Additive Only)]
_VRCLightVolumes ("VRC Light Volumes", Int) = 0

[KeywordEnum(Off, On, Dominant Dir)]
_VRCLightVolumesSpecular ("VRC Light Volumes Specular", Int) = 0
```

```hlsl
#pragma shader_feature_local_fragment _VRCLIGHTVOLUMES_OFF _VRCLIGHTVOLUMES_ON _VRCLIGHTVOLUMES_ADDITIVE_ONLY
#pragma shader_feature_local_fragment _VRCLIGHTVOLUMESSPECULAR_OFF _VRCLIGHTVOLUMESSPECULAR_ON _VRCLIGHTVOLUMESSPECULAR_DOMINANT_DIR
```

`LightVolumes.cginc` をインクルードすべきか，VRC Light Volumesの処理を通すべきかを判断するのに4つのマクロのいずれかが有効であることを随所で判定するのは煩雑なので，下記のマクロ `USE_VRCLIGHTVOLUMES` を設けた．

```hlsl
#if defined(_VRCLIGHTVOLUMES_ON) || defined(_VRCLIGHTVOLUMES_ADDITIVE_ONLY) || defined(_VRCLIGHTVOLUMESSPECULAR_ON) || defined(_VRCLIGHTVOLUMESSPECULAR_DOMINANT_DIR)
#    define USE_VRCLIGHTVOLUMES
#endif  // defined(_VRCLIGHTVOLUMES_ON) || defined(_VRCLIGHTVOLUMES_ADDITIVE_ONLY) || defined(_VRCLIGHTVOLUMESSPECULAR_ON) || defined(_VRCLIGHTVOLUMESSPECULAR_DOMINANT_DIR)
```

このマクロを以て必要ファイルのインクルードを行う．
パッケージマネージャで導入されている前提のパスとしているが，unitypackageで導入されている場合にも対応したい場合は，

```hlsl
#if defined(USE_VRCLIGHTVOLUMES)
#    include "Packages/red.sim.lightvolumes/Shaders/LightVolumes.cginc"
#endif  // defined(USE_VRCLIGHTVOLUMES)
```

`calcLightingCustom()` は下記の通り．
`ShadeSHPerPixel()` を置き換える形で使用する．

```hlsl
/*!
 * Calculate lighting with custom method.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLightingCustom(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    const float3 worldViewDir = normalize(_WorldSpaceCameraPos - worldPos);
#if defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    const float3 worldLightDir = UnityWorldSpaceLightDir(worldPos);
#else
    const float3 worldLightDir = normalize(UnityWorldSpaceLightDir(worldPos));
#endif  // defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    const fixed3 lightCol = _LightColor0.rgb * atten;

    // Lambertian reflectance.
    const float nDotL = dot(worldNormal, worldLightDir);
    const half3 diffuse = lightCol * pow(nDotL * 0.5 + 0.5, 2.0);  // will be mul instruction.

    // Specular reflection.
    const float nDotH = dot(worldNormal, normalize(worldLightDir + worldViewDir));
    const half3 specular = pow(max(0.0, nDotH), _SpecPower) * _SpecColor.rgb * lightCol;

    // Ambient color.
#if UNITY_SHOULD_SAMPLE_SH
#    if defined(USE_VRCLIGHTVOLUMES)
    ambient = calcLightVolumeEmission(color.rgb, worldPos, worldNormal, worldViewDir, 0.0, 0.0);
#    else
    ambient = ShadeSHPerPixel(worldNormal, ambient, worldPos);
#    endif  // defined(USE_VRCLIGHTVOLUMES)
#endif  // UNITY_SHOULD_SAMPLE_SH

    const half4 outColor = half4((diffuse + ambient) * color.rgb + specular, color.a);

    return outColor;
}
```

`calcLightingUnity()` は下記の通り．
`LightingXXX_GI()` と `LightingXXX()` の間にVRC Light Volumesの処理を入れる．
この処理を通した際は `UnityGIInput` の `indirect.diffuse` をゼロベクトルにする必要がある．
これが `ShadeSHPerPixel()` を無効にする処理に該当しており，サンプルコードの

```hlsl
#pragma surface surf Standard keepalpha fullforwardshadows exclude_path:deferred noambient
```

の `noambient` に対応する処理となっている．
（`noambient` が指定されている場合，サーフェースシェーダーの出力コード中で `sutf()` 関数の呼び出し元で `indirect.diffuse` をゼロとする処理が追加される）

```hlsl
/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLightingUnity(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    // Uniform variable requirements:
    //
    // Variant                             | `_Glossiness` | `_Metallic` | `_SpecColor` | `_SpecPower`
    // ------------------------------------|---------------|-------------|--------------|-------------
    // `_LIGHTING_UNITY_LAMBERT`           |               |             |              |
    // `_LIGHTING_UNITY_BLINN_PHONG`       | o             |             | o            | o
    // `_LIGHTING_UNITY_STANDARD`          | o             | o           |              |
    // `_LIGHTING_UNITY_STANDARD_SPECULAR` | o             |             | o            |

#if defined(_LIGHTING_UNITY_STANDARD)
#    define LightingUnity_GI(so, giInput, gi) LightingStandard_GI(so, giInput, gi)
#    define LightingUnity(so, worldViewDir, gi) LightingStandard(so, worldViewDir, gi)
    SurfaceOutputStandard so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutputStandard, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = half3(0.0, 0.0, 0.0);
    so.Metallic = _Metallic;
    so.Smoothness = _Glossiness;
    so.Occlusion = 1.0;
    so.Alpha = color.a;
#elif defined(_LIGHTING_UNITY_STANDARD_SPECULAR)
#    define LightingUnity_GI(so, giInput, gi) LightingStandardSpecular_GI(so, giInput, gi)
#    define LightingUnity(so, worldViewDir, gi) LightingStandardSpecular(so, worldViewDir, gi)
    SurfaceOutputStandardSpecular so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutputStandardSpecular, so);
    so.Albedo = color.rgb;
    so.Specular = _SpecColor.rgb;
    so.Normal = worldNormal;
    so.Emission = half3(0.0, 0.0, 0.0);
    so.Smoothness = _Glossiness;
    so.Occlusion = 1.0;
    so.Alpha = color.a;
#else
    SurfaceOutput so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutput, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = fixed3(0.0, 0.0, 0.0);
#    if defined(_LIGHTING_UNITY_BLINN_PHONG)
#        define LightingUnity_GI(so, giInput, gi) LightingBlinnPhong_GI(so, giInput, gi)
#        define LightingUnity(so, worldViewDir, gi) LightingBlinnPhong(so, worldViewDir, gi)
    so.Specular = _SpecPower / 128.0;
    so.Gloss = _Glossiness;
    // NOTE: _SpecColor is used in UnityBlinnPhongLight() used in LightingBlinnPhong().
#    else
#        define LightingUnity_GI(so, giInput, gi) LightingLambert_GI(so, giInput, gi)
#        define LightingUnity(so, worldViewDir, gi) LightingLambert(so, gi)
#    endif  // defined(_LIGHTING_UNITY_BLINN_PHONG)
    so.Alpha = color.a;
#endif  // defined(_LIGHTING_UNITY_STANDARD)

    UnityGI gi = getGI(worldPos, atten);
    const float3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
#if defined(UNITY_PASS_FORWARDBASE)
    const float4 lmap = float4(0.0, 0.0, 0.0, 0.0);
    UnityGIInput giInput = getGIInput(gi.light, worldPos, worldNormal, worldViewDir, atten, lmap, ambient);
    LightingUnity_GI(so, giInput, /* inout */ gi);
#endif  // defined(UNITY_PASS_FORWARDBASE)

#if UNITY_SHOULD_SAMPLE_SH && !defined(LIGHTMAP_ON)
#    if defined(USE_VRCLIGHTVOLUMES)
    if (_UdonLightVolumeEnabled && _UdonLightVolumeCount != 0) {
#    if defined(_LIGHTING_UNITY_STANDARD) || defined(_LIGHTING_UNITY_STANDARD_SPECULAR) || defined(_LIGHTING_UNITY_BLINN_PHONG)
        const half glossiness = _Glossiness;
#    else
        const half glossiness = 0.0;
#    endif  // defined(_LIGHTING_UNITY_STANDARD) || defined(_LIGHTING_UNITY_STANDARD_SPECULAR) || defined(_LIGHTING_UNITY_BLINN_PHONG)
#    if defined(_LIGHTING_UNITY_STANDARD)
        const half metallic = _Metallic;
#    else
        const half metallic = 0.0;
#    endif  // defined(_LIGHTING_UNITY_STANDARD)
        gi.indirect.diffuse = half3(0.0, 0.0, 0.0);
        so.Emission += calcLightVolumeEmission(color.rgb, worldPos, worldNormal, worldViewDir, glossiness, metallic);
    }
#    endif  // defined(USE_VRCLIGHTVOLUMES)
#endif  // UNITY_SHOULD_SAMPLE_SH && !defined(LIGHTMAP_ON)

    half4 col = LightingUnity(so, worldViewDir, gi);
#if defined(UNITY_PASS_FORWARDBASE)
    col.rgb += so.Emission;
#endif  // defined(UNITY_PASS_FORWARDBASE)

    return col;

#undef LightingUnity_GI
#undef LightingUnity
}
```

### LTCGI

サンプルコード: [Assets/koturn/SphereTracingTips/04_LightingSpecial/Shaders/VRCLVAndLTCGI.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/04_LightingSpecial/Shaders/VRCLVAndLTCGI.shader "Assets/koturn/SphereTracingTips/04_LightingSpecial/Shaders/VRCLVAndLTCGI.shader")

[LTCGI](https://github.com/PiMaker/ltcgi/blob/v1.6.3/Shaders/LTCGI_Simple.shader "PiMaker/ltcgi")に対応するには[公式のサンプルコード](https://github.com/PiMaker/ltcgi/blob/v1.6.3/Shaders/LTCGI_Simple.shader "PiMaker/ltcgi Shaders/LTCGI_Simple.shader")を真似て，下記の対応を行うとよい．

1. Tagsに `"LTCGI" = "_LTCGI"` のように，有効か無効かを格納するfloat型のuniform変数名を指定（この例はシェーダーに `uniform float _LTCGI` を宣言する例）<br>
   あるいは，`"LTCGI" = "ALWAYS"` と指定し，常に実行時に判定を行わないことを明示．<br>
   どちらを用いるかの判断基準は下記の通り．
    - `"LTCGI" = "変数名"`: LTCGIの処理コードをシェーダーに含め，実行時判定を行いたい場合．
    - `"LTCGI" = "ALWAYS"`: LTCGIの処理コード自体をキーワード指定 (`#pragma shader_feature_local_fragment`) で含めないようにしたい場合．
2. `LTCGI_Contribution()` を用いてdiffuseとspecularを計算．
3. 出力カラーに反映（diffuseは乗算，specularは加算）．

LTCGIはv1とv2のAPIがあるが，本記事ではv1のAPIを利用する例を示す．

まず，今回はシェーダーキーワードによりLTCGTIの有効・無効を切り替えるようにする．
プロパティとキーワード定義は下記のとおり．

```shaderlab
[Toggle(_LTCGI_ON)]
_LTCGI ("LTCGI", Int) = 0
```

```hlsl
#pragma shader_feature_local_fragment _ _LTCGI_ON
```

このキーワードを用いて，インクルードを下記のように行う．

```hlsl
#if defined(_LTCGI_ON)
#    define LTCGI_AVATAR_MODE
#    if defined(_LIGHTING_UNITY_LAMBERT)
#        define LTCGI_SPECULAR_OFF
#    endif  // defined(_LIGHTING_UNITY_LAMBERT)
#    include "Packages/at.pimaker.ltcgi/Shaders/LTCGI.cginc"
#endif  // defined(_LTCGI_ON)
```

`LTCGI_AVATAR_MODE` が定義されているとライトマップの参照を行わないなど，非staticなオブジェクト向けに最適化できる．
また， `LTCGI_SPECULAR_OFF` が定義されていると，スペキュラの計算処理を除外することができる．

前述のようにタグ設定が必要となるので，下記のように指定する．

```shaderlab
Tags
{
    "Queue" = "AlphaTest"
    // "RenderType" = "Transparent"
    "DisableBatching" = "True"
    "IgnoreProjector" = "True"
    "VRCFallback" = "Hidden"
    "LTCGI" = "ALWAYS"
}
```

あとはLTCGIを処理に組み込むだけである．

`calcLightingCustom()` は下記の通り．
`LTCGI_Contribution()` を用いて，LTCGIのDiffuseとSpecularを取得し，それを用いた結果を加算合成するだけである．

```hlsl
/*!
 * Calculate lighting with custom method.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLightingCustom(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    const float3 worldViewDir = normalize(_WorldSpaceCameraPos - worldPos);
#if defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    const float3 worldLightDir = UnityWorldSpaceLightDir(worldPos);
#else
    const float3 worldLightDir = normalize(UnityWorldSpaceLightDir(worldPos));
#endif  // defined(USING_LIGHT_MULTI_COMPILE) && defined(USING_DIRECTIONAL_LIGHT)
    const fixed3 lightCol = _LightColor0.rgb * atten;

    // Lambertian reflectance.
    const float nDotL = dot(worldNormal, worldLightDir);
    const half3 diffuse = lightCol * pow(nDotL * 0.5 + 0.5, 2.0);  // will be mul instruction.

    // Specular reflection.
    const float nDotH = dot(worldNormal, normalize(worldLightDir + worldViewDir));
    const half3 specular = pow(max(0.0, nDotH), _SpecPower) * _SpecColor.rgb * lightCol;

    // Ambient color.
#if UNITY_SHOULD_SAMPLE_SH
#    if defined(USE_VRCLIGHTVOLUMES)
    ambient = calcLightVolumeEmission(color.rgb, worldPos, worldNormal, worldViewDir, 0.0, 0.0);
#    else
    ambient = ShadeSHPerPixel(worldNormal, ambient, worldPos);
#    endif  // defined(USE_VRCLIGHTVOLUMES)
#endif  // UNITY_SHOULD_SAMPLE_SH

    half4 outColor = half4((diffuse + ambient) * color.rgb + specular, color.a);

#if defined(_LTCGI_ON)
    float3 ltcgiSpecular = float3(0.0, 0.0, 0.0);
    float3 ltcgiDiffuse = float3(0.0, 0.0, 0.0);
    LTCGI_Contribution(
       worldPos,
       worldNormal,
       worldViewDir,
       1.0 - lossiness,
       float2(0.0, 0.0),
       /* inout */ ltcgiDiffuse,
       /* inout */ ltcgiSpecular);
#        if defined(LTCGI_SPECULAR_OFF)
    outColor.rgb += color.rgb * ltcgiDiffuse;
#        else
    outColor.rgb += color.rgb * ltcgiDiffuse + ltcgiSpecular;
#        endif  // defined(LTCGI_SPECULAR_OFF)
#    endif  // defined(_LTCGI_ON)
#endif  // defined(_LTCGI_ON)

    return outColor;
}
```

`calcLightingUnity()` は下記の通り．
VRC Light Volumesのように， `ShadeSHPerPixel()` を置き換えるものではないため，`gi.indirect.diffuse` をゼロに設定する処理は必要なく，単に追加で加算合成する色として扱うだけでよい．

```hlsl
/*!
 * Calculate lighting.
 * @param [in] color  Base color.
 * @param [in] worldPos  World coordinate.
 * @param [in] worldNormal  Normal in world space.
 * @param [in] atten  Light attenuation.
 * @param [in] ambient  Ambient light.
 * @return Lighting applied color.
 */
half4 calcLightingUnity(half4 color, float3 worldPos, float3 worldNormal, half atten, half3 ambient)
{
    // Uniform variable requirements:
    //
    // Variant                             | `_Glossiness` | `_Metallic` | `_SpecColor` | `_SpecPower`
    // ------------------------------------|---------------|-------------|--------------|-------------
    // `_LIGHTING_UNITY_LAMBERT`           |               |             |              |
    // `_LIGHTING_UNITY_BLINN_PHONG`       | o             |             | o            | o
    // `_LIGHTING_UNITY_STANDARD`          | o             | o           |              |
    // `_LIGHTING_UNITY_STANDARD_SPECULAR` | o             |             | o            |

#if defined(_LIGHTING_UNITY_STANDARD)
#    define LightingUnity_GI(so, giInput, gi) LightingStandard_GI(so, giInput, gi)
#    define LightingUnity(so, worldViewDir, gi) LightingStandard(so, worldViewDir, gi)
    SurfaceOutputStandard so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutputStandard, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = half3(0.0, 0.0, 0.0);
    so.Metallic = _Metallic;
    so.Smoothness = _Glossiness;
    so.Occlusion = 1.0;
    so.Alpha = color.a;
#elif defined(_LIGHTING_UNITY_STANDARD_SPECULAR)
#    define LightingUnity_GI(so, giInput, gi) LightingStandardSpecular_GI(so, giInput, gi)
#    define LightingUnity(so, worldViewDir, gi) LightingStandardSpecular(so, worldViewDir, gi)
    SurfaceOutputStandardSpecular so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutputStandardSpecular, so);
    so.Albedo = color.rgb;
    so.Specular = _SpecColor.rgb;
    so.Normal = worldNormal;
    so.Emission = half3(0.0, 0.0, 0.0);
    so.Smoothness = _Glossiness;
    so.Occlusion = 1.0;
    so.Alpha = color.a;
#else
    SurfaceOutput so;
    UNITY_INITIALIZE_OUTPUT(SurfaceOutput, so);
    so.Albedo = color.rgb;
    so.Normal = worldNormal;
    so.Emission = fixed3(0.0, 0.0, 0.0);
#    if defined(_LIGHTING_UNITY_BLINN_PHONG)
#        define LightingUnity_GI(so, giInput, gi) LightingBlinnPhong_GI(so, giInput, gi)
#        define LightingUnity(so, worldViewDir, gi) LightingBlinnPhong(so, worldViewDir, gi)
    so.Specular = _SpecPower / 128.0;
    so.Gloss = _Glossiness;
    // NOTE: _SpecColor is used in UnityBlinnPhongLight() used in LightingBlinnPhong().
#    else
#        define LightingUnity_GI(so, giInput, gi) LightingLambert_GI(so, giInput, gi)
#        define LightingUnity(so, worldViewDir, gi) LightingLambert(so, gi)
#    endif  // defined(_LIGHTING_UNITY_BLINN_PHONG)
    so.Alpha = color.a;
#endif  // defined(_LIGHTING_UNITY_STANDARD)

    UnityGI gi = getGI(worldPos, atten);
    const float3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
#if defined(UNITY_PASS_FORWARDBASE)
    const float4 lmap = float4(0.0, 0.0, 0.0, 0.0);
    UnityGIInput giInput = getGIInput(gi.light, worldPos, worldNormal, worldViewDir, atten, lmap, ambient);
    LightingUnity_GI(so, giInput, /* inout */ gi);
#endif  // defined(UNITY_PASS_FORWARDBASE)

#if UNITY_SHOULD_SAMPLE_SH && !defined(LIGHTMAP_ON)
#    if defined(USE_VRCLIGHTVOLUMES)
#    if defined(_LIGHTING_UNITY_STANDARD) || defined(_LIGHTING_UNITY_STANDARD_SPECULAR) || defined(_LIGHTING_UNITY_BLINN_PHONG)
        const half glossiness = _Glossiness;
#    else
        const half glossiness = 0.0;
#    endif  // defined(_LIGHTING_UNITY_STANDARD) || defined(_LIGHTING_UNITY_STANDARD_SPECULAR) || defined(_LIGHTING_UNITY_BLINN_PHONG)
    if (_UdonLightVolumeEnabled && _UdonLightVolumeCount != 0) {
#        if defined(_LIGHTING_UNITY_STANDARD)
        const half metallic = _Metallic;
#        else
        const half metallic = 0.0;
#        endif  // defined(_LIGHTING_UNITY_STANDARD)
        gi.indirect.diffuse = half3(0.0, 0.0, 0.0);
        so.Emission += calcLightVolumeEmission(color.rgb, worldPos, worldNormal, worldViewDir, glossiness, metallic);
    }
#    endif  // defined(USE_VRCLIGHTVOLUMES)

#    if defined(_LTCGI_ON)
    float3 ltcgiSpecular = float3(0.0, 0.0, 0.0);
    float3 ltcgiDiffuse = float3(0.0, 0.0, 0.0);
    LTCGI_Contribution(
       worldPos,
       worldNormal,
       worldViewDir,
       1.0 - glossiness,
       float2(0.0, 0.0),
       /* inout */ ltcgiDiffuse,
       /* inout */ ltcgiSpecular);
#        if defined(LTCGI_SPECULAR_OFF)
    so.Emission += color.rgb * ltcgiDiffuse;
#        else
    so.Emission += color.rgb * ltcgiDiffuse + ltcgiSpecular;
#        endif  // defined(LTCGI_SPECULAR_OFF)
#    endif  // defined(_LTCGI_ON)
#endif  // UNITY_SHOULD_SAMPLE_SH && !defined(LIGHTMAP_ON)

    half4 col = LightingUnity(so, worldViewDir, gi);
#if defined(UNITY_PASS_FORWARDBASE)
    col.rgb += so.Emission;
#endif  // defined(UNITY_PASS_FORWARDBASE)

    return col;

#undef LightingUnity_GI
#undef LightingUnity
}
```

---

## テクスチャを貼り付ける

レイマーチングでは3Dモデルが頂点単位で持っているuv座標をレイマーチングの描画対象に結びつけないため，テクスチャを描画オブジェクトに貼り付けるには一工夫必要となる．

### Tri-Planar

Tri-Planarとはx軸，y軸，z軸の三方向からの平面マッピングを行う手法である．
基本形は下記のような形となる．

```hlsl
/*!
 * @brief Sample texture using Tri-Planar texture mapping.
 *
 * @param [in] tex  2D texture samplar.
 * @param [in] pos  Object/World space position.
 * @param [in] normal  Normal.
 *
 * @return Sampled color.
 */
half4 tex2DTriPlanar(sampler2D tex, float3 pos, float3 normal)
{
    float3 blending = normalize(max(abs(normal), 0.00001));
    blending /= dot(blending, (1.0).xxx);
    const half4 xaxis = tex2D(tex, pos.yz);
    const half4 yaxis = tex2D(tex, pos.xz);
    const half4 zaxis = tex2D(tex, pos.xy);
    return xaxis * blending.x + yaxis * blending.y + zaxis * blending.z;
}
```

`pos`, `normal` は3次元の座標とその座標における法線を意味するが，ワールド空間のものかオブジェクト空間のものを用いるかは一考の余地がある．
ワールド空間でのレイマーチングであれば，Tri-Planarの入力としてもワールド空間における座標と法線を用いても問題はないが，移動や回転等の変化があるオブジェクト空間でのレイマーチングであればオブジェクト空間の座標と法線を用いる方がよいと思う．

---

## アルゴリズムの改良

### レイの開始点，終了点

#### レイの開始点をメッシュ表面に設定

下記のどちらかの場合に適用可能な手法である．

- 投影先がQuadでCube内部にレイマーチングオブジェクトを描画する場合
- 投影先がCubeかつ `Cull Back` の指定があり，Cube内部にレイマーチングオブジェクトを描画する場合

特に描画対象がQuadの場合は窓から覗き込むと向こう側に景色が見えるという場合は，開始点はメッシュの表面からにしておいた方がよい．
modを利用して景色が無限に広がるようなものを描画する場合，Quadより手前に描画されることで，違和感のある見え方になるためだ．


[![メッシュの表面をレイの開始点とする](/images/koturn-unity-sphere-tracing-tips/StartFromFrontOfMesh.png)](/images/koturn-unity-sphere-tracing-tips/StartFromFrontOfMesh.png "メッシュの表面をレイの開始点とする")


#### Cull Front

ポリゴンの裏面に対して描画を行う場合はレイの開始点をカメラではなく，メッシュ表面に設定することができる．

### レイ進行アルゴリズムの改善

#### Over-relaxation sphere tracing

2014年の発表手法．
レイを多めに進めて，境界面を超過した場合に引き返すという手法．

- [論文](https://diglib.eg.org/items/8ea5fa60-fe2f-4fef-8fd0-3783cb3200f0)

アルゴリズムの疑似コードは下記の通り．
$\boldsymbol{f}$ は距離関数， $\boldsymbol{s}$ はレイの始点からの距離（進行長）を引数に受け取って，レイの現在位置（ベクトル）を返す関数である．

[![Over-relaxation sphere tracingの疑似コード](/images/koturn-unity-sphere-tracing-tips/PseudoCode_OverRelaxation.png)](/images/koturn-unity-sphere-tracing-tips/PseudoCode_OverRelaxation.png "Over-relaxation sphere tracingの疑似コード")

マーチングループ部分の実装のみを抜粋すると下記の通り．

```hlsl
#if defined(_CALCSPACE_WORLD)
const float3 rayOrigin = _WorldSpaceCameraPos;
#else
const float3 rayOrigin = mul(unity_WorldToObject, float4(_WorldSpaceCameraPos, 1.0)).xyz;
#endif  // defined(_CALCSPACE_WORLD)
const float3 rayDir = normalize(fi.fragPos - rayOrigin);

const float3 rcpScales = rcp(_Scales);
const float marchingFactor = rsqrt(dot(rayDir * rcpScales, rayDir * rcpScales));

float rayLength = 0.0;
float r = asfloat(0x7f800000);  // +inf
float d = 0.0;
for (int i = 0; abs(r) >= _MinRayLength && rayLength < _MaxRayLength && i < _MaxLoop; i++) {
    const float nextRayLength = rayLength + d * marchingFactor;
    const float nextR = map((rayOrigin + rayDir * nextRayLength) * rcpScales);
    if (d <= r + abs(nextR)) {
        d = _OverRelaxFactor * nextR;
        rayLength = nextRayLength;
        r = nextR;
    } else {
        d = r;
    }
}

if (abs(r) >= _MinRayLength) {
    discard;
}
```

疑似コードと上記コード中の関数と変数の対応関係は下記の表のとおり．

| 疑似コード中の関数・変数 | コード中の関数・変数・計算 |
|-|-|
| $f$ | `map` |
| $\boldsymbol{s}$ | 対応関数なし，`rayOrigin + rayDir * nextRayLength` の計算が該当 |
| $\epsilon$ | `_MinRayLength` |
| $t_{max}$ | `_MaxRayLength` |
| $i_{max}$ | `_MaxLoop` |
| $\omega$ | `_RelaxFactor` |
| $t$ | `rayLength` |
| $T$ | `nextRayLength` |
| $z$ | `d` |
| $r$ | `r` |
| $R$ | `nextR` |

#### Accelerating Sphere Tracing

サンプルコード: [Assets/koturn/SphereTracingTips/06_Algorithm/Shaders/Accelerating.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/06_Algorithm/Shaders/Accelerating.shader "Assets/koturn/SphereTracingTips/06_Algorithm/Shaders/Accelerating.shader")

2019/02 の発表手法．
前手法と比較して，境界面の超過が発生しにくいように改良した手法といえる．

- [ポスター](https://www.researchgate.net/publication/331547302_Accelerating_Sphere_Tracing "Accelerating Sphere Tracing")
- [論文](https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing "Accelerating Sphere Tracing")

アルゴリズムの疑似コードは下記の通り．

[![Accelerating Sphere Tracingの疑似コード](/images/koturn-unity-sphere-tracing-tips/PseudoCode_Accelerating.png)](/images/koturn-unity-sphere-tracing-tips/PseudoCode_Accelerating.png "Accelerating Sphere Tracingの疑似コード")

マーチングループ部分の実装のみを抜粋すると下記の通り．

```hlsl
#if defined(_CALCSPACE_WORLD)
const float3 rayOrigin = _WorldSpaceCameraPos;
#else
const float3 rayOrigin = mul(unity_WorldToObject, float4(_WorldSpaceCameraPos, 1.0)).xyz;
#endif  // defined(_CALCSPACE_WORLD)
const float3 rayDir = normalize(fi.fragPos - rayOrigin);

const float3 rcpScales = rcp(_Scales);
const float marchingFactor = rsqrt(dot(rayDir * rcpScales, rayDir * rcpScales));

float rayLength = 0.0;
float r = map((rayOrigin + rayDir * rayLength) * rcpScales);
float d = r;
for (int i = 1; r >= _MinRayLength && (rayLength + r * marchingFactor) < _MaxRayLength && i < _MaxLoop; i++) {
    const float nextRayLength = rayLength + d * marchingFactor;
    const float nextR = map((rayOrigin + rayDir * nextRayLength) * rcpScales);
    if (d <= r + abs(nextR)) {
        const float deltaR = nextR - r;
        const float2 zr = d.xx + deltaR * float2(1.0, -1.0);
        d = nextR + _AccelarationFactor * nextR * (zr.x / zr.y);
        rayLength = nextRayLength;
        r = nextR;
    } else {
        d = r;
    }
}

if (abs(r) >= _MinRayLength) {
    discard;
}
```

疑似コードと上記コード中の関数と変数の対応関係は下記の表のとおり．

| 疑似コード中の関数・変数 | コード中の関数・変数・計算 |
|-|-|
| $f$ | `map` |
| $\boldsymbol{s}$ | 対応関数なし，`rayOrigin + rayDir * nextRayLength` の計算が該当 |
| $\epsilon$ | `_MinRayLength` |
| $t_{max}$ | `_MaxRayLength` |
| $i_{max}$ | `_MaxLoop` |
| $\omega$ | `_AccelarationFactor` |
| $t$ | `rayLength` |
| $T$ | `nextRayLength` |
| $z$ | `d` |
| $r$ | `r` |
| $R$ | `nextR` |

疑似コード中の $\dfrac{T - t + R - r}{T - t - (R - r)}$ の分母と分子を求める際に計算のベクトル化が可能なため，下記のように計算している．

```hlsl
const float deltaR = nextR - r;
const float2 zr = d.xx + deltaR * float2(1.0, -1.0);
```

#### Automatic Step Size Relaxation in Sphere Tracing

サンプルコード: [Assets/koturn/SphereTracingTips/06_Algorithm/Shaders/AutomaticStepSizeRelaxation.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/06_Algorithm/Shaders/AutomaticStepSizeRelaxation.shader "Assets/koturn/SphereTracingTips/06_Algorithm/Shaders/AutomaticStepSizeRelaxation.shader")

2023/05 の発表手法．

論文中には前述の2手法の疑似コードの記載もあるため，単純に実装の参考にするにはこの論文のみ参照するだけでも問題ない．

- [論文](https://www.researchgate.net/publication/370902411_Automatic_Step_Size_Relaxation_in_Sphere_Tracing "Automatic Step Size Relaxation in Sphere Tracing")

アルゴリズムの疑似コードは下記の通り．

[![Automatic Step Size Relaxation in Sphere Tracingの疑似コード](/images/koturn-unity-sphere-tracing-tips/PseudoCode_AutomaticStepSizeRelaxation.png)](/images/koturn-unity-sphere-tracing-tips/PseudoCode_AutomaticStepSizeRelaxation.png "Automatic Step Size Relaxation in Sphere Tracingの疑似コード")

マーチングループ部分の実装のみを抜粋すると下記の通り．

```hlsl
#if defined(_CALCSPACE_WORLD)
const float3 rayOrigin = _WorldSpaceCameraPos;
#else
const float3 rayOrigin = mul(unity_WorldToObject, float4(_WorldSpaceCameraPos, 1.0)).xyz;
#endif  // defined(_CALCSPACE_WORLD)
const float3 rayDir = normalize(fi.fragPos - rayOrigin);

const float3 rcpScales = rcp(_Scales);
const float marchingFactor = rsqrt(dot(rayDir * rcpScales, rayDir * rcpScales));

float rayLength = 0.0;
float r = map((rayOrigin + rayDir * rayLength) * rcpScales);
float d = r;
float m = -1.0;

for (int i = 1; r >= _MinRayLength && (rayLength + r * marchingFactor) < _MaxRayLength && i < _MaxLoop; i++) {
    const float nextRayLength = rayLength + d * marchingFactor;
    const float nextR = map((rayOrigin + rayDir * nextRayLength) * rcpScales);
    if (d <= r + abs(nextR)) {
        m = lerp(m, (nextR - r) / d, _AutoRelaxFactor);
        rayLength = nextRayLength;
        r = nextR;
    } else {
        m = -1.0;
    }
    d = 2.0 * r / (1.0 - m);
}

if (r >= _MinRayLength) {
    discard;
}
```

疑似コードと上記コード中の関数と変数の対応関係は下記の表のとおり．

| 疑似コード中の関数・変数 | コード中の関数・変数・計算 |
|-|-|
| $f$ | `map` |
| $\boldsymbol{s}$ | 対応関数なし，`rayOrigin + rayDir * nextRayLength` の計算が該当 |
| $\epsilon$ | `_MinRayLength` |
| $t_{max}$ | `_MaxRayLength` |
| $i_{max}$ | `_MaxLoop` |
| $\beta$ | `_AutoRelaxFactor` |
| $t$ | `rayLength` |
| $T$ | `nextRayLength` |
| $z$ | `d` |
| $r$ | `r` |
| $R$ | `nextR` |
| $m$ | `m` |
| $M$ | `(nextR - r) / d` |

---

## Conservative Depth Output

サンプルコード: [Assets/koturn/SphereTracingTips/07_ConservationDepthOutput/Shaders/ConservationDepthOutput.shader](https://github.com/koturn/SphereTracingTips/blob/main/Assets/koturn/SphereTracingTips/07_ConservationDepthOutput/Shaders/ConservationDepthOutput.shader "Assets/koturn/SphereTracingTips/07_ConservationDepthOutput/Shaders/ConservationDepthOutput.shader")

`SV_Depth` の出力を有効にした場合，Early-Zが行われなくなる．
そのため，例えば描画対象のレイマーチングオブジェクトがCubeに内包されるものであり，他の不透明メッシュオブジェクトにより遮蔽される場合であっても描画処理は行われてしまう．

[![ポリゴンメッシュによりレイマーチングオブジェクトが遮蔽される例](/images/koturn-unity-sphere-tracing-tips/OcclusionByPolygonMesh.png )](/images/koturn-unity-sphere-tracing-tips/OcclusionByPolygonMesh.png "ポリゴンメッシュによりレイマーチングオブジェクトが遮蔽される例")

`SV_Depth` の代わりに `SV_DepthLessEqual`, `SV_DepthGreaterEqual` というセマンティクスを用いれば，投影先メッシュより後方，あるいは前方への深度出力を行わない仮定を与えることができ，投影面に対してEarly-Zを行うことが可能となる．

| セマンティクス | 説明 |
|-|-|
| `SV_DepthLessEqual` | 投影先メッシュより後ろ側にレイマーチングオブジェクトを描画する際に用いる．Quadへの投影や背面カリング (`Cull Back`) を有効にしているCubeへの投影の場合など． |
| `SV_DepthGreaterEqual` | 投影先メッシュより前側にレイマーチングオブジェクトを描画する際に用いる．前面カリング (`Cull Front`) を有効にしているCubeへの投影の場合など． |

サンプルコードでは各セマンティクスをキーワードによる切り替え可能とするために，ベースとするシェーダーのトグル式のプロパティから `KeywordEnum` による選択式へ変更している．

```shaderlab
[KeywordEnum(Off, On, LessEqual, GreaterEqual)]
_SVDepth ("SV_Depth ouput", Int) = 1
```

```hlsl
#pragma shader_feature_local_fragment _SVDEPTH_OFF _SVDEPTH_ON _SVDEPTH_LESSEQUAL _SVDEPTH_GREATEREQUAL
```

`SV_DepthLessEqual` または `SV_DepthGreaterEqual` を使用するためには `#pragma target 5.0` の指定が必要となる．
Unity 2021.2 からは特定のキーワードが有効時にシェーダーモデルの変更が可能となっているので，不必要に `#pragma target 5.0` を使用したくないのであれば下記のように指定するとよい．

```hlsl
#if UNITY_VERSION >= 202030
#    pragma target 3.0
#    pragma target 5.0 _SVDEPTH_LESSEQUAL _SVDEPTH_GREATEREQUAL
#else
#    pragma target 5.0
#endif  // UNITY_VERSION >= 202030
```

セマンティクスの吸収は下記のマクロ定義で行っている．

```hlsl
#if defined(_SVDEPTH_ON)
#    define DEPTH_SEMANTICS SV_Depth
#elif defined(_SVDEPTH_LESSEQUAL)
#    define DEPTH_SEMANTICS SV_DepthLessEqual
#elif defined(_SVDEPTH_GREATEREQUAL)
#    define DEPTH_SEMANTICS SV_DepthGreaterEqual
#endif  // defined(_SVDEPTH_ON)
```

他ヘッダファイルへの分離を考えているなら，下記のようにシェーダーモデルが非対応であるときに `SV_DepthLessEqual` または `SV_DepthGreaterEqual` を `SV_Depth` へフォールバックするのもアリだと思う．

```hlsl
#if defined(_SVDEPTH_ON)
#    define DEPTH_SEMANTICS SV_Depth
#elif defined(_SVDEPTH_LESSEQUAL)
#    if SHADER_TARGET >= 45
#        define DEPTH_SEMANTICS SV_DepthLessEqual
#    else
#        define DEPTH_SEMANTICS SV_Depth
#    endif  // SHADER_TARGET >= 45
#elif defined(_SVDEPTH_GREATEREQUAL)
#    if SHADER_TARGET >= 45
#        define DEPTH_SEMANTICS SV_DepthGreaterEqual
#    else
#        define DEPTH_SEMANTICS SV_Depth
#    endif  // SHADER_TARGET >= 45
#endif  // defined(_SVDEPTH_ON)
```


同時にベースとするシェーダーで `_SVDEPTH_ON` の定義有無で判定していた箇所は `DEPTH_SEMANTICS` の定義有無での判定に差し替えた．


### Cull BackのCubeでの出力結果

レイマーチングオブジェクトとQuadを交差させた場合の描画結果の具体例を示す．

投影面より後ろ側にレイマーチングブジェクトを描画するにも関わらず `SV_DepthGreaterEqual` を指定している場合，Quadとの交差しているあたりで正常に描画が行われなくなっている．

| `SV_Depth` | `SV_DepthLessEqual` | `SV_DepthGreaterEqual` |
|-|-|-|
| [![Cull BackのCubeへの投影とQuadとの交差 SV_Depth](/images/koturn-unity-sphere-tracing-tips/CubeCullBackSVDepth.png)](/images/koturn-unity-sphere-tracing-tips/CubeCullBackSVDepth.png "Cull BackのCubeへの投影とQuadとの交差") | [![Cull BackのCubeへの投影とQuadとの交差 SV_DepthLessEqual](/images/koturn-unity-sphere-tracing-tips/CubeCullBackSVDepthLessEqual.png)](/images/koturn-unity-sphere-tracing-tips/CubeCullBackSVDepthLessEqual.png "Cull BackのCubeへの投影とQuadとの交差 SV_DepthLessEqual") | [![Cull BackのCubeへの投影とQuadとの交差 SV_DepthGreaterEqual](/images/koturn-unity-sphere-tracing-tips/CubeCullBackSVDepthGreaterEqual.png )](/images/koturn-unity-sphere-tracing-tips/CubeCullBackSVDepthGreaterEqual.png "Cull BackのCubeへの投影とQuadとの交差 SV_DepthGreaterEqual") |

### Cull FrontのCubeでの出力結果

投影面がQuadで遮蔽されている場合， `SV_DepthLessEqual` が指定されていると先に投影面のZ-Testが行われてしまうため，描画が正常に行われなっている．
全面的に描画されなくなるのではなく，部分的に描画が欠ける現象となっている．

| `SV_Depth` | `SV_DepthLessEqual` | `SV_DepthGreaterEqual` |
|-|-|-|
| [![Cull FrontのCubeへの投影とQuadとの交差 SV_Depth](/images/koturn-unity-sphere-tracing-tips/CubeCullFrontSVDepth.png)](/images/koturn-unity-sphere-tracing-tips/CubeCullFrontSVDepth.png "Cull FrontのCubeへの投影とQuadとの交差") | [![Cull FrontのCubeへの投影とQuadとの交差 SV_DepthLessEqual](/images/koturn-unity-sphere-tracing-tips/CubeCullFrontSVDepthLessEqual.png)](/images/koturn-unity-sphere-tracing-tips/CubeCullFrontSVDepthLessEqual.png "Cull FrontのCubeへの投影とQuadとの交差 SV_DepthLessEqual") | [![Cull FrontのCubeへの投影とQuadとの交差 SV_DepthGreaterEqual](/images/koturn-unity-sphere-tracing-tips/CubeCullFrontSVDepthGreaterEqual.png )](/images/koturn-unity-sphere-tracing-tips/CubeCullFrontSVDepthGreaterEqual.png "Cull FrontのCubeへの投影とQuadとの交差 SV_DepthGreaterEqual") |

---

## その他

### ループのアンロール

[VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】](https://butadiene.booth.pm/items/1829988 "VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】")で記載されているコードはマーチングループに対して `[unroll]` の指定がある．
しかし `[unroll]` はコンパイル時間と出力コードサイズを肥大化させるため，使用するにあたっては慎重に検討する方がよい．

あまりに巨大なコードサイズのシェーダーになると，実行時の読み込みに時間を要することになるかもしれない．
また，コンパイルに1分も要するとなると，開発においてもトライアル＆エラーを繰り返しにくくなる．

---

## 参考文献

- [VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】](https://butadiene.booth.pm/items/1829988 "VRChatでどうしても レイマーチングをしたいあなたへ贈る本【Ver 1.1】 - Butadiene Works - BOOTH")
- [Unity でオブジェクトスペースの Raymarching をフォワードレンダリングでやってみた - 凹みTips](https://tips.hecomi.com/entry/2018/12/31/211448 "Unity でオブジェクトスペースの Raymarching をフォワードレンダリングでやってみた - 凹みTips")
- [Inigo Quilez :: computer graphics, mathematics, shaders, fractals, demoscene and more](https://iquilezles.org/articles/normalsSDF/ "Inigo Quilez :: computer graphics, mathematics, shaders, fractals, demoscene and more")
- [Unity で距離関数の記述だけでレイマーチングができる uRaymarching を Forward / XR 対応した - 凹みTips](https://tips.hecomi.com/entry/2019/01/27/233137 "Unity で距離関数の記述だけでレイマーチングができる uRaymarching を Forward / XR 対応した - 凹みTips")
- [Unity - Manual: Targeting shader models and GPU features in HLSL](https://docs.unity3d.com/2022.3/Documentation/Manual/SL-ShaderCompileTargets.html "Unity - Manual: Targeting shader models and GPU features in HLSL")
- [sqrt (sm4 - asm) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/sqrt--sm4---asm- "sqrt (sm4 - asm) - Win32 apps | Microsoft Learn")
- [div (sm4 - asm) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/div--sm4---asm- "div (sm4 - asm) - Win32 apps | Microsoft Learn")
- [rsq (sm4 - asm) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/rsq--sm4---asm- "rsq (sm4 - asm) - Win32 apps | Microsoft Learn")
- [rsqrt - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-rsqrt "rsqrt - Win32 apps | Microsoft Learn")
- [Semantics - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics "Semantics - Win32 apps | Microsoft Learn")
- [高速に逆平方根を計算するアルゴリズム](https://lipoyang.hatenablog.com/entry/2021/02/06/194619 "高速逆平方根 (fast inverse square root) のアルゴリズム解説 - 滴了庵日録")
- [hecomi/uRaymarching](https://github.com/hecomi/uRaymarching "hecomi/uRaymarching")
- [lilxyzw/Shader-MEMO Assets/SPSITest.shader](https://github.com/lilxyzw/Shader-MEMO/blob/main/Assets/SPSITest.shader "Shader-MEMO/Assets/SPSITest.shader at main · lilxyzw/Shader-MEMO")
- [pema99/shader-knowledge raymarching.md ](https://github.com/pema99/shader-knowledge/blob/main/raymarching.md "shader-knowledge/raymarching.md at main · pema99/shader-knowledge")
- [REDSIM/VRCLightVolumes](https://github.com/REDSIM/VRCLightVolumes "REDSIM/VRCLightVolumes: Nextgen voxel based light probes replacement for VRChat")
- [PiMaker/ltcgi](https://github.com/PiMaker/ltcgi "PiMaker/ltcgi: Optimized plug-and-play realtime area lighting using the linearly transformed cosine algorithm for Unity/VRChat.")
- [Amplified Shader Editor](https://assetstore.unity.com/packages/tools/visual-scripting/amplify-shader-editor-68570 "Amplify Shader Editor | Visual Scripting | Unity Asset Store")
- [koturn/2626d9e33d3fc2b0ea2237c5afe6d8a3](https://gist.github.com/koturn/2626d9e33d3fc2b0ea2237c5afe6d8a3 "https://gist.github.com/koturn/2626d9e33d3fc2b0ea2237c5afe6d8a3")
- [Introduction to Shaders for VRChatter](https://docs.google.com/presentation/d/1BMLSOJMTqasoe8CeDAXFBDUbPzKQVRJ6CSup-rJcnRU/edit#slide=id.p "Introduction to Shaders for VRChatter")
- [2DテクスチャをUVを使わずに投影するTri-Planar Texture Mapping #WebGL - Qiita](https://qiita.com/edo_m18/items/c8995fe91778895c875e "2DテクスチャをUVを使わずに投影するTri-Planar Texture Mapping #WebGL - Qiita")
- [Triplanar Mapping - Unityを頑張るblog](https://ssr-maguro.hatenablog.com/entry/2020/01/22/192000 "Triplanar Mapping - Unityを頑張るblog")
- [Enhanced Sphere Tracing](https://diglib.eg.org/items/8ea5fa60-fe2f-4fef-8fd0-3783cb3200f0 "Enhanced Sphere Tracing")
- [(PDF) Accelerating Sphere Tracing](https://www.researchgate.net/publication/331547302_Accelerating_Sphere_Tracing "Accelerating Sphere Tracing")
- [(PDF) Accelerating Sphere Tracing](https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing "Accelerating Sphere Tracing")
- [(PDF) Automatic Step Size Relaxation in Sphere Tracing](https://www.researchgate.net/publication/370902411_Automatic_Step_Size_Relaxation_in_Sphere_Tracing "Automatic Step Size Relaxation in Sphere Tracing")
- [2023-09-02 レイトレ合宿9 redflash[4D]](https://docs.google.com/presentation/d/1f05HU58XD2w_71CJOdiEqOsBI8L2TYRTMndNT9MPqpI/edit?slide=id.gbd0ef54b81_0_79#slide=id.gbd0ef54b81_0_79 "2023-09-02 レイトレ合宿9 redflash[4D]")
- [Direct3D 11.3 Functional Specification](https://microsoft.github.io/DirectX-Specs/d3d/archive/D3D11_3_FunctionalSpec.htm "Direct3D 11.3 Functional Specification")
- [D3D11詳解 新しいDirectXがもたらすイノベーションとは？](https://cedec.cesa.or.jp/2009/ssn_archive/pdf/sep1st/SS03.pdf "D3D11詳解 新しいDirectXがもたらすイノベーションとは？")
- [デモシーンへようこそ 4KBで映像作品を作る技術、およびゲーム開発への応用](http://ftp7.jp.netbsd.org/pub/osdn.net/libraryzip/68289/C17_184.pdf "デモシーンへようこそ 4KBで映像作品を作る技術、およびゲーム開発への応用")
