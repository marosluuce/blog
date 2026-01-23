+++
date = '2026-01-21T09:18:24-05:00'
draft = false
title = 'An Unreal Action System'
+++

## Chipping Away At Awe

Unreal Engine is huge<sup>[citation needed]</sup>. It's billed as "The most powerful real-time 3D creation tool" and the 35.4 GB install size underscores that. I've wanted to learn Unreal for a long time but it's felt out of reach despite 13+ years of professional software development experience and 3+ years working with Unity for a large game studio. Perhaps it's the name or my rusty C++, though Rider helps with that.

Thankfully a friend recommended this [course by Tom Looman](https://courses.tomlooman.com/p/unrealengine-cpp). I'm more than half through the course and quite enjoying myself. Some aspects of working with Unreal seem very familiar now. This may be in part simply getting used to the editor and dusting off my C++.

A big source of comfort is recognizing familiar design patterns as I follow along and implement my own features. It shouldn't be surprising that software design patterns are applicable in Unreal. There's a mountain of things I haven't learned yet, but I feel possibility instead of dread.

You may be in a similar position to me, interested and intimidated. If that's the case, I want to walk through a common feature of games, an action system. The journey felt familiar to me and building it boosted my confidence.

This will focus on C++, though it could be done in Blueprints as well. For the sake of brevity, I will constrain this to the specifics of the action system and skip over things like imports, meshes, null checks, etc.I recommend having at least a little familiarity with Unreal already.

If you don't have any experience, this is a [good place to start](https://dev.epicgames.com/documentation/en-us/unreal-engine/first-hour-in-unreal-engine).

## A First Action
Almost every game lets players take actions. They can jump, swing a sword, or do any number of things. Casting a spell is common and it's fairly straightforward to write a casting function or two.

I'll begin by adding the ability to cast a fireball and magic missile.

The cast functions will:
- Trigger a casting animation
- Delay while the animation finishes
- Spawn the spell projectile

Then I'll bind casting the spell to trigger with a key press.

```cpp
// MyCharacter.h
UCLASS()
class EXAMPLE_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UInputAction> FireballAction;

    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UInputAction> MagicMissileAction;

    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UAnimMontage> SpellAnimation;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> FireballClass;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> MagicMissileClass;

    UPROPERTY(EditDefaultsOnly)
    float SpellCastDelay;

    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    void BeginCastFireball();

    void CastFireball();

    void BeginCastMagicMissile();

    void CastMagicMissile();
};
```

```cpp
// MyCharacter.cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    // Skipping Enhanced Input boilerplate, see Further Reading

    UEnhancedInputComponent* InputComponent =
        Cast<UEnhancedInputComponent>(PlayerInputComponent);
    InputComponent->BindAction(FireballAction, ETriggerEvent::Triggered,
        this, &AMyCharacter::BeginCastFireball);
    InputComponent->BindAction(MagicMissileAction, ETriggerEvent::Triggered,
        this, &AMyCharacter::BeginCastMagicMissile);
}

void AMyCharacter::BeginCastFireball()
{
    PlayAnimMontage(SpellAnimation);

    FTimerHandle AttackTimer;
    GetWorldTimerManager().SetTimer(AttackTimer, this,
        &AMyCharacter::CastFireball, SpellCastDelay);
}

void AMyCharacter::CastFireball()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParameters.Instigator = this;

    GetWorld()->SpawnActor<AActor>(FireballClass, GetTransform(), SpawnParameters);
}

void AMyCharacter::BeginCastMagicMissile()
{
    PlayAnimMontage(SpellAnimation);

    FTimerHandle AttackTimer;
    GetWorldTimerManager().SetTimer(AttackTimer, this,
        &AMyCharacter::CastMagicMissile, SpellCastDelay);
}

void AMyCharacter::CastMagicMissile()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParameters.Instigator = this;

    GetWorld()->SpawnActor<AActor>(MagicMissileClass, GetTransform(),
        SpawnParameters);
}
```

This works, but there are problems. There's duplication between the two spells. They share a casting animation and delay. Adding new spells requires two functions, one to start the animation and another to finish casting the spell. The problems only grow as more spells and casting actors are added.

## Concentrating Power

One way to fix this is to extract spell casting to a class. All the unique attributes of casting a spell can be collected, like which spell is being cast, how long does casting take, and which animation to use. Those properties can be wrapped in a class with a convenient trigger method, like `Cast`.

```cpp
// SpellCast.h
UCLASS()
class EXAMPLE_API USpellCast : public UObject
{
    GENERATED_BODY()

public:
    USpellCast();

    UFUNCTION(BlueprintCallable)
    void Cast(AActor* InstigatorActor);

    // Necessary due to deriving from UObject
    virtual UWorld* GetWorld() const override;

protected:
    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UAnimMontage> CastAnimation;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> SpellClass;

    UPROPERTY(EditDefaultsOnly)
    float AnimationDelay;

    UPROPERTY()
    AActor* Instigator;

    void SpawnSpell();
};
```

```cpp
// SpellCast.cpp
USpellCast::USpellCast()
{
    AnimationDelay = 0.2f;
}

void USpellCast::Cast(AActor* InstigatorActor)
{
    Instigator = InstigatorActor;
    if (ACharacter* Character = Cast<ACharacter>(Instigator))
    {
        Character->PlayAnimMontage(CastAnimation);

        FTimerHandle CastTimer;
        GetWorld()->GetTimerManager().SetTimer(CastTimer, this,
                &USpellCast::SpawnSpell, AnimationDelay);
    }
}

void USpellCast::SpawnSpell()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParameters.Instigator = Cast<APawn>(Instigator);

    GetWorld()->SpawnActor<AActor>(SpellClass,
        Instigator->GetTransform(), SpawnParameters);
}

UWorld* USpellCast::GetWorld() const
{
    // Fall back to the "owning" GetWorld
    if (AActor* Outer = GetTypedOuter<AActor>())
    {
        return Outer->GetWorld();
    }

    return nullptr;
}
```

Now all the spell casting code lives in a single class. Different spells can have unique casting animations and delays. Triggering a spell is just a call `Cast` with a single parameter, the instigating actor.

Using this pattern, it's much quicker to bind keys or npc logic to trigger different spells. Here's what `MyCharacter` looks like using `SpellCast`.

```cpp
// MyCharacter.h
UCLASS()
class EXAMPLE_API AMyCharacter : public ACharacter
{
    // ...
protected:
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<USpellCast> FireballClass;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<USpellCast> MagicMissileClass;

    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    void CastFireball();

    void CastMagicMissile();
};
```

```cpp
// MyCharacter.cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    // ...
    UEnhancedInputComponent* InputComponent =
        Cast<UEnhancedInputComponent>(PlayerInputComponent);
    InputComponent->BindAction(FireballAction, ETriggerEvent::Triggered,
        this, &AMyCharacter::CastFireball);
    InputComponent->BindAction(MagicMissileAction, ETriggerEvent::Triggered,
        this, &AMyCharacter::CastMagicMissile);
}

void AMyCharacter::CastFireball()
{
    USpellCast* Fireball = NewObject<USpellCast>(this, FireballClass);
    Fireball->Cast(this);
}

void AMyCharacter::CastMagicMissile()
{
    USpellCast* MagicMissile = NewObject<USpellCast>(this, MagicMissileClass);
    MagicMissile->Cast(this);
}
```

This is a big improvement. `MyCharacter` is much smaller, with fewer methods and parameters. Adding a new spell takes less time to wire up.

However `MyCharacter` can only cast spells and require changes in C++ to swap one spell for another, which is slow.

## Generalizing Actions

Introducing a generic action abstraction provides significantly more flexibility. To do this, I can add a new parent class with a name and an overridable `Execute` method.

```cpp
// Action.h
UCLASS()
class EXAMPLE_API UAction : public UObject
{
    GENERATED_BODY()

public:
    virtual void Execute(AActor* InstigatorActor);

    // Necessary due to deriving from UObject
    virtual UWorld* GetWorld() const override;

    UPROPERTY(EditDefaultsOnly)
    FName ActionName;
};
```

```cpp
// Action.cpp
void UAction::Execute(AActor* InstigatorActor)
{
}

UWorld* UAction::GetWorld() const
{
    // Fall back to the "owning" GetWorld
    if (AActor* Outer = GetTypedOuter<AActor>())
    {
        return Outer->GetWorld();
    }

    return nullptr;
}
```

To update the spell cast:
-  `SpellCast` derives from `Action`
-  `SpellCast` is renamed to `SpellCastAction` for easier grouping
- `Cast` is renamed to `Execute`, overriding it
- `GetWorld` is removed, using the implementation in `Action`

```cpp
// SpellCast.h => SpellCastAction.h
UCLASS()
class EXAMPLE_API USpellCastAction : public UAction
{
    // ...
public:
    // Renamed from Cast
    virtual void Execute(AActor* InstigatorActor) override;
    // ...
};
```

```cpp
// SpellCast => SpellCastAction.cpp
void USpellCastAction::Execute(AActor* InstigatorActor)
{
    // Renamed from Cast
}
```

At last, I can create a new `ActorComponent`. This will be a generic component that can be attached to any actor that wants to take actions. It will hold a list of actions granted when starting play and each action can be triggered by name.

```cpp
// ActionComponent.h
UCLASS()
class EXAMPLE_API UActionComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;

    UPROPERTY(EditDefaultsOnly)
    TArray<TSubclassOf<UAction>> StartingActions;

    UPROPERTY()
    TArray<TObjectPtr<UAction>> Actions;

    UFUNCTION(BlueprintCallable)
    void ExecuteByName(FName ActionName);
};
```

```cpp
// ActionComponent.cpp
void UActionComponent::BeginPlay()
{
    for (TSubclassOf<UAction> Action : StartingActions)
    {
        UAction* ToAdd = NewObject<UAction>(this, Action);
        Actions.Add(ToAdd);
    }
}

void UActionComponent::ExecuteByName(FName ActionName)
{
    for (TObjectPtr<UAction> Action : Actions)
    {
        if (Action->ActionName == ActionName)
        {
            Action->Execute(GetOwner());
            break;
        }
    }
}
```

Adding this component to any actor allows it to maintain a unique list of actions available to them. Here is what `MyCharacter` looks like using `ActionComponent`.

```cpp
// MyCharacter.h
UCLASS()
class EXAMPLE_API AMyCharacter : public ACharacter
{
    // ...
public:
    AMyCharacter();

protected:
    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UActionComponent> ActionComponent;

    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UInputAction> PrimaryAction;

    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UInputAction> SecondaryAction;

    UPROPERTY(EditDefaultsOnly)
    FName PrimaryActionName;

    UPROPERTY(EditDefaultsOnly)
    FName SecondaryActionName;

    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    void TriggerPrimaryAction();

    void TriggerSecondaryAction();
    // ...
};
```

```cpp
// MyCharacter.cpp
AMyCharacter::AMyCharacter()
{
    ActionComponent = CreateDefaultSubobject<UActionComponent>("ActionComponent");
}

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    // ...
    UEnhancedInputComponent* InputComponent =
        Cast<UEnhancedInputComponent>(PlayerInputComponent);
    InputComponent->BindAction(PrimaryAction, ETriggerEvent::Triggered,
        this, &AMyCharacter::TriggerPrimaryAction);
    InputComponent->BindAction(SecondaryAction, ETriggerEvent::Triggered,
        this, &AMyCharacter::TriggerSecondaryAction);
}

void AMyCharacter::TriggerPrimaryAction()
{
    ActionComponent->ExecuteByName(PrimaryActionName);
}

void AMyCharacter::TriggerSecondaryAction()
{
    ActionComponent->ExecuteByName(SecondaryActionName);
}
```

With this component it's very easy to add it to other actors, like a goblin wizard, with a distinct list of actions.

```cpp
// GoblinWizard.h
UCLASS()
class EXAMPLE_API AGoblinWizard : public ACharacter
{
    GENERATED_BODY()
public:
    AGoblinWizard();

protected:
    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UActionComponent> ActionComponent;

    UPROPERTY(EditDefaultsOnly)
    FName PrimaryActionName;

    UFUNCTION(BlueprintCallable)
    void TriggerPrimaryAction();
};
```

```cpp
// GoblinWizard.cpp
AGoblinWizard::AGoblinWizard()
{
    ActionComponent = CreateDefaultSubobject<UActionComponent>("ActionComponent");
}

void AGoblinWizard::TriggerPrimaryAction()
{
    ActionComponent->ExecuteByName(PrimaryActionName);
}
```

In addition to adding `ActionComponent`, I've used primary action and/or secondary action on `MyCharacter` and `GoblinWizard`. A huge benefit to this approach is iteration speed. An actor's starting actions and what primary or secondary actions trigger can be configured in the editor. This is significantly faster than updating and recompiling every change in C++.

## Next Steps

At this point, I'm done for now. The `Action` allows any actor to do whatever I can dream up. I could add an action that lets a character turn on a light or a stranger one like transforming into a car for a minute. The possibilities are endless.

The `ActionComponent` can be extended too. It could allow a character to "learn" a new spell when they read a book, adding a new spell to their list of actions. It could also be improved to prevent multiple actions from triggering at the same time or to avoid relying on strings to trigger the desired action. I'm sure there's a lot more that I haven't thought of.

## Wrapping Up

This is a good exercise in building reusable pieces in Unreal and it should show that it's not too scary working in Unreal. It made me feel more confident and hopefully it helps you too. That said, for a real project it's probably a good idea to look into Unreal's provided ability system instead of writing something from scratch, even if it is fun.

## Further Reading

- [Unreal's Official Game Ability System](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-for-unreal-engine)
- [Enhanced Input Tutorial](https://dev.epicgames.com/community/learning/tutorials/6dp3/unreal-engine-using-the-enhancedinput-system-in-c)
- [Enhanced Input Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/enhanced-input-in-unreal-engine)
