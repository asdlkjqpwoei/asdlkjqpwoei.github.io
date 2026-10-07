---
title: Lyra Sample
categories: Unreal Engine
---
# Version

使用Unreal Engine 5.2的Lyra

# Game Features

模块化玩法功能的框架，使玩法和功能成为一个个名为Game Feature插件，降低Pawn、Character等“热点”类的臃肿，降低维护的难度。

极度依赖Asset Manager。

## Key Class

* UGameFeaturesSubsystemSettings

> Settings for the Game Features framework.

继承UDeveloperSettings，Game Features框架的设置，可以在编辑器中或者配置文件中进行修改和配置。

* UGameFeaturesSubsystem

> The manager subsystem for game features.

继承UEngineSubsystem，负责初始化、加载或者卸载Game Feature的框架管理类。

* UGameFeatureData

> Data related to a game feature, a collection of code and content that adds a separable discrete feature to the game.

继承UPrimaryDataAsset，Game Feature数据，代码和内容的集合，可独立添加到游戏中的功能数据组件。

* UGameFeatureStateMachine

> A state machine to manage transitioning a game feature plugin from just a URL into a fully loaded and active plugin, including registering its contents with other game systems.

继承UObject，管理Game Feature从仅有URL已知到完全加载和激活的过程，包括将它的内容注册进其他的游戏系统。

* UGameFeatureAction

> Represents an action to be taken when a game feature is activated.

继承UObject，代表一个Game Feature被激活时要执行的动作。

* UGameFeatureAction_WorldActionBase

> Base class for GameFeatureActions that wish to do something world specific.

继承UGameFeatureAction，Lyra中的GameFeatureAcitons基类，在Game Feature流程中触发的激活回调都会调用此类定义的回调函数。

## Flow Diagram

{% asset_img "Unreal Engine 5 Lyra Starter Game Game Feature Work Flow.png" Game Feature Work Flow %}

## State Machine

在*Engine\Plugins\Experimental\GameFeatures\Source\GameFeatures\Public\GameFeatureTypes.h*文件中查看所有状态的描述和定义。

在*Engine\Plugins\Experimental\GameFeatures\Source\GameFeatures\Private\GameFeaturePluginStateMachine.cpp*中有所有状态执行的动作逻辑（每个结构体都几乎会有个覆写的UpdateState函数）

1. Uninitialized: 初始状态，未设置、还未被设置、未初始化。
2. UnknownStatus: 初始化后的状态，仅仅获取到该Game Feature的URL，其他信息均未知。（Game Feature信息有名称、是否确认启用、URL等，主要信息就是该Game Feature的URL。因为不知道URL到底是一个链接还是路径，所以未知）
3. CheckingStatus: 检查中的状态，一种从UnknownStatus到StatusKnown的过渡状态。
4. StatusKnown: 已经获取足够的Plugins信息后的状态，依据该信息进入Downloading或者Installed状态。(已经清楚该Game Feature的URL是链接或路径)
5. Downloading: 如果Game Feature的URL是一个链接，就会进入该状态尝试下载。
6. Installed: 处于该状态时，该Game Feature已经存在于硬盘。
7. Mounting: 一种从Installed到Regsitered的过渡状态之一，会把该Game Feature反序列化并进行预加载。（好像也是加载该Game Feature C++模块的状态）
8. WaitingForDependencies: 一种从Installed到Regsitered的过渡状态之一，如果该Game Feature有其他的依赖，并且未被加载，就会进入该状态。
9. Registering: 一种从Installed到Regsitered的过渡状态之一，读取设置并注册该Game Feature所有的UE资产。
10. Regsitered: 该Game Feature设置和资产已经被注册后所处的状态，可进入Loading状态。
11. Loading: 该状态就代表正式加载Game Feature的资产进入内存。
12. Loaded: Game Feature的所有相关数据和资产已经全部加载进入内存中，但没有与游戏系统一起注册，就会进入此状态，等待被激活。
13. Activating: 与游戏系统一起注册，正式进入激活。
14. Active: 全面激活，已经在游戏中产生影响和效果。

## Key Implementation

* Engine\Plugins\Experimental\GameFeatures\Source\GameFeatures\Private\GameFeaturesSubsystem.cpp
```Cpp
void UGameFeaturesSubsystem::LoadBuiltInGameFeaturePlugin(const TSharedRef<IPlugin>& Plugin, FBuiltInPluginAdditionalFilters AdditionalFilter)
{
	UE_SCOPED_ENGINE_ACTIVITY(TEXT("Loading GameFeaturePlugin %s"), *Plugin->GetName());

	UAssetManager::Get().PushBulkScanning();

	const FString& PluginDescriptorFilename = Plugin->GetDescriptorFileName();

	// Make sure you are in a game feature plugins folder. All GameFeaturePlugins are rooted in a GameFeatures folder.
	if (!PluginDescriptorFilename.IsEmpty() && GetDefault<UGameFeaturesSubsystemSettings>()->IsValidGameFeaturePlugin(FPaths::ConvertRelativePathToFull(PluginDescriptorFilename)) && FPaths::FileExists(PluginDescriptorFilename))
	{
		FString PluginURL;
		bool bIsFileProtocol = true;
		if (GetPluginURLByName(Plugin->GetName(), PluginURL))
		{
			bIsFileProtocol = UGameFeaturesSubsystem::IsPluginURLProtocol(PluginURL, EGameFeaturePluginProtocol::File);
		}
		else
		{
			PluginURL = GetPluginURL_FileProtocol(PluginDescriptorFilename);
		}

		// 检查该Game Feature是否存在及可否经过Game Feature加载策略的筛选
		if (bIsFileProtocol && GameSpecificPolicies->IsPluginAllowed(PluginURL))
		{
			FGameFeaturePluginDetails PluginDetails;
			if (GetGameFeaturePluginDetails(PluginDescriptorFilename, PluginDetails))
			{
				FBuiltInGameFeaturePluginBehaviorOptions BehaviorOptions;
				bool bShouldProcess = AdditionalFilter(PluginDescriptorFilename, PluginDetails, BehaviorOptions);
				// 给Game Feature建立状态机
				if (bShouldProcess)
				{
					UGameFeaturePluginStateMachine* StateMachine = FindOrCreateGameFeaturePluginStateMachine(PluginURL);

					const EBuiltInAutoState InitialAutoState = (BehaviorOptions.AutoStateOverride != EBuiltInAutoState::Invalid) ? BehaviorOptions.AutoStateOverride : PluginDetails.BuiltInAutoState;
						
					const EGameFeaturePluginState DestinationState = ConvertInitialFeatureStateToTargetState(InitialAutoState);

					// If we're already at the destination or beyond, don't transition back
					FGameFeaturePluginStateRange Destination(DestinationState, EGameFeaturePluginState::Active);
					// 加载Game Feature插件，初始化状态机。
					ChangeGameFeatureDestination(StateMachine, Destination, 
						FGameFeaturePluginChangeStateComplete::CreateUObject(this, &ThisClass::LoadBuiltInGameFeaturePluginComplete, StateMachine, Destination));
				}
			}
		}
	}

	UAssetManager::Get().PopBulkScanning();
}

void UGameFeaturesSubsystem::ChangeGameFeatureDestination(UGameFeaturePluginStateMachine* Machine, const FGameFeaturePluginStateRange& StateRange, FGameFeaturePluginChangeStateComplete CompleteDelegate)
{
	Machine->SetDestination(StateRange, FGameFeatureStateTransitionComplete::CreateUObject(this, &ThisClass::ChangeGameFeatureTargetStateComplete, CompleteDelegate));
	// ......
}

// 当Game Feature达到Registering状态时，Game Feature Action的Registering回调会被调用。
void UGameFeaturesSubsystem::OnGameFeatureRegistering(const UGameFeatureData* GameFeatureData, const FString& PluginName, const FString& PluginURL)
{
	CallbackObservers(EObserverCallback::Registering, PluginURL, &PluginName, GameFeatureData);

	for (UGameFeatureAction* Action : GameFeatureData->GetActions())
	{
		if (Action != nullptr)
		{
			Action->OnGameFeatureRegistering();
		}
	}
}

// 当Game Feature达到Activating状态时，Game Feature Action的Activating回调会被调用。
void UGameFeaturesSubsystem::OnGameFeatureActivating(const UGameFeatureData* GameFeatureData, const FString& PluginName, FGameFeatureActivatingContext& Context, const FString& PluginURL)
{
	CallbackObservers(EObserverCallback::Activating, PluginURL, &PluginName, GameFeatureData);

	for (UGameFeatureAction* Action : GameFeatureData->GetActions())
	{
		if (Action != nullptr)
		{
			Action->OnGameFeatureActivating(Context);
		}
	}
}
```

* Engine\Plugins\Experimental\GameFeatures\Private\GameFeaturePluginStateMachine.cpp
```Cpp
bool UGameFeaturePluginStateMachine::SetDestination(FGameFeaturePluginStateRange InDestination, FGameFeatureStateTransitionComplete OnFeatureStateTransitionComplete, FDelegateHandle* OutCallbackHandle /*= nullptr*/)
{
	check(IsValidDestinationState(InDestination.MinState));
	check(IsValidDestinationState(InDestination.MaxState));

	if (!InDestination.IsValid())
	{
		// Invalid range
		return false;
	}

	if (CurrentStateInfo.State == EGameFeaturePluginState::Terminal && !InDestination.Contains(EGameFeaturePluginState::Terminal))
	{
		// Can't tranistion away from terminal state
		return false;
	}

	if (!IsRunning())
	{
		// Not running so any new range is acceptable

		if (OutCallbackHandle)
		{
			OutCallbackHandle->Reset();
		}

		FDestinationGameFeaturePluginState* CurrState = AllStates[CurrentStateInfo.State]->AsDestinationState();

		if (InDestination.Contains(CurrentStateInfo.State))
		{
			OnFeatureStateTransitionComplete.ExecuteIfBound(this, MakeValue());
			return true;
		}
		
		if (CurrentStateInfo.State < InDestination)
		{
			FDestinationGameFeaturePluginState* MinDestState = AllStates[InDestination.MinState]->AsDestinationState();
			FDelegateHandle CallbackHandle = MinDestState->OnDestinationStateReached.Add(MoveTemp(OnFeatureStateTransitionComplete));
			if (OutCallbackHandle)
			{
				*OutCallbackHandle = CallbackHandle;
			}
		}
		else if (CurrentStateInfo.State > InDestination)
		{
			FDestinationGameFeaturePluginState* MaxDestState = AllStates[InDestination.MaxState]->AsDestinationState();
			FDelegateHandle CallbackHandle = MaxDestState->OnDestinationStateReached.Add(MoveTemp(OnFeatureStateTransitionComplete));
			if (OutCallbackHandle)
			{
				*OutCallbackHandle = CallbackHandle;
			}
		}

		StateProperties.Destination = InDestination;
		UpdateStateMachine();

		return true;
	}

	if (TOptional<FGameFeaturePluginStateRange> NewDestination = StateProperties.Destination.Intersect(InDestination))
	{
		// The machine is already running so we can only transition to this range if it overlaps with our current range.
		// We can satisfy both ranges in this case.

		if (OutCallbackHandle)
		{
			OutCallbackHandle->Reset();
		}

		if (CurrentStateInfo.State < StateProperties.Destination)
		{
			StateProperties.Destination = *NewDestination;

			if (InDestination.Contains(CurrentStateInfo.State))
			{
				OnFeatureStateTransitionComplete.ExecuteIfBound(this, MakeValue());
				return true;
			}

			FDestinationGameFeaturePluginState* MinDestState = AllStates[InDestination.MinState]->AsDestinationState();
			FDelegateHandle CallbackHandle = MinDestState->OnDestinationStateReached.Add(MoveTemp(OnFeatureStateTransitionComplete));
			if (OutCallbackHandle)
			{
				*OutCallbackHandle = CallbackHandle;
			}
		}
		else if(CurrentStateInfo.State > StateProperties.Destination)
		{
			StateProperties.Destination = *NewDestination;

			if (InDestination.Contains(CurrentStateInfo.State))
			{
				OnFeatureStateTransitionComplete.ExecuteIfBound(this, MakeValue());
				return true;
			}

			FDestinationGameFeaturePluginState* MaxDestState = AllStates[InDestination.MaxState]->AsDestinationState();
			FDelegateHandle CallbackHandle = MaxDestState->OnDestinationStateReached.Add(MoveTemp(OnFeatureStateTransitionComplete));
			if (OutCallbackHandle)
			{
				*OutCallbackHandle = CallbackHandle;
			}
		}
		else
		{
			checkf(false, TEXT("IsRunning() returned true but state machine has reached destination!"));
		}

		return true;
	}

	// The requested state range is completely outside the the current state range so reject the request
	return false;
}

void UGameFeaturePluginStateMachine::UpdateStateMachine()
{
	EGameFeaturePluginState CurrentState = GetCurrentState();
	if (bInUpdateStateMachine)
	{
		UE_LOG(LogGameFeatures, Verbose, TEXT("Game feature state machine skipping update for %s in ::UpdateStateMachine. Current State: %s"), *GetGameFeatureName(), *UE::GameFeatures::ToString(CurrentState));
		return;
	}

	TOptional<TGuardValue<bool>> ScopeGuard(InPlace, bInUpdateStateMachine, true);
	
	using StateIt = std::underlying_type<EGameFeaturePluginState>::type;

	auto DoCallbacks = [this](const UE::GameFeatures::FResult& Result, StateIt Begin, StateIt End)
	{
		for (StateIt iState = Begin; iState < End; ++iState)
		{
			if (FDestinationGameFeaturePluginState* DestState = AllStates[iState]->AsDestinationState())
			{
				// Use a local callback on the stack. If SetDestination() is called from the callback then we don't want to stomp the callback
				// for the new state transition request.
				// Callback from terminal state could also trigger a GC that would destroy the state machine
				FDestinationGameFeaturePluginState::FOnDestinationStateReached LocalOnDestinationStateReached(MoveTemp(DestState->OnDestinationStateReached));
				DestState->OnDestinationStateReached.Clear();

				LocalOnDestinationStateReached.Broadcast(this, Result);
			}
		}
	};

	auto DoCallback = [&DoCallbacks](const UE::GameFeatures::FResult& Result, StateIt InState)
	{
		DoCallbacks(Result, InState, InState + 1);
	};

	bool bKeepProcessing = false;
	int32 NumTransitions = 0;
	const int32 MaxTransitions = 10000;
	do
	{
		bKeepProcessing = false;

		FGameFeaturePluginStateStatus StateStatus;
		// 达到注册状态才会开始加载Game Feature Data资产。
		AllStates[CurrentState]->UpdateState(StateStatus);

		if (StateStatus.TransitionToState == CurrentState)
		{
			UE_LOG(LogGameFeatures, Fatal, TEXT("Game feature state %s transitioning to itself. GameFeature: %s"), *UE::GameFeatures::ToString(CurrentState), *GetGameFeatureName());
		}

		if (StateStatus.TransitionToState != EGameFeaturePluginState::Uninitialized)
		{
			UE_LOG(LogGameFeatures, Verbose, TEXT("Game feature '%s' transitioning state (%s -> %s)"), *GetGameFeatureName(), *UE::GameFeatures::ToString(CurrentState), *UE::GameFeatures::ToString(StateStatus.TransitionToState));
			AllStates[CurrentState]->EndState();
			CurrentStateInfo = FGameFeaturePluginStateInfo(StateStatus.TransitionToState);
			CurrentState = StateStatus.TransitionToState;
			check(CurrentState != EGameFeaturePluginState::MAX);
			AllStates[CurrentState]->BeginState();

			if (CurrentState == EGameFeaturePluginState::Terminal)
			{
				// Remove from gamefeature subsystem before calling back in case this GFP is reloaded on callback,
				// but make sure we don't get destroyed from a GC during a callback
				UGameFeaturesSubsystem::Get().BeginTermination(this);
			}

			if (StateProperties.bTryCancel && AllStates[CurrentState]->GetStateType() != EGameFeaturePluginStateType::Transition)
			{
				StateProperties.Destination = FGameFeaturePluginStateRange(CurrentState);

				StateProperties.bTryCancel = false;
				bKeepProcessing = false;

				// Make sure bInUpdateStateMachine is not set while processing callbacks if we are at our destination
				ScopeGuard.Reset();

				// For all callbacks, return the CanceledResult
				DoCallbacks(UE::GameFeatures::CanceledResult, 0, EGameFeaturePluginState::MAX);

				// Must be called after transtition callbacks, UGameFeaturesSubsystem::ChangeGameFeatureTargetStateComplete may remove the this machine from the subsystem
				FGameFeaturePluginStateMachineProperties::FOnTransitionCanceled LocalOnTransitionCanceled(MoveTemp(StateProperties.OnTransitionCanceled));
				StateProperties.OnTransitionCanceled.Clear();
				LocalOnTransitionCanceled.Broadcast(this);
			}
			else if (const bool bError = !StateStatus.TransitionResult.HasValue(); bError)
			{
				check(IsValidErrorState(CurrentState));
				StateProperties.Destination = FGameFeaturePluginStateRange(CurrentState);

				bKeepProcessing = false;
				
				// Make sure bInUpdateStateMachine is not set while processing callbacks if we are at our destination
				ScopeGuard.Reset();

				// In case of an error, callback all possible callbacks
				DoCallbacks(StateStatus.TransitionResult, 0, EGameFeaturePluginState::MAX);
			}
			else
			{
				bKeepProcessing = AllStates[CurrentState]->GetStateType() == EGameFeaturePluginStateType::Transition || !StateProperties.Destination.Contains(CurrentState);
				if (!bKeepProcessing)
				{
					// Make sure bInUpdateStateMachine is not set while processing callbacks if we are at our destination
					ScopeGuard.Reset();
				}

				DoCallback(StateStatus.TransitionResult, CurrentState);
			}

			if (CurrentState == EGameFeaturePluginState::Terminal)
			{
				check(bKeepProcessing == false);
				// Now that callbacks are done this machine can be cleaned up
				UGameFeaturesSubsystem::Get().FinishTermination(this);
				MarkAsGarbage();
			}
		}

		if (NumTransitions++ > MaxTransitions)
		{
			UE_LOG(LogGameFeatures, Fatal, TEXT("Infinite loop in game feature state machine transitions. Current state %s. GameFeature: %s"), *UE::GameFeatures::ToString(CurrentState), *GetGameFeatureName());
		}
	} while (bKeepProcessing);
}
```

