---
title: Unreal Engine Basis
categories: Unreal Engine
---

# Reflection(反射)

虚幻反射系统是虚幻引擎的基础技术之一，并且支撑着许多其他苏童的运行比如编辑器中的细节面板、序列化、垃圾回收、网络复制和蓝图与C++的通讯。虚幻反射系统会收集、查询和操作有关C++的类、结构、函数、类成员函数、和枚举。

虚幻反射系统的使用是可选的。

## Usage

头文件中包含下列语句可以被标记为该翻译单元（文件/类）拥有需要被反射的类型，加入虚幻反射系统
`include ”<FileName>.generated.h”`
使用下列的宏语句来标记需要被加入至反射系统的变量。
`UENUM()`, `UCLASS()`, `USTRUCT()`, `UFUNCTION()`, `UPROPERTY()`

## Work Flow

Unreal Build Tool(UBT)和Unreal Header Tool(UHT)协同工作，生成运行虚幻反射系统所必须的数据。UBT负责扫描所有的头文件，标记所有任意头文件具有至少一种反射类型的模块。UHT负责收集和更新虚幻反射系统所必须的数据，遍历所有的头文件，构建虚幻反射系统数据，生成包含反射数据的C++代码。

UHT是独立的程序，不依赖任何的反射数据文件，避免先有鸡还是先有蛋的问题。

## .generated.h content

类声明或者结构体声明中的GENERATED_UCLASS_BODY()或GENERATED_USTRCUT_BODY()生成一些函数，生成的函数像StaticClass()或者StaticStruct()这种便于从蓝图或者C++获取反射数据的函数，这些内容必须作为成员存在于需要被反射的类型中。

# Delegate (委托)

委托是一种可定义的事件，可以调用或者回应。

## Multicast Delegate(多播委托)

在代码中，任意数量的实体都可以回应该多播委托，并从该委托中接受输入数据和使用它们。

## Dynamic Delegate(动态委托)

动态委托可以在蓝图中被保存或者被加载，在蓝图系统中，它们也叫事件分发器。

## Event Dispatcher(事件分发器)

## Implementation

* Engine\Source\Runtime\Core\Public\Delegates\DelegateCombinations.h
 
```Cpp
#define DECLARE_DELEGATE_OneParam( DelegateName, Param1Type ) FUNC_DECLARE_DELEGATE( DelegateName, void, Param1Type )
```

* Engine\Source\Runtime\Core\Public\Delegates\Delegate.h

```Cpp
#define FUNC_DECLARE_DELEGATE( DelegateName, Return Type, ... ) \
typedef TDelegate<ReturnType(__VA_ARGS__)> DelegateName
//TDelegate模板类类名，被转换成别名DelegateName
Tdelegate<ReturnType(__VA_ARGS__)> DelegateVariableName = DelegateName DelegateVariableName
```

* Engine\Source\Runtime\Core\Public\Delegates\DelegateSignatureImpl.inl

```Cpp
template <typename InRetValType, typename... ParamTypes, typename UserPolicy>
class TDelegate<InRetValType(ParamTypes...), UserPolicy> : public TDelegateBase<UserPolicy>
/*
UserPolicy不是参数，默认为FDefaultDelegateUserPolicy，可能在需要修改委托系统时注意。模板参数接受函数返回值类型(InRetValType，可为void)、参数类型(ParamTypes)和参数数量(...)。
在该类的内部还有BindUObject、CreateUObject、Execute的定义，BindUObject不过是将CreateUObject封装了，全部皆为内联函数。
*/
```

# Memory Management(内存管理)

## Garbage Collection(垃圾回收)
垃圾回收系统会追踪所有的UObject及其子类的对象，包括AActor和UActorComponent。当创建新的UObject对象时，虚幻引擎会自动地将新UObject对象添加至内部UObject对象列表，即便有不合理地使用，也不会轻易地造成内存泄露，但是很容易造成程序崩溃。

注意：

所有的UObject对象不应使用标准C++的new，只能使用默认的创建函数NewObject、SpawnActor、CreateDefaultSubobject。

UObject对象满足一下条件之一，会一直保持存活：

1. 通过其他对象对该对象进行强引用
2. 通过其他对象对该对象调用UObject::AddReferenceObjects
3. 调用UObject::AddToRoot添加他们至根集合，这个一般没有必要。

未满足以上条件之一的对象，会在下一轮垃圾回收周期被标记为无法访问并且加入垃圾回收队列。

可通过调用MarkPendingKillu或者MarkAsGarbage在对象上，强制下一个垃圾回收周期摧毁对象。部分类不支持该方式销毁，比如AActor和UActorComponent。

根据以上的原理，防止对象被回收的办法：标记为UPROPERTY，调用AddRoot函数。

# Serialization(序列化)

一般是指将数据结构转换成其他格式方便传输/存储，提取其中的数据结构也就叫反序列化(Deserialization)。

# 性能优化相关

## 一点经验积累

1. 更应该使用FName作为Hash key而非FString，FString的Hash Function每一次都要遍历字符串进行Hash运算，O(n)时间复杂度，而FName在构造时就已经完成了Hash运算并且存储到全局表中，且自身只由索引和数字后缀组成，在查找时只用现成的索引和数字组成的键进行查找，O(1)时间复杂度。

2. 由于Unreal Engine 5拥抱大世界，所有计算都从float转变为double，大幅提高了CPU的计算负担，在移动平台上的开销提高。

3. 使用数学计算处理逻辑会比碰撞检测开销低得多。

4. 应尽量避免使用网格体碰撞，尽量采用简单碰撞体。

# Render

## 渲染相关线程

RenderThread, RHIThread

## 渲染管线（流程）

Unreal Engine使用延迟渲染方案(Deferred Rendering)，并且渲染依赖图(Render Dependency Graph)进行任务调度和资源管理。

大概如下

1. 预渲染和剔除

1.1. 遮挡剔除

检查物体是否有被遮挡。

1.2. 视锥体剔除

检查物体是否在摄像机视线内。

1.3. 提前加载部分深度信息

1.4. 如果启用Nanite，则执行Nanite专用的剔除办法，将场景细分为网格簇(Cluster)，在GPU上进行剔除。(Nanite VisBuffer Pass, VisBuffer)

2. 填充Geometry Buffer(G-Buffer，物体的物理属性)

把物体的各种属性渲染到几张不同的贴图(Render Target)。

* 底色(Base Color)

* 法线(Normal)

* 材质属性，金属度(Roughness)、(Metallic)

* 深度(Depth)，物体与摄像机的距离

如果启用Nanite，则需要将剔除和预渲染时记录下的Nanite几何体信息解码并写入G-Bufer。

1. 阴影处理

Unreal Engine 5 使用虚拟阴影贴图(Virtual Shadow Maps)，采用SMRT(Shadow Map Ray Tracing)，类似光线追踪的采样算法。

注：虚拟阴影贴图导致胶囊体阴影无法使用(UE5.3)，而不使用Virtual Shadow Maps则无法使用Nanite。

4. 直接光照、间接光照处理

间接与全局光照，Lumen是UE5采用基于光线追踪的实时全局光照。(Lumen Scene Card, Lighting Pass, Reflection)

5. 光照整合与后处理

将直接光、间接光、阴影整合。

半透明物体无法存在于G-Buffer，在处理完光照和阴影后单独渲染，由远到近。

后处理(Temporal Super Resolution)

## 具体实现(施工中)

* Engine\Source\Runtime\Renderer\Private\DeferredShadingRenderer.cpp

