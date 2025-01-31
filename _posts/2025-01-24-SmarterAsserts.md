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
### 1. [Exception Handling](https://www.geeksforgeeks.org/exception-handling-c/){: target="_blank"}
  
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

### 2. [Print to log](https://dev.epicgames.com/documentation/en-us/unreal-engine/logging-in-unreal-engine){: target="_blank"}
  
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

### 3. [Asserts](https://dev.epicgames.com/documentation/en-us/unreal-engine/asserts-in-unreal-engine){: target="_blank"}
  
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