* Engine\Plugins\Experimental\GameFeatures\Source\GameFeatures\Private\GameFeaturePluginStateMachine.cpp
```Cpp
struct FGameFeaturePluginState_Registering : public FGameFeaturePluginState
{
	virtual void UpdateState(FGameFeaturePluginStateStatus& StateStatus) override
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(GFP_Registering);
		const FString PluginFolder = FPaths::GetPath(StateProperties.PluginInstalledFilename);
		UGameplayTagsManager::Get().AddTagIniSearchPath(PluginFolder / TEXT("Config") / TEXT("Tags"));

		const FString BackupGameFeatureDataPath = FString::Printf(TEXT("/%s/%s.%s"), *StateProperties.PluginName, *StateProperties.PluginName, *StateProperties.PluginName);

		FString PreferredGameFeatureDataPath = TEXT("/") + StateProperties.PluginName + TEXT("/GameFeatureData.GameFeatureData");
		// Allow game feature location to be overriden globally and from within the plugin
		FString OverrideIniPathName = StateProperties.PluginName + TEXT("_Override");
		FString OverridePath = GConfig->GetStr(TEXT("GameFeatureData"), *OverrideIniPathName, GGameIni);
		if (OverridePath.IsEmpty())
		{
			const FString SettingsOverride = PluginFolder / TEXT("Config") / TEXT("Settings.ini");
			if (FPaths::FileExists(SettingsOverride))
			{
				GConfig->LoadFile(SettingsOverride);
				OverridePath = GConfig->GetStr(TEXT("GameFeatureData"), TEXT("Override"), SettingsOverride);
				GConfig->UnloadFile(SettingsOverride);
			}
		}
		if (!OverridePath.IsEmpty())
		{
			PreferredGameFeatureDataPath = OverridePath;
		}
		
		auto LoadGameFeatureData = [](const FString& Path) -> UGameFeatureData*
		{
			TSharedPtr<FStreamableHandle> GameFeatureDataHandle;
			if (FPackageName::DoesPackageExist(Path))
			{
				//开始加载Game Feature Data资产。
				GameFeatureDataHandle = UGameFeaturesSubsystem::LoadGameFeatureData(Path);
				// @todo make this async. For now we just wait
				if (GameFeatureDataHandle.IsValid())
				{
					GameFeatureDataHandle->WaitUntilComplete(0.0f, false);
					return Cast<UGameFeatureData>(GameFeatureDataHandle->GetLoadedAsset());
				}
			}

			return nullptr;
		};

		StateProperties.GameFeatureData = LoadGameFeatureData(PreferredGameFeatureDataPath);
		if (!StateProperties.GameFeatureData)
		{
			StateProperties.GameFeatureData = LoadGameFeatureData(BackupGameFeatureDataPath);
		}

		if (StateProperties.GameFeatureData)
		{
			StateProperties.GameFeatureData->InitializeBasePluginIniFile(StateProperties.PluginInstalledFilename);
			StateStatus.SetTransition(EGameFeaturePluginState::Registered);

			check(StateProperties.AddedPrimaryAssetTypes.Num() == 0);
			UGameFeaturesSubsystem::Get().AddGameFeatureToAssetManager(StateProperties.GameFeatureData, StateProperties.PluginName, StateProperties.AddedPrimaryAssetTypes);
			// 触发Game Feature Registering回调委托
			UGameFeaturesSubsystem::Get().OnGameFeatureRegistering(StateProperties.GameFeatureData, StateProperties.PluginName, StateProperties.PluginIdentifier.GetFullPluginURL());
		}
		else
		{
			// The gamefeaturedata does not exist. The pak file may not be openable or this is a builtin plugin where the pak file does not exist.
			StateStatus.SetTransitionError(EGameFeaturePluginState::ErrorRegistering, GetErrorResult(TEXT("Plugin_Missing_GameFeatureData")));
		}
	}
};
```

# Game Abilities System

> The Gameplay Ability System is a highly flexible framework for building the types of abilities and attributes that you might find in an RPG or MOBA title. You can build actions or passive abilities for the characters in your games to use, and status effects that can build up or wear down various attributes as a result of these actions, additionally you can implement "cooldown" timers or resource costs to regulate the usage of these actions, change the level of the ability and its effects at each level, activate particle, sound effects, and more. The Gameplay Ability System can help you to design, implement, and efficiently network in-game abilities from as simple as jumping to as complex as your favorite character's ability set in any modern RPG or MOBA title.

游戏技能系统是一个高度灵活的Gameplay框架，虽说相当合适做RPG或者MOBA，但是其实用来做别的类型应该也可以。

## Key Concept

* Gameplay Cue

> Gameplay Cues are Actors and UObjects responsible for running visual and sound effects, and are the preferred method for replicating cosmetic feedback in a multiplayer game. When you create a Gameplay Cue, you run the logic for the effects you want to play inside its Event Graph. Gameplay Cues can be associated with a series of Gameplay Tags, and any Gameplay Effect matching those tags will automatically apply them. 

Gameplay Cue极度依赖Gameplay Tag去添加、运行视觉特效和音效，在Cpp中没有明确的UGameplayCue类，但有UGameplayCueManager这种管理类，负责分发Gameplay Cue和生成GameplayCueNotify Actor。

## Key Class

* UAbilitySystemComponent

> The core ActorComponent for interfacing with the Gameplay Abilities System.

继承UGameplayTasksComponent、IGameplayTagAssetInteface和IAbilitySystemReplicationProxyInterface，与游戏玩法技能系统交互的核心Actor组件，所有打算使用Gameplay Abilities System的Actor必须拥有该组件。

* UGameplayAbility

> Abilities define custom gameplay logic that can be activated by players or external game logic.

继承UObject和IGameplayTaskOwnerInterface，游戏技能的玩法逻辑定义和实现，可由玩家激活或者外部游戏逻辑。

* FGameplayAttribute

> Describes a FGameplayAttributeData or float property inside an attribute set. Using this provides editor UI and helper functions.

用来存储游戏数值和相关属性的结构体，例如移动速度、健康值、攻击速度等。

* UGameplayAbilitySet

> This is an example DataAsset that could be used for defining a set of abilities to give to an AbilitySystemComponent and bind to an input command.

继承UDataAsset，一个数据资产定义，包含着UGameplayAbility成员和EGameplayAbilityInputBind成员的结构体数组，其中EGameplayAbility是负责输入绑定的枚举定义。

* FGameplayAbilitySpec

> An activatable ability spec, hosted on the ability system component. This defines both what the ability is (what class, what level, input binding etc)
> and also holds runtime state that must be kept outside of the ability being instanced/activated.

继承FFastArraySerializerItem，可激活技能的详细信息类，一般由技能系统组件管理。该类包含技能的详细定义信息（类型、等级、输入绑定等），并且保持未被实例化或者未被激活的技能的运行状态。

* FGameplayAbilitySpecHandle

> Handle that points to a specific granted ability. These are globally unique.

全局唯一，指向特定要赋予的技能的引用。只有一个32位整数成员。

* FGameplayAbilityActorInfo

> Cached data associated with an Actor using an Ability.
> -Initialized from an AActor* in InitFromActor
> -Abilities use this to know what to actor upon. E.g., instead of being coupled to a specific actor class.
> -These are generally passed around as pointers to support polymorphism.
> -Projects can override UAbilitySystemGlobals::AllocAbilityActorInfo to override the default struct type that is created.

用来缓存使用该技能的Actor信息类。

* FGameplayAbilityActivationInfo

> Data tied to a specific activation of an ability.
> -Tell us whether we are the authority, if we are predicting, confirmed, etc.
> -Holds current and previous PredictionKey
> -Generally not meant to be subclassed in projects.
> -Passed around by value since the struct is small.

技能激活的相关信息类。

* UGameplayEffect

> The GameplayEffect definition. This is the data asset defined in the editor that drives everything.
> This is only blueprintable to allow for templating gameplay effects. Gameplay effects should NOT contain blueprint graphs.

继承UObject和IGameplayTagAssetInterface，字面意思似乎是玩法效果，容易理解成特效美术相关的意义。

> Gameplay Effects provide a way to alter Gameplay Attributes instantaneously or over time (commonly known as "buffs" and "debuffs"), such as subtracting magic points when casting a spell, granting a movement speed boost while a "Sprint" Ability is active, or gradually restoring health points over the lifetime of a dose of healing medicine.

实际上根据官方文档、注释描述和具体的蓝图脚本逻辑，这是在游戏运行时随意改动Game Attributes的类，可以理解成一个游戏增益/减益效果的类，比如按Shift进行冲刺可以是一种加速增益、遭到毒药侵害随时间减少健康值是一种伤害性减益。

* FGameplayEffectSpec

> GameplayEffect Specification. Tells us:
> -What UGameplayEffect (const data)
> -What Level
> -Who instigated
> FGameplayEffectSpec is modifiable. We start with initial conditions and modifications be applied to it. In this sense, it is stateful/mutable but it
> is still distinct from an FActiveGameplayEffect which in an applied instance of an FGameplayEffectSpec.

技能效果的详细信息类

* UAttributeSet

> Defines the set of all GameplayAttributes for your game
> Games should subclass this and add FGameplayAttributeData properties to represent attributes like health, damage, etc
> AttributeSets are added to the actors as subobjects, and then registered with the AbilitySystemComponent
> It often desired to have several sets per project that inherit from each other
> You could make a base health set, then have a player set that inherits from it and adds more attributes

继承UObject，可以添加给Actor的属性集合类，会被注册进AbilitySystemComponent，GameEffect的逻辑及相关数值计算都会影响此类的成员。

* FGameplayEffectExecutionDefinition

> Struct representing the definition of a custom execution for a gameplay effect.
> Custom executions run special logic from an outside class each time the gameplay effect executes.

游戏效果的具体逻辑执行定义的结构体，每一次游戏效果的执行都可以从外部类中运行。

* UGameplayEffectCalculation

继承UObject

* UGameplayEffectExecutionCalculation

继承UGameplayEffectCalculation，有虚成员函数Execute，负责执行游戏效果的具体逻辑和相关的机制计算。

* Game Ability Task

> AbilityTasks are small, self contained operations that can be performed while executing an ability. They are latent/asynchronous is nature. They will generally follow the pattern of 'start something and wait until it is finished or interrupted'.

类似Async Task(异步任务)，可在执行技能时同步进行的一种小型且独立的操作。一般都是异步的，可用来等待某个流程完成后执行操作。可以继承Ability Task来进行扩展达到业务需求，可参考官方AbilityTask_PlayMontageAndWait。

## Flow Diagram

{% asset_img "Unreal Engine 5 Lyra Starter Game Game Ability System Work Flow.png" Game Ability System Work Flow %}

## Key Implementation

* Engine\Plugins\Runtime\GameplayAbilities\Source\GameplayAbilities\Private\Abilties\GameplayAbilitySet.cpp
```Cpp
void UGameplayAbilitySet::GiveAbilities(UAbilitySystemComponent* AbilitySystemComponent) const
{
	for (const FGameplayAbilityBindInfo& BindInfo : Abilities)
	{
		if (BindInfo.GameplayAbilityClass)
		{
			AbilitySystemComponent->GiveAbility(FGameplayAbilitySpec(BindInfo.GameplayAbilityClass, 1, (int32)BindInfo.Command));
		}
	}
}
```

