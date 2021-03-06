渲染之十八 实时全局光照、探针体和LOD组
======

 * 支持实时全局光照
 * 计算自发光对全局光照的贡献
 * 使用光照探针代理体
 * 将 LOD 组与全局光照结合
 * 交叉淡入淡出 LOD 组

这篇是关于渲染的系列教程的第18部分。在第17部分完成烘焙全局光照后，我们继续支持实时全局光照，之后，我们还将支持光照探针代理体和交叉淡入淡出LOD组。

从本篇开始，教程使用 Unity 2017.1.0f3 制作，不适用于旧版本，因为我们最终将使用新的着色器功能。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/tutorial-image.jpg)  
*静态 LOD 组和实时全局光照的结合*

# 1 实时全局光照

烘焙光照对于静态几何体非常有效，并且借助光照探针也非常适合动态几何体。但是，它无法处理动态光源。混合模式下的光源可以进行一些实时调整，但是调整过多会使烘焙的间接光严重不协调。因此，当你有户外场景时，太阳必须保持不变，它不能像现实生活中那样东升西落，因为这就需要逐渐改变全局光照。所以场景必须保持在某个时间点不变。

为了使间接光照与移动的太阳一起工作，Unity 使用 Enlighten 系统来实时计算全局光照。除了在运行时实时计算光照贴图和探针外，它的工作方式与烘焙间接光一样。

想弄清楚间接光需要知道光如何在静态表面之间反弹，关键问题是哪些表面可能受到哪些表面的影响，以及其程度。这些工作量太大，不能实时完成，因此，这些数据由编辑器处理并存储以便在运行时使用。 Enlighten 使用这些数据来计算实时光照贴图和探针。即使这样，它也只适用于低分辨率光照贴图。

## 1.1 开启实时全局光照

实时全局光照和烘焙是独立的两个选项，您可以同时关闭、开启一个或两个都开启。其复选框位于 Lighting 窗口的 Realtime Lighting 部分。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-gi-enabled.png)  
*实时全局光照和烘焙同时开启*

要查看实时全局光照，请将我们测试场景中主灯的模式设置为 Realtime。既然我们没有其他光源，这样事实上等于关闭了烘焙光照，即使它已启用。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-light.png)  
*实时的主光源*  

确保场景中的所有对象都使用我们的白色材质。就像上次一样，球体都是动态的，其他一切物体都是静态。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-gi-dynamic-only.png)  
*只有动态物体接受实时光照*

事实证明，只有动态对象才能从实时全局光照中受益。静态物体会变得更暗，那是因为探针数据并入了实时光照贴图。所以静态对象也必须对实时光照贴图采样，而不仅是烘焙光照贴图，而我们的着色器还没有处理过实时光照贴图。

## 1.2 烘焙实时全局光照

Unity 在编辑模式下已生成实时光照贴图，因此您始终可以看到实时全局光照的效果。在编辑模式和播放模式之间切换时，这些图会重新生成，但内容是一样的。您可以切换到 Lighting 窗口的 Object maps 标签卡，选择一个静态对象，来检查它的实时光照贴图。选择 Realtime Intensity 以查看实时光照贴图数据。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-intensity.png)  
*实时光照贴图，所选物体是屋顶*

虽然实时光照贴图已经烘焙好了，并且它们看起来可能没什么问题，但我们的 meta pass 实际上使用的是错误的坐标。实时全局光照有自己的光照贴图坐标，和静态光照贴图的坐标不同。 Unity 根据光照贴图和物体的设置自动生成这些坐标，并存储在第三个 UV 通道中。 因此，将此参数添加到 My Lightmapping 中的 VertexData。

```c
struct VertexData {
	float4 vertex : POSITION;
	float2 uv : TEXCOORD0;
	float2 uv1 : TEXCOORD1;
	float2 uv2 : TEXCOORD2;
};
```

现在，MyLightmappingVertexProgram 必须使用第二个或第三个UV，来处理静态或动态光照贴图的缩放和偏移。我们可以使用 UnityMetaVertexPosition 函数来获取正确的数据。

