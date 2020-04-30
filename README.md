# _**Unofficial RuntimeMeshComponent User Guide**_

This document contains some quick notes on RMC v4, focused mainly on C++ for Unreal Engine v4.23+.



# RMC Overview


## Installation

**Step #1:** Clone or download the RMC project source into `YourProject/Plugins` (the RMC plugin source should be under `YourProject/Plugins/RuntimeMeshComponent/Source` after you've done this).

**Step #2:** Modify `YourProject/Source/YourProject/YourProject.Build.cs`, adding the following line:

```c#
PublicDependencyModuleNames.AddRange(new string[] { "RenderCore", "RHI", "RuntimeMeshComponent" });
```

**Step #3:** Restart Unreal Engine, check that it detects the plugin and compiles correctly & refresh your IDE project files from the editor.


## Using the RMC v4

To use an RMC v4 mesh in your game, you'll need:
1. A mesh 'provider' to provide mesh & collision data.
2. A runtime mesh component, initialised with your provider object.
3. An actor to hold and manage your component.


## Creating Meshes

With the RMC you have 4 main options to create meshes: 
1. Use the built-in box/sphere providers to generate simple pre-coded meshes.
2. Use the built-in static mesh provider to render a given static mesh.
3. Use the built-in static provider to build a custom mesh from code in a fairly straight-forward manner, similar to RMC v3 and Epic's ProceduralMeshComponent.
4. Implement your own provider to get the most power and flexibility out of the RMC (e.g.: easy section invalidation and regeneration, multi-threaded mesh creation, selection between various levels of collision support, highly-configurable LOD, etc).


## Main Library Components

### Providers <sup>([`URuntimeMeshProvider` / `FRuntimeMeshProviderProxy`](https://github.com/Koderz/RuntimeMeshComponent/blob/master/Source/RuntimeMeshComponent/Public/RuntimeMeshProvider.h))</sup>

A provider is an object that provides mesh, collision and material data to the RMC.  There are several built-in providers or you can build your own.  See Providers In-Depth and Implementing a Provider further below for more info.

#### Built-in Providers