* Engine\Plugins\Runtime\GameplayAbilities\Source\GameplayAbilities\Private\AbilitySystemComponent_Abilities.cpp
```Cpp
FGameplayAbilitySpecHandle UAbilitySystemComponent::GiveAbility(const FGameplayAbilitySpec& Spec)
{
	if (!IsValid(Spec.Ability))
	{
		ABILITY_LOG(Error, TEXT("GiveAbility called with an invalid Ability Class."));

		return FGameplayAbilitySpecHandle();
	}

	if (!IsOwnerActorAuthoritative())
	{
		ABILITY_LOG(Error, TEXT("GiveAbility called on ability %s on the client, not allowed!"), *Spec.Ability->GetName());

		return FGameplayAbilitySpecHandle();
	}

	// If locked, add to pending list. The Spec.Handle is not regenerated when we receive, so returning this is ok.
	if (AbilityScopeLockCount > 0)
	{
		AbilityPendingAdds.Add(Spec);
		return Spec.Handle;
	}

	ABILITYLIST_SCOPE_LOCK();
	// 把技能添加到该技能系统组件的技能容器
	FGameplayAbilitySpec& OwnedSpec = ActivatableAbilities.Items[ActivatableAbilities.Items.Add(Spec)];
	
	//网络复制相关，检查技能实例化策略，是否需要将技能在Actor上进行实例化。
	if (OwnedSpec.Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerActor)
	{
		// Create the instance at creation time
		CreateNewInstanceOfAbility(OwnedSpec, Spec.Ability);
	}
	
	OnGiveAbility(OwnedSpec);
	MarkAbilitySpecDirty(OwnedSpec, true);

	return OwnedSpec.Handle;
}

FGameplayAbilitySpecHandle UAbilitySystemComponent::GiveAbilityAndActivateOnce(FGameplayAbilitySpec& Spec, const FGameplayEventData* GameplayEventData)
{
	if (!IsValid(Spec.Ability))
	{
		ABILITY_LOG(Error, TEXT("GiveAbilityAndActivateOnce called with an invalid Ability Class."));

		return FGameplayAbilitySpecHandle();
	}

	if (Spec.Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::NonInstanced || Spec.Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalOnly)
	{
		ABILITY_LOG(Error, TEXT("GiveAbilityAndActivateOnce called on ability %s that is non instanced or won't execute on server, not allowed!"), *Spec.Ability->GetName());

		return FGameplayAbilitySpecHandle();
	}

	if (!IsOwnerActorAuthoritative())
	{
		ABILITY_LOG(Error, TEXT("GiveAbilityAndActivateOnce called on ability %s on the client, not allowed!"), *Spec.Ability->GetName());

		return FGameplayAbilitySpecHandle();
	}

	Spec.bActivateOnce = true;

	FGameplayAbilitySpecHandle AddedAbilityHandle = GiveAbility(Spec);

	FGameplayAbilitySpec* FoundSpec = FindAbilitySpecFromHandle(AddedAbilityHandle);

	if (FoundSpec)
	{
		FoundSpec->RemoveAfterActivation = true;

		if (!InternalTryActivateAbility(AddedAbilityHandle, FPredictionKey(), nullptr, nullptr, GameplayEventData))
		{
			// We failed to activate it, so remove it now
			ClearAbility(AddedAbilityHandle);

			return FGameplayAbilitySpecHandle();
		}
	}

	return AddedAbilityHandle;
}

bool UAbilitySystemComponent::TryActivateAbilitiesByTag(const FGameplayTagContainer& GameplayTagContainer, bool bAllowRemoteActivation)
{
	TArray<FGameplayAbilitySpec*> AbilitiesToActivate;
	GetActivatableGameplayAbilitySpecsByAllMatchingTags(GameplayTagContainer, AbilitiesToActivate);

	bool bSuccess = false;

	for (auto GameplayAbilitySpec : AbilitiesToActivate)
	{
		bSuccess |= TryActivateAbility(GameplayAbilitySpec->Handle, bAllowRemoteActivation);
	}

	return bSuccess;
}

bool UAbilitySystemComponent::TryActivateAbilityByClass(TSubclassOf<UGameplayAbility> InAbilityToActivate, bool bAllowRemoteActivation)
{
	bool bSuccess = false;

	const UGameplayAbility* const InAbilityCDO = InAbilityToActivate.GetDefaultObject();

	for (const FGameplayAbilitySpec& Spec : ActivatableAbilities.Items)
	{
		if (Spec.Ability == InAbilityCDO)
		{
			bSuccess |= TryActivateAbility(Spec.Handle, bAllowRemoteActivation);
			break;
		}
	}

	return bSuccess;
}

bool UAbilitySystemComponent::TryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate, bool bAllowRemoteActivation)
{
	FGameplayTagContainer FailureTags;
	FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(AbilityToActivate);
	if (!Spec)
	{
		ABILITY_LOG(Warning, TEXT("TryActivateAbility called with invalid Handle"));
		return false;
	}

	// don't activate abilities that are waiting to be removed
	if (Spec->PendingRemove || Spec->RemoveAfterActivation)
	{
		return false;
	}

	UGameplayAbility* Ability = Spec->Ability;

	if (!Ability)
	{
		ABILITY_LOG(Warning, TEXT("TryActivateAbility called with invalid Ability"));
		return false;
	}

	const FGameplayAbilityActorInfo* ActorInfo = AbilityActorInfo.Get();

	// make sure the ActorInfo and then Actor on that FGameplayAbilityActorInfo are valid, if not bail out.
	if (ActorInfo == nullptr || !ActorInfo->OwnerActor.IsValid() || !ActorInfo->AvatarActor.IsValid())
	{
		return false;
	}

		
	const ENetRole NetMode = ActorInfo->AvatarActor->GetLocalRole();

	// This should only come from button presses/local instigation (AI, etc).
	if (NetMode == ROLE_SimulatedProxy)
	{
		return false;
	}

	bool bIsLocal = AbilityActorInfo->IsLocallyControlled();

	// Check to see if this a local only or server only ability, if so either remotely execute or fail
	if (!bIsLocal && (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalOnly || Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted))
	{
		if (bAllowRemoteActivation)
		{
			ClientTryActivateAbility(AbilityToActivate);
			return true;
		}

		ABILITY_LOG(Log, TEXT("Can't activate LocalOnly or LocalPredicted ability %s when not local."), *Ability->GetName());
		return false;
	}

	if (NetMode != ROLE_Authority && (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerOnly || Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerInitiated))
	{
		if (bAllowRemoteActivation)
		{
			FScopedCanActivateAbilityLogEnabler LogEnabler;
			if (Ability->CanActivateAbility(AbilityToActivate, ActorInfo, nullptr, nullptr, &FailureTags))
			{
				// No prediction key, server will assign a server-generated key
				CallServerTryActivateAbility(AbilityToActivate, Spec->InputPressed, FPredictionKey());
				return true;
			}
			else
			{
				NotifyAbilityFailed(AbilityToActivate, Ability, FailureTags);
				return false;
			}
		}

		ABILITY_LOG(Log, TEXT("Can't activate ServerOnly or ServerInitiated ability %s when not the server."), *Ability->GetName());
		return false;
	}

	return InternalTryActivateAbility(AbilityToActivate);
}

bool UAbilitySystemComponent::InternalTryActivateAbility(FGameplayAbilitySpecHandle Handle, FPredictionKey InPredictionKey, UGameplayAbility** OutInstancedAbility, FOnGameplayAbilityEnded::FDelegate* OnGameplayAbilityEndedDelegate, const FGameplayEventData* TriggerEventData)
{
	const FGameplayTag& NetworkFailTag = UAbilitySystemGlobals::Get().ActivateFailNetworkingTag;
	
	InternalTryActivateAbilityFailureTags.Reset();

	if (Handle.IsValid() == false)
	{
		ABILITY_LOG(Warning, TEXT("InternalTryActivateAbility called with invalid Handle! ASC: %s. AvatarActor: %s"), *GetPathName(), *GetNameSafe(GetAvatarActor_Direct()));
		return false;
	}

	FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
	if (!Spec)
	{
		ABILITY_LOG(Warning, TEXT("InternalTryActivateAbility called with a valid handle but no matching ability was found. Handle: %s ASC: %s. AvatarActor: %s"), *Handle.ToString(), *GetPathName(), *GetNameSafe(GetAvatarActor_Direct()));
		return false;
	}

	// Lock ability list so our Spec doesn't get destroyed while activating
	ABILITYLIST_SCOPE_LOCK();

	const FGameplayAbilityActorInfo* ActorInfo = AbilityActorInfo.Get();

	// make sure the ActorInfo and then Actor on that FGameplayAbilityActorInfo are valid, if not bail out.
	if (ActorInfo == nullptr || !ActorInfo->OwnerActor.IsValid() || !ActorInfo->AvatarActor.IsValid())
	{
		return false;
	}

	// This should only come from button presses/local instigation (AI, etc)
	ENetRole NetMode = ROLE_SimulatedProxy;

	// Use PC netmode if its there
	if (APlayerController* PC = ActorInfo->PlayerController.Get())
	{
		NetMode = PC->GetLocalRole();
	}
	// Fallback to avataractor otherwise. Edge case: avatar "dies" and becomes torn off and ROLE_Authority. We don't want to use this case (use PC role instead).
	else if (AActor* LocalAvatarActor = GetAvatarActor_Direct())
	{
		NetMode = LocalAvatarActor->GetLocalRole();
	}

	if (NetMode == ROLE_SimulatedProxy)
	{
		return false;
	}

	bool bIsLocal = AbilityActorInfo->IsLocallyControlled();

	UGameplayAbility* Ability = Spec->Ability;

	if (!Ability)
	{
		ABILITY_LOG(Warning, TEXT("InternalTryActivateAbility called with invalid Ability"));
		return false;
	}

	// Check to see if this a local only or server only ability, if so don't execute
	if (!bIsLocal)
	{
		if (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalOnly || (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted && !InPredictionKey.IsValidKey()))
		{
			// If we have a valid prediction key, the ability was started on the local client so it's okay

			ABILITY_LOG(Warning, TEXT("Can't activate LocalOnly or LocalPredicted ability %s when not local! Net Execution Policy is %d."), *Ability->GetName(), (int32)Ability->GetNetExecutionPolicy());

			if (NetworkFailTag.IsValid())
			{
				InternalTryActivateAbilityFailureTags.AddTag(NetworkFailTag);
				NotifyAbilityFailed(Handle, Ability, InternalTryActivateAbilityFailureTags);
			}

			return false;
		}		
	}

	if (NetMode != ROLE_Authority && (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerOnly || Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerInitiated))
	{
		ABILITY_LOG(Warning, TEXT("Can't activate ServerOnly or ServerInitiated ability %s when not the server! Net Execution Policy is %d."), *Ability->GetName(), (int32)Ability->GetNetExecutionPolicy());

		if (NetworkFailTag.IsValid())
		{
			InternalTryActivateAbilityFailureTags.AddTag(NetworkFailTag);
			NotifyAbilityFailed(Handle, Ability, InternalTryActivateAbilityFailureTags);
		}

		return false;
	}

	// If it's instance once the instanced ability will be set, otherwise it will be null
	UGameplayAbility* InstancedAbility = Spec->GetPrimaryInstance();

	const FGameplayTagContainer* SourceTags = nullptr;
	const FGameplayTagContainer* TargetTags = nullptr;
	if (TriggerEventData != nullptr)
	{
		SourceTags = &TriggerEventData->InstigatorTags;
		TargetTags = &TriggerEventData->TargetTags;
	}

	{
		// If we have an instanced ability, call CanActivateAbility on it.
		// Otherwise we always do a non instanced CanActivateAbility check using the CDO of the Ability.
		UGameplayAbility* const CanActivateAbilitySource = InstancedAbility ? InstancedAbility : Ability;
		FScopedCanActivateAbilityLogEnabler LogEnabler;

		if (!CanActivateAbilitySource->CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, &InternalTryActivateAbilityFailureTags))
		{
			NotifyAbilityFailed(Handle, CanActivateAbilitySource, InternalTryActivateAbilityFailureTags);
			return false;
		}
	}

	// If we're instance per actor and we're already active, don't let us activate again as this breaks the graph
	if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerActor)
	{
		if (Spec->IsActive())
		{
			if (Ability->bRetriggerInstancedAbility && InstancedAbility)
			{
				bool bReplicateEndAbility = true;
				bool bWasCancelled = false;
				InstancedAbility->EndAbility(Handle, ActorInfo, Spec->ActivationInfo, bReplicateEndAbility, bWasCancelled);
			}
			else
			{
				ABILITY_LOG(Verbose, TEXT("Can't activate instanced per actor ability %s when their is already a currently active instance for this actor."), *Ability->GetName());
				return false;
			}
		}
	}

	// Make sure we have a primary
	if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerActor && !InstancedAbility)
	{
		ABILITY_LOG(Warning, TEXT("InternalTryActivateAbility called but instanced ability is missing! NetMode: %d. Ability: %s"), (int32)NetMode, *Ability->GetName());
		return false;
	}

	// Setup a fresh ActivationInfo for this AbilitySpec.
	Spec->ActivationInfo = FGameplayAbilityActivationInfo(ActorInfo->OwnerActor.Get());
	FGameplayAbilityActivationInfo &ActivationInfo = Spec->ActivationInfo;

	// If we are the server or this is local only
	if (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalOnly || (NetMode == ROLE_Authority))
	{
		// if we're the server and don't have a valid key or this ability should be started on the server create a new activation key
		bool bCreateNewServerKey = NetMode == ROLE_Authority &&
			(!InPredictionKey.IsValidKey() ||
			 (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerInitiated ||
			  Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerOnly));
		if (bCreateNewServerKey)
		{
			ActivationInfo.ServerSetActivationPredictionKey(FPredictionKey::CreateNewServerInitiatedKey(this));
		}
		else if (InPredictionKey.IsValidKey())
		{
			// Otherwise if available, set the prediction key to what was passed up
			ActivationInfo.ServerSetActivationPredictionKey(InPredictionKey);
		}

		// we may have changed the prediction key so we need to update the scoped key to match
		FScopedPredictionWindow ScopedPredictionWindow(this, ActivationInfo.GetActivationPredictionKey());

		// ----------------------------------------------
		// Tell the client that you activated it (if we're not local and not server only)
		// ----------------------------------------------
		if (!bIsLocal && Ability->GetNetExecutionPolicy() != EGameplayAbilityNetExecutionPolicy::ServerOnly)
		{
			if (TriggerEventData)
			{
				ClientActivateAbilitySucceedWithEventData(Handle, ActivationInfo.GetActivationPredictionKey(), *TriggerEventData);
			}
			else
			{
				ClientActivateAbilitySucceed(Handle, ActivationInfo.GetActivationPredictionKey());
			}
			
			// This will get copied into the instanced abilities
			ActivationInfo.bCanBeEndedByOtherInstance = Ability->bServerRespectsRemoteAbilityCancellation;
		}

		// ----------------------------------------------
		//	Call ActivateAbility (note this could end the ability too!)
		// ----------------------------------------------

		// Create instance of this ability if necessary
		if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerExecution)
		{
			InstancedAbility = CreateNewInstanceOfAbility(*Spec, Ability);
			InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
		else if (InstancedAbility)
		{
			InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
		else
		{
			Ability->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
	}
	else if (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted)
	{
		// Flush server moves that occurred before this ability activation so that the server receives the RPCs in the correct order
		// Necessary to prevent abilities that trigger animation root motion or impact movement from causing network corrections
		if (!ActorInfo->IsNetAuthority())
		{
			ACharacter* AvatarCharacter = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
			if (AvatarCharacter)
			{
				UCharacterMovementComponent* AvatarCharMoveComp = Cast<UCharacterMovementComponent>(AvatarCharacter->GetMovementComponent());
				if (AvatarCharMoveComp)
				{
					AvatarCharMoveComp->FlushServerMoves();
				}
			}
		}

		// This execution is now officially EGameplayAbilityActivationMode:Predicting and has a PredictionKey
		FScopedPredictionWindow ScopedPredictionWindow(this, true);

		ActivationInfo.SetPredicting(ScopedPredictionKey);
		
		// This must be called immediately after GeneratePredictionKey to prevent problems with recursively activating abilities
		if (TriggerEventData)
		{
			ServerTryActivateAbilityWithEventData(Handle, Spec->InputPressed, ScopedPredictionKey, *TriggerEventData);
		}
		else
		{
			CallServerTryActivateAbility(Handle, Spec->InputPressed, ScopedPredictionKey);
		}

		// When this prediction key is caught up, we better know if the ability was confirmed or rejected
		ScopedPredictionKey.NewCaughtUpDelegate().BindUObject(this, &UAbilitySystemComponent::OnClientActivateAbilityCaughtUp, Handle, ScopedPredictionKey.Current);

		if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerExecution)
		{
			// For now, only NonReplicated + InstancedPerExecution abilities can be Predictive.
			// We lack the code to predict spawning an instance of the execution and then merge/combine
			// with the server spawned version when it arrives.

			if (Ability->GetReplicationPolicy() == EGameplayAbilityReplicationPolicy::ReplicateNo)
			{
				InstancedAbility = CreateNewInstanceOfAbility(*Spec, Ability);
				InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
			}
			else
			{
				ABILITY_LOG(Error, TEXT("InternalTryActivateAbility called on ability %s that is InstancePerExecution and Replicated. This is an invalid configuration."), *Ability->GetName() );
			}
		}
		else if (InstancedAbility)
		{
			InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
		else 
		{
			Ability->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
	}
	
	if (InstancedAbility)
	{
		if (OutInstancedAbility)
		{
			*OutInstancedAbility = InstancedAbility;
		}

		InstancedAbility->SetCurrentActivationInfo(ActivationInfo);	// Need to push this to the ability if it was instanced.
	}

	MarkAbilitySpecDirty(*Spec);

	AbilityLastActivatedTime = GetWorld()->GetTimeSeconds();

	return true;
}

FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(const FGameplayEffectSpec &Spec, FPredictionKey PredictionKey)
{
#if WITH_SERVER_CODE
	SCOPE_CYCLE_COUNTER(STAT_AbilitySystemComp_ApplyGameplayEffectSpecToSelf);
#endif

	// Scope lock the container after the addition has taken place to prevent the new effect from potentially getting mangled during the remainder
	// of the add operation
	FScopedActiveGameplayEffectLock ScopeLock(ActiveGameplayEffects);

	FScopeCurrentGameplayEffectBeingApplied ScopedGEApplication(&Spec, this);

	const bool bIsNetAuthority = IsOwnerActorAuthoritative();

	// Check Network Authority
	if (!HasNetworkAuthorityToApplyGameplayEffect(PredictionKey))
	{
		return FActiveGameplayEffectHandle();
	}

	// Don't allow prediction of periodic effects
	if(PredictionKey.IsValidKey() && Spec.GetPeriod() > 0.f)
	{
		if(IsOwnerActorAuthoritative())
		{
			// Server continue with invalid prediction key
			PredictionKey = FPredictionKey();
		}
		else
		{
			// Client just return now
			return FActiveGameplayEffectHandle();
		}
	}

	// Are we currently immune to this? (ApplicationImmunity)
	const FActiveGameplayEffect* ImmunityGE=nullptr;
	if (ActiveGameplayEffects.HasApplicationImmunityToSpec(Spec, ImmunityGE))
	{
		OnImmunityBlockGameplayEffect(Spec, ImmunityGE);
		return FActiveGameplayEffectHandle();
	}

	// Check AttributeSet requirements: make sure all attributes are valid
	// We may want to cache this off in some way to make the runtime check quicker.
	// We also need to handle things in the execution list
	for (const FGameplayModifierInfo& Mod : Spec.Def->Modifiers)
	{
		if (!Mod.Attribute.IsValid())
		{
			ABILITY_LOG(Warning, TEXT("%s has a null modifier attribute."), *Spec.Def->GetPathName());
			return FActiveGameplayEffectHandle();
		}
	}

	// check if the effect being applied actually succeeds
	float ChanceToApply = Spec.GetChanceToApplyToTarget();
	if ((ChanceToApply < 1.f - SMALL_NUMBER) && (FMath::FRand() > ChanceToApply))
	{
		return FActiveGameplayEffectHandle();
	}

	// Get MyTags.
	//	We may want to cache off a GameplayTagContainer instead of rebuilding it every time.
	//	But this will also be where we need to merge in context tags? (Headshot, executing ability, etc?)
	//	Or do we push these tags into (our copy of the spec)?

	{
		// Note: static is ok here since the scope is so limited, but wider usage of MyTags is not safe since this function can be recursively called
		static FGameplayTagContainer MyTags;
		MyTags.Reset();

		GetOwnedGameplayTags(MyTags);

		if (Spec.Def->ApplicationTagRequirements.RequirementsMet(MyTags) == false)
		{
			return FActiveGameplayEffectHandle();
		}

		if (!Spec.Def->RemovalTagRequirements.IsEmpty() && Spec.Def->RemovalTagRequirements.RequirementsMet(MyTags) == true)
		{
			return FActiveGameplayEffectHandle();
		}
	}

	// Custom application requirement check
	for (const TSubclassOf<UGameplayEffectCustomApplicationRequirement>& AppReq : Spec.Def->ApplicationRequirements)
	{
		if (*AppReq && AppReq->GetDefaultObject<UGameplayEffectCustomApplicationRequirement>()->CanApplyGameplayEffect(Spec.Def, Spec, this) == false)
		{
			return FActiveGameplayEffectHandle();
		}
	}
	bIsNetDirty = true;

	// Clients should treat predicted instant effects as if they have infinite duration. The effects will be cleaned up later.
	bool bTreatAsInfiniteDuration = GetOwnerRole() != ROLE_Authority && PredictionKey.IsLocalClientKey() && Spec.Def->DurationPolicy == EGameplayEffectDurationType::Instant;

	// Make sure we create our copy of the spec in the right place
	// We initialize the FActiveGameplayEffectHandle here with INDEX_NONE to handle the case of instant GE
	// Initializing it like this will set the bPassedFiltersAndWasExecuted on the FActiveGameplayEffectHandle to true so we can know that we applied a GE
	FActiveGameplayEffectHandle	MyHandle(INDEX_NONE);
	bool bInvokeGameplayCueApplied = Spec.Def->DurationPolicy != EGameplayEffectDurationType::Instant; // Cache this now before possibly modifying predictive instant effect to infinite duration effect.
	bool bFoundExistingStackableGE = false;

	FActiveGameplayEffect* AppliedEffect = nullptr;

	FGameplayEffectSpec* OurCopyOfSpec = nullptr;
	TSharedPtr<FGameplayEffectSpec> StackSpec;
	{
		if (Spec.Def->DurationPolicy != EGameplayEffectDurationType::Instant || bTreatAsInfiniteDuration)
		{
			AppliedEffect = ActiveGameplayEffects.ApplyGameplayEffectSpec(Spec, PredictionKey, bFoundExistingStackableGE);
			if (!AppliedEffect)
			{
				return FActiveGameplayEffectHandle();
			}

			MyHandle = AppliedEffect->Handle;
			OurCopyOfSpec = &(AppliedEffect->Spec);

			// Log results of applied GE spec
			if (UE_LOG_ACTIVE(VLogAbilitySystem, Log))
			{
				ABILITY_VLOG(GetOwnerActor(), Log, TEXT("Applied %s"), *OurCopyOfSpec->Def->GetFName().ToString());

				for (const FGameplayModifierInfo& Modifier : Spec.Def->Modifiers)
				{
					float Magnitude = 0.f;
					Modifier.ModifierMagnitude.AttemptCalculateMagnitude(Spec, Magnitude);
					ABILITY_VLOG(GetOwnerActor(), Log, TEXT("         %s: %s %f"), *Modifier.Attribute.GetName(), *EGameplayModOpToString(Modifier.ModifierOp), Magnitude);
				}
			}
		}

		if (!OurCopyOfSpec)
		{
			StackSpec = TSharedPtr<FGameplayEffectSpec>(new FGameplayEffectSpec(Spec));
			OurCopyOfSpec = StackSpec.Get();
			UAbilitySystemGlobals::Get().GlobalPreGameplayEffectSpecApply(*OurCopyOfSpec, this);
			OurCopyOfSpec->CaptureAttributeDataFromTarget(this);
		}

		// if necessary add a modifier to OurCopyOfSpec to force it to have an infinite duration
		if (bTreatAsInfiniteDuration)
		{
			// This should just be a straight set of the duration float now
			OurCopyOfSpec->SetDuration(UGameplayEffect::INFINITE_DURATION, true);
		}
	}

	if (OurCopyOfSpec)
	{
		// Update (not push) the global spec being applied [we want to switch it to our copy, from the const input copy)
		UAbilitySystemGlobals::Get().SetCurrentAppliedGE(OurCopyOfSpec);
	}
	

	// We still probably want to apply tags and stuff even if instant?
	// If bSuppressStackingCues is set for this GameplayEffect, only add the GameplayCue if this is the first instance of the GameplayEffect
	if (!bSuppressGameplayCues && bInvokeGameplayCueApplied && AppliedEffect && !AppliedEffect->bIsInhibited && 
		(!bFoundExistingStackableGE || !Spec.Def->bSuppressStackingCues))
	{
		// We both added and activated the GameplayCue here.
		// On the client, which will invoke the gameplay cue from an OnRep, it will need to look at the StartTime to determine
		// if the Cue was actually added+activated or just added (due to relevancy)

		// Fixme: what if we wanted to scale Cue magnitude based on damage? E.g, scale an cue effect when the GE is buffed?

		if (OurCopyOfSpec->StackCount > Spec.StackCount)
		{
			// Because PostReplicatedChange will get called from modifying the stack count
			// (and not PostReplicatedAdd) we won't know which GE was modified.
			// So instead we need to explicitly RPC the client so it knows the GC needs updating
			UAbilitySystemGlobals::Get().GetGameplayCueManager()->InvokeGameplayCueAddedAndWhileActive_FromSpec(this, *OurCopyOfSpec, PredictionKey);
		}
		else
		{
			// Otherwise these will get replicated to the client when the GE gets added to the replicated array
			InvokeGameplayCueEvent(*OurCopyOfSpec, EGameplayCueEvent::OnActive);
			InvokeGameplayCueEvent(*OurCopyOfSpec, EGameplayCueEvent::WhileActive);
		}
	}
	
	// Execute the GE at least once (if instant, this will execute once and be done. If persistent, it was added to ActiveGameplayEffects above)
	
	// Execute if this is an instant application effect
	if (bTreatAsInfiniteDuration)
	{
		// This is an instant application but we are treating it as an infinite duration for prediction. We should still predict the execute GameplayCUE.
		// (in non predictive case, this will happen inside ::ExecuteGameplayEffect)

		if (!bSuppressGameplayCues)
		{
			UAbilitySystemGlobals::Get().GetGameplayCueManager()->InvokeGameplayCueExecuted_FromSpec(this, *OurCopyOfSpec, PredictionKey);
		}
	}
	else if (Spec.Def->DurationPolicy == EGameplayEffectDurationType::Instant)
	{
		if (OurCopyOfSpec->Def->OngoingTagRequirements.IsEmpty())
		{
			ExecuteGameplayEffect(*OurCopyOfSpec, PredictionKey);
		}
		else
		{
			ABILITY_LOG(Warning, TEXT("%s is instant but has tag requirements. Tag requirements can only be used with gameplay effects that have a duration. This gameplay effect will be ignored."), *Spec.Def->GetPathName());
		}
	}

	if (Spec.GetPeriod() != UGameplayEffect::NO_PERIOD && Spec.TargetEffectSpecs.Num() > 0)
	{
		ABILITY_LOG(Warning, TEXT("%s is periodic but also applies GameplayEffects to its target. GameplayEffects will only be applied once, not every period."), *Spec.Def->GetPathName());
	}
	
	// evaluate if any active effects need to be removed by the application of this effect
	if (bIsNetAuthority)
	{
		ActiveGameplayEffects.AttemptRemoveActiveEffectsOnEffectApplication(*OurCopyOfSpec, MyHandle);
	}
	
	// ------------------------------------------------------
	// Apply Linked effects
	// todo: this is ignoring the returned handles, should we put them into a TArray and return all of the handles?
	// ------------------------------------------------------
	for (const FGameplayEffectSpecHandle& TargetSpec: Spec.TargetEffectSpecs)
	{
		if (TargetSpec.IsValid())
		{
			ApplyGameplayEffectSpecToSelf(*TargetSpec.Data.Get(), PredictionKey);
		}
	}

	UAbilitySystemComponent* InstigatorASC = Spec.GetContext().GetInstigatorAbilitySystemComponent();

	// Send ourselves a callback	
	OnGameplayEffectAppliedToSelf(InstigatorASC, *OurCopyOfSpec, MyHandle);

	// Send the instigator a callback
	if (InstigatorASC)
	{
		InstigatorASC->OnGameplayEffectAppliedToTarget(this, *OurCopyOfSpec, MyHandle);
	}

	return MyHandle;
}

void UAbilitySystemComponent::ExecuteGameplayEffect(FGameplayEffectSpec &Spec, FPredictionKey PredictionKey)
{
#if WITH_SERVER_CODE
	SCOPE_CYCLE_COUNTER(STAT_AbilitySystemComp_ExecuteGameplayEffect);
#endif

	// Should only ever execute effects that are instant application or periodic application
	// Effects with no period and that aren't instant application should never be executed
	check( (Spec.GetDuration() == UGameplayEffect::INSTANT_APPLICATION || Spec.GetPeriod() != UGameplayEffect::NO_PERIOD) );

	if (UE_LOG_ACTIVE(VLogAbilitySystem, Log))
	{
		ABILITY_VLOG(GetOwnerActor(), Log, TEXT("Executed %s"), *Spec.Def->GetFName().ToString());
		
		for (const FGameplayModifierInfo& Modifier : Spec.Def->Modifiers)
		{
			float Magnitude = 0.f;
			Modifier.ModifierMagnitude.AttemptCalculateMagnitude(Spec, Magnitude);
			ABILITY_VLOG(GetOwnerActor(), Log, TEXT("         %s: %s %f"), *Modifier.Attribute.GetName(), *EGameplayModOpToString(Modifier.ModifierOp), Magnitude);
		}
	}
	bIsNetDirty = true;

	ActiveGameplayEffects.ExecuteActiveEffectsFrom(Spec, PredictionKey);
}
```

