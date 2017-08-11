---
layout: post
title: Subassembly Targets (Having Fun Yet?)
category: CivilDev
tags:
    - c3d2014
    - corridors
---

In my previous post, I demonstrated how to create a ‘Corridor’ from scratch. The 
sample builds a very basic Corridor, creates a Baseline and a Baseline Region. 
For more complex corridors, there is a little more work to do; especially, if 
you want to leverage the full dynamism of the ‘Corridor’ object. Any 
self-respecting Corridor will require you set the subassembly targets in it. 
Given the e-mails, comments, and other communications I received lately, it 
doesn't seem straight forward to work with the targets, and I have to admit that 
they are a little bit confusing. In this post, I will try to explain how to work 
with subassembly targets and how you should used them through the API.

### Targets are Exposed by Subassemblies

One of the steps of designing a subassembly is to specify the target types it 
supports. When the Subassembly object gets applied, as part of an Assembly 
object, to the Corridor, you are able to set the targets in it. As you saw in my 
previous post, the Assembly used is specified in the BaselineRegion when created, 
and it is applied across all the stations in the region. It doesn't make sense, 
thereof, to set the targets in the AppliedAssembly object directly. Instead, we 
set the targets in the BaselineRegion object and the parameters are used by all 
the AppliedAssembly and contained AppliedSubassembly objects.

The `BaselineRegion.GetTargets()` and `BaselineRegion.SetTargets()` methods are 
provided by the API to access and modify targets. These methods are also exposed 
by the Baseline and Corridor objects. It is important to understand that targets 
on baselines and corridors aggregate the targets in their regions. This means 
that having a corridor with multiple baselines that in turn contain multiple 
regions, a call to `Corridor.GetTargets()` will return targets for all regions 
in all baselines for the corridor. The methods in Corridor and Baseline are 
provided for convenience; although, I am still looking for a good situation when 
to use them.

### Work with Targets at the BaselineRegion Level

Again, the `GetTargets()` and `SetTargets()` methods in Corridor and Baseline 
are provided for convenience in case of a special situation that I still haven't 
found (if you know of any, please let me know). You will always want to control 
the targets at the BaselineRegion level, and as a matter of fact, it is the only 
way the code will behave as you expected. I modified the sample from my previous 
post to set some targets.

**C#**

```csharp
public void CreateCorridor(CivilDocument document)
{
    _document = document;
    createCorridorObject();
    createCorridorBaseline();
    createBaselineRegion();
    assignTargets();
    _corridor.Rebuild();
}

private void assignTargets()
{
    // These will return empty collections because the
    // Corridor object has not been built yet.
    //
    // SubassemblyTargetInfoCollection targets = 
    //      _corridor.GetTargets();
    // SubassemblytargetInfoCollection targets = 
    //      _baseline.GetTargets();

    // Getting the targets from the BaselineRegion 
    // works because it access the information from
    // the specified Assembly object when creating the
    // BaselineRegion.
    //
    SubassemblyTargetInfoCollection targets = 
        _region.GetTargets();
    foreach (SubassemblyTargetInfo target in targets)
    {
        assignTarget(target);
        
    }
    // The collection is empty if retrieved from the
    // Corridor or Baseline object; therefore these
    // calls will do nothing.
    // 
    // _corridor.SetTargets(targets);
    // _baseline.SetTargets(targets);

    // Regions allow you to specify the targets.
    //
    _region.SetTargets(targets);
}
```

**VB.NET**

```vb
Public Sub CreateCorridor(document As CivilDocument)
    _document = document
    createCorridorObject()
    createCorridorBaseline()
    createBaselineRegion()
    assignTargets()
    _corridor.Rebuild()
End Sub

Private Sub assignTargets()
    ' These will return empty collections because the
    ' Corridor object has not been built yet.
    '
    ' SubassemblyTargetInfoCollection targets = 
    '      _corridor.GetTargets();
    ' SubassemblytargetInfoCollection targets = 
    '      _baseline.GetTargets();

    ' Getting the targets from the BaselineRegion 
    ' works because it access the information from
    ' the specified Assembly object when creating the
    ' BaselineRegion.
    '
    Dim targets As SubassemblyTargetInfoCollection =
        _region.GetTargets()
    For Each target As SubassemblyTargetInfo In targets

        assignTarget(target)
    Next
    ' The collection is empty if retrieved from the
    ' Corridor or Baseline object; therefore these
    ' calls will do nothing.
    ' 
    ' _corridor.SetTargets(targets);
    ' _baseline.SetTargets(targets);

    ' Regions allow you to specify the targets.
    '
    _region.SetTargets(targets)
End Sub
```

The implementation of `CreateCorridor()` hasn't changed a lot. It just adds a 
new call to `assignTargets()`. Notice that this call is done before we "rebuild" 
the Corridor object. The `assignTargets()` method first calls `GetTargets()` on 
the BaselineRegion object. This returns a `SubassemblyTargetInfoCollection` 
object containing information about all the supported targets, which is stored 
as a `SubassemblyTargetInfo` object. For each one of them, we assign the 
corresponding target (`assignTarget()`).

If we tried to do the same operation at the BaselineRegion or Corridor 
object level, it wouldn't work. Targets for the BaselineRegion and the Corridor 
objects are not defined until the Corridor object is built. Since we are setting 
targets before we actually build the corridor, calling `GetTargets()` on 
BaselineRegion or Corridor will return an empty collection 
(`SubassemblyTargetInfoCollection`).