```Cpp
void FDeferredShadingSceneRenderer::Render(FRDGBuilder& GraphBuilder, const FSceneRenderUpdateInputs* SceneUpdateInputs)
{
	{
		FRayTracingVisualizationData& RayTracingVisualizationData = GetRayTracingVisualizationData();

		if (RayTracingVisualizationData.HasOverrides())
		{
			// When activating the view modes from the command line, automatically enable the RayTracingDebug show flag for convenience.
			ViewFamily.EngineShowFlags.SetRayTracingDebug(true);
		}
	}

	// If this is scene capture rendering depth pre-pass, we'll take the shortcut function RenderSceneCaptureDepth if optimization switch is on.
	const ERendererOutput RendererOutput = GetRendererOutput();

	const bool bNaniteEnabled = ShouldRenderNanite();
	const bool bHasRayTracedOverlay = HasRayTracedOverlay(ViewFamily);

#if !UE_BUILD_SHIPPING
	RenderCaptureInterface::FScopedCapture RenderCapture(GCaptureNextDeferredShadingRendererFrame-- == 0, GraphBuilder, TEXT("DeferredShadingSceneRenderer"));
	// Prevent overflow every 2B frames.
	GCaptureNextDeferredShadingRendererFrame = FMath::Max(-1, GCaptureNextDeferredShadingRendererFrame);
#endif

	GPU_MESSAGE_SCOPE(GraphBuilder);

#if RHI_RAYTRACING
	if (SceneUpdateInputs && RendererOutput == FSceneRenderer::ERendererOutput::FinalSceneColor)
	{
		GRayTracingGeometryManager->PreRender();

		// TODO: should only process build requests once per frame
		RHI_BREADCRUMB_EVENT_STAT(GraphBuilder.RHICmdList, RayTracingGeometry, "RayTracingGeometry");
		SCOPED_GPU_STAT(GraphBuilder.RHICmdList, RayTracingGeometry);

		GRayTracingGeometryManager->ProcessBuildRequests(GraphBuilder.RHICmdList);
	}

	FRayTracingShaderBindingTable& RayTracingSBT = Scene->RayTracingSBT;
	FRayTracingScene& RayTracingScene = Scene->RayTracingScene;
	RayTracingSBT.ResetMissAndCallableShaders();

	for (FViewInfo& View : Views)
	{
		if (IStereoRendering::IsStereoEyeView(View) && IStereoRendering::IsASecondaryView(View))
		{
			continue;
		}

		View.SetRayTracingSceneViewHandle(RayTracingScene.AddView(View.GetViewKey()));
		RayTracingScene.SetViewParams(View.GetRayTracingSceneViewHandle(), View.ViewMatrices, View.RayTracingCullingParameters);
	}
#endif

	FInitViewTaskDatas InitViewTaskDatas = OnRenderBegin(GraphBuilder, SceneUpdateInputs);

	FUpdateExposureCompensationCurveLUTTaskData UpdateExposureCompensationCurveLUTTaskData;
	BeginUpdateExposureCompensationCurveLUT(Views, &UpdateExposureCompensationCurveLUTTaskData);

	FRDGExternalAccessQueue ExternalAccessQueue;
	TUniquePtr<FVirtualTextureUpdater> VirtualTextureUpdater;
	FLumenSceneFrameTemporaries LumenFrameTemporaries(Views);

	FGPUSceneScopeBeginEndHelper GPUSceneScopeBeginEndHelper(GraphBuilder, Scene->GPUScene, GPUSceneDynamicContext);

	const bool bUseVirtualTexturing = UseVirtualTexturing(ShaderPlatform);

	// Virtual texturing isn't needed for depth prepass
	if (bUseVirtualTexturing && RendererOutput != ERendererOutput::DepthPrepassOnly)
	{
		FVirtualTextureUpdateSettings Settings;
		Settings.EnableThrottling(!ViewFamily.bOverrideVirtualTextureThrottle);

		VirtualTextureUpdater = FVirtualTextureSystem::Get().BeginUpdate(GraphBuilder, FeatureLevel, this, Settings);
		VirtualTextureFeedbackBegin(GraphBuilder, Views, GetActiveSceneTexturesConfig().Extent);
	}

	if (SceneUpdateInputs)
	{
		{
			TRACE_CPUPROFILER_EVENT_SCOPE(CommitFinalPipelineState);
			for (FSceneRenderer* Renderer : SceneUpdateInputs->Renderers)
			{
				// Compute & commit the final state of the entire dependency topology of the renderer.
				static_cast<FDeferredShadingSceneRenderer*>(Renderer)->CommitFinalPipelineState();
			}
		}

		// Initialize global system textures (pass-through if already initialized).
		GSystemTextures.InitializeTextures(GraphBuilder.RHICmdList, FeatureLevel);
	}

	UE::Tasks::TTask<void> UpdateLightFunctionAtlasTask;
	if (LightFunctionAtlas.IsLightFunctionAtlasEnabled())
	{
		UpdateLightFunctionAtlasTask = LaunchSceneRenderTask<void>(TEXT("UpdateLightFunctionAtlas"), [this]
			{
				UpdateLightFunctionAtlasTaskFunction();
			}, UE::Tasks::FTask());
	}

	FShadowSceneRenderer& ShadowSceneRenderer = GetSceneExtensionsRenderers().GetRenderer<FShadowSceneRenderer>();
	{
		if (RendererOutput == ERendererOutput::FinalSceneColor)
		{
			// 1. Update sky atmosphere
			// This needs to be done prior to start Lumen scene lighting to ensure directional light color is correct, as the sun color needs atmosphere transmittance
			{
				const bool bPathTracedAtmosphere = ViewFamily.EngineShowFlags.PathTracing && Views.Num() > 0 && PathTracing::UsesReferenceAtmosphere(Views[0]);
				if (ShouldRenderSkyAtmosphere(Scene, ViewFamily.EngineShowFlags) && !bPathTracedAtmosphere)
				{
					for (int32 LightIndex = 0; LightIndex < NUM_ATMOSPHERE_LIGHTS; ++LightIndex)
					{
						if (Scene->AtmosphereLights[LightIndex])
						{
							PrepareSunLightProxy(*Scene->GetSkyAtmosphereSceneInfo(),LightIndex, *Scene->AtmosphereLights[LightIndex]);
						}
					}
				}
				else
				{
					Scene->ResetAtmosphereLightsProperties();
				}
			}

			// 2. Update lumen scene
			{
				InitViewTaskDatas.LumenFrameTemporaries = &LumenFrameTemporaries;
	
				// Important that this uses consistent logic throughout the frame, so evaluate once and pass in the flag from here
				// NOTE: Must be done after  system texture initialization
				// TODO: This doesn't take into account the potential for split screen views with separate shadow caches
				const bool bEnableVirtualShadowMaps = UseVirtualShadowMaps(ShaderPlatform, FeatureLevel) && ViewFamily.EngineShowFlags.DynamicShadows && !bHasRayTracedOverlay;
				VirtualShadowMapArray.Initialize(GraphBuilder, Scene->GetVirtualShadowMapCache(), bEnableVirtualShadowMaps, ViewFamily.EngineShowFlags);
	
				if (InitViewTaskDatas.LumenFrameTemporaries)
				{
					BeginUpdateLumenSceneTasks(GraphBuilder, *InitViewTaskDatas.LumenFrameTemporaries);
				}
	
				BeginGatherLumenLights(*InitViewTaskDatas.LumenFrameTemporaries, InitViewTaskDatas.LumenDirectLighting, InitViewTaskDatas.VisibilityTaskData, UpdateLightFunctionAtlasTask);
			}
		}

		if (bNaniteEnabled)
		{
			TArray<FConvexVolume, TInlineAllocator<2>> NaniteCullingViews;

			// For now we'll share the same visibility results across all views
			for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
			{
				FViewInfo& View = Views[ViewIndex];
				NaniteCullingViews.Add(View.ViewFrustum);
			}

			FNaniteVisibility& NaniteVisibility = Scene->NaniteVisibility[ENaniteMeshPass::BasePass];
			const FNaniteRasterPipelines&  NaniteRasterPipelines  = Scene->NaniteRasterPipelines[ENaniteMeshPass::BasePass];
			const FNaniteShadingPipelines& NaniteShadingPipelines = Scene->NaniteShadingPipelines[ENaniteMeshPass::BasePass];

			NaniteVisibility.BeginVisibilityFrame();

			NaniteBasePassVisibility.Visibility = &NaniteVisibility;
			NaniteBasePassVisibility.Query = NaniteVisibility.BeginVisibilityQuery(
				Allocator,
				*Scene,
				NaniteCullingViews,
				&NaniteRasterPipelines,
				&NaniteShadingPipelines,
				InitViewTaskDatas.VisibilityTaskData->GetComputeRelevanceTask()
			);
		}
	}
	ShaderPrint::BeginViews(GraphBuilder, Views);

	ON_SCOPE_EXIT
	{
		ShaderPrint::EndViews(Views);
	};

	GetSceneExtensionsRenderers().PreInitViews(GraphBuilder);

	if (RendererOutput == ERendererOutput::FinalSceneColor)
	{
		if (SceneUpdateInputs)
		{
			PrepareDistanceFieldScene(GraphBuilder, ExternalAccessQueue, *SceneUpdateInputs);
		}

		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
		{
			FViewInfo& View = Views[ViewIndex];
			RDG_GPU_MASK_SCOPE(GraphBuilder, View.GPUMask);

			ShadingEnergyConservation::Init(GraphBuilder, View);

			FGlintShadingLUTsStateData::Init(GraphBuilder, View);
		}

#if RHI_RAYTRACING
		if (FamilyPipelineState[&FFamilyPipelineState::bRayTracing])
		{
			for (FViewInfo& View : Views)
			{
				if (View.ViewState != nullptr)
				{
					if (View.ViewState->Scene == nullptr)
					{
						// link view state to the scene
						View.ViewState->Scene = Scene;
						Scene->ViewStates.Add(View.ViewState);
					}
				}
			}

			InitViewTaskDatas.RayTracingGatherInstances = RayTracing::CreateGatherInstancesTaskData(Allocator, *Scene, Views.Num());

			for (FViewInfo& View : Views)
			{
				const FPerViewPipelineState& ViewPipelineState = GetViewPipelineState(View);

				RayTracing::AddView(*InitViewTaskDatas.RayTracingGatherInstances, View, ViewPipelineState.DiffuseIndirectMethod, ViewPipelineState.ReflectionsMethod);
			}

			RayTracing::BeginGatherInstances(*InitViewTaskDatas.RayTracingGatherInstances, InitViewTaskDatas.VisibilityTaskData->GetFrustumCullTask());
		}
#endif
	}

	UE::SVT::GetStreamingManager().BeginAsyncUpdate(GraphBuilder);

	bool bVisualizeNanite = false;
	if (bNaniteEnabled)
	{
		Nanite::GGlobalResources.Update(GraphBuilder);
		Nanite::GStreamingManager.BeginAsyncUpdate(GraphBuilder);

		FNaniteVisualizationData& NaniteVisualization = GetNaniteVisualizationData();
		if (Views.Num() > 0)
		{
			FName NaniteViewMode = Views[0].CurrentNaniteVisualizationMode;
			
			EDebugViewShaderMode DebugViewShaderMode = ViewFamily.GetDebugViewShaderMode();
			if (DebugViewShaderMode == DVSM_ShadowCasters)
			{
				NaniteViewMode = FName("ShadowCasters");
				ViewFamily.EngineShowFlags.SetVisualizeNanite(true);
			}

			if (NaniteVisualization.Update(NaniteViewMode))
			{
				// When activating the view modes from the command line, automatically enable the VisualizeNanite show flag for convenience.
				ViewFamily.EngineShowFlags.SetVisualizeNanite(true);
			}

			bVisualizeNanite = NaniteVisualization.IsActive() && ViewFamily.EngineShowFlags.VisualizeNanite;
		}
	}

	CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderOther);

	SCOPED_NAMED_EVENT(FDeferredShadingSceneRenderer_Render, FColor::Emerald);

#if WITH_MGPU
	ComputeGPUMasks(&GraphBuilder.RHICmdList);
#endif // WITH_MGPU

	// By default, limit our GPU usage to only GPUs specified in the view masks.
	RDG_GPU_MASK_SCOPE(GraphBuilder, ViewFamily.EngineShowFlags.PathTracing ? FRHIGPUMask::All() : AllViewsGPUMask);
	RDG_EVENT_SCOPE(GraphBuilder, "Scene");
	RDG_GPU_STAT_SCOPE_VERBOSE(GraphBuilder, Unaccounted, *ViewFamily.ProfileDescription);
	
	if (RendererOutput == ERendererOutput::FinalSceneColor)
	{
		SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_Render_Init);
		RDG_RHI_GPU_STAT_SCOPE(GraphBuilder, AllocateRendertargets);

		// Force the subsurface profiles and specular profiles textures to be updated.
		SubsurfaceProfile::UpdateSubsurfaceProfileTexture(GraphBuilder, ShaderPlatform);
		SpecularProfile::UpdateSpecularProfileTextureAtlas(GraphBuilder, ShaderPlatform);

		// Force the rect light texture & IES texture to be updated.
		RectLightAtlas::UpdateAtlasTexture(GraphBuilder, FeatureLevel);
		IESAtlas::UpdateAtlasTexture(GraphBuilder, ShaderPlatform);
	}

	FSceneTexturesConfig& SceneTexturesConfig = GetActiveSceneTexturesConfig();
	const FRDGSystemTextures& SystemTextures = FRDGSystemTextures::Create(GraphBuilder);

	const bool bAllowStaticLighting = !bHasRayTracedOverlay && IsStaticLightingAllowed();

	// if DDM_AllOpaqueNoVelocity was used, then velocity should have already been rendered as well
	const bool bIsEarlyDepthComplete = (DepthPass.EarlyZPassMode == DDM_AllOpaque || DepthPass.EarlyZPassMode == DDM_AllOpaqueNoVelocity);

	// Use read-only depth in the base pass if we have a full depth prepass.
	const bool bAllowReadOnlyDepthBasePass = bIsEarlyDepthComplete
		&& !ViewFamily.EngineShowFlags.ShaderComplexity
		&& !ViewFamily.UseDebugViewPS()
		&& !ViewFamily.EngineShowFlags.Wireframe
		&& !ViewFamily.EngineShowFlags.LightMapDensity;

	const FExclusiveDepthStencil::Type BasePassDepthStencilAccess =
		bAllowReadOnlyDepthBasePass
		? FExclusiveDepthStencil::DepthRead_StencilWrite
		: FExclusiveDepthStencil::DepthWrite_StencilWrite;

	FRendererViewDataManager& ViewDataManager = *GraphBuilder.AllocObject<FRendererViewDataManager>(GraphBuilder, *Scene, GetSceneUniforms(), AllViews);
	FInstanceCullingManager& InstanceCullingManager = *GraphBuilder.AllocObject<FInstanceCullingManager>(GraphBuilder, *Scene, GetSceneUniforms(), ViewDataManager);

	::Substrate::PreInitViews(*Scene);

	FSceneTextures::InitializeViewFamily(GraphBuilder, ViewFamily, FamilySize);
	FSceneTextures& SceneTextures = GetActiveSceneTextures();

	{
		RDG_EVENT_SCOPE_STAT(GraphBuilder, VisibilityCommands, "VisibilityCommands");
		RDG_GPU_STAT_SCOPE(GraphBuilder, VisibilityCommands);
		BeginInitViews(GraphBuilder, SceneTexturesConfig, InstanceCullingManager, ExternalAccessQueue, InitViewTaskDatas);
	}

#if !UE_BUILD_SHIPPING
	if (CVarStallInitViews.GetValueOnRenderThread() > 0.0f)
	{
		SCOPE_CYCLE_COUNTER(STAT_InitViews_Intentional_Stall);
		FPlatformProcess::Sleep(CVarStallInitViews.GetValueOnRenderThread() / 1000.0f);
	}
#endif

	extern TSet<IPersistentViewUniformBufferExtension*> PersistentViewUniformBufferExtensions;

	for (IPersistentViewUniformBufferExtension* Extension : PersistentViewUniformBufferExtensions)
	{
		Extension->BeginFrame();

		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
		{
			// Must happen before RHI thread flush so any tasks we dispatch here can land in the idle gap during the flush
			Extension->PrepareView(&Views[ViewIndex]);
		}
	}

	if (RendererOutput == ERendererOutput::FinalSceneColor)
	{
		// Prepare the scene for rendering this frame.

#if RHI_RAYTRACING
		if (ViewFamily.EngineShowFlags.PathTracing)
		{
			if (ShouldPrepareRayTracingDecals(*Scene, ViewFamily))
			{
				// Calculate decal grid for ray tracing per view since decal fade is view dependent
				// TODO: investigate reusing the same grid for all views (ie: different callable shader SBT entries for each view so fade alpha is still correct for each view)

				for (FViewInfo& View : Views)
				{
					View.RayTracingDecalUniformBuffer = CreateRayTracingDecalData(GraphBuilder, *Scene, View, RayTracingSBT.NumCallableShaderSlots);
					View.bHasRayTracingDecals = true;
					RayTracingSBT.NumCallableShaderSlots += Scene->Decals.Num();
				}
			}
			else
			{
				TRDGUniformBufferRef<FRayTracingDecals> NullRayTracingDecalUniformBuffer = CreateNullRayTracingDecalsUniformBuffer(GraphBuilder);

				for (FViewInfo& View : Views)
				{
					View.RayTracingDecalUniformBuffer = NullRayTracingDecalUniformBuffer;
					View.bHasRayTracingDecals = false;
				}
			}

			// If we might be path tracing the clouds -- call the path tracer's method for cloud callable shader setup
			// this will skip work if cloud rendering is not being used
			PreparePathTracingCloudMaterial(GraphBuilder, Scene, Views);
		}

		if (IsRayTracingEnabled(ViewFamily.GetShaderPlatform()) && ShouldCompileRayTracingShadersForProject(ViewFamily.GetShaderPlatform()))
		{
			if (!ViewFamily.EngineShowFlags.PathTracing)
			{
				// get the default lighting miss shader (to implicitly fill in the MissShader library before the RT pipeline is created)
				GetRayTracingLightingMissShader(GetGlobalShaderMap(FeatureLevel));
				RayTracingSBT.NumMissShaderSlots++;
			}

			if (ViewFamily.EngineShowFlags.LightFunctions)
			{
				// gather all the light functions that may be used (and also count how many miss shaders we will need)
				FRayTracingLightFunctionMap RayTracingLightFunctionMap;
				if (ViewFamily.EngineShowFlags.PathTracing)
				{
					RayTracingLightFunctionMap = GatherLightFunctionLightsPathTracing(Scene, ViewFamily.EngineShowFlags, FeatureLevel);
				}
				else
				{
					RayTracingLightFunctionMap = GatherLightFunctionLights(Scene, ViewFamily.EngineShowFlags, FeatureLevel);
				}
				if (!RayTracingLightFunctionMap.IsEmpty())
				{
					// If we got some light functions in our map, store them in the RDG blackboard so downstream functions can use them.
					// The map itself will be strictly read-only from this point on.
					GraphBuilder.Blackboard.Create<FRayTracingLightFunctionMap>(MoveTemp(RayTracingLightFunctionMap));
				}
			}
		}
#endif // RHI_RAYTRACING

#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
		Scene->DebugRender(Views);
#endif
	}

	InitViewTaskDatas.VisibilityTaskData->FinishGatherDynamicMeshElements(BasePassDepthStencilAccess, InstanceCullingManager, VirtualTextureUpdater.Get());

	// Notify the FX system that the scene is about to be rendered.
	// TODO: These should probably be moved to scene extensions
	if (FXSystem && Views.IsValidIndex(0))
	{
		SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_FXSystem_PreRender);
		const bool bAllowGPUParticleUpdate = IsHeadLink();
		FXSystem->PreRender(GraphBuilder, GetSceneViews(), GetSceneUniforms(), bAllowGPUParticleUpdate);
		if (FGPUSortManager* GPUSortManager = FXSystem->GetGPUSortManager())
		{
			GPUSortManager->OnPreRender(GraphBuilder);
		}
	}

	{
		RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, UpdateGPUScene);
		RDG_EVENT_SCOPE_STAT(GraphBuilder, GPUSceneUpdate, "GPUSceneUpdate");
		RDG_GPU_STAT_SCOPE(GraphBuilder, GPUSceneUpdate);

		for (int32 ViewIndex = 0; ViewIndex < AllViews.Num(); ViewIndex++)
		{
			FViewInfo& View = *AllViews[ViewIndex];
			RDG_GPU_MASK_SCOPE(GraphBuilder, View.GPUMask);

			Scene->GPUScene.UploadDynamicPrimitiveShaderDataForView(GraphBuilder, View);
			Scene->GPUScene.DebugRender(GraphBuilder, GetSceneUniforms(), View);
		}

		// Must be called after all views have flushed the dynamic primitives.
		ViewDataManager.InitInstanceState(GraphBuilder);

		if (Views.Num() > 0)
		{
			FViewInfo& View = Views[0];
			Scene->UpdatePhysicsField(GraphBuilder, View);
		}
	}


	if (FSceneCullingRenderer* SceneCullingRenderer = GetSceneExtensionsRenderers().GetRendererPtr<FSceneCullingRenderer>())
	{
		SceneCullingRenderer->DebugRender(GraphBuilder, Views);
	}

	//SceneCullingInfo.SceneUniformBuffer = GetSceneUniforms().GetBuffer(GraphBuilder);
	GetSceneExtensionsRenderers().UpdateViewData(GraphBuilder, ViewDataManager);

	// Allow scene extensions to affect the scene uniform buffer after GPU scene has fully updated
	GetSceneExtensionsRenderers().UpdateSceneUniformBuffer(GraphBuilder, GetSceneUniforms());

	// Must happen after visibility state & scene UB has been updated.
	InstanceCullingManager.BeginDeferredCulling(GraphBuilder);

	const bool bUseGBuffer = IsUsingGBuffers(ShaderPlatform);
	const bool bShouldRenderVolumetricFog = ShouldRenderVolumetricFog();
	const bool bShouldRenderLocalFogVolume = ShouldRenderLocalFogVolume(Scene, ViewFamily);
	const bool bShouldRenderLocalFogVolumeDuringHeightFogPass = ShouldRenderLocalFogVolumeDuringHeightFogPass(Scene, ViewFamily);
	const bool bShouldRenderLocalFogVolumeInVolumetricFog = ShouldRenderLocalFogVolumeInVolumetricFog(Scene, ViewFamily, bShouldRenderLocalFogVolume);
	const bool bShouldRenderLocalFogVolumeVisualizationPass = ShouldRenderLocalFogVolumeVisualizationPass(Scene, ViewFamily);

	const bool bRenderDeferredLighting = ViewFamily.EngineShowFlags.Lighting
		&& FeatureLevel >= ERHIFeatureLevel::SM5
		&& ViewFamily.EngineShowFlags.DeferredLighting
		&& bUseGBuffer
		&& !bHasRayTracedOverlay;

	bool bAnyLumenEnabled = false;

	// Virtual texturing isn't needed for depth prepass
	if (bUseVirtualTexturing && RendererOutput != ERendererOutput::DepthPrepassOnly)
	{
		// Note, should happen after the GPU-Scene update to ensure rendering to runtime virtual textures is using the correctly updated scene
		FVirtualTextureSystem::Get().EndUpdate(GraphBuilder, MoveTemp(VirtualTextureUpdater), FeatureLevel);
	}

	FMaterialCacheTagProvider::Get().Update(GraphBuilder);

	UE::Tasks::TTask<FSortedLightSetSceneInfo*> GatherAndSortLightsTask;

	if (RendererOutput == ERendererOutput::FinalSceneColor)
	{
#if RHI_RAYTRACING
		if (FamilyPipelineState[&FFamilyPipelineState::bRayTracing])
		{
			RayTracing::FinishGatherInstances(
				GraphBuilder,
				*InitViewTaskDatas.RayTracingGatherInstances,
				RayTracingScene,
				RayTracingSBT,
				DynamicReadBufferForRayTracing,
				Allocator);
		}
#endif // RHI_RAYTRACING

		if (!bHasRayTracedOverlay)
		{
			for (const FViewInfo& View : Views)
			{
				bAnyLumenEnabled = bAnyLumenEnabled
					|| GetViewPipelineState(View).DiffuseIndirectMethod == EDiffuseIndirectMethod::Lumen
					|| GetViewPipelineState(View).ReflectionsMethod == EReflectionsMethod::Lumen;
			}
		}

		{
			extern bool IsVSMOnePassProjectionEnabled(const FEngineShowFlags& ShowFlags);
			extern UE::Tasks::FTask GetGatherAndSortLightsPrerequisiteTask(const FDynamicShadowsTaskData* TaskData);

			auto* SortedLightSet = GraphBuilder.AllocObject<FSortedLightSetSceneInfo>();
			const bool bShadowedLightsInClustered = ShouldUseClusteredDeferredShading(ViewFamily.GetShaderPlatform())
				&& IsVSMOnePassProjectionEnabled(ViewFamily.EngineShowFlags)
				&& VirtualShadowMapArray.IsEnabled();

			TArray<UE::Tasks::FTask, TInlineAllocator<2>> IssuedTasksCompletionEvents;
			IssuedTasksCompletionEvents.Add(GetGatherAndSortLightsPrerequisiteTask(InitViewTaskDatas.DynamicShadows));
			IssuedTasksCompletionEvents.Add(UpdateLightFunctionAtlasTask);

			GatherAndSortLightsTask = LaunchSceneRenderTask<FSortedLightSetSceneInfo*>(UE_SOURCE_LOCATION, [this, SortedLightSet, bShadowedLightsInClustered]
			{
				GatherAndSortLights(*SortedLightSet, bShadowedLightsInClustered);
				return SortedLightSet;
			}, IssuedTasksCompletionEvents);
		}
	}

	// force using occ queries for wireframe if rendering is parented or frozen in the first view
	check(Views.Num());
	#if (UE_BUILD_SHIPPING || UE_BUILD_TEST)
		const bool bIsViewFrozen = false;
	#else
		const bool bIsViewFrozen = Views[0].State && ((FSceneViewState*)Views[0].State)->bIsFrozen;
	#endif

	
	const bool bIsOcclusionTesting = DoOcclusionQueries()
		&& (!ViewFamily.EngineShowFlags.Wireframe || bIsViewFrozen);
	const bool bNeedsPrePass = ShouldRenderPrePass();

	// Sanity check - Note: Nanite forces a Z prepass in ShouldForceFullDepthPass()
	check(!UseNanite(ShaderPlatform) || bNeedsPrePass);

	GetSceneExtensionsRenderers().PreRender(GraphBuilder);
	GEngine->GetPreRenderDelegateEx().Broadcast(GraphBuilder);

	if (DepthPass.IsComputeStencilDitherEnabled())
	{
		AddDitheredStencilFillPass(GraphBuilder, Views, SceneTextures.Depth.Target, DepthPass);
	}

	if (bNaniteEnabled)
	{
		// Must happen before any Nanite rendering in the frame
		Nanite::GStreamingManager.EndAsyncUpdate(GraphBuilder);

		const TMap<uint32, uint32> ModifiedResources = Nanite::GStreamingManager.GetAndClearModifiedResources();
#if RHI_RAYTRACING
		Nanite::GRayTracingManager.RequestUpdates(ModifiedResources);
#endif
	}

	// Virtual texturing isn't needed for depth prepass
	if (bUseVirtualTexturing && RendererOutput != ERendererOutput::DepthPrepassOnly)
	{
		FVirtualTextureSystem::Get().FinalizeRequests(GraphBuilder, this);
	}

	{
		RDG_RHI_GPU_STAT_SCOPE(GraphBuilder, VisibilityCommands);
		EndInitViews(GraphBuilder, LumenFrameTemporaries, InstanceCullingManager, InitViewTaskDatas);
	}

	// Substrate initialisation is always run even when not enabled.
	// Need to run after EndInitViews() to ensure ViewRelevance computation are completed
	const bool bSubstrateEnabled = Substrate::IsSubstrateEnabled();
	Substrate::InitialiseSubstrateFrameSceneData(GraphBuilder, *this);

	UE::SVT::GetStreamingManager().EndAsyncUpdate(GraphBuilder);

	FHairStrandsBookmarkParameters& HairStrandsBookmarkParameters = *GraphBuilder.AllocObject<FHairStrandsBookmarkParameters>();
	if (IsHairStrandsEnabled(EHairStrandsShaderType::All, Scene->GetShaderPlatform()) && RendererOutput == ERendererOutput::FinalSceneColor)
	{
		CreateHairStrandsBookmarkParameters(Scene, Views, AllViews, HairStrandsBookmarkParameters);
		check(Scene->HairStrandsSceneData.TransientResources);
		HairStrandsBookmarkParameters.TransientResources = Scene->HairStrandsSceneData.TransientResources;
		RunHairStrandsBookmark(GraphBuilder, EHairStrandsBookmark::ProcessTasks, HairStrandsBookmarkParameters);

		// Interpolation needs to happen after the skin cache run as there is a dependency 
		// on the skin cache output.
		const bool bRunHairStrands = HairStrandsBookmarkParameters.HasInstances() && (Views.Num() > 0);
		if (bRunHairStrands)
		{
			RunHairStrandsBookmark(GraphBuilder, EHairStrandsBookmark::ProcessCardsAndMeshesInterpolation_PrimaryView, HairStrandsBookmarkParameters);
		}
		else
		{
			for (FViewInfo& View : Views)
			{
				View.HairStrandsViewData.UniformBuffer = HairStrands::CreateDefaultHairStrandsViewUniformBuffer(GraphBuilder, View);
			}
		}
	}

	ExternalAccessQueue.Submit(GraphBuilder);

	const bool bShouldRenderSkyAtmosphere = ShouldRenderSkyAtmosphere(Scene, ViewFamily.EngineShowFlags);
	const ESkyAtmospherePassLocation SkyAtmospherePassLocation = GetSkyAtmospherePassLocation();
	FSkyAtmospherePendingRDGResources SkyAtmospherePendingRDGResources;
	if (SkyAtmospherePassLocation == ESkyAtmospherePassLocation::BeforePrePass && bShouldRenderSkyAtmosphere)
	{
		// Generate the Sky/Atmosphere look up tables overlaping the pre-pass
		RenderSkyAtmosphereLookUpTables(GraphBuilder, /* out */ SkyAtmospherePendingRDGResources);
	}

	RenderWaterInfoTexture(GraphBuilder, *this, Scene);

	const bool bShouldRenderVelocities = ShouldRenderVelocities();
	const EShaderPlatform Platform = GetViewFamilyInfo(Views).GetShaderPlatform();
	const bool bBasePassCanOutputVelocity = FVelocityRendering::BasePassCanOutputVelocity(Platform);
	const bool bHairStrandsEnable = HairStrandsBookmarkParameters.HasInstances() && Views.Num() > 0 && IsHairStrandsEnabled(EHairStrandsShaderType::Strands, Platform);
	const bool bForceVelocityOutput = bHairStrandsEnable || ShouldRenderDistortion();

	auto RenderPrepassAndVelocity = [&](auto& InViews, auto& InNaniteBasePassVisibility, auto& NaniteRasterResults, auto& PrimaryNaniteViews, FSceneTextures& LocalSceneTextures)
	{
		FRDGTextureRef FirstStageDepthBuffer = nullptr;
		{
			// Both compute approaches run earlier, so skip clearing stencil here, just load existing.
			const ERenderTargetLoadAction StencilLoadAction = DepthPass.IsComputeStencilDitherEnabled()
				? ERenderTargetLoadAction::ELoad
				: ERenderTargetLoadAction::EClear;

			const ERenderTargetLoadAction DepthLoadAction = ERenderTargetLoadAction::EClear;
			AddClearDepthStencilPass(GraphBuilder, LocalSceneTextures.Depth.Target, DepthLoadAction, StencilLoadAction);

			// Draw the scene pre-pass / early z pass, populating the scene depth buffer and HiZ
			if (bNeedsPrePass)
			{
				RenderPrePass(GraphBuilder, InViews, LocalSceneTextures.Depth.Target, InstanceCullingManager, &FirstStageDepthBuffer);
			}
			else
			{
				// We didn't do the prepass, but we still want the HMD mask if there is one
				RenderPrePassHMD(GraphBuilder, InViews, LocalSceneTextures.Depth.Target);
			}

			// special pass for DDM_AllOpaqueNoVelocity, which uses the velocity pass to finish the early depth pass write
			if (bShouldRenderVelocities && Scene->EarlyZPassMode == DDM_AllOpaqueNoVelocity && RendererOutput == ERendererOutput::FinalSceneColor)
			{
				// Render the velocities of movable objects.  Don't bind the velocity render target for custom render passes (it's not used downstream), to avoid needing to clear it again.
				RenderVelocities(GraphBuilder, InViews, LocalSceneTextures, EVelocityPass::Opaque, bForceVelocityOutput, /*bBindRenderTarget=*/ InViews[0].CustomRenderPass == nullptr);
			}
		}

		{
			Scene->WaitForCacheNaniteMaterialBinsTask();

			if (bNaniteEnabled && InViews.Num() > 0)
			{
				RenderNanite(GraphBuilder, InViews, LocalSceneTextures, bIsEarlyDepthComplete, InNaniteBasePassVisibility, NaniteRasterResults, PrimaryNaniteViews, FirstStageDepthBuffer);
			}
		}

		if (FirstStageDepthBuffer)
		{
			LocalSceneTextures.PartialDepth = FirstStageDepthBuffer;
			AddResolveSceneDepthPass(GraphBuilder, InViews, LocalSceneTextures.PartialDepth);
		}
		else
		{
			// Setup default partial depth to be scene depth so that it also works on transparent emitter when partial depth has not been generated.
			LocalSceneTextures.PartialDepth = LocalSceneTextures.Depth;
		}
		LocalSceneTextures.SetupMode = ESceneTextureSetupMode::SceneDepth;
		LocalSceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &LocalSceneTextures, FeatureLevel, LocalSceneTextures.SetupMode);

		AddResolveSceneDepthPass(GraphBuilder, InViews, LocalSceneTextures.Depth);
	};

	FDBufferTextures DBufferTextures = CreateDBufferTextures(GraphBuilder, SceneTextures.Config.Extent, ShaderPlatform);

	// Initialise local fog volume with dummy data before volumetric cloud view initialization (further down) which can bind LFV data.
	// Also need to do this before custom render passes (included in AllViews), as base pass rendering may bind LFV data.
	SetDummyLocalFogVolumeForViews(GraphBuilder, AllViews);

	if (CustomRenderPassInfos.Num() > 0)
	{
		QUICK_SCOPE_CYCLE_COUNTER(STAT_CustomRenderPasses);
		RDG_EVENT_SCOPE_STAT(GraphBuilder, CustomRenderPasses, "CustomRenderPasses");
		RDG_GPU_STAT_SCOPE(GraphBuilder, CustomRenderPasses);

		// If the main view family has MSAA enabled, initialize and use the separate non-MSAA version of the FSceneTextures stored in the first
		// custom render pass (also used by the other custom render passes).
		FSceneTextures* CustomRenderPassSceneTextures = &SceneTextures;
		if (SceneTextures.Config.NumSamples > 1)
		{
			FSceneTextures::InitializeViewFamily(GraphBuilder, CustomRenderPassInfos[0].ViewFamily, FamilySize);
			CustomRenderPassSceneTextures = const_cast<FSceneTextures*>(&CustomRenderPassInfos[0].ViewFamily.GetSceneTextures());
			
			// Make sure a separate FSceneTextures structure was allocated in the FSceneRenderer constructor when custom render passes were initialized!
			check(CustomRenderPassSceneTextures != &SceneTextures);
		}

		// We want to reset the scene texture uniform buffer to its original state after custom render passes,
		// so they can't affect downstream rendering.
		ESceneTextureSetupMode OriginalSceneTextureSetupMode = CustomRenderPassSceneTextures->SetupMode;
		TRDGUniformBufferRef<FSceneTextureUniformParameters> OriginalSceneTextureUniformBuffer = CustomRenderPassSceneTextures->UniformBuffer;

		for (int32 i = 0; i < CustomRenderPassInfos.Num(); ++i)
		{
			FCustomRenderPassBase* CustomRenderPass = CustomRenderPassInfos[i].CustomRenderPass;
			TArray<FViewInfo>& CustomRenderPassViews = CustomRenderPassInfos[i].Views;
			FNaniteShadingCommands& NaniteBasePassShadingCommands = CustomRenderPassInfos[i].NaniteBasePassShadingCommands;
			check(CustomRenderPass);

			CustomRenderPass->BeginPass(GraphBuilder);

			{
				QUICK_SCOPE_CYCLE_COUNTER(STAT_CustomRenderPass);
				RDG_EVENT_SCOPE(GraphBuilder, "CustomRenderPass[%d] %s", i, *CustomRenderPass->GetDebugName());

				CustomRenderPass->PreRender(GraphBuilder);

				TArray<Nanite::FRasterResults, TInlineAllocator<2>> NaniteRasterResults;
				TArray<Nanite::FPackedView, SceneRenderingAllocator> PrimaryNaniteViews;
				FNaniteBasePassVisibility DummyNaniteBasePassVisibility;
				RenderPrepassAndVelocity(CustomRenderPassViews, DummyNaniteBasePassVisibility, NaniteRasterResults, PrimaryNaniteViews, *CustomRenderPassSceneTextures);

				const FSingleLayerWaterPrePassResult* SingleLayerWaterPrePassResult = nullptr;
				if (ShouldRenderSingleLayerWaterDepthPrepass(CustomRenderPassViews))
				{
					SingleLayerWaterPrePassResult = RenderSingleLayerWaterDepthPrepass(GraphBuilder, CustomRenderPassViews, *CustomRenderPassSceneTextures, ESingleLayerWaterPrepassLocation::BeforeBasePass, NaniteRasterResults);
				}

				const FSceneCaptureCustomRenderPassUserData& SceneCaptureUserData = FSceneCaptureCustomRenderPassUserData::Get(CustomRenderPass);

				if (CustomRenderPass->GetRenderMode() == FCustomRenderPassBase::ERenderMode::DepthAndBasePass)
				{
					CustomRenderPassSceneTextures->SetupMode |= ESceneTextureSetupMode::SceneColor;
					CustomRenderPassSceneTextures->UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, CustomRenderPassSceneTextures, FeatureLevel, CustomRenderPassSceneTextures->SetupMode);

					if (bNaniteEnabled)
					{
						Nanite::BuildShadingCommands(GraphBuilder, *Scene, ENaniteMeshPass::BasePass, NaniteBasePassShadingCommands, Nanite::EBuildShadingCommandsMode::Custom);
					}

					RenderBasePass(*this, GraphBuilder, CustomRenderPassViews, *CustomRenderPassSceneTextures, DBufferTextures, BasePassDepthStencilAccess, /*ForwardScreenSpaceShadowMaskTexture=*/nullptr, InstanceCullingManager, bNaniteEnabled, NaniteBasePassShadingCommands, NaniteRasterResults);

					if (ShouldRenderSingleLayerWater(CustomRenderPassViews))
					{
						// GBuffer code paths in RenderSingleLayerWater don't use the bIsCameraUnderWater flag, so just pass in false.  Normally this is
						// computed by a render extension, but those aren't run for custom render passes.
						FSceneWithoutWaterTextures SceneWithoutWaterTextures;
						RenderSingleLayerWater(GraphBuilder, CustomRenderPassViews, *CustomRenderPassSceneTextures, SingleLayerWaterPrePassResult, /*bShouldRenderVolumetricCloud=*/false, SceneWithoutWaterTextures, LumenFrameTemporaries, /*bIsCameraUnderWater=*/false);
					}

					FCustomRenderPassBase::ERenderOutput RenderOutput = CustomRenderPass->GetRenderOutput();
					if (RenderOutput == FCustomRenderPassBase::ERenderOutput::BaseColor || RenderOutput == FCustomRenderPassBase::ERenderOutput::Normal ||
						!SceneCaptureUserData.UserSceneTextureBaseColor.IsNone() || !SceneCaptureUserData.UserSceneTextureNormal.IsNone() || !SceneCaptureUserData.UserSceneTextureSceneColor.IsNone())
					{
						// CopySceneCaptureComponentToTarget uses scene texture uniforms
						CustomRenderPassSceneTextures->SetupMode |= ESceneTextureSetupMode::GBuffers;
						CustomRenderPassSceneTextures->UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, CustomRenderPassSceneTextures, FeatureLevel, CustomRenderPassSceneTextures->SetupMode);
					}

					if (CustomRenderPass->IsTranslucentIncluded())
					{
						// Empty defaults
						FTranslucencyLightingVolumeTextures TranslucencyLightingVolumeTextures;
						FTranslucencyPassResourcesMap TranslucencyResourceMap(CustomRenderPassViews.Num());
						const bool bStandardTranslucentCanRenderSeparate = false;
						FRDGTextureMSAA TranslucencySharedDepthTexture;
						FSeparateTranslucencyDimensions CustomTranslucencyDimensions = { SceneTexturesConfig.Extent };

						FReflectionCaptureShaderData EmptyData;
						TUniformBufferRef<FReflectionCaptureShaderData> EmptyReflectionCaptureUniformBuffer = TUniformBufferRef<FReflectionCaptureShaderData>::CreateUniformBufferImmediate(EmptyData, UniformBuffer_SingleFrame);
						for (FViewInfo& View : CustomRenderPassViews)
						{
							View.ReflectionCaptureUniformBuffer = EmptyReflectionCaptureUniformBuffer;
						}

						RenderTranslucency(*this, GraphBuilder, *CustomRenderPassSceneTextures, TranslucencyLightingVolumeTextures, &TranslucencyResourceMap, CustomRenderPassViews, ETranslucencyView::AboveWater, CustomTranslucencyDimensions, InstanceCullingManager, bStandardTranslucentCanRenderSeparate, TranslucencySharedDepthTexture);
					}
				}

				CopySceneCaptureComponentToTarget(GraphBuilder, *CustomRenderPassSceneTextures, CustomRenderPass->GetRenderTargetTexture(), ViewFamily, CustomRenderPassViews);

				if (!SceneCaptureUserData.UserSceneTextureBaseColor.IsNone())
				{
					// User Scene Textures are stored to "SceneTextures" for downstream use, not in "CustomRenderPassSceneTextures", only used during Custom Render Pass rendering
					bool bFirstRender;
					FRDGTextureRef BaseColorSceneTexture = SceneTextures.FindOrAddUserSceneTexture(GraphBuilder, 0, SceneCaptureUserData.UserSceneTextureBaseColor, SceneCaptureUserData.SceneTextureDivisor, bFirstRender, nullptr, CustomRenderPassViews[0].ViewRect);
#if !(UE_BUILD_SHIPPING)
					SceneTextures.UserSceneTextureEvents.Add({ EUserSceneTextureEvent::CustomRenderPass, NAME_None, (uint16)FCustomRenderPassBase::ERenderOutput::BaseColor, 0, (const UMaterialInterface*)CustomRenderPass });
#endif

					CustomRenderPass->OverrideRenderOutput(FCustomRenderPassBase::ERenderOutput::BaseColor);
					CopySceneCaptureComponentToTarget(GraphBuilder, *CustomRenderPassSceneTextures, BaseColorSceneTexture, ViewFamily, CustomRenderPassViews);
				}

				if (!SceneCaptureUserData.UserSceneTextureNormal.IsNone())
				{
					bool bFirstRender;
					FRDGTextureRef NormalSceneTexture = SceneTextures.FindOrAddUserSceneTexture(GraphBuilder, 0, SceneCaptureUserData.UserSceneTextureNormal, SceneCaptureUserData.SceneTextureDivisor, bFirstRender, nullptr, CustomRenderPassViews[0].ViewRect);
#if !(UE_BUILD_SHIPPING)
					SceneTextures.UserSceneTextureEvents.Add({ EUserSceneTextureEvent::CustomRenderPass, NAME_None, (uint16)FCustomRenderPassBase::ERenderOutput::Normal, 0, (const UMaterialInterface*)CustomRenderPass });
#endif

					CustomRenderPass->OverrideRenderOutput(FCustomRenderPassBase::ERenderOutput::Normal);
					CopySceneCaptureComponentToTarget(GraphBuilder, *CustomRenderPassSceneTextures, NormalSceneTexture, ViewFamily, CustomRenderPassViews);
				}

				if (!SceneCaptureUserData.UserSceneTextureSceneColor.IsNone())
				{
					bool bFirstRender;
					FRDGTextureRef SceneColorSceneTexture = SceneTextures.FindOrAddUserSceneTexture(GraphBuilder, 0, SceneCaptureUserData.UserSceneTextureSceneColor, SceneCaptureUserData.SceneTextureDivisor, bFirstRender, nullptr, CustomRenderPassViews[0].ViewRect);
#if !(UE_BUILD_SHIPPING)
					SceneTextures.UserSceneTextureEvents.Add({ EUserSceneTextureEvent::CustomRenderPass, NAME_None, (uint16)FCustomRenderPassBase::ERenderOutput::SceneColorAndAlpha, 0, (const UMaterialInterface*)CustomRenderPass });
#endif

					CustomRenderPass->OverrideRenderOutput(FCustomRenderPassBase::ERenderOutput::SceneColorAndAlpha);
					CopySceneCaptureComponentToTarget(GraphBuilder, *CustomRenderPassSceneTextures, SceneColorSceneTexture, ViewFamily, CustomRenderPassViews);
				}

				CustomRenderPass->PostRender(GraphBuilder);

				// Mips are normally generated in UpdateSceneCaptureContentDeferred_RenderThread, but that doesn't run when the
				// scene capture runs as a custom render pass.  The function does nothing if the render target doesn't have mips.
				if (CustomRenderPassViews[0].bIsSceneCapture)
				{
					FGenerateMips::Execute(GraphBuilder, FeatureLevel, CustomRenderPass->GetRenderTargetTexture(), FGenerateMipsParams());
				}

			#if WITH_MGPU
				DoCrossGPUTransfers(GraphBuilder, CustomRenderPass->GetRenderTargetTexture(), CustomRenderPassViews, false, FRHIGPUMask::All(), nullptr);
			#endif
			}

			CustomRenderPass->EndPass(GraphBuilder);

			// Restore original scene texture uniforms
			CustomRenderPassSceneTextures->SetupMode = OriginalSceneTextureSetupMode;
			CustomRenderPassSceneTextures->UniformBuffer = OriginalSceneTextureUniformBuffer;
		}
	}

	TArray<Nanite::FRasterResults, TInlineAllocator<2>> NaniteRasterResults;
	TArray<Nanite::FPackedView, SceneRenderingAllocator> PrimaryNaniteViews;
	RenderPrepassAndVelocity(Views, NaniteBasePassVisibility, NaniteRasterResults, PrimaryNaniteViews, SceneTextures);

	// Run Nanite compute commands early in the frame to allow some task overlap on the CPU until the base pass runs.
	if (bNaniteEnabled && RendererOutput != ERendererOutput::DepthPrepassOnly && !bHasRayTracedOverlay)
	{
		Nanite::BuildShadingCommands(GraphBuilder, *Scene, ENaniteMeshPass::BasePass, Scene->NaniteShadingCommands[ENaniteMeshPass::BasePass]);
		if (bAnyLumenEnabled && RendererOutput == ERendererOutput::FinalSceneColor)
		{
			Nanite::BuildShadingCommands(GraphBuilder, *Scene, ENaniteMeshPass::LumenCardCapture, Scene->NaniteShadingCommands[ENaniteMeshPass::LumenCardCapture]);
		}
	}

	FComputeLightGridOutput ComputeLightGridOutput = {};

	FCompositionLighting CompositionLighting(InitViewTaskDatas.Decals, Views, SceneTextures, [this](int32 ViewIndex)
	{
		return GetViewPipelineState(Views[ViewIndex]).AmbientOcclusionMethod == EAmbientOcclusionMethod::SSAO;
	});

	const auto RenderOcclusionLambda = [&]() -> Froxel::FRenderer 
	{
		const int32 AsyncComputeMode = CVarSceneDepthHZBAsyncCompute.GetValueOnRenderThread();
		bool bAsyncCompute = AsyncComputeMode != 0;

		FBuildHZBAsyncComputeParams AsyncComputeParams = {};
		if (AsyncComputeMode == 2)
		{
			AsyncComputeParams.Prerequisite = ComputeLightGridOutput.CompactLinksPass;
		}

		bool bShouldGenerateFroxels = DoesVSMWantFroxels(ShaderPlatform);

		Froxel::FRenderer FroxelRenderer(bShouldGenerateFroxels, GraphBuilder, Views);

		RenderOcclusion(GraphBuilder, SceneTextures, bIsOcclusionTesting,
			bAsyncCompute ? &AsyncComputeParams : nullptr, FroxelRenderer);

		CompositionLighting.ProcessAfterOcclusion(GraphBuilder);

		return FroxelRenderer;
	};

	const bool bShouldRenderVolumetricCloudBase = ShouldRenderVolumetricCloud(Scene, ViewFamily.EngineShowFlags);
	const bool bShouldRenderVolumetricCloud = bShouldRenderVolumetricCloudBase && (!ViewFamily.EngineShowFlags.VisualizeVolumetricCloudConservativeDensity && !ViewFamily.EngineShowFlags.VisualizeVolumetricCloudEmptySpaceSkipping);
	const bool bShouldVisualizeVolumetricCloud = bShouldRenderVolumetricCloudBase && (!!ViewFamily.EngineShowFlags.VisualizeVolumetricCloudConservativeDensity || !!ViewFamily.EngineShowFlags.VisualizeVolumetricCloudEmptySpaceSkipping);
	const bool bAsyncComputeVolumetricCloud = IsVolumetricRenderTargetEnabled() && IsVolumetricRenderTargetAsyncCompute();
	const bool bVolumetricRenderTargetRequired = bShouldRenderVolumetricCloud && !bHasRayTracedOverlay;

	Froxel::FRenderer FroxelRenderer;

	FRDGTextureRef ViewFamilyTexture = TryCreateViewFamilyTexture(GraphBuilder, ViewFamily);
	FRDGTextureRef ViewFamilyDepthTexture = TryCreateViewFamilyDepthTexture(GraphBuilder, ViewFamily);
	if (RendererOutput == ERendererOutput::DepthPrepassOnly)
	{
		const FSingleLayerWaterPrePassResult* SingleLayerWaterPrePassResult = nullptr;
		if (ShouldRenderSingleLayerWaterDepthPrepass(Views))
		{
			SingleLayerWaterPrePassResult = RenderSingleLayerWaterDepthPrepass(GraphBuilder, Views, SceneTextures, ESingleLayerWaterPrepassLocation::BeforeBasePass, NaniteRasterResults);
		}

		FroxelRenderer = RenderOcclusionLambda();

		CopySceneCaptureComponentToTarget(GraphBuilder, SceneTextures, ViewFamilyTexture, ViewFamilyDepthTexture, ViewFamily, Views);
	}
	else
	{
		GVRSImageManager.PrepareImageBasedVRS(GraphBuilder, ViewFamily, SceneTextures);

		if (!IsForwardShadingEnabled(ShaderPlatform))
		{
			// Dynamic shadows are synced later when using the deferred path to make more headroom for tasks.
			FinishInitDynamicShadows(GraphBuilder, InitViewTaskDatas.DynamicShadows, InstanceCullingManager);
		}

		// Update groom only visible in shadow
		if (IsHairStrandsEnabled(EHairStrandsShaderType::All, Scene->GetShaderPlatform()) && RendererOutput == ERendererOutput::FinalSceneColor)
		{
			UpdateHairStrandsBookmarkParameters(Scene, Views, HairStrandsBookmarkParameters);

			// Interpolation for cards/meshes only visible in shadow needs to happen after the shadow jobs are completed
			const bool bRunHairStrands = HairStrandsBookmarkParameters.HasInstances() && (Views.Num() > 0);
			if (bRunHairStrands)
			{
				RunHairStrandsBookmark(GraphBuilder, EHairStrandsBookmark::ProcessCardsAndMeshesInterpolation_ShadowView, HairStrandsBookmarkParameters);
			}
		}

		// Early occlusion queries
		const bool bOcclusionBeforeBasePass = ((DepthPass.EarlyZPassMode == EDepthDrawingMode::DDM_AllOccluders) || bIsEarlyDepthComplete);

		if (bOcclusionBeforeBasePass)
		{
			FroxelRenderer = RenderOcclusionLambda();
		}

		// End early occlusion queries

		for (FSceneViewExtensionRef& ViewExtension : ViewFamily.ViewExtensions)
		{
			ViewExtension->PreRenderBasePass_RenderThread(GraphBuilder, ShouldRenderPrePass() /*bDepthBufferIsPopulated*/);
		}

		{
			SCOPE_CYCLE_COUNTER(STAT_WaitGatherAndSortLightsTask);
			GatherAndSortLightsTask.Wait();
		}

		{
			RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, PrepareForwardLightData);
			SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_PrepareForwardLightData);

			const FSortedLightSetSceneInfo* SortedLightSet = GatherAndSortLightsTask.GetResult();

			if (!ViewFamily.EngineShowFlags.PathTracing)
			{
				ComputeLightGridOutput = PrepareForwardLightData(GraphBuilder, true, *SortedLightSet);

				// Store this flag if lights are injected in the grids, check with 'AreLightsInLightGrid()'
				bAreLightsInLightGrid = true;
			}
			else
			{
				SetDummyForwardLightUniformBufferOnViews(GraphBuilder, ShaderPlatform, Views);
			}

			CSV_CUSTOM_STAT(LightCount, All,  float(SortedLightSet->SortedLights.Num()), ECsvCustomStatOp::Set);
			CSV_CUSTOM_STAT(LightCount, Batched, float(SortedLightSet->UnbatchedLightStart), ECsvCustomStatOp::Set);
			CSV_CUSTOM_STAT(LightCount, Unbatched, float(SortedLightSet->SortedLights.Num()) - float(SortedLightSet->UnbatchedLightStart), ECsvCustomStatOp::Set);
		}

		LightFunctionAtlas.RenderLightFunctionAtlas(GraphBuilder, Views);

		// Run before RenderSkyAtmosphereLookUpTables for cloud shadows to be valid.
		InitVolumetricCloudsForViews(GraphBuilder, bShouldRenderVolumetricCloudBase, InstanceCullingManager);

		BeginAsyncDistanceFieldShadowProjections(GraphBuilder, SceneTextures, InitViewTaskDatas.DynamicShadows);

		// Run local fog volume culling before base pass and after HZB generation to benefit from more culling.
		InitLocalFogVolumesForViews(Scene, Views, ViewFamily, GraphBuilder, bShouldRenderVolumetricFog, false /*bool bUseHalfResLocalFogVolume*/);

		if (bShouldRenderVolumetricCloudBase)
		{
			InitVolumetricRenderTargetForViews(GraphBuilder, Views, SceneTextures);
		}
		else
		{
			ResetVolumetricRenderTargetForViews(GraphBuilder, Views);
		}

		// Generate sky LUTs
		// TODO: Valid shadow maps (for volumetric light shafts) have not yet been generated at this point in the frame. Need to resolve dependency ordering!
		// This also must happen before the BasePass for Sky material to be able to sample valid LUTs.
		if (SkyAtmospherePassLocation == ESkyAtmospherePassLocation::BeforeBasePass && bShouldRenderSkyAtmosphere)
		{
			// Generate the Sky/Atmosphere look up tables
			RenderSkyAtmosphereLookUpTables(GraphBuilder, /* out */ SkyAtmospherePendingRDGResources);

			SkyAtmospherePendingRDGResources.CommitToSceneAndViewUniformBuffers(GraphBuilder, /* out */ ExternalAccessQueue);
		}
		else if (SkyAtmospherePassLocation == ESkyAtmospherePassLocation::BeforePrePass && bShouldRenderSkyAtmosphere)
		{
			SkyAtmospherePendingRDGResources.CommitToSceneAndViewUniformBuffers(GraphBuilder, /* out */ ExternalAccessQueue);
		}

		// Capture the SkyLight using the SkyAtmosphere and VolumetricCloud component if available.
		const bool bRealTimeSkyCaptureEnabled = Scene->SkyLight && Scene->SkyLight->bRealTimeCaptureEnabled && Views.Num() > 0 && ViewFamily.EngineShowFlags.SkyLighting;
		const bool bPathTracedAtmosphere = ViewFamily.EngineShowFlags.PathTracing && Views.Num() > 0 && PathTracing::UsesReferenceAtmosphere(Views[0]);
		if (bRealTimeSkyCaptureEnabled && !bPathTracedAtmosphere)
		{
			// Sky capture accesses the view uniform buffer which uses LUT's.
			ExternalAccessQueue.Submit(GraphBuilder);

			FViewInfo& MainView = Views[0];
			Scene->AllocateAndCaptureFrameSkyEnvMap(GraphBuilder, *this, MainView, bShouldRenderSkyAtmosphere, bShouldRenderVolumetricCloud, InstanceCullingManager, ExternalAccessQueue);
		}

		const ECustomDepthPassLocation CustomDepthPassLocation = GetCustomDepthPassLocation(ShaderPlatform);
		if (CustomDepthPassLocation == ECustomDepthPassLocation::BeforeBasePass)
		{
			QUICK_SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_CustomDepthPass_BeforeBasePass);
			if (RenderCustomDepthPass(GraphBuilder, SceneTextures.CustomDepth, SceneTextures.GetSceneTextureShaderParameters(FeatureLevel), NaniteRasterResults, PrimaryNaniteViews))
			{
				SceneTextures.SetupMode |= ESceneTextureSetupMode::CustomDepth;
				SceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &SceneTextures, FeatureLevel, SceneTextures.SetupMode);
			}
		}

		// Single layer water depth prepass. Needs to run before VSM page allocation. If there's a full depth prepass, it can run before the base pass, otherwise after.
		// Running before the base pass allows for some optimizations to save work in the base pass and lighting stages.
		const FSingleLayerWaterPrePassResult* SingleLayerWaterPrePassResult = nullptr;
		const ESingleLayerWaterPrepassLocation SingleLayerWaterPrepassLocation = GetSingleLayerWaterDepthPrepassLocation(bIsEarlyDepthComplete, CustomDepthPassLocation);
		const bool bShouldRenderSingleLayerWaterDepthPrepass = !bHasRayTracedOverlay && ShouldRenderSingleLayerWaterDepthPrepass(Views);
		if (bShouldRenderSingleLayerWaterDepthPrepass && SingleLayerWaterPrepassLocation == ESingleLayerWaterPrepassLocation::BeforeBasePass)
		{
			SingleLayerWaterPrePassResult = RenderSingleLayerWaterDepthPrepass(GraphBuilder, Views, SceneTextures, SingleLayerWaterPrepassLocation, NaniteRasterResults);
		}

		// Lumen updates need access to sky atmosphere LUT.
		ExternalAccessQueue.Submit(GraphBuilder);

		UpdateLumenScene(GraphBuilder, LumenFrameTemporaries);

		FRDGTextureRef HalfResolutionDepthCheckerboardMinMaxTexture = nullptr;
		FRDGTextureRef HalfResolutionDepthMinMaxTexture = nullptr;
		FRDGTextureRef QuarterResolutionDepthMinMaxTexture = nullptr;
		bool bQuarterResMinMaxDepthRequired = bShouldRenderVolumetricCloud && ShouldVolumetricCloudTraceWithMinMaxDepth(Views);

		auto GenerateQuarterResDepthMinMaxTexture = [&](auto& GraphBuilder, auto& Views, auto& SceneDepthTexture)
		{
			if (bQuarterResMinMaxDepthRequired)
			{
				check(SceneDepthTexture != nullptr);					// Must receive a valid texture
				check(HalfResolutionDepthMinMaxTexture == nullptr);		// Only generate it once
				check(QuarterResolutionDepthMinMaxTexture == nullptr);	// Only generate it once
				CreateQuarterResolutionDepthMinAndMaxFromDepthTexture(GraphBuilder, Views, SceneDepthTexture, HalfResolutionDepthMinMaxTexture, QuarterResolutionDepthMinMaxTexture);
			}
			else
			{
				HalfResolutionDepthCheckerboardMinMaxTexture = CreateHalfResolutionDepthCheckerboardMinMax(GraphBuilder, Views, SceneDepthTexture);
			}
		};
		
		FRDGTextureRef ForwardScreenSpaceShadowMaskTexture = nullptr;
		FRDGTextureRef ForwardScreenSpaceShadowMaskHairTexture = nullptr;
		bool bShadowMapsRenderedEarly = false;
		if (IsForwardShadingEnabled(ShaderPlatform))
		{
			// With forward shading we need to render shadow maps early
			ensureMsgf(!VirtualShadowMapArray.IsEnabled(), TEXT("Virtual shadow maps are not supported in the forward shading path"));
			RenderShadowDepthMaps(GraphBuilder, InitViewTaskDatas.DynamicShadows, InstanceCullingManager, ExternalAccessQueue);
			bShadowMapsRenderedEarly = true;

			if (bHairStrandsEnable)
			{
				RDG_EVENT_SCOPE(GraphBuilder, "Hair");

				RunHairStrandsBookmark(GraphBuilder, EHairStrandsBookmark::ProcessStrandsInterpolation, HairStrandsBookmarkParameters);
				if (!bHasRayTracedOverlay)
				{
					RenderHairPrePass(GraphBuilder, Scene, SceneTextures, Views, InstanceCullingManager, HairStrandsBookmarkParameters.CullingResults);
					RenderHairBasePass(GraphBuilder, Scene, SceneTextures, Views, InstanceCullingManager);
				}
			}

			RenderForwardShadowProjections(GraphBuilder, SceneTextures, ForwardScreenSpaceShadowMaskTexture, ForwardScreenSpaceShadowMaskHairTexture);

			// With forward shading we need to render volumetric fog before the base pass
			ComputeVolumetricFog(GraphBuilder, SceneTextures);
		}
		else if ( CVarShadowMapsRenderEarly.GetValueOnRenderThread() )
		{
			// Disable early shadows if VSM is enabled, but warn
			ensureMsgf(!VirtualShadowMapArray.IsEnabled(), TEXT("Virtual shadow maps are not supported with r.shadow.ShadowMapsRenderEarly. Early shadows will be disabled"));
			if (!VirtualShadowMapArray.IsEnabled())
			{
				RenderShadowDepthMaps(GraphBuilder, InitViewTaskDatas.DynamicShadows, InstanceCullingManager, ExternalAccessQueue);
				bShadowMapsRenderedEarly = true;
			}
		}

		ExternalAccessQueue.Submit(GraphBuilder);

		{
			RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, DeferredShadingSceneRenderer_DBuffer);
			SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_DBuffer);
			CompositionLighting.ProcessBeforeBasePass(GraphBuilder, DBufferTextures, InstanceCullingManager, Scene->SubstrateSceneData);
		}
		
		if (IsForwardShadingEnabled(ShaderPlatform))
		{
			RenderIndirectCapsuleShadows(GraphBuilder, SceneTextures);
		}

		FTranslucencyLightingVolumeTextures TranslucencyLightingVolumeTextures;

		if (bRenderDeferredLighting && GbEnableAsyncComputeTranslucencyLightingVolumeClear && GSupportsEfficientAsyncCompute)
		{
			TranslucencyLightingVolumeTextures.Init(GraphBuilder, Views, ERDGPassFlags::AsyncCompute);
		}

		FRDGBufferRef DynamicGeometryScratchBuffer = nullptr;
#if RHI_RAYTRACING
		
		ERHIPipeline DynamicRTResourceAccessPipelines = Lumen::UseAsyncCompute(ViewFamily) ? ERHIPipeline::All : ERHIPipeline::Graphics;

		// Async AS builds can potentially overlap with BasePass.
		bool bNeedToSetupRayTracingRenderingData = DispatchRayTracingWorldUpdates(GraphBuilder, DynamicGeometryScratchBuffer, DynamicRTResourceAccessPipelines);

		/** Should be called somewhere before "SetupRayTracingRenderingData" */
		SetupRayTracingLightDataForViews(GraphBuilder);
#endif

		if (!bHasRayTracedOverlay)
		{
#if RHI_RAYTRACING
			// Lumen scene lighting requires ray tracing scene to be ready if HWRT shadows are desired
			if (bNeedToSetupRayTracingRenderingData && Lumen::UseHardwareRayTracedSceneLighting(ViewFamily))
			{
				SetupRayTracingRenderingData(GraphBuilder, *InitViewTaskDatas.RayTracingGatherInstances);
				bNeedToSetupRayTracingRenderingData = false;
			}
#endif

			LLM_SCOPE_BYTAG(Lumen);
			BeginGatheringLumenSurfaceCacheFeedback(GraphBuilder, Views[0], LumenFrameTemporaries);
			RenderLumenSceneLighting(GraphBuilder, LumenFrameTemporaries, InitViewTaskDatas.LumenDirectLighting);
		}

		{
			if (!bHasRayTracedOverlay)
			{
				RenderBasePass(*this, GraphBuilder, Views, SceneTextures, DBufferTextures, BasePassDepthStencilAccess, ForwardScreenSpaceShadowMaskTexture, InstanceCullingManager, bNaniteEnabled, Scene->NaniteShadingCommands[ENaniteMeshPass::BasePass], NaniteRasterResults);
			}

			if (!bAllowReadOnlyDepthBasePass)
			{
				AddResolveSceneDepthPass(GraphBuilder, Views, SceneTextures.Depth);
			}

			if (bNaniteEnabled)
			{
				if (bVisualizeNanite)
				{
					FNanitePickingFeedback PickingFeedback = { 0 };

					Nanite::AddVisualizationPasses(
						GraphBuilder,
						Scene,
						SceneTextures,
						ViewFamily.EngineShowFlags,
						Views,
						NaniteRasterResults,
						PickingFeedback,
						VirtualShadowMapArray
					);

					OnGetOnScreenMessages.AddLambda([this, PickingFeedback, RenderFlags = NaniteRasterResults[0].RenderFlags, ScenePtr = Scene](FScreenMessageWriter& ScreenMessageWriter)->void
					{
						Nanite::DisplayPicking(ScenePtr, PickingFeedback, RenderFlags, ScreenMessageWriter);
					});
				}
			}

			// VisualizeVirtualShadowMap TODO
		}

		FRDGTextureRef ExposureIlluminanceSetup = nullptr;
		if (!bHasRayTracedOverlay)
		{
			// Extract emissive from SceneColor (before lighting is applied)
			ExposureIlluminanceSetup = AddSetupExposureIlluminancePass(GraphBuilder, Views, SceneTextures);
		}

		if (ViewFamily.EngineShowFlags.VisualizeLightCulling)
		{
			FRDGTextureRef VisualizeLightCullingTexture = GraphBuilder.CreateTexture(SceneTextures.Color.Target->Desc, TEXT("SceneColorVisualizeLightCulling"));
			AddClearRenderTargetPass(GraphBuilder, VisualizeLightCullingTexture, FLinearColor::Transparent);
			SceneTextures.Color.Target = VisualizeLightCullingTexture;

			// When not in MSAA, assign to both targets.
			if (SceneTexturesConfig.NumSamples == 1)
			{
				SceneTextures.Color.Resolve = SceneTextures.Color.Target;
			}
		}

		if (bRenderDeferredLighting)
		{
			// mark GBufferA for saving for next frame if it's needed
			ExtractNormalsForNextFrameReprojection(GraphBuilder, SceneTextures, Views);
		}

		// Rebuild scene textures to include GBuffers.
		SceneTextures.SetupMode |= ESceneTextureSetupMode::GBuffers;
		if (bShouldRenderVelocities && (bBasePassCanOutputVelocity || Scene->EarlyZPassMode == DDM_AllOpaqueNoVelocity))
		{
			SceneTextures.SetupMode |= ESceneTextureSetupMode::SceneVelocity;
		}
		SceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &SceneTextures, FeatureLevel, SceneTextures.SetupMode);

		if (bRealTimeSkyCaptureEnabled)
		{
			Scene->ValidateSkyLightRealTimeCapture(GraphBuilder, Views[0], SceneTextures.Color.Target);
		}

		VisualizeVolumetricLightmap(GraphBuilder, SceneTextures);

		// Occlusion after base pass
		if (!bOcclusionBeforeBasePass)
		{
			FroxelRenderer = RenderOcclusionLambda();
		}

		// End occlusion after base

		if (!bUseGBuffer)
		{
			AddResolveSceneColorPass(GraphBuilder, Views, SceneTextures.Color);
		}

		// Render hair
		if (bHairStrandsEnable && !IsForwardShadingEnabled(ShaderPlatform))
		{
			RDG_EVENT_SCOPE(GraphBuilder, "Hair");

			RunHairStrandsBookmark(GraphBuilder, EHairStrandsBookmark::ProcessStrandsInterpolation, HairStrandsBookmarkParameters);
			if (!bHasRayTracedOverlay)
			{
				RenderHairPrePass(GraphBuilder, Scene, SceneTextures, Views, InstanceCullingManager, HairStrandsBookmarkParameters.CullingResults);
				RenderHairBasePass(GraphBuilder, Scene, SceneTextures, Views, InstanceCullingManager);
			}
		}

		if (ShouldRenderHeterogeneousVolumes(Scene) && !bHasRayTracedOverlay)
		{
			RenderHeterogeneousVolumeShadows(GraphBuilder, SceneTextures);
		}

		// Post base pass for material classification
		// This needs to run before virtual shadow map, in order to have ready&cleared classified SSS data
		if (Substrate::IsSubstrateEnabled() && !bHasRayTracedOverlay)
		{
			RDG_EVENT_SCOPE_STAT(GraphBuilder, Substrate, "Substrate");
			RDG_GPU_STAT_SCOPE(GraphBuilder, Substrate);

			// Substrate DBufferPass (optional)
			if (Substrate::IsDBufferPassEnabled(ShaderPlatform))
			{
				RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, DeferredShadingSceneRenderer_DBuffer);
				SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_DBuffer);
				Substrate::AddSubstrateDBufferBasePass(GraphBuilder, Views, SceneTextures, DBufferTextures, InitViewTaskDatas.Decals, InstanceCullingManager, Scene->SubstrateSceneData);
			}

			// Substrate classifation is done either in a standalone pass (here) or done within the StochasticLightingTileClassificationMark pass
			const bool bNeedsClassificationPass = !(RequiresStochasticLightingPass() && Substrate::UsesStochasticLightingClassification(ShaderPlatform));
			if (bNeedsClassificationPass)
			{
				Substrate::AddSubstrateMaterialClassificationPass(GraphBuilder, SceneTextures, DBufferTextures, Views);
			}
			{
				Substrate::AddSubstrateDBufferPass(GraphBuilder, SceneTextures, DBufferTextures, Views);
				Substrate::AddSubstrateSampleMaterialPass(GraphBuilder, Scene, SceneTextures, Views);
			}
		}

		FAsyncLumenIndirectLightingOutputs AsyncLumenIndirectLightingOutputs;

		if (!bHasRayTracedOverlay && RequiresStochasticLightingPass())
		{
			// Decals may modify GBuffers so they need to be done first. Can decals read velocities and/or custom depth? If so, they need to be rendered earlier too.
			CompositionLighting.ProcessAfterBasePass(GraphBuilder, InstanceCullingManager, FCompositionLighting::EProcessAfterBasePassMode::OnlyBeforeLightingDecals, Scene->SubstrateSceneData);
			AsyncLumenIndirectLightingOutputs.bHasDrawnBeforeLightingDecals = true;

			RDG_EVENT_SCOPE_STAT(GraphBuilder, RenderDeferredLighting, "StochasticLighting");
			RDG_GPU_STAT_SCOPE(GraphBuilder, RenderDeferredLighting);

			StochasticLightingTileClassificationMark(GraphBuilder, LumenFrameTemporaries, SceneTextures);
		}

		// Copy lighting channels out of stencil before deferred decals which overwrite those values
		TArray<FRDGTextureRef, TInlineAllocator<2>> NaniteShadingMask;
		if (bNaniteEnabled && Views.Num() > 0)
		{
			check(Views.Num() == NaniteRasterResults.Num());
			for (const Nanite::FRasterResults& Results : NaniteRasterResults)
			{
				NaniteShadingMask.Add(Results.ShadingMask);
			}
		}
		FRDGTextureRef LightingChannelsTexture = CopyStencilToLightingChannelTexture(GraphBuilder, SceneTextures.Stencil, NaniteShadingMask);

		// Single layer water depth prepass. Needs to run before VSM page allocation.
		if (bShouldRenderSingleLayerWaterDepthPrepass && SingleLayerWaterPrepassLocation == ESingleLayerWaterPrepassLocation::AfterBasePass)
		{
			SingleLayerWaterPrePassResult = RenderSingleLayerWaterDepthPrepass(GraphBuilder, Views, SceneTextures, SingleLayerWaterPrepassLocation, NaniteRasterResults);
		}

		GraphBuilder.FlushSetupQueue();
		
		TSharedPtr<FMegaLightsFrameTemporaries> MegaLightsContext = nullptr;

		// Shadows, lumen and fog after base pass
		if (!bHasRayTracedOverlay)
		{
#if RHI_RAYTRACING
			// When Lumen HWRT is running async we need to wait for ray tracing scene before dispatching the work
			if (bNeedToSetupRayTracingRenderingData && Lumen::UseAsyncCompute(ViewFamily))
			{
				SetupRayTracingRenderingData(GraphBuilder, *InitViewTaskDatas.RayTracingGatherInstances);
				bNeedToSetupRayTracingRenderingData = false;
			}
#endif // RHI_RAYTRACING

			DispatchAsyncLumenIndirectLightingWork(
				GraphBuilder,
				SceneTextures,
				InstanceCullingManager,
				LumenFrameTemporaries,
				InitViewTaskDatas.DynamicShadows,
				LightingChannelsTexture,
				AsyncLumenIndirectLightingOutputs);

			// Kick off volumetric clouds async dispatch after Lumen
			// Lumen has a dependency on the opaque so should run first
			// Volumetric Clouds have a depedency on translucent, so should run second and overlap opaque work after Lumen async is done
			if (bShouldRenderVolumetricCloud && bAsyncComputeVolumetricCloud)
			{
				GenerateQuarterResDepthMinMaxTexture(GraphBuilder, Views, SceneTextures.Depth.Resolve);

				bool bSkipVolumetricRenderTarget = false;
				bool bSkipPerPixelTracing = true;
				RenderVolumetricCloud(GraphBuilder, SceneTextures, bSkipVolumetricRenderTarget, bSkipPerPixelTracing,
					HalfResolutionDepthCheckerboardMinMaxTexture, QuarterResolutionDepthMinMaxTexture, true, InstanceCullingManager);
			}

			if (!bShadowMapsRenderedEarly && ShadowSceneRenderer.GetVirtualShadowMapArray().IsEnabled())
			{
				FFrontLayerTranslucencyData FrontLayerTranslucencyData = RenderFrontLayerTranslucency(GraphBuilder, Views, SceneTextures, true /*VSM page marking*/);
				ShadowSceneRenderer.BeginMarkVirtualShadowMapPages(GraphBuilder, SingleLayerWaterPrePassResult, FrontLayerTranslucencyData, FroxelRenderer);
			}

			// Do MegaLights sampling before VSM pages are marked and rendered so they can be specialized
			// based on the selected samples.
			const FSortedLightSetSceneInfo& SortedLightSet = *GatherAndSortLightsTask.GetResult();
			if (bRenderDeferredLighting && SortedLightSet.MegaLightsLightStart < SortedLightSet.SortedLights.Num())
			{
				MegaLightsContext = GenerateMegaLightsSamples(
					GraphBuilder,
					SceneTextures,
					LumenFrameTemporaries,
					LightingChannelsTexture);
			}

			// If we haven't already rendered shadow maps, render them now (due to forward shading or r.shadow.ShadowMapsRenderEarly)
			if (!bShadowMapsRenderedEarly)
			{
				RenderShadowDepthMaps(GraphBuilder, InitViewTaskDatas.DynamicShadows, InstanceCullingManager, ExternalAccessQueue);
			}
			CheckShadowDepthRenderCompleted();

#if RHI_RAYTRACING
			// Lumen scene lighting requires ray tracing scene to be ready if HWRT shadows are desired
			if (bNeedToSetupRayTracingRenderingData && Lumen::UseHardwareRayTracedSceneLighting(ViewFamily))
			{
				SetupRayTracingRenderingData(GraphBuilder, *InitViewTaskDatas.RayTracingGatherInstances);
				bNeedToSetupRayTracingRenderingData = false;
			}
#endif // RHI_RAYTRACING
		}

		ExternalAccessQueue.Submit(GraphBuilder);

		// End shadow and fog after base pass

#if RHI_RAYTRACING
		if (IsRayTracingEnabled(ViewFamily.GetShaderPlatform()) && GRHISupportsRayTracingShaders)
		{
			for (int32 ViewExt = 0; ViewExt < ViewFamily.ViewExtensions.Num(); ++ViewExt)
			{
				if (EnumHasAnyFlags(ViewFamily.ViewExtensions[ViewExt]->GetFlags(), ESceneViewExtensionFlags::SubscribesToPostTLASBuild))
				{
					if (bNeedToSetupRayTracingRenderingData)
					{
						SetupRayTracingRenderingData(GraphBuilder, *InitViewTaskDatas.RayTracingGatherInstances);
						bNeedToSetupRayTracingRenderingData = false;
					}

					for (int32 ViewIndex = 0; ViewIndex < ViewFamily.Views.Num(); ++ViewIndex)
					{
						ViewFamily.ViewExtensions[ViewExt]->PostTLASBuild_RenderThread(GraphBuilder, Views[ViewIndex]);
					}
				}
			}
		}
#endif // RHI_RAYTRACING

		if (bNaniteEnabled)
		{
			// Needs doing after shadows such that the checks for shadow atlases etc work.
			Nanite::ListStatFilters(this);

			if (GNaniteShowStats != 0)
			{
				for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
				{
					const FViewInfo& View = Views[ViewIndex];
					if (IStereoRendering::IsAPrimaryView(View))
					{
						Nanite::PrintStats(GraphBuilder, View);
					}
				}
			}
		}

		{
			if (FVirtualShadowMapArrayCacheManager* CacheManager = VirtualShadowMapArray.CacheManager)
			{
				// Do this even if VSMs are disabled this frame to clean up any previously extracted data
				CacheManager->ExtractFrameData(
					GraphBuilder,
					VirtualShadowMapArray,
					*this,
					ViewFamily.EngineShowFlags.VirtualShadowMapPersistentData);
			}
		}

		if (CustomDepthPassLocation == ECustomDepthPassLocation::AfterBasePass)
		{
			QUICK_SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_CustomDepthPass_AfterBasePass);
			if (RenderCustomDepthPass(GraphBuilder, SceneTextures.CustomDepth, SceneTextures.GetSceneTextureShaderParameters(FeatureLevel), NaniteRasterResults, PrimaryNaniteViews))
			{
				SceneTextures.SetupMode |= ESceneTextureSetupMode::CustomDepth;
				SceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &SceneTextures, FeatureLevel, SceneTextures.SetupMode);
			}
		}

		// If we are not rendering velocities in depth or base pass then do that here.
		if (bShouldRenderVelocities && !bBasePassCanOutputVelocity && (Scene->EarlyZPassMode != DDM_AllOpaqueNoVelocity))
		{
			RenderVelocities(GraphBuilder, Views, SceneTextures, EVelocityPass::Opaque, bHairStrandsEnable);
		}

		// Pre-lighting composition lighting stage
		// e.g. deferred decals, SSAO
		{
			RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, AfterBasePass);
			SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_AfterBasePass);

			if (!IsForwardShadingEnabled(ShaderPlatform))
			{
				AddResolveSceneDepthPass(GraphBuilder, Views, SceneTextures.Depth);
			}

			const FCompositionLighting::EProcessAfterBasePassMode Mode = AsyncLumenIndirectLightingOutputs.bHasDrawnBeforeLightingDecals ?
				FCompositionLighting::EProcessAfterBasePassMode::SkipBeforeLightingDecals : FCompositionLighting::EProcessAfterBasePassMode::All;

			CompositionLighting.ProcessAfterBasePass(GraphBuilder, InstanceCullingManager, Mode, Scene->SubstrateSceneData);
		}

		// Rebuild scene textures to include velocity, custom depth, and SSAO.
		SceneTextures.SetupMode |= ESceneTextureSetupMode::All;
		SceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &SceneTextures, FeatureLevel, SceneTextures.SetupMode);

		if (!IsForwardShadingEnabled(ShaderPlatform))
		{
			// Clear stencil to 0 now that deferred decals are done using what was setup in the base pass.
			AddClearStencilPass(GraphBuilder, SceneTextures.Depth.Target);
		}

#if RHI_RAYTRACING
		// If Lumen did not force an earlier ray tracing scene sync, we must wait for it here.
		if (bNeedToSetupRayTracingRenderingData)
		{
			SetupRayTracingRenderingData(GraphBuilder, *InitViewTaskDatas.RayTracingGatherInstances);
			bNeedToSetupRayTracingRenderingData = false;
		}
#endif // RHI_RAYTRACING

		GraphBuilder.FlushSetupQueue();

		if (bRenderDeferredLighting)
		{
			RDG_EVENT_SCOPE_STAT(GraphBuilder, RenderDeferredLighting, "RenderDeferredLighting");
			RDG_GPU_STAT_SCOPE(GraphBuilder, RenderDeferredLighting);
			RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, RenderLighting);

			SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_Lighting);
			SCOPED_NAMED_EVENT(RenderLighting, FColor::Emerald);

			TArray<FRDGTextureRef> DynamicBentNormalAOTextures;

			RenderDiffuseIndirectAndAmbientOcclusion(
				GraphBuilder,
				SceneTextures,
				LumenFrameTemporaries,
				LightingChannelsTexture,
				/* bCompositeRegularLumenOnly = */ false,
				/* bIsVisualizePass = */ false,
				AsyncLumenIndirectLightingOutputs);

			if (IsTranslucencyLightingVolumeUsingVoxelMarking())
			{
				for (FViewInfo& View : Views)
				{
					if (View.TranslucencyVolumeMarkData[0].MarkTexture == nullptr || View.TranslucencyVolumeMarkData[1].MarkTexture == nullptr)
					{
						LumenTranslucencyReflectionsMarkUsedProbes(
							GraphBuilder,
							*this,
							View,
							SceneTextures,
							nullptr);
					}
				}
			}

			// These modulate the scenecolor output from the basepass, which is assumed to be indirect lighting
			RenderIndirectCapsuleShadows(GraphBuilder, SceneTextures);

			// These modulate the scene color output from the base pass, which is assumed to be indirect lighting
			RenderDFAOAsIndirectShadowing(GraphBuilder, SceneTextures, DynamicBentNormalAOTextures);

			// Clear the translucent lighting volumes before we accumulate
			if ((GbEnableAsyncComputeTranslucencyLightingVolumeClear && GSupportsEfficientAsyncCompute) == false)
			{
				TranslucencyLightingVolumeTextures.Init(GraphBuilder, Views, ERDGPassFlags::Compute);
			}

#if RHI_RAYTRACING
			// Only used by ray traced shadows
			if (IsRayTracingEnabled() && Views[0].bHasRayTracingShadows && Views[0].IsRayTracingAllowedForView())
			{
				RenderDitheredLODFadingOutMask(GraphBuilder, Views[0], SceneTextures.Depth.Target);
			}
#endif

			GatherTranslucencyVolumeMarkedVoxels(GraphBuilder);

			const FSortedLightSetSceneInfo& SortedLightSet = *GatherAndSortLightsTask.GetResult();

			RenderLights(GraphBuilder, SceneTextures, LightingChannelsTexture, SortedLightSet);

			if (MegaLightsContext.IsValid())
			{
				RenderMegaLights(
					GraphBuilder,
					MegaLightsContext,
					SceneTextures,
					NaniteShadingMask,
					LightingChannelsTexture);
			}

			RenderTranslucencyLightingVolume(GraphBuilder, TranslucencyLightingVolumeTextures, SortedLightSet);

			// Do DiffuseIndirectComposite after Lights so that async Lumen work can overlap
			RenderDiffuseIndirectAndAmbientOcclusion(
				GraphBuilder,
				SceneTextures,
				LumenFrameTemporaries,
				LightingChannelsTexture,
				/* bCompositeRegularLumenOnly = */ true,
				/* bIsVisualizePass = */ false,
				AsyncLumenIndirectLightingOutputs);

			// Render diffuse sky lighting and reflections that only operate on opaque pixels
			RenderDeferredReflectionsAndSkyLighting(GraphBuilder, SceneTextures, LumenFrameTemporaries, DynamicBentNormalAOTextures);

#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
			// Renders debug visualizations for global illumination plugins
			RenderGlobalIlluminationPluginVisualizations(GraphBuilder, LightingChannelsTexture);
#endif

			AddSubsurfacePass(GraphBuilder, SceneTextures, Views);

			Substrate::AddSubstrateOpaqueRoughRefractionPasses(GraphBuilder, SceneTextures, Views);

			{
				RenderHairStrandsSceneColorScattering(GraphBuilder, SceneTextures.Color.Target, Scene, Views);
			}

		#if RHI_RAYTRACING
			if (ShouldRenderRayTracingSkyLight(Scene->SkyLight, Scene->GetShaderPlatform()) 
				//@todo - integrate RenderRayTracingSkyLight into RenderDiffuseIndirectAndAmbientOcclusion
				&& GetViewPipelineState(Views[0]).DiffuseIndirectMethod != EDiffuseIndirectMethod::Lumen
				&& ViewFamily.EngineShowFlags.GlobalIllumination)
			{
				FRDGTextureRef SkyLightTexture = nullptr;
				FRDGTextureRef SkyLightHitDistanceTexture = nullptr;
				RenderRayTracingSkyLight(GraphBuilder, SceneTextures.Color.Target, SkyLightTexture, SkyLightHitDistanceTexture);
				CompositeRayTracingSkyLight(GraphBuilder, SceneTextures, SkyLightTexture, SkyLightHitDistanceTexture);
			}
		#endif

			if (Substrate::IsSubstrateEnabled())
			{
				// Now remove all the Substrate tile stencil tags used by deferred tiled light passes. Make later marks such as responssive AA works.
				AddClearStencilPass(GraphBuilder, SceneTextures.Depth.Target);
			}
		}
		else if (HairStrands::HasViewHairStrandsData(Views) && ViewFamily.EngineShowFlags.Lighting)
		{
			const FSortedLightSetSceneInfo& SortedLightSet = *GatherAndSortLightsTask.GetResult();
			RenderLightsForHair(GraphBuilder, SceneTextures, SortedLightSet, ForwardScreenSpaceShadowMaskHairTexture, LightingChannelsTexture);
			RenderDeferredReflectionsAndSkyLightingHair(GraphBuilder);
		}

		// Volumetric fog after Lumen GI and shadow depths
		if (!IsForwardShadingEnabled(ShaderPlatform) && !bHasRayTracedOverlay)
		{
			ComputeVolumetricFog(GraphBuilder, SceneTextures);
		}

		if (ShouldRenderHeterogeneousVolumes(Scene) && !bHasRayTracedOverlay)
		{
			RenderHeterogeneousVolumes(GraphBuilder, SceneTextures);
		}

		GraphBuilder.FlushSetupQueue();

		if (bShouldRenderVolumetricCloud && !bHasRayTracedOverlay)
		{
			if (!bAsyncComputeVolumetricCloud)
			{
				if(IsVolumetricRenderTargetEnabled())
				{
					GenerateQuarterResDepthMinMaxTexture(GraphBuilder, Views, SceneTextures.Depth.Resolve);
				}

				// Generate the volumetric cloud render target
				bool bSkipVolumetricRenderTarget = false;
				bool bSkipPerPixelTracing = true;
				RenderVolumetricCloud(GraphBuilder, SceneTextures, bSkipVolumetricRenderTarget, bSkipPerPixelTracing,
					HalfResolutionDepthCheckerboardMinMaxTexture, QuarterResolutionDepthMinMaxTexture, false, InstanceCullingManager);
			}
			// Reconstruct the volumetric cloud render target to be ready to compose it over the scene
			ReconstructVolumetricRenderTarget(GraphBuilder, Views, SceneTextures.Depth.Resolve, HalfResolutionDepthCheckerboardMinMaxTexture, bAsyncComputeVolumetricCloud);
		}

		TArray<FScreenPassTexture, TInlineAllocator<4>> TSRFlickeringInputTextures;
		if (!bHasRayTracedOverlay)
		{
			// Extract TSR's moire heuristic luminance before rendering translucency into the scene color.
			for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
			{
				FViewInfo& View = Views[ViewIndex];
				if (NeedTSRAntiFlickeringPass(View))
				{
					if (TSRFlickeringInputTextures.Num() == 0)
					{
						TSRFlickeringInputTextures.SetNum(Views.Num());
					}

					TSRFlickeringInputTextures[ViewIndex] = AddTSRMeasureFlickeringLuma(GraphBuilder, View.ShaderMap, FScreenPassTexture(SceneTextures.Color.Target, View.ViewRect));
				}
			}
		}

		const bool bShouldRenderTranslucency = !bHasRayTracedOverlay && ShouldRenderTranslucency();

		// Union of all translucency view render flags.
		ETranslucencyView TranslucencyViewsToRender = bShouldRenderTranslucency ? GetTranslucencyViews(Views) : ETranslucencyView::None;

		FTranslucencyPassResourcesMap TranslucencyResourceMap(Views.Num());

		const bool bIsCameraUnderWater = EnumHasAnyFlags(TranslucencyViewsToRender, ETranslucencyView::UnderWater);
		FRDGTextureRef LightShaftOcclusionTexture = nullptr;
		const bool bShouldRenderSingleLayerWater = !bHasRayTracedOverlay && ShouldRenderSingleLayerWater(Views);
		FSceneWithoutWaterTextures SceneWithoutWaterTextures;
		auto RenderLightShaftSkyFogAndCloud = [&]()
		{
			// Draw Lightshafts
			if (!bHasRayTracedOverlay && ViewFamily.EngineShowFlags.LightShafts)
			{
				SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_RenderLightShaftOcclusion);
				LightShaftOcclusionTexture = RenderLightShaftOcclusion(GraphBuilder, SceneTextures);
			}

			// Draw the sky atmosphere
			if (!bHasRayTracedOverlay && bShouldRenderSkyAtmosphere && !IsForwardShadingEnabled(ShaderPlatform))
			{
				SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_RenderSkyAtmosphere);
				RenderSkyAtmosphere(GraphBuilder, SceneTextures);
			}

			// Draw fog.
			bool bHeightFogHasComposedLocalFogVolume = false;
			if (!bHasRayTracedOverlay && ShouldRenderFog(ViewFamily))
			{
				RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, RenderFog);
				SCOPED_NAMED_EVENT(RenderFog, FColor::Emerald);
				SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_RenderFog);
				const bool bFogComposeLocalFogVolumes = (bShouldRenderLocalFogVolumeInVolumetricFog && bShouldRenderVolumetricFog) || bShouldRenderLocalFogVolumeDuringHeightFogPass;
				RenderFog(GraphBuilder, SceneTextures, LightShaftOcclusionTexture, bFogComposeLocalFogVolumes);
				bHeightFogHasComposedLocalFogVolume = bFogComposeLocalFogVolumes;
			}

			if (!bHasRayTracedOverlay) 
			{
				// Local Fog Volumes (LFV) rendering order is first HeightFog, then LFV, then volumetric fog on top.
				// LFVs are rendered as part of the regular height fog + volumetric fog pass when volumetric fog is enabled and it is requested to voxelise LFVs into volumetric fog.
				// Otherwise, they are rendered in an independent pass (this for instance make it independent of the near clip plane optimization).
				if (!bHeightFogHasComposedLocalFogVolume)
				{
					RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, RenderLocalFogVolume);
					SCOPED_NAMED_EVENT(RenderLocalFogVolume, FColor::Emerald);
					SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_RenderLocalFogVolume);
					RenderLocalFogVolume(Scene, Views, ViewFamily, GraphBuilder, SceneTextures, LightShaftOcclusionTexture);
				}
				// Also compose on top the visualisation pass if enabled.
				if (bShouldRenderLocalFogVolumeVisualizationPass)
				{
					RenderLocalFogVolumeVisualization(Scene, Views, ViewFamily, GraphBuilder, SceneTextures);
				}
			}

			// After the height fog, Draw volumetric clouds (having fog applied on them already) when using per pixel tracing,
			if (!bHasRayTracedOverlay && bShouldRenderVolumetricCloud)
			{
				bool bSkipVolumetricRenderTarget = true;
				bool bSkipPerPixelTracing = false;
				RenderVolumetricCloud(GraphBuilder, SceneTextures, bSkipVolumetricRenderTarget, bSkipPerPixelTracing,
					HalfResolutionDepthCheckerboardMinMaxTexture, QuarterResolutionDepthMinMaxTexture, false, InstanceCullingManager);
			}

			// Or composite the off screen buffer over the scene.
			if (bVolumetricRenderTargetRequired)
			{
				const bool bComposeWithWater = bIsCameraUnderWater ? false : bShouldRenderSingleLayerWater;
				ComposeVolumetricRenderTargetOverScene(
					GraphBuilder, Views, SceneTextures.Color.Target, SceneTextures.Depth.Target,
					bComposeWithWater,
					SceneWithoutWaterTextures, SceneTextures);
			}
		};

		if (bShouldRenderSingleLayerWater)
		{
			if (bIsCameraUnderWater)
			{
				RenderLightShaftSkyFogAndCloud();

				RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, RenderTranslucency);
				SCOPED_NAMED_EVENT(RenderTranslucency, FColor::Emerald);
				SCOPE_CYCLE_COUNTER(STAT_TranslucencyDrawTime);
				const bool bStandardTranslucentCanRenderSeparate = false;
				FRDGTextureMSAA SharedDepthTexture;
				RenderTranslucency(*this, GraphBuilder, SceneTextures, TranslucencyLightingVolumeTextures, &TranslucencyResourceMap, Views, ETranslucencyView::UnderWater, SeparateTranslucencyDimensions, InstanceCullingManager, bStandardTranslucentCanRenderSeparate, SharedDepthTexture);
				EnumRemoveFlags(TranslucencyViewsToRender, ETranslucencyView::UnderWater);
			}

			RenderSingleLayerWater(GraphBuilder, Views, SceneTextures, SingleLayerWaterPrePassResult, bShouldRenderVolumetricCloud, SceneWithoutWaterTextures, LumenFrameTemporaries, bIsCameraUnderWater);

			// Replace main depth texture with the output of the SLW depth prepass which contains the scene + water. Stencil is cleared to 0.
			if (SingleLayerWaterPrePassResult)
			{
				SceneTextures.Depth = SingleLayerWaterPrePassResult->DepthPrepassTexture;
			}
		}

		// Rebuild scene textures to include scene color.
		SceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &SceneTextures, FeatureLevel, SceneTextures.SetupMode);

		if (!bHasRayTracedOverlay)
		{
			// Extract TSR's thin geometry coverage after SLW but before rendering translucency into the scene color.
			for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
			{
				FViewInfo& View = Views[ViewIndex];
				if (NeedTSRAntiFlickeringPass(View))
				{
					if (TSRFlickeringInputTextures.Num() == 0)
					{
						TSRFlickeringInputTextures.SetNum(Views.Num());
					}

					AddTSRMeasureThinGeometryCoverage(GraphBuilder, View.ShaderMap, SceneTextures, TSRFlickeringInputTextures[ViewIndex]);
				}
			}
		}

		if (!bIsCameraUnderWater)
		{
			RenderLightShaftSkyFogAndCloud();
		}

		FRDGTextureRef ExposureIlluminance = nullptr;
		if (!bHasRayTracedOverlay)
		{
			ExposureIlluminance = AddCalculateExposureIlluminancePass(GraphBuilder, Views, SceneTextures, TranslucencyLightingVolumeTextures, ExposureIlluminanceSetup);
		}

		RenderOpaqueFX(GraphBuilder, GetSceneViews(), GetSceneUniforms(), FXSystem, FeatureLevel, SceneTextures.UniformBuffer);

		FRendererModule& RendererModule = static_cast<FRendererModule&>(GetRendererModule());
		RendererModule.RenderPostOpaqueExtensions(GraphBuilder, Views, SceneTextures);

		if (Scene->GPUScene.ExecuteDeferredGPUWritePass(GraphBuilder, Views, EGPUSceneGPUWritePass::PostOpaqueRendering))
		{
			InstanceCullingManager.BeginDeferredCulling(GraphBuilder);
		}

		if (GetHairStrandsComposition() == EHairStrandsCompositionType::BeforeTranslucent)
		{
			RDG_EVENT_SCOPE_STAT(GraphBuilder, HairRendering, "HairRendering");
			RDG_GPU_STAT_SCOPE(GraphBuilder, HairRendering);
			RenderHairComposition(GraphBuilder, Views, SceneTextures.Color.Target, SceneTextures.Depth.Target, SceneTextures.Velocity, TranslucencyResourceMap);
		}

#if DEBUG_ALPHA_CHANNEL
		if (ShouldMakeDistantGeometryTranslucent())
		{
			SceneTextures.Color = MakeDistanceGeometryTranslucent(GraphBuilder, Views, SceneTextures);
			SceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &SceneTextures, FeatureLevel, SceneTextures.SetupMode);
		}
#endif

		// Experimental voxel test code
		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
		{
			const FViewInfo& View = Views[ViewIndex];
		
			Nanite::DrawVisibleBricks( GraphBuilder, *Scene, View, SceneTextures );
		}

		// Composite Heterogeneous Volumes
		if (!bHasRayTracedOverlay && ShouldRenderHeterogeneousVolumes(Scene) &&
			(GetHeterogeneousVolumesComposition() == EHeterogeneousVolumesCompositionType::BeforeTranslucent))
		{
			CompositeHeterogeneousVolumes(GraphBuilder, SceneTextures);
		}

		// Draw translucency.
		FRDGTextureMSAA TranslucencySharedDepthTexture;
		FFrontLayerTranslucencyData FrontLayerTranslucencyData;
		if (!bHasRayTracedOverlay && TranslucencyViewsToRender != ETranslucencyView::None)
		{
			RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, RenderTranslucency);
			SCOPED_NAMED_EVENT(RenderTranslucency, FColor::Emerald);
			SCOPE_CYCLE_COUNTER(STAT_TranslucencyDrawTime);

			RDG_EVENT_SCOPE(GraphBuilder, "Translucency");

			// Raytracing doesn't need the distortion effect.
			const bool bShouldRenderDistortion = TranslucencyViewsToRender != ETranslucencyView::RayTracing && ShouldRenderDistortion();

			// Lumen/VSM translucent front layer
			FrontLayerTranslucencyData = RenderFrontLayerTranslucency(GraphBuilder, Views, SceneTextures, false /*VSM page marking*/);

#if RHI_RAYTRACING
			if (EnumHasAnyFlags(TranslucencyViewsToRender, ETranslucencyView::RayTracing))
			{
				if (!RenderRayTracedTranslucency(GraphBuilder, SceneTextures, LumenFrameTemporaries, FrontLayerTranslucencyData))
				{
					RenderRayTracingTranslucency(GraphBuilder, SceneTextures.Color);
				}

				EnumRemoveFlags(TranslucencyViewsToRender, ETranslucencyView::RayTracing);
			}
#endif

			for (FViewInfo& View : Views)
			{
				if (GetViewPipelineState(View).ReflectionsMethod == EReflectionsMethod::Lumen)
				{
					RenderLumenFrontLayerTranslucencyReflections(GraphBuilder, View, SceneTextures, LumenFrameTemporaries, FrontLayerTranslucencyData);
				}
			}

			// Sort objects' triangles
			for (FViewInfo& View : Views)
			{
				if (OIT::IsSortedTrianglesEnabled(View.GetShaderPlatform()))
				{
					OIT::AddSortTrianglesPass(GraphBuilder, View, Scene->OITSceneData, FTriangleSortingOrder::BackToFront);
				}
			}

			{
				// Render all remaining translucency views.
				const bool bStandardTranslucentCanRenderSeparate = bShouldRenderDistortion; // It is only needed to render standard translucent as separate when there is distortion (non self distortion of transmittance/specular/etc.)
				RenderTranslucency(*this, GraphBuilder, SceneTextures, TranslucencyLightingVolumeTextures, &TranslucencyResourceMap, Views, TranslucencyViewsToRender, SeparateTranslucencyDimensions, InstanceCullingManager, bStandardTranslucentCanRenderSeparate, TranslucencySharedDepthTexture);
			}

			// Compose hair before velocity/distortion pass since these pass write depth value, 
			// and this would make the hair composition fails in this cases.
			if (GetHairStrandsComposition() == EHairStrandsCompositionType::AfterTranslucent)
			{
				RDG_EVENT_SCOPE_STAT(GraphBuilder, HairRendering, "HairRendering");
				RDG_GPU_STAT_SCOPE(GraphBuilder, HairRendering);

				RenderHairComposition(GraphBuilder, Views, SceneTextures.Color.Target, SceneTextures.Depth.Target, SceneTextures.Velocity, TranslucencyResourceMap);
			}

			if (bShouldRenderDistortion)
			{
				RenderDistortion(GraphBuilder, SceneTextures.Color.Target, SceneTextures.Depth.Target, SceneTextures.Velocity, TranslucencyResourceMap);
			}

			if (bShouldRenderVelocities && CVarTranslucencyVelocity.GetValueOnRenderThread() != 0)
			{
				const bool bRecreateSceneTextures = !HasBeenProduced(SceneTextures.Velocity);

				RenderVelocities(GraphBuilder, Views, SceneTextures, EVelocityPass::Translucent, false);

				if (bRecreateSceneTextures)
				{
					// Rebuild scene textures to include newly allocated velocity.
					SceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &SceneTextures, FeatureLevel, SceneTextures.SetupMode);
				}

				RenderVelocities(GraphBuilder, Views, SceneTextures, EVelocityPass::TranslucentClippedDepth, false, /*bBindRenderTarget=*/ false);
			}
		}
		else if (GetHairStrandsComposition() == EHairStrandsCompositionType::AfterTranslucent)
		{
			RDG_EVENT_SCOPE_STAT(GraphBuilder, HairRendering, "HairRendering");
			RDG_GPU_STAT_SCOPE(GraphBuilder, HairRendering);

			RenderHairComposition(GraphBuilder, Views, SceneTextures.Color.Target, SceneTextures.Depth.Target, SceneTextures.Velocity, TranslucencyResourceMap);
		}

#if !UE_BUILD_SHIPPING
		if (CVarForceBlackVelocityBuffer.GetValueOnRenderThread())
		{
			SceneTextures.Velocity = SystemTextures.Black;

			// Rebuild the scene texture uniform buffer to include black.
			SceneTextures.UniformBuffer = CreateSceneTextureUniformBuffer(GraphBuilder, &SceneTextures, FeatureLevel, SceneTextures.SetupMode);
		}
#endif

		{
			if (HairStrandsBookmarkParameters.HasInstances())
			{
				HairStrandsBookmarkParameters.SceneColorTexture = SceneTextures.Color.Target;
				HairStrandsBookmarkParameters.SceneDepthTexture = SceneTextures.Depth.Target;
				RenderHairStrandsDebugInfo(GraphBuilder, Scene, Views, HairStrandsBookmarkParameters);
			}
		}

		if (VirtualShadowMapArray.IsEnabled())
		{
			VirtualShadowMapArray.RenderDebugInfo(GraphBuilder, Views);
		}

		for (FViewInfo& View : Views)
		{
			ShadingEnergyConservation::Debug(GraphBuilder, View, SceneTextures);
		}

		if (!bHasRayTracedOverlay && ViewFamily.EngineShowFlags.LightShafts)
		{
			SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_RenderLightShaftBloom);
			RenderLightShaftBloom(GraphBuilder, SceneTextures, /* inout */ TranslucencyResourceMap);
		}

		{
			// Light shaft (rendered just above) can render in separate transluceny at low resolution according to r.SeparateTranslucencyScreenPercentage. 
			// So we can only upsample that buffer if required after the light shaft bloom pass.
			UpscaleTranslucencyIfNeeded(GraphBuilder, SceneTextures, TranslucencyViewsToRender, /* inout */ &TranslucencyResourceMap, TranslucencySharedDepthTexture);
			TranslucencyViewsToRender = ETranslucencyView::None;
		}

		FPathTracingResources PathTracingResources;

#if RHI_RAYTRACING
		if (IsRayTracingEnabled())
		{
			// Path tracer requires the full ray tracing pipeline support, as well as specialized extra shaders.
			// Most of the ray tracing debug visualizations also require the full pipeline, but some support inline mode.
			
			if (ViewFamily.EngineShowFlags.PathTracing 
				&& FDataDrivenShaderPlatformInfo::GetSupportsPathTracing(Scene->GetShaderPlatform()))
			{
				for (const FViewInfo& View : Views)
				{
					RenderPathTracing(GraphBuilder, View, SceneTextures.UniformBuffer, SceneTextures.Color.Target, SceneTextures.Depth.Target,PathTracingResources);
				}
			}
			else if (ViewFamily.EngineShowFlags.RayTracingDebug)
			{
				// TODO: This will include visible bindings for all views, but we could potentially also provide a way to get visible bindings for a single view
				// Although that would require running deduplication logic separately for each view in VisibleRayTracingShaderBindingsFinalizeTask
				TConstArrayView<FRayTracingShaderBindingData> VisibleRayTracingShaderBindings = RayTracing::GetVisibleShaderBindings(*InitViewTaskDatas.RayTracingGatherInstances);

				for (const FViewInfo& View : Views)
				{
					FRayTracingPickingFeedback PickingFeedback = {};
					RenderRayTracingDebug(GraphBuilder, *Scene, View, SceneTextures, VisibleRayTracingShaderBindings, PickingFeedback);

					OnGetOnScreenMessages.AddLambda([this, &View, PickingFeedback](FScreenMessageWriter& ScreenMessageWriter)->void
						{
							RayTracingDebugDisplayOnScreenMessages(ScreenMessageWriter, View);
							RayTracingDisplayPicking(PickingFeedback, ScreenMessageWriter);
						});
				}
			}
		}
#endif
		RendererModule.RenderOverlayExtensions(GraphBuilder, Views, SceneTextures);

		if (ViewFamily.EngineShowFlags.PhysicsField && Scene->PhysicsField)
		{
			RenderPhysicsField(GraphBuilder, Views, Scene->PhysicsField, SceneTextures.Color.Target);
		}

		if (ViewFamily.EngineShowFlags.VisualizeDistanceFieldAO && ShouldRenderDistanceFieldLighting(Scene->DistanceFieldSceneData, Views))
		{
			// Use the skylight's max distance if there is one, to be consistent with DFAO shadowing on the skylight
			const float OcclusionMaxDistance = Scene->SkyLight && !Scene->SkyLight->bWantsStaticShadowing ? Scene->SkyLight->OcclusionMaxDistance : Scene->DefaultMaxDistanceFieldOcclusionDistance;
			TArray<FRDGTextureRef> DummyOutput;
			RenderDistanceFieldLighting(GraphBuilder, SceneTextures, FDistanceFieldAOParameters(OcclusionMaxDistance), DummyOutput, false, ViewFamily.EngineShowFlags.VisualizeDistanceFieldAO);
		}

		// Draw visualizations just before use to avoid target contamination
		if (ViewFamily.EngineShowFlags.VisualizeMeshDistanceFields || ViewFamily.EngineShowFlags.VisualizeGlobalDistanceField)
		{
			RenderMeshDistanceFieldVisualization(GraphBuilder, SceneTextures);
		}

		if (bRenderDeferredLighting)
		{
			RenderLumenMiscVisualizations(GraphBuilder, SceneTextures, LumenFrameTemporaries);
			RenderDiffuseIndirectAndAmbientOcclusion(
				GraphBuilder,
				SceneTextures,
				LumenFrameTemporaries,
				LightingChannelsTexture,
				/* bCompositeRegularLumenOnly = */ false,
				/* bIsVisualizePass = */ true,
				AsyncLumenIndirectLightingOutputs);
		}

		if (ViewFamily.EngineShowFlags.StationaryLightOverlap)
		{
			RenderStationaryLightOverlap(GraphBuilder, SceneTextures, LightingChannelsTexture);
		}

		// Composite Heterogeneous Volumes
		if (!bHasRayTracedOverlay && ShouldRenderHeterogeneousVolumes(Scene) &&
			(GetHeterogeneousVolumesComposition() == EHeterogeneousVolumesCompositionType::AfterTranslucent))
		{
			CompositeHeterogeneousVolumes(GraphBuilder, SceneTextures);
		}

		if (bShouldVisualizeVolumetricCloud && !bHasRayTracedOverlay)
		{
			RenderVolumetricCloud(GraphBuilder, SceneTextures, false, true, HalfResolutionDepthCheckerboardMinMaxTexture, QuarterResolutionDepthMinMaxTexture, false, InstanceCullingManager);
			ReconstructVolumetricRenderTarget(GraphBuilder, Views, SceneTextures.Depth.Resolve, HalfResolutionDepthCheckerboardMinMaxTexture, false);
			ComposeVolumetricRenderTargetOverSceneForVisualization(GraphBuilder, Views, SceneTextures.Color.Target, SceneTextures);
			RenderVolumetricCloud(GraphBuilder, SceneTextures, true, false, HalfResolutionDepthCheckerboardMinMaxTexture, QuarterResolutionDepthMinMaxTexture, false, InstanceCullingManager);
		}

		if (!bHasRayTracedOverlay)
		{
			AddSparseVolumeTextureViewerRenderPass(GraphBuilder, *this, SceneTextures);
		}

		RenderTranslucencyVolumeVisualization(GraphBuilder, SceneTextures, TranslucencyLightingVolumeTextures);

		// Resolve the scene color for post processing.
		AddResolveSceneColorPass(GraphBuilder, Views, SceneTextures.Color);

		RendererModule.RenderPostResolvedSceneColorExtension(GraphBuilder, SceneTextures);

		CopySceneCaptureComponentToTarget(GraphBuilder, SceneTextures, ViewFamilyTexture, ViewFamilyDepthTexture, ViewFamily, Views);

		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
		{
			const FViewInfo& View = Views[ViewIndex];

			if (((View.FinalPostProcessSettings.DynamicGlobalIlluminationMethod == EDynamicGlobalIlluminationMethod::ScreenSpace && ScreenSpaceRayTracing::ShouldKeepBleedFreeSceneColor(View))
				|| GetViewPipelineState(View).DiffuseIndirectMethod == EDiffuseIndirectMethod::Lumen
				|| GetViewPipelineState(View).ReflectionsMethod == EReflectionsMethod::Lumen)
				&& !View.bStatePrevViewInfoIsReadOnly)
			{
				// Keep scene color and depth for next frame screen space ray tracing.
				FSceneViewState* ViewState = View.ViewState;
				GraphBuilder.QueueTextureExtraction(SceneTextures.Depth.Resolve, &ViewState->PrevFrameViewInfo.DepthBuffer);
				GraphBuilder.QueueTextureExtraction(SceneTextures.Color.Resolve, &ViewState->PrevFrameViewInfo.ScreenSpaceRayTracingInput);
			}
		}

		// Finish rendering for each view.
		if (ViewFamily.bResolveScene && ViewFamilyTexture)
		{
			RDG_EVENT_SCOPE_STAT(GraphBuilder, Postprocessing, "PostProcessing");
			RDG_GPU_STAT_SCOPE(GraphBuilder, Postprocessing);
			SCOPED_NAMED_EVENT(PostProcessing, FColor::Emerald);

			FinishUpdateExposureCompensationCurveLUT(GraphBuilder.RHICmdList, &UpdateExposureCompensationCurveLUTTaskData);

			FPostProcessingInputs PostProcessingInputs;
			PostProcessingInputs.ViewFamilyTexture = ViewFamilyTexture;
			PostProcessingInputs.ViewFamilyDepthTexture = ViewFamilyDepthTexture;
			PostProcessingInputs.CustomDepthTexture = SceneTextures.CustomDepth.Depth;
			PostProcessingInputs.ExposureIlluminance = ExposureIlluminance;
			PostProcessingInputs.SceneTextures = SceneTextures.UniformBuffer;
			PostProcessingInputs.bSeparateCustomStencil = SceneTextures.CustomDepth.bSeparateStencilBuffer;
			PostProcessingInputs.PathTracingResources = PathTracingResources;

			FRDGTextureRef InstancedEditorDepthTexture = nullptr; // Used to pass instanced stereo depth data from primary to secondary views

			GraphBuilder.FlushSetupQueue();

			if (ViewFamily.UseDebugViewPS())
			{
				for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
				{
					const FViewInfo& View = Views[ViewIndex];
					const Nanite::FRasterResults* NaniteResults = bNaniteEnabled ? &NaniteRasterResults[ViewIndex] : nullptr;
					RDG_GPU_MASK_SCOPE(GraphBuilder, View.GPUMask);
					RDG_EVENT_SCOPE_CONDITIONAL(GraphBuilder, Views.Num() > 1, "View%d", ViewIndex);
					PostProcessingInputs.TranslucencyViewResourcesMap = FTranslucencyViewResourcesMap(TranslucencyResourceMap, ViewIndex);
					AddDebugViewPostProcessingPasses(GraphBuilder, View, ViewIndex, GetSceneUniforms(), PostProcessingInputs, NaniteResults, &VirtualShadowMapArray);
				}
			}
			else
			{
				for (int32 ViewExt = 0; ViewExt < ViewFamily.ViewExtensions.Num(); ++ViewExt)
				{
					for (int32 ViewIndex = 0; ViewIndex < ViewFamily.Views.Num(); ++ViewIndex)
					{
						FViewInfo& View = Views[ViewIndex];
						RDG_GPU_MASK_SCOPE(GraphBuilder, View.GPUMask);
						PostProcessingInputs.TranslucencyViewResourcesMap = FTranslucencyViewResourcesMap(TranslucencyResourceMap, ViewIndex);
						ViewFamily.ViewExtensions[ViewExt]->PrePostProcessPass_RenderThread(GraphBuilder, View, PostProcessingInputs);
					}
				}
				for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
				{
					const FViewInfo& View = Views[ViewIndex];
					const int32 NaniteResultsIndex = View.bIsInstancedStereoEnabled ? View.PrimaryViewIndex : ViewIndex;
					const Nanite::FRasterResults* NaniteResults = bNaniteEnabled ? &NaniteRasterResults[NaniteResultsIndex] : nullptr;
					RDG_GPU_MASK_SCOPE(GraphBuilder, View.GPUMask);
					RDG_EVENT_SCOPE_CONDITIONAL(GraphBuilder, Views.Num() > 1, "View%d", ViewIndex);

					PostProcessingInputs.TranslucencyViewResourcesMap = FTranslucencyViewResourcesMap(TranslucencyResourceMap, ViewIndex);

					if (IsPostProcessVisualizeCalibrationMaterialEnabled(View))
					{
						const UMaterialInterface* DebugMaterialInterface = GetPostProcessVisualizeCalibrationMaterialInterface(View);
						check(DebugMaterialInterface);

						AddVisualizeCalibrationMaterialPostProcessingPasses(GraphBuilder, View, PostProcessingInputs, DebugMaterialInterface);
					}
					else
					{
						const FPerViewPipelineState& ViewPipelineState = GetViewPipelineState(View);

						FScreenPassTexture TSRFlickeringInput;
						if (ViewIndex < TSRFlickeringInputTextures.Num())
						{
							TSRFlickeringInput = TSRFlickeringInputTextures[ViewIndex];
						}

						// If we're using instanced stereo, only the primary view simple element collectors will be populated with elements.
						// However, since post processing is always rendered per-view, we need to mirror the collectors to any instanced secondary views.
						if (View.bIsSinglePassStereo && View.StereoPass == EStereoscopicPass::eSSP_SECONDARY)
						{
							const FViewInfo& PrimaryView = Views[View.PrimaryViewIndex];

							View.SimpleElementCollector = PrimaryView.SimpleElementCollector;
							View.EditorSimpleElementCollector = PrimaryView.EditorSimpleElementCollector;
#if UE_ENABLE_DEBUG_DRAWING
							View.DebugSimpleElementCollector = PrimaryView.DebugSimpleElementCollector;
#endif
						}

						AddPostProcessingPasses(
							GraphBuilder,
							View, ViewIndex,
							GetSceneUniforms(),
							ViewPipelineState.DiffuseIndirectMethod,
							ViewPipelineState.ReflectionsMethod,
							PostProcessingInputs,
							NaniteResults,
							InstanceCullingManager,
							&VirtualShadowMapArray,
							LumenFrameTemporaries,
							SceneWithoutWaterTextures,
							TSRFlickeringInput,
							InstancedEditorDepthTexture);
					}
				}
			}
		}

		if (bUseVirtualTexturing)
		{
			VirtualTexture::EndFeedback(GraphBuilder);
		}

		// After AddPostProcessingPasses in case of Lumen Visualizations writing to feedback
		FinishGatheringLumenSurfaceCacheFeedback(GraphBuilder, Views[0], LumenFrameTemporaries, FrontLayerTranslucencyData, SceneTextures);

#if RHI_RAYTRACING
		RayTracingScene.PostRender(GraphBuilder);
#endif

		if (ViewFamily.bResolveScene && ViewFamilyTexture)
		{
			GVRSImageManager.DrawDebugPreview(GraphBuilder, ViewFamily, ViewFamilyTexture);
		}

		GEngine->GetPostRenderDelegateEx().Broadcast(GraphBuilder);
	}

	FinishUpdateExposureCompensationCurveLUT(GraphBuilder.RHICmdList, &UpdateExposureCompensationCurveLUTTaskData);
	
	GetSceneExtensionsRenderers().PostRender(GraphBuilder);

#if WITH_MGPU
	if (ViewFamily.bMultiGPUForkAndJoin)
	{
		DoCrossGPUTransfers(GraphBuilder, ViewFamilyTexture, Views, CrossGPUTransferFencesDefer.Num() > 0, RenderTargetGPUMask, CrossGPUTransferDeferred.GetReference());
	}
	FlushCrossGPUTransfers(GraphBuilder);
#endif

	{
		SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_RenderFinish);

		RDG_EVENT_SCOPE_STAT(GraphBuilder, FrameRenderFinish, "FrameRenderFinish");
		RDG_GPU_STAT_SCOPE(GraphBuilder, FrameRenderFinish);

		OnRenderFinish(GraphBuilder, ViewFamilyTexture);
		GraphBuilder.AddDispatchHint();
		GraphBuilder.FlushSetupQueue();
	}

	QueueSceneTextureExtractions(GraphBuilder, SceneTextures);

	::Substrate::PostRender(*Scene);
	HairStrands::PostRender(*Scene);
	HeterogeneousVolumes::PostRender(*Scene, Views);

	// Release the view's previous frame histories so that their memory can be reused at the graph's execution.
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		Views[ViewIndex].PrevViewInfo = FPreviousViewInfo();
	}

	if (NaniteBasePassVisibility.Visibility)
	{
		NaniteBasePassVisibility.Visibility->FinishVisibilityFrame();
		NaniteBasePassVisibility.Visibility = nullptr;
	}

	if (Scene->InstanceCullingOcclusionQueryRenderer)
	{
		Scene->InstanceCullingOcclusionQueryRenderer->EndFrame(GraphBuilder);
	}
}
```