```c
Interpolators MyLightmappingVertexProgram (VertexData v) {
	Interpolators i;
//	v.vertex.xy = v.uv1 * unity_LightmapST.xy + unity_LightmapST.zw;
//	v.vertex.z = v.vertex.z > 0 ? 0.0001 : 0;
//
//	i.pos = UnityObjectToClipPos(v.vertex);
	i.pos = UnityMetaVertexPosition(
		v.vertex, v.uv1, v.uv2, unity_LightmapST, unity_DynamicLightmapST
	);

	i.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
	i.uv.zw = TRANSFORM_TEX(v.uv, _DetailTex);
	return i;
}
```

> **UnityMetaVertexPosition 函数长什么样子？**
> 它做的就是我们以前做的事情，除了它使用 unity_MetaVertexControl 所提供的标记来决定使用哪套坐标和光照贴图。
```c
float4 UnityMetaVertexPosition (
	float4 vertex, float2 uv1, float2 uv2,
	float4 lightmapST, float4 dynlightmapST
) {
	if (unity_MetaVertexControl.x) {
		vertex.xy = uv1 * lightmapST.xy + lightmapST.zw;
		// OpenGL right now needs to actually use incoming vertex position,
		// so use it in a very dummy way
		vertex.z = vertex.z > 0 ? 1.0e-4f : 0.0f;
	}
	if (unity_MetaVertexControl.y) {
		vertex.xy = uv2 * dynlightmapST.xy + dynlightmapST.zw;
		// OpenGL right now needs to actually use incoming vertex position,
		// so use it in a very dummy way
		vertex.z = vertex.z > 0 ? 1.0e-4f : 0.0f;
	}
	return UnityObjectToClipPos(vertex);
}
```

请注意，meta pass 既用于烘焙也用于实时光照贴图。因此，当使用实时全局光照时，它也将包含在最终构建中。

## 1.3 采样实时光照贴图

为了采样实时光照贴图，My Lighting 中的 VertexData 也要添加第三套 UV。

```c
struct VertexData {
	float4 vertex : POSITION;
	float3 normal : NORMAL;
	float4 tangent : TANGENT;
	float2 uv : TEXCOORD0;
	float2 uv1 : TEXCOORD1;
	float2 uv2 : TEXCOORD2;
};
```

当使用实时光照贴图时，我们必须将其光照贴图坐标添加到插值器中。Unity 内置的标准着色器将两种光照贴图坐标集合在同一个插值器中——与其他一些数据复用——但我们可以为两者分别使用单独的插值器。 我们知道当定义了 DYNAMICLIGHTMAP_ON 关键字时会有动态光照数据。它属于 multi_compile_fwdbase 指令所代表的关键字列表的一部分。

```c
struct Interpolators {
	…

	#if defined(DYNAMICLIGHTMAP_ON)
		float2 dynamicLightmapUV : TEXCOORD7;
	#endif
};
```

像填充静态光照贴图坐标一样填充坐标，注意获取动态光照贴图的缩放和偏移是通过 unity_DynamicLightmapST 变量。

```c
Interpolators MyVertexProgram (VertexData v) {
	…

	#if defined(LIGHTMAP_ON) || ADDITIONAL_MASKED_DIRECTIONAL_SHADOWS
		i.lightmapUV = v.uv1 * unity_LightmapST.xy + unity_LightmapST.zw;
	#endif

	#if defined(DYNAMICLIGHTMAP_ON)
		i.dynamicLightmapUV =
			v.uv2 * unity_DynamicLightmapST.xy + unity_DynamicLightmapST.zw;
	#endif

	…
}
```

实时光照贴图的采样在 CreateIndirectLight 函数中完成。复制一下 #if defined(LIGHTMAP_ON) 代码块并做一些更改：首先，新块基于 DYNAMICLIGHTMAP_ON 关键字；其次，它应该使用 DecodeRealtimeLightmap 而非 DecodeLightmap，因为实时贴图使用的颜色格式不同。由于实时贴图可能会和烘焙照明叠加，因此不要直接赋值给 indirectLight.diffuse，而是使用一个中间变量以在最后加上去；最后，我们应该在既没使用烘焙也没使用实时光照贴图时采样球谐波。

