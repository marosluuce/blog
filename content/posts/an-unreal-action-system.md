+++
date = '2026-01-21T09:18:24-05:00'
draft = true
title = 'An Unreal Action System'
+++

## Chipping Away At Awe

Unreal Engine is huge<sup>[citation needed]</sup>. It's billed as "The most powerful real-time 3D creation tool" and it is daunting to look at the 35.4 GB install size. I've wanted to learn Unreal for a long time but it's felt out of reach despite 13+ years of professional software development experience and 3+ years working with Unity for a large game studio. Perhaps it's the name or my inexperience with C++, though Rider helps offset that.

Thankfully a friend recommended this [course by Tom Looman](https://courses.tomlooman.com/p/unrealengine-cpp). After several aborted attempts, I'm more than half through the course and am enjoying myself. Some aspects of working with Unreal seem very familiar now. This may in part be simply getting used to the editor and dusting off my C++.

A big source of comfort is recognizing familiar design patterns as I follow along and implement my own features. It shouldn't be surprising that software design patterns are applicable in Unreal. There's still a mountain of things I still don't understand, but it's paired with a feeling of possibility instead of dread.

You may be in a similar position to me, interested and intimidated. If that's the case, I want to walk through building a common feature of games, an action system. It's something that felt familiar to me while learning Unreal and built up my confidence.

This will focus on C++, though it could be built in Blueprints as well. For the sake of brevity, I will constrain this to building the action system and skip over things like models, collisions, null checks, etc. I recommend having some familiarity with Unreal already.

If you don't have any experience, this is a [good place to start](https://dev.epicgames.com/documentation/en-us/unreal-engine/first-hour-in-unreal-engine).

## A First Action
Almost every game lets players take actions. They can jump, swing a sword, or do any number of other things. Casting a spell is common and it's fairly straightforward to write a casting function or two.

I'll start by adding the ability to cast a fireball and magic missile.

The cast functions will:
- Trigger a casting animation
- Delay while the animation finishes
- Spawn the spell projectile

Then they'll be bound to trigger with a key press.

```cpp
// MyCharacter.h
UCLASS()
class EXAMPLE_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsAnywhere)
    TSubclassOf<UAnimMontage> SpellAnimation;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> FireballClass;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> MagicMissileClass;

    UPROPERTY(EditDefaultsAnywhere)
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

This works, but already has scaling issues. Adding new spells requires two functions. Changes to spell casting require changes in each spell function. The spells also share a casting animation and delay. These limitations only grow over time as more spells and casting actors are added.

## Concentrating Power

One way to centralize the spell casting behavior is to use the Command pattern. All the unique attributes of casting a spell can be collected, like which spell is being cast, how long does casting take, and which animation to use. Those properties can be wrapped in an spell action class with a convenient trigger method, like `Execute`.

```cpp
// SpellAction.h
UCLASS()
class EXAMPLE_API USpellAction : public UObject
{
    GENERATED_BODY()

public:
    USpellAction();

    void Execute(AActor* InstigatorActor);

protected:
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UAnimMontage> SpellAnimation;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> SpellClass;

    UPROPERTY(EditAnywhere)
    float AnimationDelay;

    UPROPERTY()
    TObjectPtr<AActor*> Instigator;

    void SpawnSpell();
}
```

```cpp
// SpellAction.cpp
USpellAction()
{
    AnimationDelay = 0.2f;
}

void USpellAction::Execute(AActor* InstigatorActor)
{
    Instigator = InstigatorActor;
    if (ACharacter* Character = Cast<ACharacter>(Instigator))
    {
        Instigator->PlayAnimMontage(CastAnimation);

        FTimerHandle CastTimer;
        GetWorldTimerManager().SetTimer(CastTimer, this,
                &USpellAction::CastSpell, AnimationDelay);
    }
}

void USpellAction::CastSpell()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = Instigator;

    GetWorld()->SpawnActor<AActor>(SpellClass, GetTransform(), SpawnParameters);
}
```

Now all the spell casting code lives in a single class. Different spells can have unique casting animations and delays. Triggering a spell is just a call `Execute` with a single parameter, the instigating actor. There's probably a way to grab the actor during creation, but this is simple enough.

Using this pattern, it's much quicker to bind keys or npc logic to trigger different spells. Here's what the character class looks like using this new spell action.

```cpp
// MyCharacter.h
UCLASS()
class EXAMPLE_API AMyCharacter : public ACharacter
{
    // ...
protected:
    UPROPERTY(EditDefaultsAnywhere)
    TObjectPtr<USpellAction> Fireball;