* Engine/Source/Runtime/Renderer/Private/Nanite/NaniteCullRaster.cpp
```Cpp
void FRenderer::AddPass_PrimitiveFilter()
{
	LLM_SCOPE_BYTAG(Nanite);
	
	const uint32 PrimitiveCount = uint32(Scene.GetMaxPersistentPrimitiveIndex());

	if (PrimitiveCount == 0)
	{
		return;
	}

	const bool bHLODActive = Scene.SceneLODHierarchy.IsActive();
	const uint32 HiddenHLODPrimitiveCount = bHLODActive && SceneView.ViewState ? SceneView.ViewState->HLODVisibilityState.ForcedHiddenPrimitiveMap.CountSetBits() : 0;
	const uint32 HiddenPrimitiveCount = SceneView.HiddenPrimitives.Num() + HiddenHLODPrimitiveCount;
	const uint32 ShowOnlyPrimitiveCount = SceneView.ShowOnlyPrimitives.IsSet() ? SceneView.ShowOnlyPrimitives->Num() : 0u;
	
	EFilterFlags HiddenFilterFlags = Configuration.HiddenFilterFlags;
	
	if (!SceneView.Family->EngineShowFlags.StaticMeshes)
	{
		HiddenFilterFlags |= EFilterFlags::StaticMesh;
	}

	if (!SceneView.Family->EngineShowFlags.InstancedStaticMeshes)
	{
		HiddenFilterFlags |= EFilterFlags::InstancedStaticMesh;
	}

	if (!SceneView.Family->EngineShowFlags.InstancedFoliage)
	{
		HiddenFilterFlags |= EFilterFlags::Foliage;
	}

	if (!SceneView.Family->EngineShowFlags.InstancedGrass)
	{
		HiddenFilterFlags |= EFilterFlags::Grass;
	}

	if (!SceneView.Family->EngineShowFlags.Landscape)
	{
		HiddenFilterFlags |= EFilterFlags::Landscape;
	}

	const bool bAnyPrimitiveFilter = (HiddenPrimitiveCount + ShowOnlyPrimitiveCount) > 0;
	const bool bAnyFilterFlags = HiddenFilterFlags != EFilterFlags::None;
	
	if (CVarNaniteFilterPrimitives.GetValueOnRenderThread() != 0 && (bAnyPrimitiveFilter || bAnyFilterFlags))
	{
		const uint32 DWordCount = FMath::DivideAndRoundUp(PrimitiveCount, 32u); // 32 primitive bits per uint32
		const uint32 PrimitiveFilterBufferElements = FMath::RoundUpToPowerOfTwo(DWordCount);

		PrimitiveFilterBuffer = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateStructuredDesc(sizeof(uint32), PrimitiveFilterBufferElements), TEXT("Nanite.PrimitiveFilter"));

		// Zeroed initially to indicate "all primitives unfiltered / visible"
		AddClearUAVPass(GraphBuilder, GraphBuilder.CreateUAV(PrimitiveFilterBuffer), 0);

		// Create buffer from "show only primitives" set
		if (ShowOnlyPrimitiveCount > 0)
		{
			TArray<uint32, SceneRenderingAllocator> ShowOnlyPrimitiveIds;
			ShowOnlyPrimitiveIds.Reserve(FMath::RoundUpToPowerOfTwo(ShowOnlyPrimitiveCount));

			const TSet<FPrimitiveComponentId>& ShowOnlyPrimitivesSet = SceneView.ShowOnlyPrimitives.GetValue();
			for (TSet<FPrimitiveComponentId>::TConstIterator It(ShowOnlyPrimitivesSet); It; ++It)
			{
				ShowOnlyPrimitiveIds.Add(It->PrimIDValue);
			}

			// Add extra entries to ensure the buffer is valid pow2 in size
			ShowOnlyPrimitiveIds.SetNumZeroed(FMath::RoundUpToPowerOfTwo(ShowOnlyPrimitiveCount));

			// Sort the buffer by ascending value so the GPU binary search works properly
			Algo::Sort(ShowOnlyPrimitiveIds);

			ShowOnlyPrimitivesBuffer = CreateUploadBuffer(
				GraphBuilder,
				TEXT("Nanite.ShowOnlyPrimitivesBuffer"),
				sizeof(uint32),
				ShowOnlyPrimitiveIds.Num(),
				ShowOnlyPrimitiveIds.GetData(),
				sizeof(uint32) * ShowOnlyPrimitiveIds.Num()
			);
		}

		// Create buffer from "hidden primitives" set
		if (HiddenPrimitiveCount > 0)
		{
			TArray<uint32, SceneRenderingAllocator> HiddenPrimitiveIds;
			HiddenPrimitiveIds.Reserve(FMath::RoundUpToPowerOfTwo(HiddenPrimitiveCount));

			for (TSet<FPrimitiveComponentId>::TConstIterator It(SceneView.HiddenPrimitives); It; ++It)
			{
				HiddenPrimitiveIds.Add(It->PrimIDValue);
			}

			// HLOD visibily state
			if (HiddenHLODPrimitiveCount > 0)
			{
				for (TConstSetBitIterator It(SceneView.ViewState->HLODVisibilityState.ForcedHiddenPrimitiveMap); It; ++It)
				{
					const int32 Index = It.GetIndex();
					const FPrimitiveComponentId& PrimitiveComponentId = Scene.PrimitiveComponentIds[Index];
					HiddenPrimitiveIds.Add(PrimitiveComponentId.PrimIDValue);
				}
			}

			// Add extra entries to ensure the buffer is valid pow2 in size
			HiddenPrimitiveIds.SetNumZeroed(FMath::RoundUpToPowerOfTwo(HiddenPrimitiveCount));

			// Sort the buffer by ascending value so the GPU binary search works properly
			Algo::Sort(HiddenPrimitiveIds);

			HiddenPrimitivesBuffer = CreateUploadBuffer(
				GraphBuilder,
				TEXT("Nanite.HiddenPrimitivesBuffer"),
				sizeof(uint32),
				HiddenPrimitiveIds.Num(),
				HiddenPrimitiveIds.GetData(),
				sizeof(uint32) * HiddenPrimitiveIds.Num()
			);
		}

		FPrimitiveFilter_CS::FParameters* PassParameters = GraphBuilder.AllocParameters<FPrimitiveFilter_CS::FParameters>();

		PassParameters->NumPrimitives = PrimitiveCount;
		PassParameters->HiddenFilterFlags = uint32(HiddenFilterFlags);
		PassParameters->NumHiddenPrimitives = FMath::RoundUpToPowerOfTwo(HiddenPrimitiveCount);
		PassParameters->NumShowOnlyPrimitives = FMath::RoundUpToPowerOfTwo(ShowOnlyPrimitiveCount);
		PassParameters->Scene = SceneUniformBuffer;
		PassParameters->PrimitiveFilterBuffer = GraphBuilder.CreateUAV(PrimitiveFilterBuffer);

		if (HiddenPrimitivesBuffer != nullptr)
		{
			PassParameters->HiddenPrimitivesList = GraphBuilder.CreateSRV(HiddenPrimitivesBuffer, PF_R32_UINT);
		}

		if (ShowOnlyPrimitivesBuffer != nullptr)
		{
			PassParameters->ShowOnlyPrimitivesList = GraphBuilder.CreateSRV(ShowOnlyPrimitivesBuffer, PF_R32_UINT);
		}

		FPrimitiveFilter_CS::FPermutationDomain PermutationVector;
		PermutationVector.Set<FPrimitiveFilter_CS::FHiddenPrimitivesListDim>(HiddenPrimitivesBuffer != nullptr);
		PermutationVector.Set<FPrimitiveFilter_CS::FShowOnlyPrimitivesListDim>(ShowOnlyPrimitivesBuffer != nullptr);

		auto ComputeShader = SharedContext.ShaderMap->GetShader<FPrimitiveFilter_CS>(PermutationVector);
		FComputeShaderUtils::AddPass(
			GraphBuilder,
			RDG_EVENT_NAME("PrimitiveFilter"),
			ComputeShader,
			PassParameters,
			FComputeShaderUtils::GetGroupCountWrapped(PrimitiveCount, 64)
		);
	}
}

void AddPass_InitClusterCullArgs(
	FRDGBuilder& GraphBuilder,
	FGlobalShaderMap* ShaderMap,
	FRDGEventName&& PassName,
	FRDGBufferUAVRef QueueStateUAV,
	FRDGBufferRef ClusterCullArgs,
	uint32 CullingPass
)
{
	FInitClusterCullArgs_CS::FParameters* PassParameters = GraphBuilder.AllocParameters< FInitClusterCullArgs_CS::FParameters >();

	PassParameters->OutQueueState			= QueueStateUAV;
	PassParameters->OutClusterCullArgs		= GraphBuilder.CreateUAV(ClusterCullArgs);
	PassParameters->MaxCandidateClusters	= Nanite::FGlobalResources::GetMaxCandidateClusters();
	PassParameters->InitIsPostPass			= (CullingPass == CULLING_PASS_OCCLUSION_POST) ? 1 : 0;

	auto ComputeShader = ShaderMap->GetShader<FInitClusterCullArgs_CS>();
	FComputeShaderUtils::AddPass(
		GraphBuilder,
		Forward<FRDGEventName>(PassName),
		ComputeShader,
		PassParameters,
		FIntVector(1, 1, 1)
	);
}

void AddPass_InitNodeCullArgs(
	FRDGBuilder& GraphBuilder,
	FGlobalShaderMap* ShaderMap,
	FRDGEventName&& PassName,
	FRDGBufferUAVRef QueueStateUAV,
	FRDGBufferRef NodeCullArgs0,
	FRDGBufferRef NodeCullArgs1,
	uint32 CullingPass
)
{
	FInitNodeCullArgs_CS::FParameters* PassParameters = GraphBuilder.AllocParameters< FInitNodeCullArgs_CS::FParameters >();

	PassParameters->OutQueueState			= QueueStateUAV;
	PassParameters->OutNodeCullArgs0		= GraphBuilder.CreateUAV(NodeCullArgs0);
	PassParameters->OutNodeCullArgs1		= GraphBuilder.CreateUAV(NodeCullArgs1);
	PassParameters->MaxNodes				= Nanite::FGlobalResources::GetMaxNodes();
	PassParameters->InitIsPostPass			= (CullingPass == CULLING_PASS_OCCLUSION_POST) ? 1 : 0;

	auto ComputeShader = ShaderMap->GetShader<FInitNodeCullArgs_CS>();
	FComputeShaderUtils::AddPass(
		GraphBuilder,
		Forward<FRDGEventName>(PassName),
		ComputeShader,
		PassParameters,
		FIntVector(2, 1, 1)
	);
}


void FRenderer::AddPass_NodeAndClusterCull(
	FRDGEventName&& PassName,
	const FNodeAndClusterCullSharedParameters& SharedParameters,
	FRDGBufferRef CurrentIndirectArgs,
	FRDGBufferRef NextIndirectArgs,
	uint32 NodeLevel,
	uint32 CullingPass,
	uint32 CullingType
	)
{
	FNodeAndClusterCull_CS::FParameters* PassParameters = GraphBuilder.AllocParameters< FNodeAndClusterCull_CS::FParameters >();
	PassParameters->SharedParameters	= SharedParameters;
	PassParameters->NodeLevel			= NodeLevel;
	
	FNodeAndClusterCull_CS::FPermutationDomain PermutationVector;
	PermutationVector.Set<FNodeAndClusterCull_CS::FCullingPassDim>(CullingPass);
	PermutationVector.Set<FNodeAndClusterCull_CS::FMultiViewDim>(bMultiView);
	PermutationVector.Set<FNodeAndClusterCull_CS::FVirtualTextureTargetDim>(IsUsingVirtualShadowMap());
	PermutationVector.Set<FNodeAndClusterCull_CS::FMaterialCacheDim>(IsMaterialCache());
	PermutationVector.Set<FNodeAndClusterCull_CS::FSplineDeformDim>(NaniteSplineMeshesSupported()); // TODO: Nanite-Skinning - leverage this?
	PermutationVector.Set<FNodeAndClusterCull_CS::FDebugFlagsDim>(IsDebuggingEnabled());
	PermutationVector.Set<FNodeAndClusterCull_CS::FCullingTypeDim>(CullingType);
	auto ComputeShader = SharedContext.ShaderMap->GetShader<FNodeAndClusterCull_CS>(PermutationVector);

	if (CullingType == NANITE_CULLING_TYPE_NODES || CullingType == NANITE_CULLING_TYPE_CLUSTERS)
	{
		if (CullingType == NANITE_CULLING_TYPE_NODES)
		{
			PassParameters->CurrentNodeIndirectArgs = GraphBuilder.CreateSRV(CurrentIndirectArgs);
			PassParameters->NextNodeIndirectArgs = GraphBuilder.CreateUAV(NextIndirectArgs);
		}
		
		PassParameters->IndirectArgs = CurrentIndirectArgs;
		FComputeShaderUtils::AddPass(
			GraphBuilder,
			Forward<FRDGEventName>(PassName),
			ComputeShader,
			PassParameters,
			CurrentIndirectArgs,
			NodeLevel * NANITE_NODE_CULLING_ARG_COUNT * sizeof(uint32)
		);
	}
	else if(CullingType == NANITE_CULLING_TYPE_PERSISTENT_NODES_AND_CLUSTERS)
	{
		FComputeShaderUtils::AddPass(
			GraphBuilder,
			Forward<FRDGEventName>(PassName),
			ComputeShader,
			PassParameters,
			FIntVector(GRHIPersistentThreadGroupCount, 1, 1)
		);
	}
	else
	{
		checkf(false, TEXT("Unknown culling type: %d"), CullingType);
	}
}

void FRenderer::AddPass_NodeAndClusterCull( uint32 CullingPass )
{
	FNodeAndClusterCullSharedParameters SharedParameters;
	SharedParameters.Scene = SceneUniformBuffer;
	SharedParameters.CullingParameters = CullingParameters;
	SharedParameters.MaxNodes = Nanite::FGlobalResources::GetMaxNodes();
	SharedParameters.MaxAssemblyTransforms = Nanite::FGlobalResources::GetMaxVisibleAssemblyParts();
	SharedParameters.ClusterPageData = Nanite::GStreamingManager.GetClusterPageDataSRV(GraphBuilder);
	SharedParameters.HierarchyBuffer = Nanite::GStreamingManager.GetHierarchySRV(GraphBuilder);

	check(DrawPassIndex == 0 || RenderFlags & NANITE_RENDER_FLAG_HAS_PREV_DRAW_DATA); // sanity check
	if (RenderFlags & NANITE_RENDER_FLAG_HAS_PREV_DRAW_DATA)
	{
		SharedParameters.InTotalPrevDrawClusters = GraphBuilder.CreateSRV(TotalPrevDrawClustersBuffer);
	}
	else
	{
		FRDGBufferRef Dummy = GSystemTextures.GetDefaultStructuredBuffer(GraphBuilder, 8);
		SharedParameters.InTotalPrevDrawClusters = GraphBuilder.CreateSRV(Dummy);
	}

	SharedParameters.QueueState = GraphBuilder.CreateUAV(QueueState);
	SharedParameters.CandidateNodes = GraphBuilder.CreateUAV(CandidateNodesBuffer);
	SharedParameters.CandidateClusters = GraphBuilder.CreateUAV(CandidateClustersBuffer);
	if (ClusterBatchesBuffer)
	{
		SharedParameters.ClusterBatches = GraphBuilder.CreateUAV(ClusterBatchesBuffer);
	}

	if (CullingPass == CULLING_PASS_NO_OCCLUSION || CullingPass == CULLING_PASS_OCCLUSION_MAIN)
	{
		SharedParameters.VisibleClustersArgsSWHW = GraphBuilder.CreateUAV(MainRasterizeArgsSWHW);
	}
	else
	{
		SharedParameters.OffsetClustersArgsSWHW = GraphBuilder.CreateSRV(MainRasterizeArgsSWHW);
		SharedParameters.VisibleClustersArgsSWHW = GraphBuilder.CreateUAV(PostRasterizeArgsSWHW);
	}

	SharedParameters.OutVisibleClustersSWHW = GraphBuilder.CreateUAV(VisibleClustersSWHW);
	SharedParameters.InOutAssemblyTransforms = GraphBuilder.CreateUAV(AssemblyTransformsBuffer);
	SharedParameters.OutStreamingRequests = GraphBuilder.CreateUAV(StreamingRequests);
	SharedParameters.VirtualShadowMap = VirtualTargetParameters;

	if (StatsBuffer)
	{
		SharedParameters.OutStatsBuffer = StatsBufferSkipBarrierUAV;
	}

	if (IsDebuggingEnabled())
	{
		FRDGBufferRef DebugBuffer = nullptr;
		if ((DebugFlags & NANITE_DEBUG_FLAG_WRITE_ASSEMBLY_META) != 0)
		{
			DebugBuffer = AssemblyMetaBuffer;
		}
		else
		{			
			DebugBuffer = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateByteAddressDesc(4u), TEXT("Nanite.DummyDebugBuffer"));
		}
		SharedParameters.OutDebugBuffer = GraphBuilder.CreateUAV(DebugBuffer);
	}
	else
	{
		SharedParameters.OutDebugBuffer = nullptr;
	}

	SharedParameters.LargePageRectThreshold = CVarLargePageRectThreshold.GetValueOnRenderThread();
	SharedParameters.StreamingRequestsBufferVersion = GStreamingManager.GetStreamingRequestsBufferVersion();
	SharedParameters.StreamingRequestsBufferSize = StreamingRequests->Desc.NumElements;
	SharedParameters.DepthBucketsMinZ = CVarNaniteDepthBucketsMinZ.GetValueOnRenderThread();
	SharedParameters.DepthBucketsMaxZ = CVarNaniteDepthBucketsMaxZ.GetValueOnRenderThread();

	check(ViewsBuffer);

	if (CVarNanitePersistentThreadsCulling.GetValueOnRenderThread())
	{
		AddPass_NodeAndClusterCull(
			RDG_EVENT_NAME("NodeAndClusterCull"),
			SharedParameters,
			nullptr,
			nullptr,
			0u,
			CullingPass,
			NANITE_CULLING_TYPE_PERSISTENT_NODES_AND_CLUSTERS);
	}
	else
	{
		RDG_EVENT_SCOPE(GraphBuilder, "NodeAndClusterCull");

		
		// Ping-pong between two sets of indirect args to get around that indirect args resource state is read-only.
		FRDGBufferRef NodeCullArgs0 = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateIndirectDesc((NANITE_MAX_CLUSTER_HIERARCHY_DEPTH + 1) * NANITE_NODE_CULLING_ARG_COUNT), TEXT("Nanite.CullArgs0"));
		FRDGBufferRef NodeCullArgs1 = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateIndirectDesc((NANITE_MAX_CLUSTER_HIERARCHY_DEPTH + 1) * NANITE_NODE_CULLING_ARG_COUNT), TEXT("Nanite.CullArgs1"));

		FRDGBufferUAVRef QueueStateUAV = GraphBuilder.CreateUAV(QueueState);

		AddPass_InitNodeCullArgs(GraphBuilder, SharedContext.ShaderMap, RDG_EVENT_NAME("InitNodeCullArgs"), QueueStateUAV, NodeCullArgs0, NodeCullArgs1, CullingPass);

		const uint32 MaxLevels = Nanite::GStreamingManager.GetMaxHierarchyLevels();
		for (uint32 NodeLevel = 0; NodeLevel < MaxLevels; NodeLevel++)
		{
			AddPass_NodeAndClusterCull(
				RDG_EVENT_NAME("NodeCull_%d", NodeLevel),
				SharedParameters,
				(NodeLevel & 1) ? NodeCullArgs1 : NodeCullArgs0,
				(NodeLevel & 1) ? NodeCullArgs0 : NodeCullArgs1,
				NodeLevel,
				CullingPass,
				NANITE_CULLING_TYPE_NODES);
		}

		FRDGBufferRef ClusterCullArgs = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateIndirectDesc(3), TEXT("Nanite.ClusterCullArgs"));
		AddPass_InitClusterCullArgs(GraphBuilder, SharedContext.ShaderMap, RDG_EVENT_NAME("InitClusterCullArgs"), QueueStateUAV, ClusterCullArgs, CullingPass);

		AddPass_NodeAndClusterCull(
			RDG_EVENT_NAME("ClusterCull"),
			SharedParameters,
			ClusterCullArgs,
			nullptr,
			0,
			CullingPass,
			NANITE_CULLING_TYPE_CLUSTERS);
	}
}

void FRenderer::AddPass_InstanceHierarchyAndClusterCull( uint32 CullingPass )
{
	LLM_SCOPE_BYTAG(Nanite);

	checkf(GRHIPersistentThreadGroupCount > 0, TEXT("GRHIPersistentThreadGroupCount must be configured correctly in the RHI."));

	FRDGBufferRef Dummy = GSystemTextures.GetDefaultStructuredBuffer(GraphBuilder, 8);

	{
		RDG_EVENT_SCOPE(GraphBuilder, "InstanceCulling");

		FInstanceWorkGroupParameters InstanceWorkGroupParameters;
		// Run hierarchical instance culling pass
		if (InstanceHierarchyDriver.IsEnabled())
		{
			InstanceWorkGroupParameters = InstanceHierarchyDriver.DispatchCullingPass(GraphBuilder, CullingPass, *this);
		}

		// make sure the passes can overlap
		FRDGBufferUAVRef QueueStateSkipBarrierUAV = GraphBuilder.CreateUAV( QueueState, ERDGUnorderedAccessViewFlags::SkipBarrier );
		FRDGBufferUAVRef CandidateNodesUAV = GraphBuilder.CreateUAV( CandidateNodesBuffer, ERDGUnorderedAccessViewFlags::SkipBarrier );
		FRDGBufferUAVRef OccludedInstancesSkipBarrierUAV = nullptr;
		FRDGBufferUAVRef OccludedInstancesArgsSkipBarrierUAV = nullptr;
	
		if (CullingPass == CULLING_PASS_OCCLUSION_MAIN)
		{
			OccludedInstancesSkipBarrierUAV = GraphBuilder.CreateUAV( OccludedInstances, ERDGUnorderedAccessViewFlags::SkipBarrier );
			OccludedInstancesArgsSkipBarrierUAV = GraphBuilder.CreateUAV( OccludedInstancesArgs, ERDGUnorderedAccessViewFlags::SkipBarrier );
		}

		auto DispatchInstanceCullPass = [&](const FInstanceWorkGroupParameters& InstanceWorkGroupParameters, TOptional<uint32> MaxInstanceWorkGroupsOverride = {}, bool bIsExplicitDraw = false)
		{
			FInstanceCull_CS::FParameters SharedParameters;

			SharedParameters.NumInstances						= NumInstancesPreCull;
			SharedParameters.MaxNodes							= Nanite::FGlobalResources::GetMaxNodes();
			SharedParameters.ImposterMaxPixels					= CVarNaniteImposterMaxPixels.GetValueOnRenderThread();
			SharedParameters.IsExplicitDraw						= bIsExplicitDraw ? 1 : 0;

			SharedParameters.Scene = SceneUniformBuffer;
			SharedParameters.RasterParameters = RasterContext.Parameters;
			SharedParameters.CullingParameters = CullingParameters;

			SharedParameters.ImposterAtlas = Nanite::GStreamingManager.GetImposterDataSRV(GraphBuilder);

			SharedParameters.OutQueueState = QueueStateSkipBarrierUAV;

			SharedParameters.VirtualShadowMap = VirtualTargetParameters;

			if (StatsBuffer)
			{
				SharedParameters.OutStatsBuffer					= StatsBufferSkipBarrierUAV;
			}

			SharedParameters.OutCandidateNodes = CandidateNodesUAV;
			if (CullingPass == CULLING_PASS_NO_OCCLUSION)
			{
				if( InstanceDrawsBuffer )
				{
					SharedParameters.InInstanceDraws			= GraphBuilder.CreateSRV( InstanceDrawsBuffer );
				}
			}
			else if (CullingPass == CULLING_PASS_OCCLUSION_MAIN)
			{
				SharedParameters.OutOccludedInstances		= OccludedInstancesSkipBarrierUAV;
				SharedParameters.OutOccludedInstancesArgs	= OccludedInstancesArgsSkipBarrierUAV;
			}
			else if (!IsValid(InstanceWorkGroupParameters))
			{
				SharedParameters.InInstanceDraws				= GraphBuilder.CreateSRV( OccludedInstances );
				SharedParameters.InOccludedInstancesArgs		= GraphBuilder.CreateSRV( OccludedInstancesArgs );
			}

			SharedParameters.InstanceWorkGroupParameters = InstanceWorkGroupParameters;

			if (PrimitiveFilterBuffer)
			{
				SharedParameters.InPrimitiveFilterBuffer		= GraphBuilder.CreateSRV(PrimitiveFilterBuffer);
			}

			check(ViewsBuffer);
			const bool bUseExplicitListCullingPass = InstanceDrawsBuffer != nullptr;
			const uint32 InstanceCullingPass = bUseExplicitListCullingPass ? CULLING_PASS_EXPLICIT_LIST : CullingPass;
			FInstanceCull_CS::FPermutationDomain PermutationVector;
			PermutationVector.Set<FInstanceCull_CS::FCullingPassDim>(InstanceCullingPass);
			PermutationVector.Set<FInstanceCull_CS::FMultiViewDim>(bMultiView);
			PermutationVector.Set<FInstanceCull_CS::FPrimitiveFilterDim>(PrimitiveFilterBuffer != nullptr);
			PermutationVector.Set<FInstanceCull_CS::FDebugFlagsDim>(IsDebuggingEnabled());
			PermutationVector.Set<FInstanceCull_CS::FDepthOnlyDim>(RasterContext.RasterMode == EOutputBufferMode::DepthOnly);
			// Make sure these permutations are orthogonally enabled WRT CULLING_PASS_EXPLICIT_LIST as they can never co-exist
			check(!(IsUsingVirtualShadowMap() && bUseExplicitListCullingPass));
			check(!(IsValid(InstanceWorkGroupParameters) && bUseExplicitListCullingPass));
			PermutationVector.Set<FInstanceCull_CS::FVirtualTextureTargetDim>(IsUsingVirtualShadowMap() && !bUseExplicitListCullingPass);
			PermutationVector.Set<FInstanceCull_CS::FMaterialCacheDim>(IsMaterialCache());
			bool bGroupWorkBuffer = IsValid(InstanceWorkGroupParameters) && !bUseExplicitListCullingPass;
			PermutationVector.Set<FInstanceCull_CS::FUseGroupWorkBufferDim>(bGroupWorkBuffer);

			if (bGroupWorkBuffer)
			{
				FInstanceCull_CS::FParameters* PassParameters = GraphBuilder.AllocParameters< FInstanceCull_CS::FParameters >(&SharedParameters);
				PassParameters->IndirectArgs = InstanceWorkGroupParameters.InInstanceWorkArgs->GetParent();

				// Get the general (not specialized for static) CS and use that to clear any unused graph resources. There is no difference between the permutations.
				PermutationVector.Set<FInstanceCull_CS::FStaticGeoDim>(false);
				auto GeneralComputeShader = SharedContext.ShaderMap->GetShader<FInstanceCull_CS>(PermutationVector);
				PermutationVector.Set<FInstanceCull_CS::FStaticGeoDim>(true);
				auto StaticComputeShader = SharedContext.ShaderMap->GetShader<FInstanceCull_CS>(PermutationVector);
				ClearUnusedGraphResources(GeneralComputeShader, PassParameters);

				GraphBuilder.AddPass(
					RDG_EVENT_NAME("InstanceCull - GroupWork"),
					PassParameters,
					ERDGPassFlags::Compute,
					[DeferredSetupContext = InstanceHierarchyDriver.DeferredSetupContext, 
					bAllowStaticGeometryPath = InstanceHierarchyDriver.bAllowStaticGeometryPath,
					MaxInstanceWorkGroupsOverride,
					PassParameters, 
					PermutationVector, 
					GeneralComputeShader, 
					StaticComputeShader](FRDGAsyncTask, FRHIComputeCommandList& RHICmdList) mutable
					{
						PassParameters->MaxInstanceWorkGroups = MaxInstanceWorkGroupsOverride.Get(DeferredSetupContext->GetMaxInstanceWorkGroups());

						// always run the general path, everything gets funneled here if the static path is disabled
						FComputeShaderUtils::DispatchIndirect(RHICmdList, GeneralComputeShader, *PassParameters, PassParameters->IndirectArgs->GetIndirectRHICallBuffer(), 4 * sizeof(uint32));

						// Run the static dispatch after to bias the more expensive clusters to the start of the queue.
						if (bAllowStaticGeometryPath)
						{
							FComputeShaderUtils::DispatchIndirect(RHICmdList, StaticComputeShader, *PassParameters, PassParameters->IndirectArgs->GetIndirectRHICallBuffer(), 0);
						}
					});
			}
			else 
			{
				auto ComputeShader = SharedContext.ShaderMap->GetShader<FInstanceCull_CS>(PermutationVector);
				FInstanceCull_CS::FParameters* PassParameters = GraphBuilder.AllocParameters< FInstanceCull_CS::FParameters >(&SharedParameters);
				if (InstanceCullingPass == CULLING_PASS_OCCLUSION_POST)
				{
					PassParameters->IndirectArgs = OccludedInstancesArgs;
					FComputeShaderUtils::AddPass(
						GraphBuilder,
						RDG_EVENT_NAME( "InstanceCull" ),
						ComputeShader,
						PassParameters,
						PassParameters->IndirectArgs,
						0
					);
				}
				else
				{
					FComputeShaderUtils::AddPass(
						GraphBuilder,
						InstanceCullingPass == CULLING_PASS_EXPLICIT_LIST ? RDG_EVENT_NAME("InstanceCull - Explicit List") : RDG_EVENT_NAME("InstanceCull"),
						ComputeShader,
						PassParameters,
						FComputeShaderUtils::GetGroupCountWrapped(NumInstancesPreCull, 64)
					);
				}
			}
		};

		if (ExplicitChunkDrawInfo)
		{
			check(InstanceWorkGroupParameters.InViewDrawRanges);

			TArray<uint32> InstanceWorkArgs;
			InstanceWorkArgs.SetNumZeroed(8);
			InstanceWorkArgs[4] = ExplicitChunkDrawInfo->NumChunks;
			InstanceWorkArgs[5] = 1;
			InstanceWorkArgs[6] = 1;

			FRDGBufferRef IndirectArgsRDG = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateRawIndirectDesc(InstanceWorkArgs.Num() * sizeof(InstanceWorkArgs[0])), TEXT("Nanite.ExplicitChunkInstanceWorkArgs"));
			GraphBuilder.QueueBufferUpload(IndirectArgsRDG, InstanceWorkArgs.GetData(), InstanceWorkArgs.Num() * sizeof(InstanceWorkArgs[0]));

			FInstanceWorkGroupParameters ExplicitInstanceWorkGroupParameters;
			ExplicitInstanceWorkGroupParameters.InInstanceWorkArgs = GraphBuilder.CreateSRV(IndirectArgsRDG, PF_R32_UINT);
			ExplicitInstanceWorkGroupParameters.InInstanceWorkGroups = GraphBuilder.CreateSRV(ExplicitChunkDrawInfo->ExplicitChunkDraws);
			ExplicitInstanceWorkGroupParameters.InstanceIds = GraphBuilder.CreateSRV(ExplicitChunkDrawInfo->InstanceIds);
			ExplicitInstanceWorkGroupParameters.InViewDrawRanges = InstanceWorkGroupParameters.InViewDrawRanges;
			DispatchInstanceCullPass(ExplicitInstanceWorkGroupParameters, ExplicitChunkDrawInfo->NumChunks, true /*bIsExplicitDraw*/);
		}

		// We need to add an extra pass to cover for the post-pass occluded instances, this is a workaround for an issue where the instances from 
		// pre-pass & hierarchy cull were not able to co-exist in the same args, for obscure reasons. We should perhaps re-merge them.
		if (CullingPass == CULLING_PASS_OCCLUSION_POST && IsValid(InstanceWorkGroupParameters))
		{
			static FInstanceWorkGroupParameters DummyInstanceWorkGroupParameters;
			DispatchInstanceCullPass(DummyInstanceWorkGroupParameters);
		}
		DispatchInstanceCullPass(InstanceWorkGroupParameters);
	}

	AddPass_NodeAndClusterCull( CullingPass );

	{
		FCalculateSafeRasterizerArgs_CS::FParameters* PassParameters = GraphBuilder.AllocParameters< FCalculateSafeRasterizerArgs_CS::FParameters >();

		const bool bPrevDrawData		= (RenderFlags & NANITE_RENDER_FLAG_HAS_PREV_DRAW_DATA) != 0;
		const bool bPostPass			= (CullingPass == CULLING_PASS_OCCLUSION_POST) != 0;

		if (bPrevDrawData)
		{
			PassParameters->InTotalPrevDrawClusters		= GraphBuilder.CreateSRV(TotalPrevDrawClustersBuffer);
		}
		else
		{
			PassParameters->InTotalPrevDrawClusters		= GraphBuilder.CreateSRV(Dummy);
		}

		if (bPostPass)
		{
			PassParameters->OffsetClustersArgsSWHW		= GraphBuilder.CreateSRV(MainRasterizeArgsSWHW);
			PassParameters->InRasterizerArgsSWHW		= GraphBuilder.CreateSRV(PostRasterizeArgsSWHW);
			PassParameters->OutSafeRasterizerArgsSWHW	= GraphBuilder.CreateUAV(SafePostRasterizeArgsSWHW);
		}
		else
		{
			PassParameters->InRasterizerArgsSWHW		= GraphBuilder.CreateSRV(MainRasterizeArgsSWHW);
			PassParameters->OutSafeRasterizerArgsSWHW	= GraphBuilder.CreateUAV(SafeMainRasterizeArgsSWHW);
		}

		PassParameters->OutClusterCountSWHW				= GraphBuilder.CreateUAV(ClusterCountSWHW);
		PassParameters->OutClusterClassifyArgs			= GraphBuilder.CreateUAV(ClusterClassifyArgs);
		
		PassParameters->MaxVisibleClusters				= Nanite::FGlobalResources::GetMaxVisibleClusters();
		PassParameters->RenderFlags						= RenderFlags;
		
		FCalculateSafeRasterizerArgs_CS::FPermutationDomain PermutationVector;
		PermutationVector.Set<FCalculateSafeRasterizerArgs_CS::FIsPostPass>(bPostPass);

		auto ComputeShader = SharedContext.ShaderMap->GetShader< FCalculateSafeRasterizerArgs_CS >(PermutationVector);

		FComputeShaderUtils::AddPass(
			GraphBuilder,
			RDG_EVENT_NAME("CalculateSafeRasterizerArgs"),
			ComputeShader,
			PassParameters,
			FIntVector(1, 1, 1)
		);
	}
}
```