```c
		#if defined(LIGHTMAP_ON)
			indirectLight.diffuse =
				DecodeLightmap(UNITY_SAMPLE_TEX2D(unity_Lightmap, i.lightmapUV));
			
			#if defined(DIRLIGHTMAP_COMBINED)
				float4 lightmapDirection = UNITY_SAMPLE_TEX2D_SAMPLER(
					unity_LightmapInd, unity_Lightmap, i.lightmapUV
				);
				indirectLight.diffuse = DecodeDirectionalLightmap(
					indirectLight.diffuse, lightmapDirection, i.normal
				);
			#endif

			ApplySubtractiveLighting(i, indirectLight);
//		#else
//			indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
		#endif

		#if defined(DYNAMICLIGHTMAP_ON)
			float3 dynamicLightDiffuse = DecodeRealtimeLightmap(
				UNITY_SAMPLE_TEX2D(unity_DynamicLightmap, i.dynamicLightmapUV)
			);

			#if defined(DIRLIGHTMAP_COMBINED)
				float4 dynamicLightmapDirection = UNITY_SAMPLE_TEX2D_SAMPLER(
					unity_DynamicDirectionality, unity_DynamicLightmap,
					i.dynamicLightmapUV
				);
            	indirectLight.diffuse += DecodeDirectionalLightmap(
            		dynamicLightDiffuse, dynamicLightmapDirection, i.normal
            	);
			#else
				indirectLight.diffuse += dynamicLightDiffuse;
			#endif
		#endif

		#if !defined(LIGHTMAP_ON) && !defined(DYNAMICLIGHTMAP_ON)
			indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
		#endif
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-gi-everything.png)  
*所有物体都接受实时光照*  

现在我们的着色器使用了实时光照贴图。如果使用了距离阴影遮罩，一开始它可能看起来与使用混合烘焙照明没什么区别，但是在播放模式下关闭光源时，差异就会特别明显。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/mixed-light-runtime-disabled.png)  
*关闭了混合光源之后间接光还在*  

混合光源关闭之后，间接光会保持不变。 相比之下，实时全局光照的间接光会消失—— 并重新出现——理论上应该如此。 但是，在烘焙完全结束之前可能需要一小段时间，Enlighten 系统会逐渐调整光照贴图和探针，调整完成的速度取决于场景的复杂程度和 Graphics Setting 里的 Realtime Global Illumination CPU 质量等级设置。

[*视频：开关实时全局光照下的实时灯光。*](https://thumbs.gfycat.com/DenseLinearCornsnake-mobile.mp4)  
 

所有实时光源都对实时全局光照有贡献。然而，它的典型用途仅是用于主方向光，代表天空中移动的太阳。它完全适用于所有方向光源。点光源和聚光灯也可以工作，但是不会产生阴影。因此，当使用带阴影的点光源或聚光灯时，会出现不正确的间接光照。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/spotlight-inspector.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-gi-spotlight.png)  
*实时聚光灯，间接光没有阴影。*  

## 1.4 自发光光源

实时全局光照还可用于自发光的静态对象。这使得可以通过实时间接光调整自发光效果。让我们来试试，向场景中添加一个静态球体，并为其提供一种材质，该材质使用具有黑色反照率和白色自发光的着色器。一开始，我们只能通过静态光照贴图看到自发光的间接光的影响。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/baked-emissive.png)  
*自发光球体烘焙全局光*  

要将自发光烘焙到静态光照贴图中，我们必须在着色器 GUI 中设置材质的全局照明标记。由于我们总是将标志设置为 BakedEmissive ，因此自发光会烘焙到光照贴图中。当自发光恒定时，这很好，但这样就不允许有任何修改。

为了同时支持自发光的烘焙和实时照明，我们必须使其可配置。 我们可以通过 MaterialEditor.LightmapEmissionProperty 方法为 MyLightingShaderGUI 添加一个选项来实现。方法的唯一参数是属性面板上的的缩进级别。

```c#  
	void DoEmission () {
		MaterialProperty map = FindProperty("_EmissionMap");
		Texture tex = map.textureValue;
		EditorGUI.BeginChangeCheck();
		editor.TexturePropertyWithHDRColor(
			MakeLabel(map, "Emission (RGB)"), map, FindProperty("_Emission"),
			emissionConfig, false
		);
		editor.LightmapEmissionProperty(2);
		if (EditorGUI.EndChangeCheck()) {
			if (tex != map.textureValue) {
				SetKeyword("_EMISSION_MAP", map.textureValue);
			}

			foreach (Material m in editor.targets) {
				m.globalIlluminationFlags =
					MaterialGlobalIlluminationFlags.BakedEmissive;
			}
		}
	}