* Engine\Plugins\Runtime\GameplayAbilities\Source\GameplayAbilities\Private\Abilities\GameplayAbility.cpp
```Cpp
void UGameplayAbility::CallActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, FOnGameplayAbilityEnded::FDelegate* OnGameplayAbilityEndedDelegate, const FGameplayEventData* TriggerEventData)
{
	PreActivate(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
	ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
}

void UGameplayAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
	if (bHasBlueprintActivate)
	{
		// A Blueprinted ActivateAbility function must call CommitAbility somewhere in its execution chain.
		K2_ActivateAbility();
	}
	else if (bHasBlueprintActivateFromEvent)
	{
		if (TriggerEventData)
		{
			// A Blueprinted ActivateAbility function must call CommitAbility somewhere in its execution chain.
			K2_ActivateAbilityFromEvent(*TriggerEventData);
		}
		else
		{
			UE_LOG(LogAbilitySystem, Warning, TEXT("Ability %s expects event data but none is being supplied. Use Activate Ability instead of Activate Ability From Event."), *GetName());
			bool bReplicateEndAbility = false;
			bool bWasCancelled = true;
			EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
		}
	}
	else
	{
		// Native child classes may want to override ActivateAbility and do something like this:

		// Do stuff...

		if (CommitAbility(Handle, ActorInfo, ActivationInfo))		// ..then commit the ability...
		{			
			//	Then do more stuff...
		}
	}
}

bool UGameplayAbility::CommitAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, OUT FGameplayTagContainer* OptionalRelevantTags)
{
	// Last chance to fail (maybe we no longer have resources to commit since we after we started this ability activation)
	if (!CommitCheck(Handle, ActorInfo, ActivationInfo, OptionalRelevantTags))
	{
		return false;
	}

	CommitExecute(Handle, ActorInfo, ActivationInfo);

	// Fixme: Should we always call this or only if it is implemented? A noop may not hurt but could be bad for perf (storing a HasBlueprintCommit per instance isn't good either)
	K2_CommitExecute();

	// Broadcast this commitment
	ActorInfo->AbilitySystemComponent->NotifyAbilityCommit(this);

	return true;
}

void UGameplayAbility::CommitExecute(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo)
{
	ApplyCooldown(Handle, ActorInfo, ActivationInfo);

	ApplyCost(Handle, ActorInfo, ActivationInfo);
}

void UGameplayAbility::ApplyCooldown(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo) const
{
	UGameplayEffect* CooldownGE = GetCooldownGameplayEffect();
	if (CooldownGE)
	{
		ApplyGameplayEffectToOwner(Handle, ActorInfo, ActivationInfo, CooldownGE, GetAbilityLevel(Handle, ActorInfo));
	}
}

void UGameplayAbility::ApplyCost(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo) const
{
	UGameplayEffect* CostGE = GetCostGameplayEffect();
	if (CostGE)
	{
		ApplyGameplayEffectToOwner(Handle, ActorInfo, ActivationInfo, CostGE, GetAbilityLevel(Handle, ActorInfo));
	}
}

FActiveGameplayEffectHandle UGameplayAbility::ApplyGameplayEffectToOwner(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const UGameplayEffect* GameplayEffect, float GameplayEffectLevel, int32 Stacks) const
{
	if (GameplayEffect && (HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo)))
	{
		FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(Handle, ActorInfo, ActivationInfo, GameplayEffect->GetClass(), GameplayEffectLevel);
		if (SpecHandle.IsValid())
		{
			SpecHandle.Data->StackCount = Stacks;
			return ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
		}
	}

	// We cannot apply GameplayEffects in this context. Return an empty handle.
	return FActiveGameplayEffectHandle();
}

FActiveGameplayEffectHandle UGameplayAbility::ApplyGameplayEffectSpecToOwner(const FGameplayAbilitySpecHandle AbilityHandle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEffectSpecHandle SpecHandle) const
{
	// This batches all created cues together
	FScopedGameplayCueSendContext GameplayCueSendContext;

	if (SpecHandle.IsValid() && (HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo)))
	{
		UAbilitySystemComponent* const AbilitySystemComponent = ActorInfo->AbilitySystemComponent.Get();
		return AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get(), AbilitySystemComponent->GetPredictionKeyForNewAction());

	}
	return FActiveGameplayEffectHandle();
}
```

* Engine\Plugins\Runtime\GameplayAbilities\Source\GameplayAbilities\Private\GameplayEffect.cpp
```Cpp
/** This is the main function that executes a GameplayEffect on Attributes and ActiveGameplayEffects */
void FActiveGameplayEffectsContainer::ExecuteActiveEffectsFrom(FGameplayEffectSpec &Spec, FPredictionKey PredictionKey)
{
#if WITH_SERVER_CODE
	SCOPE_CYCLE_COUNTER(STAT_ExecuteActiveEffectsFrom);
#endif

	if (!Owner)
	{
		return;
	}

	FGameplayEffectSpec& SpecToUse = Spec;

	// Capture our own tags.
	// TODO: We should only capture them if we need to. We may have snapshotted target tags (?) (in the case of dots with exotic setups?)

	SpecToUse.CapturedTargetTags.GetActorTags().Reset();
	Owner->GetOwnedGameplayTags(SpecToUse.CapturedTargetTags.GetActorTags());

	SpecToUse.CalculateModifierMagnitudes();

	// ------------------------------------------------------
	//	Modifiers
	//		These will modify the base value of attributes
	// ------------------------------------------------------
	
	bool ModifierSuccessfullyExecuted = false;

	for (int32 ModIdx = 0; ModIdx < SpecToUse.Modifiers.Num(); ++ModIdx)
	{
		const FGameplayModifierInfo& ModDef = SpecToUse.Def->Modifiers[ModIdx];
		
		FGameplayModifierEvaluatedData EvalData(ModDef.Attribute, ModDef.ModifierOp, SpecToUse.GetModifierMagnitude(ModIdx, true));
		ModifierSuccessfullyExecuted |= InternalExecuteMod(SpecToUse, EvalData);
	}

	// ------------------------------------------------------
	//	Executions
	//		This will run custom code to 'do stuff'
	// ------------------------------------------------------
	
	TArray< FGameplayEffectSpecHandle, TInlineAllocator<4> > ConditionalEffectSpecs;

	bool GameplayCuesWereManuallyHandled = false;

	for (const FGameplayEffectExecutionDefinition& CurExecDef : SpecToUse.Def->Executions)
	{
		bool bRunConditionalEffects = true; // Default to true if there is no CalculationClass specified.

		if (CurExecDef.CalculationClass)
		{
			const UGameplayEffectExecutionCalculation* ExecCDO = CurExecDef.CalculationClass->GetDefaultObject<UGameplayEffectExecutionCalculation>();
			check(ExecCDO);

			// Run the custom execution
			FGameplayEffectCustomExecutionParameters ExecutionParams(SpecToUse, CurExecDef.CalculationModifiers, Owner, CurExecDef.PassedInTags, PredictionKey);
			FGameplayEffectCustomExecutionOutput ExecutionOutput;
			ExecCDO->Execute(ExecutionParams, ExecutionOutput);

			bRunConditionalEffects = ExecutionOutput.ShouldTriggerConditionalGameplayEffects();

			// Execute any mods the custom execution yielded
			TArray<FGameplayModifierEvaluatedData>& OutModifiers = ExecutionOutput.GetOutputModifiersRef();

			const bool bApplyStackCountToEmittedMods = !ExecutionOutput.IsStackCountHandledManually();
			const int32 SpecStackCount = SpecToUse.StackCount;

			for (FGameplayModifierEvaluatedData& CurExecMod : OutModifiers)
			{
				// If the execution didn't manually handle the stack count, automatically apply it here
				if (bApplyStackCountToEmittedMods && SpecStackCount > 1)
				{
					CurExecMod.Magnitude = GameplayEffectUtilities::ComputeStackedModifierMagnitude(CurExecMod.Magnitude, SpecStackCount, CurExecMod.ModifierOp);
				}
				ModifierSuccessfullyExecuted |= InternalExecuteMod(SpecToUse, CurExecMod);
			}

			// If execution handled GameplayCues, we dont have to.
			if (ExecutionOutput.AreGameplayCuesHandledManually())
			{
				GameplayCuesWereManuallyHandled = true;
			}
		}

		if (bRunConditionalEffects)
		{
			// If successful, apply conditional specs
			for (const FConditionalGameplayEffect& ConditionalEffect : CurExecDef.ConditionalGameplayEffects)
			{
				if (ConditionalEffect.CanApply(SpecToUse.CapturedSourceTags.GetActorTags(), SpecToUse.GetLevel()))
				{
					FGameplayEffectSpecHandle SpecHandle = ConditionalEffect.CreateSpec(SpecToUse.GetEffectContext(), SpecToUse.GetLevel());
					if (SpecHandle.IsValid())
					{
						ConditionalEffectSpecs.Add(SpecHandle);
					}
				}
			}
		}
	}

	// ------------------------------------------------------
	//	Invoke GameplayCue events
	// ------------------------------------------------------
	
	// If there are no modifiers or we don't require modifier success to trigger, we apply the GameplayCue.
	const bool bHasModifiers = SpecToUse.Modifiers.Num() > 0;
	const bool bHasExecutions = SpecToUse.Def->Executions.Num() > 0;
	const bool bHasModifiersOrExecutions = bHasModifiers || bHasExecutions;

	// If there are no modifiers or we don't require modifier success to trigger, we apply the GameplayCue.
	bool InvokeGameplayCueExecute = (!bHasModifiersOrExecutions) || !Spec.Def->bRequireModifierSuccessToTriggerCues;

	if (bHasModifiersOrExecutions && ModifierSuccessfullyExecuted)
	{
		InvokeGameplayCueExecute = true;
	}

	// Don't trigger gameplay cues if one of the executions says it manually handled them
	if (GameplayCuesWereManuallyHandled)
	{
		InvokeGameplayCueExecute = false;
	}

	if (InvokeGameplayCueExecute && SpecToUse.Def->GameplayCues.Num())
	{
		// TODO: check replication policy. Right now we will replicate every execute via a multicast RPC

		ABILITY_LOG(Log, TEXT("Invoking Execute GameplayCue for %s"), *SpecToUse.ToSimpleString());

		UAbilitySystemGlobals::Get().GetGameplayCueManager()->InvokeGameplayCueExecuted_FromSpec(Owner, SpecToUse, PredictionKey);
	}

	// Apply any conditional linked effects
	for (const FGameplayEffectSpecHandle& TargetSpec : ConditionalEffectSpecs)
	{
		if (TargetSpec.IsValid())
		{
			Owner->ApplyGameplayEffectSpecToSelf(*TargetSpec.Data.Get(), PredictionKey);
		}
	}
}

bool FActiveGameplayEffectsContainer::InternalExecuteMod(FGameplayEffectSpec& Spec, FGameplayModifierEvaluatedData& ModEvalData)
{
	SCOPE_CYCLE_COUNTER(STAT_InternalExecuteMod);

	check(Owner);

	bool bExecuted = false;

	UAttributeSet* AttributeSet = nullptr;
	UClass* AttributeSetClass = ModEvalData.Attribute.GetAttributeSetClass();
	if (AttributeSetClass && AttributeSetClass->IsChildOf(UAttributeSet::StaticClass()))
	{
		AttributeSet = const_cast<UAttributeSet*>(Owner->GetAttributeSubobject(AttributeSetClass));
	}

	if (AttributeSet)
	{
		FGameplayEffectModCallbackData ExecuteData(Spec, ModEvalData, *Owner);

		/**
		 *  This should apply 'gamewide' rules. Such as clamping Health to MaxHealth or granting +3 health for every point of strength, etc
		 *	PreAttributeModify can return false to 'throw out' this modification.
		 */
		if (AttributeSet->PreGameplayEffectExecute(ExecuteData))
		{
			float OldValueOfProperty = Owner->GetNumericAttribute(ModEvalData.Attribute);
			ApplyModToAttribute(ModEvalData.Attribute, ModEvalData.ModifierOp, ModEvalData.Magnitude, &ExecuteData);

			FGameplayEffectModifiedAttribute* ModifiedAttribute = Spec.GetModifiedAttribute(ModEvalData.Attribute);
			if (!ModifiedAttribute)
			{
				// If we haven't already created a modified attribute holder, create it
				ModifiedAttribute = Spec.AddModifiedAttribute(ModEvalData.Attribute);
			}
			ModifiedAttribute->TotalMagnitude += ModEvalData.Magnitude;

			{
				SCOPE_CYCLE_COUNTER(STAT_PostGameplayEffectExecute);
				/** This should apply 'gamewide' rules. Such as clamping Health to MaxHealth or granting +3 health for every point of strength, etc */
				AttributeSet->PostGameplayEffectExecute(ExecuteData);
			}

#if ENABLE_VISUAL_LOG
			if (FVisualLogger::IsRecording())
			{
				DebugExecutedGameplayEffectData DebugData;
				DebugData.GameplayEffectName = Spec.Def->GetName();
				DebugData.ActivationState = "INSTANT";
				DebugData.Attribute = ModEvalData.Attribute;
				DebugData.Magnitude = Owner->GetNumericAttribute(ModEvalData.Attribute) - OldValueOfProperty;
				DebugExecutedGameplayEffects.Add(DebugData);
			}
#endif // ENABLE_VISUAL_LOG

			bExecuted = true;
		}
	}
	else
	{
		// Our owner doesn't have this attribute, so we can't do anything
		ABILITY_LOG(Log, TEXT("%s does not have attribute %s. Skipping modifier"), *Owner->GetPathName(), *ModEvalData.Attribute.GetName());
	}

	return bExecuted;
}
```

## Questions/Unfinished Part

* FPredictionKey

多人游戏/网络复制相关，详情要阅读GameplayPrediction头文件注释。

# Enhanced Input



# Lyra Input

Lyra的输入处理是基于Enhanced Input的

## Key Class

* UGameFeatureAction_AddInputConfig

> Registers a Player Mappable Input config to the Game User Settings
> Expects that local players are set up to use the EnhancedInput system.

继承UGameFeatureAction_WorldActionBase，将玩家输入映射配置注册进Game User Settings。

* UGameFeatureAction_AddInputBinding

> Adds InputMappingContext to local players' EnhancedInput system. 
> Expects that local players are set up to use the EnhancedInput system.

继承UGameFeatureAction_WorldActionBase，将InputMappingContext加入到本地玩家增强输入系统中。

* ULyraAssetManager

> Game implementation of the asset manager that override functionality and store game-specific types.
> It is expected that most game will want to override AssetManager as it provides a good place for game-specific loading logic.
> This class is used by setting 'AssetManagerClassName' in DefaultEngine.ini

继承UAssetManager，Lyra的资产管理类，扩展资产管理的功能，并且可以存储一些特定游戏类型的资源。

* ULyraHeroComponent

> Component that sets up input and camera handling for player controlled pawns (or bots that simulate players).
> This depends on a PawnExtensionComponent to coordinate initialization.

继承UPawnComponent和IGameFrameworkInitStateInterface，设置和初始化玩家角色输入（或者模拟玩家的AI输入）、控制摄像机的角色组件。与PawnExtensionComponent角色组件进行协同初始化。

玩家输入初始化的关键函数`InitializaPlayerInput(UInputComponent* PlayerInputComponent)`。

* ULyraInputConfig



* ULyraInputComponent

* ULyraGameplayTags

* ULyraPawnData

* ULyraPawnExtension

* ULyraSettingsLocal

## Flow Diagram

由于输入配置存在PawnData中，而PawnData又是一种重要的输入模块资产，会被GameFeature加载。所以先理清楚PawnData的来源

### PawnData Data Flow

{% asset_img "Unreal Engine 5 Lyra Starter Game Pawn Data Flow.png" Lyra Pawn Data Flow %}

