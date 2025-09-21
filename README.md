


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

Gameplay video: [https://www.youtube.com/watch?v=_B-iSA1eJA4&ab_channel=%C5%9Eamil%C3%96zel](https://www.youtube.com/watch?v=rrsV4eMGDK8)


## Contents
- [Gameplay Systems](#Gameplay-Systems)
     - [Target Lock System](#1-Target-Lock-System)
- [Combat Abilities](#Combat-Abilities)
     - [Combo Abilities](#1-Combo-Abilities) 
     - [Parry](#2-Parry)
     - [Shadow Attack](#3-Shadow-Attack)
     - [Shadow Finisher](#3-Shadow-Finisher)
- [AI](#AI)
     - [State Manager](#1-State-Manager)
          - [Movement State](#1.1-Movement-State)
          - [Attack State](#1.2-Attack-State)
          - [InComingAttack State](#1.3-InComingAttack-State)
     - [Intend Handler](#2-Intend-Handler)
     - [Behavior Decision](#3-Behavior-Decision)
          - [Services](#Services)
          - [Scoring](#Scoring)
    
     


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

Developing a robust and intelligent AI for a fast-paced combat system was one of the most challenging aspects of this project. My journey began with traditional Behavior Trees, then moved to State Trees, and eventually a hybrid approach. However, none of these off-the-shelf solutions provided the granular control, complex data flow management, and sophisticated debugging capabilities required for the kind of dynamic AI I envisioned.

Ultimately, I decided to build a custom, data-driven state machine to achieve 100% control over the AI's behavior. This system allows for precise management of complex states and transitions, ensuring the AI can make intelligent, context-aware decisions in combat, leading to a more challenging and engaging gameplay experience.


### **1 State Manager** 

 he UAC_StateManager is a core component that defines and governs the distinct behavioral states of an enemy character (e.g., Idle, Patrol, Combat). This component acts as the central brain of the AI, handling state transitions and managing the flow of behavior.

Unlike traditional systems where state logic is scattered across different nodes, this custom manager instantiates all possible states from a list of class references defined in a data asset. This approach ensures a modular and clean structure, where each state's logic is self-contained.

Key Features:
- State Instantiation: The manager creates a single instance for each state (UStateBase) defined in a data asset on BeginPlay.

- State Transitions: The RequestStateTreeEnter function handles all state changes. It first validates the transition using an EnterCondition() check on the new state and, if successful, calls OnExit() on the current state before calling OnEnter() on the new one.

- Centralized Decision-Making: UAC_StateManager delegates the decision-making process to the BehaviorDecisionComponent by calling the SelectNewBestAttack() function, ensuring a clear separation of concerns.

- Dynamic Debugging: A built-in debug mode visually displays the AI's current state in real-time within the world, a crucial feature for a complex system.

```c++
void UAC_StateManager::RequestStateTreeEnter(const FGameplayTag& StateTag)
{
    // ...
    UStateBase* FindedState = GetStateWithTag(StateTag);
    if (FindedState->EnterCondition())
    {
        if (CurrentState)
        {
            CurrentState->OnExit();
        }

        FindedState->OnEnter();
        CurrentState = FindedState;
    }
    // ...
}
```

#### **1.1 Movement State** 

The UMovementState is one of the most dynamic states within the AI system. Its primary purpose is to manage the AI's movement to get into the optimal position for its next action. It constantly evaluates the tactical situation and uses a sophisticated decision-making process to find the most suitable movement chain.

- OnEnter: When the AI enters this state, it immediately calls the SelectNewAttackAbility() function from the Behavior Decision component to determine the best attack. This is a crucial step as it dictates which movement chain should be executed to get the AI within range for that specific attack.

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

- OnTick: On every frame, the OnTick function calls TryEnterToAttackState(). This is the core of the state's logic; it continuously checks if the AI is now in range to perform the selected attack.

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

- StartMovementChain: The StartMovementChain function delegates the actual movement logic to the MovementManagerComponent. It also subscribes to a delegate (OnMovementChainEnded) to know when the movement sequence is complete.

- Exit Logic: The state's exit logic is a clear example of the system's reactive design. The AI will exit the Movement State and request to enter the Attack State as soon as it gets within range of its selected attack. If the movement chain ends before the AI is in range, it re-evaluates and may start a new movement chain or transition to a different state.



#### **1.2 Attack State** 

The UAttackStateBase is where the AI's offensive actions are managed. Once the AI has successfully positioned itself within a suitable range, this state takes over to execute the pre-selected attack ability. This state also handles the entire lifecycle of the attack, from activation to completion.

```c++
void UAttackStateBase::OnEnter_Implementation()
{
	Super::OnEnter_Implementation();
	Enemy->GetEnemyMovementManagerComponent()->StopMovementAbilities();
	SelectAndMakeAttack();
}
```

- OnEnter: Upon entering the AttackState, the AI immediately calls SelectAndMakeAttack(). It first gets the best attack data from the Behavior Decision component and then attempts to activate the corresponding ability. Any prior movement abilities are immediately stopped to allow the attack animation to take priority.
  
- Attack Execution: The MakeAttack() function uses the TryActivateAbilityByClassAndReturnInstance method to fire the attack. This method ensures that the ability is activated through the Gameplay Ability System (GAS), allowing for all the benefits of that framework (cooldowns, costs, etc.).

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

- Event-Driven Exit: The state does not rely on a tick to determine when to exit. Instead, it subscribes to the OnGameplayAbilityEndedWithDataBP delegate. Once the attack ability has completed its execution (e.g., the attack animation finishes), this delegate fires, triggering the ExitRequest and allowing the AI to transition to its next state.

#### **1.3 InComingAttack State** 

The UInComingAttackState is the AI's reactive, defensive state. It's triggered by the Intend Handler when the AI detects a significant incoming attack and must make a split-second decision on how to react.

- Decision and Action: The core of this state is the SelectAndMakeInComingAttackReaction() function, which delegates the decision-making to the Behavior Decision Component. Based on the highest-scoring defensive reaction (parry, take damage, etc.), it then calls the appropriate function to execute that action.

```c++
void UInComingAttackState::OnEnter_Implementation()
{
    Super::OnEnter_Implementation();
    SelectAndMakeInComingAttackReaction();
}

void UInComingAttackState::MakeTakeDamage(const UBDS_ComingAttackReactionBase* BestComingAttackReaction)
{
    if (DamageSubsystem)
    {
        if (!DamageSubsystem->OnDamageDealt.IsAlreadyBound(this, &UInComingAttackState::OnDamageDealt))
        {
            DamageSubsystem->OnDamageDealt.AddDynamic(this, &UInComingAttackState::OnDamageDealt);
        }
    }
    // ...
}

void UInComingAttackState::MakeParryAbility(const UBDS_ComingAttackReactionBase* BestComingAttackReaction)
{
    UGAS_GameplayAbilityBase* ActivatedParryAbility = EnemyASC->TryActivateAbilityByClassAndReturnInstance(EnemyParryAbilityClass);
    if (ActivatedParryAbility)
    {
        if (!ActivatedParryAbility->OnGameplayAbilityEndedWithDataBP.IsAlreadyBound(this, &UInComingAttackState::OnParryAbilityEnded))
        {
            ActivatedParryAbility->OnGameplayAbilityEndedWithDataBP.AddDynamic(this, &UInComingAttackState::OnParryAbilityEnded);
        }
    }
}
```
- Delegates for Event Handling: The state uses delegates extensively to manage its flow. For a TakeDamage reaction, it binds to the OnDamageDealt delegate to trigger the AI's own damage-receiving ability. For a Parry reaction, it listens for the OnGameplayAbilityEndedWithDataBP delegate of the parry ability to know when it's safe to exit the state.

- Execution and Transition: The MakeParryAbility() and MakeTakeDamage() functions are responsible for activating the relevant Gameplay Abilities. Once the defensive ability is complete or the damage is processed, the state calls ExitRequest, allowing the AI to seamlessly transition back to its primary combat loop (e.g., back to Movement or Attack states).

### **2. Intend Handler** 

The UAC_IntendHandlerBase component serves as the AI's perceptual layer, acting as a bridge between environmental stimuli and the AI's decision-making process. Its primary purpose is to identify critical moments—such as detecting a player, becoming vulnerable, or detecting an incoming attack—and to notify the State Manager to initiate a new behavioral sequence.

The Intend Handler achieves this by subscribing to various delegates and events, rather than constantly polling for changes. This event-driven approach is highly performant and responsive, ensuring the AI can react instantly to dynamic combat situations.

Key Functions:

- Target Detection: Scans the environment for valid targets and alerts the State Manager when a new one is found.

```c++
void UAC_IntendHandlerBase::OnTargetDetected(AActor* DetectedTarget)
{
	OwnerStateManager->OnTargetDetected();
}
```
- Event-Driven State Changes: The Intend Handler is the primary trigger for reactive state transitions. It registers delegates for critical events:

- Vulnerable State: When the enemy's Vulnerable tag is added (e.g., after a parry or stagger), the OnVulnerableTagAdded delegate fires, immediately pushing the AI into a Vulnerable state.

```c++
void UAC_IntendHandlerBase::OnVulnerableTagAdded(const UAbilitySystemComponent* AbilitySystemComponent, const FGameplayTag& Tag)
{
	OwnerStateManager->RequestStateTreeEnter(GAS_Tags::TAG_AI_State_Vulnerable);
}
```

- Incoming Attack & Reaction Logic: This system is the AI's proactive defensive layer, managing how the AI responds to detected incoming attacks from the player. It is a critical example of the tight integration between the Intend Handler, Behavior Decision, and State Manager components.

- Detection and Analysis: When the player activates a melee attack, the OnTargetAbilityActivated function within the Intend Handler processes this event. It calculates the exact ComingAttackHitTime and combines all relevant tags into a FComingAttackPayload.

- Decision-Making: The Intend Handler sends this payload to the Behavior Decision Component, which uses its GetBestComingAttackReaction service to determine the highest-scoring defensive reaction (e.g., parry, take damage, etc.).

- Execution with Precision: The SendEventToDefense function takes over to execute the chosen reaction. If the optimal reaction requires a specific timing (e.g., parrying a few frames before the hit), it uses a timer to delay the reaction, ensuring the AI performs the action at the precise moment for maximum effectiveness.

```c++
void UAC_IntendHandlerBase::SendEventToDefense(FComingAttackPayload EventPayload)
{
    UBDS_ComingAttackReactionBase* BestReaction = OwnerBehaviorDecisionComp->GetBestComingAttackReaction(EventPayload);
    // ...
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
    }
}

void UAC_IntendHandlerBase::TriggerIncomingAttackReaction(UBDS_ComingAttackReactionBase* Reaction, FComingAttackPayload Payload)
{
	OwnerStateManager->ComingAttackPayload = Payload;
	OwnerStateManager->RequestStateTreeEnter(GAS_Tags::TAG_AI_State_InComingAttack);
}
```

### **3. Behavior Decision** 

The UAC_BehaviorDecision component is the AI's tactical layer, responsible for choosing the next action (attack, movement, or a combination of both) based on a dynamic scoring system. It operates within the Combat state, as triggered by the StateManager. This design ensures that the AI's actions are context-aware and purposeful.



#### **Services** 

##### Core Services
The system is built on a service-oriented architecture, where each decision-making logic is encapsulated within its own class derived from UBehaviorDecisionServiceBase. This modular design prevents a single, cluttered component and allows for easy expansion with new AI behaviors.

- UBDS_GetBestAttack: This service is responsible for selecting the most suitable attack from a list of possibilities.

- UBDS_GetBestMovementChain: This service determines the optimal movement sequence for the AI, often to accompany an attack or to reposition itself.

- UBDS_ComingAttackReactionBase: This is a base class for services that decide how the AI should react to an incoming attack, such as dodging, parrying, or blocking.

The UAC_BehaviorDecision component initializes these services and delegates all decision-making tasks to them, acting as the central hub for the AI's tactical choices.

#### **Scoring** 

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




