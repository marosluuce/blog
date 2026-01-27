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

Things change quite frequently in games. In Super Mario Bros. scores and coins go up and time ticks down. There are several ways to implement similar UI elements in Unreal and ensure they provide accurate information. Some approaches require less code, but are more heavy-weight. Other ways require more setup and are far more flexible.

I want focus on a simple game where a player runs right, crosses checkpoints, and jumps over gaps. They have a score. Every time the player reaches a checkpoint they get 500 points. If they fall in a pit they lose 300 points and restart from the last checkpoint. Let's assume I've already created a UI widget to display the score on the screen and I'll walk through different ways to ensure that the score is always up to date.

## A Naive First Approach

UI widgets allow binding a property to a function. I can create a binding to the text field of my widget. In the bound function I can read the score, set the text to the score, and return. The good news that this is quick to write. The bad news is this runs every frame.

In a game that runs at 30 frames per second, this widget has ~33.3 ms to update the score before the next frame starts. At 60 frames per second, that time decreases to ~16.6 ms. If the game ran at a blistering 144 frames per second, there's only ~6.9 ms available.

There may be plenty of time for a score widget to update, but everything else in the game needs to update within that shrinking time window as well. Eventually so many things may be updating that the game can't finish in the space of one frame and the player experiences lag.

## Custom Events To The Rescue

A more conservative approach is only handling updates when the underlying data changes. If the player standings still, it's unlikely that the score has changed and updating the UI unnecessary. The way to do this for the score element is by creating an event and publishing new values whenever the score changes. This allows the score widget to subscribe to those changes and ensure the UI only updates when the data has changed.

Thankfully there's a handy macro for defining a custom event.

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

This example may be a little artificial but it's good practice.

Defining custom events is a very powerful technique that ensures changes are only processed when they occur, freeing up time for things that actually need to update on each frame.

## Further Reading
- [Multicast Delegates in Unreal](https://dev.epicgames.com/documentation/en-us/unreal-engine/multicast-delegates-in-unreal-engine)
