---
layout: post
title: Dragging Labels in 2014
category: CivilDev
tags:
    - c3d2014
    - labels
---

Once in a while, we need to deprecate a feature of the API because is confusing. 
Still, we need to be careful because many of you might be using the feature. The 
process is to deprecate the feature, so you have time to port the code and then, 
in a future release, remove the feature. Hopefully, your code is well abstracted 
so the impact of the changes is well centralized and easy to port.

In our latest release, we had to deprecate the “set” implementation of the 
‘DraggedOffset’ property for label objects. We run into many problems with this 
property during our automation because of the behavior of some of the label 
objects. We found ourselves having many problems verifying the behavior was 
correct, and that can only mean that you will run into many problems also, so we 
decided to make the property read-only and provide a better way to expose the 
functionality.

The source of the problem is the way certain labels behave when they are dragged. 
Some labels will change the anchor point (AnchorInfo.Location) when the label is 
dragged to certain location. How and when this happens is not intuitive nor can 
be determined, so it becomes very confusing. Let me illustrate the problem.

The following screen-shot shows a point label where we have added a leader so the 
problem is easier to see:

![point label with leader](/img/2013/point_label_leader.png)

As you can see, the leader is anchored to the “middle-center” of the point, and 
the label text is aligned to the “middle-right” of the leader. If you were going 
to read the ‘DraggedOffset’ property of that label, the result would be a 
Vector3d with the distance from the center of the point to the middle left of 
the text.

Let’s say you now drag the label. As you can see in the next screen-shot, the 
leader is not anchored to the side of the point and the location of the text 
still aligned to the “middle-right” of the leader.

![point label dragged](/img/2013/point_label_dragged.png)

The real problem here is that if you set the ‘DraggedOffset’, to drag the label, 
to a value of, let’s say, Vector3d(100, 100, 0), and immediately after you read 
the value of the property, the returned vector is not guaranteed to be the same 
because the anchor point has moved.

To fix this issue, we deprecated the “set” method of ‘DraggedOffset’ and 
provided a ‘LabelLocation’ property, which allows you to set and get an absolute 
location. Moving the label using this property guarantees its location.

Still, many of you might be using the ‘DraggedOffset’ property to move the label, 
and you will need to port your code. You have the option to calculate the new 
location according to the offset value you already have and change the 
‘LabelLocation’ property. This may involve repeating the computation in several 
places in your code, which is not ideal.

A better solution is to use ‘Extension Methods’. This feature of .NET, which has 
been around for a while, is one of my favorite feature. It allows me to define 
the interface I want and centralize code and still make client code look like is 
using native methods in an object. You can define an extension class to the 
Label class in Civil 3d that does the calculation and use an extension method to 
replicate the work that ‘DraggedOffset.set’ used to do.

**C#**
```csharp
public static class MyLabelExtensions
{
    public static void DragLabel(this Label label, Vector3d offset)
    {
        Point3d newLocation = label.AnchorInfo.Location + offset;
        label.LabelLocation = newLocation;
    }
}
```

**VB.NET**
```vb
Module MyLabelExtensions
    <Extension()>
    Public Sub DragLabel(ByRef label As Label, ByVal offset As Vector3d)
        Dim newLocation As Point3d = label.AnchorInfo.Location + offset
        label.LabelLocation = newLocation
    End Sub
End Module
```

Have fun.

