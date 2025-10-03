# Shadow Samurai Showcase

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
     - [The Behavior Decision and Scoring](#2-The-Behavior-Decision-and-Scoring)
          - [Behavior Decision](#21-Behavior-Decision)
          - [Services](#22-Services)
               - [Service Base](#221-Service-Base)
               - [Get Best Movement](#222-Get-Best-Movement)
               - [Get Best Attack Base](#223-Get-Best-Attack)
               - [Get Best IncomingAttack Reaction](#224-Get-Best-IncomingAttack-Reaction)
     - [The Execution Layer](#3-The-Execution-Layer)
          - [Movement System](#31-Movement-System)
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

The core API for interaction and management is defined by a set of `Key Virtual Functions` that align with the AI's execution cycle: 

- <ins>OnEnter():</ins> Executed immediately when the AI transitions into this state. This is the activation point where crucial initialization logic is handled, such as binding delegates or halting prior movement.

- <ins>OnTick(float DeltaTime):</ins> This function serves as the State's primary update loop, called every frame while the AI is in this state. It is utilized for continuous checks and updates, primarily monitoring distance, time-sensitive events, or evaluating exit conditions.

- <ins>OnExit():</ins> Called just before the AI leaves the state, this function is solely responsible for essential cleanup logic, ensuring that anything initiated in `OnEnter()` or during execution is safely terminated, like unbinding delegates or resetting temporary variables.

- <ins>EnterCondition() and ExitCondition():</ins> These virtual functions provide an additional, powerful layer of `self-governance`. They allow the state itself to dynamically check if the tactical conditions are currently right for it to be safely entered or exited, providing a crucial safety net for complex state transitions.

In essence, UStateBase is the contract for behavior, defining the rigorous API that the StateManager uses to interact with and manage all the different behavioral implementations.

#### **1.2.2 Movement State** 
The `UMovementState` is one of the most dynamic states within the AI system. Its primary purpose is to manage the AI's movement, ensuring it gets into the `optimal position to execute a pre-selected attack`. It is highly reactive and continuously evaluates the tactical situation to find the most suitable movement chain.

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

- <ins>Movement Delegation:</ins> The `StartMovementChain()` function is responsible for initiating the movement sequence and dynamically `binding a completion callback`. It calls the [Movement Manager](#31-Movement-System) to start the chain and then binds `OnMovementChainEnded` to the manager's delegate. This dynamic binding ensures the `UMovementState` is notified the moment the AI reaches its target or the movement fails.
```c++
void UMovementState::StartMovementChain(TSubclassOf<class UGAS_GameplayAbilityBase> SelectedAttackAbilityClass)
{
	MovementManagerComponent->StartMovementChain(SelectedAttackAbilityClass);

	if (!MovementManagerComponent->OnMovementChainEnded.IsAlreadyBound(this, &UMovementState::OnMovementChainEnded)) 
	{
		MovementManagerComponent->OnMovementChainEnded.AddDynamic(this, &UMovementState::OnMovementChainEnded);
	}
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
The `UAttackStateBase` manages the AI's offensive operations. Once the AI has successfully positioned itself within a suitable range (transitioning from the Movement State), this state takes over to execute a `pre-selected attack ability`. This state meticulously handles the entire lifecycle of the attack, from activation and execution to completion and cleanup, ensuring a fluid and responsive combat experience.

- <ins>OnEnter:</ins> Upon entering the `AttackState`, the AI first calls `StopMovementAbilities()` to halt any ongoing movement. It then immediately calls `SelectAndMakeAttack()`, which attempts to activate the corresponding ability from the `Gameplay Ability System (GAS)`.
```c++
void UAttackStateBase::OnEnter_Implementation()
{
	Super::OnEnter_Implementation(); 
	Enemy->GetEnemyMovementManagerComponent()->StopMovementAbilities();
	SelectAndMakeAttack();
}
```

- <ins>Attack Execution and Dynamic Binding:</ins> The `MakeAttack()` function is central to the state's lifecycle management and `execution`. It first calls `TryActivateAbilityByClassAndReturnInstance` to `initiate the ability`. If the activation is successful, the state establishes a `dynamic binding` by adding the `OnAttackAbilityEnded()` callback to the activated ability's `OnGameplayAbilityEndedWithDataBP` delegate. This ensures the `AttackState` is notified and can proceed with its exit logic the precise moment the ability concludes.
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
```
```c++
void UAttackStateBase::OnAttackAbilityEnded(const FAbilityEndedDataBP& DodgeAbilityEndedData)
{
	ExitRequest("OnAttackAbilityEnded");
}
```

- <ins>Cleanup on Exit:</ins> The `OnExit()` function is crucial for preventing `memory leaks` and ensuring component `integrity`. It checks the `LastUsedAttack` reference and confirms whether the `OnGameplayAbilityEndedWithDataBP` delegate is still dynamically bound. If it is, the delegate is `explicitly removed` using `RemoveDynamic()`. Finally, it clears the `LastUsedAttack` reference, ensuring the state is `fully reset` and ready for its next use without leaving `dangling pointers` or active bindings.
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
The `UInComingAttackState` is the AI's `reactive, defensive state`. It's triggered by the `Intend Handler` when the AI detects a significant incoming attack and must make a split-second decision on how to react.

- <ins>OnEnter: Decision and Action Delegation:</ins> The state begins by calling `SelectAndMakeInComingAttackReaction()` immediately upon entry. This function delegates the core decision-making to the `Behavior Decision Component`. Based on the highest-scoring defensive reaction (e.g., parry, take damage), it then directs the flow to the appropriate execution function (`MakeTakeDamage` or `MakeParryAbility`).
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
```c++
void UInComingAttackState::OnParryAbilityEnded(const FAbilityEndedDataBP& DodgeAbilityEndedData)
{
	if (EnemyASC->HasMatchingGameplayTag(GAS_Tags::TAG_Gameplay_State_InCombat_ParryKnockback))
	{
		UE_LOG(LogTemp, Warning, TEXT("State Manager: OnParryAbilityEnded with knocback, now we listen knocback removed for exit"));
		EnemyTagDelegatesComp->RegisterDelegateForTag(GAS_Tags::TAG_Gameplay_State_InCombat_ParryKnockback, EListenMode::OnRemoved).BindDynamic(this, &UInComingAttackState::OnParryKnocbackTagRemoved);
	}
	else
	{
		ExitRequest("Parry Ability Ended without knocback.");
	}
}
```

- <ins>Robust Cleanup:</ins> The `OnExit()` function is critical for maintaining system `stability` and preventing the memory issues associated with dynamic binding. It performs explicit cleanup by checking all potential `active bindings` to the `Gameplay Ability System (GAS)` delegates `(OnGameplayAbilityEndedWithDataBP)` and immediately unbinding them using `RemoveDynamic()`. This robust process ensures the state is fully reset and ready for its next use `without leaving dangling delegates or active references` that could lead to unintended behavior or crashes.
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
The `UAC_IntendHandlerBase` component serves as the AI's `perceptual layer`, acting as a crucial bridge between environmental stimuli and the AI's core decision making process. Its primary purpose is to identify critical moments such as target detection, vulnerability, or incoming attacks and to notify the [State Manager](#11-State-Manager) to initiate a new behavioral sequence.

The Intend Handler achieves this by subscribing to various delegates and events, rather than constantly polling for changes. This `event-driven approach` ensures the system is `highly performant and instantly responsive` to dynamic combat situations.

Core Event Triggers:

- <ins>Target Detection:</ins> The component continuously scans the environment for valid targets. When a new target is confirmed, it alerts the `State Manager` to initiate the appropriate combat sequence, typically moving the AI out of an Idle or Patrol state.
```c++
void UAC_IntendHandlerBase::OnTargetDetected(AActor* DetectedTarget)
{
	OwnerStateManager->OnTargetDetected();
}
```

- <ins>Vulnerable State:</ins> When the enemy's `Vulnerable` tag is added (e.g., after a parry or stagger), the `OnVulnerableTagAdded` delegate fires, immediately pushing the AI into a Vulnerable state.
```c++
void UAC_IntendHandlerBase::OnVulnerableTagAdded(const UAbilitySystemComponent* AbilitySystemComponent, const FGameplayTag& Tag)
{
	OwnerStateManager->RequestStateTreeEnter(GAS_Tags::TAG_AI_State_Vulnerable);
}
```

Proactive Defense: Incoming Attack Reaction Logic

This system serves as the AI's proactive defensive layer, managing precise reactions to detected attacks. It is a critical example of the tight integration between the `Intend Handler`, `Behavior Decision Component`, and `State Manager`. The entire flow ensures that when the AI detects an incoming attack, it calculates the optimal defensive reaction and executes it with frame precision.

- <ins>Detection and Payload Creation:</ins> When the player initiates a melee attack, the `OnTargetAbilityActivated` function processes the event. This function's responsibility is to gather all necessary attack data including static and dynamic `Gameplay Tags` and the crucial ComingAttackHitTime and package them into a structured `FComingAttackPayload`. This payload contains all the intelligence required for the AI to make a defensive choice.
```c++
void UAC_IntendHandlerBase::OnTargetAbilityActivated(UGameplayAbility* Ability)
{
	if (!Ability)
	{
		return;
	}

	UGA_MeleeAttackBase* MeleeAttackAbility = Cast<UGA_MeleeAttackBase>(Ability);
	if (!MeleeAttackAbility)
	{
		return;
	}

	FGameplayTagContainer CombinedTags;

	// Add static tags
	CombinedTags.AppendTags(Ability->GetAssetTags());

	// Add dynamic tags from current spec
	if (const FGameplayAbilitySpec* Spec = Ability->GetCurrentAbilitySpec())
	{
		CombinedTags.AppendTags(Spec->DynamicAbilityTags);
	}

	float AttackTime = GetAttackNotifyTriggerTime(MeleeAttackAbility, CombinedTags);
	FComingAttackPayload Payload(MeleeAttackAbility, AttackTime, CombinedTags);
	SendEventToDefense(Payload);
}
```

- <ins>Decision and Precision Execution:</ins> The `SendEventToDefense` function orchestrates both the final decision and the execution timing. It delegates the choice of the optimal reaction to the `Behavior Decision Component (GetBestComingAttackReaction)`. If the chosen reaction requires specific timing (e.g., parrying a few frames before the hit), the logic schedules the reaction using a `timer (SetTimer)`. This scheduling ensures the AI performs the action at the `precise moment` for maximum effectiveness, guaranteeing responsiveness that is not tied to the tick rate.
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
The shift from the `Control Flow (I)` (which determines which high-level state the AI is currently in) to this `Behavior Decision (II)` section is the transition to tactical execution. Here, the AI answers the question: `"Given my current state and combat context, what specific, optimal action should I take right now?"`

This system is built around the central UAC_BehaviorDecision component, which acts as an `orchestrator and data provider`. Its primary role is not to invent actions, but to manage and query a set of specialized Behavior `Decision Services (BDS)`. These services contain the true "intelligence," using a dynamic `scoring system` to weigh every available action be it an `Attack`, a `Movement Chain`, or a `Defensive Reaction` and determine the single best course of action data to provide back to the Control Flow.

### **2.1 Behavior Decision** 
The `UAC_BehaviorDecision` component is the AI's `tactical action layer`, specifically responsible for selecting the optimal action data based on dynamic scoring. This component acts as the `data source` for the AI's overall control logic, providing the necessary tactical actions and data required by the main AI system to proceed with execution, regardless of the current high-level state.

- <ins>Service Initialization:</ins> The core orchestration happens in `InitalizeServiceses()`. This function dynamically creates and initializes the three main decision-making services, injecting all the required context data `(Owner, Target, ASC, Behavior State, and relevant Data Assets)` via the `FBehaviorServiceInitParams` structure.
```c++
void UAC_BehaviorDecision::InitalizeServiceses()
{
    for (UBDS_ComingAttackReactionBase* ReactionInstance : ComingAttackReactionAsset->ComingAttackReactions)
    {
        if (ReactionInstance) 
        {
            FBehaviorServiceInitParams ComingAttackReactionServiceInitData = FBehaviorServiceInitParams(
                ComingAttackReactionAsset, OwnerEnemyBase, OwnerController, OwnerEnemyASC, HeroBase, HeroMovementListenerComp, BehaviorState);
            ReactionInstance->Initialize(ComingAttackReactionServiceInitData);
        }
    }

    GetBestAttackService = NewObject<UBDS_GetBestAttack>(GetOwner());
    FBehaviorServiceInitParams GetBestAttackServiceInitData = FBehaviorServiceInitParams
    (AttackAbilityAsset, OwnerEnemyBase, OwnerController, OwnerEnemyASC, HeroBase, HeroMovementListenerComp, BehaviorState);
    GetBestAttackService->Initialize(GetBestAttackServiceInitData);

    GetBestMovementChainService = NewObject<UBDS_GetBestMovementChain>(GetOwner());
    FBehaviorServiceInitParams GetBestMovementChainServiceInitData = FBehaviorServiceInitParams(
        AttackAbilityMovementChainMapAsset, OwnerEnemyBase, OwnerController, OwnerEnemyASC, HeroBase, HeroMovementListenerComp, BehaviorState);
    GetBestMovementChainService->Initialize(GetBestMovementChainServiceInitData);
}
```

Behavior Service APIs:
The `UAC_BehaviorDecision` exposes the following three public, Blueprint-callable API functions. These functions serve as the bridge, delegating the complex scoring logic entirely to the managed services and simply returning the optimal data structure.

- <ins>Attack Selection:</ins> The `GetBestAttack()` function calls the dedicated attack service `(UBDS_GetBestAttack)` to execute the scoring logic across all available attacks and returns the most suitable attack ability data `(FAttackData)`.
```c++
FAttackData UAC_BehaviorDecision::GetBestAttack()
{
    if (!GetBestAttackService) 
    {
        UE_LOG(LogTemp, Warning, TEXT("GetBestAttackService is null in: %s"), *GetName());
        return FAttackData();
    }

    FAttackData BestAttack = GetBestAttackService->GetBestAttack();
    LastSelectedAttackAbilityData = BestAttack;
    return BestAttack;
}
```

- <ins>Movement Chain Selection:</ins> The `GetBestMovementChain()` function takes the selected attack ability as input and finds the corresponding, most effective movement chain (e.g., Dash in, Retreat, Strafe) that places the AI in the optimal position for execution.
```c++
TArray<FMovementAbilityData> UAC_BehaviorDecision::GetBestMovementChain(TSubclassOf<UGAS_GameplayAbilityBase> SelectedAbilityClass)
{
    if (!GetBestMovementChainService)
    {
        UE_LOG(LogTemp, Warning, TEXT("GetBestMovementChainService is null in: %s"), *GetName());
        return TArray<FMovementAbilityData>();
    }

    UMovementChainAsset* BestMovementChainDataAsset = nullptr;

    if (IsValid(GetBestMovementChainService)) 
    {
        BestMovementChainDataAsset = GetBestMovementChainService->GetBestMovementChain(SelectedAbilityClass);
    }

    if (!BestMovementChainDataAsset) 
    {
        UE_LOG(LogTemp, Warning, TEXT("BestMovementChainDataAsset is null in: %s"), *GetName());
        return TArray<FMovementAbilityData>();
    }

    return BestMovementChainDataAsset->MovementChain;
}
```

- <ins>Incoming Attack Reaction:</ins> The `GetBestComingAttackReaction()` function is responsible for the AI's defensive decision-making. It iterates through all registered Reaction Services, checks if they are `enabled` in the current context, calculates a `score` for each, and selects the reaction with the highest score (e.g., Parry, Dodge).
```c++
UBDS_ComingAttackReactionBase* UAC_BehaviorDecision::GetBestComingAttackReaction(FComingAttackPayload ComingAttackPayload)
{
    UBDS_ComingAttackReactionBase* BestComingAttackInstance = nullptr;

    if (!IsValid(ComingAttackReactionAsset) || !ComingAttackPayload.ComingAttack)
    {
        return BestComingAttackInstance;
    }

    EComingAttackReaction BestReaction = EComingAttackReaction::TakeDamage;
    float BestScore = -FLT_MAX;

    for (UBDS_ComingAttackReactionBase* ReactionInstance : ComingAttackReactionAsset->ComingAttackReactions)
    {
        if (!ReactionInstance) 
        {
            UE_LOG(LogTemp, Warning, TEXT("ReactionInstance is null in: %s"), *GetName());
            continue;
        }

        if (!ReactionInstance->IsEnable(ComingAttackPayload))
        {
            continue;
        }

        float ComingAttackReactionScore = ReactionInstance->CalculateComingAttackReactionScore(ComingAttackPayload);
        UE_LOG(LogTemp, Log, TEXT("[AI] Reaction %s → Score: %.2f"), *ReactionInstance->ComingAttackReactionName.ToString(), ComingAttackReactionScore);

        if (ComingAttackReactionScore > BestScore)
        {
            BestScore = ComingAttackReactionScore;
            BestComingAttackInstance = ReactionInstance;
        }
    }

    if (BestComingAttackInstance)
    {
        BestComingAttackInstance->InitializeAfterSelection();
        LastSelectedComingAttackReaction = BestComingAttackInstance;
        UE_LOG(LogTemp, Log, TEXT("[AI] SelectedReaction %s"), *BestComingAttackInstance->ComingAttackReactionName.ToString());
    }

    return BestComingAttackInstance;
}
```

### **2.2 Services** 
The tactical complexity of the AI system requires a `highly modular and scalable approach` to data analysis and decision-making. This is achieved through a `Service-Oriented Architecture`, where specific, specialized services handle distinct decision types.

Crucially, these Services are the core tools utilized by the `UAC_BehaviorDecision` component. They encapsulate all the complex scoring, utility calculations, and contextual data analysis, allowing the UAC_BehaviorDecision to remain a clean, centralized manager that merely queries the Services for the best possible action data.

#### **2.2.1 Service Base** 
The `UBehaviorDecisionServiceBase` class is the foundation for the AI's tactical decision-making system. It is a foundational abstract class designed to be inherited by specific services that handle different types of decisions, such as selecting an attack or a movement chain.

The class utilizes an `FBehaviorServiceInitParams` struct to initialize itself. This struct provides all the necessary references (like the `Enemy`, `EnemyController`, and `Hero`) upon creation, ensuring that each service has access to the data it needs without having to manually find it.

#### **2.2.2 Get Best Movement** 
The `UBDS_GetBestMovementChain` is a specialized service responsible for determining the optimal movement sequence for the AI. It operates as part of the `Behavior Decision` component, evaluating various movement "chains" based on a `dynamic scoring system` to select the most advantageous one for the current combat situation, specifically after an attack ability has been selected.

Data-Driven Design: Movement Chain Assets

The entire movement decision process is driven by three key Data Assets 

- <ins>UMovementChainAsset:</ins> Defines a `single, specific sequence of movement abilities` (the "chain"). Contains all scoring modifiers and constraints (Min Range, Score Bias, etc.) needed by the service to evaluate its utility.
  
![Ekran görüntüsü 2025-04-08 203150](https://github.com/user-attachments/assets/86c9b586-279b-4f5f-970e-aac7f4be6cf1)
```c++
UCLASS(BlueprintType)
class UMovementChainAsset : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Name of this movement chain. Used for debugging or referencing in logic."))
    FName MovementChainName;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Sequence of movement abilities that make up this chain. Executed in order."))
    TArray<FMovementAbilityData> MovementChain;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Optional score modifiers based on current behavior state (e.g., aggressive, defensive)."))
    TMap<EBehaviorState, float> BehaviorStateModifiers;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Score curve based on distance to target. High values make this chain more likely when far/close depending on the curve."))
    UCurveFloat* DistanceScoreCurve = nullptr;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Score bonus applied if the target is currently moving."))
    float ScoreModifierWhenTargetIsMoving = 0.0f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Score bonus applied if the target is not moving)."))
    float ScoreModifierWhenTargetIsNotMoving = 0.0f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Minimum target distance required for this chain to be considered."))
    float MinRange;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, meta = (ToolTip = "Flat score bias added to this chain's total score. Useful to prioritize certain chains."))
    float ScoreBias = 0.f;
};
```

- <ins>FAttackAbilityMovementChains:</ins> Maps a single `Attack Ability Class` to an array of UMovementChainAsset instances. This defines the pool of possible movement chains for that specific attack.
  
![Ekran görüntüsü 2025-04-08 203016](https://github.com/user-attachments/assets/0d3c1f2a-014f-401c-aa8c-e54f421c2181)
```c++
USTRUCT(BlueprintType)
struct FAttackAbilityMovementChains
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSubclassOf<UGAS_GameplayAbilityBase> AttackAbilityClass;

    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TArray<UMovementChainAsset*> MovementChainAssets;
};
```

- <ins>UAttackAbilityMovementChainMapAsset:</ins> The primary container. Holds an array of `FAttackAbilityMovementChains` structs, allowing the AI to look up all relevant movement options for any given attack ability.
```c++
UCLASS(BlueprintType)
class UAttackAbilityMovementChainMapAsset : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TArray<FAttackAbilityMovementChains> ChainMappings;
};
```

Dynamic Scoring & Decision Logic:

- <ins>GetBestMovementChain:</ins> The `GetBestMovementChain()` function orchestrates the decision process. It first retrieves the list of valid movement chains associated with the `SelectedAbilityClass`. It then iterates through each potential chain, applies dedicated scoring functions to calculate its utility, and selects the chain with the highest total score.
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

        if (TotalScore > BestScore)
        {
            BestScore = TotalScore;
            BestMovementChainDistanceScore = DistanceScore;
            BestMovementChainTargetMovementScore = TargetMovementScore;
            BestMovementChainDataAsset = MovementChainAsset;
        }
    }

    ApplyDirectionPoliciesToSelectedMovementChain(BestMovementChainDataAsset);
    return BestMovementChainDataAsset;
}
```

Scoring Components:

- <ins>Distance Scoring:</ins> The `CalculateMovementChainScoreBasedOnTargetDistance()` function evaluates how a movement chain's distance to the target affects its score. The logic applies bonuses or penalties based on the target's recent movement (if the target is moving or standing still) and applies a `hard penalty` (-100.0f) if the AI is already closer than the chain's `MinRange`, effectively eliminating that chain from selection.
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

- <ins>Behavior State Modifiers:</ins> The `CalculateMovementChainScoreBasedOnBehaviorState()` function incorporates the AI's current `BehaviorState` (e.g., `Aggressive`, `Defensive`) into the score by referencing the `BehaviorStateModifiers` map defined in the `UMovementChainAsset`. This allows designers to prioritize certain chains based on the AI's top-level tactical role.
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

Direction Policies:

- <ins>Direction Policies:</ins> The `ApplyDirectionPoliciesToSelectedMovementChain()` function is the final, critical step, where abstract policies are converted into concrete movement directions. It iterates through every movement ability in the selected chain and uses the logic inherited from the `UBehaviorDecisionServiceBase` to resolve the `DirectionPolicyTag` (e.g., `TAG_AI_Direction_Policy_Random` or `TAG_AI_Direction_Policy_PlayerLastDirection`) into a final `ResolvedDirectionTag`.
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
The `UBDS_GetBestAttack` is a specialized service designed to select the `most optimal offensive ability` for the AI from a list of possibilities. It operates by dynamically scoring each potential attack based on several key tactical factors, ensuring the AI's actions are precise and well-timed.

Data-Driven Design: Attack Data Assets

The attack selection process is driven by the following Data Asset structures, which allow designers to configure attack properties and scoring biases via the editor:

![image](https://github.com/user-attachments/assets/ee2008a4-0939-410b-a892-146fd0b9d1c9)

- <ins>FAttackData:</ins> Defines the parameters for a single attack ability, including its `AbilityClass`, `ScoreBias`, and `ComboIndex` information.
```c++
USTRUCT(BlueprintType)
struct FAttackData
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Ability class that defines the actual gameplay logic and range values"))
    TSubclassOf<class UGAS_GameplayAbilityBase> AbilityClass;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Optional score modifiers per behavior state"))
    TMap<EBehaviorState, float> BehaviorStateScoreModifiers;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Whether this attack is part of a combo chain"))
    bool bIsComboAttack;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Combo index used for ordering within a combo chain "), meta = (EditCondition = "bIsComboAttack"))
    int32 ComboIndex = 0;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, meta = (ToolTip = "Base score bias applied to AI decision-making"))
    float ScoreBias = 0.f;
};
```

- <ins>UAttackAbilityAsset:</ins> The primary container. Holds an array of all possible `FAttackData` structures the AI can choose from.
```c++
UCLASS(BlueprintType)
class UAttackAbilityAsset : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TArray<FAttackData> AttackAbilities;
};
```

Dynamic Scoring & Decision Logic

- <ins>GetBestAttack:</ins> The `GetBestAttack()` function orchestrates the attack selection process. It iterates through all available attacks, performs a `Cooldown Check`, calculates `Distance` and `Combo` scores, and selects the one with the highest total utility.
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

    LastSelectedAttackAbilityData = BestAttack;
    return BestAttack;
}
```
 
Scoring Components:

- <ins>Distance Scoring:</ins> The `CalculateAttackAbilityScoreBasedOnTargetDistance()` function calculates a score based on how close the target is to the attack's `ideal range` (AbilityMaxRange). The closer the target is to the ideal range, the higher the score. A vital safety check is performed: if the distance is `below the attack's` MinRange, a hard penalty (-100) is applied, preventing the AI from selecting attacks for which it is too close.
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

- <ins>Combo Scoring:</ins> The `CalculateComboScore()` function is essential for creating fluid, sequential attack patterns. It checks if the previously selected attack (`LastSelectedAttackAbilityData`) was part of a combo. If so, it gives a `significant score bonus` (`+100.0f`) to the current candidate if it represents the `next logical` step (`ComboIndex + 1`) in that sequence. This strongly biases the AI towards completing combos once initiated.
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

#### **2.2.4 Get Best InComingAttack Reaction** 
The `UBDS_ComingAttackReactionBase` is the AI's specialized defensive decision-making service. It is a `foundational abstract class` designed to evaluate an incoming attack and select the `best possible defensive action` (e.g., `Parry`, `Dodge`, or `Take Damage`). This service is a core component of the AI's reactive behavior, ensuring it can respond intelligently and unpredictably to a player's attacks

Data-Driven Design: UBDS_ComingAttackReactionBase 

Unlike other services, this class is both the Base Service and the Data Container. Specific defensive reactions (Parry, Dodge) are instantiated as children of this class and grouped inside the UComingAttackReactionAsset.

- <ins>UBDS_ComingAttackReactionBase:</ins> Defines scoring functions, chance roll logic, and required timing parameters (e.g., `MinimumTimeBeforeHitToReact`).
```c++
UCLASS(Blueprintable, DefaultToInstanced, EditInLineNew)
class GAS_TEMPLATESP_API UBDS_ComingAttackReactionBase : public UBehaviorDecisionServiceBase
```

- <ins>UComingAttackReactionAsset:</ins> The primary container. Holds an array of `instantiated` `UBDS_ComingAttackReactionBase` children (the specific defensive actions) that the AI will evaluate.
```c++
UCLASS(BlueprintType)
class UComingAttackReactionAsset : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TArray<TObjectPtr<UBDS_ComingAttackReactionBase>> ComingAttackReactions;
};
```

Dynamic Scoring Logic:

Individual reaction services (derived from this base class) are scored based on the incoming attack's context (`FComingAttackPayload`). The `CalculateComingAttackReactionScore()` function sums the relevant weighted scores, with the final selection being the reaction that yields the highest total score
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

Scoring Components:

- <ins>Behavior State Score:</ins> The `CalculateBehaviorStateScore()` function integrates the AI's current `BehaviorState` into the decision. By utilizing `BehaviorStateScoreModifiers`, designers can prioritize specific defensive reactions (e.g., favoring `Parry` when Aggressive) based on the AI's current tactical role
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

- <ins>Tag Score:</ins> The `CalculateTagScore()` function applies a score bonus based on the `GameplayTags` associated with the incoming attack. This ensures the AI chooses the appropriate defense for the type of threat (e.g., scoring `Parry` higher if the incoming attack is tagged `HeavyAttack`).
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

Execution Precision and Unpredictability:

Before the winning reaction is executed, two critical checks are performed: an `Enable Check` based on timing, and a `Chance Roll` to introduce realistic unpredictability.

- <ins>Enable Check:</ins> The `IsEnable()` function is a critical check that prevents the AI from attempting actions that it cannot complete in time. The reaction is only considered if the time to impact (`ComingAttackHitTime`) is greater than the `MinimumTimeBeforeHitToReact` required by that defensive action, AND it passes the chance roll.
```c++
bool UBDS_ComingAttackReactionBase::IsEnable(FComingAttackPayload ComingAttackPayload) const
{
    return PassesFinalChanceRoll() && ComingAttackPayload.ComingAttackHitTime > MinimumTimeBeforeHitToReact;
}
```

- <ins>Chance Roll:</ins> The `PassesFinalChanceRoll()` function adds an element of unpredictability to the AI's decisions. For a Parry reaction, the chance to succeed is based on the AI's Posture value, while for a Dodge reaction, it is based on a pre-defined `BaseChance`. This adds depth to the system and prevents the AI from becoming too predictable.
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
```
```c++
bool UBDS_ComingAttackReactionBase::PassesChanceRoll() const
{
    const float Roll = FMath::FRandRange(0.f, 1.f);  
    const bool bPassed = Roll <= BaseChance;

    return bPassed;
}
```
```c++
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

    return bPassed;
}
```

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



		  