* Engine\Source\Runtime\Render\Private\Lumen\LumenSceneDirectLighting.cpp

```Cpp
void FDeferredShadingSceneRenderer::RenderDirectLightingForLumenScene(
	FRDGBuilder& GraphBuilder,
	const FLumenSceneFrameTemporaries& FrameTemporaries,
	const FLumenDirectLightingTaskData* LightingTaskData,
	const FLumenCardUpdateContext& CardUpdateContext,
	ERDGPassFlags ComputePassFlags)
{
	LLM_SCOPE_BYTAG(Lumen);

	if (LightingTaskData)
	{
		LightingTaskData->Task.Wait();
	}

	if (CVarLumenLumenSceneDirectLighting.GetValueOnRenderThread() != 0 && CardUpdateContext.MaxUpdateTiles > 0)
	{
		RDG_EVENT_SCOPE(GraphBuilder, "DirectLighting");
		QUICK_SCOPE_CYCLE_COUNTER(RenderDirectLightingForLumenScene);

		check(LightingTaskData);
		const FViewInfo& MainView = Views[0];
		FLumenSceneData& LumenSceneData = *Scene->GetLumenSceneData(Views[0]);

		int32 NumViewOrigins = FrameTemporaries.ViewOrigins.Num();

		TRDGUniformBufferRef<FLumenCardScene> LumenCardSceneUniformBuffer = FrameTemporaries.LumenCardSceneUniformBuffer;

		TConstArrayView<FLumenGatheredLight> GatheredLights = LightingTaskData->GatheredLights;
		const bool bHasRectLights = LightingTaskData->bHasRectLights;

		FRDGBufferRef LumenPackedLights = CreateStructuredBuffer(GraphBuilder, TEXT("Lumen.DirectLighting.Lights"), LightingTaskData->PackedLightData, ERDGInitialDataFlags::NoCopy);
		FRDGBufferRef LumenLightInfluenceSpheres = CreateStructuredBuffer(GraphBuilder, TEXT("Lumen.DirectLighting.LightInfluenceSpheres"), LightingTaskData->LightInfluenceSpheres, ERDGInitialDataFlags::NoCopy);

		LumenSceneDirectLighting::FLightDataParameters LumenLightData;
		LumenLightData.LumenPackedLights = GraphBuilder.CreateSRV(LumenPackedLights);
		LumenLightData.LumenLightInfluenceSpheres = GraphBuilder.CreateSRV(LumenLightInfluenceSpheres);

		const bool bUseHardwareRayTracedDirectLighting = Lumen::UseHardwareRayTracedDirectLighting(ViewFamily);

		// Experimental Stochastic lighting path.
		if (LumenSceneDirectLighting::UseStochasticLighting(ViewFamily))
		{
			ComputeStochasticLighting(GraphBuilder, Scene, Views[0], FrameTemporaries, LightingTaskData, CardUpdateContext, ComputePassFlags, LumenLightData);
			return;
		}

		FLightTileCullContext CullContext;
		FLumenCardTileUpdateContext CardTileUpdateContext;
		CullDirectLightingTiles(GraphBuilder, Views, FrameTemporaries, CardUpdateContext, LumenCardSceneUniformBuffer, GatheredLights, LightingTaskData->StandaloneLightIndices, LumenLightData, CullContext, CardTileUpdateContext, ComputePassFlags);

		// 8 bits per shadow mask texel. But if colored light function atlas is used, then 16bits per shadow mask texel.
		const uint32 ShadowMaskTilesSizeFactor = GetLightFunctionAtlasFormat() > 0 ? 2 : 1;
		const uint32 ShadowMaskTilesSize = FMath::Max(ShadowMaskTilesSizeFactor * 16 * CullContext.MaxCulledCardTiles, 1024u);
		FRDGBufferRef ShadowMaskTiles = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateStructuredDesc(sizeof(uint32), ShadowMaskTilesSize), TEXT("Lumen.DirectLighting.ShadowMaskTiles"));

		// 1 uint per packed shadow trace
		FRDGBufferRef ShadowTraceAllocator = nullptr;
		FRDGBufferRef ShadowTraces = nullptr;
		if (bUseHardwareRayTracedDirectLighting)
		{
			const uint32 MaxShadowTraces = FMath::Max(Lumen::CardTileSize * Lumen::CardTileSize * CullContext.MaxCulledCardTiles, 1024u);

			ShadowTraceAllocator = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateStructuredDesc(sizeof(uint32), 1), TEXT("Lumen.DirectLighting.ShadowTraceAllocator"));
			ShadowTraces = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateStructuredDesc(sizeof(uint32), MaxShadowTraces), TEXT("Lumen.DirectLighting.ShadowTraces"));
			AddClearUAVPass(GraphBuilder, GraphBuilder.CreateUAV(ShadowTraceAllocator), 0, ComputePassFlags);
		}

		// Compute shadow mask based on light attenuation (IES/LightFunction/Distance fall) to reduce need for shadow tracing done after.
		{
			SCOPED_NAMED_EVENT_TEXT("Light Attenuation ShadowMask ", FColor::Green);
			RDG_EVENT_SCOPE_FINAL(GraphBuilder, "Light Attenuation ShadowMask");

			FRDGBufferUAVRef ShadowMaskTilesUAV = GraphBuilder.CreateUAV(ShadowMaskTiles, ERDGUnorderedAccessViewFlags::SkipBarrier);
			FRDGBufferUAVRef ShadowTraceAllocatorUAV = ShadowTraceAllocator ? GraphBuilder.CreateUAV(ShadowTraceAllocator, ERDGUnorderedAccessViewFlags::SkipBarrier) : nullptr;
			FRDGBufferUAVRef ShadowTracesUAV = ShadowTraces ? GraphBuilder.CreateUAV(ShadowTraces, ERDGUnorderedAccessViewFlags::SkipBarrier) : nullptr;
			FRDGBufferSRVRef TileShadowDownsampleFactorAtlasSRV = GraphBuilder.CreateSRV(FrameTemporaries.TileShadowDownsampleFactorAtlas, PF_R32_UINT);

			int32 NumShadowedLights = 0;
			for (int32 OriginIndex = 0; OriginIndex < NumViewOrigins; ++OriginIndex)
			{
				const FViewInfo& View = *FrameTemporaries.ViewOrigins[OriginIndex].ReferenceView;

				NumShadowedLights = ComputeShadowMaskFromLightAttenuation(
					GraphBuilder,
					Scene,
					View,
					LumenCardSceneUniformBuffer,
					GatheredLights,
					LightingTaskData->StandaloneLightIndices,
					LightingTaskData->ViewBatchedLightParameters[OriginIndex],
					CullContext.LightTileScatterParameters,
					LumenLightData,
					OriginIndex,
					NumViewOrigins,
					LightingTaskData->bHasLightFunctions,
					ShadowMaskTilesUAV,
					ShadowTraceAllocatorUAV,
					ShadowTracesUAV,
					TileShadowDownsampleFactorAtlasSRV,
					ComputePassFlags);
			}

			// Clear to mark resource as used if it wasn't ever written to
			if (ShadowTracesUAV && NumShadowedLights == 0)
			{
				AddClearUAVPass(GraphBuilder, ShadowTracesUAV, 0);
			}
		}

		FRDGBufferRef ShadowTraceIndirectArgs = GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateIndirectDesc<FRHIDispatchIndirectParameters>(1), TEXT("Lumen.DirectLighting.CompactedShadowTraceIndirectArgs"));
		if (ShadowTraceAllocator)
		{
			FInitShadowTraceIndirectArgsCS::FParameters* PassParameters = GraphBuilder.AllocParameters<FInitShadowTraceIndirectArgsCS::FParameters>();
			PassParameters->RWShadowTraceIndirectArgs = GraphBuilder.CreateUAV(ShadowTraceIndirectArgs);
			PassParameters->ShadowTraceAllocator = GraphBuilder.CreateSRV(ShadowTraceAllocator);

			auto ComputeShader = Views[0].ShaderMap->GetShader<FInitShadowTraceIndirectArgsCS>();

			FComputeShaderUtils::AddPass(
				GraphBuilder,
				RDG_EVENT_NAME("InitShadowTraceIndirectArgs"),
				ComputePassFlags,
				ComputeShader,
				PassParameters,
				FIntVector(1, 1, 1));
		}

		// Offscreen shadowing
		{
			SCOPED_NAMED_EVENT_TEXT("Offscreen shadows", FColor::Green);
			RDG_EVENT_SCOPE_FINAL(GraphBuilder, "Offscreen shadows");

			FRDGBufferUAVRef ShadowMaskTilesUAV = GraphBuilder.CreateUAV(ShadowMaskTiles, ERDGUnorderedAccessViewFlags::SkipBarrier);

			FDistanceFieldObjectBufferParameters ObjectBufferParameters;

			if (!bUseHardwareRayTracedDirectLighting)
			{
				ObjectBufferParameters = DistanceField::SetupObjectBufferParameters(GraphBuilder, Scene->DistanceFieldSceneData);

				// Patch DF heightfields with Lumen heightfields
				ObjectBufferParameters.SceneHeightfieldObjectBounds = GraphBuilder.CreateSRV(GraphBuilder.RegisterExternalBuffer(LumenSceneData.HeightfieldBuffer));
				ObjectBufferParameters.SceneHeightfieldObjectData = nullptr;
				ObjectBufferParameters.NumSceneHeightfieldObjects = LumenSceneData.Heightfields.Num();
			}

			for (int32 OriginIndex = 0; OriginIndex < NumViewOrigins; ++OriginIndex)
			{
				const FViewInfo& View = *FrameTemporaries.ViewOrigins[OriginIndex].ReferenceView;

				if (bUseHardwareRayTracedDirectLighting)
				{
					FLumenDirectLightingStochasticData StochasticData;
					TraceLumenHardwareRayTracedDirectLightingShadows(
						GraphBuilder,
						Scene,
						View,
						OriginIndex,
						FrameTemporaries,
						StochasticData,
						LumenLightData,
						ShadowTraceIndirectArgs,
						ShadowTraceAllocator,
						ShadowTraces,
						CullContext.LightTileAllocator,
						CullContext.LightTiles,
						ShadowMaskTilesUAV,
						ComputePassFlags);
				}
				else
				{
					TraceDistanceFieldShadows(
						GraphBuilder,
						Scene,
						View,
						LumenCardSceneUniformBuffer,
						GatheredLights,
						LightingTaskData->StandaloneLightIndices,
						LightingTaskData->ViewBatchedLightParameters[OriginIndex],
						CullContext.LightTileScatterParameters,
						LumenLightData,
						ObjectBufferParameters,
						OriginIndex,
						NumViewOrigins,
						ShadowMaskTilesUAV,
						ComputePassFlags);
				}
			}
		}

		// Apply lights
		{
			RDG_EVENT_SCOPE(GraphBuilder, "Lights");

			FRDGBufferSRVRef ShadowMaskTilesSRV = GraphBuilder.CreateSRV(ShadowMaskTiles->HasBeenProduced() ? ShadowMaskTiles : GSystemTextures.GetDefaultStructuredBuffer(GraphBuilder, sizeof(uint32)));
			FRDGBufferSRVRef CardTilesSRV = GraphBuilder.CreateSRV(CardTileUpdateContext.CardTiles);
			FRDGBufferSRVRef LightTileOffsetNumPerCardTileSRV = GraphBuilder.CreateSRV(CullContext.LightTileOffsetNumPerCardTile);
			FRDGBufferSRVRef LightTilesPerCardTileSRV = GraphBuilder.CreateSRV(CullContext.LightTilesPerCardTile);
			FRDGTextureUAVRef DirectLightingAtlasUAV = GraphBuilder.CreateUAV(FrameTemporaries.DirectLightingAtlas);

			RenderDirectLightIntoLumenCardsBatched(
				GraphBuilder,
				Views,
				FrameTemporaries,
				LumenCardSceneUniformBuffer,
				LumenLightData,
				ShadowMaskTilesSRV,
				CardTilesSRV,
				LightTileOffsetNumPerCardTileSRV,
				LightTilesPerCardTileSRV,
				DirectLightingAtlasUAV,
				CardTileUpdateContext.DispatchCardTilesIndirectArgs,
				bHasRectLights,
				ComputePassFlags);
		}

		// Update Final Lighting
		Lumen::CombineLumenSceneLighting(
			Scene,
			MainView,
			GraphBuilder,
			FrameTemporaries,
			CardUpdateContext,
			CardTileUpdateContext,
			ComputePassFlags);

		// Draw direct lighting stats & Lumen cards/tiles
		if (GetLumenLightingStatMode() == 3)
		{
			AddLumenSceneDirectLightingStatsPass(
				GraphBuilder,
				Scene,
				MainView,
				FrameTemporaries,
				LightingTaskData,
				CardUpdateContext,
				CardTileUpdateContext,
				ShadowTraceAllocator,
				ComputePassFlags);
		}
	}
	else if (CVarLumenLumenSceneDirectLighting.GetValueOnRenderThread() == 0)
	{
		AddClearRenderTargetPass(GraphBuilder, FrameTemporaries.DirectLightingAtlas);
	}
}
```

