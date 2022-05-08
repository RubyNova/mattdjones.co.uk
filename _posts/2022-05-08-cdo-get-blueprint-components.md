---
title: "How to get the Blueprint components of an actor's Class Default Object."
layout: posts
toc_label: "Table of Contents"
toc_icon: "heart"
excerpt: "Useful to retrieve valuable data without having to spawn an instance."
tags: c++ blueprints
header:
    teaser: assets/posts/cdo-get-blueprint-components/teaser.png
---

## Intro

By default, Class Default Objects only contain their C++ components, and will not return Blueprint spawned components when calling `AActor::GetComponents()`. 

However, there is another way to retrieve them.

## Retrieving blueprint spawned actors

We can easily get a list of blueprint components by iterating through the construction script's nodes generated nodes:

```cpp
bool UMyBlueprintFunctionLibrary::GetClassDefaultObjectBlueprintComponents(UObject* ClassDefaultObject, TArray<UActorComponent*>& OutComponents)
{
    // Check that the CDO is actually a class default object.
    checkf(ClassDefaultObject, TEXT("Cannot get the blueprint components of a null CDO."));
    checkf(ClassDefaultObject->HasAnyFlags(RF_ClassDefaultObject), TEXT("Cannot get the blueprint components of a non-CDO object."));

    // Here, we can check if the class is a blueprint. 
    if (UBlueprintGeneratedClass* BlueprintClass = ClassDefaultObject->GetTypedOuter<UBlueprintGeneratedClass>())
    {
        // Retrieve the hierarchy of blueprint classes for this CDO.
        TArray<const UBlueprintGeneratedClass*> BlueprintClasses;
        UBlueprintGeneratedClass::GetGeneratedClassesHierarchy(BlueprintClass, BlueprintClasses);
        for (const UBlueprintGeneratedClass* Class : BlueprintClasses)
        {
            if (Class->SimpleConstructionScript)
            {
                // And now we get the component from the nodes.
                TArray<USCS_Node*> CDONodes = Class->SimpleConstructionScript->GetAllNodes();
                for (USCS_Node* Node : CDONodes)
                {
                    UActorComponent* CDOComponent = Node->GetActualComponentTemplate(Cast<UBlueprintGeneratedClass>(BlueprintClass));
                    OutComponents.Add(CDOComponent);
                }
            }
        }

        return true;
    }

    return false;
}
```

## Usage

With the above function, we can easily retrieve a CDO's C++ and Blueprint components.

```cpp
AActor* ClassDefault = MyBlueprintClass->GetDefaultObject<AActor>();

TArray<UActorComponent*> Components;
ClassDefault->GetComponents(Components);
UMyBlueprintFunctionLibrary::GetClassDefaultObjectBlueprintComponents(ClassDefault, Components);

// Components will now contain both C++ and Blueprint spawned actors.
```

I've learned this neat trick while working for a level editor, where the class default object held "emplacements", which were scene components used as positions where rooms would connect to each other. These components were blueprint spawned, and as such I needed to be able to get them from a CDO.
