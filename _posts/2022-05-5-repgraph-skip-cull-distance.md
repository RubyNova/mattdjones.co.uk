---
title: "Replication Graph: Disabling net cull distance checks for specific actors."
layout: posts
toc_label: "Table of Contents"
toc_icon: "heart"
excerpt: "Allowing you to have an actor be both always relevant and spatialized, on a per connection basis."
header:
    teaser: assets/posts/repgraph-skip-cull-distance/teaser.png
---

**Danger:** This tutorial requires an engine modification, proceed with caution. The version used is UE4.27, but this should work in previous and following versions.
{: .notice--danger}
**Warning:** This tutorial also assumes you already have an understanding of the replication graph.
{: .notice--warning}


## Intro 

While working on the replication graph for Post Scriptum, I quickly came to realize that we'd need to have certain actors be always relevant to one team, and spatialized for another.
This of course is based on the team of the connection. Players would have other players and vehicles in their team be always relevant, while the enemy team would be spatialized as normal.<br/>

This constraint was due to how the map marker system works: markers are tied to actors, meaning an actor that's not relevant to you won't appear on the map. This is an issue for team-owned actors, since you're likely to want to know where your team is to coordinate your attack or defense. 

Rewriting this system would be too much of a hassle, so I opted for the following approach.

## Preparing the replication nodes.

In order for this to work, you'll need the following nodes:

- `UReplicationGraphNode_GridSpatialization2D`: The classic grid spatialization node, which will replicate the actors based on position and cull distance.
- A custom node that replicates its actors to specific connections. Let's call it `UReplicationGraphNode_AlwaysRelevant_ForTeam`.

For Post Scriptum, we actually have three team relevancy nodes (Neutral, TeamOne, TeamTwo). For simplicity's sake, we'll ignore "Neutral" here, as they are deemed relevant to everyone.
{: .notice--info}

Let's focus on a simple implementation of `GatherActorListsForConnection()`:

```cpp
void UReplicationGraphNode_AlwaysRelevant_ForTeam::GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params)
{
    // Assuming the node is holding the team of the actors it holds.
    // Iterate through the net viewers until we find one that's of the same team.
    for (const FNetViewer& Viewer : Params.Viewers)
    {
        if (const AMyPlayerController* PC = Cast<AMyPlayerController>(Viewer.InViewer))
        {
            if (PC->GetTeam() == NodeTeam)
            {
                // Add the replicated actors and return. Any other viewers sharing this connection will get them anyway.
                Params.OutGatheredReplicationLists.AddReplicationActorList(ReplicationActorList);
                return;
            }
        }
    }
}
```

And done, this simple node will now replicate actors to a specific team.    

## Routing the actors

In order for this to work, you'll need to route the team relevant actors to both the team node and the grid spatialization node.

```cpp
void UMyReplicationGraph::RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo, FGlobalActorReplicationInfo& GlobalInfo)
{
    // Substitute with your current routing and team logic, such as replication policies (i.e. Fortnite)
    if (IsTeamRelevantActor(ActorInfo))
    {
        const ETeamEnum Team = IMyTeamInterface::Execute_GetTeam(ActorInfo.Actor);
        if (TeamNodes.Contains(Team))
        {
            TeamNodes[Team]->NotifyAddNetworkActor(ActorInfo);
        }

        // You'll also want to determine if this is a Dynamic/Static/Dormancy based actor.
        GridSpatializationNode->AddActor_Dynamic(ActorInfo, GlobalInfo);
        return;
    }

    // ...
}

void UMyReplicationGraph::RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo)
{
    if (IsTeamRelevantActor(ActorInfo))
    {
        const ETeamEnum Team = IMyTeamInterface::Execute_GetTeam(ActorInfo.Actor);
        if (TeamNodes.Contains(Team))
        {
            TeamNodes[Team]->NotifyRemoveNetworkActor(ActorInfo);
        }

        GridSpatializationNode->RemoveActor_Dynamic(ActorInfo);
        return;
    }

    // ...
}
```

**Warning:** If your actors can change team, remember to handle that occurrence and move them to the correct team node, via a delegate.
{: .notice--warning}

## The pitfall.

Grid spatialized actors require a net cull distance to work. You cannot set it to zero like normal AlwaysRelevant actors.<br/>

The issue is that the replication graph does not know an actor is always relevant, it only gathers the replication lists from nodes and performs the check if the cull distance is greater than zero.

As such, team relevant actors using the nodes above will still be removed from clients based on the net cull distance. Let's fix that.

## Modifying the engine.

There are two things we need to modify in the engine:

- The `FConnectionGatherActorListParameters`, in which we'll add a `TSet<>` of actors that will skip the net cull distance check for the replication frame.
- The two replication paths called by `UReplicationGraph::ServerReplicateActors()`, named `ReplicateActorListsForConnections_Default()` and `ReplicateActorListsForConnections_FastShared()`, in which we'll check against the `TSet<>` to skip any culling for actors marked as so.

