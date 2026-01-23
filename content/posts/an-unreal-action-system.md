+++
date = '2026-01-21T09:18:24-05:00'
draft = true
title = 'An Unreal Action System'
+++

## Chipping Away At Awe

Unreal Engine is huge<sup>[citation needed]</sup>. It's billed as "The most powerful real-time 3D creation tool" and the 35.4 GB install size underscores that. I've wanted to learn Unreal for a long time but it's felt out of reach despite 13+ years of professional software development experience and 3+ years working with Unity for a large game studio. Perhaps it's the name or my rusty C++, though Rider helps with that.

Thankfully a friend recommended this [course by Tom Looman](https://courses.tomlooman.com/p/unrealengine-cpp). I'm more than half through the course and quite enjoying myself. Some aspects of working with Unreal seem very familiar now. This may be in part simply getting used to the editor and dusting off my C++.

A big source of comfort is recognizing familiar design patterns as I follow along and implement my own features. It shouldn't be surprising that software design patterns are applicable in Unreal. There's a mountain of things I haven't learned yet, but I feel possibility instead of dread.

You may be in a similar position to me, interested and intimidated. If that's the case, I want to walk through a common feature of games, an action system. The journey felt familiar to me and building it boosted my confidence.

This will focus on C++, though it could be done in Blueprints as well. For the sake of brevity, I will constrain this to the specifics of the action system and skip over things like import, models, null checks, etc. I recommend having at least a little familiarity with Unreal already.

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
    TSubclassOf<UAnimMontage> SpellAnimation;

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
}
```

```cpp
// MyCharacter.cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
{
    // Skipping Enhanced Input boilerplate, see Further Reading

    UEnhancedInputComponent* InputComponent =
        Cast<UEnhancedInputComponent>(PlayerInputComponent);
    InputComponent->BindAction(SpellAction1, ETriggerEvent::Triggered,
        this, &MyCharacter::BeginCastFireball);
    InputComponent->BindAction(SpellAction2, ETriggerEvent::Triggered,
        this, &MyCharacter::BeginCastMagicMissile);
}

void AMyCharacter::BeginCastFireball()
{
    PlayAnimMontage(SpellAnimation);

    FTimerHandle AttackTimer;
    GetWorldTimerManager().SetTimer(AttackTimer, this,
        &MyCharacter::CastFireball, SpellCastDelay);
}

void AMyCharacter::CastFireball()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = this;

    GetWorld()->SpawnActor<AActor>(FireballClass, GetTransform(), SpawnParameters);
}

void AMyCharacter::BeginCastMagicMissile()
{
    PlayAnimMontage(SpellAnimation);

    FTimerHandle AttackTimer;
    GetWorldTimerManager().SetTimer(AttackTimer, this,
        &MyCharacter::CastMagicMissile, SpellCastDelay);
}

void AMyCharacter::CastMagicMissile()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = this;

    GetWorld()->SpawnActor<AActor>(MagicMissileClass, GetTransform(),
        SpawnParameters);
}
```

This works, but a lot of duplication between the two spells. Adding new spells requires two functions, one to start the animation and another to finish casting the spell. Changes to spell casting require changes in each spell function. The spells also share a casting animation and delay. These limitations only grow over time as more spells and casting actors are added.

## Concentrating Power

One way to centralize the spell casting behavior extracting a spell cast class. All the unique attributes of casting a spell can be collected, like which spell is being cast, how long does casting take, and which animation to use. Those properties can be wrapped in an spell cast class with a convenient trigger method, like `Cast`.

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

protected:
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UAnimMontage> SpellAnimation;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> SpellClass;

    UPROPERTY(EditDefaultsOnly)
    float AnimationDelay;

    UPROPERTY()
    TObjectPtr<AActor*> Instigator;

    void SpawnSpell();
}
```

```cpp
// SpellCast.cpp
USpellCast()
{
    AnimationDelay = 0.2f;
}

void USpellCast::Cast(AActor* InstigatorActor)
{
    Instigator = InstigatorActor;
    if (ACharacter* Character = Cast<ACharacter>(Instigator))
    {
        Instigator->PlayAnimMontage(CastAnimation);

        FTimerHandle CastTimer;
        GetWorldTimerManager().SetTimer(CastTimer, this,
                &USpellAction::SpawnSpell, AnimationDelay);
    }
}

void USpellCast::SpawnSpell()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = Instigator;

    GetWorld()->SpawnActor<AActor>(SpellClass, GetTransform(), SpawnParameters);
}
```

Now all the spell casting code lives in a single class. Different spells can have unique casting animations and delays. Triggering a spell is just a call `Cast` with a single parameter, the instigating actor.

Using this pattern, it's much quicker to bind keys or npc logic to trigger different spells. Here's what the character class looks like using this new spell class.

