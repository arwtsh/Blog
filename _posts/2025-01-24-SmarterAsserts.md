---
title: Smarter Asserts
date: 2025-01-24 14:54:00 -0700
categories: [Unreal, Plugins]
tags: [unreal, plugin, tool, editor, logging, error handling]     # TAG names should always be lowercase
image:
  path: assets/img/banners/SmarterAssertsBanner.png
  lqip: data:image/webp;base64,L05OQnwH9Fo~?uofM{M{00Xn%MVr
---

## Summary

Error handling is very important in computer programming. It is how most programmers avoid bugs and glitches. There are many different types of errors, both minor and major, and so there are different types of error handling. I noticed Unreal Engine had a significant blind spot in error handling, and have created a fix for it.

## Definition of Error

If you have read to this point, you probably don't need the dictionary explanation of an error. Instead, I will specify what types of errors this article is about. Unreal is written in C++, and although the source code is able to be modified and recompiled, this article does not consider error handling in the language or Unreal itself. (Thread-safe, memory leaks, hardware restrictions)

This article only considers errors created by the programmers working on a game made in Unreal. Both custom C++ code and Blueprint scripts can have errors. Null pointer errors are the most common, but other runtime errors can also apply. There are some logic errors which can have Error handling, but only those that can be forseen. A good example of this is sequence-breaking the game. If a player goes into an area they shouldn't be in, it wouldn't be that difficult for a programmer to add a check to move the player to an appropriate position. A bad example would be the player getting stuck in geometry, since making a check to see if the player is stuck would require editing Unreal's source code.

## Existing Error Handling

Unreal has quite a few options for error handling.
### 1. [Exception Handling](https://www.geeksforgeeks.org/exception-handling-c/)
  
  If you're familiar with C++, you might know what this is. While Unreal is built in C++, it does not support exceptions. It may be possible in very specific use cases, but it is better to not upset Unreal (it's an angry master). Instead, return a bool or enum that represents the error and pass the output by reference in the parameters.
  
  ```cpp
    bool bIsInitialized;
    
    // Returns true if there is no error
    // @param result The objected activated in the pool.
    UFUNCTION(BlueprintCallable)
    bool UObjectPool::Request(UPoolableObject*& result);
  ```
  {: file='ObjectPool.h'}

  ```cpp
    bool UObjectPool::Request(UPoolableObject*& result)
    {
      if(!bIsInitialized)
        return false;

      //Do something and set result
      return true
    }
  ```
  {: file='ObjectPool.cpp'}

  Functions created in Blueprint are also able to have multiple outputs.

  > This is good for core or backend mechanics. In this example, a weapon can execute custom logic when it fails to fire a projectile from the pool. This is more explicit than the object pool returning a nullptr.
  {: .prompt-tip }

  > This can make APIs cluttered. Also, it can be disruptive to C++ programmers who have to use the out parameter as the real returned value.
  {: .prompt-warning }

### 2. [Print to Log](https://dev.epicgames.com/documentation/en-us/unreal-engine/logging-in-unreal-engine)
  
  Displays an error message of some sort. In Unreal, there are two main destinations for output, the log and the on-screen debug messages. Unfortunantly, not every developer looks in the output log, and the on-screen message can be missed if it times out or gets pushed offscreen by newer messages.
  
  ```cpp
    bool bIsInitialized;

    // Returns the objected activated in the pool.
    UFUNCTION(BlueprintCallable)
    UPoolableObject* UObjectPool::Request();
  ```
  {: file='ObjectPool.h'}

  ```cpp
    UPoolableObject* UObjectPool::Request()
    {
      if(!bIsInitialized)
      {
        UE_LOG(LogTemp, Error, TEXT("Object pools need to be initialized before an object can be requested from it."));

        if (GEngine)
        {
          GEngine->AddOnScreenDebugMessage(-1, 5.f, FColor::Red, TEXT("Object pools need to be initialized before an object can be requested from it."));
        }

        return nullptr;
      }
    }
  ```
  {: file='ObjectPool.cpp'}

  The Blueprint function `Print String` or `Print Text` can output to the screen and log.

  > This is good for minor or non-fatal errors. Idealy, a programmer would notice the error, find why the error occured, and then fix the bug.
  {: .prompt-tip }

  > Programmers might not notice the error, or purposefully ignore it thinking it doesn't have any impact on what they are currently testing. Unlike Unity, errors logged will NOT pause the game.
  {: .prompt-warning }

### 3. [Asserts](https://dev.epicgames.com/documentation/en-us/unreal-engine/asserts-in-unreal-engine)
  
  Completely crash the engine and bring up the crash reporter. Unreal has several macros that help the engine correctly crash. This is C++ only, and is Unreal's suggested method of handling errors in C++. If you are debugging from Visual Studio, Rider, or other IDE, it will create a breakpoint instead of completely crashing the game.

  ```cpp
    bool bIsInitialized;
    
    // Returns true if there is no error
    // @param result The objected activated in the pool.
    UFUNCTION(BlueprintCallable)
    UPoolableObject* UObjectPool::Request();
  ```
  {: file='ObjectPool.h'}

  ```cpp
    UPoolableObject* UObjectPool::Request()
    {
      checkf(bIsInitialized, TEXT("Object pools need to be initialized before an object can be requested from it."))

      //Do something and set result
      return true
    }
  ```
  {: file='ObjectPool.cpp'}

  Unreal does not have any Blueprint nodes that assert.

  > This form of error handling is only good when everyone testing the game has a debugger attached (such as running Unreal from Visual Studio, Rider, or another IDE). If a Blueprint-only programmer or designer opened the .uproject directly, an assert will completely crash the editor.
  {: .prompt-warning }

## Problem

When I was creating an object pooling system, I set it up so the `Initialize` or `InitializeAsync` functions would need to be called on the object pool before it can be used. At the time, I was the only C++ programmer on the team. The designers or Blueprint programmers would be the ones who would use my system to create a weapon system, enemy drop system, and any other system that would benefit from object pooling. 

I had to find the best way to handle the error of what would happen if they tried to use the object pool without initializing it. Naturally, they should know to Initialize it first because of the documentation, but everyone makes mistakes in game dev. Even me, and I'm the one that programmed the object pooling system! 

If I used the `Exception Handling` method, then it would be up to the other team members to handle what would happen if an error occured. This is very easy to ignore, though. (Which they most likely would do if they didn't look at the documentation to call Initialize first.) 

If I used the `Print to Log` method, I would have to print to the screen. The other team members don't look in the log often, and if they would, it would most likely be flooded with other Unreal messages, making my error message less visible. Printing to the screen is more visible, but still not as obvious as it could be. Also, the code for printing to the screen is big and clunky. 

If I used `Asserts`, then the other team members would be very annoyed. They don't normally run Unreal with a debugger, so they would have to restart the entire editor after it crashes. While this would successfully notify them of the error they made, it is a *little* to extreme.