```

每次自发光属性发生变化时，我们还必须注意取消覆盖标记。实际上的情况更为复杂。其中一个标记选项是 EmissiveIsBlack，它表示可以跳过自发光的计算。新材质默认设置此标记。为了使间接自发光工作，无论我们选择实时还是烘焙，我们都必须保证不要设置此标记。可以通过始终屏蔽标记值的 EmissiveIsBlack 位来实现此目的。

```c#
			foreach (Material m in editor.targets) {
				m.globalIlluminationFlags &=
					~MaterialGlobalIlluminationFlags.EmissiveIsBlack;
			}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-emissive-inspector.png) 
![](https://catlikecoding.com/unity/tutorials/rendering/part-18/realtime-global-illumination/realtime-emissive.png)  
*实时全局光照下的自发光球体*  

烘焙和实时全局光照之间可见的差异是实时光照贴图的分辨率通常低于烘焙贴图的分辨率。 因此，当不打算动态改变自发光并且仍然在使用烘焙光照时，请充分利用其分辨率更高的优点。

> **设计 EmissiveIsBlack 的目的是?**  
> 这是一个优化，可以跳过部分GI烘焙过程。但是，它是自发光颜色确实是黑色时才能设置的标记。 既然由着色器 GUI 负责设置，就要在通过 Inspector 编辑材质时确定，至少 Unity的标准着色器就是这样做的。因此，如果之后又通过脚本或动画系统更改了自发光颜色，标记不会自动修改。许多人不理解为什么他们的动画自发光不会影响实时全局光照，这就是原因之一。如果您想在运行时更改自发光，那么有一个技巧就是，不要将自发光颜色设置为纯黑色。
> 
> 我们不使用这种方法，而是使用 LightmapEmissionProperty，这样可以提供完全关闭自发光影响全局光的选项。此选项对于用户是含义明确的，没有任何隐藏的行为。 不想使用自发光？确保其全局光照设置为 None 就行了。

## 1.5 自发光动起来

自发光的实时全局光照仅适用于静态对象。虽然物体是静态的，但是它们的材质的自发光是可动的并且将由全局照明系统取用。让我们尝试一下在白色和黑色自发光之间振荡的简单组件。

```c#
using UnityEngine;

public class EmissiveOscillator : MonoBehaviour {

	Material emissiveMaterial;

	void Start () {
		emissiveMaterial = GetComponent<MeshRenderer>().material;
	}
	
	void Update () {
		Color c = Color.Lerp(
			Color.white, Color.black,
			Mathf.Sin(Time.time * Mathf.PI) * 0.5f + 0.5f
		);
		emissiveMaterial.SetColor("_Emission", c);
	}
}
```

将此组件添加到我们的自发光球体上。在播放模式下，其自发光将具有动画效果，但间接光尚未受到影响。我们必须通知实时全局光照系统它有工作要做，调用相应的 MeshRenderer 的 Renderer.UpdateGIMaterials 方法。

```c#
	MeshRenderer emissiveRenderer;
	Material emissiveMaterial;

	void Start () {
		emissiveRenderer = GetComponent<MeshRenderer>();
		emissiveMaterial = emissiveRenderer.material;
	}
	
	void Update () {
		…
		emissiveMaterial.SetColor("_Emission", c);
		emissiveRenderer.UpdateGIMaterials();
	}
```

[*视频：动态自发光的实时全局光照*](https://thumbs.gfycat.com/VillainousFlakyBlueandgoldmackaw-mobile.mp4)  
 

调用 UpdateGIMaterials 会触发物体自发光的完整更新，并使用其 meta pass 渲染它。 当自发光不仅是纯色时，这是很有必要的，比如使用纹理时。如果仅仅是纯色，我们可以使用一个快捷方法，调用 DynamicGI.SetEmissive，参数是自发光颜色，这比使用 meta pass 渲染更快，因此在可以的时候请尽可能利用它的优点。

```c#
//		emissiveRenderer.UpdateGIMaterials();
		DynamicGI.SetEmissive(emissiveRenderer, c);
```

# 2 光照探针代理体

烘焙和实时光照影响动态物体都要通过光照探针。根据物体的位置对探针数据插值，然后用于GI计算。这种方法适用于小物体，但对于较大的物体来说太粗糙了。

例如，将一个拉长的立方体添加到测试场景中，使其受到不同光照的影响。它应该使用我们的白色材质。由于它是一个动态的立方体，最终会使用单个点来确定其GI贡献。移动移动它的位置，使这个点被遮蔽，此时整个立方体都会变暗，显然这是错误的。为了使错误明显，我使用了烘焙主光源，所以所有照明都来自烘焙和实时GI数据。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/large-object.png)  
*较大的动态物体，错误的光照*

为了使光照探针适用于这种情况，我们可以使用光照探针代理体(Light Probe Proxy Volume，LPPV)。它会为着色器提供插值的探针网格而非单个点的数据。它需要线性过滤的浮点3D纹理支持——只有较新的显卡才有的功能。除此之外，还要在图像等级设置中启用LPPV支持。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/support-lppv.png)  
*开启LPPV*  