    UPROPERTY(EditDefaultsAnywhere)
    TObjectPtr<USpellAction> MagicMissile;

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
    Fireball->Execute(this);
}

void AMyCharacter::CastMagicMissile()
{
    MagicMissile->Execute(this);
}
```

This is a big improvement. The character class is much smaller, with fewer methods and parameters. Adding a new spell takes less time to wire up.

However the character still can only cast spells and requires changes in C++ to swap one spell for another.

## Generalizing Actions

Adding a generic action abstraction provides significantly more flexibility. So I can a new parent class with a name and an `Execute` method. Other classes will override and implement their own `Execute`.

```cpp
// Action.h
UCLASS()
class EXAMPLE_API UAction : public UObject
{
    GENERATED_BODY()

public:
    virtual void Execute(AActor* InstigatorActor);

private:
    UPROPERTY(EditDefaultsAnywhere)
    FName ActionName;
}
```

```cpp
// Action.cpp
virtual void Execute(AActor* InstigatorActor)
{
}
```

Now the spell action will derive from the action class, overriding `Execute`.

```cpp
// SpellAction.h
UCLASS()
class EXAMPLE_API USpellAction : public UAction
{
    // ...
public:
    virtual void Execute(AActor* InstigatorActor);
    // ...
}
```

```cpp
// SpellAction.cpp
void Execute(AActor* InstigatorActor)
{
    // No changes necessary
}
```

At last, I can create a new ActorComponent. This will be attached to any actor that needs to take actions. It will hold a list of actions and they can be triggered by name.

```cpp
// ActionComponent.h
UCLASS()
class EXAMPLE_API UActionComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;

    UPROPERTY(EditDefaultsAnywhere)
    TArray<TSubclassOf<UAction>> DefaultActions;

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
    for (TSubclassOf<UAction> Action : DefaultActions)
    {
        UAction* ToAdd = NewObject<UAction>(this, Action);
        Actions.Add(ToAdd);
    }
}

void UActionComponent::Execute(FName ActionName)
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
    UPROPERTY(EditDefaultsAnywhere)
    TObjectPtr<UActionComponent> ActionComponent;

    UPROPERTY(EditDefaultsAnywhere)
    FName PrimaryAction;

    UPROPERTY(EditDefaultsAnywhere)
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

In addition to adding the new action component, I've updated the character to have a primary and secondary attack, instead of specifically named spells. This allows the character's actions to be configured in the editor by changing the primary and secondary action names. This is faster than recompiling every change in C++, allowing for faster iteration.

With this component it's also very easy to add it to other actors, like a goblin wizard, with their own list of actions.

```cpp
// GoblinWizard.h
UCLASS()
class EXAMPLE_API AGoblinWizard : public ACharacter
{
    GENERATED_BODY()
public:
    AGoblinWizard();

protected:
    UPROPERTY(EditDefaultsAnywhere)
    TObjectPtr<UActionComponent> ActionComponent;

    UPROPERTY(EditDefaultsAnywhere)
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

## Wrapping Up

At this point, the action system is complete enough but far from finished. Nothing prevents multiple actions from triggering at the same time. The action system could be extended to support an effect with duration or actions that trigger immediately. Actions could be granted as part of gameplay, instead of being preconfigured. Or maybe the character is stunned and can't perform an action for a period of time. The possibilities are endless.

This is a good exercise in building reusable pieces in Unreal and seeing how it's not too scary working in Unreal. Hopefully it builds confidence, like it did for me. That said, for a real project it's probably a good idea to look into Unreal's provided ability system instead of writing something from scratch.

## Further Reading

- [Unreal's Official Game Ability System](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-for-unreal-engine)
- [Enhanced Input Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/enhanced-input-in-unreal-engine)
