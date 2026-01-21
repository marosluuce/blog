+++
date = '2026-01-21T09:18:24-05:00'
draft = true
title = 'An Unreal Ability System'
+++

## Chipping Away At Awe

Learning Unreal Engine is daunting<sup>[citation needed]</sup>. It's billed as "The most powerful real-time 3D creation tool" and I feel that power when I look at the 35.4 GB install size. I've been curious for a long time but learning Unreal has felt out of reach despite 13+ years of professional software development experience and 3+ years working in Unity for a large game studio. Perhaps it's the name or maybe my inexperience with C++, though Rider helps offset that.

Thankfully I found this [course by Tom Looman](https://courses.tomlooman.com/p/unrealengine-cpp). After several aborted attempts, I've compled more than half the course and am thoroughly enjoying myself. Some aspects of working with Unreal seem very familiar now. This may be due to getting used to the editor and dusting off my C++. One big factor is recognizing familiar design patterns as I follow along and implement my own features. It shouldn't be surprising that software design patterns are applicable in Unreal. There's still a mountain of things I still don't understand, but it's paired with a feeling of possibility instead of dread.

You may be in a similar position to me, interested and intimidated. In that case, I want to share a journey of building a common feature of most games, an ability system. It's what first felt familiar to me while learning Unreal and built up my confidence.

This will focus on C++, though it should be applicable to Blueprints as well. For the sake of brevity, I will try to constrain myself to building the ability system. I'll skip over things like models, collisions, null checks, etc.

## Casting Spells
Almost every game gives players abilities. They can jump, swing a sword, or do any number of other things. It's fairly straighforward to write a function for a character to cast a spell.

The functions will:
- Trigger a casting animation
- Delay while the animation finishes
- Spawn the spell projectile

Then the function can be bound to trigger with a key press.

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
    // Skipping some Enhanced Input boilerplate, see Further Reading

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
This example binds two actions to cast two different spells. They share a casting animation and delay. It's not too much code, but there's already some duplicated logic.

I  can also create an enemy, like a goblin wizard, who can also cast spells. A fast way to implement that is to copy-and-paste the fireball code into the new actor class.

```cpp
// GoblinWizard.h
UCLASS()
class EXAMPLE_API AGoblinWizard : public ACharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UAnimMontage> SpellAnimation;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> FireballClass;

    UPROPERTY(EditDefaultsAnywhere)
    float SpellCastDelay;
}
```

```cpp
// GoblinWizard.cpp
void AGoblinWizard::BeginCastFireball()
{
    PlayAnimMontage(SpellAnimation);

    FTimerHandle AttackTimer;
    GetWorldTimerManager().SetTimer(AttackTimer, this,
        &AGoblinWizard::CastFireball, SpellCastDelay);
}

void AGoblinWizard::BeginCastFireball()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = this;

    GetWorld()->SpawnActor<AActor>(FireballClass, GetTransform(), SpawnParameters);
}
```

This works, but has already has scaling issues. Adding a new spell requires re-implementing the spell function each time. Any changes to spell casting would require changes in each spell function. This burden only increases over time as more spells and casting actors are added.

## Concetrating Power

One way to centralize spell casting is to use the Command pattern. I can collect all the unique attributes of casting a spell, like which spell is being cast, how long does casting take, and which animation to use. Then those properties can be wrapped in an spell casting class with a convenient trigger method, like `execute`.

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

Now all the spell casting code lives in a single class. Different spell actions can have unique animations and delays. Triggering a spell is just a call `Execute` with a single parameter, the instigating actor.

Using this pattern, it's now much quicker to bind keys or npc logic to trigger different spells. Here's what the character class looks like using this new spell action.

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

    void BindInputs();
    void CastFireball();
    void CastMagicMissile();
}
```

```cpp
// MyCharacter.cpp
void AMyCharacter::BindInputs()
{
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

The character class is much smaller, with fewer methods and parameters. Adding a new spell takes less time to wire up. Overall, this is a big improvement.

However this spell action is still specific to spells and changes in C++ are required to swap one spell for another.

## Generalizing Actions

Adding another layer of abstraction provides significantly more flexibility. First I can create an action class with a name and an execute method.

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

Next the spell action should derive from the new action class and override `Execute`.

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

Finally I can create a new ActorComponent to instantiate and hold a list of actions, executable by name.

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
            Action.execute(this);
            break;
        }
    }
}
```

This new component can be added to any actor allowing them to maintain a unique list of actions available to them.

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

Now the character's actions can be configured in the editor by changing the primary and secondary action names. This is faster than recompiling every change in C++, allowing for faster iteration. It's also very easy to add this action component to other actors, like that goblin wizard, with their own list of actions.

## Wrapping Up

At this point, the action system is complete but far from finished. Nothing prevents multiple actions from triggering at the same time. The action system could be extended to support effect durations or actions that trigger immediately. Actions could be granted as part of gameplay, instead of being preconfigured. Or maybe the character is stunned and loses the access to an action for a period of time. The possibilities are endless.

This is a good exercise in building reusable pieces in Unreal and seeing how small. Hopefully it gives confidence in trying more things in Unreal, like it did for me. That said, for a real project it's probably a good idea to look into Unreal's provided ability system instead of writing something from scratch.

## Further Reading

- [Unreal's Official Game Ability System](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-for-unreal-engine)
- [Enhanced Input Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/enhanced-input-in-unreal-engine)