## 2.1 为物体增加一个LPPV

LPPV 可以以各种方式设置，最直接的是作为使用它物体的组件添加。您可以通过 *Component/Rendering/Light Probe Proxy Volume* 添加。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/lppv-inspector.png)  
*LPPV 组件*  

运行时，LPPV 会在光照探针之间插值，就像是一般动态物体的网格一样。内插值被缓存起来，Refresh Mode 控制缓存何时更新。 默认值为 Automatic ，意味着在动态GI更改和探针组移动时会发生更新。Bounding Box Mode 控制代理体的定位方式。 Automatic Local  意味着它填充到它所附加的物体的边界框。这些默认设置很适用于我们的立方体，因此我们保留它们。

要让我们的立方体实际使用 LPPV，我们必须将其 MeshRenderer 的 Light Probes 模式设置为 Use Proxy Volume。其默认行为是使用对象本身的 LPPV 组件，但您也可以强制它使用另一个代理体。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/mesh-renderer-settings.png)  
*使用代理体而非普通的探针*  

自动分辨率模式不适用于我们的拉长立方体。因此，将 Resolution Mode 设置为 Custom，并确保在立方体的角处有采样点，沿着长边要布置多个采样点。选择物体后可以看到这些采样点。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/custom-resolution.png)  
![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/lppv-gizmos.png)  
*自定义探针分辨率以适合拉长的立方体*  

## 2.2 采样代理体

立方体变黑了，因为我们的着色器尚不支持采样 LPPV。为了使它正确工作，我们必须调整 CreateIndirectLight 内的球谐函数部分代码。UNITY_LIGHT_PROBE_PROXY_VOLUME 定义为1时，标识着启用了 LPPV。在这种情况下我们先什么也不做，看看会发生什么。

```c
		#if !defined(LIGHTMAP_ON) && !defined(DYNAMICLIGHTMAP_ON)
			#if UNITY_LIGHT_PROBE_PROXY_VOLUME
			#else
				indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
			#endif
		#endif
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/no-more-sh.png)  
*没有球谐函数了*  

事实证明，所有球谐函数都被禁用了，对于没有使用 LPPV 的动态物体也是如此。这是因为 UNITY_LIGHT_PROBE_PROXY_VOLUME 是在项目范围内定义的，而不是针对每个物体定义的。单个对象是否使用 LPPV 由 UnityShaderVariables 中定义的 unity_ProbeVolumeParams 的 x分量指示。如果它是 1，那么物体有 LPPV，否则我们应该使用常规球谐函数。

```c
			#if UNITY_LIGHT_PROBE_PROXY_VOLUME
				if (unity_ProbeVolumeParams.x == 1) {
				}
				else {
					indirectLight.diffuse +=
						max(0, ShadeSH9(float4(i.normal, 1)));
				}
			#else
				indirectLight.diffuse += max(0, ShadeSH9(float4(i.normal, 1)));
			#endif
```

要对代理体采样，我们应该使用 SHEvalLinearL0L1_SampleProbeVolume 函数代替 ShadeSH9。此函数在 UnityCG 中定义，需要世界坐标作为额外参数。

```c
				if (unity_ProbeVolumeParams.x == 1) {
					indirectLight.diffuse = SHEvalLinearL0L1_SampleProbeVolume(
						float4(i.normal, 1), i.worldPos
					);
					indirectLight.diffuse = max(0, indirectLight.diffuse);
				}
