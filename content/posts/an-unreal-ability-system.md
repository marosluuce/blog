+++
date = '2026-01-21T09:18:24-05:00'
draft = true
title = 'An Unreal Ability System'
+++

## Chipping Away At Awe

Learning Unreal Engine is daunting<sup>[citation needed]</sup>. It's billed as "The most powerful real-time 3D creation tool" and I feel that power when I look at the 35.4 GB install size. I've been intrigued for a long time but learning Unreal has felt out of reach despite 13+ years of professional software development experience and 3+ years working in Unity for a large game studio. Perhaps it's the name or maybe my inexperience with C++, though Rider helps offset that.

Thankfully I found this [course by Tom Looman](https://courses.tomlooman.com/p/unrealengine-cpp). After several aborted attempts, I've compled more than half the course and am thoroughly enjoying myself. Some aspects of working with Unreal seem very familiar after the initial hump. This may be due to time spent getting used to the editor and dusting off my C++. A bigger factor is recognizing familiar design patterns as I follow along and implement my own features. It shouldn't be a surprise that software design patterns are applicable in Unreal. There's still a mountain of things I still don't understand, but it's paired with a feeling of possibility now instead of dread.

You may be in a similar position to me, interested and intimidated. In that case, I want to share an example of how to build a building block of most games, an ability system. It's what first felt familiar to me while learning Unreal and built up my confidence. This example will be in C++, though it should be applicable to Blueprints as well.

## Casting Spells
In every almost every game, players have abilities. Players can jump, cast a spell, or swing a sword. To let a character cast a spell, it's straightforward to write a function that triggers an casting animation, waits, and spawns a spell then bind a key so that every key press triggers that function.

This approach works at first and scales all right as the player can cast more spells. What if npcs can cast spells too? Implementing each kind of spell cast separately is one option.

```cpp
// MyCharacter.h
UCLASS()
class EXAMPLE_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsAnywhere)
    TSubclassOf<UAnimMontage> FireballAnimation;

    UPROPERTY(EditDefaultsAnywhere)
    TSubclassOf<UAnimMontage> MagicMissileAnimation;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> FireballClass;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> MagicMissileClass;

    UPROPERTY(EditDefaultsAnywhere)
    float FireballTimeout;

    UPROPERTY(EditDefaultsAnywhere)
    float MagicMissileTimeout;

    void BindInputs();
    void BeginCastFireball();
    void CastFireball();
    void BeginCastMagicMissile();
    void CastMagicMissile();
}
```

```cpp
// MyCharacter.cpp
void AMyCharacter::BindInputs()
{
    InputComponent->BindAction(SpellAction1, ETriggerEvent::Triggered,
        this, &MyCharacter::BeginCastFireball);
    InputComponent->BindAction(SpellAction2, ETriggerEvent::Triggered,
        this, &MyCharacter::BeginCastMagicMissile);
}

void AMyCharacter::BeginCastFireball()
{
    PlayAnimMontage(FireballAnimation);

    FTimerHandle AttackTimer;
    GetWorldTimerManager().SetTimer(AttackTimer, this,
        &MyCharacter::CastFireball, FireballTimeout);
}

void AMyCharacter::CastFireball()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = this;

    GetWorld()->SpawnActor<AActor>(FireballClass, GetTransform(), SpawnParameters);
}

void AMyCharacter::BeginCastMagicMissile()
{
    PlayAnimMontage(MagicMissileAnimation);

    FTimerHandle AttackTimer;
    GetWorldTimerManager().SetTimer(AttackTimer, this,
        &MyCharacter::CastMagicMissile, MagicMissileTimeout);
}

void AMyCharacter::CastMagicMissile()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = this;

    GetWorld()->SpawnActor<AActor>(MagicMissileClass, GetTransform(),
        SpawnParameters);
}
```

Each spell is bound to a different action on `MyCharacter`. Triggingering the action begins casting the spell. It plays an animation, waits a set amount of time, and triggers the spell creation. This is easy to apply to an enemy too with some quick duplication and tweaking.

```cpp
// GoblinWizard.h
UCLASS()
class EXAMPLE_API AGoblinWizard : public ACharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UAnimMontage> FireballAnimation;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> FireballClass;

    UPROPERTY(EditDefaultsAnywhere)
    float FireballTimeout;
}
```

```cpp
// GoblinWizard.cpp
void AGoblinWizard::BeginCastFireball()
{
    PlayAnimMontage(FireballAnimation);

    FTimerHandle AttackTimer;
    GetWorldTimerManager().SetTimer(AttackTimer, this,
        &AGoblinWizard::CastFireball, FireballTimeout);
}

void AGoblinWizard::BeginCastFireball()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = this;

    GetWorld()->SpawnActor<AActor>(FireballClass, GetTransform(), SpawnParameters);
}
```

## Gathering Power

Another option is collecting all the unique attributes of the spell cast, like which spell is being cast, who is casting it, and perhaps which animation to use. Then wrap those properties in an spell casting class with a convenient trigger method, like `execute`.

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
    float AnimationTimeout;

    TObjPtr<AActor*> Instigator;

    void SpawnSpell();
}
```

```cpp
// SpellAction.cpp
USpellAction()
{
    AnimationTimeout = 0.2f;
}

void USpellAction::Execute(AActor* InstigatorActor)
{
    Instigator = InstigatorActor;

    PlayAnimMontage(CastAnimation);

    FTimerHandle CastTimer;
    GetWorldTimerManager().SetTimer(CastTimer, this,
        &USpellAction::CastSpell, AnimationTimeout);
}

void USpellAction::CastSpell()
{
    FActorSpawnParameters SpawnParameters;
    SpawnParamaters.Instigator = Instigator;

    GetWorld()->SpawnActor<AActor>(SpellClass, GetTransform(), SpawnParameters);
}
```

Now all the spell casting code has been refactored to use a single class and looks familiar. It's the Command Pattern, one of the most common software design patterns. All the different parameters are controlled per instance of the spell action class. The spell can be triggered by a call to `Execute` with a single parameter, the instigating actor. Using this pattern, it's now much easier to quickly bind keys or npc logic to trigger different spells. Here's what the character class might look like using this new spell action.

```cpp
// MyCharacter.h
UCLASS()
class EXAMPLE_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(EditDefaultsAnywhere)
    TObjPtr<USpellAction> Fireball;

    UPROPERTY(EditDefaultsAnywhere)
    TObjPtr<USpellAction> MagicMissile;

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

## Generalizing Abilities

In fact this can be taken a step further. Add a parent Action class that spell derives from with the execute method and a name. Then toss in an ActorComponent to hold a list of actions triggerable by name. This starts to look like a rudimentary game ability system that can be added to any actor.

```cpp
// Action.h
UCLASS()
class EXAMPLE_API UAction : public UObject
{
    GENERATED_BODY()

public:
    virtual void Execute(AActor* InstigatorActor);
}
```

```cpp
// Action.cpp
virtual void Execute(AActor* InstigatorActor)
{
}
```

```cpp
// SpellAction.h
UCLASS()
class EXAMPLE_API USpellAction : public UAction
{
    // ...

public:
    virtual void Execute(AActor* InstigatorActor);

```

```cpp
// SpellAction.cpp
virtual void Execute(AActor* InstigatorActor)
{
    Super::Execute(Instigator);

    // ...
}
```

## Further Reading

- Read about Unreal's Official [Game Ability System](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-for-unreal-engine)
