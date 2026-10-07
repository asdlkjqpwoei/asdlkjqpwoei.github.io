---
title: CryEngine
categories: Game Engine
---
# CryEngine

## Installation

开源免费的CryEngine已经不再更新

TODO

## Render

1. Scene Traversal

遍历场景的Octree(八叉树)，利用CryEngine自有的Coverage Buffer(C-Buffer)进行软预剔除

将不同的物体分类存入不同的渲染队列

2. RenderDLL

将渲染项转换为底层图形指令。

引入了 Tiled Shading（分块着色），处理海量动态光源。

3. Shadow Maps
 
生成级联阴影贴图（CSM）。

4. Z-Prepass
 
预深度遍历，减少 Overdraw。

5. Scene G-Buffer

生成基础几何信息（深度、法线、粗糙度、金属度、反照率）。

6. Lighting Pass

执行 SSDO（环境定向遮蔽） 和 SVOGI（基于体素的全局光照）。提供无需烘焙的实时 GI。

7. Post-Processing
 
运动模糊、景深以及Lens Flare 效果。

## 具体实现(施工中)
* \Code\CryEngine\RenderDll\XRenderD3D9\D3DRendPipeline.cpp

```Cpp
// Render thread only scene rendering
void CD3D9Renderer::RT_RenderScene(CRenderView* pRenderView)
{
	FUNCTION_PROFILER_RENDERER();
	PROFILE_LABEL_SCOPE_DYNAMIC((pRenderView->IsRecursive() ? "SCENE_REC" : "SCENE"), "SCENE");

	gcpRendD3D->SetCurDownscaleFactor(gcpRendD3D->m_CurViewportScale);

	{
		PROFILE_FRAME(WaitForRenderView);
		pRenderView->SwitchUsageMode(CRenderView::eUsageModeReading);
	}

	const CTimeValue Time = iTimer->GetAsyncTime();

	// Only Billboard rendering doesn't use CRenderOutput
	if (!pRenderView->GetRenderOutput() && !pRenderView->IsBillboardGenView())
	{
		pRenderView->AssignRenderOutput(GetActiveDisplayContext()->GetRenderOutput());
		pRenderView->GetRenderOutput()->BeginRendering(pRenderView);
	}

	std::shared_ptr<CGraphicsPipeline> pActiveGraphicsPipeline = pRenderView->GetGraphicsPipeline();
	uint32 shaderRenderingFlags = pActiveGraphicsPipeline->GetRenderFlags();

	CFlashTextureSourceSharedRT::SetupSharedRenderTargetRT();

	if (!pRenderView->IsRecursive())
	{
		D3D11_VIEWPORT viewport = RenderViewportToD3D11Viewport(pRenderView->GetViewport());
		pActiveGraphicsPipeline->GetVrProjectionManager()->Configure(viewport, pRenderView->GetCurrentEye() == CCamera::eEye_Right);
	}

	int nSaveDrawNear     = CV_r_nodrawnear;
	int nSaveDrawCaustics = CV_r_watercaustics;
	if (shaderRenderingFlags & SHDF_NO_DRAWCAUSTICS)
		CV_r_watercaustics = 0;
	if (shaderRenderingFlags & SHDF_NO_DRAWNEAR)
		CV_r_nodrawnear = 1;

	m_bDeferredDecals = false;

	m_vSceneLuminanceInfo = Vec4(1.0f, 1.0f, 1.0f, 1.0f);
	m_fAdaptedSceneScale  = m_fAdaptedSceneScaleLBuffer = m_fScotopicSceneScale = 1.0f;

	// This scope is the only one allowed to utilize the graphics pipeline
	{
		CRY_ASSERT(shaderRenderingFlags & SHDF_ALLOWHDR);

		{
			PROFILE_FRAME(WaitForParticleRendItems);
			SyncComputeVerticesJobs();
			UnLockParticleVideoMemory(GetRenderFrameID());
		}

		pActiveGraphicsPipeline->Update(EShaderRenderingFlags(shaderRenderingFlags));

		// Creating CompiledRenderObjects should happen after Update() call of the GraphicsPipeline, as it requires access to initialized Render Targets
		// If some pipeline stage manages/retires resources used in compiled objects, they should also be handled in Update()
		pRenderView->CompileModifiedRenderObjects();

		// Sort transparent lists that might have refractive items that will require resolve passes.
		// This is done after the CompileModifiedRenderObjects we need to project render items AABB.
		pRenderView->StartOptimizeTransparentRenderItemsResolvesJob();

		pActiveGraphicsPipeline->Execute();

		//////////////////////////////////////////////////////////////////////////
		// Normally it does this:
		//  [HDR,  renderResolution] CRenderingResources::s_ptexHDRTarget ->
		//  [HDR,  outputResolution] CRenderOutput::m_pColorTarget ->
		//  [HDR, displayResolution] CRenderingDisplayContext::m_pColorTarget ->
		//  [LDR, displayResolution] BackBuffer
		//
		// Depending on the setting any of the targets can substitute an adjacent buffer, say:
		//  [HDR,  renderResolution] BackBuffer
		//
		// This would happen if the back-buffer is HDR and matches the rendering resolution
		ResolveSupersampledRendering(pActiveGraphicsPipeline);
		ResolveSubsampledOutput(pActiveGraphicsPipeline);
		ResolveHighDynamicRangeDisplay(pActiveGraphicsPipeline);
		// Everything after this location will render directly into the display buffer/resolution
		//////////////////////////////////////////////////////////////////////////

		pActiveGraphicsPipeline->ClearState();
	}

	////////////////////////////////////////////////

	CV_r_nodrawnear            = nSaveDrawNear;
	CV_r_watercaustics         = nSaveDrawCaustics;

	{
		PROFILE_FRAME(RenderViewEndFrame);
		pRenderView->SwitchUsageMode(CRenderView::eUsageModeReadingDone);
	}

	SRenderStatistics::Write().m_fRenderTime += iTimer->GetAsyncTime().GetDifferenceInSeconds(Time);

	if (CRendererCVars::CV_r_FlushToGPU >= 1)
		GetDeviceObjectFactory().FlushToGPU();

	m_maskRenderPhaseLog[m_SceneRecurseCount - 1] |= eRP_RenderScene;
}
```