```

> **SHEvalLinearL0L1_SampleProbeVolume 是如何工作的**
> 顾名思义，此函数仅包括前两个球谐频段——L0 和 L1。Unity 不会将第三个频段用于 LPPV。因此，我们得到了质量较低的光照近似值，但我们在多个世界空间采样之间进行插值，而不是使用单个点。这是代码：
```c
half3 SHEvalLinearL0L1_SampleProbeVolume (half4 normal, float3 worldPos) {
	const float transformToLocal = unity_ProbeVolumeParams.y;
	const float texelSizeX = unity_ProbeVolumeParams.z;

	//The SH coefficients textures and probe occlusion
	// are packed into 1 atlas.
	//-------------------------
	//| ShR | ShG | ShB | Occ |
	//-------------------------

	float3 position = (transformToLocal == 1.0f) ?
		mul(unity_ProbeVolumeWorldToObject, float4(worldPos, 1.0)).xyz :
		worldPos;
	float3 texCoord = (position - unity_ProbeVolumeMin.xyz) *
		unity_ProbeVolumeSizeInv.xyz;
	texCoord.x = texCoord.x * 0.25f;

	// We need to compute proper X coordinate to sample. Clamp the
	// coordinate otherwize we'll have leaking between RGB coefficients
	float texCoordX =
		clamp(texCoord.x, 0.5f * texelSizeX, 0.25f - 0.5f * texelSizeX);

	// sampler state comes from SHr (all SH textures share the same sampler)
	texCoord.x = texCoordX;
	half4 SHAr = UNITY_SAMPLE_TEX3D_SAMPLER(
		unity_ProbeVolumeSH, unity_ProbeVolumeSH, texCoord
	);
	texCoord.x = texCoordX + 0.25f;
	half4 SHAg = UNITY_SAMPLE_TEX3D_SAMPLER(
		unity_ProbeVolumeSH, unity_ProbeVolumeSH, texCoord
	);
	texCoord.x = texCoordX + 0.5f;
	half4 SHAb = UNITY_SAMPLE_TEX3D_SAMPLER(
		unity_ProbeVolumeSH, unity_ProbeVolumeSH, texCoord
	);
	// Linear + constant polynomial terms
	half3 x1;
	x1.r = dot(SHAr, normal);
	x1.g = dot(SHAg, normal);
	x1.b = dot(SHAb, normal);

	return x1;
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/lppv-too-dark.png)  
*LPPV 采样，在伽马空间过于暗*  

我们的着色器现在根据需要对 LPPV 进行采样，但结果太暗。至少，在伽马色彩空间中工作时就是这种情况。这是因为球谐函数数据存储在线性空间中。因此可能需要进行色彩空间转换。

