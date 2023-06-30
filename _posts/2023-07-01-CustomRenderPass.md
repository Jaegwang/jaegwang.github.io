---
title: 언리얼엔진에서 커스텀 쉐이더 적용
date: 2023-07-01 00:03:20 +0900
categories: [UnrealEngine]
tags: [programming, unreal engine]
---

언리얼엔진에서 커스텀 쉐이더 코드를 적용하는 방법을 시도하고자 합니다.

일반적으로 **FMeshPassProcessor::Process**에서 어떤 쉐이더를 사용할지 결정하게 됩니다.

``` cpp
template<bool bPositionOnly>
bool FCustomDepthPassMeshProcessor::Process(
	const FMeshBatch& RESTRICT MeshBatch,
	uint64 BatchElementMask,
	int32 StaticMeshId,
	const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,
	const FMaterialRenderProxy& RESTRICT MaterialRenderProxy,
	const FMaterial& RESTRICT MaterialResource,
	ERasterizerFillMode MeshFillMode,
	ERasterizerCullMode MeshCullMode)
{
	const FVertexFactory* VertexFactory = MeshBatch.VertexFactory;

	TMeshProcessorShaders<
		TDepthOnlyVS<bPositionOnly>,
		FDepthOnlyPS> DepthPassShaders;

	FShaderPipelineRef ShaderPipeline;
	if (!GetDepthPassShaders<bPositionOnly>(
		MaterialResource,
		VertexFactory->GetType(),
		FeatureLevel,
		MaterialResource.MaterialUsesPixelDepthOffset_RenderThread(),
		DepthPassShaders.VertexShader,
		DepthPassShaders.PixelShader,
		ShaderPipeline
		))
	{
		return false;
	}

	FMeshMaterialShaderElementData ShaderElementData;
	ShaderElementData.InitializeMeshMaterialData(ViewIfDynamicMeshCommand, PrimitiveSceneProxy, MeshBatch, StaticMeshId, false);

	const FMeshDrawCommandSortKey SortKey = CalculateMeshStaticSortKey(DepthPassShaders.VertexShader, DepthPassShaders.PixelShader);

	BuildMeshDrawCommands(
		MeshBatch,
		BatchElementMask,
		PrimitiveSceneProxy,
		MaterialRenderProxy,
		MaterialResource,
		PassDrawRenderState,
		DepthPassShaders,
		MeshFillMode,
		MeshCullMode,
		SortKey,
		bPositionOnly ? EMeshPassFeatures::PositionOnly : EMeshPassFeatures::Default,
		ShaderElementData);

	return true;
}
```

Reference
---
- [(UE4 4.27) UE4는 맞춤형 MeshPass를 추가하여 모바일 단말기의 에지 글로우를 구현합니다.](https://blog.csdn.net/qq_29523119/article/details/121026939?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2~default~BlogCommendFromBaidu~Rate-1-121026939-blog-100702560.235%5Ev38%5Epc_relevant_default_base&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2~default~BlogCommendFromBaidu~Rate-1-121026939-blog-100702560.235%5Ev38%5Epc_relevant_default_base&utm_relevant_index=1)
- [(UE4 4.27) 커스텀 PrimitiveComponent](https://blog.csdn.net/qq_29523119/article/details/120806384?spm=1001.2014.3001.5501)
- [언리얼 4 렌더링 프로그래밍(셰이더) [볼륨 13: 나만의 MeshDrawPass 사용자 정의]](https://blog.csdn.net/cpongo11/article/details/100702560?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-100702560-blog-121026939.235%5Ev38%5Epc_relevant_default_base&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-100702560-blog-121026939.235%5Ev38%5Epc_relevant_default_base&utm_relevant_index=2)
- [언리얼 엔진에 커스텀 패스 추가](https://zhuanlan.zhihu.com/p/91506412)