```cpp
// MyCharacter.h
UCLASS()
class EXAMPLE_API AMyCharacter : public ACharacter
{
    // ...
protected:
    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<USpellCast> Fireball;

    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<USpellCast> MagicMissile;

    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    void CastFireball();

    void CastMagicMissile();
}
```

```cpp
// MyCharacter.cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
{
    // ...
    InputComponent->BindAction(SpellAction1, ETriggerEvent::Triggered,
        this, &MyCharacter::CastFireball);
    InputComponent->BindAction(SpellAction2, ETriggerEvent::Triggered,
        this, &MyCharacter::CastMagicMissile);
}

void AMyCharacter::CastFireball()
{
    Fireball->Cast(this);
}

void AMyCharacter::CastMagicMissile()
{
    MagicMissile->Cast(this);
}
```

This is a big improvement. The character class is much smaller, with fewer methods and parameters. Adding a new spell takes less time to wire up.

However the character still can only cast spells and requires changes in C++ to swap one spell for another.

## Generalizing Actions

Introducing a generic action abstraction provides significantly more flexibility. To do this, I can add a new parent class with a name and an `Execute` method. Other classes will override and implement their own `Execute`.

```cpp
// Action.h
UCLASS()
class EXAMPLE_API UAction : public UObject
{
    GENERATED_BODY()

public:
    virtual void Execute(AActor* InstigatorActor);

private:
    UPROPERTY(EditDefaultsOnly)
    FName ActionName;
}
```

```cpp
// Action.cpp
virtual void Execute(AActor* InstigatorActor)
{
}
```

Now spell cast will derive from the action class. `Cast` will be renamed to `Execute` and override it. It'll also be easier to group this class with other actions if `SpellCast` becomes `SpellCastAction`.

```cpp
// SpellCast.h => SpellCastAction.h
UCLASS()
class EXAMPLE_API USpellCastAction : public UAction
{
    // ...
public:
    virtual void Execute(AActor* InstigatorActor);
    // ...
}
```

```cpp
// SpellCast => SpellCastAction.cpp
void Execute(AActor* InstigatorActor)
{
    // Renamed from Cast, no changes necessary
}
```

At last, I can create a new ActorComponent. This will be a generic action component that can be attached to any actor that wants to take actions. It will hold a list of actions granted when starting play and each action can be triggered by name.

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
}
```

```cpp
// ActionComponent.cpp
void UActionComponent::BeginPlay() override
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
        if (Action->Name == ActionName)
        {
            Action.execute(GetOwner());
            break;
        }
    }
}
```

Adding this component to any actor allows it to maintain a unique list of actions available to them. Here is what the character looks like using this new component;

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
    FName PrimaryAction;

    UPROPERTY(EditDefaultsOnly)
    FName SecondaryAction;

    void PrimaryAction();
    void SecondaryAction();
    // ...
}
```

```cpp
// MyCharacter.cpp
AMyCharacter()
{
    ActionComponent = CreateDefaultSubobject<UActionComponent>("ActionComponent");
}

void AMyCharacter::PrimaryAction()
{
    ActionComponent->ExecuteByName(PrimaryAction);
}

void AMyCharacter::SecondaryAction()
{
    ActionComponent->ExecuteByName(SecondaryAction);
}
```

With this component it's very easy to add it to other actors, like a goblin wizard, with their own list of actions.

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
    FName PrimaryAction;
}
```

```cpp
// GoblinWizard.cpp
AGoblinWizard()
{
    ActionComponent = CreateDefaultSubobject<UActionComponent>("ActionComponent");
}

void AGoblinWizard::PrimaryAction()
{
    ActionComponent->ExecuteByName(PrimaryAction);
}
```

A huge benefit to this approach is iteration speed. In addition to adding the new action component, I've used primary action and/or secondary action on the character and goblin wizard. This allows the actions to be configured in the editor by changing names of the action. This is significantly faster than updating and recompiling every change in C++.

## Next Steps

At this point, I'm done for now. The `Action` allows any actor to do whatever I can dream up. I could add an action that lets a character turn on a light or a stranger one like transforming into a car for a minute. The possibilities are endless.

The `ActionComponent` can be extended too. It could allow a character to "learn" a new spell when they read a book, adding a new spell to their list of actions. It could also be improved to prevent multiple actions from triggering at the same time or to avoid relying on strings to use the desired action. I'm sure there's a lot more that could be done.

## Wrapping Up

This is a good exercise in building reusable pieces in Unreal and it should show that it's not too scary working in Unreal. It made me feel more confident and hopefully it helps you too. That said, for a real project it's probably a good idea to look into Unreal's provided ability system instead of writing something from scratch, even if it is fun.

## Further Reading

- [Unreal's Official Game Ability System](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-for-unreal-engine)
- [Enhanced Input Tutorial](https://dev.epicgames.com/community/learning/tutorials/6dp3/unreal-engine-using-the-enhancedinput-system-in-c)
- [Enhanced Input Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/enhanced-input-in-unreal-engine)