### TargetIds is a Disconnected Collection

I have been planning a post to explain the concept of a connected vs. 
disconnected collection for the longest time. What stops me is that it will 
reveal a major inconsistency through out the API. One of these days, I'll come 
clean and talk about this topic, and I will point out how we violate this rule 
in the API. In the meantime, you need to understand that the `TargetIds` 
property of `SubassemblyTargetInfo` is of type `ObjectIdCollection` and 
therefore, it is a disconnected collection. Let’s take a look at some code, and 
then I'll explain how this affects you.

**C#**

```csharp
private void assignSurfaceTarget(
    SubassemblyTargetInfo target)
{
    // The 'Add()' method of 'ObjectIdCollection'
    // knows nothing about subassembly targets; therefore
    // this call doesn't work.
    //
    // target.TargetIds.Add(surfaceId);

    target.TargetIds = _targetSurfaces;

    // Alternatively, you can get the collection,
    // manipulate it, and then set it again.
    //
    // ObjectIdCollection ids = target.TargetIds;
    // ... do whatever manipulations (add/remove targets)
    // target.TargetIds = ids;
    //
    // This will work, but trust me, my way (starting
    // clean) it is easier most of the time.
}

// The following 2 methods are implemented under the
// assumption that the subassembly name contains a
// substring "Right" or "Left" depending on its side.
// This is a big assumption that works on this
// particular example, but you will have to get more
// creative.
//
private void assignElevationTarget(
    SubassemblyTargetInfo target)
{
    if (isRightSide(target.SubassemblyName))
    {
        target.TargetIds = _targetRightElevation;
    }
    else
    {
        target.TargetIds = _targetLeftElevation;
    }
}

private void assignOffsetTarget(
    SubassemblyTargetInfo target)
{
    if (isRightSide(target.SubassemblyName))
    {
        target.TargetIds = _targetRightOffset;
    }
    else
    {
        target.TargetIds = _targetLeftOffset;
    }
}
```

**VB.NET**

```vb
Private Sub assignTarget(target As SubassemblyTargetInfo)
    Select Case target.TargetType
        Case SubassemblyLogicalNameType.Surface
            assignSurfaceTarget(target)
            Exit Select

        Case SubassemblyLogicalNameType.Elevation
            assignElevationTarget(target)
            Exit Select

        Case SubassemblyLogicalNameType.Offset
            assignOffsetTarget(target)
            Exit Select
    End Select
End Sub

Private Sub assignSurfaceTarget(
        target As SubassemblyTargetInfo)
    ' The 'Add()' method of 'ObjectIdCollection'
    ' knows nothing about subassembly targets; therefore
    ' this call doesn't work.
    '
    ' target.TargetIds.Add(surfaceId);

    target.TargetIds = _targetSurfaces

    ' Alternatively, you can get the collection,
    ' manipulate it, and then set it again.
    '
    ' ObjectIdCollection ids = target.TargetIds;
    ' ... do whatever manipulations (add/remove targets)
    ' target.TargetIds = ids;
    '
    ' This will work, but trust me, my way (starting
    ' clean) it is easier most of the time.
End Sub

' The following 2 methods are implemented under the
' assumption that the subassembly name contains a
' substring "Right" or "Left" depending on its side.
' This is a big assumption that works on this
' particular example, but you will have to get more
' creative.
'
Private Sub assignElevationTarget(
        target As SubassemblyTargetInfo)
    If isRightSide(target.SubassemblyName) Then
        target.TargetIds = _targetRightElevation
    Else
        target.TargetIds = _targetLeftElevation
    End If
End Sub

Private Sub assignOffsetTarget(
        target As SubassemblyTargetInfo)
    If isRightSide(target.SubassemblyName) Then
        target.TargetIds = _targetRightOffset
    Else
        target.TargetIds = _targetLeftOffset
    End If
End Sub
```

The `SubassemblyTargetInfo.TargetIds` property sets/gets an `ObjectIdCollection` 
containing the object ids of the targeted objects. `ObjectIdCollection` knows 
nothing about subassembly targets and there is no hook exposed in the class to 
do any work when an object is added or removed from the collection. The 
`ObjectIdCollection` is what we call in the API team a disconnected collection. 
Because of this, you cannot write this type of code 
`target.TargetIds.Add(newId)`. This will add the object id to a temporary 
collection that will get discarded right after the call. 

You can also get the collection, add/remove items from it, and then set it again 
in the property. I find this way a lot more difficult to use because you need to 
know exactly what the collection already contains to remove items that you don’t 
need, and then add the new items to it. Generally, it is easier to start with a 
new `ObjectIdCollection` that contains exactly what you want and set it, and 
that's exactly what I do in the example.

In a future post, I will talk about connected and disconnected collections and 
what you need to understand when working with the API. This will reveal several 
inconsistencies throughout the API, but it is important for you to understand 
those inconsistencies. For now, I hope I explained well enough how to work with 
subassembly targets and the best practices that you should use. If not, drop me 
a comment or an e-mail with any further questions you might have.

As always, you can download the full source code for the sample from the 
[Civilized Development](https://bitbucket.org/IsaacRodriguez/civilizeddevelopment) 
repository at BitBucket, which also contains an updated “CorridorFromScratch.dwg” 
to work with the new version of the sample.