### Parameters

**Warning:** Do not copy and paste the struct below, only amend based on the differences marked below.
{: .notice--warning}

```cpp
// Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraphTypes.h
// Around line 1270
struct FConnectionGatherActorListParameters
{
    // <-- Modify the two constructors to include the new TSet<> -->
    UE_DEPRECATED(4.26, "Please use the constructor that takes a viewer array.")
    FConnectionGatherActorListParameters(FNetViewer& InViewer, UNetReplicationGraphConnection& InConnectionManager, TSet<FName>& InClientVisibleLevelNamesRef, uint32 InReplicationFrameNum, FGatheredReplicationActorLists& InOutGatheredReplicationLists, TSet<FActorRepListType>& InOutSkipNetCullDistanceActors)
        : ConnectionManager(InConnectionManager), ReplicationFrameNum(InReplicationFrameNum), OutGatheredReplicationLists(InOutGatheredReplicationLists), OutSkipNetCullDistanceActors(InOutSkipNetCullDistanceActors), ClientVisibleLevelNamesRef(InClientVisibleLevelNamesRef)
    {
        Viewers.Emplace(InViewer);
    }

    FConnectionGatherActorListParameters(FNetViewerArray& InViewers, UNetReplicationGraphConnection& InConnectionManager, TSet<FName>& InClientVisibleLevelNamesRef, uint32 InReplicationFrameNum, FGatheredReplicationActorLists& InOutGatheredReplicationLists, TSet<FActorRepListType>& InOutSkipNetCullDistanceActors)
        : Viewers(InViewers), ConnectionManager(InConnectionManager), ReplicationFrameNum(InReplicationFrameNum), OutGatheredReplicationLists(InOutGatheredReplicationLists), OutSkipNetCullDistanceActors(InOutSkipNetCullDistanceActors), ClientVisibleLevelNamesRef(InClientVisibleLevelNamesRef)
    {
    }
    
    // ...

    /** Out: The data nodes are going to add to */
    FGatheredReplicationActorLists& OutGatheredReplicationLists;

    // <-- Add the TSet<> below the OutGatheredReplicationLists. -->
    /** Out: Actors that will skip net cull distance checks this replication frame. */
    TSet<FActorRepListType>& OutSkipNetCullDistanceActors;

    // ...
};
```

Nodes will now be able to mark actors that need to skip net cull distance checks in `GatherActorListsForConnection()`

### ServerReplicateActors()

Now, we need to pass the `TSet<>` to the replication paths so they'll be able to check against it.

```cpp
// Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp
// Around line 987, UReplicationGraph::ServerReplicateActors()

    // --------------------------------------------------------------------------------------------------------------
    // GATHER list of ReplicationLists for this connection
    // --------------------------------------------------------------------------------------------------------------
    
    // Init the TSet<> after the replication actor lists.
    FGatheredReplicationActorLists GatheredReplicationListsForConnection;
    TSet<FActorRepListType> SkipNetCullDistanceActorsForConnection;

    // ...

    // Around line 1022
    // Pass in the TSet<> to the two replication paths.

    // --------------------------------------------------------------------------------------------------------------
    // PROCESS gathered replication lists
    // --------------------------------------------------------------------------------------------------------------
    {
        QUICK_SCOPE_CYCLE_COUNTER(NET_ReplicateActors_ProcessGatheredLists);

        ReplicateActorListsForConnections_Default(ConnectionManager, GatheredReplicationListsForConnection, SkipNetCullDistanceActorsForConnection, ConnectionViewers);
        ReplicateActorListsForConnections_FastShared(ConnectionManager, GatheredReplicationListsForConnection, SkipNetCullDistanceActorsForConnection, ConnectionViewers);
    }
```

We'll need to update the signature of the replication paths to pass in the set.

```cpp
    /** Default Replication Path */
    void ReplicateActorListsForConnections_Default(UNetReplicationGraphConnection* ConnectionManager, FGatheredReplicationActorLists& GatheredReplicationListsForConnection, TSet<FActorRepListType>& SkipNetCullDistanceActorsForConnection, FNetViewerArray& Viewers);

    /** "FastShared" Replication Path */
    void ReplicateActorListsForConnections_FastShared(UNetReplicationGraphConnection* ConnectionManager, FGatheredReplicationActorLists& GatheredReplicationListsForConnection, TSet<FActorRepListType>& SkipNetCullDistanceActorsForConnection, FNetViewerArray& Viewers);
```

**Warning:** Remember to also do it in the .cpp file.
{: .notice--warning}

### Replication Paths

Finally, we'll skip the cull distance check in both paths, if the actor is part of the set.