* Engine\Source\Runtime\Renderer\Private\Lumen\LumenSceneLighting.cpp

```Cpp
void FDeferredShadingSceneRenderer::RenderLumenSceneLighting(
	FRDGBuilder& GraphBuilder,
	const FLumenSceneFrameTemporaries& FrameTemporaries,
	const FLumenDirectLightingTaskData* DirectLightingTaskData)
{
	LLM_SCOPE_BYTAG(Lumen);
	TRACE_CPUPROFILER_EVENT_SCOPE(FDeferredShadingSceneRenderer::RenderLumenSceneLighting);

	FLumenSceneData& LumenSceneData = *Scene->GetLumenSceneData(Views[0]);

	bool bAnyLumenActive = false;

	for (const FViewInfo& View : Views)
	{
		const FPerViewPipelineState& ViewPipelineState = GetViewPipelineState(View);
		bAnyLumenActive = bAnyLumenActive || ViewPipelineState.DiffuseIndirectMethod == EDiffuseIndirectMethod::Lumen;
	}

	if (bAnyLumenActive)
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(RenderLumenSceneLighting);
		QUICK_SCOPE_CYCLE_COUNTER(RenderLumenSceneLighting);
		RDG_EVENT_SCOPE_STAT(GraphBuilder, LumenSceneLighting, "LumenSceneLighting%s", LumenCardRenderer.bPropagateGlobalLightingChange ? TEXT(" PROPAGATE GLOBAL CHANGE!") : TEXT(""));
		RDG_GPU_STAT_SCOPE(GraphBuilder, LumenSceneLighting);

		const ERDGPassFlags ComputePassFlags = LumenSceneLighting::UseAsyncCompute(ViewFamily) ? ERDGPassFlags::AsyncCompute : ERDGPassFlags::Compute;

		LumenSceneData.IncrementSurfaceCacheUpdateFrameIndex();

		if (LumenSceneData.bDebugClearAllCachedState)
		{
			AddClearRenderTargetPass(GraphBuilder, FrameTemporaries.DirectLightingAtlas);
			AddClearRenderTargetPass(GraphBuilder, FrameTemporaries.IndirectLightingAtlas);
			AddClearRenderTargetPass(GraphBuilder, FrameTemporaries.RadiosityNumFramesAccumulatedAtlas);
			AddClearRenderTargetPass(GraphBuilder, FrameTemporaries.FinalLightingAtlas);
			if (FrameTemporaries.DiffuseLightingAndSecondMomentHistoryAtlas)
			{
				AddClearRenderTargetPass(GraphBuilder, FrameTemporaries.DiffuseLightingAndSecondMomentHistoryAtlas);
			}
			if (FrameTemporaries.NumFramesAccumulatedHistoryAtlas)
			{
				AddClearRenderTargetPass(GraphBuilder, FrameTemporaries.NumFramesAccumulatedHistoryAtlas);
			}
		}

		LumenRadiosity::FFrameTemporaries RadiosityFrameTemporaries;
		LumenRadiosity::InitFrameTemporaries(GraphBuilder, LumenSceneData, ViewFamily, Views, RadiosityFrameTemporaries);

		FLumenCardUpdateContext DirectLightingCardUpdateContext;
		FLumenCardUpdateContext IndirectLightingCardUpdateContext;
		Lumen::BuildCardUpdateContext(
			GraphBuilder,
			LumenSceneData,
			Views,
			FrameTemporaries,
			RadiosityFrameTemporaries.bIndirectLightingHistoryValid,
			DirectLightingCardUpdateContext,
			IndirectLightingCardUpdateContext,
			ComputePassFlags);

		// Pointing cards debug data
		if (GetLumenLightingStatMode() > 2)
		{
			FLumenSceneFrameTemporaries* NonCstFrameTemporaries = const_cast<FLumenSceneFrameTemporaries*>(&FrameTemporaries);
			NonCstFrameTemporaries->DebugData = TraceLumenHardwareRayTracedDebug(GraphBuilder, Scene, Views[0], 0 /*ViewIndex*/, FrameTemporaries, ComputePassFlags);
		}

		RenderDirectLightingForLumenScene(
			GraphBuilder,
			FrameTemporaries,
			DirectLightingTaskData,
			DirectLightingCardUpdateContext,
			ComputePassFlags);

		RenderRadiosityForLumenScene(
			GraphBuilder,
			FrameTemporaries,
			RadiosityFrameTemporaries,
			IndirectLightingCardUpdateContext,
			ComputePassFlags);

		LumenSceneData.bFinalLightingAtlasContentsValid = true;

		GraphBuilder.FlushSetupQueue();
	}
}
```

