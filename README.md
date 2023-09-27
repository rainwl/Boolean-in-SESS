# Boolean in SESS

## Overview
Due to the molding scene, endoscope scene, and physical scene, three sets of models with similar shapes but different vertices, triangles and coordinate axes are used, and there is no base coordinate system on the hardware, each base coordinate is the simulated bone origin after calibration, and there are errors such as assembly jitter. And we have to do two Boolean operations of different algorithms at the same time, and calculate some extra things to communicate between the game engine and the physics engine. So here's the calculation.
For now, Unity will do most of the computing. But I think this is not a good solution, although the calculation is not difficult, but too cumbersome, if you do not have to document the step by step down, later generations to see the code may be difficult to understand why so many matrix operations.
## Algorithm
First,we should know that because of there's no base-axis in reality,the origin coordinate of the optics is the bone origin and 
will be changed after every calibration.

![No Axis](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/No%20Base%20Axis.png)

Thus,the bone origin is a standard coordinate.

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/standardTransform.png)

However,because of the *Mechanical Assembly Error* and *Optics Error*,in a specific frame,we can't guarantee the
bone is at origin(0,0,0),we should record this transformation **T1**`Matrix4x4`.

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/assemblyOpticsError.png)

So,when should we record this transformation?

*Record this per frame in Update until the angle variation of bone is exceed 5 degree*

*If during recording,the optics is covered by something,keep the last data*

**Ensure that the software is started in a prone position**

When do boolean,record the current transform and get a transform between current and reference,called **T2**`Matrix4x4`.

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/T2.png)

*No distinction is made between prone and lateral positions*

Then,do boolean.We can get the current transform of the trePan in Unity coordinate.This transform is called **T3**`Matrix4x4`.

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/Dobool.png)

In this case,we could do C# boolean.

When do C++ Boolean,the bone.obj from IO we call it **bone**,we do transform **T1** and **T2** to it.

```c++
bone.vertices[i] = T1.MultiplyPoint(bone.vertices[i]);
bone.vertices[i] = T2.MultiplyPoint(bone.vertices[i]);
```
and we do **T3** to trepan.obj from IO which we call **trepan**

```c++
trepan.vertices[i] = T3.MultiplyPoint(trepan.vertices[i]);
```

Finally,do C++ boolean.


In the same way,we can get the BoolCenter and Pivot in Unity Coordinate.

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/Pivot.png)

```c++
var currentMatrix = Pivot.localToWorldMatrix;
currentMatrix = Matrix4x4.Inverse(T1) * currentMatrix;
var t2Translation = T2.GetColumn(3);
currentMatrix = Matrix4x4.Translate(-t2Translation) * currentMatrix;

Pivot.position = currentMatrix.GetColumn(3);
Pivot.rotation = Quaternion.LookRotation(currentMatrix.GetColumn(2), currentMatrix.GetColumn(1));
```

This pivot is the pivot which needed in flex with getting rid of **T1** and **T2**'s translate.

And at this point,we pass the T2.rotation to Flex using `IO` or `shared memory`,at present,we pass this value using euler angles.

In Unity endoscope scene,we have been passed the `mesh` from boolean scene,and give a T2.rotation and a fixed translation (0,1,0) because of flex.

So far, we have dealt with C# and C++ booleans, and have done a good job of alignment and passing.

**But**

Since the three scenes of boolean, endoscope and physics respectively adopted 3 different sets of models during modeling, we also need to deal with the offset between the models.

**Although**

The motion trajectory are two Irregular arcs,we also could represent it with a transform of matrix4x4,so it could be decomposed
 with a rotation and a position.

We assume that the object moves along only one arc.

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/T2_2.png)

Next, get rid of the translation part of **T2**, that is, align the axis to the original axis

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/Rot.png)

and,the vertices and triangles are similar but not the same .

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/model.png)

but,the offset between two models is const,i.e.,the T2's translation is const,only the rotation is different.

we prepare the `A` and `B` 's transform,and we can get the `A'` and `B'` 's transform,so we can get the runtime transform **T4**,

```c++
var rotationMatrix = Matrix4x4.Rotate(T2.rotation);
var resultTransform = rotationMatrix.MultiplyPoint(origin);
T4 = Matrix4x4.Translate(resultTransform)*T2.localToWorldMatrix;
```

![](https://pic4rain.oss-cn-beijing.aliyuncs.com/img/offset.png)

## Implement

**Content Involved**

`AdsorbentCheck` `BlindCheck` `BoneCheck` `ResetBoneCX` `MySceneManager` `FurRenderer` `TrepanBoolean`

`Boolean.prefab` `Body_CC.prefab`