```c
				if (unity_ProbeVolumeParams.x == 1) {
					indirectLight.diffuse = SHEvalLinearL0L1_SampleProbeVolume(
						float4(i.normal, 1), i.worldPos
					);
					indirectLight.diffuse = max(0, indirectLight.diffuse);
					#if defined(UNITY_COLORSPACE_GAMMA)
			            indirectLight.diffuse =
			            	LinearToGammaSpace(indirectLight.diffuse);
			        #endif
				}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/light-probe-proxy-volumes/lppv-correct.png)  
*LPPV 采样，颜色正确*  


# 3 LOD 组

当一个物体最终只覆盖应用程序窗口的一小部分时，您渲染它不需要过于细致的模型。您可以根据对象的视图大小使用不同的模型。这被称为细节级别(level of detail，LOD)。Unity 允许我们通过 LOD Group 组件执行此操作。

## 3.1 创建 LOD 层级

问题的核心是在不同的细节级别使用同一模型的多个版本。最高级别——LOD 0——具有最多的顶点、子对象、动画、复杂材质等。后面的级别会变得越来越简单，渲染也越来越便宜。理想情况下，Unity 切换相邻 LOD 级别时，用户应当无法轻易看出差别，否则突然的改变是很明显的。但在研究这种技术时，我们会使用明显不同的模型。

创建一个空的 GameObject 并给它添加两个子物体。第一个是标准球体，第二个是标准立方体，各轴缩放均设置为 0.75。正如预期，结果看起来像是球体和立方体重叠。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/lod-object-scene.png)![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/lod-object-hierarchy.png)  
*把球和立方体作为一个物体*

通过 Component/Rendering/LOD Group 将 LOD Group 组件添加到父物体。您会获得一个具有默认设置的 LOD 组，具有三个 LOD 级别。这里的百分比是指物体边界框所覆盖的部分的高度占窗口总高度的比。因此，默认情况下，当垂直高度下降到窗口高度的 60％ 时切换到 LOD 1，当减小到30％时，切换到 LOD 2。 当低于 10％ 时，根本不会被渲染。您可以通过拖动 LOD 框的边缘来更改这些阈值。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/lod-component.png)  
*LOD Group 组件*  

实际的阈值可能被 LOD Bias 修改，如组件的 Inspector 上提到的。LOD Bias 是项目 Quality 设置的一个参数，默认值为 2，这意味着阈值实际上减半。最大 LOD 级别也可以设置，从而跳过最高的若干级别。

要使其工作，您必须告诉组件每个 LOD 级别使用哪些物体，方法是选择 LOD 块并将物体添加到其 Renderers 列表。虽然您可以添加场景中的任何对象，但一般只添加其子对象。将球体用于 LOD 0，将立方体用于 LOD 1。我们将 LOD 2 留空，因此实际上只有两个 LOD 级别。如果需要，可以通过右键上下文菜单删除和插入 LOD 级别。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/lod-0-inspector.png)  
*把球体添加到 LOD 0*  

配置LOD级别后，您可以通过移动相机来查看它们的运行情况。如果物体足够大，会显示为球体，否则会显示为立方体或根本不会显示。


[运行时的 LOD 切换](https://thumbs.gfycat.com/ShyAffectionateFairyfly-mobile.mp4)  

## 3.2 烘焙全局光照和 LOD 组

因为LOD组的渲染方式取决于其视图大小，所以它们自然是动态的。但是，您仍然可以将它们设置为静态。把整个物体层次结构设为静态，包括根物体和两个子对象，然后将主光源设置为烘焙，看看会发生什么。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/baked-lighting.png)  
*使用烘焙光*

看起来烘焙静态光照贴图时总是使用 LOD 0。我们最终总是看到球体对阴影和间接光的贡献，即使 LOD 组切换到立方体或剔除级别。但请注意，立方体既然使用静态光照贴图，就不使用光照探针了，对吧？关掉探针组试试。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/baked-lighting-without-probes.png)  
*不带探针的烘焙光*  

禁用探针组会使立方体变暗。这意味着他们不再接收间接光。这是因为在烘焙过程中确定间接光时使用 LOD 0。要找到其他 LOD 层级的间接光，Unity 最好的办法就是依靠烘焙的光照探针。因此，即使我们在运行时不需要光照探针，我们也需要它，以为我们的立方体提供间接光。

## 3.3 实时全局光照和 LOD 组

当只使用实时全局光照时，方法类似，只有一点：现在我们的立方体在运行时使用光照探针。您可以选择球体或立方体来验证一下。选择立方体后，您可以看到辅助线，显示使用哪种光照探针。球体不显示，因为它们使用动态光照贴图。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/realtime-gi-lod-1-probes.png)  
*LOD 1 在实时全局光照下使用探针*  

当烘焙和实时全局光照同时使用时，情况会变得更加复杂。在这种情况下，立方体应该使用光照贴图获取烘焙全局光照，使用探针以获得实时全局光照。不幸的是，这是不可能的，因为光照贴图和球谐函数不能同时使用，只能选一个。由于光照贴图数据也可用于立方体，因此Unity最终选择了它。因此，立方体不受实时全局光照的影响。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/lod-1-only-baked.png)  
*LOD 1 只有烘焙光照，这里使用了较暗的主光源*  

一个重要的细节是 LOD 层级的烘焙和渲染是完全独立的，不需要使用完全相同的设置。如果最终实时光照比烘焙光照更重要，您可以强制立方体使用光照探针，把立方体标记为光照贴图非静态，同时保持球体还是静态。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/lod-1-forced-probes.png)  
*LOD 1 强制使用光照探针*

## 3.4 LOD 层级之间的交叉淡入淡出

LOD 组的缺点是，当 LOD 级别发生变化时，视觉上的突变非常明显，物体突然出现、消失或改变形状。这种突变可以通过相邻 LOD 级别之间的交叉淡化来减轻，方法是将组的淡入淡出模式设置为交叉渐变。另外还有一种淡入淡出模式，Unity 将其用于 SpeedTree 对象，我们不会使用。

启用交叉渐变时，每个 LOD 级别会显示 Fade Transition Width 字段，该字段控制其块有多大比例用于淡入淡出。例如，当设置为 0.5 时，LOD 的一半范围都用于淡入下一层级。或者可以对淡入进行动画处理，在这种情况下，LOD 层级切换需要大约半秒钟。


![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/cross-fade-inspector.png)  
*0.5 宽度的交叉淡入*

如果启用交叉渐变，在组切换时，会同时渲染两个 LOD 层级的内容。

## 3.5 支持交叉渐变

Unity 的标准着色器默认情况下不支持交叉淡入淡出。您必须复制标准着色器并为 LOD_FADE_CROSSFADE 关键字添加 multi-compile 指令。我们还需要在 My First Lighting 着色器添加该指令以支持交叉淡入淡出。将指令添加到除 meta pass 之外的所有 pass。。。

```c
#pragma multi_compile _ LOD_FADE_CROSSFADE
```

我们在 LOD 级别之间转换时会使用抖动。该方法适用于前向渲染和延迟渲染，也适用于阴影。

在创建半透明阴影时，我们已经使用过抖动。它需要片段的屏幕空间坐标，这迫使我们为顶点和片段程序使用不同的插值器结构。 因此，让我们复制 My Lighting 中的 Interpolators 结构，将其重命名为 InterpolatorsVertex。

```c
struct InterpolatorsVertex {
	…
};