# Animation(动画)

## Animation Notify

> Animation Notifications (Animation Notifies or just Notifies) provide a way for you to create repeatable events synchronized to Animation Sequences. These events can be sounds (such as footsteps for walk or run animations), spawning particles, and other types. Animation Notifies have any number of different uses, and the system can be extended with custom types. 

动画通知是一种可以插在动画蓝图的帧中并且广播事件的类，可以用来在动画中的某一帧广播事件触发逻辑，比如动画中角色脚落地就触发通知，使音效系统开始进行同步工作，播放声音。

可以通过继承相关的类来扩展动画通知类。

## Notify

> The most basic kind of Animation Notify you can create is simply called a Notify, which causes different pre-made events to be triggered at a specified time. The following Notifies can be found when viewing the Add Notify… menu. Selecting one will create the Notify keyframe at your cursor position.

简单的动画通知类型，可以使用各种内置的一次性事件，一般只会在触发时发挥作用。

## Notify State

> Notify States work similar to standard Notifies, however they operate over a duration, rather than a single event. Because of this, they provide three distinct events: a start, an update, and an end. These events can be accessed when creating Notify State child classes. 

相比Notify，它会发挥作用一段时间，然后在停止发挥作用时再广播一次事件。同样会提供与Notify不同的内置事件。

## Header Source

