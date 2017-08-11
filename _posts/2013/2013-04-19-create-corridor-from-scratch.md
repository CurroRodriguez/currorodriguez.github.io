---
layout: post
title: Create a Corridor from Scratch
category: CivilDev
tags:
    - c3d2014
    - corridors
---

The Civil 3D API provides the necessary functionality to create a Corridor in 
the same way we will do through the application UI. The corridor object is on of 
the most complex objects provided by Civil 3D, and it is necessary to understand 
its structure to be able to create one programmatically. In this post, we will 
go through the process of creating a corridor based on an existing drawing. The 
full source for the example is available at the 
[Civilized Development](https://bitbucket.org/IsaacRodriguez/civilizeddevelopment) 
repository at [BitBucket](http://bitbucket.org/). The repository contains a test 
drawing (CorridorFromScratch.dwg) that can be used to test the provided code.

As I mentioned before, the Corridor object is one of the most complex objects in 
Civil 3D. Creating one programmatically requires some work, and we want to 
maintain our code as clean as possible. I decided to use a separate class 
(`CorridorCreator`) from the command to do the work. The class provides an 
interface where we can set the parameters for the creation of the Corridor and a 
`CreateCorridor()` method that creates the Corridor object. The code in our 
command class remains fairly simple to follow:

**C#**

```csharp
private void executeCreateCorridorFromScratch()
{
  using (Transaction tr = startTransaction())
  {
    CorridorCreator creator = new CorridorCreator()
    {
      CorridorName = promptForString("\nEnter corridor name: "),
      BaselineName = promptForString("\nEnter baseline name: "),
      RegionName = promptForString("\nEnter region name: "),
      AlignmentId = _desiredAlignmentId,
      AssemblyId = _desiredAssemblyId
    };
    creator.CreateCorridor(_civildoc);

    tr.Commit();
  }
}
```

**VB.NET**

```vb
Private Sub executeCreateCorridorFromScratch()
    Using tr As Transaction = startTransaction()
        Dim creator As New CorridorCreator() With { 
            .CorridorName = promptForString(vbLf & "Enter corridor name: "), 
            .BaselineName = promptForString(vbLf & "Enter baseline name: "),
            .RegionName = promptForString(vbLf & "Enter region name: "),
            .AlignmentId = _desiredAlignmentId,
            .AssemblyId = _desiredAssemblyId
        }
        creator.CreateCorridor(_civildoc)

        tr.Commit()
    End Using
End Sub
```

`CorridorCreator` exposes properties to set the parameters of the Corridor 
object, which we set from user input. Once we have the parameters initialized, 
we call `CreateCorridor()` to create the Corridor object based on the input. 
Notice that we inject the CivilDocument object in our call to `CreateCorridor()`. 
This allow us to create the corridor in any open drawing. But there is a catch; 
all the creation parameters must belong to the same drawing or the process will 
fail.

The implementation of the `CreateCorridor()` methods shows us the necessary 
steps to build the Corridor object. We will then dive into the implementation of 
these steps.

**C#**

```csharp
public void CreateCorridor(CivilDocument document)
{
    _document = document;
    createCorridorObject();
    createCorridorBaseline();
    createBaselineRegion();
    _corridor.Rebuild();
}
```

**VB.NET**

```vb
Public Sub CreateCorridor(document As CivilDocument)
    _document = document
    createCorridorObject()
    createCorridorBaseline()
    createBaselineRegion()
    _corridor.Rebuild()
End Sub
```

The first step is to create a new Corridor in the CivilDocument. The CivilDocument 
class provides a CorridorCollection property, encapsulated by the private 
property `_corridors` in the example, which allows us to create a new Corridor 
object by name.

**C#**

```csharp
private void createCorridorObject()
{
    ObjectId id = _corridors.Add(CorridorName);
    _corridor = id.GetObject(OpenMode.ForWrite) as Corridor;
}

private CorridorCollection _corridors
{
    get { return _document.CorridorCollection; }
}
```

**VB.NET**

```vb
Private Sub createCorridorObject()
    Dim id As ObjectId = _corridors.Add(CorridorName)
    _corridor = TryCast(id.GetObject(OpenMode.ForWrite), Corridor)
End Sub

Private ReadOnly Property _corridors() As CorridorCollection
    Get
        Return _document.CorridorCollection
    End Get
End Property
```

Once we create the new Corridor object and we open it for write, we need to add 
a Baseline to it. The Corridor object contains a Baselines property that 
provides an `Add()` method to add a new Baseline. This method requires a 
baseline name, an alignment id, and a profile id.

**C#**

```csharp
private void createCorridorBaseline()
{
    _baseline = _corridor.Baselines.Add(BaselineName, AlignmentId, ProfileId);
}
``` 

**VB.NET**

```vb
Private Sub createBaselineRegion()
    _baseline.BaselineRegions.Add(RegionName, AssemblyId)
End Sub
```

Creating the regions completes the components of the Corridor. If we committed 
the Transaction at this point, the Corridor will be created; however, you will 
see in prospector that the object needs to be rebuilt. Fortunately, the API 
provides a `Rebuild()` method in the Corridor object that allows us to do that, 
and we can execute it before the Transaction is committed.

As you can see, the new version of the API provides all the facilities to build 
a Corridor object from scratch. There are more complex and involved operations 
that we can perform when creating our Corridors programmatically, and we will 
look at them in future posts. For now, feel free to download the full source from 
the Civilized Development repository and play with it to familiarize your-self 
with the new API.
