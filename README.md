# Shadow-Samurai-Showcase

This is a personal project built using a modified version of the [https://github.com/Qhaxi/GAS_Template-for-SinglePlayer-2D](https://github.com/Qhax11/GAS_Template-for-SinglePlayer-2D).
Since it's still a work in progress, and not all systems are production-ready, the full source code is not shared publicly.

Instead, this repository focuses on showcasing the core gameplay systems, architecture decisions, and implementation logic through:

- Isolated code snippets
- Technical breakdowns
- Gameplay videos and visuals

The goal is to demonstrate modular design, combat-focused GAS usage, and single-player action mechanics in Unreal Engine.

The following gameplay video is from an early prototype using an older version of the system.
While it's outdated, it still provides a good idea of the core gameplay direction and what I'm trying to achieve.

Gameplay video: 

https://github.com/user-attachments/assets/ad2094bd-ed0a-4734-b12d-c078d2cd2895

[https://www.youtube.com/watch?v=_B-iSA1eJA4&ab_channel=%C5%9Eamil%C3%96zel](https://www.youtube.com/watch?v=rrsV4eMGDK8)


## Contents
- [Gameplay Systems](#Gameplay-Systems)
     - [Target Lock System](#1-Target-Lock-System)
- [Combat Abilities](#Combat-Abilities)
     - [Combo Abilities](#1-Combo-Abilities) 
     - [Parry](#2-Parry)
     - [Shadow Attack](#3-Shadow-Attack)
     - [Shadow Finisher](#3-Shadow-Finisher)
- [AI](#AI)
     - [The Core Architecture and Control Flow](#1-The-Core-Architecture-and-Control-Flow)
          - [State Manager](#11-State-Manager)
          - [States](#12-States)
               - [Base State](#121-Base-State)
               - [Movement State](#122-Movement-State)
               - [Attack State](#123-Attack-State)
               - [InComingAttack State](#124-InComingAttack-State)
          - [Intend Handler](#13-Intend-Handler)
     - [The Behavior Decision and Scoring](#2-Behavior-Decision-and-Scoring)
          - [Behavior Decision](#21-Behavior-Decision)
          - [Services](#22-Services)
               - [Service Base](#221-Service-Base)
               - [Get Best Movement](#222-Get-Best-Movement)
               - [Get Best Attack Base](#223-Get-Best-Attack)
               - [Get Best IncomingAttack Reaction](#224-Get-Best-IncomingAttack-Reaction)
     - [The Execution Layer](#3-The-Execution-Layer)
          - [Movement System](#31-Get-Best-IncomingAttack-Reaction)
          - [Movement Abilities](#32-Movement-Abilities)
               - [Chase Target](#321-Chase-Target)
               - [Strafing](#322-Strafing)
     - [Contextual and External Systems](#4-Contextual-and-External-Systems)
          - [Crowd Manager](#41-Crowd-Manager)
         
## Gameplay Systems

### **1. Target Lock System**
Locking onto enemies and switching targets dynamically during combat. Includes target highlighting, camera adjustments, and orientation logic.

This system allows the player to lock onto enemies, rotate the camera and character toward the current target, and dynamically switch targets using horizontal mouse movement. The system is fully modular and built on top of GAS and input actions.

In-game preview:

https://github.com/user-attachments/assets/86938f1b-b053-4e41-8f8f-916d75851006

When activated:
- The system performs a trace to find nearby hostile actors.
- Gameplay tags are applied to both the hero and target to reflect the lock state.
```c++
void UAC_TargetLockSystem::StartTargetLock()
{
	if (!TracingDataStart)
	{
		UE_LOG(LogTemp, Warning, TEXT("TargetingData is null in: %s, cannot initialize TargetLockSystem."), *GetName());
		return;
	}

	TArray<AActor*> OutResultActors;
	TracingDataStart->Trace->CreateTraceWithTeamFilter(GetWorld(), HeroBase, ETeamAttitude::Hostile, OutResultActors);

	if (!OutResultActors.IsValidIndex(0))
	{
		return;
	}

	ChangeTarget(OutResultActors[0]);

	HeroASC->AddLooseGameplayTag(GAS_Tags::TAG_Gameplay_State_TargetLockSystem_Hero_TargetLocked);
	bLocked = true;
	SetComponentTickEnabled(true);

	OnStartTargetLock.Broadcast();
}

void UAC_TargetLockSystem::EndTargetLock()
{
	if (!CurrentTargetASC)
	{
		UE_LOG(LogTemp, Warning, TEXT("CurrentTargetASC is null in %s"), *GetName());
		return;
	}

	HeroASC->RemoveLooseGameplayTag(GAS_Tags::TAG_Gameplay_State_TargetLockSystem_Hero_TargetLocked);
	CurrentTargetASC->RemoveLooseGameplayTag(GAS_Tags::TAG_Gameplay_State_TargetLockSystem_Enemy_Targeted);
	CurrentTarget = nullptr;
	bLocked = false;

	OnEndTargetLock.Broadcast();

	SetComponentTickEnabled(false);
}
```
- Once a target is found, both the camera and character smoothly rotate toward it every tick.
- While locked, players can switch targets left/right based on the horizontal input axis.

```c++
void UAC_TargetLockSystem::LookMouse(const FInputActionValue& Value)
{
	if (!bLocked || HeroASC->HasMatchingGameplayTag(GAS_Tags::TAG_Gameplay_State_AbilityTargeting))
	{
		return;
	}

	const FVector2D VectorValue = Value.Get<FVector2D>();

	// Cooldown mechanism: Each direction can only trigger the action once per second.
	const float CurrentTime = GetWorld()->GetTimeSeconds(); 

	if ((VectorValue.X > Threshold) && (CurrentTime - TryToFindNewTargetLastExecutionTimeRight >= TryToFindNewTargetExecutionCooldown))
	{
		TryToFindNewTarget(ETargetChangeDirection::TCD_Right);
		TryToFindNewTargetLastExecutionTimeRight = CurrentTime;
	}

	if ((VectorValue.X < -Threshold) && (CurrentTime - TryToFindNewTargetLastExecutionTimeLeft >= TryToFindNewTargetExecutionCooldown))
	{
		TryToFindNewTarget(ETargetChangeDirection::TCD_Left);
		TryToFindNewTargetLastExecutionTimeLeft = CurrentTime;
	}
}

void UAC_TargetLockSystem::TryToFindNewTarget(TEnumAsByte<ETargetChangeDirection> TargetChangeDirection)
{
	if (!TracingDataTargetChange || !TracingDataCheckForFrontActor)
	{
		UE_LOG(LogTemp, Warning, TEXT("TracingDataTargetChange or TracingDataCheckForFrontActor is null in: %s"), *GetName());
		return;
	}

	if (!HeroBase) 
	{
		UE_LOG(LogTemp, Warning, TEXT("HeroBase is null in: %s"), *GetName());
		return;
	}

	TArray<AActor*> OutResultActors;
	TracingDataTargetChange->Trace->CreateTraceWithTeamFilter(GetWorld(), HeroBase, ETeamAttitude::Hostile, OutResultActors);

	OutResultActors.Remove(CurrentTarget);

	if (OutResultActors.IsEmpty())
	{
		return;
	}

	TArray<AActor*> LeftActors;
	TArray<AActor*> RightActors;

	SplitActorsByPositionRelativeToHero(OutResultActors, LeftActors, RightActors);

	AActor* FoundNewTarget = nullptr;
	if (TargetChangeDirection == ETargetChangeDirection::TCD_Left) 
	{
		FoundNewTarget = FindNearestActor(CurrentTarget, LeftActors);
	}
	else if(TargetChangeDirection == ETargetChangeDirection::TCD_Right)
	{
		FoundNewTarget = FindNearestActor(CurrentTarget, RightActors);
	}

	// If there is an enemy directly in the player's line of sight (viewing direction), we select it as the new target
    // even if it's further away than the current target. This prioritizes enemies in front of the player,
    // ensuring a more dynamic target selection based on the player's perspective.
	if (FoundNewTarget)
	{
		TArray<AActor*> CheckForFrontActors;
		FRotator LookAtRotation = UKismetMathLibrary::FindLookAtRotation(HeroBase->GetActorLocation(), FoundNewTarget->GetActorLocation());
		TracingDataCheckForFrontActor->Trace->CreateTraceWithTeamFilterWithDirection(GetWorld(), HeroBase, ETeamAttitude::Hostile, LookAtRotation, CheckForFrontActors);
		if (CheckForFrontActors.IsValidIndex(0)) 
		{
			// If the actor in front of the player is different from the current target, set it as the new target.
			if (CheckForFrontActors[0] != CurrentTarget) 
			{
				FoundNewTarget = CheckForFrontActors[0];
			}
		}
	}
	
	ChangeTarget(FoundNewTarget);
}
```
- If the current target dies (despawns), the lock automatically ends.

```c++
void UAC_TargetLockSystem::OnEnemyDeSpawn(AGAS_CharacterBase* Enemy)
{
	if (Enemy == CurrentTarget) 
	{
		EndTargetLock();
	}
}
```


## Combat Abilities
Characters can take damage from various sources, including environmental hazards and enemy attacks. Players can also deal damage using abilities or other gameplay mechanisms. The system includes UI elements that display the amount of damage taken or dealt, providing feedback to the player during combat.

### **1. Combo Abilities**
The combo system is designed to handle sequenced melee attacks, using a data-driven structure that supports chaining multiple abilities in order. Each combo chain is defined in a Data Asset, which specifies the ability class, montage section, and range.

In-game preview:

https://github.com/user-attachments/assets/0308b885-7e32-4ee0-942d-bd06d70460e6



When a combo starts:

- A ComboChainTracker is initialized with the selected ability sequence.

 ```c++
USTRUCT()
struct FActiveComboChainTracker
{
	GENERATED_BODY()

	UPROPERTY()
	FComboChainData ComboChain = FComboChainData();

	UPROPERTY()
	int32 CurrentIndex = 0;

	UPROPERTY()
	TSubclassOf<UGA_ComboMeleeAttack> CurrentAbilityClass = nullptr;

	UPROPERTY()
	UGameplayAbility* CurrentAbilityInstance = nullptr;

	UPROPERTY()
	FGameplayAbilitySpecHandle CurrentAbilitySpecHandle;

	UPROPERTY()
	bool bNextAttackAllowed = true;

	bool IsCurrentComboValid() const
	{
		return ComboChain.ComboAbilities.IsValidIndex(CurrentIndex);
	}

	bool IsChainFinished() const
	{
		return CurrentIndex >= ComboChain.ComboAbilities.Num();
	}

	const FComboAbilityData* GetCurrentCombo() const
	{
		return IsCurrentComboValid() ? &ComboChain.ComboAbilities[CurrentIndex] : nullptr;
	}

	void Advance()
	{
		++CurrentIndex;
	}

	void Reset()
	{
		CurrentIndex = 0;
	}
};

void UAC_HeroMeleeComboManager::InitComboChainTracker()
{
	if (!ComboChainAsset || !ComboChainAsset->ComboChains.IsValidIndex(SelectedComboIndex))
	{
		return;
	}

	ActiveComboChainTracker.ComboChain = ComboChainAsset->ComboChains[SelectedComboIndex];
	ActiveComboChainTracker.CurrentIndex = 0;

	const FComboAbilityData* FirstCombo = ActiveComboChainTracker.GetCurrentCombo();
	if (FirstCombo)
	{
		ActiveComboChainTracker.CurrentAbilityClass = FirstCombo->ComboAbilityClass;
	}
}
```

- Each attack is a separate UGA_ComboMeleeAttack derived ability.

- ![image](https://github.com/user-attachments/assets/6b1d9c79-12cf-475f-8197-fe3aeb29f019)

- After an animation event is triggered (e.g. “CanExecuteNext”), the system allows activating the next ability in the chain.

```c++
void UGA_ComboMeleeAttack::OnEventReceived(FGameplayTag EventTag, FGameplayEventData EventData)
{
	if (EventTag == GAS_Tags::TAG_Gameplay_AttackEvent_CanActivateNextAttack) 
	{
		OnCanExecuteNextAttack.Broadcast();
	}
	else 
	{
		Super::OnEventReceived(EventTag, EventData);
	}
}
```

- If the chain completes or is interrupted, it resets.

```c++
void UAC_HeroMeleeComboManager::OnComboMeleeAttackAbilityEnd(const FAbilityEndedData& EndedData)
{
	if (EndedData.AbilitySpecHandle != ActiveComboChainTracker.CurrentAbilitySpecHandle)
	{
		return;
	}

	if (EndedData.bWasCancelled)
	{
		if (ActiveComboChainTracker.IsChainFinished())
		{
			ActiveComboChainTracker.Reset();
			OnComboEnded.Broadcast();
		}
	}
	// If ComboMelee ability ended as normal
	else if(!EndedData.bWasCancelled)
	{
		ActiveComboChainTracker.Reset();
		OnComboEnded.Broadcast();
	}

	ActiveComboChainTracker.bNextAttackAllowed = true;
}
```
  
### **2. Parry**
The parry system allows the player to negate incoming attacks by timing a defensive ability just before getting hit.
If the parry is successful, the attacker is knocked back or staggered through a triggered follow-up ability.

In-game preview:


https://github.com/user-attachments/assets/8b71a822-1348-4409-87a1-5e5ed62223c6



The system works as follows:

The parry ability (UGA_ParryBase) applies a GameplayTag that marks the character as "parrying".

```c++
UGA_ParryBase::UGA_ParryBase()
{
	FGameplayTagContainer AbiltiyTags;
	AbiltiyTags.AddTag(GAS_Tags::TAG_Gameplay_Ability_Parry);
	SetAssetTags(AbiltiyTags);

	// This tag is used to detect parry state in Execution Calculation.
	ActivationOwnedTags.AddTag(GAS_Tags::TAG_Gameplay_State_InCombat_Parry);
}
```

During the attack’s ExecutionCalculation, if the target has the parry tag, a special check is performed via CalculateParry.

```c++
if (Params.TargetASC->HasMatchingGameplayTag(GAS_Tags::TAG_Gameplay_State_InCombat_Parry))
{
	if (CalculateParry(Params))
	{
		TriggerGameplayEvent(Params, GAS_Tags::TAG_Gameplay_AbilityTriggerEvent_ParryKnockback);
		return;
	}
}
```

If the check succeeds, a Gameplay Event is triggered, allowing the parry ability to process the result (e.g. knockback, posture break, slow-mo).

```c++
void UGA_ParryKnockbackBase::ActivateAbility(const FGameplayAbilitySpecHandle Handle, 
	const FGameplayAbilityActorInfo* OwnerInfo, 
	const FGameplayAbilityActivationInfo ActivationInfo, 
	const FGameplayEventData* TriggerEventData)
{
	Super::ActivateAbility(Handle, OwnerInfo, ActivationInfo, TriggerEventData);

	if (!ParryKnockbackEffect)
	{
		return;
	}

	FGameplayEffectSpecHandle EffectSpecHandle = GetAbilitySystemComponentFromActorInfo()->MakeOutgoingSpec(ParryKnockbackEffect, 1.0f, TriggerEventData->ContextHandle);
	if (!EffectSpecHandle.IsValid())
	{
		return;
	}

	GetAbilitySystemComponentFromActorInfo()->ApplyGameplayEffectSpecToSelf(*EffectSpecHandle.Data);

	BP_ApplyForce(TriggerEventData->Instigator);
}
```

### **3. Shadow Attack** 

ShadowAttack is a special ability that allows the player character to instantly teleport to a selected location or enemy, then trigger a follow-up attack using a pre-configured gameplay ability from the HeroShadowTargetActor.


- Ability Flow
- Activation Phase

ActivationOwnedTags includes AbilityTargeting_Shadow to prevent other input reactions during targeting.

SpawnAndSetupTargetActor() is called to place the shadow target in the world based on lock-on state or line tracing.

```c++
void UGA_HeroShadowAttack::SpawnAndSetupTargetActor(FRotator Rotation, FVector Location)
{
    AGAS_HeroBase* HeroBase = Cast<AGAS_HeroBase>(GetAvatarActorFromActorInfo());
    if (!HeroBase || !TraceData) 
    {
        UE_LOG(LogTemp, Warning, TEXT("HeroBase or TraceData is null in: %s"), *GetName());
        Super::SpawnAndSetupTargetActor(Rotation, Location);
        return;
    }

    FVector ShadowSpawnLocation = Location;

    if (HeroBase->GetAbilitySystemComponent()->HasMatchingGameplayTag(GAS_Tags::TAG_Gameplay_State_TargetLockSystem_Hero_TargetLocked))
    {
        FVector CurrentTargetLocation = HeroBase->GetTargetLockSystemComponent()->CurrentTarget->GetActorLocation();
        FRotator LookAtRotation = UKismetMathLibrary::FindLookAtRotation(ShadowSpawnLocation, CurrentTargetLocation);
        Super::SpawnAndSetupTargetActor(FRotator(0, LookAtRotation.Yaw, 0), ShadowSpawnLocation);

        SetShadowToShadowController(); 
        HeroBase->GetHeroShadowControllerComponent()->SetShadowLocationWithCumulativeMouseValuesTargetLocked();
    }
    else
    {
        ShadowSpawnLocation = HeroBase->GetHeroShadowControllerComponent()->GetHeroShadowLocationFromLineTrace();

        TArray<AActor*> OutResultActors;
        TraceData->Trace->CreateTraceWithTeamFilterWithLocation(GetWorld(), HeroBase, ETeamAttitude::Hostile, ShadowSpawnLocation, OutResultActors);

        if (OutResultActors.IsValidIndex(0))
        {
            FRotator LookAtRotation = UKismetMathLibrary::FindLookAtRotation(ShadowSpawnLocation, OutResultActors[0]->GetActorLocation());
            Super::SpawnAndSetupTargetActor(FRotator(0, LookAtRotation.Yaw, 0), ShadowSpawnLocation);
        }
        else
        {
            Super::SpawnAndSetupTargetActor(HeroBase->GetActorRotation(), ShadowSpawnLocation);
        }
        //SetShadowToShadowController();
    }
}
```

- Target Selection Phase

If lock-on is active:

Shadow actor is placed near the current enemy target.


If no lock-on:

A trace is performed to find potential targets.

Shadow target is stored in the Shadow Controller component for later use.

- Confirmation Phase

When player confirms the shadow location:

The player teleports to the shadow actor's position and rotation.

AbilityTargeting_Shadow tag is manually removed early to block further combo input while attack plays.

If the HeroShadowTargetActor has a valid AbilityClass, that ability (e.g., a damage or animation ability) is triggered.

BP_OnTargetActorConfirm() is called for Blueprint integration or visual scripting.

- In-game preview for player: 
 
https://github.com/user-attachments/assets/f8cd5cb3-24e0-4640-a64c-c19014647c26

- In-game preview for AI: 


https://github.com/user-attachments/assets/52071bd4-a53a-4ec6-a5d6-25605eae1345

## AI
`Developing a robust and intelligent AI for a fast-paced combat system was one of the most challenging aspects of this project.` To meet the demands of dynamic combat and achieve granular control, I created a custom, data-driven system. This architecture provides several key advantages:

- <ins>Granular Control:</ins> Offers `100% control` over the enemy's decision-making process, ensuring intelligent, context-aware actions in combat.

- <ins>Modular Behavior:</ins> Utilizes a custom State Machine framework, where individual behaviors are self-contained and easily modified or extended.

- <ins>Data-Driven Transitions:</ins> All state changes are managed by external data assets, simplifying debugging and balancing across various enemy types.

- <ins>Clean Flow:</ins> The system separates the `Control Flow (Manager)` from the `Behavior Logic (States)`, resulting in a cleaner and more maintainable codebase.

This section details the custom architecture I developed, starting with the evolution of my AI design philosophy and the core structure that governs the enemy's behavior.

## **1. The Core Architecture and Control Flow** 
My journey to this final system began by experimenting with off-the-shelf solutions. I started with traditional `Behavior Trees`, which offered flexibility but quickly became unwieldy for managing complex data flow in combat. I then moved to `State Trees`, which improved structure but still fell short on scalability. Eventually, I explored a `hybrid approach`, attempting to combine the strengths of both.

However, none of these provided the necessary granular control, complex data flow management, and sophisticated debugging capabilities required. Ultimately, this led to the decision to build a `custom, data-driven state machine` to achieve 100% control over the AI's behavior. This system allows for precise management of complex states and transitions, ensuring the AI can make intelligent, context-aware decisions in combat, leading to a more challenging and engaging gameplay experience.

In this section, I detail the architectural evolution and the fundamental structure of the AI system. This architecture is built around two primary custom classes that provide the necessary control and modularity:

- <ins>State Manager (UAC_StateManager):</ins> The central component responsible for orchestrating the overall control flow and managing transitions between states.

- <ins>State Classes (UStateBase):</ins> Individual, self-contained modules responsible for executing the specific behavior logic of the AI (e.g., Attack, Movement, InComingAttack).

### **1.1 State Manager** 
The `UAC_StateManager` component serves as the core of the AI's behavioral system, acting as a custom state machine that orchestrates all of the enemy character's actions. Unlike traditional systems with hard-coded state logic, this manager handles state transitions and manages the flow of behavior by creating and running instances of the UStateBase class. This approach ensures a modular and clean structure, where each state's logic is entirely self-contained.

Key Functions and Logic:

- <ins>State Instantiation & Initialization:</ins> The manager takes a list of state classes from a data asset and creates a single instance of each at `BeginPlay`. These instances are then initialized with a single `FStateInitParams` struct that contains references to all other necessary components, ensuring the states have access to everything they need to function.
```c++
void UAC_StateManager::CreateStates()
{
	FStateInitParams StateInitParams = FStateInitParams(OwnerEnemyBase, OwnerController, OwnerEnemyASC, 
		EnemyTagDelegatesComponent, BehaviorDecisionComponent, OwnerController->GetTargetActor(), 
		Cast<UGAS_AbilitySystemComponent>(OwnerController->GetTargetHero()->GetAbilitySystemComponent()), this);

	for (TSubclassOf<UStateBase> StateClass : StateClassArray)
	{
		if (!*StateClass)
		{
			UE_LOG(LogTemp, Warning, TEXT("Invalid state class in array."));
			continue;
		}

		UStateBase* NewState = NewObject<UStateBase>(this, StateClass);
		if (!NewState || !NewState->StateTag.IsValid())
		{
			UE_LOG(LogTemp, Warning, TEXT("Failed to instantiate: %s"), *StateClass->GetName());
			continue;
		}

		NewState->StateInitalize(StateInitParams);
		StateInstances.Add(NewState);
	}
}
```

- <ins>State Transitions:</ins> The `RequestStateTreeEnter()` and `RequestStateTreeExit()` functions are the sole entry points for changing states. They validate the transition using the `EnterCondition()` and `ExitCondition()` checks on the new and current states, respectively. If the checks pass, the manager correctly calls `OnExit()` on the old state before calling `OnEnter()` on the new one, ensuring a clean and safe transition.
```c++
void UAC_StateManager::RequestStateTreeEnter(const FGameplayTag& StateTag)
{
	if (!StateTag.IsValid() || !bActive)
	{
		return;
	}

	if (bEnableDebug)
	{
		UE_LOG(LogTemp, Warning, TEXT("[State Manager]: %s state has been requested to enter"), *StateTag.ToString());
	}

	UStateBase* FindedState = GetStateWithTag(StateTag);
	if (!FindedState) 
	{
		UE_LOG(LogTemp, Warning, TEXT("[State Manager]: %s FindedState is null!"));
		return;
	}

	if (FindedState->EnterCondition())
	{
		if (CurrentState)
		{
			CurrentState->OnExit();
		}

		FindedState->OnEnter();
		CurrentState = FindedState;
		return;
	}
	else
	{
		if (bEnableDebug)
		{
			UE_LOG(LogTemp, Warning, TEXT("[State Manager]: Condition of %s is false, cannot enter"), *FindedState->GetName());
		}
	}
}
```
```c++
void UAC_StateManager::RequestStateTreeExit(const FGameplayTag& StateTag, const FGameplayTag& TransactionTag, FString Reason)
{
	if (!StateTag.IsValid() || !bActive)
	{
		return;
	}

	if (bEnableDebug)
	{
		UE_LOG(LogTemp, Warning, TEXT("[State Manager]: %s state has been requested to exit, reason is: %s"), *StateTag.ToString(), *Reason);
	}

	UStateBase* FindedState = GetStateWithTag(StateTag);
	if (!FindedState)
	{
		UE_LOG(LogTemp, Warning, TEXT("[State Manager]: %s FindedState is null!"));
		return;
	}

	if (!FindedState->ExitCondition()) 
	{
		if (bEnableDebug)
		{
			UE_LOG(LogTemp, Warning, TEXT("[State Manager]: Condition of %s is false, cannot exit"), *StateTag.ToString());
		}
		return;
	}

	// If request coming with trancastion tag we directly enter
	if (TransactionTag.IsValid()) 
	{
		RequestStateTreeEnter(TransactionTag);
		return;
	}

	FAttackData NewSelectedAttack = SelectNewBestAttack();
	if (IsAttackInRange(NewSelectedAttack.AbilityClass))
	{
		RequestStateTreeEnter(GAS_Tags::TAG_AI_State_Attack);
	}
	else
	{
		RequestStateTreeEnter(GAS_Tags::TAG_AI_State_Movement);
	}
}
```

### **1.2 States** 
The entire philosophy of the custom AI system is built upon the isolation and `encapsulation of behavior` into distinct `State` objects. This approach dictates that the logic for deciding what to do and how to do it is entirely contained within the state instance itself, achieving complete decoupling from the central `StateManager`. Every action, from the most basic repositioning to the most complex counter-attack, is handled by a dedicated State. The Manager's only job is to direct control flow to the currently active State; the State, in turn, is responsible for executing its designated behavior and requesting the next necessary transition.

#### **1.2.1 Base State** 
All concrete behavioral states inherit from the `UStateBase` class, the foundational C++ blueprint that provides the common structure and defines the critical lifecycle for every state. This design is paramount, ensuring that all behaviors, regardless of complexity, function consistently and predictably within the central framework.

The reliable execution of State logic hinges on proper `initialization`. This is achieved through the `FStateInitParams` struct, which is passed to a state upon creation. This struct contains a consolidated and pre validated list of key pointers such as the `Enemy`, `EnemyController`, and `BehaviorDecisionComponent` that the state will need to perform its logic. This singular initialization step is vital, as it prevents states from having to manually find and validate these references during runtime, guaranteeing a clean and immediate operational readiness.
```c++
USTRUCT()
struct FStateInitParams
{
    GENERATED_BODY()

public:
    // Core character references
    UPROPERTY()
    AGAS_EnemyBase* Enemy = nullptr;
    UPROPERTY()
    AAIControllerBase* EnemyController = nullptr;
    UPROPERTY()
    UGAS_AbilitySystemComponent* EnemyASC = nullptr;

    // Specialized component and target references
    UPROPERTY()
    UAC_BehaviorDecision* BehaviorDecisionComponent = nullptr;
    UPROPERTY()
    AActor* HeroTarget = nullptr;
    UPROPERTY()
    UGAS_AbilitySystemComponent* HeroTargetASC = nullptr;
    // ... (rest of the struct body)
};
```

The core API for interaction and management is defined by a set of `Key Virtual Functions` that map directly to the AI's execution cycle:

- <ins>OnEnter():</ins> Executed immediately when the AI transitions into this state. This is the activation point where crucial initialization logic is handled, such as binding delegates or halting prior movement.

- <ins>OnTick(float DeltaTime):</ins> This function serves as the State's primary update loop, called every frame while the AI is in this state. It is utilized for continuous checks and updates, primarily monitoring distance, time-sensitive events, or evaluating exit conditions.

- <ins>OnExit():</ins> Called just before the AI leaves the state. This function is solely responsible for the essential cleanup logic, ensuring that anything initiated in `OnEnter()` or during execution (like unbinding delegates or resetting temporary variables) is safely terminated.

- <ins>EnterCondition() and ExitCondition():</ins> These virtual functions provide an additional, powerful layer of `self-governance`. They allow the state itself to dynamically check if the tactical conditions are currently right for it to be safely entered or exited, providing a crucial safety net for complex state transitions.

In essence, UStateBase is the contract for behavior, defining the rigorous API that the StateManager uses to interact with and manage all the different behavioral implementations.

#### **1.2.2 Movement State** 
The `UMovementState` is one of the most dynamic states within the AI system. Its primary purpose is to manage the AI's movement, ensuring it gets into the optimal position to execute a pre-selected attack. It is highly reactive and continuously evaluates the tactical situation to find the most suitable movement chain.

- <ins>OnEnter & Attack Selection:</ins> When the AI enters this state, it immediately calls `SelectNewAttackAbility()`. This is a crucial initial step, as the chosen attack's range directly dictates which movement chain `StartMovementChain()` the AI needs to perform.
```c++
void UMovementState::OnEnter_Implementation()
{
	Super::OnEnter_Implementation();
	SelectedAttack = SelectNewAttackAbility();
	if (!SelectedAttack.AbilityClass)
	{
		ExitRequest("SelectedAttack ability class is null");
		return;
	}
	SelectedAttackCDO = SelectedAttack.AbilityClass->GetDefaultObject<UGAS_GameplayAbilityBase>();
	StartMovementChain(SelectedAttack.AbilityClass);
}
```

- <ins>Transition Logic:</ins> The `TryEnterToAttackState()` function is the heart of this state. It checks if the AI is in range of its selected attack. If the condition is met, it immediately stops all movement abilities and requests to exit the `MovementState` and enter the `AttackState`, ensuring a seamless transition.
```c++
void UMovementState::OnTick_Implementation(float DeltaTime)
{
	TryEnterToAttackState();
}

void UMovementState::TryEnterToAttackState()
{
	if (!SelectedAttackCDO) return;
	if (IsAttackInRange(SelectedAttack.AbilityClass))
	{
		MovementManagerComponent->StopMovementAbilities();
		ExitRequest("Target is in range", GAS_Tags::TAG_AI_State_Attack);
	}
}
```

#### **1.2.3 Attack State** 
The `UAttackStateBase` is where the AI's offensive actions are managed. Once the AI has successfully positioned itself within a suitable range, this state takes over to execute a pre-selected attack ability. This state also handles the entire lifecycle of the attack, from activation to completion, ensuring a fluid and responsive combat experience.

- <ins>OnEnter & Attack Execution:</ins> Upon entering the `AttackState`, the AI first calls `StopMovementAbilities()` to halt any ongoing movement. It then immediately calls `SelectAndMakeAttack()`, which attempts to activate the corresponding ability from the `Gameplay Ability System (GAS)`.
```c++
void UAttackStateBase::OnEnter_Implementation()
{
	Super::OnEnter_Implementation(); 
	Enemy->GetEnemyMovementManagerComponent()->StopMovementAbilities();
	SelectAndMakeAttack();
}
```

- <ins>Dynamic Binding:</ins> The `MakeAttack()` function is a key part of the process. It first attempts to activate the ability and, if successful, dynamically `binds a callback` to the ability's `OnGameplayAbilityEndedWithDataBP` delegate. This ensures the `OnAttackAbilityEnded()` function is called at the precise moment the attack concludes.
```c++
void UAttackStateBase::MakeAttack()
{
	UGAS_GameplayAbilityBase* ActivatedAbility = EnemyASC->TryActivateAbilityByClassAndReturnInstance(
		BehaviorDecisionComponent->LastSelectedAttackAbilityData.AbilityClass);

	if (ActivatedAbility)
	{
		if (!ActivatedAbility->OnGameplayAbilityEndedWithDataBP.IsAlreadyBound(this, &UAttackStateBase::OnAttackAbilityEnded))
		{
			ActivatedAbility->OnGameplayAbilityEndedWithDataBP.AddDynamic(this, &UAttackStateBase::OnAttackAbilityEnded);
		}
		LastUsedAttack = ActivatedAbility;
	}
}

void UAttackStateBase::OnAttackAbilityEnded(const FAbilityEndedDataBP& DodgeAbilityEndedData)
{
	ExitRequest("OnAttackAbilityEnded");
}
```

- <ins>Cleanup on Exit:</ins> The `OnExit()` function is crucial for preventing memory leaks and unwanted behavior. It checks if the `OnGameplayAbilityEndedWithDataBP` delegate is still bound and, if so, unbinds it. It also clears the `LastUsedAttack` reference. This robust cleanup ensures the state is ready for its next use.
```c++
void UAttackStateBase::OnExit_Implementation()
{
	Super::OnExit_Implementation();

	if (LastUsedAttack)
	{
		if (LastUsedAttack->OnGameplayAbilityEndedWithDataBP.IsAlreadyBound(this, &UAttackStateBase::OnAttackAbilityEnded))
		{
			LastUsedAttack->OnGameplayAbilityEndedWithDataBP.RemoveDynamic(this, &UAttackStateBase::OnAttackAbilityEnded);
		}
		LastUsedAttack = nullptr;
	}
}
```

#### **1.2.4 InComingAttack State** 
The `UInComingAttackState` is the AI's reactive, defensive state. It's triggered by the `Intend Handler` when the AI detects a significant incoming attack and must make a split-second decision on how to react.

- <ins>Decision and Action:</ins> The core of this state is the `SelectAndMakeInComingAttackReaction()` function, which delegates the decision-making to the `Behavior Decision Component`. Based on the highest-scoring defensive reaction (e.g., parry, take damage), it then calls the appropriate function to execute that action.
```c++
void UInComingAttackState::OnEnter_Implementation()
{
    Super::OnEnter_Implementation();
    SelectAndMakeInComingAttackReaction();
}

bool UInComingAttackState::SelectAndMakeInComingAttackReaction()
{
	UBDS_ComingAttackReactionBase* SelectedBestReaction = BehaviorDecisionComponent->LastSelectedComingAttackReaction;
	if (!SelectedBestReaction) 
	{
		UE_LOG(LogTemp, Warning, TEXT("SelectedBestReaction is null in: %s"), *GetName());
		return false;
	}

	if (SelectedBestReaction->ReactionType == EComingAttackReaction::TakeDamage)
	{
		MakeTakeDamage(SelectedBestReaction);
		return true;
	}
	else if(SelectedBestReaction->ReactionType == EComingAttackReaction::Parry)
	{
		MakeParryAbility(SelectedBestReaction);
		return true;
	}

	return false;
}
```

- <ins>Delegates for Event Handling:</ins> The state uses delegates extensively to manage its flow. For a Take Damage reaction, it binds to the `OnDamageDealt` delegate to trigger the AI's own damage-receiving ability. For a Parry reaction, it listens for the `OnGameplayAbilityEndedWithDataBP` delegate of the parry ability to know when it's safe to exit the state.

- <ins>Execution and Transition:</ins> The `MakeParryAbility()` and `MakeTakeDamage()` functions are responsible for activating the relevant Gameplay Abilities. Once the defensive ability is complete or the damage is processed, the state calls `ExitRequest`, allowing the AI to seamlessly transition back to its primary combat loop.
```c++
void UInComingAttackState::OnTakeDamageAbilityEnded(const FAbilityEndedDataBP& DodgeAbilityEndedData)
{
	ExitRequest("OnTakeDamageAbilityEnded");
}
```

- <ins>Robust Cleanup:</ins> The `OnExit()` function is critical for maintaining a clean and bug-free system. It unbinds all delegates that were used during the state's execution, preventing unintended behavior or memory leaks. This includes delegates from the `DamageSubsystem` and any activated abilities, ensuring the state is fully reset and ready for its next use.
```c++
void UInComingAttackState::OnExit_Implementation()
{
	Super::OnExit_Implementation();

	if (LastUsedTakeDamageAbility)
	{
		if (LastUsedTakeDamageAbility->OnGameplayAbilityEndedWithDataBP.IsAlreadyBound(this, &UInComingAttackState::OnTakeDamageAbilityEnded))
		{
			LastUsedTakeDamageAbility->OnGameplayAbilityEndedWithDataBP.RemoveDynamic(this, &UInComingAttackState::OnTakeDamageAbilityEnded);
			UE_LOG(LogTemp, Warning, TEXT("State Manager: %s ability's end bind is removed."), *LastUsedTakeDamageAbility->GetName());
		}
	}

	if (LastUsedParryAbility)
	{
		if (LastUsedParryAbility->OnGameplayAbilityEndedWithDataBP.IsAlreadyBound(this, &UInComingAttackState::OnParryAbilityEnded))
		{
			LastUsedParryAbility->OnGameplayAbilityEndedWithDataBP.RemoveDynamic(this, &UInComingAttackState::OnParryAbilityEnded);
			UE_LOG(LogTemp, Warning, TEXT("State Manager: %s ability's end bind is removed."), *LastUsedParryAbility->GetName());
		}
	}
}
```

### **1.3 Intend Handler** 
The `UAC_IntendHandlerBase` component serves as the AI's `perceptual layer`, acting as a bridge between environmental stimuli and the AI's decision-making process. Its primary purpose is to identify critical moments—such as detecting a player, becoming vulnerable, or detecting an incoming attack—and to notify the `State Manager` to initiate a new behavioral sequence.

The Intend Handler achieves this by subscribing to various delegates and events, rather than constantly polling for changes. This `event-driven approach` is highly performant and responsive, ensuring the AI can react instantly to dynamic combat situations.

Key Functions:

- <ins>Target Detection:</ins> Scans the environment for valid targets and alerts the `State Manager` when a new one is found.
```c++
void UAC_IntendHandlerBase::OnTargetDetected(AActor* DetectedTarget)
{
	OwnerStateManager->OnTargetDetected();
}
```

- <ins>Event-Driven State Changes:</ins> The `Intend Handler` is the primary trigger for reactive state transitions. It registers delegates for critical events:

- <ins>Vulnerable State:</ins> When the enemy's `Vulnerable` tag is added (e.g., after a parry or stagger), the `OnVulnerableTagAdded` delegate fires, immediately pushing the AI into a Vulnerable state.
```c++
void UAC_IntendHandlerBase::OnVulnerableTagAdded(const UAbilitySystemComponent* AbilitySystemComponent, const FGameplayTag& Tag)
{
	OwnerStateManager->RequestStateTreeEnter(GAS_Tags::TAG_AI_State_Vulnerable);
}
```

-<ins> Incoming Attack & Reaction Logic:</ins> This system is the AI's proactive defensive layer, managing how the AI responds to detected incoming attacks from the player. It is a critical example of the tight integration between the `Intend Handler`, `Behavior Decision`, and `State Manager component`.

- <ins>Detection and Analysis:</ins> When the player activates a melee attack, the `OnTargetAbilityActivated` function within the Intend Handler processes this event. It calculates the exact ComingAttackHitTime and combines all relevant tags into a FComingAttackPayload.

- <ins>Decision-Making:</ins> The `Intend Handler` sends this payload to the Behavior Decision Component, which uses its GetBestComingAttackReaction service to determine the highest-scoring defensive reaction (e.g., parry, take damage, etc.).

- <ins>Execution with Precision:</ins> The `SendEventToDefense` function takes over to execute the chosen reaction. If the optimal reaction requires a specific timing (e.g., parrying a few frames before the hit), it uses a timer to delay the reaction, ensuring the AI performs the action at the precise moment for maximum effectiveness.

```c++
void UAC_IntendHandlerBase::SendEventToDefense(FComingAttackPayload EventPayload)
{
	UBDS_ComingAttackReactionBase* BestReaction = OwnerBehaviorDecisionComp->GetBestComingAttackReaction(EventPayload);
	if (!BestReaction) 
	{
		UE_LOG(LogTemp, Warning, TEXT("BestReaction is null in: %s"), *GetName());
		return;
	}

	bool IsShadowAttack = EventPayload.ComingAttackTags.HasTagExact(GAS_Tags::TAG_Gameplay_Ability_Combat_Attack_MeleeCombo_ShadowLinked);
	if (BestReaction->ReactionType == EComingAttackReaction::TakeDamage)
	{
		TriggerIncomingAttackReaction(BestReaction, EventPayload);
		return;
	}

	const float PreferredDelay = EventPayload.ComingAttackHitTime - BestReaction->PreferredTriggerTimeBeforeHit;

	if (PreferredDelay <= 0.f)
	{
		TriggerIncomingAttackReaction(BestReaction, EventPayload);
	}
	else
	{
		FTimerHandle ReactionDelayTimer;
		GetWorld()->GetTimerManager().SetTimer(ReactionDelayTimer, FTimerDelegate::CreateUObject(
			this, &UAC_IntendHandlerBase::TriggerIncomingAttackReaction, BestReaction, EventPayload), PreferredDelay, false);

		UE_LOG(LogTemp, Warning, TEXT("IncomingAttack Reaction delayed by %.2f seconds."), PreferredDelay);
	}
}

void UAC_IntendHandlerBase::TriggerIncomingAttackReaction(UBDS_ComingAttackReactionBase* Reaction, FComingAttackPayload Payload)
{
	OwnerStateManager->ComingAttackPayload = Payload;
	OwnerStateManager->RequestStateTreeEnter(GAS_Tags::TAG_AI_State_InComingAttack);
}
```

## **2. The Behavior Decision and Scoring** 
The transition from the `Control Flow (I)` to this `Behavior Decision (II)` marks the point where the AI moves from knowing what state it is in to `deciding what tactical action to take next`. This mechanism is the source of the AI's "intelligence," ensuring that every action is context-aware and purposeful, rather than simply randomized.

### **2.1 Behavior Decision** 
The `UAC_BehaviorDecision` component is the AI's tactical layer, responsible for choosing the next action (attack, movement, or a combination of both) based on a dynamic scoring system. It operates within the `Combat state`, as triggered by the `StateManager`. This design ensures that the AI's actions are `context-aware and purposeful`.

### **2.2 Services** 
The tactical complexity of the AI system requires a `highly modular and scalable approach` to data analysis and decision-making. This is achieved through a `Service-Oriented Architecture`, where specific, specialized services handle distinct decision types.

#### **2.2.1 Service Base** 
The `UBehaviorDecisionServiceBase` class is the foundation for the AI's tactical decision-making system. It is a foundational abstract class designed to be inherited by specific services that handle different types of decisions, such as selecting an `attack` or a `movement chain`.

The class utilizes an `FBehaviorServiceInitParams` struct to initialize itself. This struct provides all the necessary references (like the `Enemy`, `EnemyController`, and `Hero`) upon creation, ensuring that each service has access to the data it needs without having to manually find it.

- <ins>Core Functionality:</ins> The base class provides common helper functions that all decision-making services might need. A key example is ApplyDirectionPoliciesToMovementAbility(), which determines the AI's movement direction based on pre-defined policies, such as moving away from the player's last direction or a random direction. This approach ensures that the logic for common tasks is centralized and reusable, preventing code duplication.

- <ins>Service-Oriented Architecture:</ins> This base class is a critical part of the system's `service-oriented` design. By inheriting from `UBehaviorDecisionServiceBase`, new decision-making services can be easily added to the AI. Each new service can implement its own specific logic while still benefiting from the shared initialization and helper functions provided by the base class. This modularity makes the system highly scalable and easy to maintain.

#### **2.2.2 Get Best Movement** 

The `UBDS_GetBestMovementChain` is a specialized service responsible for determining the optimal movement sequence for the AI. It operates as part of the Behavior Decision component, evaluating various movement "chains" based on a dynamic scoring system to select the most advantageous one for the current combat situation.

- <ins>Dynamic Scoring & Decision Logic:</ins> This service works by assigning a score to each potential `movement chain`. The chain with the highest final score is chosen. The score is calculated by combining multiple factors, each handled by its own dedicated function:
```c++
UMovementChainAsset* UBDS_GetBestMovementChain::GetBestMovementChain(TSubclassOf<UGAS_GameplayAbilityBase> SelectedAbilityClass)
{
    if (!SelectedAbilityClass || !AttackAbilityMovementChainMapAsset)
    {
        UE_LOG(LogTemp, Warning, TEXT("SelectedAbilityClass or AttackAbilityMovementChainMapAsset is null in: %s!"), *GetName());
        return nullptr;
    }

    UMovementChainAsset* BestMovementChainDataAsset = nullptr;

    float BestScore = -FLT_MAX;
    float BestMovementChainDistanceScore = 0.f;
    float BestMovementChainTargetMovementScore = 0.f;

    TArray<UMovementChainAsset*> AbilityMovementChainAssets = GetMovementChainsForSelectedAttackAbility(SelectedAbilityClass);
    if (AbilityMovementChainAssets.IsEmpty())
    {
        return nullptr;
    }

    for (UMovementChainAsset* MovementChainAsset : AbilityMovementChainAssets)
    {
        if (!MovementChainAsset) 
        {
            continue;
        }

        float DistanceScore = CalculateMovementChainScoreBasedOnTargetDistance(MovementChainAsset);
        float TargetMovementScore = CalculateMovementChainScoreBasedOnTargetMovement(MovementChainAsset);
        float BehaviorStateScore = CalculateMovementChainScoreBasedOnBehaviorState(MovementChainAsset);

        float TotalScore = MovementChainAsset->ScoreBias + DistanceScore + TargetMovementScore + BehaviorStateScore;

        UE_LOG(LogTemp, Log, TEXT("[AI] MovementChain %s → Score: %.2f"), *MovementChainAsset->MovementChainName.ToString(), TotalScore);

        if (TotalScore > BestScore)
        {
            BestScore = TotalScore;
            BestMovementChainDistanceScore = DistanceScore;
            BestMovementChainTargetMovementScore = TargetMovementScore;
            BestMovementChainDataAsset = MovementChainAsset;
        }
    }

    ApplyDirectionPoliciesToSelectedMovementChain(BestMovementChainDataAsset);
    /*
    if (GEngine && EnableSelectedDebug)
    {
        GEngine->AddOnScreenDebugMessage(10, 3.5f, FColor::Cyan,
            FString::Printf(TEXT(">> Selected MovementChain: %s | DistanceScore: %.1f | TargetMovementScore: %.1f "),
                *BestMovementChainDataAsset->MovementChainName.ToString(), BestMovementChainDistanceScore, BestMovementChainTargetMovementScore));
    }
    */
    return BestMovementChainDataAsset;
}
```

- <ins>Distance Scoring:</ins> `CalculateMovementChainScoreBasedOnTargetDistance()` evaluates how a movement chain's distance to the target affects its score. For instance, a movement chain designed for close-quarters combat might get a negative score if the enemy is far from the player, discouraging its selection. This is often handled by a curve asset, allowing for fine-grained control over the scoring.
```c++
float UBDS_GetBestMovementChain::CalculateMovementChainScoreBasedOnTargetDistance(UMovementChainAsset* MovementChainAsset)
{
    float Score = 0.0f;

    if (!HeroMovementListenerComp) 
    {
        return Score;
    }

    const float HeroDisplacement = HeroMovementListenerComp->GetDisplacementInLastSeconds(2.0f);

    if (HeroDisplacement > 50.0f)
    {
        Score += MovementChainAsset->ScoreModifierWhenTargetIsMoving;
    }
    else
    {
        Score += MovementChainAsset->ScoreModifierWhenTargetIsNotMoving;
    }

    // Penalty
    if (EnemyController->GetTargetHeroDistance() < MovementChainAsset->MinRange) 
    {
        Score = -100.0f;
    }

    return Score;
}
```

- <ins>Target Movement Scoring:</ins> `CalculateMovementChainScoreBasedOnTargetMovement()` adjusts the score based on whether the target (the player) is currently moving or standing still. This allows the AI to choose different movement patterns in response to a mobile or stationary player, making its behavior feel more intelligent and adaptive.

- <ins>Behavior State Modifiers:</ins> `CalculateMovementChainScoreBasedOnBehaviorState()` incorporates the AI's current BehaviorState (e.g., `Aggressive`, `Defensive`) into the score. This allows you to create different sets of movement chains that are preferred for specific tactical situations.
```c++
float UBDS_GetBestMovementChain::CalculateMovementChainScoreBasedOnBehaviorState(UMovementChainAsset* MovementChainAsset)
{
    float Score = 0.0f;

    if (!MovementChainAsset)
    {
        return Score;
    }

    if (const float* FoundScore = MovementChainAsset->BehaviorStateModifiers.Find(BehaviorState))
    {
        Score += *FoundScore;
    }

    return Score;
}
```

- <ins>Direction Policies:</ins> The `ApplyDirectionPoliciesToSelectedMovementChain()` function is the final step. It takes the chosen movement chain and applies a specific direction policy to each of its movement abilities. For example, a policy might dictate that the AI should always dash away from the player's last direction, making the AI's movements more unpredictable and effective.
```c++
bool UBDS_GetBestMovementChain::ApplyDirectionPoliciesToSelectedMovementChain(UMovementChainAsset* SelectedMovementChainAsset)
{
    if (!SelectedMovementChainAsset)
    {
        UE_LOG(LogTemp, Warning, TEXT("SelectedMovementChainAsset is null in: %s"), *GetName());
        return false;
    }
    bool bChanged = false;

    FGameplayTag HeroLastDirectionGameplayTag = HeroMovementListenerComp->GetHeroLastMovementDirectionTagByLastInput();

    for (FMovementAbilityData& MovementAbilityInChain : SelectedMovementChainAsset->MovementChain)
    {
        if (!MovementAbilityInChain.DirectionPolicyTag.IsValid())
        {
            continue;
        }

        if (MovementAbilityInChain.DirectionPolicyTag == GAS_Tags::TAG_AI_Direction_Policy_PlayerLastDirection)
        {
            if (HeroLastDirectionGameplayTag.IsValid())
            {
                MovementAbilityInChain.ResolvedDirectionTag = HeroLastDirectionGameplayTag;
                bChanged = true;
            }
        }
        else if (MovementAbilityInChain.DirectionPolicyTag == GAS_Tags::TAG_AI_Direction_Policy_Random)
        {
            MovementAbilityInChain.ResolvedDirectionTag = GetRandomDirectionTag();
            bChanged = true;
        }
    }

    return bChanged;
}
```

#### **2.2.3 Get Best Attack** 

The `UBDS_GetBestAttack` is a specialized service designed to select the `most optimal offensive ability` for the AI from a list of possibilities. It operates by dynamically scoring each potential attack based on several key factors, ensuring the AI's actions are tactical and well-timed.

- <ins>Dynamic Scoring & Decision Logic:</ins> This service works by assigning a score to each potential attack. The attack with the highest final score is chosen.
```c++
FAttackData UBDS_GetBestAttack::GetBestAttack()
{
    if (!IsValid(AttackAbilityAsset) || !EnemyASC)
    {
        UE_LOG(LogTemp, Warning, TEXT("AttackAbilityAsset or OwnerEnemyASC is null in: %s !"), *GetName());
        return FAttackData();
    }

    float BestScore = -FLT_MAX;
    FAttackData BestAttack;
    float BestAttackDistanceScore = 0.f;

    for (const FAttackData& Attack : AttackAbilityAsset->AttackAbilities)
    {
        if (!Attack.AbilityClass)
        {
            continue;
        }

        bool IsInCooldown = Attack.AbilityClass->GetDefaultObject<UGAS_GameplayAbilityBase>()->IsOnCooldown(EnemyASC);
        if (IsInCooldown)
        {
            continue;
        }

        float DistanceScore = CalculateAttackAbilityScoreBasedOnTargetDistance(Attack, EnemyController->GetTargetHeroDistance());

        float ComboScore = CalculateComboScore(Attack);

        float TotalScore = Attack.ScoreBias + DistanceScore + ComboScore;

        if (TotalScore > BestScore)
        {
            BestScore = TotalScore;
            BestAttack = Attack;
        }
    }

    /*
    if (GEngine && EnableSelectedDebug)
    {
        GEngine->AddOnScreenDebugMessage(9, 3.5f, FColor::Red,
            FString::Printf(TEXT(">> Selected Attack: %s | DistanceScore: %.1f"),
                *BestAttack.AbilityClass->GetName(), BestAttackDistanceScore));
    }
    */

    LastSelectedAttackAbilityData = BestAttack;
    return BestAttack;
}
```
 
 The score is calculated by combining multiple factors:

- Cooldown & Validity Check: The service first checks if an attack is on `cooldown`. If it is, the attack is immediately disregarded, ensuring the AI only considers abilities that are ready to use.

- Distance Scoring: The `CalculateAttackAbilityScoreBasedOnTargetDistance()` function evaluates the attack's effectiveness based on the distance to the target. It assigns a higher score to attacks that are a more appropriate range. For instance, a melee attack will receive a high score when the AI is close to the player, while a ranged attack will get a higher score when the player is farther away.
```c++
float UBDS_GetBestAttack::CalculateAttackAbilityScoreBasedOnTargetDistance(FAttackData AttackData, float DistanceToTarget)
{
    float AbilityMinRange = AttackData.AbilityClass->GetDefaultObject<UGAS_GameplayAbilityBase>()->MinRange;
    float AbilityMaxRange = AttackData.AbilityClass->GetDefaultObject<UGAS_GameplayAbilityBase>()->MaxRange;
    if (AbilityMaxRange <= 0.f)
    {
        return 0.0f;
    }

    // Ideal attack point: MaxRange
    float DistanceFromIdeal = FMath::Abs(DistanceToTarget - AbilityMaxRange);

    // Calculate the score inversely proportional to distance
    float Score = 1.f - (DistanceFromIdeal / AbilityMaxRange);

   // If the distance is BELOW the Minimum Range, apply extra penalty (optional)
    if (DistanceToTarget < AbilityMinRange)
    {
        Score = -100; // Disable if too close
    }

    return Score;
}
```

- Combo Scoring: The `CalculateComboScore()` function is a crucial part of the system that adds depth and fluidity to the AI's behavior. If the AI has just finished an attack that is part of a combo chain, this function gives a significant score bonus to the next logical attack in that sequence. This prevents the AI from choosing random abilities and allows it to perform coherent, multi-step attack patterns.
```c++
float UBDS_GetBestAttack::CalculateComboScore(FAttackData AttackData)
{
    // If there is no valid last attack or it wasn't part of a combo chain
    if (!LastSelectedAttackAbilityData.AbilityClass || !LastSelectedAttackAbilityData.bIsComboAttack)
    {
        return 0.0f;
    }

    // If the current candidate isn't a combo attack, skip
    if (!AttackData.bIsComboAttack)
    {
        return 0.0f;
    }

    // Expected combo index is always the next step after the last selected
    int32 ExpectedNextIndex = LastSelectedAttackAbilityData.ComboIndex + 1;

    // If this attack matches the expected combo step, give it a strong score
    if (AttackData.ComboIndex == ExpectedNextIndex)
    {
        return 100.0f;
    }

    return 0.0f;
}
```

The final score for each attack is a combination of these scores and a pre-defined ScoreBias. The service then selects the attack with the highest total score, ensuring a dynamic and intelligent choice every time.

#### **2.1.4 Get Best InComingAttack Reaction** 

Coming Attack Reaction Decision Service
The `UBDS_ComingAttackReactionBase` is the AI's specialized defensive decision-making service. It is designed to evaluate an incoming attack and select the `best possible defensive action`, such as a `parry`, `dodge`, or `taking damage`. This service is a core component of the AI's reactive behavior, ensuring it can respond intelligently to a player's attacks.

- Dynamic Scoring & Decision Logic: This service works by assigning a score to each potential defensive reaction. The reaction with the `highest final score is chosen`. The score is calculated by combining multiple factors:
```c++
float UBDS_ComingAttackReactionBase::CalculateComingAttackReactionScore(FComingAttackPayload ComingAttackPayload)
{
    float BehaviorScore = CalculateBehaviorStateScore();
    float TagScore = CalculateTagScore(ComingAttackPayload);

    float TotalScore = BehaviorScore + TagScore + ScoreBias;
    UE_LOG(LogTemp, Log, TEXT("[AI] Reaction %s → Score: %.2f"), *ComingAttackReactionName.ToString(), TotalScore);

    return TotalScore;
}
```

- Behavior State Score: `CalculateBehaviorStateScore()` adjusts the score based on the AI's current `BehaviorState` (e.g., `Aggressive`, `Defensive`). This allows you to create defensive reactions that are preferred for specific tactical situations.
```c++
float UBDS_ComingAttackReactionBase::CalculateBehaviorStateScore() const
{
	if (const float* Mod = BehaviorStateScoreModifiers.Find(BehaviorState))
	{
		return *Mod;
	}

	return 0.f;
}
```

- Tag Score: `CalculateTagScore()` gives a score bonus based on the tags associated with the incoming attack. For instance, an attack with a `HeavyAttack` tag might increase the score of a Parry reaction, while a `LightAttack` tag might increase the score of a `Dodge` reaction.
```c++
float UBDS_ComingAttackReactionBase::CalculateTagScore(const FComingAttackPayload ComingAttackPayload) const
{
	float Score = 0.f;

	for (const auto& Pair : TagScoreModifiers)
	{
		if (ComingAttackPayload.ComingAttackTags.HasTag(Pair.Key))
		{
			Score += Pair.Value;
		}
	}

	return Score;
}
```

- Chance Roll: The `PassesFinalChanceRoll()` function adds an element of unpredictability to the AI's decisions. For a Parry reaction, the chance to succeed is based on the AI's Posture value, while for a Dodge reaction, it is based on a pre-defined `BaseChance`. This adds depth to the system and prevents the AI from becoming too predictable.
```c++
bool UBDS_ComingAttackReactionBase::PassesFinalChanceRoll() const
{
    if (ReactionType == EComingAttackReaction::Dodge)
    {
        return PassesChanceRoll();
    }
    else if (ReactionType == EComingAttackReaction::Parry)
    {
        return PassesChanceRollBasedOnPosture();
    }
    // ReactionType == EComingAttackReaction::TakeDamage
    else 
    {
        return true;
    }
}

bool UBDS_ComingAttackReactionBase::PassesChanceRoll() const
{
    const float Roll = FMath::FRandRange(0.f, 1.f);  
    const bool bPassed = Roll <= BaseChance;

    UE_LOG(LogTemp, Log, TEXT("[AI] Static chance reaction %s: roll %.2f <= base chance %.2f → %s"),
        *ComingAttackReactionName.ToString(),
        Roll,
        BaseChance,
        bPassed ? TEXT("PASS") : TEXT("FAIL"));

    return bPassed;
}

bool UBDS_ComingAttackReactionBase::PassesChanceRollBasedOnPosture() const
{
    UAS_Base* BaseAttributes = const_cast<UAS_Base*>(EnemyASC->GetSet<UAS_Base>());
    if (!BaseAttributes)
    {
        return false;
    }

    const float PostureValue = BaseAttributes->GetPosture();  
    const float Roll = FMath::FRandRange(0.f, 100.f);

    const bool bPassed = Roll <= PostureValue;

    UE_LOG(LogTemp, Log, TEXT("[AI] Posture-based reaction %s: roll %.2f <= posture %.2f → %s"),
        *ComingAttackReactionName.ToString(),
        Roll,
        PostureValue,
        bPassed ? TEXT("PASS") : TEXT("FAIL"));

    return bPassed;
}
```

- Execution with Precision: The `IsEnable()` function is a critical check that ensures the AI has enough time to react. If the time to impact is less than the `MinimumTimeBeforeHitToReact`, the reaction is not considered, preventing the AI from attempting actions that it cannot complete in time.
```c++
bool UBDS_ComingAttackReactionBase::IsEnable(FComingAttackPayload ComingAttackPayload) const
{
    return PassesFinalChanceRoll() && ComingAttackPayload.ComingAttackHitTime > MinimumTimeBeforeHitToReact;
}
```

The final score for each reaction is a combination of these scores and a pre-defined `ScoreBias`. The service then selects the reaction with the highest total score, ensuring a dynamic and intelligent defensive choice every time.

#### **3.2 Scoring** 

Every decision within the system is driven by a dynamic scoring mechanism. Services evaluate all available options and select the one with the highest score, which is calculated based on multiple factors.

- Attack Selection Logic:
The UBDS_GetBestAttack service evaluates potential attacks by a combination of scores:

- Cooldown & Validity: It first checks if an ability is on cooldown. If it is, the ability is immediately disqualified from consideration.

- Distance Scoring: The CalculateAttackAbilityScoreBasedOnTargetDistance function uses a custom formula to favor attacks whose effective range (MaxRange) closely matches the current distance to the player.

- Combo Scoring: This is a crucial part of creating fluid AI attacks. The CalculateComboScore function assigns a very high score (100.0f) to an attack if it is the next step in a combo chain, encouraging the AI to complete its attack sequences.

- Score Bias: A designer-adjustable value that allows manual prioritization of certain attacks.


![image](https://github.com/user-attachments/assets/ee2008a4-0939-410b-a892-146fd0b9d1c9)

 - Distance to Target: A higher score is given if the attack's effective range matches the current distance to the player.

 - Cooldown Status: Abilities on cooldown are given a score of negative infinity to ensure they are never selected.

 - Score Bias: A designer-adjustable value to manually prioritize certain attacks over others.

 - GetBestMovementChain(...): Picks the optimal movement chain to accompany the selected attack. This score is influenced by:

![Ekran görüntüsü 2025-04-08 203016](https://github.com/user-attachments/assets/0d3c1f2a-014f-401c-aa8c-e54f421c2181)

![Ekran görüntüsü 2025-04-08 203150](https://github.com/user-attachments/assets/86c9b586-279b-4f5f-970e-aac7f4be6cf1)

 - Distance: The score is boosted if the movement chain will put the AI in an ideal range for its next attack.

 - Target Movement: The AI's movement score changes based on whether the player is moving, allowing the AI to react intelligently by either closing the gap or creating distance.

- Direction Policies:
This system adds another layer of complexity by allowing the AI to dynamically choose its movement direction. The movement chain's direction (e.g., move left, move right, roll forward) is determined at runtime based on defined policies. This includes reacting to the player's last movement direction or randomizing its own movement for unpredictable behavior.

## **3. The Execution Layer** 
The most robust tactical decision is worthless if the system cannot translate it into precise, reliable physical action. This layer is responsible for taking the winning `Movement Chain` or `Attack Ability` selected by the `Behavior Decision Component` and executing it flawlessly in the game world, leveraging the power and reliability of the` Gameplay Ability System (GAS`).

#### **3.1 Movement System** 
The `UAC_EnemyMovementManager` component acts as the dedicated execution orchestrator for all AI movement. Its singular purpose is to take the high-level, scored movement decision from the `BehaviorDecisionComponent` and translate it into a sequence of low-level, executable GAS Abilities. This clean separation of concerns ensures that the movement logic remains tightly coupled with the core Ability System, guaranteeing reliability, replication, and precise control over movement effects.

Movement Chain Philosophy
The core of the system is the `Movement Chain`. Unlike traditional approaches that execute one movement action at a time, this system uses the `FMovementChainTracker` struct to manage an ordered sequence of `FMovementAbilityData` entries. This allows the AI to perform complex, multi-step maneuvers (like a dash followed by a `strafe`, followed by a `final jump`) as a single, atomic tactical action, rather than relying on multiple state transitions.

Key Execution Logic
- StartMovementChain: This is the primary entry point, called by the `Movement State` after the `Behavior Decision Component` has selected the best chain based on the chosen attack.

It first queries the BehaviorDecisionComp to receive the final, scored array of `FMovementAbilityData`.

It initializes the MovementChainTracker with this array and calls `TryExecuteNextMovementAbilityInChain()`.

- TryExecuteNextMovementAbilityInChain: This function iterates through the active chain. If the chain is finished, it calls StopMovementAbilities() and broadcasts the OnMovementChainEnded delegate, signaling the Movement State that the tactical goal is complete and a new decision is required.

If a movement ability exists in the chain, it calls `TryActivateMovementAbilityWithEventData()`.

- GAS Activation and Event Data: The system leverages `FGameplayEventData` to pass contextual information (like the direction tag resolved by the Decision Service) directly to the `GAS ability`. This ensures the ability executes exactly as intended by the decision-making layer.

Upon successful activation, it dynamically binds `OnMovementAbilityEnded()` to the ability's delegate, establishing the crucial event-driven link required for sequence execution.

- Event-Driven Advancement: The `OnMovementAbilityEnded()` function is the heart of the sequence control. When a movement ability completes (or is canceled):

If the ability was canceled, the entire chain is terminated immediately via `StopMovementAbilities()`, prioritizing immediate responsiveness to interruption.

If the ability completed successfully, the tracker advances its index (MovementChainTracker.Advance()) and calls `TryExecuteNextMovementAbilityInChain()` to initiate the next ability in the sequence.

- Cancellation and Cleanup: The `StopMovementAbilities()` and `CancelMovementAbilities()` functions provide robust control. `CancelMovementAbilities()` specifically targets abilities with the `TAG_Gameplay_Ability_Movement` tag, ensuring that only movement abilities are interrupted without affecting potential concurrent abilities (e.g., Passive Effects). This ensures reliable and instantaneous stop commands when a transition to the Attack State or InComingAttack State is necessary.

### **3.2 Movement Abilities** 
The `Movement Chain` philosophy is only made possible by the robust, centralized logic contained within the `UGA_EnemyMovementBase` class. This class is the foundational `GAS Ability` from which all specific AI movements (chasing, strafing, etc.) inherit. It defines the core execution lifecycle and the necessary integration points with the AI Controller and Navigation System.

#### **3.2.1 Movement Base** 
This class functions as the contract for all executable movement, ensuring that every movement ability is inherently synchronized, cancellable, and integrates seamlessly with the movement chain logic managed by the `UAC_EnemyMovementManager`.

Core Execution and Integration
The `UGA_EnemyMovementBase` is an `InstancedPerActor` ability tagged with `TAG_Gameplay_Ability_Movement`, guaranteeing strict control over its execution.

- Ability Activation: Upon ActivateAbility, the ability first validates its core dependencies (`EnemyCharacter`, `EnemyController`, `EnemyMovementComp`). Crucially, it sets the enemy's `MaxWalkSpeed` using the configurable `MovementSpeed` property. This allows designers to assign distinct, custom movement speeds to specific abilities (e.g., a "fast dash" ability vs. a "slow strafe" ability).

- Move Request: The primary execution logic resides in `RequestMoveToLocation` and `RequestMoveToTarget`. These functions are responsible for packaging the movement goal—either a static world location or the player actor—into an `FAIMoveRequest`.

The movement request uses configurable parameters like `AcceptanceRadius` for pathfinding and sets `SetAllowPartialPath(true)` to maximize the AI's adaptability in dynamic environments.

Path failure is handled immediately; if the `MoveTo` request fails, the ability is ended with cancellation, preventing the chain from stalling.

- Event-Driven Completion: Unlike tick-based movement, the ability relies entirely on the `OnRequestFinished` delegate of the `PathFollowingComponent` to know when the move is complete.

The OnMoveCompleted callback processes the result. If the move was successful, it calls EndAbility(..., bWasCancelled=false), signaling a clean transition to the next ability in the chain. If the move failed or was aborted, it calls EndAbility(..., bWasCancelled=true), triggering an immediate termination of the entire movement chain.

- Robust Cleanup and Safety
The EndAbility function is critical for maintaining a clean execution context. Its robust cleanup ensures the system remains bug-free and responsive:

It stops all current movement on the EnemyController.

It immediately removes the binding from the PathFollowingComponent->OnRequestFinished delegate. This prevents the ability from firing the end event (or causing crashes) long after it has concluded, eliminating the risk of zombie delegates.

It clears any active MovementTimerHandle instances, guaranteeing that all time-based movement logic ceases immediately.

This meticulous cleanup ensures that the `UGA_EnemyMovementBase` is a reliable, atomic unit of action, ready to be instantiated and reused for any movement chain.

#### **3.2.2 Chase Target** 
The `UGA_EnemyChaseTarget` ability is the concrete implementation of the base movement system focused on immediate and persistent target acquisition. Its primary function is to direct the AI to its designated target actor (typically the player) and maintain pursuit for a tactically defined duration.

- Execution and Duration Control
This ability is activated exclusively via a GameplayEvent (TAG_AI_AbilityTriggerEvent_Movement_ChaseTarget), ensuring it is triggered precisely by the Movement Manager as part of a chain sequence.

- Target Acquisition: Upon activation, it inherits the `RequestMoveToTarget(AActor* TargetActor)` function from the Movement Base to initiate navigation towards the player. This is a continuous pursuit, where the AI Controller constantly updates the target location.

- Timed Execution: This ability introduces a crucial element of time-based control. It utilizes the `TriggerEventData->EventMagnitude` to set a `Movement Timer`.

If the EventMagnitude is greater than zero, the ability sets a timer that executes the `OnChaseTimeEnd()` callback upon expiration.

- OnChaseTimeEnd() then cleanly ends the ability, guaranteeing that the AI does not chase indefinitely but rather for the exact duration specified by the Movement Chain data asset.

This time-based self-termination mechanism ensures that the Chase Target ability remains a highly modular and predictable component, allowing designers to precisely control the duration of pursuit within any complex movement sequence.

#### **3.2.3 Strafing** 
Bu, AI'ınızın ne kadar sofistike olduğunu gösteren kritik bir yetenek. Strafing yeteneği sadece hareket etmiyor, aynı zamanda Çevresel Sorgulama Sistemini (EQS) kullanarak bilinçli bir karar veriyor ve Behavior Decision katmanından gelen kararları uyguluyor.

Önceki bölümlerin teknik ve akıcı üslubuna uygun olarak, Strafing Base yeteneği dokümantasyonunu hazırlıyorum:

3.2.2 Strafing Base
The UGA_EnemyStrafingBase is a highly specialized movement ability that demonstrates the system's deep integration with environmental awareness and the Environmental Query System (EQS). Unlike simple pursuit, this ability's purpose is tactical repositioning and evasion, requiring the AI to find the optimal lateral position relative to the target.

EQS Integration and Directional Control
This ability is a prime example of the execution layer leveraging contextual data provided by the decision layer.

Directional Input: Upon activation, the ability extracts the intended strafe direction (e.g., TAG_AI_Direction_Resolved_Left or TAG_AI_Direction_Resolved_Right) from the TriggerEventData->InstigatorTags. This tag is the definitive policy decision made by the Get Best Movement service.

EQS Query Execution: The StartEQSForStrafingLocation() function translates the received direction tag into a standardized float value via ConvertStrafeDirectionTagToFloat(). This value is then passed as the StrafeDirectionParam to the designated EQSQueryTemplate. This architecture allows a single EQS template to be dynamically customized to search for a safe position specifically to the left or right of the AI.

Movement Translation: The OnStrafingLocationQueryFinished() callback processes the EQS result. It retrieves the BestLocation and uses the base class's RequestMoveToLocation() function to initiate movement toward the optimal, safely queried position.

Time-Based Termination
Similar to the Chase Target ability, the UGA_EnemyStrafingBase is duration-controlled. The TriggerEventData->EventMagnitude sets a timer that executes OnStrafingTimeEnd(). This mechanism ensures the strafing action does not continue indefinitely, but ceases after the tactically defined period, allowing the Movement Manager to advance to the next step in the chain or request a new decision. The ability is designed to end with a cancellation (bWasCancelled=true) upon timer expiration, signaling to the Movement Manager that the sequence should not be interrupted mid-move but concluded precisely at the end of the duration.

## **4. Contextual and External Systems** 

### **4.1 Crowd Manager** 



		  
