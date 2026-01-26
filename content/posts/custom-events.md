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

## Intro

Things change frequently in games. Take Super Mario Bros. where time ticks down and scores and coins go up. In Unreal there are several ways to implement UI elements and ensure they provide accurate information. Some of these require less code, but are more computationally expensive. Others require some setup, but are far more flexible.

I will focus on a scenario where a player runs right, crosses checkpoints, and jumps over gaps. They have a score. Every time the player reaches a checkpoint they get 500 points. If they fall in a pit and restart the level, they lose 300 points. Let's assume I've created a basic UI widget to display the score on the screen and I'll walk through how to ensure that the score is always up to date.

## A Naive First Approach

UI widgets allow binding properties to functions. I can create a binding to the text field of my widget. In that function I can read the score from somewhere, set the UI widget text to the score, and call it a day. The good news that this is quick to write. The bad news is this runs every frame.

In a game that runs at 30 frames per second, this widget has ~33.3 ms to update the score before the next frame starts. At 60 frames per second, that time decreases to ~16.6 ms. If the game ran at a blistering 144 frames per second, there's only ~6.9 ms available. That may be plenty of time for a score widget to update, but everything in the game likely needs to update within that shrinking time window as well.

Eventually so many things are updating that the game can keep up and the player experiences lag. A more conservative approach would be to only update the score when the score changes. If the player isn't doing anything in the game, it's likely that the score is unchanged and the UI is being needlessly updated.

## Custom Events To The Rescue

To ensure that the UI is only updated when the score changes, I can use a macro to define a custom event that will be triggered whenever the score changes.

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnScoreChanged, uint32, NewScore);
```

This macro creates a type that can be used as a property on an object, allowing other objects to bind to it. This version only passes a single parameter but there are variants of the macro that allow up to 9 parameters. Take special note of the required comma between the type `int32` and identifier `NewScore`. It's easy to miss.

Using `BlueprintAssignable` allows objects to bind to the event in C++ and Blueprints.

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
    float Score;
}
```

```cpp
// MyPlayerState.cpp

void AMyPlayerState::UpdateScore(int32 Delta)
{
    Score += Delta;

    OnScoreChanged.Broadcast(Score, Delta);
}
```

That's it actually. Every time something calls `UpdateScore` anything bound to the new OnScoreChanged event should receive the new score and delta.

## Binding The UI

## Wrapping Up

Defining custom events is a very powerful technique that ensures changes are only processed when they occur, freeing up time for things that actually need to update on each frame.

## Further Reading
- [Multicast Delegates in Unreal](https://dev.epicgames.com/documentation/en-us/unreal-engine/multicast-delegates-in-unreal-engine)