```cpp
// Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp
// Around line 1249, UReplicationGraph::ReplicateActorListsForConnections_Default()

    // ...

    // -------------------
    // Distance Scaling
    // -------------------
    if (GlobalData.Settings.DistancePriorityScale > 0.f)
    {
        float SmallestDistanceSq = TNumericLimits<float>::Max();
        int32 ViewersThatSkipActor = 0;

        // Avoid checking in the set if bDoDistanceCull is disabled.
        bool bSkipDistanceCulling = !bDoDistanceCull || SkipNetCullDistanceActorsForConnection.Contains(Actor);
        
        for (const FNetViewer& CurViewer : Viewers)
        {
            const float DistSq = (GlobalData.WorldLocation - CurViewer.ViewLocation).SizeSquared();
            SmallestDistanceSq = FMath::Min<float>(DistSq, SmallestDistanceSq);

            // <-- This is where the magic happens. -->
            // Figure out if we should be skipping this actor
            if (!bSkipDistanceCulling && ConnectionData.GetCullDistanceSquared() > 0.f && DistSq > ConnectionData.GetCullDistanceSquared())
            {
                ++ViewersThatSkipActor;
                continue;
            }
        }

        // ...
```

```cpp
// Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp
// Around line 1564, UReplicationGraph::ReplicateActorListsForConnections_FastShared()

    // ...

    // Determine if this actor has any view relevancy to any connection this client has
    bool bNoViewRelevency = true;
    // Check if we should skip distance culling for this actor.
    bool bSkipCullDistanceCheck = SkipNetCullDistanceActorsForConnection.Contains(Actor);
    for (const FNetViewer& CurView : Viewers)
    {
        const FVector& ConnectionViewLocation = CurView.ViewLocation;
        const FVector& ConnectionViewDir = CurView.ViewDir;

        // Simple dot product rejection: only fast rep actors in front of this connection
        const FVector DirToActor = GlobalActorInfo.WorldLocation - ConnectionViewLocation;
        if (!(FVector::DotProduct(DirToActor, ConnectionViewDir) < 0.f))
        {
            bNoViewRelevency = false;
            break;
        }

        // <-- And this is where it happens for the _FastPath() -->
        // Simple distance cull
        const float DistSq = DirToActor.SizeSquared();
        if (!(DistSq > (ConnectionData.GetCullDistanceSquared() * FastSharedDistanceRequirementPct)) || bSkipCullDistanceCheck)
        {
            bNoViewRelevency = false;
            break;
        }
    }

    // ...
```

Congratulations, if you've done it correctly, the engine should now support per-connection distance culling bypass. 

## Adding it to our nodes.

The last step is for our custom node to keep track of actors that should ignore distance culling.

First off, add a set to your node.

```cpp
	// Actors that should skip net cull distance check.
	TSet<FActorRepListType> SkipDistanceCheckActors;
```

Then, override `NotifyAddNetworkActor()` and `NotifyRemoveNetworkActor()`, and add/remove actors to/from the set.

```cpp
void UReplicationGraphNode_AlwaysRelevant_ForTeam::NotifyAddNetworkActor(const FNewReplicatedActorInfo& ActorInfo)
{
    Super::NotifyAddNetworkActor(ActorInfo);

    SkipDistanceCheckActors.Add(ActorInfo.Actor);
}

void UReplicationGraphNode_AlwaysRelevant_ForTeam::NotifyRemoveNetworkActor(const FNewReplicatedActorInfo& ActorInfo, bool bWarnIfNotFound)
{
    Super::NotifyRemoveNetworkActor(ActorInfo, bWarnIfNotFound);
    
    SkipDistanceCheckActors.Remove(ActorInfo.Actor);
}
```

Finally, update your `GatherActorListsForConnection()` to pass in the set to the parameters.

```cpp
void UReplicationGraphNode_AlwaysRelevant_ForTeam::GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params)
{
    // Assuming the node is holding the team of the actors it holds.
    // Iterate through the net viewers until we find one that's of the same team.
    for (const FNetViewer& Viewer : Params.Viewers)
    {
        if (const AMyPlayerController* PC = Cast<AMyPlayerController>(Viewer.InViewer))
        {
            if (PC->GetTeam() == NodeTeam)
            {
                // Add the replicated actors and return. Any other viewers sharing this connection will get them anyway.
                // Also prevent them from being distance culled.
                Params.OutGatheredReplicationLists.AddReplicationActorList(ReplicationActorList);
                Params.OutSkipNetCullDistanceActors.Append(SkipDistanceCheckActors);

                return;
            }
        }
    }
}
```

And you're done, these actors will no longer be culled for connections they're always relevant to, as long as they're in this node.

This could probably be added to the official engine repository through a pull request, and I'm currently working on getting one up.
{: .notice--info}