* Engine\Source\Runtime\Engine\Classes\Animation\AnimNotifies\AnimNotify.h
```Cpp
UCLASS(abstract, Blueprintable, const, hidecategories=Object, collapsecategories)
class ENGINE_API UAnimNotify : public UObject
{
	GENERATED_UCLASS_BODY()

	/** 
	 * Implementable event to get a custom name for the notify
	 */
	UFUNCTION(BlueprintNativeEvent)
	FString GetNotifyName() const;

	UFUNCTION(BlueprintImplementableEvent, meta=(AutoCreateRefTerm="EventReference"))
	bool Received_Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, const FAnimNotifyEventReference& EventReference) const;

#if WITH_EDITORONLY_DATA
	/** Color of Notify in editor */
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotify)
	FColor NotifyColor;

	/** Whether this notify instance should fire in animation editors */
	UPROPERTY(EditAnywhere, BlueprintReadWrite, AdvancedDisplay, Category=AnimNotify)
	bool bShouldFireInEditor;
#endif // WITH_EDITORONLY_DATA

#if WITH_EDITOR
	virtual void OnAnimNotifyCreatedInEditor(FAnimNotifyEvent& ContainingAnimNotifyEvent) {};
	virtual bool CanBePlaced(UAnimSequenceBase* Animation) const { return true; }
	virtual void ValidateAssociatedAssets() {}

	/** Override this to prevent firing this notify type in animation editors */
	virtual bool ShouldFireInEditor() { return bShouldFireInEditor; }
#endif

	UE_DEPRECATED(5.0, "Please use the other Notify function instead")
	virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation);
    // 一般都需要覆写此类来达到业务需求
	virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, const FAnimNotifyEventReference& EventReference);
	virtual void BranchingPointNotify(FBranchingPointNotifyPayload& BranchingPointPayload);

	// @todo document 
	virtual FString GetEditorComment() 
	{ 
		return TEXT(""); 
	}

	/** TriggerWeightThreshold to use when creating notifies of this type */
	UFUNCTION(BlueprintNativeEvent)
	float GetDefaultTriggerWeightThreshold() const;

	// @todo document 
	virtual FLinearColor GetEditorColor() 
	{ 
#if WITH_EDITORONLY_DATA
		return FLinearColor(NotifyColor); 
#else
		return FLinearColor::Black;
#endif // WITH_EDITORONLY_DATA
	}

	/**
	 * We don't instance UAnimNotify objects along with the animations they belong to, but
	 * we still need a way to see which world this UAnimNotify is currently operating on.
	 * So this retrieves a contextual world pointer, from the triggering animation/mesh.  
	 * 
	 * @return NULL if this isn't in the middle of a Received_Notify(), otherwise it's the world belonging to the Mesh passed to Received_Notify()
	 */
	virtual class UWorld* GetWorld() const override;

	/** UObject Interface */
	virtual void PostLoad() override;
	PRAGMA_DISABLE_DEPRECATION_WARNINGS // Suppress compiler warning on override of deprecated function
	UE_DEPRECATED(5.0, "Use version that takes FObjectPreSaveContext instead.")
	virtual void PreSave(const class ITargetPlatform* TargetPlatform) override;
	PRAGMA_ENABLE_DEPRECATION_WARNINGS
	virtual void PreSave(FObjectPreSaveContext ObjectSaveContext) override;
	/** End UObject Interface */

	/** This notify is always a branching point when used on Montages. */
	bool bIsNativeBranchingPoint;

protected:
	UObject* GetContainingAsset() const;

private:
	/* The mesh we're currently triggering a UAnimNotify for (so we can retrieve per instance information) */
	class USkeletalMeshComponent* MeshContext;
};
```

* Engine\Source\Runtime\RenderCore\Public\RenderGraphBuilder.h

```Cpp
	/** Initializes various view types. Assumes that the underlying RHI viewable resource type is assigned. */
	void InitViewRHI(FRHICommandListBase& RHICmdList, FRDGView* View);
	void InitBufferViewRHI(FRHICommandListBase& RHICmdList, FRDGBufferSRV* SRV);
	void InitBufferViewRHI(FRHICommandListBase& RHICmdList, FRDGBufferUAV* UAV);
	void InitTextureViewRHI(FRHICommandListBase& RHICmdList, FRDGTextureSRV* SRV);
	void InitTextureViewRHI(FRHICommandListBase& RHICmdList, FRDGTextureUAV* UAV);
```

* Engine\Source\Runtime\Engine\Classes\Animation\AnimNotifies\AnimNotifyState.h
```Cpp
UCLASS(abstract, editinlinenew, Blueprintable, const, hidecategories=Object, collapsecategories, meta=(ShowWorldContextPin))
class ENGINE_API UAnimNotifyState : public UObject
{
	GENERATED_UCLASS_BODY()

	/** 
	 * Implementable event to get a custom name for the notify
	 */
	UFUNCTION(BlueprintNativeEvent)
	FString GetNotifyName() const;

	UFUNCTION(BlueprintImplementableEvent, meta=(AutoCreateRefTerm="EventReference"))
	bool Received_NotifyBegin(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, float TotalDuration, const FAnimNotifyEventReference& EventReference) const;
	
	UFUNCTION(BlueprintImplementableEvent, meta=(AutoCreateRefTerm="EventReference"))
	bool Received_NotifyTick(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, float FrameDeltaTime, const FAnimNotifyEventReference& EventReference) const;

	UFUNCTION(BlueprintImplementableEvent, meta=(AutoCreateRefTerm="EventReference"))
	bool Received_NotifyEnd(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, const FAnimNotifyEventReference& EventReference) const;

#if WITH_EDITORONLY_DATA
	/** Color of Notify in editor */
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotify)
	FColor NotifyColor;
	
	/** Whether this notify state instance should fire in animation editors */
	UPROPERTY(EditAnywhere, BlueprintReadWrite, AdvancedDisplay, Category=AnimNotify)
	bool bShouldFireInEditor;
#endif // WITH_EDITORONLY_DATA

#if WITH_EDITOR
	virtual void OnAnimNotifyCreatedInEditor(FAnimNotifyEvent& ContainingAnimNotifyEvent) {};
	virtual bool CanBePlaced(UAnimSequenceBase* Animation) const { return true; }
	virtual void ValidateAssociatedAssets() {}

	/** Override this to prevent firing this notify state type in animation editors */
	virtual bool ShouldFireInEditor() { return bShouldFireInEditor; }
#endif

	UE_DEPRECATED(5.0, "This function is deprecated. Use the other NotifyBegin instead.")
	virtual void NotifyBegin(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, float TotalDuration);
	UE_DEPRECATED(5.0, "This function is deprecated. Use the other NotifyTick instead.")
	virtual void NotifyTick(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, float FrameDeltaTime);
	UE_DEPRECATED(5.0, "This function is deprecated. Use the other NotifyEnd instead.")
	virtual void NotifyEnd(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation);

	virtual void NotifyBegin(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, float TotalDuration, const FAnimNotifyEventReference& EventReference);
	virtual void NotifyTick(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, float FrameDeltaTime, const FAnimNotifyEventReference& EventReference);
	virtual void NotifyEnd(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, const FAnimNotifyEventReference& EventReference);
	
	virtual void BranchingPointNotifyBegin(FBranchingPointNotifyPayload& BranchingPointPayload);
	virtual void BranchingPointNotifyTick(FBranchingPointNotifyPayload& BranchingPointPayload, float FrameDeltaTime);
	virtual void BranchingPointNotifyEnd(FBranchingPointNotifyPayload& BranchingPointPayload);

	// @todo document 
	virtual FString GetEditorComment() 
	{ 
		return TEXT(""); 
	}

	/** TriggerWeightThreshold to use when creating notifies of this type */
	UFUNCTION(BlueprintNativeEvent)
	float GetDefaultTriggerWeightThreshold() const;

	// @todo document 
	virtual FLinearColor GetEditorColor() 
	{ 
#if WITH_EDITORONLY_DATA
		return FLinearColor(NotifyColor); 
#else
		return FLinearColor::Black;
#endif // WITH_EDITORONLY_DATA
	}

	/** UObject Interface */
	virtual void PostLoad() override;
	virtual void PreSave(FObjectPreSaveContext ObjectSaveContext) override;
	/** End UObject Interface */

	/** This notify is always a branching point when used on Montages. */
	bool bIsNativeBranchingPoint;

protected:
	UObject* GetContainingAsset() const;
};
```

## Inverse Kinematics(反向运动学)和Forward Kinematics(正向运动学)

动画一般都是以角色在平地为基准制作的，但是游戏中的地形其实会有很多不平坦的地形，反向运动学是为了解决这种反常理的情况出现的。
普通的动画是正向运动学。输入一组局部姿势，输出一个全局动作和每个关节的蒙皮矩阵。而反向运动学相反，是某个关节想要的全局动作作为输入，求出其他部位的局部姿势，使得全局动作得以表现。
典型的例子可以有FPS中角色持枪，让左右手都以正常的姿势持有武器。角色运动中遇到斜坡上升，让角色的落脚点与地形贴合。

## Rag Doll(布娃娃)

角色死亡或失去意识时的常用技术，与物理模块相关。

## Control Rig(控制绑定)

像是给美术师和动画师的工具，允许直接在引擎中给角色添加和调试动画。

# Multiplay

Unreal Engine支持开发网络游戏，网络架构上使用C/S架构，网络同步部分高度集成状态同步支持，没有原生帧同步技术支持。从Actor开始集成了网络复制支持，提供三种远程过程调用(Remote Procedure Call)。

## Dedicated Server(专用服务器)

仅负责游戏逻辑运算和网络通信并且没有渲染模块的Unreal Engine程序。

## Unreal Engine传输层

Unreal Engine为了低延迟使用UDP协议，但是UDP存在不可靠性，Unreal Engine选择基于UDP，在应用层建立了一套状态机和确认机制维护可靠性。

### 具体实现(施工中)

* Engine\Source\Runtime\Engine\Classes\Engine\NetDriver.h
```Cpp
UCLASS(Abstract, customConstructor, transient, MinimalAPI, config=Engine)
class UNetDriver : public UObject, public FExec
{
	GENERATED_UCLASS_BODY()
	...

	/**
	 * Common initialization between server and client connection setup
	 * 
	 * @param bInitAsClient are we a client or server
	 * @param InNotify notification object to associate with the net driver
	 * @param URL destination
	 * @param bReuseAddressAndPort whether to allow multiple sockets to be bound to the same address/port
	 * @param Error output containing an error string on failure
	 *
	 * @return true if successful, false otherwise (check Error parameter)
	 */
	ENGINE_API virtual bool InitBase(bool bInitAsClient, FNetworkNotify* InNotify, const FURL& URL, bool bReuseAddressAndPort, FString& Error);

	/**
	 * Initialize the net driver in client mode
	 *
	 * @param InNotify notification object to associate with the net driver
	 * @param ConnectURL remote ip:port of host to connect to
	 * @param Error resulting error string from connection attempt
	 * 
	 * @return true if successful, false otherwise (check Error parameter)
	 */
	virtual bool InitConnect(class FNetworkNotify* InNotify, const FURL& ConnectURL, FString& Error ) PURE_VIRTUAL( UNetDriver::InitConnect, return true;);

	/**
	 * Initialize the network driver in server mode (listener)
	 *
	 * @param InNotify notification object to associate with the net driver
	 * @param ListenURL the connection URL for this listener
	 * @param bReuseAddressAndPort whether to allow multiple sockets to be bound to the same address/port
	 * @param Error out param with any error messages generated 
	 *
	 * @return true if successful, false otherwise (check Error parameter)
	 */
	virtual bool InitListen(class FNetworkNotify* InNotify, FURL& ListenURL, bool bReuseAddressAndPort, FString& Error) PURE_VIRTUAL( UNetDriver::InitListen, return true;);
	...
}
```

* Engine\Source\Runtime\Engine\Private\NetDriver.cpp
```Cpp
bool UNetDriver::InitConnectionClass(void)
{
	if (NetConnectionClass == NULL && NetConnectionClassName != TEXT(""))
	{
		NetConnectionClass = LoadClass<UNetConnection>(NULL,*NetConnectionClassName,NULL,LOAD_None,NULL);
		if (NetConnectionClass == NULL)
		{
			UE_LOG(LogNet, Error,TEXT("Failed to load class '%s'"),*NetConnectionClassName);
		}
	}

	ChildNetConnectionClass = UChildConnection::StaticClass();

	return NetConnectionClass != NULL;
}

bool UNetDriver::InitBase(bool bInitAsClient, FNetworkNotify* InNotify, const FURL& URL, bool bReuseAddressAndPort, FString& Error)
{
	// Read any timeout overrides from the URL
	if (const TCHAR* InitialConnectTimeoutOverride = URL.GetOption(TEXT("InitialConnectTimeout="), nullptr))
	{
		float ParsedValue;
		LexFromString(ParsedValue, InitialConnectTimeoutOverride);
		if (ParsedValue != 0.0f)
		{
			InitialConnectTimeout = ParsedValue;
		}
	}
	if (const TCHAR* ConnectionTimeoutOverride = URL.GetOption(TEXT("ConnectionTimeout="), nullptr))
	{
		float ParsedValue;
		LexFromString(ParsedValue, ConnectionTimeoutOverride);
		if (ParsedValue != 0.0f)
		{
			ConnectionTimeout = ParsedValue;
		}
	}
	if (URL.HasOption(TEXT("NoTimeouts")))
	{
		bNoTimeouts = true;
	}

	LastTickDispatchRealtime = FPlatformTime::Seconds();
	bool bSuccess = InitConnectionClass();

	if (!bInitAsClient)
	{
		ConnectionlessHandler.Reset();
		
		if (!IsUsingIrisReplication())
		{
			InitReplicationDriverClass();
			SetReplicationDriver(UReplicationDriver::CreateReplicationDriver(this, URL, GetWorld()));
		}

		DDoS.Init(FMath::Clamp(GetNetServerMaxTickRate(), 1, 1000));

		DDoS.NotifySeverityEscalation.BindLambda(
			[this](FString SeverityCategory)
		{
			GEngine->BroadcastNetworkDDosSEscalation(this->GetWorld(), this, SeverityCategory);
		});
	}

#if DO_ENABLE_NET_TEST
	bool bSettingFound(false);
	FPacketSimulationSettings PacketSettings;

	for (const FString& URLOption : URL.Op)
	{
		bSettingFound |= PacketSettings.ParseSettings(*URLOption);
	}

	if( bSettingFound )
	{
		SetPacketSimulationSettings(PacketSettings);
	}
#endif //#if DO_ENABLE_NET_TEST

	Notify = InNotify;

	// At the time being using NetTokens and NetTokenStore is only available when compiling with iris.
	// Init NetToken stores
	if (!GetNetTokenStore())
	{
		using namespace UE::Net;

		NetTokenStore = MakeUnique<FNetTokenStore>();

		FNetTokenStore::FInitParams NetTokenStoreInitParams;
		NetTokenStoreInitParams.Authority = !bInitAsClient ? FNetToken::ENetTokenAuthority::Authority : FNetToken::ENetTokenAuthority::None;
		NetTokenStoreInitParams.MaxConnections = ConnectionIdHandler.GetMaxConnectionIdCount();
		NetTokenStore->Init(NetTokenStoreInitParams);

		// TODO: make this configurable from config so that users can provide their own custom NetTokenStores
		// Order is important, changing order will require new netversion
		// When we move it to the config TypeId will be made explicit
		NetTokenStore->CreateAndRegisterDataStore<FStringTokenStore>();
		NetTokenStore->CreateAndRegisterDataStore<FNameTokenStore>();
		NetTokenStore->CreateAndRegisterDataStore<FGameplayTagTokenStore>();
		OnNetTokenStoreReadyDelegate.Broadcast(this);
	}
	
	if (IsUsingIrisReplication() && !ReplicationSystem)
	{
		CreateReplicationSystem(bInitAsClient);
	}

	
	if (NetDriverDefinition == NAME_GameNetDriver)
	{
		UpdateCrashContext(ECrashContextUpdate::UpdateRepModel);

#if WITH_PUSH_MODEL
		// Disable PushModel globally when the GameNetDriver is running with Iris.
		// Temp until we fully integrate Iris PushModel support.
		const bool bAllowPushModelHandles = !IsUsingIrisReplication();
		UEPushModelPrivate::SetHandleCreationAllowed(bAllowPushModelHandles);
#endif
	}

	UE_LOG(LogNet, Log, TEXT("InitBase %s (NetDriverDefinition %s) using replication model %s"), *NetDriverName.ToString(), *NetDriverDefinition.ToString(), *GetReplicationModelName());
	
	InitNetTraceId();

	// NotifyGameInstanceUpdate might have been called prior to setting up the NetTraceId so we call it again
	NotifyGameInstanceUpdated();

	if (!bInitAsClient)
	{
		InitDestroyedStartupActors();
	}

	CachedGlobalNetTravelCount = GEngine->GetGlobalNetTravelCount();

	// Add all of the metrics used by the networking system and register metrics listeners.
	SetupNetworkMetrics();

	if (ShouldRegisterMetricsDatabaseListeners())
	{
		SetupNetworkMetricsListeners(bInitAsClient);
	}

	return bSuccess;
}
```

## State Synchronization(状态同步)

服务器持有最高级别信任度的游戏实例，客户端仅负责接收实时服务器游戏状态并渲染和接收玩家指令并于服务器通信。

### 具体过程概述

1. 根据设置好的同步频率，扫描所有被标记为Replicated的Actor，当服务器当前数据状态与上次发送给客户端的快照对比产生差异时，便将其标记并加入队列。

2. 将数据序列化成方便网络传输的数据格式，并且聚合包体减少开销。

3. 通过自定义的UDP协议将包体传输给客户端，客户端接收并解包，执行得到的指令和逻辑。

### 优点

1. 高度安全，所有逻辑都在服务端。

2. 断线重连仅需同步最新状态。

### 缺点

1. 网络宽带占用相对较高，需要包含物体的大量数据信息，如物理位置及状态和功能上的实时状态。

2. 需要良好的网络预测处理网络延迟带来的延迟表现问题。

## Lockstep / Frame Sync(帧同步)

服务器仅负责网络通信，不负责游戏逻辑运算，客户端运行相同的游戏逻辑、渲染并且通知服务器。服务器收集"第()帧玩家()做了()"的信息并且广播给所有客户端

### 优点

1. 网络宽带占用低，仅需传输玩家操作信息。

2. 更好且更加精确的反馈。

### 缺点

1. 客户端拥有运算权和一定的数据信任，带来游戏作弊隐患。

2. 断线重连必须从第一帧开始计算。

## Remote Procedure Call(远程过程调用)

适用于瞬时逻辑，不需持久存在影响的。

### Client RPC

客户端发送请求信息至服务端。

### Server RPC

服务端发送反馈信息回客户端。

### Multicast

服务端向多个涉及到事件的客户端发送广播通知。

## Property Replication(属性复制)

将数据复制给所有受影响的客户端保持数据一致性，确保所有客户端都是得到相同的表现。持久存在影响的逻辑，比如血量显示、人物状态等。属性复制仅能由服务端发起并进行。

### GetLifetimeReplicatedProps

用来配置属性复制的规则，精细的控制每个属性复制的规则以控制网络开销。

开启属性复制的Actor生成后会被调用一次，构建属性复制数据结构(FRepLayout)。

在每一个网络同步周期内，都会被调用一次检查同步状况（是否符合条件、是否有改变）。

#### FRepLayout