- Box/Sphere ([`URuntimeMeshProviderBox`](https://github.com/Koderz/RuntimeMeshComponent/blob/master/Source/RuntimeMeshComponent/Public/Providers/RuntimeMeshProviderBox.h)/[`URuntimeMeshProviderSphere`](https://github.com/Koderz/RuntimeMeshComponent/blob/master/Source/RuntimeMeshComponent/Public/Providers/RuntimeMeshProviderSphere.h)) 

  Renders a simple sphere or cuboid.
- Static Mesh ([`URuntimeMeshProviderStaticMesh`](https://github.com/Koderz/RuntimeMeshComponent/blob/master/Source/RuntimeMeshComponent/Public/Providers/RuntimeMeshProviderStaticMesh.h)) 

   Renders a given `UStaticMesh`.
- Static ([`URuntimeMeshProviderStatic`](https://github.com/Koderz/RuntimeMeshComponent/blob/master/Source/RuntimeMeshComponent/Public/Providers/RuntimeMeshProviderStatic.h)) 

   Used to create custom static meshes from your own code. [Here is an example.](https://github.com/Koderz/RuntimeMeshComponent-Examples/blob/master/Source/RuntimeMeshExamples/SimpleExamples/BasicStaticProviderRMC.cpp)
- Collision (``) -- TODO
- Normals (``) -- TODO
- Memory Cache (``) -- TODO

### Runtime Mesh Component <sup>([`URuntimeMeshComponent`](https://github.com/Koderz/RuntimeMeshComponent/blob/master/Source/RuntimeMeshComponent/Public/RuntimeMeshComponent.h))</sup>

A runtime mesh component is an actor component which adds mesh and collision data to the actor via a [runtime mesh]() object.  It requires initialisation with a runtime mesh data provider via `Initialize(…)`<sup>1</sup>.

<sub>

```cpp
1. void URuntimeMeshComponent::Initialize(URuntimeMeshProvider* Provider)
```
</sub>

### Runtime Mesh Actor <sup> ([`ARuntimeMeshActor`](https://github.com/Koderz/RuntimeMeshComponent/blob/master/Source/RuntimeMeshComponent/Public/RuntimeMeshActor.h))</sup>

The runtime mesh actor utility class is a simple extension of the base `AActor` class that manages a single `URuntimeMeshComponent` by default, along with some methods to help manage mesh mobility settings.



# RMC In-Depth

In RMC v4 the RMC gets all mesh data from a provider. There are two provider types, proxy and non-proxy providers.


## Providers vs Proxy Providers

Providers and proxy providers are very similar.  Proxy providers require you to implement a few extra functions related to managing mesh data that then allow the RMC to use multi-threaded mesh creation. (Flicking through the source, it looks like the multi-threaded creation may not yet be implemented...)

The other key difference between providers and proxy providers is that proxy providers still require a regular provider as a wrapper (parent) around the proxy.  In this case the provider functions are all implemented in the proxy rather than in the parent and the parent provider has its `GetProxy()` method overridden to return your proxy.


## Using a Provider

The simplest way to use an existing provider is to create a `ARuntimeMeshActor` and override the `GenerateMeshes()` method to:
1. Create the provider object.  If you're using a proxy provider, you only create the parent provider here.
2. Set the provider's parameters.
3. Initialise the runtime mesh component with the provider.

For some simple examples of this in C++ [look here](https://github.com/Koderz/RuntimeMeshComponent-Examples/tree/master/Source/RuntimeMeshExamples/SimpleExamples).


## RMC Execution Flow

### Setup and Initialisation

1. Your runtime mesh actor's `OnConstruction()` or `BeginPlay()` method calls its own `GenerateMeshes()` method, which you will have overriden to:
   1. Create the runtime mesh provider object (if are using a proxy this is just the parent provider, not the proxy).
   2. Set the provider parameters.
   3. Call `Initialize(..)` on the runtime mesh component, passing in the provider object.
2. The runtime mesh component creates a `URuntimeMesh` and passes your provider object to the runtime mesh's `Initialize(..)` method.
3. *[only relevant if using a proxy provider]* The runtime mesh calls your provider's `SetupProxy()` method. This causes your provider to:
   1. Call `GetProxy()` on itself, which you will have overriden to return your proxy (see examples).
   2. Call `UpdateProxyParameters(..)` on your proxy, passing in a pointer to itself and `bIsInitialSetup` set to true.  You will have overridden this method to enable your proxy provider to set its own initial state and parameters from the public parameters of the parent provider (set in step 1.2.).
4. The runtime mesh then calls `Initialize()` on your provider/proxy.  You will have overridden this method to initialise your provider by configuring LODs, creating sections, setting up materials, etc.
5. Next, the runtime mesh calls `MarkSectionDirty(INDEX_NONE, INDEX_NONE)` on itself, forcing itself to ask your provider to generate the first set of mesh data by calling `GetSectionMeshForLOD(..)` on your provider/proxy for each valid section/LOD.
6. Finally, the runtime mesh calls `MarkCollisionDirty()` on itself, registering itself to generate collision data (via your provider) on a subsequent tick.


### Game Loop

TODO -- updating params, section invalidation and regeneration, regenerating collision data, etc


# **EVERYTHING AFTER THIS IS AN UNFINISHED MESS**


# Implementing Your Own Provider

## Implementing Regular Providers

First, set up `void Initialize(URuntimeProvider* Provider)`.

On initialisation

`GetSectionMeshForLOD` is called when the RMC is missing mesh data for that section or when the section has been marked as dirty using `MarkSectionDirty`.

Collision data is gathered after the mesh.

Do not use random or network data in your provider, only pass the data you need for the mesh otherwise the mesh may change unexpectedly.


### `Initialize(…)`

Create your ...

...

### `GetSectionMeshForLOD(…)`

`GetSectionMeshForLOD` should only return false when the given section or LOD is invalid/out of scope.  The function is passed an `FRuntimeMeshRenderableMeshData` struct by reference, it is expected that you will fill all mesh data fields in this struct (`Positions`, `Tangents`, `TexCoords`, `Colors`, `Triangles`) except for `AdjancencyTriangles` which is only for tesselation or if you use other Providers stacked on top, for example to provide normals and tangents.

### `GetBounds()`

Returns an `FBoxSphereBounds`.

Improperly placed and sized bounds will result in rendering issues (if too small) and performance problems (if too large).

### Collision Configuration

To enable collisions, first override `HasCollisionMesh`.

`GetCollisionSettings` will be called if `HasCollisionMesh` is true and `GetCollisionMesh` can also be implemented for complex collisions.  You are expected to merge different sections collisions by yourself (if you want to).

#### Collision Events

Collision events will return the face index that can be used to trace back to the collider.


### Marking Sections Dirty

To mark all sections dirty at once, call `MarkSectionDirty(INDEX_NONE, INDEX_NONE)` (or just set one of these to delete all sections for a single LOD or all LODs for a single section).


## Implementing Proxy Providers

NB: Whenever you implement a proxy provider you will still need a regular provider wrapper, called the parent provider, which manages your proxy provider and acts as the go-between between the runtime mesh component and your proxy provider.


TODO fix this section


by extending `FRuntimeMeshProviderProxy`
`URuntimeMeshProvider` 


When implementing a proxy provider there are three extra methods, `UpdateProxyParameters(…)`<sup>1</sup>, `MarkProxyParametersDirty()`<sup>2</sup> and `IsThreadSafe()`<sup>3</sup>, which must be overridden ... to enable thread-safe mesh data creation.

Calling `MarkProxyParametersDirty()` will cause `UpdateProxyParameters(…)` to be called when ready, which allows your code to update the mesh data when parameters change.


<sub>

```cpp
1. void UpdateProxyParameters(URuntimeMeshProvider* ParentProvider, bool bIsInitialSetup);
2. void MarkProxyParametersDirty();
3. bool IsThreadSafe();
```  
</sub>



# Advanced Topics

TODO 



# Common Problems and Known Issues

TODO



# Other Resources

TODO



# Glossary

- Section -- 
- LOD -- 
- Tris/Triangles -- 
- Normals -- 
- UV --
- Dithering --
- Update frequency --
- Bounds --
- etc... TODO