### Key Implementation

* LyraHeroComponent.cpp: virtual void InitializaPlayerInput(UInputComponent* PlayerInputComponent)
```Cpp
if (const ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
{
	// 从PawnExetension组件获取PawnData
	if (const ULyraPawnData* PawnData = PawnExtComp->GetPawnData<ULyraPawnData>())
	{
		// 在PawnData中获取应该激活的输入配置
		if (const ULyraInputConfig* InputConfig = PawnData->InputConfig)
		{
			// Register any default input configs with the settings so that they will be applied to the player during AddInputMappings
			for (const FMappableConfigPair& Pair : DefaultInputConfigs)
			{
				if (Pair.bShouldActivateAutomatically && Pair.CanBeActivated())
				{
					FModifyContextOptions Options = {};
					Options.bIgnoreAllPressedKeysUntilRelease = false;
					// 将获取到的输入映射加入EnhancedInput实例中进行注册
					// Actually add the config to the local player							
					Subsystem->AddPlayerMappableConfig(Pair.Config.LoadSynchronous(), Options);	
				}
			}

			// The Lyra Input Component has some additional functions to map Gameplay Tags to an Input Action.
			// If you want this functionality but still want to change your input component class, make it a subclass
			// of the ULyraInputComponent or modify this component accordingly.
			ULyraInputComponent* LyraIC = Cast<ULyraInputComponent>(PlayerInputComponent);
			if (ensureMsgf(LyraIC, TEXT("Unexpected Input Component class! The Gameplay Abilities will not be bound to their inputs. Change the input component to ULyraInputComponent or a subclass of it.")))
			{
				// Add the key mappings that may have been set by the player
				// 输入配置
				LyraIC->AddInputMappings(InputConfig, Subsystem);

				// This is where we actually bind and input action to a gameplay tag, which means that Gameplay Ability Blueprints will
				// be triggered directly by these input actions Triggered events. 
				TArray<uint32> BindHandles;
				
				// 
				LyraIC->BindAbilityActions(InputConfig, this, &ThisClass::Input_AbilityInputTagPressed, &ThisClass::Input_AbilityInputTagReleased, /*out*/ BindHandles);

				// 将输入配置、标签、输入触发事件、具体逻辑进行绑定
				LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Move, ETriggerEvent::Triggered, this, &ThisClass::Input_Move, /*bLogIfNotFound=*/ false);
				LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Look_Mouse, ETriggerEvent::Triggered, this, &ThisClass::Input_LookMouse, /*bLogIfNotFound=*/ false);
				LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Look_Stick, ETriggerEvent::Triggered, this, &ThisClass::Input_LookStick, /*bLogIfNotFound=*/ false);
				LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Crouch, ETriggerEvent::Triggered, this, &ThisClass::Input_Crouch, /*bLogIfNotFound=*/ false);
				LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_AutoRun, ETriggerEvent::Triggered, this, &ThisClass::Input_AutoRun, /*bLogIfNotFound=*/ false);
			}
		}
	}
}
```

### Class Diagram

{% asset_img "Unreal Engine 5 Lyra Starter Game Input Module Class Diagram.png" Lyra Input Class Diagram %}

### Flow Diagram

{% asset_img "Unreal Engine 5 Lyra Starter Game Input Module Work Flow.png" Lyra Input Work Flow %}

## Note

不使用除Enhanced Input之外的插件时，需要关注代码：

```Cpp
return ULyraAssetManager::Get().GetDefaultPawnData();

return GetAsset(DefaultPawnData);

UPROPERTY(Config)
TSoftObjectPtr<ULyraPawnData> DefaultPawnData;
```

可在Character蓝图类中指定默认值。

# Lyra Animation

## Required Plugin

* Animation Locomotion Library

> Collection of techniques for driving locomotion animations.

运动动画后期处理技术的代码库。

* Animation Warping

> Framework for animation and pose warping. This plugin includes Stride, Orientation, and Slope Warping alongside the Root Motion Delta animation attribute.

姿势扭曲的动画框架，可提供步幅扭曲、方向扭曲、根运动Delta动画属性等功能。

## Solution Design

ABP_Mannequin_Base->ALI_ItemAnimLayers->ABP_ItemAnimLayersBase->ABP_<定义了要使用的具体动画的蓝图>

ABP_Mannequine_Base负责角色动画状态机、上下半身动画分离处理再合并、线程安全更新动画所需要的数据、一部分后期处理。

ALI_ItemAnimLayers是纯接口层。

ABP_ItemAnimLayersBase负责实现具体的接口和大多数动画后期处理，例如瞄准偏移(Aim Offset)、步幅扭曲(Stride Warping)、距离匹配(Distance Match)等。

可使用Unreal Engine中的Reference View了解大致的

## Notice

### BlueprintThreadSafeUpdateAnimation

?线程安全更新动画数据，将这部分工作交给工作线程而不是游戏线程以提升性能，但是可能在C++中的NativeUpdateAnimation中进行该部分工作可能可以获得更好的性能?

### SelectCardinalDirectionFromAngle

从字面意义和函数具体实现来看，是从得到的角色面向方向与速度的夹角推算出角色朝向的主要方向(Cardinal Direction)，主要方向有四个：前方(Forward)、后方(Backward)、左方(Left)、右方(Right)。

比如人物面朝前方往右偏前方移动，那么该函数的结果，主要方向为前方(Forward)。

?DeadZone意义位置，推测为摄像机的视野盲区，可能跟FOV有关系?

其他获取到的参数值是为了得到角色的面向方向与速度方向的夹角，根据四个主要方向的定义范围，判断该夹角的值是否位于某个方向的范围内，最后返回。

?八向移动?

### Root Motion

不启用根运动时，如果动画有运动数据，那么骨骼网格和角色会分离，动画播放结束以后骨骼网格会回到原点。启用以后骨骼会驱动角色动作，随着角色驱动动作。

用Miaximo的前进动画时，因为本身带有运动数据，然后根运动应该是默认关闭的，出现角色循环前进一段距离然后回到原点的情况。解决的办法应该是把胶囊体移速设置为0然后开启跟运动，但是估计效果仍然不好。

### Stride Warping (步幅扭曲)

动态调整角色的动画步幅来匹配胶囊体的移动速度。与距离匹配函数库配合来减少滑步的负面视觉影响。

在蓝图中，还提供了贴墙情况下的处理实现(UpdateCycleAnim)。

* Engine\Plugins\Animation\AnimationWarping\Source\Runtime\Private\BoneControllers\AnimNode_StrideWarping.cpp
```Cpp
void FAnimNode_StrideWarping::EvaluateSkeletalControl_AnyThread(FComponentSpacePostContext& Output, Tarray<FboneTransform>& OutBoneTransforms)
{
	SCOPE_CYCLE_COUNTER(STAT_StrideWarping_Eval);
	check(OutBoneTransforms.IsEmpty());

	const FVector PreviousStrideDirection = ActualStrideDirection;
	ActualStrideDirection = StrideDirection;
	ActualStrideScale = StrideScale;

	bool bGraphDrivenWarping = false;
	const UE::Anim::IAnimRootMotionProvider* RootMotionProvider = UE::Anim::IAnimRootMotionProvider::Get();

	if (Mode == EWarpingEvaluationMode::Graph)
	{
		bGraphDrivenWarping = !!RootMotionProvider;
		ensureMsgf(bGraphDrivenWarping, TEXT("Graph driven Stride Warping expected a valid root motion delta provider interface."));
	}

	FTransform RootMotionTransformDelta = FTransform::Identity;

	if (bGraphDrivenWarping)
	{
		// Graph driven stride warping will override the manual stride direction with the intent of the current animation sub-graph's accumulated root motion
		bGraphDrivenWarping = RootMotionProvider->ExtractRootMotion(Output.CustomAttributes, RootMotionTransformDelta);
		if (bGraphDrivenWarping)
		{
			CachedRootMotionDeltaTranslation = RootMotionTransformDelta.GetTranslation();
			// If there's no root motion delta, keep the previous stride direction
			ActualStrideDirection = CachedRootMotionDeltaTranslation.GetSafeNormal(UE_SMALL_NUMBER, PreviousStrideDirection);
		}
		else
		{
			// Early exit on missing root motion delta attribute
			return;
		}
	}

	const FBoneContainer& RequiredBones = Output.Pose.GetPose().GetBoneContainer();
	const FTransform IKFootRootTransform = Output.Pose.GetComponentSpaceTransform(IKFootRootBone.GetCompactPoseIndex(RequiredBones));
	const FVector ResolvedFloorNormal = FloorNormalDirection.AsComponentSpaceDirection(AnimInstanceProxy, IKFootRootTransform);
	const FVector ResolvedGravityDirection = GravityDirection.AsComponentSpaceDirection(AnimInstanceProxy, IKFootRootTransform);

	if (bOrientStrideDirectionUsingFloorNormal)
	{
		const FVector StrideWarpingAxis = ResolvedFloorNormal ^ ActualStrideDirection;
		ActualStrideDirection = StrideWarpingAxis ^ ResolvedFloorNormal;
	}
	// Get all foot IK Transforms
	for (auto& Foot : FootData)
	{
		Foot.IKFootBoneTransform = Output.Pose.GetComponentSpaceTransform(Foot.IKFootBoneIndex);
	}

	if (bGraphDrivenWarping)
	{
		if (CachedRootMotionDeltaSpeed <= MinRootMotionSpeedThreshold)
		{
			// If root motion speed is under the threshold, snap back to no stride adjustment.
			// If interpolation is on, it will blend the result
			ActualStrideScale = 1.0f;
		}
		else
		{
			// Graph driven stride scale factor will be determined by the ratio of the
			// locomotion (capsule/physics) speed against the animation root motion speed
			ActualStrideScale = LocomotionSpeed / CachedRootMotionDeltaSpeed;
		}
	}

	// Allow the opportunity for stride scale clamping and interpolation regardless of evaluation mode
	ActualStrideScale = StrideScaleModifierState.ApplyTo(StrideScaleModifier, ActualStrideScale, CachedDeltaTime);

	if (bGraphDrivenWarping)
	{
		// Forward the side effects of stride warping on the root motion contribution for this sub-graph
		RootMotionTransformDelta.ScaleTranslation(ActualStrideScale);
		const bool bRootMotionOverridden = RootMotionProvider->OverrideRootMotion(RootMotionTransformDelta, Output.CustomAttributes);
		ensureMsgf(bRootMotionOverridden, TEXT("Graph driven Stride Warping expected a root motion delta to be present in the attribute stream prior to warping/overriding it."));
	}

	// Scale IK feet bones along Stride Warping Axis, from the Thigh bone location.
	for (auto& Foot : FootData)
	{
		// Stride Warping along Stride Warping Axis
		const FVector IKFootLocation = Foot.IKFootBoneTransform.GetLocation();
		const FVector ThighBoneLocation = Output.Pose.GetComponentSpaceTransform(Foot.ThighBoneIndex).GetLocation();

		// Project Thigh Bone Location on plane made of FootIKLocation and FloorPlaneNormal, along Gravity Dir.
		// This will be the StrideWarpingPlaneOrigin
		const FVector StrideWarpingPlaneOrigin = (FMath::Abs(ResolvedGravityDirection | ResolvedFloorNormal) > DELTA) ? FMath::LinePlaneIntersection(ThighBoneLocation, ThighBoneLocation + ResolvedGravityDirection, IKFootLocation, ResolvedFloorNormal) : IKFootLocation;

		// Project FK Foot along StrideWarping Plane, this will be our Scale Origin
		const FVector ScaleOrigin = FVector::PointPlaneProject(IKFootLocation, StrideWarpingPlaneOrigin, ActualStrideDirection);

		// Now the ScaleOrigin and IKFootLocation are forming a line parallel to the floor, and we can scale the IK foot.
		const FVector WarpedLocation = ScaleOrigin + (IKFootLocation - ScaleOrigin) * ActualStrideScale;
		Foot.IKFootBoneTransform.SetLocation(WarpedLocation);
	}

	FVector PelvisOffset = FVector::ZeroVector;
	const FCompactPoseBoneIndex PelvisBoneIndex = PelvisBone.GetCompactPoseIndex(RequiredBones);

	FTransform PelvisTransform = Output.Pose.GetComponentSpaceTransform(PelvisBoneIndex);
	const FVector InitialPelvisLocation = PelvisTransform.GetLocation();

	TArray<float, TInlineAllocator<10>> FKFootDistancesToPelvis;
	FKFootDistancesToPelvis.Reserve(FootData.Num());

	TArray<FVector, TInlineAllocator<10>> IKFootLocations;
	IKFootLocations.Reserve(FootData.Num());
	
	for (auto& Foot : FootData)
	{
		const FVector FKFootLocation = Output.Pose.GetComponentSpaceTransform(Foot.FKFootBoneIndex).GetLocation();
		FKFootDistancesToPelvis.Add(FVector::Dist(FKFootLocation, InitialPelvisLocation));

		const FVector IKFootLocation = Foot.IKFootBoneTransform.GetLocation();
		IKFootLocations.Add(IKFootLocation);
	}

	// Adjust Pelvis down if needed to keep foot contact with the ground and prevent over-extension
	PelvisTransform = PelvisIKFootSolver.Solve(PelvisTransform, FKFootDistancesToPelvis, IKFootLocations, CachedDeltaTime);

	// Add adjusted pelvis transform
	check(!PelvisTransform.ContainsNaN());
	OutBoneTransforms.Add(FBoneTransform(PelvisBoneIndex, PelvisTransform));

	// Compute final offset to use below
	PelvisOffset = (PelvisTransform.GetLocation() - InitialPelvisLocation);

	// Rotate Thigh bones to help IK, and maintain leg shape.
	if (bCompensateIKUsingFKThighRotation)
	{
		for (auto& Foot : FootData)
		{
			const FTransform ThighTransform = Output.Pose.GetComponentSpaceTransform(Foot.ThighBoneIndex);
			const FTransform FKFootTransform = Output.Pose.GetComponentSpaceTransform(Foot.FKFootBoneIndex);
			
			FTransform AdjustedThighTransform = ThighTransform;
			AdjustedThighTransform.AddToTranslation(PelvisOffset);

			const FVector InitialDir = (FKFootTransform.GetLocation() - ThighTransform.GetLocation()).GetSafeNormal();
			const FVector TargetDir = (Foot.IKFootBoneTransform.GetLocation() - AdjustedThighTransform.GetLocation()).GetSafeNormal();
			
			// Find Delta Rotation take takes us from Old to New dir
			const FQuat DeltaRotation = FQuat::FindBetweenNormals(InitialDir, TargetDir);

			// Rotate our Joint quaternion by this delta rotation
			AdjustedThighTransform.SetRotation(DeltaRotation * AdjustedThighTransform.GetRotation());

			// Add adjusted thigh transform
			check(!AdjustedThighTransform.ContainsNaN());
			OutBoneTransforms.Add(FBoneTransform(Foot.ThighBoneIndex, AdjustedThighTransform));

			// Clamp IK Feet bone based on FK leg. To prevent over-extension and preserve animated motion.
			if (bClampIKUsingFKLimits)
			{
				const float FKLength = FVector::Dist(FKFootTransform.GetLocation(), ThighTransform.GetLocation());
				const float IKLength = FVector::Dist(Foot.IKFootBoneTransform.GetLocation(), AdjustedThighTransform.GetLocation());
				if (IKLength > FKLength)
				{
					const FVector ClampedFootLocation = AdjustedThighTransform.GetLocation() + TargetDir * FKLength;
					Foot.IKFootBoneTransform.SetLocation(ClampedFootLocation);
				}
			}
		}
	}

	// Add final IK feet transforms
	for (auto& Foot : FootData)
	{
		check(!Foot.IKFootBoneTransform.ContainsNaN());
		OutBoneTransforms.Add(FBoneTransform(Foot.IKFootBoneIndex, Foot.IKFootBoneTransform));
	}

	// Sort OutBoneTransforms so indices are in increasing order.
	OutBoneTransforms.Sort(FCompareBoneTransformIndex());
}
```

### Orientation Warping (方向扭曲)

隔离并扭曲动画姿势的腿部IK骨骼，契合根骨骼运动的动态更新移动方向。
用于填充动画序列中的覆盖缺口，减少手动创建间隙动画或创建过多混合空间过度的必要性。
上下半身分离、锁定瞄准、八向移动会有比较直观的效果。