struct Interpolators {
	…
};

…

InterpolatorsVertex MyVertexProgram (VertexData v) {
	InterpolatorsVertex i;
	…
}
```

当我们必须交叉淡入淡出时，片段程序的插值器必须包含 vpos，否则我们可以保持通常的位置。

```c
struct Interpolators {
	#if defined(LOD_FADE_CROSSFADE)
		UNITY_VPOS_TYPE vpos : VPOS;
	#else
		float4 pos : SV_POSITION;
	#endif

	…
};
```

我们可以在片段程序开始时使用 UnityApplyDitherCrossFade 函数来执行交叉渐变。

```c
FragmentOutput MyFragmentProgram (Interpolators i) {
	#if defined(LOD_FADE_CROSSFADE)
		UnityApplyDitherCrossFade(i.vpos);
	#endif

	…
}
```

> **UnityApplyDitherCrossFade 是如何工作的？**
> 这个函数定义在 UnityCG 中。它类似于我们在第十二节半透明阴影中所使用的抖动，除了抖动级别对于整个物体是均匀的。因此，不需要在抖动级别之间进行混合。它使用16个抖动级别，存储在 4×64 的 2D 纹理中，而不是 4×4×16 的 3D 纹理。
```c
sampler2D _DitherMaskLOD2D;

void UnityApplyDitherCrossFade(float2 vpos) {
	vpos /= 4; // the dither mask texture is 4x4
	// quantized lod fade by 16 levels
	vpos.y = frac(vpos.y) * 0.0625 /* 1/16 */ + unity_LODFade.y;
	clip(tex2D(_DitherMaskLOD2D, vpos).a - 0.5);
}
```
> unity_LODFade 变量定义在 UnityShaderVariables 中。其 Y 分量包含对象的淡入淡出量，分十六个级别。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/dithering-geometry.png)  
*通过抖动淡入淡出物体*

交叉渐变现在可以用于几何体了。为了使它适用于阴影，我们必须调整 My Shadows。首先，当我们交叉淡入时，必须使用 vpos。其次，我们还必须在片段程序的开头使用 UnityApplyDitherCrossFade。

```c
struct Interpolators {
	#if SHADOWS_SEMITRANSPARENT || defined(LOD_FADE_CROSSFADE)
		UNITY_VPOS_TYPE vpos : VPOS;
	#else
		float4 positions : SV_POSITION;
	#endif

	…
};

…

float4 MyShadowFragmentProgram (Interpolators i) : SV_TARGET {
	#if defined(LOD_FADE_CROSSFADE)
		UnityApplyDitherCrossFade(i.vpos);
	#endif

	…
}
```

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/lod-groups/dithering-shadows.png)  
[视频](https://thumbs.gfycat.com/AcceptableClosedAmurstarfish-mobile.mp4) 
*同时对物体和阴影进行交叉淡入*

因为立方体和球体相互交叉，所以我们在它们之间交叉渐变时，得到一些奇怪的自阴影。我们这样可以很方便地看到阴影之间的交叉渐变有效，但在为实际游戏创建 LOD 物体时，必须注意这些瑕疵。

下一节是 GPU Instancing

---
[原教程传送门]（https://catlikecoding.com/unity/tutorials/rendering/part-18/）