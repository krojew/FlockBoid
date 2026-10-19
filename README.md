# Flock Boid

Flock Boid simulates flocks of birds, fish, insects and similar swarms in Unreal Engine. Static mesh flocks run on the GPU inside a Niagara system. Skeletal mesh flocks, and static mesh flocks on hardware without GPU support, run on a multithreaded CPU solver that applies the same rules. You place a spawner and give it one or more Flock Config data assets. It keeps flocks alive inside its area and replaces them over time. It can also remove them while no player is near.

## Requirements

- Niagara. The plugin enables it as a dependency.
- Win64, Linux or Mac.
- GPU simulation needs a shader platform at the SM5 feature level or higher, with Niagara GPU particles allowed. Where that is missing, flocks run on the CPU. There is nothing to set up for the fallback.

## Installation

Enable Flock Boid in Edit > Plugins (Gameplay category) and restart the editor. The shipped assets live under `/FlockBoid`; turn on Show Plugin Content in the Content Browser settings to see them.

## Quick start

1. Create a Flock Config: Content Browser > Add > Miscellaneous > Data Asset, then pick Flock Config. Or duplicate one of the samples in `/FlockBoid/Samples`.
2. In Appearance, pick a static mesh and set Mesh Rotation so the mesh's nose points along +X.
3. Place a Flock Spawn Point or a Flock Spawn Volume from the Place Actors panel, or add a Flock Spawner component to any actor.
4. On the spawner, add an element to Flocks and assign the config.
5. Play.

Flocks spawn on Begin Play, so they appear in Play In Editor and Simulate, not in the editor viewport. Selecting a spawner draws its area in blue and, when Spawn Region is Around Center, the spawn radius in amber.

## Spawners

Flock Spawn Point and Flock Spawn Volume are actors built around a Flock Spawner component, which can also be added to any actor. All three have the same settings and differ only in their defaults:

- Flock Spawn Point: a sphere area with a 2000 cm radius. New flocks start within a Spawn Radius of 300 cm of the centre and spread out from there.
- Flock Spawn Volume: a box area with a half extent of 2000 x 2000 x 1000 cm. New flocks start anywhere in it.
- Flock Spawner component: a sphere area with a 2000 cm radius, and flocks start anywhere inside.

### Area and spawn region

Area is the shape the boids stay in: a sphere (Radius) or a box (Extent, a half size), centred on the component. Spawn Region decides where new flocks appear, either anywhere in the area (Whole Area) or within Spawn Radius of the centre (Around Center). A new flock starts packed into a sphere of the config's Flock Radius (300 cm by default), and its centre is kept at least that far from the boundary.

The simulation runs in the spawner's local space, which has a few consequences:

- Moving the spawner moves its flocks. A spawner attached to a moving actor carries them along.
- Yaw turns the area. Pitch and roll tilt it, and with it the direction the flock treats as up for climbing, diving and banking.
- Scale only resizes the area. A box's Extent is scaled per axis, while a sphere's Radius and the Spawn Radius are scaled by the largest axis. The flocks themselves run unscaled, so boid meshes are never stretched and the config's speeds and distances stay as authored. Scale is read when the flocks spawn, and the editor visualizer draws the scaled area.

### Entries

Each element of Flocks is one entry:

- Config: the Flock Config to spawn.
- Flock Count, Boids Per Flock: tick the box next to either to override the config's range on this spawner only.
- Seed: 0 picks a new seed every time the flocks spawn. Any other value repeats the same cluster positions, boid counts, sizes, tints and lifetimes. How the boids move afterwards still depends on frame timing.
- Enabled: skips the entry without removing it.

Every enabled entry becomes one flock component that simulates all flocks of that entry together. Flocks of the same entry steer clear of each other (with Avoid Other Flocks on) but never align or group across flocks. Flocks of different entries ignore each other completely, even on the same spawner. The same config can be used by several entries.

The spawner reads its entries, its area and the configs when the flocks spawn. Changing them later takes effect on the next spawn, except for the config's Channels setting, which is read every frame. `SpawnFlocks` replaces any flocks that already exist, so calling it after changing the entries at runtime is how to apply them.