* Engine\Source\Runtime\Engine\Public\Net\RepLayout.h
```Cpp
/**
 * This class holds all replicated properties for a given type (either a UClass, UStruct, or UFunction).
 * Helpers functions exist to read, write, and compare property state.
 *
 * There is only one FRepLayout for a given type, meaning all instances of the type share the FRepState.
 *
 * COMMANDS:
 *
 * All Properties in a RepLayout are represented as Layout Commands.
 * These commands dictate:
 *		- What the underlying data type is.
 *		- How the data is laid out in memory.
 *		- How the data should be serialized.
 *		- How the data should be compared (between instances of Objects, Structs, etc.).
 *		- Whether or not the data should trigger notifications on change (RepNotifies).
 *		- Whether or not the data is conditional (e.g. may be skipped when sending to some or all connections).
 *
 * Commands are split into 2 main types: Parent Commands (@see FRepParentCmd) and Child Commands (@see FRepLayoutCmd).
 *
 * A Parent Command represents a Top Level Property of the type represented by an FRepLayout.
 * A Child Command represents any Property (even nested properties).
 *
 * E.G.,
 *		Imagine an Object O, with 4 Properties, CA, DA, I, and S.
 *			CA is a fixed size C-Style array. This will generate 1 Parent Command and 1 Child Command for *each* element in the array.
 *			DA is a dynamic array (TArray). This will generate only 1 Parent Command and 1 Child Command, both referencing the array.
 *				Additionally, Child Commands will be added recursively for the element type of the array.
 *			S is a UStruct.
 *				All struct types generate 1 Parent Command for the Struct Property. Additionally:
 *					If the struct has a native NetSerialize method then it will generate 1 Child Command referencing the struct.
 *					If the struct has a native NetDeltaSerialize method then it will generate no Child Commands.
 *					All other structs will recursively generate Child Commands for each nested Net Property in the struct.
 *						Note, in this case there is no Child Command associated with the top level struct property.
 *			I is an integer (or other supported non-Struct type, or object reference). This will generate 1 Parent Command and 1 Child Command.
 *
 * CHANGELISTS
 *
 * Along with Layout Commands that describe the Properties in a type, RepLayout uses changelists to know
 * what Properties have changed between frames. @see FRepChangedHistory.
 *
 * Changelists are arrays of Property Handles that describe what Properties have changed, however they don't
 * track the actual values of the Properties.
 *
 * Changelists can contain "sub-changelists" for arrays. Formally, they can be described as the following grammar:
 *
 *		Terminator			::=	0
 *		Handle				::= Integer between 1 ~ 65535
 *		Number				::= Integer between 0 ~ 65535
 *		Changelist			::=	<Terminator> | <Handle><Changelist> | <Handle><Array-Changelist><Changelist>
 *		Array-Changelist:	::= <Number><Changelist>
 *
 * An important distinction is that Handles do not have a 1:1 mapping with RepLayoutCommands.
 * Handles are 1-based (as opposed to 0-based), and track a relative command index within a single
 * level of a changelist. Each Array Command, regardless of the number of child Commands it has,
 * will only be count as a single handle in its owning changelist. Each time we recurse into an Array-Changelist,
 * our handles restart at 1 for that "depth", and they correspond to the Commands associated with the Array's element type.
 *
 * In order to generate Changelists, Layout Commands are sequentially applied that compare the values
 * of an object's cached state to a object's current state. Any properties that are found to be different
 * will have their handle written into the changelist. This means handles within a changelists are
 * inherently ordered (with arrays inserted whose Handles are also ordered).
 *
 * When we want to replicate properties for an object, merge together any outstanding changelists
 * and then iterate over it using Layout Commands that serialize the necessary property data.
 *
 * Receiving is very similar, except the Handles are baked into the serialized data so no
 * explicit changelist is required. As each Handle is read, a Layout Command is applied
 * that serializes the data from the network bunch and applies it to an object.
 *
 * RETRIES AND RELIABLES
 *
 * @FSendingRepState maintains a circular buffer that tracks recently sent Changelists (@FRepChangedHistory).
 * These history items track the Changelist alongside the Packet ID that the bunches were sent in.
 * 
 * Once we receive ACKs for all associated packets, the history will be removed from the buffer.
 * If NAKs are received for any of the packets, we will merge the changelist into the next set of properties we replicate.
 *
 * If we receive no NAKs or ACKs for an extended period, to prevent overflows in the history buffer,
 * we will merge the entire buffer into a single monolithic changelist which will be sent alongside the next set of properties.
 *
 * In both cases of NAKs or no response, the merged changelists will be tracked in the latest history item
 * alongside with other sent properties.
 *
 * When "net.PartialBunchReliableThreshold" is non-zero and property data bunches are split into partial bunches above
 * the threshold, we will not generate a history item. Instead, we will rely on the reliable bunch framework for resends
 * and replication of the Object will be completely paused until the property bunches are acknowledged.
 * However, this will not affect other history items since they are still unreliable.
 */
class FRepLayout : public FGCObject, public TSharedFromThis<FRepLayout>
{
private:

	friend struct FRepStateStaticBuffer;
	friend class UPackageMapClient;
	friend class FNetSerializeCB;
	friend struct FCustomDeltaPropertyIterator;

	FRepLayout();

public:

	virtual ~FRepLayout();

	/** Creates a new FRepLayout for the given class. */
	ENGINE_API static TSharedPtr<FRepLayout> CreateFromClass(UClass* InObjectClass, const UNetConnection* ServerConnection = nullptr, const ECreateRepLayoutFlags Flags = ECreateRepLayoutFlags::None);

	/** Creates a new FRepLayout for the given struct. */
	ENGINE_API static TSharedPtr<FRepLayout> CreateFromStruct(UStruct * InStruct, const UNetConnection* ServerConnection = nullptr, const ECreateRepLayoutFlags Flags = ECreateRepLayoutFlags::None);

	/** Creates a new FRepLayout for the given function. */
	static TSharedPtr<FRepLayout> CreateFromFunction(UFunction* InFunction, const UNetConnection* ServerConnection = nullptr, const ECreateRepLayoutFlags Flags = ECreateRepLayoutFlags::None);

	/**
	 * Creates and initialize a new Shadow Buffer.
	 *
	 * Shadow Data / Shadow States are used to cache property data so that the Object's state can be
	 * compared between frames to see if any properties have changed. They are also used on clients
	 * to keep track of RepNotify state.
	 *
	 * This includes:
	 *		- Allocating memory for all Properties in the class.
	 *		- Constructing instances of each Property.
	 *		- Copying the values of the Properties from given object.
	 *
	 * @param Source	Memory buffer storing object property data.
	 */
	FRepStateStaticBuffer CreateShadowBuffer(const FConstRepObjectDataBuffer Source) const;

	/**
	 * Creates and initializes a new FReplicationChangelistMgr.
	 *
	 * @param InObject		The Object that is being managed.
	 * @param CreateFlags	Flags modifying how the manager is created.
	 */
	TSharedPtr<FReplicationChangelistMgr> CreateReplicationChangelistMgr(const UObject* InObject, const ECreateReplicationChangelistMgrFlags CreateFlags) const;

	/**
	 * Creates and initializes a new FRepState.
	 *
	 * This includes:
	 *		- Initializing the ShadowData.
	 *		- Associating and validating the appropriate ChangedPropertyTracker.
	 *		- Building initial ConditionMap.
	 *
	 * @param RepState						The RepState to initialize.
	 * @param Class							The class of the object represented by the input memory.
	 * @param Src							Memory buffer storing object property data.
	 * @param InRepChangedPropertyTracker	The PropertyTracker we want to associate with the RepState.
	 *
	 * @return A new RepState.
	 *			Note, maybe a a FRepStateBase or FRepStateSending based on parameters.
	 */
	TUniquePtr<FRepState> CreateRepState(
		const FConstRepObjectDataBuffer Source,
		TSharedPtr<FRepChangedPropertyTracker>& InRepChangedPropertyTracker,
		ECreateRepStateFlags Flags) const;

	UE_DEPRECATED(5.1, "No longer used, trackers are initialized by the replication subsystem.")
	void InitChangedTracker(FRepChangedPropertyTracker * ChangedTracker) const;

	/**
	 * Writes out any changed properties for an Object into the given data buffer,
	 * and does book keeping for the RepState of the object.
	 *
	 * Note, this does not compare properties or send them on the wire, it's only used
	 * to serialize properties.
	 *
	 * @param RepState				RepState for the object.
	 *								This is expected to be valid.
	 * @param RepChangelistState	RepChangelistState for the object.
	 * @param Data					Pointer to memory where property data is stored.
	 * @param ObjectClass			Class of the object.
	 * @param Writer				Writer used to store / write out the replicated properties.
	 * @param RepFlags				Flags used for replication.
	 */
	bool ReplicateProperties(
		FSendingRepState* RESTRICT RepState,
		FRepChangelistState* RESTRICT RepChangelistState,
		const FConstRepObjectDataBuffer Data,
		UClass* ObjectClass,
		UActorChannel* OwningChannel,
		FNetBitWriter& Writer,
		const FReplicationFlags& RepFlags) const;

	/**
	 * Reads all property values from the received buffer, and applies them to the
	 * property memory.
	 *
	 * @param OwningChannel			The channel of the Actor that owns the object whose properties we're reading.
	 * @param InObjectClass			Class of the object.
	 * @param RepState				RepState for the object.
	 *								This is expected to be valid.
	 * @param Data					Pointer to memory where read property data should be stored.
	 * @param InBunch				The data that should be read.
	 * @param bOutHasUnmapped		Whether or not unmapped GUIDs were read.
	 * @param bOutGuidsChanged		Whether or not any GUIDs were changed.
	 * @param Flags					Controls how ReceiveProperties behaves.
	 */
	bool ReceiveProperties(
		UActorChannel* OwningChannel,
		UClass* InObjectClass,
		FReceivingRepState* RESTRICT RepState,
		UObject* Object,
		FNetBitReader& InBunch,
		bool& bOutHasUnmapped,
		bool& bOutGuidsChanged,
		const EReceivePropertiesFlags InFlags) const;

	/**
	 * Finds any properties in the Shadow Buffer of the given Rep State that are currently valid
	 * (mapped or unmapped) references to other network objects, and retrieves the associated
	 * Net GUIDS.
	 *
	 * @param RepState					The RepState whose shadow buffer we'll inspect.
	 *									This is expected to be valid.
	 * @param OutReferencedGuids		Set of Net GUIDs being referenced by the RepState.
	 * @param OutTrackedGuidMemoryBytes	Total memory usage of properties containing GUID references. 
	 */
	void GatherGuidReferences(
		FReceivingRepState* RESTRICT RepState,
		struct FNetDeltaSerializeInfo& Params,
		TSet<FNetworkGUID>& OutReferencedGuids,
		int32& OutTrackedGuidMemoryBytes) const;

	/**
	 * Called to indicate that the object referenced by the FNetworkGUID is no longer mapped.
	 * This can happen if the Object was destroyed, if its level was streamed out, or for any
	 * other reason that may cause the Server (or client) to no longer be able to properly
	 * reference the object. Note, it's possible the object may become valid again later.
	 *
	 * @param RepState	The RepState that holds a reference to the object.
	 *					This is expected to be valid.
	 * @param GUID		The Network GUID of the object to unmap.
	 * @param Params	Delta Serialization Params used for Custom Delta Properties.
	 */
	bool MoveMappedObjectToUnmapped(
		FReceivingRepState* RESTRICT RepState,
		struct FNetDeltaSerializeInfo& Params,
		const FNetworkGUID& GUID) const;

	/**
	 * Attempts to update any unmapped network guids referenced by the RepState.
	 * If any guids become mapped, we will update corresponding properties on the given
	 * object to point to the referenced object.
	 *
	 * @param RepState					The RepState associated with the Object.
	 *									This is expected to be valid.
	 * @param PackageMap				The package map that controls FNetworkGUID associations.
	 * @param Object					The live game object whose properties should be updated if we map any objects.
	 * @param bOutSomeObjectsWereMapped	Whether or not we successfully mapped any references.
	 * @param bOutHasMoreUnamapped		Whether or not there are more unmapped references in the RepState.
	 */
	void UpdateUnmappedObjects(
		FReceivingRepState* RESTRICT RepState,
		UPackageMap* PackageMap,
		UObject* Object,
		struct FNetDeltaSerializeInfo& Params,
		bool& bCalledPreNetReceive,
		bool& bOutSomeObjectsWereMapped,
		bool& bOutHasMoreUnmapped) const;

	/**
	 * Fire any RepNotifies that have been queued for an object while receiving properties.
	 *
	 * @param RepState	The ReceivingRepState associated with the Object.
	 *					This is expected to be valid.
	 * @param Object	The Object that received properties.
	 */
	void CallRepNotifies(FReceivingRepState* RepState, UObject* Object) const;

	template<ERepDataBufferType DataType>
	void ValidateWithChecksum(TConstRepDataBuffer<DataType> Data, FBitArchive & Ar) const;

	uint32 GenerateChecksum(const FRepState* RepState) const;

	/**
	 * Compare all properties between source and destination buffer, and optionally update the destination
	 * buffer to match the state of the source buffer if they don't match.
	 *
	 * @param RepNotifies	RepNotifies that should be fired if we're changing properties.
	 * @param Destination	Destination buffer that will be changed if we're changing properties.
	 * @param Source		Source buffer containing desired property values.
	 * @param Flags			Controls how DiffProperties behaves.
	 *
	 * @return True if there were any properties with different values.
	 */
	template<ERepDataBufferType DstType, ERepDataBufferType SrcType>
	bool DiffProperties(
		TArray<FProperty*>* RepNotifies,
		TRepDataBuffer<DstType> Destination,
		TConstRepDataBuffer<SrcType> Source,
		const EDiffPropertiesFlags Flags) const;

	/**
	 * @see DiffProperties
	 *
	 * The main difference between this method and DiffProperties is that this method will skip
	 * any properties that are:
	 *
	 *	- Transient
	 *	- Point to Actors or ActorComponents
	 *	- Point to Objects that are non-stably named for networking.
	 *
	 * @param RepNotifies	RepNotifies that should be fired if we're changing properties.
	 * @param Destination	Destination buffer that will be changed if we're changing properties.
	 * @param Source		Source buffer containing desired property values.
	 * @param Flags			Controls how DiffProperties behaves.
	 *
	 * @return True if there were any properties with different values.
	 */
	template<ERepDataBufferType DstType, ERepDataBufferType SrcType>
	bool DiffStableProperties(
		TArray<FProperty*>* RepNotifies,
		TArray<UObject*>* ObjReferences,
		TRepDataBuffer<DstType> Destination,
		TConstRepDataBuffer<SrcType> Source) const;

	/** @see SendProperties. */
	void ENGINE_API SendPropertiesForRPC(
		UFunction* Function,
		UActorChannel* Channel,
		FNetBitWriter& Writer,
		const FConstRepObjectDataBuffer Data) const;

	/** @see ReceiveProperties. */
	void ReceivePropertiesForRPC(
		UObject* Object,
		UFunction* Function,
		UActorChannel* Channel,
		FNetBitReader& Reader,
		FRepObjectDataBuffer Data,
		TSet<FNetworkGUID>& UnmappedGuids) const;

	/** Builds shared serialization state for a multicast rpc */
	void ENGINE_API BuildSharedSerializationForRPC(const FConstRepObjectDataBuffer Data, UE::Net::FNetTokenStore* NetTokenStore = nullptr);

	/** Clears shared serialization state for a multicast rpc */
	void ENGINE_API ClearSharedSerializationForRPC();

	// Struct support
	ENGINE_API void SerializePropertiesForStruct(
		UStruct* Struct,
		FBitArchive& Ar,
		UPackageMap* Map,
		FRepObjectDataBuffer Data,
		bool& bHasUnmapped,
		const UObject* OwningObject = nullptr) const;

	/** Serializes all replicated properties of a UObject in or out of an archive (depending on what type of archive it is). */
	ENGINE_API void SerializeObjectReplicatedProperties(UObject* Object, FBitArchive & Ar) const;

	UObject* GetOwner() const { return Owner; }

	/** Currently only used for Replays / with the UDemoNetDriver. */
	void SendProperties_BackwardsCompatible(
		FSendingRepState* RESTRICT RepState,
		FRepChangedPropertyTracker* ChangedTracker,
		const FConstRepObjectDataBuffer Data,
		UNetConnection* Connection,
		FNetBitWriter& Writer,
		TArray<uint16>& Changed) const;

	/** Currently only used for Replays / with the UDemoNetDriver. */
	bool ReceiveProperties_BackwardsCompatible(
		UNetConnection* Connection,
		FReceivingRepState* RESTRICT RepState,
		FRepObjectDataBuffer Data,
		FNetBitReader& InBunch,
		bool& bOutHasUnmapped,
		const bool bEnableRepNotifies,
		bool& bOutGuidsChanged,
		UObject* OwningObject = nullptr) const;

	//~ Begin FGCObject Interface
	ENGINE_API virtual void AddReferencedObjects(FReferenceCollector& Collector) override;
	ENGINE_API virtual FString GetReferencerName() const override;
	//~ End FGCObject Interface

	/**
	 * Gets a pointer to the value of the given property in the Shadow State.
	 *
	 * @return A pointer to the property value in the shadow state, or nullptr if the property wasn't found.
	 */
	template<typename T>
	T* GetShadowStateValue(FRepShadowDataBuffer Data, const FName PropertyName)
	{
		for (const FRepParentCmd& Parent : Parents)
		{
			if (Parent.CachedPropertyName == PropertyName)
			{
				return (T*)((Data + Parent).Data);
			}
		}

		return nullptr;
	}

	template<typename T>
	const T* GetShadowStateValue(FConstRepShadowDataBuffer Data, const FName PropertyName) const
	{
		for (const FRepParentCmd& Parent : Parents)
		{
			if (Parent.CachedPropertyName == PropertyName)
			{
				return (const T*)((Data + Parent).Data);
			}
		}

		return nullptr;
	}

	const ERepLayoutFlags GetFlags() const
	{
		return Flags;
	}

	const bool IsEmpty() const
	{
		return EnumHasAnyFlags(Flags, ERepLayoutFlags::NoReplicatedProperties) || (0 == Parents.Num());
	}

	const int32 GetNumParents() const
	{
		return Parents.Num();
	}

	const FProperty* GetParentProperty(int32 Index) const
	{ 
		return Parents.IsValidIndex(Index) ? Parents[Index].Property : nullptr;
	}

	const int32 GetParentArrayIndex(int32 Index) const
	{
		return Parents.IsValidIndex(Index) ? Parents[Index].ArrayIndex : 0;
	}

	const int32 GetParentCondition(int32 Index) const
	{
		return Parents.IsValidIndex(Index) ? Parents[Index].Condition : COND_None;
	}

	const bool IsCustomDeltaProperty(int32 Index) const
	{
		return Parents.IsValidIndex(Index) ? EnumHasAnyFlags(Parents[Index].Flags, ERepParentFlags::IsCustomDelta) : false;
	}

#if WITH_PUSH_MODEL
	const bool IsPushModelProperty(int32 Index) const
	{
		return PushModelProperties.IsValidIndex(Index) ? PushModelProperties[Index] : false;
	}
#endif

	const uint16 GetCustomDeltaIndexFromPropertyRepIndex(const uint16 PropertyRepIndex) const;

	void CountBytes(FArchive& Ar) const;

private:

	void InitFromClass(UClass* InObjectClass, const UNetConnection* ServerConnection, const ECreateRepLayoutFlags Flags);

	void InitFromStruct(UStruct* InStruct, const UNetConnection* ServerConnection, const ECreateRepLayoutFlags Flags);

	void InitFromFunction(UFunction* InFunction, const UNetConnection* ServerConnection, const ECreateRepLayoutFlags Flags);

	/**
	 * Compare Property Values currently stored in the Changelist State to the Property Values
	 * in the passed in data, generating a new changelist if necessary.
	 *
	 * @param RepState				RepState for the object.
	 * @param RepChangelistState	The FRepChangelistState that contains the last cached values and changelists.
	 * @param Data					The newest Property Data available.
	 * @param RepFlags				Flags that will be used if the object is replicated.
	 * @param bForceCompare			Compare the property even if the dirty flag is not set.
	 */
	ERepLayoutResult CompareProperties(
		FSendingRepState* RESTRICT RepState,
		FRepChangelistState* RESTRICT RepChangelistState,
		const FConstRepObjectDataBuffer Data,
		const FReplicationFlags& RepFlags,
		const bool bForceCompare) const;

	/**
	 * Writes all changed property values from the input owner data to the given buffer.
	 * This is used primarily by ReplicateProperties.
	 *
	 * Note, the changelist is expected to have any conditional properties whose conditions
	 * aren't met filtered out already. See FRepState::ConditionMap and FRepLayout::FilterChangeList
	 *
	 * @param RepState			RepState for the object.
	 *							This is expected to be valid.
	 * @param ChangedTracker	Used to indicate
	 * @param Data				Pointer to the object's memory.
	 * @param ObjectClass		Class of the object.
	 * @param Writer			Writer used to store / write out the replicated properties.
	 * @param Changed			Aggregate list of property handles that need to be written.
	 * @param SharedInfo		Shared Serialization state for properties.
	 */
	void SendProperties(
		FSendingRepState* RESTRICT RepState,
		FRepChangedPropertyTracker* ChangedTracker,
		const FConstRepObjectDataBuffer Data,
		UClass* ObjectClass,
		FNetBitWriter& Writer,
		TArray<uint16>& Changed,
		const FRepSerializationSharedInfo& SharedInfo,
		const ESerializePropertyType SerializePropertyType) const;

	/**
	 * Clamps a changelist so that it conforms to the current size of either an array, or arrays within structs/arrays.
	 *
	 * @param Data				Object memory.
	 * @param Changed			The changelist to prune.
	 * @param PrunedChanged		The resulting pruned changelist.
	 */
	void PruneChangeList(
		const FConstRepObjectDataBuffer Data,
		const TArray<uint16>& Changed,
		TArray<uint16>& PrunedChanged) const;

	void MergeChangeList(
		const FConstRepObjectDataBuffer Data,
		const TArray<uint16>& Dirty1,
		const TArray<uint16>& Dirty2,
		TArray<uint16>& MergedDirty) const;

	void RebuildConditionalProperties(
		FSendingRepState* RepState,
		const FReplicationFlags RepFlags) const;

	void UpdateChangelistHistory(
		FSendingRepState* RepState,
		UClass* ObjectClass,
		const FConstRepObjectDataBuffer Data,
		UNetConnection* Connection,
		TArray<uint16>* OutMerged) const;

	void SendProperties_BackwardsCompatible_r(
		FSendingRepState* RESTRICT RepState,
		UPackageMapClient* PackageMapClient,
		FNetFieldExportGroup* NetFieldExportGroup,
		FRepChangedPropertyTracker* ChangedTracker,
		FNetBitWriter& Writer,
		const bool bDoChecksum,
		FRepHandleIterator& HandleIterator,
		const FConstRepObjectDataBuffer Source) const;

	void SendAllProperties_BackwardsCompatible_r(
		FSendingRepState* RESTRICT RepState,
		FNetBitWriter& Writer,
		const bool bDoChecksum,
		UPackageMapClient* PackageMapClient,
		FNetFieldExportGroup* NetFieldExportGroup,
		const int32 CmdStart,
		const int32 CmdEnd,
		const FConstRepObjectDataBuffer SourceData) const;

	void SendProperties_r(
		FSendingRepState* RESTRICT RepState,
		FNetBitWriter& Writer,
		const bool bDoChecksum,
		FRepHandleIterator& HandleIterator,
		const FConstRepObjectDataBuffer SourceData,
		const int32	 ArrayDepth,
		const FRepSerializationSharedInfo* const RESTRICT SharedInfo,
		const ESerializePropertyType SerializePropertyType) const;

	void BuildSharedSerialization(
		const FConstRepObjectDataBuffer Data,
		TArray<uint16>& Changed,
		const bool bWriteHandle,
		FRepSerializationSharedInfo& SharedInfo, 
		UE::Net::FNetTokenStore* NetTokenStore = nullptr) const;

	void BuildSharedSerialization_r(
		FRepHandleIterator& RepHandleIterator,
		const FConstRepObjectDataBuffer SourceData,
		const bool bWriteHandle,
		const bool bDoChecksum,
		const int32 ArrayDepth,
		FRepSerializationSharedInfo& SharedInfo) const;

	void BuildSharedSerializationForRPC_DynamicArray_r(
		const int32 CmdIndex,
		const FConstRepObjectDataBuffer Data,
		int32 ArrayDepth,
		FRepSerializationSharedInfo& SharedInfo);

	void BuildSharedSerializationForRPC_r(
		const int32 CmdStart,
		const int32 CmdEnd,
		const FConstRepObjectDataBuffer Data,
		int32 ArrayIndex,
		int32 ArrayDepth,
		FRepSerializationSharedInfo& SharedInfo);

	TSharedPtr<FNetFieldExportGroup> CreateNetfieldExportGroup() const;

	int32 FindCompatibleProperty(
		const int32 CmdStart,
		const int32 CmdEnd,
		const uint32 Checksum) const;

	bool ReceiveProperties_BackwardsCompatible_r(
		FReceivingRepState* RESTRICT RepState,
		FNetFieldExportGroup* NetFieldExportGroup,
		FNetBitReader& Reader,
		const int32 CmdStart,
		const int32 CmdEnd,
		FRepShadowDataBuffer ShadowData,
		FRepObjectDataBuffer OldData,
		FRepObjectDataBuffer Data,
		FGuidReferencesMap* GuidReferencesMap,
		bool& bOutHasUnmapped,
		bool& bOutGuidsChanged,
		UObject* OwningObject) const;

	void GatherGuidReferences_r(
		const FGuidReferencesMap* GuidReferencesMap,
		TSet<FNetworkGUID>& OutReferencedGuids,
		int32& OutTrackedGuidMemoryBytes) const;

	bool MoveMappedObjectToUnmapped_r(
		FGuidReferencesMap* GuidReferencesMap,
		const FNetworkGUID& GUID,
		const UObject* const OwningObject) const;

	void UpdateUnmappedObjects_r(
		FReceivingRepState* RESTRICT RepState, 
		FGuidReferencesMap* GuidReferencesMap,
		UObject* OriginalObject,
		UNetConnection* Connection,
		FRepShadowDataBuffer ShadowData, 
		FRepObjectDataBuffer Data, 
		const int32 MaxAbsOffset,
		bool& bCalledPreNetReceive,
		bool& bOutSomeObjectsWereMapped,
		bool& bOutHasMoreUnmapped) const;

	void SanityCheckChangeList_DynamicArray_r(
		const int32 CmdIndex, 
		const FConstRepObjectDataBuffer Data, 
		TArray<uint16>& Changed,
		int32& ChangedIndex) const;

	uint16 SanityCheckChangeList_r(
		const int32 CmdStart, 
		const int32 CmdEnd, 
		const FConstRepObjectDataBuffer Data, 
		TArray<uint16>& Changed,
		int32& ChangedIndex,
		uint16 Handle) const;

	void SanityCheckChangeList(const FConstRepObjectDataBuffer Data, TArray<uint16>& Changed) const;

	void SerializeProperties_DynamicArray_r(
		FBitArchive& Ar, 
		UPackageMap* Map,
		const int32 CmdIndex,
		FRepObjectDataBuffer Data,
		bool& bHasUnmapped,
		const int32 ArrayDepth,
		const FRepSerializationSharedInfo& SharedInfo,
		FNetTraceCollector* Collector,
		const UObject* OwningObject) const;

	void SerializeProperties_r(
		FBitArchive& Ar, 
		UPackageMap* Map,
		const int32 CmdStart, 
		const int32 CmdEnd, 
		FRepObjectDataBuffer Data,
		bool& bHasUnmapped,
		const int32 ArrayIndex,
		const int32 ArrayDepth,
		const FRepSerializationSharedInfo& SharedInfo,
		FNetTraceCollector* Collector,
		const UObject* OwningObject) const;

	void MergeChangeList_r(
		FRepHandleIterator& RepHandleIterator1,
		FRepHandleIterator& RepHandleIterator2,
		const FConstRepObjectDataBuffer SourceData,
		TArray<uint16>& OutChanged) const;

	void PruneChangeList_r(
		FRepHandleIterator& RepHandleIterator,
		const FConstRepObjectDataBuffer SourceData,
		TArray<uint16>& OutChanged) const;

	/**
	 * Splits a given Changelist into an Inactive Change List and an Active Change List.
	 * 
	 * @param Changelist				The Changelist to filter.
	 * @param InactiveParentHandles		The set of ParentCmd Indices that are not active.
	 * @param OutInactiveProperties		The properties found to be inactive.
	 * @param OutActiveProperties		The properties found to be active.
	 */
	void FilterChangeList( 
		const TArray<uint16>& Changelist,
		const TBitArray<>& InactiveParentHandles,
		TArray<uint16>& OutInactiveProperties,
		TArray<uint16>& OutActiveProperties) const;

	/** Same as FilterChangeList, but only populates an Active Change List. */
	void FilterChangeListToActive(
		const TArray<uint16>& Changelist,
		const TBitArray<>& InactiveParentHandles,
		TArray<uint16>& OutActiveProperties) const;

	void BuildChangeList_r(
		const TArray<FHandleToCmdIndex>& HandleToCmdIndex,
		const int32 CmdStart,
		const int32 CmdEnd,
		const FConstRepObjectDataBuffer Data,
		const int32 HandleOffset,
		const bool bForceArraySends,
		TArray<uint16>& Changed) const;

	void BuildHandleToCmdIndexTable_r(
		const int32 CmdStart,
		const int32 CmdEnd,
		TArray<FHandleToCmdIndex>& HandleToCmdIndex);

	ERepLayoutResult UpdateChangelistMgr(
		FSendingRepState* RESTRICT RepState,
		FReplicationChangelistMgr& InChangelistMgr,
		const UObject* InObject,
		const uint32 ReplicationFrame,
		const FReplicationFlags& RepFlags,
		const bool bForceCompare) const;

	void InitRepStateStaticBuffer(FRepStateStaticBuffer& ShadowData, const FConstRepObjectDataBuffer Source) const;
	void ConstructProperties(FRepStateStaticBuffer& ShadowData) const;
	void CopyProperties(FRepStateStaticBuffer& ShadowData, const FConstRepObjectDataBuffer Source) const;
	void DestructProperties(FRepStateStaticBuffer& RepStateStaticBuffer) const;

	/**
	 * @param CustomDeltaIndex	The index of the Custom Delta Property.
	 *							This is not the same as FProperty::RepIndex!
	 *							@see FLifetimeCustomDeltaState.
	 */
	bool SendCustomDeltaProperty(FNetDeltaSerializeInfo& Params, const uint16 CustomDeltaIndex) const;

	/**
	 * Attempts to receive the custom delta property.
	 *
	 *
	 *
	 * @param Params			Params that we'll use to receive.
	 *							The PackageMap member must be non null.
	 *							The Object member must be non null.
	 *							The Reader member must be non null.
	 * @param Property			The Property that we're trying to received.
	 *							This is expected to be a valid Replicated property that is a CustomDelta type.
	 */
	bool ReceiveCustomDeltaProperty(
		FReceivingRepState* RESTRICT ReceivingRepState,
		FNetDeltaSerializeInfo& Params,
		FStructProperty* Property) const;

	ERepLayoutResult DeltaSerializeFastArrayProperty(struct FFastArrayDeltaSerializeParams& Params, FReplicationChangelistMgr* ChangelistMgr) const;

	void GatherGuidReferencesForFastArray(struct FFastArrayDeltaSerializeParams& Params) const;

	bool MoveMappedObjectToUnmappedForFastArray(struct FFastArrayDeltaSerializeParams& Params) const;

	void UpdateUnmappedGuidsForFastArray(struct FFastArrayDeltaSerializeParams& Params) const;

	void PreSendCustomDeltaProperties(
		UObject* Object,
		UNetConnection* Connection,
		FReplicationChangelistMgr& ChangelistMgr,
		uint32 ReplicationFrame,
		TArray<TSharedPtr<INetDeltaBaseState>>& CustomDeltaStates) const;

	void PostSendCustomDeltaProperties(
		UObject* Object,
		UNetConnection* Connection,
		FReplicationChangelistMgr& ChangelistMgr,
		TArray<TSharedPtr<INetDeltaBaseState>>& CustomDeltaStates) const;

	const uint16 GetNumLifetimeCustomDeltaProperties() const;

	const uint16 GetLifetimeCustomDeltaPropertyRepIndex(const uint16 RepIndCustomDeltaPropertyIndex) const;

	FProperty* GetLifetimeCustomDeltaProperty(const uint16 CustomDeltaPropertyIndex) const;

	const ELifetimeCondition GetLifetimeCustomDeltaPropertyCondition(const uint16 RepIndCustomDeltaPropertyIndex) const;

	ERepLayoutFlags Flags;

	/** Size (in bytes) needed to allocate a single instance of a Shadow buffer for this RepLayout. */
	int32 ShadowDataBufferSize;

	/** Top level Layout Commands. */
	TArray<FRepParentCmd> Parents;

	/** All Layout Commands. */
	TArray<FRepLayoutCmd> Cmds;

	/** Converts a relative handle to the appropriate index into the Cmds array */
	TArray<FHandleToCmdIndex> BaseHandleToCmdIndex;

	/**
	 * Special state tracking for Lifetime Custom Delta Properties.
	 * Will only ever be valid if the Layout has Lifetime Custom Delta Properties.
	 */
	TUniquePtr<struct FLifetimeCustomDeltaState> LifetimeCustomPropertyState;

	/** UClass, UStruct, or UFunction that this FRepLayout represents.*/
	UStruct* Owner;

	/** Shared serialization state for a multicast rpc */
	FRepSerializationSharedInfo SharedInfoRPC;

	/** Shared comparison to default state for multicast rpc */
	TBitArray<> SharedInfoRPCParentsChanged;

#if WITH_PUSH_MODEL
	/** Properties that have push model enabled. */
	TBitArray<> PushModelProperties;
#endif

	TMap<FRepLayoutCmd*, TArray<FRepLayoutCmd>> NetSerializeLayouts;
};
```

### Replicated

无通知的属性复制。

### RepNotify

有通知的属性复制，当服务端改变变量并进行属性复制后，客户端触发回调。

## UFUNCTION specifier

1. Server

该函数仅能在服务端被调用。

2. Client

该函数仅能被拥有该对象的客户端调用。

3. Reliable

该函数RPC保证可靠性，遭遇丢包、乱序会重传。

4. Unreliable

该函数RPC不保证可靠性，如遭遇丢包、乱序不会重传。

5. NetMulticast

该函数可被客户端和服务端执行，无论对象拥有者是谁。

6. WithValidation

该函数需要多定义一个以_Validate结尾的实现函数负责数据验证，并且返回值强制为bool值，如果返回值为false，则数据验证失败，阻止逻辑进行。

## UPROPERTY specifier

1. Replicated

该变量允许属性复制，与宏DOREPLIFETIME配合使用。

2. NotReplicated

该变量不应该参与属性复制，通常用在某个具体数据结构中不应加入至网络复制的成员。

3. ReplicatedUsing

当该变量发生改变并且进行属性复制后，调用该参数所指向的回调函数。

## Macro

1. DOREPLIFETIME

将指定变量添加到该类的属性复制数据结构中，只要有变化就进行属性复制并且同步给所有受拥有该变量对象影响的客户端。

2. DOREPLIFETIME_CONDITION

满足指定条件才会进行属性复制，常见条件有：

**COND_OwnerOnly**：只复制给拥有该对象的客户端。

**COND_SkipeOnwer**：只复制给不拥有该对象的所有客户端。

**COND_SimulatedOnly**: 仅复制给模拟客户端，不发给服务器，或者自主代理。

3. DOREPLIFETIME_WITH_NOTIFY

可以显示指定复制行为，如即使该值没有改变，但服务器发了包，那么仍然触发回调。

4. DOREPLIFETIME_ACTIVE_OVERRIDE

允许动态的开关该变量进行属性复制的行为

# Gameplay GameFramework

初始化次序：Engine->GameInstance->World->Level

一些数据在GameInstance阶段还未初始化，全局相关的功能初始化可能得放在WorldContext或之后

MVC模式，在这里可以具体为Model-PlayerState, View-Character, Control-Controller。

## Actor

可放置于场景中的类，绝大多数场景物体都以此作为基础类，比如Pawn、Character等。

注：Actor本身不带变换功能，默认由组件提供。

## Component

一般用来分担Actor的功能，或者实现功能代码复用，有两大基础类UActorComponent和USceneCompoent。

USceneComponent的子类通常负责一些可见或者与视觉表现相关的功能，比如变换、提供骨骼网格体等。

UActorComponent的子类通常负责内部相关的功能，比如移动，或者生命值、背包等。

## Pawn

可接受玩家输入，默认没有移动组件和骨骼网格组件，其意义更倾向于一个受玩家控制的Actor，无其他附加功能。

## Character

Pawn的子类，默认拥有移动组件和骨骼网格组件，可接受玩家的输入控制。角色相关的控制和动画逻辑一般会集中于此。

## Controller

主要负责控制Pawn，或者Character，一般来说，一个Controller对应一个Pawn。

在线子系统中，Controller中的ControllerID就代表着玩家，ControllerID在在线子系统中会被当作NetworkID处理(PS子系统如此，其它平台未知)。

## GameMode

游戏模式，定义游戏的规则和核心玩法，例如如何获胜、玩家人数限制等。

## GameState

游戏状态，通过用来记录游戏内的全局状态，如PVP中的比分，或者游戏时间等。

支持网络复制，一般存在服务端上。

## Level

Actor和Component的容器。

有WorldSettings成员可以设置关卡域内的属性。

## World

管理所有的Level，也可以访问其中的所有Actor。有成员CurrentLevel指向当前Level。

Presistent Level指的是项目编辑器视图中已经打开的持续性关卡。

## WorldContext

负责管理多个World。

游戏程序独立运行的情况下，只有GameWorldContext。

有成员变量ThisCurrentWorld，指向当前World。

## GameInstance

全局唯一，一般用来进行一些全局唯一的功能和业务或者保存全局数据。

## Engine

最底层的基础类。

保存唯一一个GameInstance指针，在编辑器启动时，会包含两个World指针，一个是PlayWorld，一个是EditorWorld，即编辑器也是一个World。

# Class Default Object (CDO)



# Macro

## Class Specifiers



## Asserts

### Check

> The Check family is the closest to the base assert, in that members of this family halt execution when the first parameter evaluates to a false value, and do not run in shipping builds by default.

* check(<Expression>), checkSlow(<Expression>)

> Halts execution if Expression if false.

* checkf(<Expression>, <FormattedText>, ...), checkfSlow(<Expression>, <FormattedText>, ...)

> Halts execution if Expression is false and outputs FormattedText to the log.

* checkCode(<Code>)

> Executes Code within a do-while loop structure that runs once; primarily useful as a way to prepare information that another Check requires.

* checkNoEntry()

> Halts execution if the line is ever hit, similar to `check(false)`, but intended for code paths that should be unreachable.

* checkNoReentry()

> Halts execution if the line is hit more than once.

* checkNoRecursion()

> Halts execution if the line is hit more than once without leaving scope.

* unimplemented()

> Halts execution if the line is ever hit, similar to `check(false)`, but intended for virtual functions that should be overridden and not called.

### Verify

> The Verify family behaves identically to the Check family in most builds. However, Verify macros evaluate their expressions even in builds where Check macros are disabled. This means that you should use Verify macros only when the expression needs to run independently of diagnostic checks.

* verify(<Expression>), verifySlow(<Expression>)

> Halts execution if Expression is false.

* verifyf(<Expression>, <FormattedText>, ...), verifyfSlow(<Expression>, <FormattedText>, ...)

> Halts execution if Expression is false and outputs FormattedText to the log

### Ensure

> The Ensure family is similar to the Verify family, but works with non-fatal errors. This means that if an Ensure macro's expression evaluates as false, the Engine will inform the crash reporter, but will continue running.

* ensure(<Expression>)

> Notifies the crash reporter on the first time Expression is false.

* ensureMsgf(<Expression>, <FormattedText>, ...)

> Notifies the crash reporter and outputs FormattedText to the log on the first time Expression is false.

* ensureAlways(<Expression>)

> Notifies the crash reporter if Expression is false.

* ensureAlwaysMsgf(<Expression>, <FormattedText>, ...)

> Notifies the crash reporter and outputs FormattedText to the log if Expression is false.

## PURE_VIRTUAL

Unreal Engine特性，由于所有继承UObject的UClass必须能实例化，因此的一般的类不可以包含纯虚函数。一些游戏业务场景要求该类的某个虚函数必须在子类有实现，普通虚函数无法达到该要求，PURE_VIRTUAL因此诞生。

# Reference

## Official Documentation

1. [Gameplay Framework Quick Reference in Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/gameplay-framework-quick-reference-in-unreal-engine/)
2. [Asserts in Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/asserts-in-unreal-engine/)
3. [Class Specifiers | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/class-specifiers/)

## Community Wiki

1. [Unreal Property System (Reflection) | Unreal Engine Community Wiki](https://unrealcommunity.wiki/unreal-property-system-(reflection)-36d1e6)
2. [Delegates | Unreal Engine Community Wiki](https://unrealcommunity.wiki/delegates-in-ue4-raw-cpp-and-bp-exposed-xifmcmq5)
3. [Garbage Collection | Unreal Engine Community Wiki](https://unrealcommunity.wiki/garbage-collection-36d1da)

## Book

1. Game Engine Architecture [Third Edition] – Jason Gergory, 叶劲峰译

## Third Party Articles

1. [UE4学习笔记：Gameplay框架及其模块梳理（上篇） - U_N_Owen – 博客园](https://www.cnblogs.com/u-n-owen/p/16425358.html)
2. [Unreal Engine的Gameplay框架和重点 – 知乎](https://zhuanlan.zhihu.com/p/612837045)
3. [UE5 -- Control Rig与IK Rig介绍 – 知乎](https://zhuanlan.zhihu.com/p/591982020)

## Video

1. [UnrealCircle苏州UE4性能优化 | 周华 苏州谜匣数娱](https://www.bilibili.com/video/BV1FV411G72Q/?spm_id_from=333.999.0.0&vd_source=4273510487973a57f723d5c6f3c6d066)