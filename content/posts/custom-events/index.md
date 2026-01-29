+++
draft = true
date = 2026-01-26T08:51:54-05:00
title = "Custom Events in Unreal"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

## Ch-Ch-Changes

Things change frequently in games. In Labyrinth on the NES time ticks down and scores and gems climb up. To implement similar UI elements, Unreal offers several approaches to display accurate information. Oneis quick to wire up and potentially more expensive. The other requires a tad more setup, but is far more flexible.

I want focus on a single aspect in a simple game, the player's score. The score goes up if they succeed at something and goes down if they fail. I've already created a UI widget to display the score on the screen and will walk through two ways of ensuring the score is up to date.

## Tracking Scores

I'll start by keeping track of the score in a player state class, which feels like a natural place for a score. Anything that wants to update the score can call `UpdateScore` with the change amount. The `BlueprintCallable` annotation just means code in Blueprints can see the function and trigger score updates too.

```cpp
// MyPlayerState.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerState.h"
#include "SPlayerState.generated.h"

UCLASS()
class EXAMPLE_API AMyPlayerState : public APlayerState
{
public:
    UFUNCTION(BlueprintCallable)
    void UpdateScore(int32 Delta);

protected:
    int32 Score = 0;
}
```

```cpp
// MyPlayerState.cpp
void AMyPlayerState::UpdateScore(int32 Delta)
{
    Score += Delta;
}
```

The score won't appear to change in the UI widget until I link it to the player's state, so that's the next step.

## A Naive First Approach

Unreal allows UI widgets to bind a property to a function and I can create such a function binding for the text field of my score widget. In said function I can read the score, convert the integer score to text, and return that text. This make sure the score text always matches the current value. I arbitrarily chose to do this in Blueprints, but it's not much different in C++.

![Binding a UI Property](binding-ui-property.png "Binding Score to Text")

When the score widget renders, it will find the owning player and retrieve the player state. It casts it to `MyPlayerState` and if successful converts the integer score to text. The good news is binding a property like this is quick to wire up. The bad news is this function runs every frame.

In a game that runs at 30 frames per second, everything has ~33.3 ms to update before the next frame starts. At 60 frames per second, the time between frames decreases to ~16.6 ms. If the game ran at a blistering 144 frames per second, there's only ~6.9 ms available. That may be plenty of time for a score widget to update, but as the frame rate increases everything vies for a piece of a shrinking pie.

Eventually so many things may be updating within a single frame that the game can't keep up and the player experiences lag.

## Custom Events To The Rescue

A more conservative approach is to process updates only when the underlying data changes. If the player stands still it's unlikely that the score has changed and updating the UI is unnecessary. For my score widget, this means publishing an event when the score changes with any updated values. The score widget can subscribe to those changes and ensure the UI only updates if it receives a score changed event.

Thankfully Unreal provides a handy macro for defining a custom event.

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnScoreChanged, uint32, NewScore);
```

This macro creates a multicast delegate struct with the name `FOnScoreChanged` and the parameter `NewScore`. Take special note of the comma between the type `int32` and the identifier `NewScore`. It's easy to miss. This may look complicated but it's just defining an event. This delegate can be used as a property on an object and broadcast the new value to listeners. This example only uses a single parameter but there are variants of the macro that allow up to 9.

Here's a updated example of `MyPlayerState` tracking the score with a custom event.

```cpp
// MyPlayerState.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerState.h"
#include "SPlayerState.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnScoreChanged, int32, NewScore, int32, Delta);

UCLASS()
class EXAMPLE_API AMyPlayerState : public APlayerState
{
public:
    UPROPERTY(BlueprintAssignable)
    FOnScoreChanged OnScoreChanged;

    UFUNCTION(BlueprintCallable)
    void UpdateScore(int32 Delta);

protected:
    int32 Score = 0;
}
```

```cpp
// MyPlayerState.cpp
void AMyPlayerState::UpdateScore(int32 Delta)
{
    if (Delta != 0)
    {
        Score += Delta;

        OnScoreChanged.Broadcast(Score, Delta);
    }
}
```

This multicast delegate `OnScoreChanged` expects two parameters, the new score and the amount by which it changed. The `BlueprintAssignable` annotation above `OnScoreChanged` enables listeners to bind to it in C++ and Blueprints.

Any time the score needs to change callers still use `UpdateScore`. The score updates then publishes the new score and change amount to all listeners. It avoid processing on every frame and only publishes an event when the score changed.

## Binding The UI

To configure the listener, I need to subscribe to score changes in the score widget. I can do this in the score widget's construct function, `NativeConstruct`.

This constructor will:
- Set the default score to `0`
- Get (and cast) the owning player's player state
- Bind to player state score changes using `AddDynamic`

`NativeConstruct` is called each time the score widget is added to the viewport and will ensure the binding exists whenever this widget is visible.

```cpp
// MyScoreWidget.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "MyScoreWidget.generated.h"

class UTextBlock;

UCLASS()
class ACTIONROGUELIKE_API UMyScoreWidget : public UUserWidget
{
	GENERATED_BODY()

	virtual void NativeConstruct() override;

protected:

	UFUNCTION()
	void OnScoreChanged(int32 NewScore, int32 Delta);

	UPROPERTY()
	TObjectPtr<UTextBlock> Score;
};
```

```cpp
// MyScoreWidget.cpp
#include "MyScoreWidget.h"

#include "MyPlayerState.h"
#include "Components/TextBlock.h"

void UMyScoreWidget::NativeConstruct()
{
	Super::NativeConstruct();

	Score->SetText(FText::FromString(TEXT("0")));

	APlayerController* Character = GetOwningPlayer();
	AMyPlayerState* PlayerState = Character->GetPlayerState<AMyPlayerState>();

	PlayerState->OnScoreChanged.AddDynamic(this, &UMyScoreWidget::OnScoreChanged);
}

void UMyScoreWidget::OnScoreChanged(int32 NewScore, int32 Delta)
{
	Score->SetText(FText::FromString(FString::Printf(TEXT("%d"), NewScore)));
}j
```

Binding the score widget in Blueprints looks similar.

![Binding a UI Event](binding-ui-event.png "Binding Score Text to Score Changed Event")

The score widget doesn't currently use `Delta` but I could imagine showing the changed amount temporarily near the score to give a player more feedback. Or maybe another widget listens to the same event and renders the difference directly above the player's head. There are so many possibilities.

## Wrapping Up

A single UI element probably won't cause lag, even at hight frame rates, but I think it's a good introduction to defining and binding events. This is a powerful technique to ensures changes are only processed when they occur, freeing up processing time for everything else in the frame. There are many existing events for you to take advantage of, like `OnHit` for triggering something upon a collision. And when you can't find an event to fit your needs, make your own.

## Further Reading
- [Multicast Delegates in Unreal](https://dev.epicgames.com/documentation/en-us/unreal-engine/multicast-delegates-in-unreal-engine)
- [Events in Unreal](https://dev.epicgames.com/documentation/en-us/unreal-engine/events-in-unreal-engine)