## Flock lifecycle

An entry has one slot per possible flock, as many as the maximum of its Flock Count. Each slot cycles through fading in, alive, fading out and waiting empty.

- Flock Count (1): how many flocks are alive at once. A target in this range is rolled when the flocks spawn and rolled again each time a flock leaves. Empty slots refill up to the target; no flock is removed early when the target drops. A minimum of 0 lets an entry go empty at times.
- Boids Per Flock (40 to 60): rolled for every new flock.
- Lifetime (0 s): how long a flock stays before it fades out, counted from when it has finished fading in. 0 keeps it forever, which also means the flock count is rolled once and never changes.
- Respawn Delay (5 to 15 s): after any flock leaves, how long the entry waits before it spawns the next one, at a new position.
- Fade Duration (1 s): boids scale up from nothing when a flock appears and back down when it leaves. 0 makes them pop.

The first flocks start fading in as soon as the spawner spawns. The respawn delay only applies to replacements. For a population that turns over slowly, try Flock Count 2 to 4, Lifetime 60 to 120 s and Respawn Delay 10 to 30 s.

## Despawning by distance

Despawn Distance (0, off) removes a spawner's flocks while every player is far away:

- Every Distance Check Interval (0.5 s) the spawner measures the distance from its own location to the view point of each local player.
- Once every view is farther than Despawn Distance, all of its flocks are destroyed.
- They return when a view comes within Despawn Distance minus Distance Hysteresis (1000 cm), so a player standing on the boundary does not make them flicker. Keep it below Despawn Distance: at or above it the flocks never return, and the spawner logs a warning at Begin Play.
- The distance is measured from the spawner, not from the edge of its area. For a sphere with a 2000 cm radius whose boids should stay visible from 5000 cm away, use at least 7000 cm.
- Flocks that return start over from a fresh placement.
- With no local player controller there is nothing to measure from, so the check leaves the flocks as they are and `SpawnFlocks` spawns regardless.

These three settings are read-only in Blueprint. From C++, Despawn Distance and Distance Hysteresis can be changed at any time, including after Begin Play; Distance Check Interval is applied at Begin Play.

### World Partition

A spawner in a World Partition level is spatially loaded like any other actor. When its cell unloads, the spawner ends play and its flocks go with it. When the cell loads again, Begin Play spawns them again, as long as Spawn On Begin Play is on. If it is off, call `SpawnFlocks` yourself.

Loading ranges are set per grid and are usually coarse. Despawn Distance is set per spawner, which gives finer control, and it also works in levels without World Partition. You can use both, but World Partition knows nothing about the size of the flock area, so a spawner with a large area can unload while the player is still near its far side. For those, turn off Is Spatially Loaded and rely on Despawn Distance.

## Behaviour settings

Distances are in centimetres, speeds in cm/s. Defaults are in brackets.

### Appearance

- Mesh Type (Static Mesh), and the Static Mesh or Skeletal Mesh property that goes with it. The skeletal settings are covered under CPU flocks.
- Override Materials: replaces the mesh's materials slot by slot, and empty entries keep the mesh's own. GPU flocks use the first four slots.
- Mesh Rotation (0): turns the mesh so its nose points along +X.
- Scale (1): uniform scale, picked per boid from this range.
- Tint A, Tint B (white): each boid gets a random colour between the two. See Materials for how to read it.
- Cast Shadows (on).
- Cull Distance (0, never): the whole flock stops drawing past this camera distance.
- Niagara System Override (advanced): see Custom Niagara systems.

The advanced section also holds the rendering flags of an engine primitive component. They behave as they do on the component and keep the engine's defaults, except Receives Decals, which is off. Every component that draws the flock gets them, CPU or GPU:

- Shadows, used while Cast Shadows is on: Cast Contact Shadow (on), Cast Far Shadow, Cast Inset Shadow, Self Shadow Only, Cast Shadow as Two Sided, Cast Volumetric Translucent Shadow (all off). Skeletal boids drawn as components are a component each, so with Cast Inset Shadow or Self Shadow Only every boid renders its own shadow map.
- Lighting: Lighting Channels (channel 0), Affect Dynamic Indirect Lighting (on), Affect Distance Field Lighting (on), Emissive Light Source (off).
- Visibility: Render in Main Pass (on), Render in Depth Pass (on, used while the main pass is off), Visible in Reflection Captures, Visible in Real Time Sky Captures, Visible in Ray Tracing, Visible in Reflections (all on), Holdout, Visible in Scene Capture Only, Hidden in Scene Capture, Receives Decals (all off).
- Custom depth: Render Custom Depth (off), Custom Depth Stencil Value (0) and Custom Depth Stencil Write Mask (Default).
- Translucency Sort Priority (0) and Translucency Sort Distance Offset (0), which only affect translucent materials.

### Spawn

- Flock Count, Boids Per Flock: see Flock lifecycle.
- Flock Radius (300): size of the cluster a new flock starts in.
- Initial Heading (Random): a random direction, the spawner's forward axis (Forward), or away from the cluster centre (Outward).
- Initial Speed (300 to 500), clamped to Min Speed and Max Speed.

### Movement

- Min Speed (200), Max Speed (600). Min Speed stays above 0 so every boid always has a heading.
- Max Acceleration (800) caps the combined steering of all rules. Low values give wide turns, high values sharp ones.
- Vertical Freedom (1) and Max Pitch (60 degrees) limit climbing and diving, and the stricter of the two applies. Vertical Freedom 0 keeps boids on a horizontal plane.
- Banking Strength (1), Max Bank Angle (45 degrees), Bank Response (4): boids lean into turns by an angle worked out from their sideways acceleration, and Bank Response sets how fast the lean catches up. Banking Strength 0 keeps them level.

### Flocking

- Perception Radius (400): neighbours within this distance count for alignment and cohesion. It is also the cell size of the neighbour grid (see Performance).
- Field Of View (270 degrees): neighbours behind this cone are ignored. 360 sees all around.
- Separation Radius (120), Separation Weight (1.5): boids push apart when closer than the radius.
- Alignment Weight (1): steer toward the neighbours' average velocity.
- Cohesion Weight (1): steer toward the neighbours' average position.
- Max Neighbours (12, at most 64): a boid stops scanning once it has accepted this many neighbours, counting boids of other flocks that it avoids.
- Avoid Other Flocks (on): boids also keep their separation from boids of the entry's other flocks.

### Wander

Weight (0.3) and Frequency (0.15 Hz) add a slow drift made of three sine waves, offset per boid, so a calm flock does not settle into a straight line. Weight 0 turns it off.

### Containment

Weight (2) and Margin (400): boids start turning back once they are within Margin of the area's edge, and the push grows quickly toward the edge and past it. It only steers, so fast boids with a low Max Acceleration can overshoot. Raise Weight or Margin if they visibly leave the area.

### Obstacles

Weight (4), Look Ahead Time (0.75 s), Margin (100): boids steer around obstacles they would reach within Look Ahead Time at their current velocity and try to keep Margin of clearance. A boid that still ends up inside an obstacle is pushed back out to its surface.

### Influences

- Response (1) scales every attractor, repeller and orbit acting on the flock. 0 makes the flock ignore them.
- Panic Speed Multiplier (1.5): the speed of boids fleeing a repeller at full panic, as a multiple of Max Speed. Panicking boids aim for the raised speed.

### Animation

Play Rate At Min Speed (1) and Play Rate At Max Speed (1): animation speed at the two ends of the speed range, blended in between as a boid speeds up. The fast rate may be lower than the slow one, for birds that flap less once they are up to speed. Together they drive the Phase value passed to materials and the play rate of component-rendered skeletal boids.

### Interaction and Simulation

- Channels (all eight): which obstacles and influences the flock reacts to. See Obstacles and influences.
- Simulation Target (Auto): see GPU and CPU.

### Typical adjustments

The three sample configs are the quickest reference. Beyond those:

- Birds: a high Max Speed with a Max Acceleration that is low relative to it, for wide turns. Full banking, a moderate Max Pitch and a large Perception Radius.
- Fish: Vertical Freedom well below 1 so the school stays in a layer, little or no banking, Alignment Weight above Cohesion Weight for tight schooling. A box area from a Flock Spawn Volume fits a pond or a tank.
- Insects: a small Scale, a high Max Acceleration and a high Wander Weight and Frequency for erratic paths, low Alignment Weight, Banking Strength 0, Max Pitch close to 89.

When the mesh gets smaller, shrink Perception Radius, Separation Radius, Flock Radius and both margins with it.

## Obstacles and influences

Flocks never trace against level geometry. Anything they should avoid needs a Flock Obstacle component or has to sit outside the area.

Flock Obstacle and Flock Influence are components for any actor. They are picked up automatically once registered and read every frame. They can move, and they can be switched off with Enabled or have their settings changed at runtime from Blueprint. They do not need to be inside a flock's area. Selecting an actor that has one draws its shape.

### Flock Obstacle

- Shape: Sphere with Radius (200), or Box with Extent (200 x 200 x 200, a half size).
- Follows the component's transform. Scale applies too: the sphere radius by the largest axis, the box per axis.

### Flock Influence

- Mode: Attract, Repel or Orbit.
- Radius (2000): boids farther away ignore it. 0 reaches every boid.
- Strength (1).
- Falloff (1): exponent of the fade toward the edge of the radius. 1 is linear, higher keeps the effect near the centre.
- Orbit Radius (500, Orbit only): boids circle the component's up axis at this distance and are drawn toward the plane through the component. Turn the component upside down to reverse the direction.
- Panic (0, Repel only): how much fleeing boids speed up, up to the config's Panic Speed Multiplier at 1.
- Radius and Orbit Radius scale with the largest axis of the component's scale.

### Channels

Configs, obstacles and influences each have eight channels, all on by default. A flock reacts to an obstacle or influence when they share at least one channel. To scare birds without scaring fish, put the repeller on Channel 2 only and clear Channel 2 from the fish configs.

## GPU and CPU

Each entry picks its path when its flocks spawn:

1. Skeletal mesh flocks always run on the CPU.
2. Simulation Target CPU runs on the CPU.
3. Everything else runs on the GPU if it is available (an SM5-capable shader platform, Niagara GPU particles allowed and `FlockBoid.ForceCPU` off) and on the CPU otherwise. Auto and GPU behave the same here.
4. A GPU flock also drops to the CPU, with a warning in the `LogFlockBoid` log, when its Niagara system fails to load or its Boids emitter cannot hold that many particles on the platform.

Both paths share the rules and the random numbers, and apart from the four-material-slot limit on the GPU a config works the same on either. Once a boid reaches Max Neighbours the two can pick different neighbours, so dense flocks match in character rather than boid for boid.

### GPU flocks

Each GPU flock component runs its own Niagara system, `NS_Flock` unless the config overrides it, with one mesh particle per boid. The simulation runs as compute passes on the GPU. Per frame, the game thread only uploads which flocks are alive and the obstacles and influences on the flock's channels.

Niagara scalability is turned off for these systems so effect culling can never reset a flock.

The Niagara mesh renderer does not draw a Nanite mesh that has no fallback mesh. The config validator reports this as an error. Enable the mesh's fallback or simulate that flock on the CPU.

### CPU flocks

The CPU solver spreads each step over the task graph. A step launched on one tick is collected on the next, so CPU flocks are drawn one frame behind their simulation, and the game thread only waits if a step takes longer than a frame.

There are three ways to draw them:

- Static mesh: one Instanced Static Mesh component with an instance per boid.
- Skeletal mesh, Skeletal Render Mode Instanced (the default): one Instanced Skinned Mesh component for the whole flock, animated by a transform provider asset. This is the mode for large skeletal flocks. The loop plays on Animation Variations tracks (4 by default, at most 16), each started at an evenly spaced point in the sequence, and every boid is assigned one of them. All tracks play halfway between Play Rate At Min Speed and Play Rate At Max Speed, so the animation speed does not follow each boid's speed.
- Skeletal mesh, Skeletal Render Mode Components: one Skeletal Mesh component per boid, looping Animation from a random start, with its play rate following the boid's speed. Update rate optimisations are on and the pose only ticks while rendered, but the cost per boid is far higher. A component is created for every boid the flock can hold, so keep the counts small.

The render components have no collision, generate no overlaps and do not affect navigation. They receive decals only with Receives Decals on.

#### Animation provider for instanced skeletal boids

1. In the Content Browser, Add > Miscellaneous > Transform Provider Data, and pick Anim Sequence Transform Provider Data in the class picker.
2. Set Skinned Asset to the boid's skeletal mesh and add the flight loop to Sequences (or point Sequence List at a sequence list asset).
3. In the config, set Animation Provider to the new asset. Animation Sequence Index selects the sequence to loop, counting from 0.

The validator reports a missing provider, an index the provider does not have, and a provider set up for a different mesh.

### Console variables

- `FlockBoid.ForceCPU` (0): when 1, flocks run on the CPU even where the GPU could run them.
- `FlockBoid.CountScale` (1): multiplies Boids Per Flock, minimum and maximum. Results are rounded and never drop below one boid. Flock Count is not scaled.

Both apply to flocks spawned after the change, never to running ones. `FlockBoid.CountScale` is a scalability variable, so it can go into a scalability group in `Config/DefaultScalability.ini`:

```ini
[EffectsQuality@0]
FlockBoid.CountScale=0.25
```

## Materials

### Usage flags

- GPU static mesh flocks: Used with Niagara Mesh Particles.
- CPU static mesh flocks: Used with Instanced Static Meshes.
- Skeletal flocks: Used with Skeletal Mesh, plus Used with Instanced Skinned Mesh in Instanced mode.

Because GPU flocks can fall back to the CPU, their materials need Used with Instanced Static Meshes as well, and the validator checks both flags for Auto and GPU configs. The editor turns on a missing flag by itself when it first draws a material that way. Cooked builds cannot, so they draw the default material.

### Per-boid data

| Value | GPU flocks | CPU instanced meshes |
|---|---|---|
| Phase, in seconds | Dynamic Parameter, index 0, Param1 | Per Instance Custom Data 1 |
| Speed / Max Speed | Dynamic Parameter, index 0, Param2 | Per Instance Custom Data 2 |
| Bank, in radians | Dynamic Parameter, index 0, Param3 | Per Instance Custom Data 3 |
| Random, 0 to 1 | Dynamic Parameter, index 0, Param4 | Per Instance Custom Data 4 |
| Tint | Particle Color | Per Instance Custom Data 5, 6, 7 (R, G, B) |
| Unused | - | Per Instance Custom Data 0 |

- Phase starts at a random point within the first 10 seconds and advances at a rate blended by speed from Play Rate At Min Speed to Play Rate At Max Speed. It keeps growing, so wrap it in the material with a sine, frac or fmod.
- Speed is the current speed divided by Max Speed: about 0.33 at the default Min Speed, 1 at Max Speed, above 1 while panicking.
- Bank is the current lean. Positive means right wing down.
- Random stays fixed for a boid until its flock respawns.
- Custom data slot 0 is left at zero. Niagara's mesh renderer keeps the particle index there when it draws through GPU Scene, which is why the layout starts at 1 and one material can read both paths.
- Component-rendered skeletal boids only get the tint, as Custom Primitive Data 5, 6 and 7, updated when the boid's flock respawns. Read it with a vector parameter that uses custom primitive data at index 5. Phase, speed, bank and random are not available on that path.

`/FlockBoid/Materials/MF_FlockInstanceData` reads all of this and outputs Phase, Speed, Bank, Random and Tint. It works on GPU flocks and on CPU instanced flocks without switching, so one material covers a config that may run on either. A material function cannot hold the Dynamic Parameter node itself, so the node goes in the material: add a Dynamic Parameter node (index 0) and connect its RGBA output to the function's Dynamic Parameter input. Materials only used by CPU flocks can leave that input unconnected. `/FlockBoid/Materials/M_FlockBoid` is a small example material that shows the wiring.

