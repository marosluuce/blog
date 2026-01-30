+++
draft = false
date = 2026-01-30T11:35:00-05:00
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

Things change frequently in games. In Labyrinth on the NES, time ticks down while scores and gems go up. To implement similar UI elements, Unreal offers several approaches to display up to date information. One is binding functions, which can be quickly wired up but is potentially more expensive. The other is events, requiring a tad more setup, which feels familiar and flexible.

To explore these two methods, I want focus on a single aspect in a simple game, the player's score. The score goes up if they succeed at something and goes down if they fail. I've already created a UI widget to display the score on the screen and will walk through both of ways of displaying the score accurately.

## Tracking Scores

First, I need to keep track of the score and player state feels like a natural spot for that. Any caller can change the score by calling `UpdateScore` with the change amount. The annotation `BlueprintCallable` means code in Blueprints can see the function and trigger score updates too.

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

This score won't appear to change until the player's state is linked to the score widget, so that's the next step.

## A Naive First Approach

Unreal allows UI widgets to bind a property to a function and I can use that to bind a function to the text field of my score widget. I chose to do this in Blueprints, but it's not much different in C++.

![Binding a UI Property](binding-ui-property.png "Binding Score to Text")

When the score widget renders, this function will find the owning player and retrieve the player state. It then casts it to `MyPlayerState` and, if successful, converts the integer score to text. The good news is binding a property like isn't a lot of work. The bad news is this function runs every frame.

In a game that runs at 30 frames per second, everything has ~33.3 ms to update before the next frame starts. At 60 frames per second, the time between frames decreases to ~16.6 ms. If the game ran at a blistering 144 frames per second, there's only ~6.9 ms available. That may be plenty of time for a score widget to update, but everything in the game vies for a piece of the frame rate pie.

Eventually so many things are updating within a single frame that the game can't keep up and the player experiences lag.

## Events To The Rescue

A more conservative approach is to process updates only when the underlying data changes. If the player stands still it's unlikely that the score has changed and updating the UI is unnecessary. For my player state, this means publishing an event when the score changes with any updated values.

Unreal provides a handy macro for defining a custom event.

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnScoreChanged, uint32, NewScore);
```

This macro may look complicated but it's just creating a special struct with the name `FOnScoreChanged`. This can then be used to define a variable in a class and broadcast `NewScore` to all listeners. This example only uses a single parameter but there are variants of the macro that allow up to 9 parameters.

Here's an updated example of `MyPlayerState` tracking the score with a custom score updated event.

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

    UFUNCTION(BlueprintCallable)
    int32 GetScore();
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

int32 AMyPlayerState::GetScore()
{
    return Score;
}
```

Any time the score needs to change callers still use `UpdateScore`. If it's a non-zero change, the score updates then publishes the change event. To broadcast, the delegate `OnScoreChanged` expects two parameters, the new score and the amount by which it changed. The annotation `BlueprintAssignable` above `OnScoreChanged` enables listeners to bind to it in C++ and Blueprints.


## Binding The UI

To configure the listener, the score widget needs to subscribe to the score changed event. A natural place to do this is in the score widget's construct function, `NativeConstruct`.

This constructor will:
- Get (and cast) the owning player's player state to `AMyPlayerState`
- Bind to `AMyPlayerState` score changes using `AddDynamic`
- Set the default score to the current score

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

	APlayerController* Character = GetOwningPlayer();
	AMyPlayerState* PlayerState = Character->GetPlayerState<AMyPlayerState>();

	PlayerState->OnScoreChanged.AddDynamic(this, &UMyScoreWidget::OnScoreChanged);

	OnScoreChanged(PlayerState->GetScore(), 0);
}

void UMyScoreWidget::OnScoreChanged(int32 NewScore, int32 Delta)
{
	Score->SetText(FText::FromString(FString::Printf(TEXT("%d"), NewScore)));
}j
```

Whenever the score changed event is broadcast, `UMyScoreWidget::OnScoreChanged` is called. It will format and set the text value of the score widget.

Binding the score widget in Blueprints looks similar.

![Binding a UI Event](binding-ui-event.png "Binding Score Text to Score Changed Event")

Neither version of the score widget use `Delta` yet. I can imagine showing the changed amount temporarily near the score to give a player more feedback. Or maybe another widget listens to the same event and renders the difference directly above the player's head. There are so many possibilities.

## Wrapping Up

A single UI element probably won't cause lag, even at high frame rates, but this serves as a good introduction to defining and binding events. It's a powerful technique to ensures changes are only processed when they occur, freeing up processing time for everything else in the frame. There are many existing events for you to take advantage of, like `OnHit` for triggering something upon a collision. And when you can't find an event to fit your needs, make your own.

## Further Reading
- [Multicast Delegates in Unreal](https://dev.epicgames.com/documentation/en-us/unreal-engine/multicast-delegates-in-unreal-engine)
- [Events in Unreal](https://dev.epicgames.com/documentation/en-us/unreal-engine/events-in-unreal-engine)
