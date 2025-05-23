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


https://github.com/user-attachments/assets/b5b8e26f-11a1-4754-aece-478dfb53e03a


## Contents
- [Gameplay Systems](#Gameplay-Systems)
     - [Target Lock System](#1-Target-Lock-System)
- [Combat Abilities](#Combat-Abilities)
     - [Combo Abilities](#1-Combo-Abilities) 
     - [Parry](#2-Parry)
     - [Shadow Attack](#3-Shadow-Attack)
     - [Shadow Finisher](#3-Shadow-Finisher)
- [AI](#AI)
     - [AI Behavior](#1-AI-Behavior)
     - [Scoring](#2-Respawn-Handling)


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

 UAC_BehaviorDecision – AI Attack & Movement Decision System (Summary)
🔹 Purpose
This component handles attack and movement chain selection for enemy AI based on scoring logic.
Decisions are made using distance to the player, cooldowns, behavior state, and whether the player is moving.

- Selects the most suitable attack ability.

```c++
FAttackData UAC_BehaviorDecision::GetBestAttack(float DistanceToTarget)
{
    if (!AttackAbilityAsset || !OwnerEnemyASC)
    {
        UE_LOG(LogTemp, Warning, TEXT("AttackAbilityAsset or OwnerEnemyASC is null in: %s !"), *GetName());
        return FAttackData();
    }

    float BestScore = -FLT_MAX;
    FAttackData BestAttack;

    for (const FAttackData& Attack : AttackAbilityAsset->AttackAbilities)
    {
        if (!Attack.AbilityClass) 
        {
            continue;
        }

        bool bIsOnCooldown = OwnerEnemyASC->HasMatchingGameplayTag(Attack.AbilityCooldownTag);
        if (bIsOnCooldown) 
        {
            continue;
        }

        float DistanceScore = CalculateAttackAbilityScoreBasedOnTargetDistance(DistanceToTarget, Attack.MinRange, Attack.MaxRange);

        float TotalScore = Attack.ScoreBias + DistanceScore;

        UE_LOG(LogTemp, Log, TEXT("[AI] Attack %s → Score: %.2f"), *Attack.AbilityClass->GetName(), TotalScore);

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

- Considers cooldown status, distance, behavior modifiers, and score bias.

![image](https://github.com/user-attachments/assets/ee2008a4-0939-410b-a892-146fd0b9d1c9)


GetBestMovementChain(TSubclassOf<UGAS_GameplayAbilityBase>)
→ Picks the best movement chain linked to the selected attack.

```c++
TArray<FMovementAbilityData> UAC_BehaviorDecision::GetBestMovementChain(TSubclassOf<UGAS_GameplayAbilityBase> SelectedAbilityClass)
{
    if (!SelectedAbilityClass || !AttackAbilityMovementChainMapAsset)
    {
        return TArray<FMovementAbilityData>();
    }

    UMovementChainAsset* BestMovementChainDataAsset = nullptr;

    float BestScore = -FLT_MAX;
    float BestMovementChainDistanceScore = 0.f;
    float BestMovementChainTargetMovementScore = 0.f;

    TArray<UMovementChainAsset*> AbilityMovementChainAssets = GetMovementChainsForSelectedAttackAbility(SelectedAbilityClass);
    for (UMovementChainAsset* MovementChainAsset : AbilityMovementChainAssets)
    {
        if (GetTargetDistance() < MovementChainAsset->MinRange)
        {
            continue;
        }

        float DistanceScore = CalculateMovementChainScoreBasedOnTargetDistance(MovementChainAsset);
        float TargetMovementScore = CalculateMovementChainScoreBasedOnTargetMovement(MovementChainAsset);

        float TotalScore = MovementChainAsset->ScoreBias + DistanceScore + TargetMovementScore;

        if (TotalScore > BestScore)
        {
            BestScore = TotalScore;
            BestMovementChainDistanceScore = DistanceScore;
            BestMovementChainTargetMovementScore = TargetMovementScore;
            BestMovementChainDataAsset = MovementChainAsset;
        }
    }

    ApplyDirectionPoliciesToSelectedMovementChain(BestMovementChainDataAsset);

    if (GEngine && EnableSelectedDebug)
    {
        GEngine->AddOnScreenDebugMessage(10, 3.5f, FColor::Cyan,
            FString::Printf(TEXT(">> Selected MovementChain: %s | DistanceScore: %.1f | TargetMovementScore: %.1f "),
                *BestMovementChainDataAsset->MovementChainName.ToString(), BestMovementChainDistanceScore, BestMovementChainTargetMovementScore));
    }

    return BestMovementChainDataAsset->MovementChain;
}
}
```


- Uses scoring based on distance, target movement, and behavior state.

![Ekran görüntüsü 2025-04-08 203016](https://github.com/user-attachments/assets/0d3c1f2a-014f-401c-aa8c-e54f421c2181)

![Ekran görüntüsü 2025-04-08 203150](https://github.com/user-attachments/assets/86c9b586-279b-4f5f-970e-aac7f4be6cf1)


ApplyDirectionPoliciesToSelectedMovementChain()
→ Dynamically sets movement directions based on player input or randomization.

```c++
bool UAC_BehaviorDecision::ApplyDirectionPoliciesToSelectedMovementChain(UMovementChainAsset* SelectedMovementChainAsset)
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

This system is designed to be used inside a State Tree, where selected abilities and movement chains are executed after decision-making.

![Ekran görüntüsü 2025-04-08 203341](https://github.com/user-attachments/assets/586c8921-8f07-4097-b9c3-1969eaa7a5c0)