### Driving animation from Phase

Phase is meant as the clock for procedural animation in the material. For a wing flap, rotate the vertices in World Position Offset around the mesh's long axis by `sin(Phase * 2 * pi * FlapsPerSecond) * MaxAngle`, weighted by distance from that axis so the body stays put. With Play Rate At Max Speed above Play Rate At Min Speed, faster boids flap faster.

Animation baked into textures, for example with Epic's AnimToTexture plugin, plays back on a static mesh. Drive its playback time from Phase instead of scene time and a static mesh flock gets per-boid, speed-dependent animation on the GPU path.

## Custom Niagara systems

To add modules or renderers, use a copy of the default system:

1. Duplicate `/FlockBoid/Niagara/NS_Flock` into your own content.
2. Edit the copy, keeping what the flock component relies on:
   - the emitter name `Boids`;
   - the user parameters `FlockSimulation` (a Flock Simulation data interface), `BoidCount` (int), `BoidMesh` (static mesh) and `BoidMaterial0` to `BoidMaterial3` (material);
   - GPU simulation in local space;
   - the single burst of `User.BoidCount` particles, with no modules that spawn more or kill any;
   - the `NMS_FlockBoidState` module in Particle Spawn (Boid Index from `Engine.ExecIndex`) and in Particle Update (Boid Index from `Particles.FlockIndex`).
3. Assign the copy as Niagara System Override in a config's Appearance (advanced), or set it as Default Niagara System in Project Settings > Plugins > Flock Boid for every GPU flock without an override.

`NMS_FlockBoidState` writes position, velocity, mesh orientation, scale, colour, the dynamic material parameter and `Particles.FlockIndex`. Modules placed after it in Particle Update change how a boid is drawn, not where the simulation thinks it is. The Flock Simulation data interface also offers `GetBoidState` (by index) and `GetBoidCount` to your own GPU particle scripts.

The flock component sets every user parameter before the system activates and replaces the system's fixed bounds with the area grown by the containment margin and the largest boid the Scale range allows, measured from the mesh pivot. The config's Appearance settings also replace the copy's own shadow, decal, custom depth and translucency sort overrides. A system without the `FlockSimulation` parameter logs an error and the flock is not drawn.

## Blueprint and C++ API

Flock Spawner component:

- `SpawnFlocks`: spawns every enabled entry, replacing flocks that already exist. With Despawn Distance set and every view out of range, the flocks wait for a view to come close.
- `DespawnFlocks`: removes every flock and keeps the distance check from bringing them back until `SpawnFlocks` is called.
- `AreFlocksSpawned`: true once every flock component exists. GPU flocks wait for their Niagara system to load, which can take a few frames.
- `GetFlockComponents`.
- `OnFlocksSpawned`, `OnFlocksDespawned`.
- Flocks, Area, Spawn Region, Spawn Radius and Spawn On Begin Play can be set from Blueprint.

Flock Spawn Point and Flock Spawn Volume have `GetSpawner`.

The spawner creates one flock component per entry, either a Flock Gpu Component or a Flock Cpu Component:

- `GetConfig`, `GetLiveFlockCount` (fading flocks included), `GetCapacity`.
- `OnFlockSpawned(FlockComponent, Slot)`: a slot spawned a flock. Flocks that exist when the component starts are reported on its first tick.
- `OnFlockDespawned(FlockComponent, Slot)`: a flock finished fading out. When the spawner removes all flocks at once, only its `OnFlocksDespawned` fires.
- `GetNiagaraComponent` on GPU flocks, `GetRenderComponents` on CPU flocks.

The Flock subsystem, a world subsystem, has `GetFlocks` for every flock component in the world.

From C++, `UFlockCpuComponent::GetSimulation` gives the positions and velocities of a CPU flock's boids, in the flock component's space. Call `WaitForSimulation` first.