* \Code\CryEngine\RenderDll\Common\RenderOutput.cpp

```Cpp
void CRenderOutput::BeginRendering(CRenderView* pRenderView, stl::optional<uint32> overrideClearFlags)
{
	m_bHDRRendering = pRenderView && pRenderView->GetGraphicsPipeline() && pRenderView->GetGraphicsPipeline()->AllowsHDRRendering();

	CRY_ASSERT(gcpRendD3D->m_pRT->IsRenderThread());

	//////////////////////////////////////////////////////////////////////////
	// Set HDR Render Target (Back Color Buffer)
	//////////////////////////////////////////////////////////////////////////

	if (m_pDisplayContext)
	{
		CRY_ASSERT(m_OutputWidth  == CRendererCVars::GetCustomResWidth (m_pDisplayContext->IsScalable(), 0, m_pDisplayContext->GetDisplayResolution()[0]));
		CRY_ASSERT(m_OutputHeight == CRendererCVars::GetCustomResHeight(m_pDisplayContext->IsScalable(), 0, m_pDisplayContext->GetDisplayResolution()[1]));

		if (pRenderView)
		{
			// This scope is the only one allowed to produce HDR data, all the rest is LDR
			m_pDisplayContext->BeginRendering();
// 			m_pDisplayContext->SetLastCamera(CCamera::eEye_Left, pRenderView->GetCamera(CCamera::eEye_Left));
// 			m_pDisplayContext->SetLastCamera(CCamera::eEye_Right, pRenderView->GetCamera(CCamera::eEye_Right));
		}
	}
	else if (m_pDynTexture)
	{
		m_pDynTexture->Update(m_OutputWidth, m_OutputHeight);
		m_pColorTarget = m_pDynTexture->m_pTexture;
	}

	//////////////////////////////////////////////////////////////////////////
	// Set Depth Z Render Target
	//////////////////////////////////////////////////////////////////////////
	if (m_bUseTempDepthBuffer)
	{
		m_pDepthTarget = nullptr;
		m_pDepthTarget.Assign_NoAddRef(CRendererResources::CreateDepthTarget(m_OutputWidth, m_OutputHeight, Clr_Empty, eTF_Unknown));
	}

	//////////////////////////////////////////////////////////////////////////
	// Clear render targets on demand.
	//////////////////////////////////////////////////////////////////////////
	uint32 clearTargetFlag = overrideClearFlags.value_or(m_clearTargetFlag);
	ColorF clearColor = m_clearColor;

	if (pRenderView && pRenderView->IsClearTarget() && CRendererCVars::CV_r_wireframe)
	{
		// Override clear color from the Render View if given
		clearTargetFlag |= FRT_CLEAR_COLOR;
		clearColor = pRenderView->GetTargetClearColor();
	}

	if ((clearTargetFlag = clearTargetFlag & ~m_hasBeenCleared))
	{
		if (clearTargetFlag & (FRT_CLEAR_DEPTH | FRT_CLEAR_STENCIL))
			CClearSurfacePass::Execute(m_pDepthTarget,
			                           (clearTargetFlag & FRT_CLEAR_DEPTH ? CLEAR_ZBUFFER : 0) |
			                           (clearTargetFlag & FRT_CLEAR_STENCIL ? CLEAR_STENCIL : 0),
			                           Clr_FarPlane_Rev.r,
			                           Val_Stencil);

		if (clearTargetFlag & FRT_CLEAR_COLOR)
			CClearSurfacePass::Execute(m_pColorTarget, clearColor);

		m_hasBeenCleared |= clearTargetFlag;
	}

	//////////////////////////////////////////////////////////////////////////
	// Assign resources to RenderView
	//////////////////////////////////////////////////////////////////////////
	if (pRenderView)
	{
		// NOTE: for debugging to revert unsetting the target-pointers
		pRenderView->InspectRenderOutput();
	}

	//////////////////////////////////////////////////////////////////////////
	// TODO: make color and/or depth|stencil optional (currently it's enforced to have all of them)
	CRY_ASSERT(m_pColorTarget);
	CRY_ASSERT(m_pDepthTarget);
}
```