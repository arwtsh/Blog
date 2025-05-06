---
title: Easier Singletons for Unity
description: An easy way to create non-component singletons in Unity using reflection
date: 2025-05-5 10:07:00
categories: [Unity, Plugins]
tags: [unity, plugin, tool, backend]
image:
  path: assets/img/posts/easy_singletons/Banner.png
  lqip: data:image/webp;base64,L7E.d~-ND4Io-;NHIUM|00j[~qjE
---

## Summary

In game development, manager classes are vital to the game's core. Making these managers singletons can help the organization of the project. However, the traditional way of creating singletons in Unity is flawed. I created singletons separated from Unity's MonoBehavior API which only connect to the GameObject system when necessary.

## Definition of Singletons

A singleton is a class with only one instance, which is globally accessible. In Unity game development, singletons are very useful. Depending on the situation, singletons might be problematic. However, this discussion is about how to make singletons better, not alternatives to singletons. [Refactoring Guru](https://refactoring.guru/design-patterns/singleton) explains the pros and cons to the singleton pattern itself and when to use other patterns.

## Traditional Unity Singletons

Most guides on the internet will say to set up your singletons like this:
```c#
public class SingletonExample : MonoBehaviour
{
  public static SingletonExample Instance { get; private set; }

  private void Awake()
  { 
    if (Instance == null) 
    {
      DontDestroyOnLoad(gameObject)
      Instance = this;
    }
    else if (Instance != this)
    { 
      Destroy(this); 
    } 
  }
}
```
If you're familiar with Unity you probably understand what is going on here. A SingletonExample is placed on a GameObject as a component, and when the GameObject loads into the scene its Awake function is automatically called. If this is the first SingletonExample encountered, it will mark itself to not get automatically destroyed when scenes change and will set the global accessor. Otherwise, the duplicate singleton deletes itself. 

This implementation of the singleton pattern *can* work, but it is susceptible to user error and doesn't cooperate with tests. Let me elaborate.

Singletons should have a known lifetime. In the above example, the singleton is created when it is first loaded in a scene, then it persists until the game stops. It is possible for a developer to create a test scene and forget to add the singleton to a GameObject, then the singleton would be null. Often, all singletons are placed on a single prefab. This makes it slightly easier to add the core systems of the game into a test scene and allows configs on the singletons to synchronize across scenes.

There's also the ability for developers to simply delete the singleton GameObject from the scene. The developers can also add duplicates of a singleton accidentally, such as by thinking the PlayerInput component should be on the player GameObject, not realizing the input component was on the prefab GameObject already. Fortunatly, duplicate singletons are automatically deleted; unfortunatly, the player was just deleted. These situations aren't likely and can be easily undone, but every developer has done something embarrassingly stupid that confused them for a while.

## Solution

So how can user error be reduced? The answer is for singletons to automatically instantiate themselves at a specific time with a bootstrapper. With a bit of magic from the Unity API, we can attach the attribute [`RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.AfterSceneLoad)`](https://docs.unity3d.com/ScriptReference/RuntimeInitializeOnLoadMethodAttribute.html) onto a static method to automatically instantiate the singleton prefab. 

While this is good enough, I have one final question. Why are singletons components? It doesn't make sense to have the singleton pattern work with the component pattern. Singletons should have core functionality, no need to access other components. Singletons shouldn't need to have a coordinate position in the scene. Non-MonoBehaviors can still patch into the game's update loop with Coroutines and Async/Await. Serialized fields can still be exposed to the inspector by connecting a SerializedObject to the singleton. [Hextant Studios Utilities](https://github.com/hextantstudios/com.hextantstudios.utilities) has a great package for creating runtime settings, among other super cool features. 

## Easy Singletons

I have created a package to make creating singletons as easy as possible. 

Using this package, all someone has to do to create a singleton is to inherit from the base class Singleton.

```c#
public abstract class Singleton<T> where T : Singleton<T>
{
  public static T Instance { get; internal set; }

  protected virtual void OnSingletonInit() {}
}
```

Then, whenever the game starts, whether it be in a build or editor play mode, the bootstrapper automatically instantiates every class inheriting from `Singleton<T>` using reflection. After all singletons have been instantiated, a GameObject named Singleton is instantiated into the scene and marked as DontDestroyOnLoad. This is so the singletons can patch into the MonoBehavior API if necessary. Then OnSingletonInit is called on every singleton. This is called before Awake, Start, or any other MonoBehavior method.

## How to Install

You can find the source code on [GitHub](https://github.com/arwtsh/EasySingletons), along with the documentation linked in the readme. The specific instructions for installation can be found in the [documentation](https://github.com/arwtsh/EasySingletons/blob/main/Documentation~/easysingletons.md#installation-instructions).