* Engine\Plugins\Animation\AnimationWarping\Source\Runtime\Public\BoneControllers\AnimNode_OrientationWarping.cpp
```Cpp
void FAnimNode_OrientationWarping::EvaluateSkeletalControl_AnyThread(FComponentSpacePoseContext& Output, TArray<FBoneTransform>& OutBoneTransforms)
{
	SCOPE_CYCLE_COUNTER(STAT_OrientationWarping_Eval);
	check(OutBoneTransforms.Num() == 0);

	ActualOrientationAngle = OrientationAngle;

	const float DeltaSeconds = Output.AnimInstanceProxy->GetDeltaSeconds();
	const FVector RotationAxisVector = UE::Anim::GetAxisVector(RotationAxis);
	FVector RootMotionDeltaDirection = FVector::ZeroVector;
	FVector LocomotionForward = FVector::ZeroVector;

	bool bGraphDrivenWarping = false;
	const UE::Anim::IAnimRootMotionProvider* RootMotionProvider = UE::Anim::IAnimRootMotionProvider::Get();

	if (Mode == EWarpingEvaluationMode::Graph)
	{
		bGraphDrivenWarping = !!RootMotionProvider;
		ensureMsgf(bGraphDrivenWarping, TEXT("Graph driven Orientation Warping expected a valid root motion delta provider interface."));
	}

#if WITH_EDITORONLY_DATA
	bFoundRootMotionAttribute = false;
#endif

	// We will likely need to revisit LocomotionAngle participating as an input to orientation warping.
	// Without velocity information from the motion model (such as the capsule), LocomotionAngle isn't enough
	// information in isolation for all cases when deciding to warp.
	//
	// For example imagine that the motion model has stopped moving with zero velocity due to a
	// transition into a strafing stop. During that transition we may play an animation with non-zero 
	// velocity for an arbitrary number of frames. In this scenario the concept of direction is meaningless 
	// since we cannot orient the animation to match a zero velocity and consequently a zero direction, 
	// since that would break the pose. For those frames, we would incorrectly over-orient the strafe.
	//
	// The solution may be instead to pass velocity with the actor base rotation, allowing us to retain
	// speed information about the motion. It may also allow us to do more complex orienting behavior 
	// when multiple degrees of freedom can be considered.

	if (bGraphDrivenWarping)
	{
		FTransform RootMotionTransformDelta = FTransform::Identity;
		bGraphDrivenWarping = RootMotionProvider->ExtractRootMotion(Output.CustomAttributes, RootMotionTransformDelta);

		// Graph driven orientation warping will modify the incoming root motion to orient towards the intended locomotion angle
		if (bGraphDrivenWarping)
		{
#if WITH_EDITORONLY_DATA
			// Graph driven Orientation Warping expects a root motion delta to be present in the attribute stream.
			bFoundRootMotionAttribute = true;
#endif

			// In UE, forward is defined as +x; consequently this is also true when sampling an actor's velocity. Historically the skeletal 
			// mesh component forward will not match the actor, requiring us to correct the rotation before sampling the LocomotionForward.
			// In order to make orientation warping 'pure' in the future we will need to provide more context about the intent of
			// the actor vs the intent of the animation in their respective spaces. Specifically, we will need some form the following information:
			//
			// 1. Actor Forward
			// 2. Actor Velocity
			// 3. Skeletal Mesh Relative Rotation

			LocomotionAngle = FRotator::NormalizeAxis(LocomotionAngle);
			LocomotionAngle = FMath::DegreesToRadians(LocomotionAngle);
			const FQuat LocomotionRotation = FQuat(RotationAxisVector, LocomotionAngle);

			const FTransform SkeletalMeshRelativeTransform = Output.AnimInstanceProxy->GetComponentRelativeTransform();
			const FQuat SkeletalMeshRelativeRotation = SkeletalMeshRelativeTransform.GetRotation();
			LocomotionForward = SkeletalMeshRelativeRotation.UnrotateVector(LocomotionRotation.GetForwardVector()).GetSafeNormal();
			
			if (OffsetAlpha == 0.0)
			{
				// No alpha. Reset our heading offset.
				HeadingOffset = 0.0f;
			}
			else
			{
				// Accumulate a percentage of our component's frame delta, based on offset alpha
				const float LastComponentHeading = ComponentHeading;
				ComponentHeading = Output.AnimInstanceProxy->GetComponentTransform().GetRotation().GetTwistAngle(RotationAxisVector);
				HeadingOffset += FMath::UnwindRadians(LastComponentHeading - ComponentHeading) * OffsetAlpha;

				// Accumulate a percentage of our root motion's heading, based on offset alpha
				float RootMotionDeltaHeading = RootMotionTransformDelta.GetRotation().GetTwistAngle(RotationAxisVector);
				HeadingOffset += RootMotionDeltaHeading * OffsetAlpha;

				// If our alpha is decreasing, we're blending out the offset.
				// We don't need to do this when blending in because we're just accumulating partial component offset and root motion.
				const float OffsetAlphaStep = LastOffsetAlpha - OffsetAlpha;
				if (OffsetAlphaStep > 0.0f)
				{
					HeadingOffset -= OffsetAlphaStep * HeadingOffset / LastOffsetAlpha;
				}

				const float MaxOffsetRadians = FMath::DegreesToRadians(MaxOffsetAngle);
				HeadingOffset = FMath::Clamp(HeadingOffset, -MaxOffsetAngle, MaxOffsetAngle);
			}
			LastOffsetAlpha = OffsetAlpha;

			// Rotate the root motion direction by our effective offset.
			// This means we will warp our pose based on the adjusted offset, and correct accordingly.
			FQuat RootOffsetRotation = FQuat(RotationAxisVector, HeadingOffset);
			const FVector RootMotionDeltaTranslation = RootOffsetRotation.RotateVector(RootMotionTransformDelta.GetTranslation());

			// Hold previous direction if we can't calculate it from current move delta, because the root is no longer moving
			RootMotionDeltaDirection = RootMotionDeltaTranslation.GetSafeNormal(UE_SMALL_NUMBER, PreviousRootMotionDeltaDirection);

			const float RootMotionDeltaSpeed = RootMotionDeltaTranslation.Size() / DeltaSeconds;
			if (RootMotionDeltaSpeed < MinRootMotionSpeedThreshold)
			{
				// If we're under the threshold, snap orientation angle to 0, and let interpolation handle the delta
				ActualOrientationAngle = 0.0f;
				PreviousRootMotionDeltaDirection = RootMotionDeltaDirection;
			}
			else
			{
				// Capture the delta rotation from the axis of motion we care about
				FQuat WarpedRotation = FQuat::FindBetween(RootMotionDeltaDirection, LocomotionForward);

				// For interpolated warping, guarantee that PreviousOrientationAngle is relative to the current frame's root motion direction 
				float RootMotionDeltaAngleDifference = FMath::Acos(RootMotionDeltaDirection.Dot(PreviousRootMotionDeltaDirection));
				RootMotionDeltaAngleDifference *= FMath::Sign(RotationAxisVector.Dot(RootMotionDeltaDirection.Cross(PreviousRootMotionDeltaDirection)));

				PreviousRootMotionDeltaDirection = RootMotionDeltaDirection;
				PreviousOrientationAngle += RootMotionDeltaAngleDifference;

				ActualOrientationAngle = WarpedRotation.GetTwistAngle(RotationAxisVector);
				// Motion Matching may return an animation that deviates a lot from the movement direction (e.g movement direction going bwd and motion matching could return the fwd animation for a few frames)
				// When that happens, since we use the delta between root motion and movement direction, we would be over-rotating the lower body and breaking the pose during those frames
				// So, when that happens we use the inverse of the movement direction to calculate our target rotation. 
				// This feels a bit 'hacky' but its the only option I've found so far to mitigate the problem
				if (LocomotionAngleDeltaThreshold > 0.f)
				{
					if (FMath::Abs(FMath::RadiansToDegrees(ActualOrientationAngle)) > LocomotionAngleDeltaThreshold)
					{
						WarpedRotation = FQuat::FindBetween(RootMotionDeltaDirection, -LocomotionForward);
						ActualOrientationAngle = WarpedRotation.GetTwistAngle(RotationAxisVector);
					}
					
					if (FMath::Abs(FMath::RadiansToDegrees(PreviousOrientationAngle)) > LocomotionAngleDeltaThreshold)
					{
						// Previous orientation angle might be using an opposite direction too, so flip it if it exceeds the threshold as well.
						PreviousOrientationAngle = WarpedRotation.GetTwistAngle(RotationAxisVector);
					}
				}

				// Rotate the root motion delta fully by the warped angle
				const FVector WarpedRootMotionTranslationDelta = WarpedRotation.RotateVector(RootMotionDeltaTranslation);
				RootMotionTransformDelta.SetTranslation(WarpedRootMotionTranslationDelta);
			}



			// Forward the side effects of orientation warping on the root motion contribution for this sub-graph
			const bool bRootMotionOverridden = RootMotionProvider->OverrideRootMotion(RootMotionTransformDelta, Output.CustomAttributes);
			ensureMsgf(bRootMotionOverridden, TEXT("Graph driven Orientation Warping expected a root motion delta to be present in the attribute stream prior to warping/overriding it."));
		}
		else
		{
			// Early exit on missing root motion delta attribute
			return;
		}
	} 
	else
	{
		// Manual orientation warping will take the angle directly
		ActualOrientationAngle = FRotator::NormalizeAxis(ActualOrientationAngle);
		ActualOrientationAngle = FMath::DegreesToRadians(ActualOrientationAngle);
	}

	// Optionally interpolate the effective orientation towards the target orientation angle
	if (RotationInterpSpeed > 0.f)
	{
		ActualOrientationAngle = FMath::FInterpTo(PreviousOrientationAngle, ActualOrientationAngle, DeltaSeconds, RotationInterpSpeed);
		PreviousOrientationAngle = ActualOrientationAngle;
	}

	// Allow the alpha value of the node to affect the final rotation
	ActualOrientationAngle *= ActualAlpha * WarpingAlpha;

#if ENABLE_ANIM_DEBUG
	bool bDebugging = false;
#if WITH_EDITORONLY_DATA
	bDebugging = bDebugging || bEnableDebugDraw;
#else
	constexpr float DebugDrawScale = 1.f;
#endif
	const int32 DebugIndex = CVarAnimNodeOrientationWarpingDebug.GetValueOnAnyThread();
	bDebugging = bDebugging || (DebugIndex > 0);

	if (bDebugging)
	{
		const FTransform ComponentTransform = Output.AnimInstanceProxy->GetComponentTransform();
		const FVector ActorForwardDirection = Output.AnimInstanceProxy->GetActorTransform().GetRotation().GetForwardVector();
		FVector DebugArrowOffset = FVector::ZAxisVector * DebugDrawScale;

		const bool bDrawAll = DebugIndex == 3;
		const bool bDrawOffset = (DebugIndex == 2) || bDrawAll;
		const bool bDrawWarping = (DebugIndex == 1) || bDrawAll;
		if (bDrawWarping)
		{
			const FVector ForwardDirection = bGraphDrivenWarping
				? ComponentTransform.GetRotation().RotateVector(LocomotionForward)
				: ActorForwardDirection;

			Output.AnimInstanceProxy->AnimDrawDebugDirectionalArrow(
				ComponentTransform.GetLocation() + DebugArrowOffset,
				ComponentTransform.GetLocation() + DebugArrowOffset + ForwardDirection * 100.f * DebugDrawScale,
				40.f * DebugDrawScale, FColor::Red, false, 0.f, 2.f * DebugDrawScale);

			const FVector RotationDirection = bGraphDrivenWarping
				? ComponentTransform.GetRotation().RotateVector(RootMotionDeltaDirection)
				: ActorForwardDirection.RotateAngleAxis(OrientationAngle, RotationAxisVector);

			DebugArrowOffset += FVector::ZAxisVector * DebugDrawScale;
			Output.AnimInstanceProxy->AnimDrawDebugDirectionalArrow(
				ComponentTransform.GetLocation() + DebugArrowOffset,
				ComponentTransform.GetLocation() + DebugArrowOffset + RotationDirection * 100.f * DebugDrawScale,
				40.f * DebugDrawScale, FColor::Blue, false, 0.f, 2.f * DebugDrawScale);

			const float ActualOrientationAngleDegrees = FMath::RadiansToDegrees(ActualOrientationAngle);
			const FVector WarpedRotationDirection = bGraphDrivenWarping
				? RotationDirection.RotateAngleAxis(ActualOrientationAngleDegrees, RotationAxisVector)
				: ActorForwardDirection.RotateAngleAxis(ActualOrientationAngleDegrees, RotationAxisVector);

			DebugArrowOffset += FVector::ZAxisVector * DebugDrawScale;
			Output.AnimInstanceProxy->AnimDrawDebugDirectionalArrow(
				ComponentTransform.GetLocation() + DebugArrowOffset,
				ComponentTransform.GetLocation() + DebugArrowOffset + WarpedRotationDirection * 100.f * DebugDrawScale,
				40.f * DebugDrawScale, FColor::Green, false, 0.f, 2.f * DebugDrawScale);
			DebugArrowOffset += FVector::ZAxisVector * DebugDrawScale;
		}
		
		if (bDrawOffset)
		{
			const float HeadingOffetDegrees = FMath::RadiansToDegrees(HeadingOffset);
			const FVector OffsetDirection = ActorForwardDirection.RotateAngleAxis(HeadingOffetDegrees, RotationAxisVector);

			Output.AnimInstanceProxy->AnimDrawDebugDirectionalArrow(
				ComponentTransform.GetLocation() + DebugArrowOffset,
				ComponentTransform.GetLocation() + DebugArrowOffset + OffsetDirection * 100.f * DebugDrawScale,
				40.f * DebugDrawScale, FColor::Purple, false, 0.f, 2.f * DebugDrawScale);

			DebugArrowOffset += FVector::ZAxisVector * DebugDrawScale;
			Output.AnimInstanceProxy->AnimDrawDebugDirectionalArrow(
				ComponentTransform.GetLocation() + DebugArrowOffset,
				ComponentTransform.GetLocation() + DebugArrowOffset + ActorForwardDirection * 100.f * DebugDrawScale,
				40.f * DebugDrawScale, FColor::Black, false, 0.f, 2.f * DebugDrawScale);
		}
	}
#endif

	// Combine our warping and heading offsets to rotate our root bone
	const float CombinedRootOffset = FMath::UnwindRadians(ActualOrientationAngle * DistributedBoneOrientationAlpha + HeadingOffset);

	// Rotate Root Bone first, as that cheaply rotates the whole pose with one transformation.
	if (!FMath::IsNearlyZero(CombinedRootOffset, KINDA_SMALL_NUMBER))
	{
		const FQuat RootRotation = FQuat(RotationAxisVector, CombinedRootOffset);
		const FCompactPoseBoneIndex RootBoneIndex(0);

		FTransform RootBoneTransform(Output.Pose.GetComponentSpaceTransform(RootBoneIndex));
		RootBoneTransform.SetRotation(RootRotation * RootBoneTransform.GetRotation());
		RootBoneTransform.NormalizeRotation();
		Output.Pose.SetComponentSpaceTransform(RootBoneIndex, RootBoneTransform);
	}

	const int32 NumSpineBones = SpineBoneDataArray.Num();
	const bool bSpineOrientationAlpha = !FMath::IsNearlyZero(DistributedBoneOrientationAlpha, KINDA_SMALL_NUMBER);
	const bool bUpdateSpineBones = (NumSpineBones > 0) && bSpineOrientationAlpha;

	if (bUpdateSpineBones)
	{
		// Spine bones counter rotate body orientation evenly across all bones.
		for (int32 ArrayIndex = 0; ArrayIndex < NumSpineBones; ArrayIndex++)
		{
			const FOrientationWarpingSpineBoneData& BoneData = SpineBoneDataArray[ArrayIndex];
			const FQuat SpineBoneCounterRotation = FQuat(RotationAxisVector, -ActualOrientationAngle * DistributedBoneOrientationAlpha * BoneData.Weight);
			check(BoneData.Weight > 0.f);

			FTransform SpineBoneTransform(Output.Pose.GetComponentSpaceTransform(BoneData.BoneIndex));
			SpineBoneTransform.SetRotation((SpineBoneCounterRotation * SpineBoneTransform.GetRotation()));
			SpineBoneTransform.NormalizeRotation();
			Output.Pose.SetComponentSpaceTransform(BoneData.BoneIndex, SpineBoneTransform);
		}
	}

	const float IKFootRootOrientationAlpha = 1.f - DistributedBoneOrientationAlpha;
	const bool bUpdateIKFootRoot = (IKFootData.IKFootRootBoneIndex != FCompactPoseBoneIndex(INDEX_NONE)) && !FMath::IsNearlyZero(IKFootRootOrientationAlpha, KINDA_SMALL_NUMBER);

	// Rotate IK Foot Root
	if (bUpdateIKFootRoot)
	{
		const FQuat BoneRotation = FQuat(RotationAxisVector, ActualOrientationAngle * IKFootRootOrientationAlpha);

		FTransform IKFootRootTransform(Output.Pose.GetComponentSpaceTransform(IKFootData.IKFootRootBoneIndex));
		IKFootRootTransform.SetRotation(BoneRotation * IKFootRootTransform.GetRotation());
		IKFootRootTransform.NormalizeRotation();
		Output.Pose.SetComponentSpaceTransform(IKFootData.IKFootRootBoneIndex, IKFootRootTransform);

		// IK Feet 
		// These match the root orientation, so don't rotate them. Just preserve root rotation. 
		// We need to update their translation though, since we rotated their parent (the IK Foot Root bone).
		const int32 NumIKFootBones = IKFootData.IKFootBoneIndexArray.Num();
		const bool bUpdateIKFootBones = bUpdateIKFootRoot && (NumIKFootBones > 0);

		if (bUpdateIKFootBones)
		{
			const FQuat IKFootRotation = FQuat(RotationAxisVector, -ActualOrientationAngle * IKFootRootOrientationAlpha);

			for (int32 ArrayIndex = 0; ArrayIndex < NumIKFootBones; ArrayIndex++)
			{
				const FCompactPoseBoneIndex& IKFootBoneIndex = IKFootData.IKFootBoneIndexArray[ArrayIndex];

				FTransform IKFootBoneTransform(Output.Pose.GetComponentSpaceTransform(IKFootBoneIndex));
				IKFootBoneTransform.SetRotation(IKFootRotation * IKFootBoneTransform.GetRotation());
				IKFootBoneTransform.NormalizeRotation();
				Output.Pose.SetComponentSpaceTransform(IKFootBoneIndex, IKFootBoneTransform);
			}
		}
	}

	OutBoneTransforms.Sort(FCompareBoneTransformIndex());
}
```

### Distance Match (距离匹配)

通过移动输入的实际位移，调整动画，解决起步或者停步时的滑步问题。

与参考文章有所出入，有些函数已经被移除插件库。

注释中提到需要跟步幅扭曲有所配合，因为AdvanceTimeByDistanceMatching似乎仅仅是提供了播放速率的调整，并没有调整骨骼。