## Data validation

Flock Configs are checked by data validation. Errors:

- no mesh for the selected Mesh Type;
- instanced skeletal boids without an Animation Provider, or with an Animation Sequence Index the provider does not have;
- component-rendered skeletal boids without an Animation;
- Min Speed above Max Speed;
- a Flock Count or Boids Per Flock maximum below its minimum;
- more boids than an entry can hold (see Performance);
- a GPU flock with a Nanite mesh that has no fallback mesh.

Warnings:

- a material without the usage flag its path needs;
- an animation provider set up for a different skeletal mesh than the boids use.

## Performance

What drives the cost:

- Capacity. A flock component is sized for the maximum flock count times the maximum boids per flock, up front: that many GPU particles, instances or skeletal mesh components. Boids of flocks that are not alive are not simulated, but their particles and instances still exist at zero scale. Keep the maximums close to what you need. An entry holds at most 1,048,576 boids. Larger counts are lowered when the flocks spawn, boids per flock first, with a warning in the `LogFlockBoid` log.
- Max Neighbours. Each boid scans its own grid cell and the 26 around it and stops after accepting Max Neighbours neighbours. This is the main cost setting in dense flocks.
- Perception Radius. It is the grid's cell size, so a larger radius means more boids per cell to test. Its cost depends on how many boids there are and how tightly they crowd, not on the size of the area.
- Obstacles and influences. Every boid tests every obstacle and influence that reaches its flock in every substep. Only those close enough to the area to affect its boids reach a flock, except influences with an unlimited radius, which reach every flock on their channels. Use channels and Enabled to keep the lists short.
- Substeps. A frame is simulated in substeps of at most 1/30 s, up to 4. Frames longer than 4/30 s (about 133 ms) only advance that much, so flocks slow down through hitches instead of jumping.
- CPU flocks. Steering runs on worker threads, but every frame the game thread copies each boid's transform and custom data to the render components, so game thread cost grows with the boid count.
- GPU flocks. The game thread work per flock is small and does not grow with the boid count. The simulation cost lands on the GPU.
- Cull Distance only skips drawing, while Despawn Distance removes the simulation as well.

## Regenerating the Niagara assets

The `FlockAssetBuilder` commandlet rebuilds the plugin's Niagara assets. If `NS_Flock` stops compiling after an engine upgrade, run:

```
UnrealEditor-Cmd.exe <Project>.uproject -run=FlockAssetBuilder
```

It rebuilds `NMS_FlockBoidState`, the `FlockEffectType` effect type and `NS_Flock` under `/FlockBoid/Niagara`, replacing the existing assets. Nothing is saved unless the whole system compiles. It writes into the plugin's Content folder, so that folder has to be writable. Copies of `NS_Flock` in your own content are left alone.

## Tests

The FlockBoidTests editor module ships automation specs named `FlockBoid.*`, runnable from the Session Frontend or with `-ExecCmds="Automation RunTests FlockBoid"`. All of them run headless under `-nullrhi` except `FlockBoid.Rendering.Shaders`, which needs a real RHI because it checks the GPU kernels and the default system's GPU script compile.

## Known limitations

- Flocks only avoid Flock Obstacle components, not level geometry.
- Nothing is replicated. Every client simulates its own flocks, so clients do not see the same boids in the same places, and dedicated servers never spawn any.
- GPU flock state cannot be read from gameplay code.
- GPU flocks use at most four material slots.

## Content

- `/FlockBoid/Samples`: `DA_Flock_Birds`, `DA_Flock_Fish` and `DA_Flock_Insects`, three configs on engine basic shapes.
- `/FlockBoid/Niagara`: `NS_Flock`, the `NMS_FlockBoidState` module and the `FlockEffectType` effect type.
- `/FlockBoid/Materials`: `MF_FlockInstanceData` and `M_FlockBoid`.

There is no demo map. Drop a Flock Spawn Point into any level and assign a sample config.

## Support

Report problems at https://github.com/krojew/FlockBoid/issues.