* Engine\Plugins\Animation\AnimationLocomotionLibrary\Source\Runtime\Private\AnimDistanceMatchingLibrary.cpp
```Cpp
FSequenceEvaluatorReference UAnimDistanceMatchingLibrary::AdvanceTimeByDistanceMatching(const FAnimUpdateContext& UpdateContext, const FSequenceEvaluatorReference& SequenceEvaluator, float DistanceTraveled, FName DistanceCurveName, FVector2D PlayRateClamp)
{
	SequenceEvaluator.CallAnimNodeFunction<FAnimNode_SequenceEvaluator>(
		TEXT("AdvanceTimeByDistanceMatching"),
		[&UpdateContext, DistanceTraveled, DistanceCurveName, PlayRateClamp](FAnimNode_SequenceEvaluator& InSequenceEvaluator)
		{
			if (const FAnimationUpdateContext* AnimationUpdateContext = UpdateContext.GetContext())
			{
				const float DeltaTime = AnimationUpdateContext->GetDeltaTime(); 

				if (DeltaTime > 0 && DistanceTraveled > 0)
				{
					if (const UAnimSequenceBase* AnimSequence = Cast<UAnimSequence>(InSequenceEvaluator.GetSequence()))
					{
						const float CurrentTime = InSequenceEvaluator.GetExplicitTime();
						const float CurrentAssetLength = InSequenceEvaluator.GetCurrentAssetLength();
						const bool bAllowLooping = InSequenceEvaluator.GetShouldLoop();

						const USkeleton::AnimCurveUID CurveUID = UE::Anim::DistanceMatchingUtility::GetCurveUID(AnimSequence, DistanceCurveName);
						float TimeAfterDistanceTraveled = UE::Anim::DistanceMatchingUtility::GetTimeAfterDistanceTraveled(AnimSequence, CurrentTime, DistanceTraveled, CurveUID, bAllowLooping);

						// Calculate the effective playrate that would result from advancing the animation by the distance traveled.
						// Account for the animation looping.
						if (TimeAfterDistanceTraveled < CurrentTime)
						{
							TimeAfterDistanceTraveled += CurrentAssetLength;
						}
						float EffectivePlayRate = (TimeAfterDistanceTraveled - CurrentTime) / DeltaTime;

						// Clamp the effective play rate.
						if (PlayRateClamp.X >= 0.0f && PlayRateClamp.X < PlayRateClamp.Y)
						{
							EffectivePlayRate = FMath::Clamp(EffectivePlayRate, PlayRateClamp.X, PlayRateClamp.Y);
						}

						// Advance animation time by the effective play rate.
						float NewTime = CurrentTime;
						FAnimationRuntime::AdvanceTime(bAllowLooping, EffectivePlayRate * DeltaTime, NewTime, CurrentAssetLength);

						if (!InSequenceEvaluator.SetExplicitTime(NewTime))
						{
							UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Could not set explicit time on sequence evaluator, value is not dynamic. Set it as Always Dynamic."));
						}
					}
					else
					{
						UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Sequence evaluator does not have an anim sequence to play."));
					}
				}
			}
			else
			{
				UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("AdvanceTimeByDistanceMatching called with invalid context"));
			}
		});

	return SequenceEvaluator;
}

FSequenceEvaluatorReference UAnimDistanceMatchingLibrary::DistanceMatchToTarget(const FSequenceEvaluatorReference& SequenceEvaluator, float DistanceToTarget, FName DistanceCurveName)
{
	SequenceEvaluator.CallAnimNodeFunction<FAnimNode_SequenceEvaluator>(
		TEXT("DistanceMatchToTarget"),
		[DistanceToTarget, DistanceCurveName](FAnimNode_SequenceEvaluator& InSequenceEvaluator)
		{
			if (const UAnimSequenceBase* AnimSequence = Cast<UAnimSequence>(InSequenceEvaluator.GetSequence()))
			{
				const USkeleton::AnimCurveUID CurveUID = UE::Anim::DistanceMatchingUtility::GetCurveUID(AnimSequence, DistanceCurveName);
				if (AnimSequence->HasCurveData(CurveUID))
				{
					// By convention, distance curves store the distance to a target as a negative value.
					const float NewTime = UE::Anim::DistanceMatchingUtility::GetAnimPositionFromDistance(AnimSequence, -DistanceToTarget, CurveUID);
					if (!InSequenceEvaluator.SetExplicitTime(NewTime))
					{
						UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Could not set explicit time on sequence evaluator, value is not dynamic. Set it as Always Dynamic."));
					}
				}
				else
				{
					UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("DistanceMatchToTarget called with invalid DistanceCurveName or animation (%s) is missing a distance curve."), *GetNameSafe(AnimSequence));
				}
			}
			else
			{
				UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Sequence evaluator does not have an anim sequence to play."));
			}
			
		});

	return SequenceEvaluator;
}

FSequencePlayerReference UAnimDistanceMatchingLibrary::SetPlayrateToMatchSpeed(const FSequencePlayerReference& SequencePlayer, float SpeedToMatch, FVector2D PlayRateClamp)
{
	SequencePlayer.CallAnimNodeFunction<FAnimNode_SequencePlayer>(
		TEXT("SetPlayrateToMatchSpeed"),
		[SpeedToMatch, PlayRateClamp](FAnimNode_SequencePlayer& InSequencePlayer)
		{
			if (const UAnimSequence* AnimSequence = Cast<UAnimSequence>(InSequencePlayer.GetSequence()))
			{
				const float AnimLength = AnimSequence->GetPlayLength();
				if (!FMath::IsNearlyZero(AnimLength))
				{
					// Calculate the speed as: (distance traveled by the animation) / (length of the animation)
					const FVector RootMotionTranslation = AnimSequence->ExtractRootMotionFromRange(0.0f, AnimLength).GetTranslation();
					const float RootMotionDistance = RootMotionTranslation.Size2D();
					if (!FMath::IsNearlyZero(RootMotionDistance))
					{
						const float AnimationSpeed = RootMotionDistance / AnimLength;
						float DesiredPlayRate = SpeedToMatch / AnimationSpeed;
						if (PlayRateClamp.X >= 0.0f && PlayRateClamp.X < PlayRateClamp.Y)
						{
							DesiredPlayRate = FMath::Clamp(DesiredPlayRate, PlayRateClamp.X, PlayRateClamp.Y);
						}

						if (!InSequencePlayer.SetPlayRate(DesiredPlayRate))
						{
							UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Could not set play rate on sequence player, value is not dynamic. Set it as Always Dynamic."));
						}
					}
					else
					{
						UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Unable to adjust playrate for animation with no root motion delta (%s)."), *GetNameSafe(AnimSequence));
					}
				}
				else
				{
					UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Unable to adjust playrate for zero length animation (%s)."), *GetNameSafe(AnimSequence));
				}
			}
			else
			{
				UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Sequence player does not have an anim sequence to play."));
			}
		});

	return SequencePlayer;
}
```

### Pivot

?回转运动?主要是指进行反方向移动时应该执行的动画，比如前进中收到后退的输入，以现实来说应该先做出减速的动作再往后加速

### Aim Offset (瞄准偏移)



# Lyra Inventory and Lyra Equipment

## Class Diagram

{% asset_img "Unreal Engine 5 Lyra Starter Game Weapon Class Diagram.png" Inventory and Equipment %}

## Key Class

* ULyraInventoryManagerComponent

# Lyra Abilities

使用游戏技能系统来构建玩法内容，同时也会依赖Game Features来动态装卸玩法内容。

## Key Class

* FLyraAbilityGrant

Lyra技能结构体，有UGameplayAbility成员，意义大概是定义要赋予的技能。UInputAction成员目前被注释掉了，并且提到该成员可以留空。

* FLyraAttributeSetGrant

Lyra属性集结构体，有UAtrributeSet成员和UDataTable成员，意义大概也是定义要赋予的属性集，UDatatable成员是用来存储初始化数据，可以留空。

* ULyraAbilitySet

> Non-mutable data asset used to grant gameplay abilities and gameplay effects.

继承UPrimaryDataAsset，用来赋予给Actor的技能和玩法效果的不可变数据资产。

* FGameFeatureAbilitiesEntry

在Game Feature中使用的Gameplay Ability System的数据结构单位，包含有FLyraAbiltyGrant成员数组、FLyraAttributeSetGrant成员数组、ULyraAbilitySet成员数组，另外还有一个Actor成员，意义是该结构体的技能要添加到的Actor。

* UGameFeatureAction_AddAbilities

> GameFeatureAction responsible for granting abilities (and attributes) to actors of a specified type.

继承UGameFeatureAction_WorldActionBase，负责赋予Actor特定技能和激活技能相关的逻辑。Abilities相关的Game Feature激活时会调用此类定义的回调函数。

* ULyraGameplayAbility

> The base gameplay ability class used by this project.

继承UGameplayAbility，Lyra的游戏技能基类。

* ULyraGameplayAbility_FromEquipment

> An ability granted by and associated with an equipment instance.

继承ULyraGameplayAbility，一种与装备实例相关的技能类。

* ULyraGameplayAbility_RangedWeapon

> An ability granted by and associated with a ranged weapon instance

继承ULyraGameplayAbility_FromEquipment，一种与远程武器实例相关的技能类。

## Flow Chart

需要结合游戏技能系统的工作流程来构造成一个Lyra项目下完整的技能工作流程。

{% asset_img %}

## Implementation

* LyraStarterGame\Source\LyraGame\GameFeatures\GameFeatureAciton_AddAbilities.cpp
```Cpp
void UGameFeatureAction_AddAbilities::AddToWorld(const FWorldContext& WorldContext, const FGameFeatureStateChangeContext& ChangeContext)
{
	UWorld* World = WorldContext.World();
	UGameInstance* GameInstance = WorldContext.OwningGameInstance;
	FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

	if ((GameInstance != nullptr) && (World != nullptr) && World->IsGameWorld())
	{
		if (UGameFrameworkComponentManager* ComponentMan = UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GameInstance))
		{			
			int32 EntryIndex = 0;
			for (const FGameFeatureAbilitiesEntry& Entry : AbilitiesList)
			{
				if (!Entry.ActorClass.IsNull())
				{
					UGameFrameworkComponentManager::FExtensionHandlerDelegate AddAbilitiesDelegate = UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
						this, &UGameFeatureAction_AddAbilities::HandleActorExtension, EntryIndex, ChangeContext);
					TSharedPtr<FComponentRequestHandle> ExtensionRequestHandle = ComponentMan->AddExtensionHandler(Entry.ActorClass, AddAbilitiesDelegate);

					ActiveData.ComponentRequests.Add(ExtensionRequestHandle);
					EntryIndex++;
				}
			}
		}
	}
}

void UGameFeatureAction_AddAbilities::HandleActorExtension(AActor* Actor, FName EventName, int32 EntryIndex, FGameFeatureStateChangeContext ChangeContext)
{
	FPerContextData* ActiveData = ContextData.Find(ChangeContext);
	if (AbilitiesList.IsValidIndex(EntryIndex) && ActiveData)
	{
		const FGameFeatureAbilitiesEntry& Entry = AbilitiesList[EntryIndex];
		if ((EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved) || (EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved))
		{
			RemoveActorAbilities(Actor, *ActiveData);
		}
		else if ((EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded) || (EventName == ALyraPlayerState::NAME_LyraAbilityReady))
		{
			AddActorAbilities(Actor, Entry, *ActiveData);
		}
	}
}

void UGameFeatureAction_AddAbilities::AddActorAbilities(AActor* Actor, const FGameFeatureAbilitiesEntry& AbilitiesEntry, FPerContextData& ActiveData)
{
	check(Actor);
	if (!Actor->HasAuthority())
	{
		return;
	}

	// early out if Actor already has ability extensions applied
	if (ActiveData.ActiveExtensions.Find(Actor) != nullptr)
	{
		return;	
	}

	if (UAbilitySystemComponent* AbilitySystemComponent = FindOrAddComponentForActor<UAbilitySystemComponent>(Actor, AbilitiesEntry, ActiveData))
	{
		FActorExtensions AddedExtensions;
		AddedExtensions.Abilities.Reserve(AbilitiesEntry.GrantedAbilities.Num());
		AddedExtensions.Attributes.Reserve(AbilitiesEntry.GrantedAttributes.Num());
		AddedExtensions.AbilitySetHandles.Reserve(AbilitiesEntry.GrantedAbilitySets.Num());

		for (const FLyraAbilityGrant& Ability : AbilitiesEntry.GrantedAbilities)
		{
			if (!Ability.AbilityType.IsNull())
			{
				FGameplayAbilitySpec NewAbilitySpec(Ability.AbilityType.LoadSynchronous());
				FGameplayAbilitySpecHandle AbilityHandle = AbilitySystemComponent->GiveAbility(NewAbilitySpec);

				AddedExtensions.Abilities.Add(AbilityHandle);
			}
		}

		for (const FLyraAttributeSetGrant& Attributes : AbilitiesEntry.GrantedAttributes)
		{
			if (!Attributes.AttributeSetType.IsNull())
			{
				TSubclassOf<UAttributeSet> SetType = Attributes.AttributeSetType.LoadSynchronous();
				if (SetType)
				{
					UAttributeSet* NewSet = NewObject<UAttributeSet>(AbilitySystemComponent->GetOwner(), SetType);
					if (!Attributes.InitializationData.IsNull())
					{
						UDataTable* InitData = Attributes.InitializationData.LoadSynchronous();
						if (InitData)
						{
							NewSet->InitFromMetaDataTable(InitData);
						}
					}

					AddedExtensions.Attributes.Add(NewSet);
					AbilitySystemComponent->AddAttributeSetSubobject(NewSet);
				}
			}
		}

		ULyraAbilitySystemComponent* LyraASC = CastChecked<ULyraAbilitySystemComponent>(AbilitySystemComponent);
		for (const TSoftObjectPtr<const ULyraAbilitySet>& SetPtr : AbilitiesEntry.GrantedAbilitySets)
		{
			if (const ULyraAbilitySet* Set = SetPtr.Get())
			{
				Set->GiveToAbilitySystem(LyraASC, &AddedExtensions.AbilitySetHandles.AddDefaulted_GetRef());
			}
		}

		ActiveData.ActiveExtensions.Add(Actor, AddedExtensions);
	}
	else
	{
		UE_LOG(LogGameFeatures, Error, TEXT("Failed to find/add an ability component to '%s'. Abilities will not be granted."), *Actor->GetPathName());
	}
}
```

* LyraStarterGame\Source\LyraGame\AbilitySystem\LyraAbilitySet.cpp
```Cpp
void ULyraAbilitySet::GiveToAbilitySystem(ULyraAbilitySystemComponent* LyraASC, FLyraAbilitySet_GrantedHandles* OutGrantedHandles, UObject* SourceObject) const
{
	check(LyraASC);

	if (!LyraASC->IsOwnerActorAuthoritative())
	{
		// Must be authoritative to give or take ability sets.
		return;
	}

	// Grant the gameplay abilities.
	for (int32 AbilityIndex = 0; AbilityIndex < GrantedGameplayAbilities.Num(); ++AbilityIndex)
	{
		const FLyraAbilitySet_GameplayAbility& AbilityToGrant = GrantedGameplayAbilities[AbilityIndex];

		if (!IsValid(AbilityToGrant.Ability))
		{
			UE_LOG(LogLyraAbilitySystem, Error, TEXT("GrantedGameplayAbilities[%d] on ability set [%s] is not valid."), AbilityIndex, *GetNameSafe(this));
			continue;
		}

		ULyraGameplayAbility* AbilityCDO = AbilityToGrant.Ability->GetDefaultObject<ULyraGameplayAbility>();

		FGameplayAbilitySpec AbilitySpec(AbilityCDO, AbilityToGrant.AbilityLevel);
		AbilitySpec.SourceObject = SourceObject;
		AbilitySpec.DynamicAbilityTags.AddTag(AbilityToGrant.InputTag);

		const FGameplayAbilitySpecHandle AbilitySpecHandle = LyraASC->GiveAbility(AbilitySpec);

		if (OutGrantedHandles)
		{
			OutGrantedHandles->AddAbilitySpecHandle(AbilitySpecHandle);
		}
	}

	// Grant the gameplay effects.
	for (int32 EffectIndex = 0; EffectIndex < GrantedGameplayEffects.Num(); ++EffectIndex)
	{
		const FLyraAbilitySet_GameplayEffect& EffectToGrant = GrantedGameplayEffects[EffectIndex];

		if (!IsValid(EffectToGrant.GameplayEffect))
		{
			UE_LOG(LogLyraAbilitySystem, Error, TEXT("GrantedGameplayEffects[%d] on ability set [%s] is not valid"), EffectIndex, *GetNameSafe(this));
			continue;
		}

		const UGameplayEffect* GameplayEffect = EffectToGrant.GameplayEffect->GetDefaultObject<UGameplayEffect>();
		const FActiveGameplayEffectHandle GameplayEffectHandle = LyraASC->ApplyGameplayEffectToSelf(GameplayEffect, EffectToGrant.EffectLevel, LyraASC->MakeEffectContext());

		if (OutGrantedHandles)
		{
			OutGrantedHandles->AddGameplayEffectHandle(GameplayEffectHandle);
		}
	}

	// Grant the attribute sets.
	for (int32 SetIndex = 0; SetIndex < GrantedAttributes.Num(); ++SetIndex)
	{
		const FLyraAbilitySet_AttributeSet& SetToGrant = GrantedAttributes[SetIndex];

		if (!IsValid(SetToGrant.AttributeSet))
		{
			UE_LOG(LogLyraAbilitySystem, Error, TEXT("GrantedAttributes[%d] on ability set [%s] is not valid"), SetIndex, *GetNameSafe(this));
			continue;
		}

		UAttributeSet* NewSet = NewObject<UAttributeSet>(LyraASC->GetOwner(), SetToGrant.AttributeSet);
		LyraASC->AddAttributeSetSubobject(NewSet);

		if (OutGrantedHandles)
		{
			OutGrantedHandles->AddAttributeSet(NewSet);
		}
	}
}
```

## Class Diagram



## Conclusion for myself

在Actor初始化后，会触发委托，Game Feature会开始通过Actor的技能组件，把技能添加到所属的Actor。

## 具体逻辑样例实现

### 武器开火到伤害应用

#### Flow Chart



#### Key Implmentation

* LyraStarterGame\Source\LyraGame\Weapons\LyraGameplayAbility_RangedWeapon.cpp
```Cpp
void ULyraGameplayAbility_RangedWeapon::StartRangedWeaponTargeting()
{
	check(CurrentActorInfo);

	AActor* AvatarActor = CurrentActorInfo->AvatarActor.Get();
	check(AvatarActor);

	UAbilitySystemComponent* MyAbilityComponent = CurrentActorInfo->AbilitySystemComponent.Get();
	check(MyAbilityComponent);

	AController* Controller = GetControllerFromActorInfo();
	check(Controller);
	ULyraWeaponStateComponent* WeaponStateComponent = Controller->FindComponentByClass<ULyraWeaponStateComponent>();

	FScopedPredictionWindow ScopedPrediction(MyAbilityComponent, CurrentActivationInfo.GetActivationPredictionKey());

	TArray<FHitResult> FoundHits;
	PerformLocalTargeting(/*out*/ FoundHits);

	// Fill out the target data from the hit results
	FGameplayAbilityTargetDataHandle TargetData;
	TargetData.UniqueId = WeaponStateComponent ? WeaponStateComponent->GetUnconfirmedServerSideHitMarkerCount() : 0;

	if (FoundHits.Num() > 0)
	{
		const int32 CartridgeID = FMath::Rand();

		for (const FHitResult& FoundHit : FoundHits)
		{
			FLyraGameplayAbilityTargetData_SingleTargetHit* NewTargetData = new FLyraGameplayAbilityTargetData_SingleTargetHit();
			NewTargetData->HitResult = FoundHit;
			NewTargetData->CartridgeID = CartridgeID;

			TargetData.Add(NewTargetData);
		}
	}

	// Send hit marker information
	const bool bProjectileWeapon = false;
	if (!bProjectileWeapon && (WeaponStateComponent != nullptr))
	{
		WeaponStateComponent->AddUnconfirmedServerSideHitMarkers(TargetData, FoundHits);
	}

	// Process the target data immediately
	OnTargetDataReadyCallback(TargetData, FGameplayTag());
}

void ULyraGameplayAbility_RangedWeapon::PerformLocalTargeting(OUT TArray<FHitResult>& OutHits)
{
	APawn* const AvatarPawn = Cast<APawn>(GetAvatarActorFromActorInfo());

	ULyraRangedWeaponInstance* WeaponData = GetWeaponInstance();
	if (AvatarPawn && AvatarPawn->IsLocallyControlled() && WeaponData)
	{
		FRangedWeaponFiringInput InputData;
		InputData.WeaponData = WeaponData;
		InputData.bCanPlayBulletFX = (AvatarPawn->GetNetMode() != NM_DedicatedServer);

		//@TODO: Should do more complicated logic here when the player is close to a wall, etc...
		const FTransform TargetTransform = GetTargetingTransform(AvatarPawn, ELyraAbilityTargetingSource::CameraTowardsFocus);
		InputData.AimDir = TargetTransform.GetUnitAxis(EAxis::X);
		InputData.StartTrace = TargetTransform.GetTranslation();

		InputData.EndAim = InputData.StartTrace + InputData.AimDir * WeaponData->GetMaxDamageRange();

#if ENABLE_DRAW_DEBUG
		if (LyraConsoleVariables::DrawBulletTracesDuration > 0.0f)
		{
			static float DebugThickness = 2.0f;
			DrawDebugLine(GetWorld(), InputData.StartTrace, InputData.StartTrace + (InputData.AimDir * 100.0f), FColor::Yellow, false, LyraConsoleVariables::DrawBulletTracesDuration, 0, DebugThickness);
		}
#endif

		TraceBulletsInCartridge(InputData, /*out*/ OutHits);
	}
}

void ULyraGameplayAbility_RangedWeapon::TraceBulletsInCartridge(const FRangedWeaponFiringInput& InputData, OUT TArray<FHitResult>& OutHits)
{
	ULyraRangedWeaponInstance* WeaponData = InputData.WeaponData;
	check(WeaponData);

	const int32 BulletsPerCartridge = WeaponData->GetBulletsPerCartridge();

	for (int32 BulletIndex = 0; BulletIndex < BulletsPerCartridge; ++BulletIndex)
	{
		const float BaseSpreadAngle = WeaponData->GetCalculatedSpreadAngle();
		const float SpreadAngleMultiplier = WeaponData->GetCalculatedSpreadAngleMultiplier();
		const float ActualSpreadAngle = BaseSpreadAngle * SpreadAngleMultiplier;

		const float HalfSpreadAngleInRadians = FMath::DegreesToRadians(ActualSpreadAngle * 0.5f);

		const FVector BulletDir = VRandConeNormalDistribution(InputData.AimDir, HalfSpreadAngleInRadians, WeaponData->GetSpreadExponent());

		const FVector EndTrace = InputData.StartTrace + (BulletDir * WeaponData->GetMaxDamageRange());
		FVector HitLocation = EndTrace;

		TArray<FHitResult> AllImpacts;

		FHitResult Impact = DoSingleBulletTrace(InputData.StartTrace, EndTrace, WeaponData->GetBulletTraceSweepRadius(), /*bIsSimulated=*/ false, /*out*/ AllImpacts);

		const AActor* HitActor = Impact.GetActor();

		if (HitActor)
		{
#if ENABLE_DRAW_DEBUG
			if (LyraConsoleVariables::DrawBulletHitDuration > 0.0f)
			{
				DrawDebugPoint(GetWorld(), Impact.ImpactPoint, LyraConsoleVariables::DrawBulletHitRadius, FColor::Red, false, LyraConsoleVariables::DrawBulletHitRadius);
			}
#endif

			if (AllImpacts.Num() > 0)
			{
				OutHits.Append(AllImpacts);
			}

			HitLocation = Impact.ImpactPoint;
		}

		// Make sure there's always an entry in OutHits so the direction can be used for tracers, etc...
		if (OutHits.Num() == 0)
		{
			if (!Impact.bBlockingHit)
			{
				// Locate the fake 'impact' at the end of the trace
				Impact.Location = EndTrace;
				Impact.ImpactPoint = EndTrace;
			}

			OutHits.Add(Impact);
		}
	}
}

FHitResult ULyraGameplayAbility_RangedWeapon::DoSingleBulletTrace(const FVector& StartTrace, const FVector& EndTrace, float SweepRadius, bool bIsSimulated, OUT TArray<FHitResult>& OutHits) const
{
#if ENABLE_DRAW_DEBUG
	if (LyraConsoleVariables::DrawBulletTracesDuration > 0.0f)
	{
		static float DebugThickness = 1.0f;
		DrawDebugLine(GetWorld(), StartTrace, EndTrace, FColor::Red, false, LyraConsoleVariables::DrawBulletTracesDuration, 0, DebugThickness);
	}
#endif // ENABLE_DRAW_DEBUG

	FHitResult Impact;

	// Trace and process instant hit if something was hit
	// First trace without using sweep radius
	if (FindFirstPawnHitResult(OutHits) == INDEX_NONE)
	{
		Impact = WeaponTrace(StartTrace, EndTrace, /*SweepRadius=*/ 0.0f, bIsSimulated, /*out*/ OutHits);
	}

	if (FindFirstPawnHitResult(OutHits) == INDEX_NONE)
	{
		// If this weapon didn't hit anything with a line trace and supports a sweep radius, try that
		if (SweepRadius > 0.0f)
		{
			TArray<FHitResult> SweepHits;
			Impact = WeaponTrace(StartTrace, EndTrace, SweepRadius, bIsSimulated, /*out*/ SweepHits);

			// If the trace with sweep radius enabled hit a pawn, check if we should use its hit results
			const int32 FirstPawnIdx = FindFirstPawnHitResult(SweepHits);
			if (SweepHits.IsValidIndex(FirstPawnIdx))
			{
				// If we had a blocking hit in our line trace that occurs in SweepHits before our
				// hit pawn, we should just use our initial hit results since the Pawn hit should be blocked
				bool bUseSweepHits = true;
				for (int32 Idx = 0; Idx < FirstPawnIdx; ++Idx)
				{
					const FHitResult& CurHitResult = SweepHits[Idx];

					auto Pred = [&CurHitResult](const FHitResult& Other)
					{
						return Other.HitObjectHandle == CurHitResult.HitObjectHandle;
					};
					if (CurHitResult.bBlockingHit && OutHits.ContainsByPredicate(Pred))
					{
						bUseSweepHits = false;
						break;
					}
				}

				if (bUseSweepHits)
				{
					OutHits = SweepHits;
				}
			}
		}
	}

	return Impact;
}

FHitResult ULyraGameplayAbility_RangedWeapon::WeaponTrace(const FVector& StartTrace, const FVector& EndTrace, float SweepRadius, bool bIsSimulated, OUT TArray<FHitResult>& OutHitResults) const
{
	TArray<FHitResult> HitResults;
	
	FCollisionQueryParams TraceParams(SCENE_QUERY_STAT(WeaponTrace), /*bTraceComplex=*/ true, /*IgnoreActor=*/ GetAvatarActorFromActorInfo());
	TraceParams.bReturnPhysicalMaterial = true;
	AddAdditionalTraceIgnoreActors(TraceParams);
	//TraceParams.bDebugQuery = true;

	const ECollisionChannel TraceChannel = DetermineTraceChannel(TraceParams, bIsSimulated);

	if (SweepRadius > 0.0f)
	{
		GetWorld()->SweepMultiByChannel(HitResults, StartTrace, EndTrace, FQuat::Identity, TraceChannel, FCollisionShape::MakeSphere(SweepRadius), TraceParams);
	}
	else
	{
		GetWorld()->LineTraceMultiByChannel(HitResults, StartTrace, EndTrace, TraceChannel, TraceParams);
	}

	FHitResult Hit(ForceInit);
	if (HitResults.Num() > 0)
	{
		// Filter the output list to prevent multiple hits on the same actor;
		// this is to prevent a single bullet dealing damage multiple times to
		// a single actor if using an overlap trace
		for (FHitResult& CurHitResult : HitResults)
		{
			auto Pred = [&CurHitResult](const FHitResult& Other)
			{
				return Other.HitObjectHandle == CurHitResult.HitObjectHandle;
			};

			if (!OutHitResults.ContainsByPredicate(Pred))
			{
				OutHitResults.Add(CurHitResult);
			}
		}

		Hit = OutHitResults.Last();
	}
	else
	{
		Hit.TraceStart = StartTrace;
		Hit.TraceEnd = EndTrace;
	}

	return Hit;
}

void ULyraGameplayAbility_RangedWeapon::OnTargetDataReadyCallback(const FGameplayAbilityTargetDataHandle& InData, FGameplayTag ApplicationTag)
{
	UAbilitySystemComponent* MyAbilityComponent = CurrentActorInfo->AbilitySystemComponent.Get();
	check(MyAbilityComponent);

	if (const FGameplayAbilitySpec* AbilitySpec = MyAbilityComponent->FindAbilitySpecFromHandle(CurrentSpecHandle))
	{
		FScopedPredictionWindow	ScopedPrediction(MyAbilityComponent);

		// Take ownership of the target data to make sure no callbacks into game code invalidate it out from under us
		FGameplayAbilityTargetDataHandle LocalTargetDataHandle(MoveTemp(const_cast<FGameplayAbilityTargetDataHandle&>(InData)));

		const bool bShouldNotifyServer = CurrentActorInfo->IsLocallyControlled() && !CurrentActorInfo->IsNetAuthority();
		if (bShouldNotifyServer)
		{
			MyAbilityComponent->CallServerSetReplicatedTargetData(CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey(), LocalTargetDataHandle, ApplicationTag, MyAbilityComponent->ScopedPredictionKey);
		}

		const bool bIsTargetDataValid = true;

		bool bProjectileWeapon = false;

#if WITH_SERVER_CODE
		if (!bProjectileWeapon)
		{
			if (AController* Controller = GetControllerFromActorInfo())
			{
				if (Controller->GetLocalRole() == ROLE_Authority)
				{
					// Confirm hit markers
					if (ULyraWeaponStateComponent* WeaponStateComponent = Controller->FindComponentByClass<ULyraWeaponStateComponent>())
					{
						TArray<uint8> HitReplaces;
						for (uint8 i = 0; (i < LocalTargetDataHandle.Num()) && (i < 255); ++i)
						{
							if (FGameplayAbilityTargetData_SingleTargetHit* SingleTargetHit = static_cast<FGameplayAbilityTargetData_SingleTargetHit*>(LocalTargetDataHandle.Get(i)))
							{
								if (SingleTargetHit->bHitReplaced)
								{
									HitReplaces.Add(i);
								}
							}
						}

						WeaponStateComponent->ClientConfirmTargetData(LocalTargetDataHandle.UniqueId, bIsTargetDataValid, HitReplaces);
					}

				}
			}
		}
#endif //WITH_SERVER_CODE


		// See if we still have ammo
		if (bIsTargetDataValid && CommitAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo))
		{
			// We fired the weapon, add spread
			ULyraRangedWeaponInstance* WeaponData = GetWeaponInstance();
			check(WeaponData);
			WeaponData->AddSpread();

			// Let the blueprint do stuff like apply effects to the targets
			OnRangedWeaponTargetDataReady(LocalTargetDataHandle);
		}
		else
		{
			UE_LOG(LogLyraAbilitySystem, Warning, TEXT("Weapon ability %s failed to commit (bIsTargetDataValid=%d)"), *GetPathName(), bIsTargetDataValid ? 1 : 0);
			K2_EndAbility();
		}
	}

	// We've processed the data
	MyAbilityComponent->ConsumeClientReplicatedTargetData(CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey());
}
```

* LyraStarterGame\Source\LyraGame\AbilitySystem\Executions\LyraDamageExecution.cpp
```Cpp
void ULyraDamageExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
#if WITH_SERVER_CODE
	const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();
	FLyraGameplayEffectContext* TypedContext = FLyraGameplayEffectContext::ExtractEffectContext(Spec.GetContext());
	check(TypedContext);

	const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
	const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

	FAggregatorEvaluateParameters EvaluateParameters;
	EvaluateParameters.SourceTags = SourceTags;
	EvaluateParameters.TargetTags = TargetTags;

	float BaseDamage = 0.0f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().BaseDamageDef, EvaluateParameters, BaseDamage);

	const AActor* EffectCauser = TypedContext->GetEffectCauser();
	const FHitResult* HitActorResult = TypedContext->GetHitResult();

	AActor* HitActor = nullptr;
	FVector ImpactLocation = FVector::ZeroVector;
	FVector ImpactNormal = FVector::ZeroVector;
	FVector StartTrace = FVector::ZeroVector;
	FVector EndTrace = FVector::ZeroVector;

	// Calculation of hit actor, surface, zone, and distance all rely on whether the calculation has a hit result or not.
	// Effects just being added directly w/o having been targeted will always come in without a hit result, which must default
	// to some fallback information.
	if (HitActorResult)
	{
		const FHitResult& CurHitResult = *HitActorResult;
		HitActor = CurHitResult.HitObjectHandle.FetchActor();
		if (HitActor)
		{
			ImpactLocation = CurHitResult.ImpactPoint;
			ImpactNormal = CurHitResult.ImpactNormal;
			StartTrace = CurHitResult.TraceStart;
			EndTrace = CurHitResult.TraceEnd;
		}
	}

	// Handle case of no hit result or hit result not actually returning an actor
	UAbilitySystemComponent* TargetAbilitySystemComponent = ExecutionParams.GetTargetAbilitySystemComponent();
	if (!HitActor)
	{
		HitActor = TargetAbilitySystemComponent ? TargetAbilitySystemComponent->GetAvatarActor_Direct() : nullptr;
		if (HitActor)
		{
			ImpactLocation = HitActor->GetActorLocation();
		}
	}

	// Apply rules for team damage/self damage/etc...
	float DamageInteractionAllowedMultiplier = 0.0f;
	if (HitActor)
	{
		ULyraTeamSubsystem* TeamSubsystem = HitActor->GetWorld()->GetSubsystem<ULyraTeamSubsystem>();
		if (ensure(TeamSubsystem))
		{
			DamageInteractionAllowedMultiplier = TeamSubsystem->CanCauseDamage(EffectCauser, HitActor) ? 1.0 : 0.0;
		}
	}

	// Determine distance
	double Distance = WORLD_MAX;

	if (TypedContext->HasOrigin())
	{
		Distance = FVector::Dist(TypedContext->GetOrigin(), ImpactLocation);
	}
	else if (EffectCauser)
	{
		Distance = FVector::Dist(EffectCauser->GetActorLocation(), ImpactLocation);
	}
	else
	{
		ensureMsgf(false, TEXT("Damage Calculation cannot deduce a source location for damage coming from %s; Falling back to WORLD_MAX dist!"), *GetPathNameSafe(Spec.Def));
	}

	// Apply ability source modifiers
	float PhysicalMaterialAttenuation = 1.0f;
	float DistanceAttenuation = 1.0f;
	if (const ILyraAbilitySourceInterface* AbilitySource = TypedContext->GetAbilitySource())
	{
		if (const UPhysicalMaterial* PhysMat = TypedContext->GetPhysicalMaterial())
		{
			PhysicalMaterialAttenuation = AbilitySource->GetPhysicalMaterialAttenuation(PhysMat, SourceTags, TargetTags);
		}

		DistanceAttenuation = AbilitySource->GetDistanceAttenuation(Distance, SourceTags, TargetTags);
	}
	DistanceAttenuation = FMath::Max(DistanceAttenuation, 0.0f);

	// Clamping is done when damage is converted to -health
	const float DamageDone = FMath::Max(BaseDamage * DistanceAttenuation * PhysicalMaterialAttenuation * DamageInteractionAllowedMultiplier, 0.0f);

	if (DamageDone > 0.0f)
	{
		// Apply a damage modifier, this gets turned into - health on the target
		OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(ULyraHealthSet::GetDamageAttribute(), EGameplayModOp::Additive, DamageDone));
	}
#endif // #if WITH_SERVER_CODE
}
```

#### Conclusion for myself

该方案是射线检测，根据瞄准和枪械的参数进行搜索，然后通过游戏技能系统和Game Feature去调用执行伤害计算，最终再通过游戏技能系统和Game Feature应用伤害至目标Actor。

Modifier让我有点熟悉，是Dota 2创意工坊的文档中有提过，并且有过编写，Modifier可以当作Buff或者Debuff对象，与这里遇到的Modifier有一定程度的相似。

# Lyra Camera



# Lyra AI Behavior Tree



# Conclusion for myself

Lyra的代码设计大量采用ECS架构设计，能用来缓解AActor、ACharacter、AController在Gameplay代码设计上的臃肿问题。
相比Unreal Engine 4时期的ShooterGame样例，Lyra样例更加复杂，提供了更好的独立游戏模板，更好的功能模块细分思路可供参考，偏向一个AAA级游戏开发工业的细分方式。
动态装卸玩法的框架设计非常适合网游或者长期运营的游戏，但是可能不一定适用于一些大型单机游戏。用来做RPG为主的混合类型游戏相当合适，但是不代表做别的游戏类型不合适，因为其中的代码架构设计也可以用来做别的游戏。
EnhancedInput比旧版本的输入提供了更多的轮子，比如长按、点按等方式不用自己造了，但是将配置、输入方式、配置映射、游戏逻辑分开，用起来其实有一点麻烦，但是其可扩展性和灵活性相当的好。不单止可以做普通的角色射击，还可以加其他载具相关的。
第一次接触动画向开发相关的结构，Lyra动画方案提供了更优的模块化，争取了更高的实时性能。
数据驱动型的Gameplay框架，可将玩法设计完全交给了设计师。但是要求设计师有写蓝图脚本的能力，不太清楚国内的团队，国外成熟的开发团队确实可以采用这个方案，因为我了解到的国外成熟团队设计师确实具备写脚本的技能。

# Reference

## Official Documentation

- [Enhanced Input in Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/enhanced-input-in-unreal-engine/)
- [Gameplay Ability System for Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/gameplay-ability-system-for-unreal-engine/)
- [Lyra Input Settings in Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/lyra-input-settings-in-unreal-engine/)
- [Abilities in Lyra in Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/abilities-in-lyra-in-unreal-engine/)
- [Lyra Inventory and Equipment in Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/lyra-inventory-and-equipment-in-unreal-engine/)
- [Animation Notifies in Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/animation-notifies-in-unreal-engine/)
- [Data Registries in Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/data-registries-in-unreal-engine/)

## Third-Party Articles

- [GDC Vault - 'Overwatch' Gameplay Architecture and Netcode](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)
- [游戏开发中的ECS架构概述 – 知乎](https://zhuanlan.zhihu.com/p/30538626)
- [《InsideUE5》GameFeatures架构（三）初始化 - 知乎](https://zhuanlan.zhihu.com/p/473535854)
- [《InsideUE5》GameFeatures架构（四）状态机 - 知乎](https://zhuanlan.zhihu.com/p/484763722)
- [《InsideUE5》GameFeatures架构（五）AddComponents - 知乎](https://zhuanlan.zhihu.com/p/492893002)
- [UE的GAS原理深入探究一：ASC组件与GA - 知乎](https://zhuanlan.zhihu.com/p/440168260)
- [UE5 -- Lyra中的输入模块(Input) - 知乎](https://zhuanlan.zhihu.com/p/537949870)
- [UE5 -- EnhancedInput(输入增强系统) - 知乎](https://zhuanlan.zhihu.com/p/470949422)
- [UE5 Motion Warping(运动扭曲)原理剖析及UE4适配 – 知乎](https://zhuanlan.zhihu.com/p/378948277)
- [UE5 Distance Matching插件应用与源码解析 - 知乎](https://zhuanlan.zhihu.com/p/545559834)
- [上万字详解UE5动画新特性Pose Warping原理与应用 – 知乎](https://zhuanlan.zhihu.com/p/555984712)
- [UE5的动画蓝图（Lyra工程）- 知乎](https://zhuanlan.zhihu.com/p/517368184)
- [UE5新项目Gameplay框架设计(以Lyra为例) - 知乎](https://zhuanlan.zhihu.com/p/614718286)