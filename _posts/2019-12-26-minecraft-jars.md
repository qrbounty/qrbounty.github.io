---
layout: post
title:  "Patching A Minecraft Mod Without Source"
date:   2019-12-26 01:11:00 +0800
categories:
---
# Patching A Minecraft Mod Without Source
 
## Contents
1. [The Situation](#the-situation)
2. [Analyzing The Problem](#analyzing-the-problem)
3. [Getting Our Hands Dirty](#getting-our-hands-dirty)
4. [Lessons Learned](#lessons-learned)
 
## The Situation
I play Minecraft with my wife on our own private server running Forge. We're playing the current version (1.15.1) but a lot of mods are currently still on previous versions, namely 1.14. I recently decided to try getting the ["Extra Bows" mod][1] (for 1.14) loaded because I was hoping it was simple enough to be natively compatible. It worked perfectly, except the little part where the entire server and all of the clients get a fatal error when someone actually tries to _fire a bow_. Minor detail, I know.  
 
[crash gif?]  
 
At this point I figured I'd pull down the source, make some edits, and compile as needed, however the source code was nowhere to be found in the official mod repository. Great.  
 
For some reason I was hellbent on making this mod work, probably because I had a stack of 10 or 15 other mods I wanted that weren't compatible yet. I don't code in Java much anymore but I've messed with .jar files in the past, mostly just to try to decompile and analyze, not patch. This time around I didn't want to have to try to fully decompile, edit, and recompile since I don't have any Java dev tools installed, and because it sounded like a fun little challenge get things working without the source.   
 
## Analyzing The Problem
Let's take a look at the state of things when we try to fire an ~~error~~ arrow.  
 
The Crash:
```
java.lang.IllegalAccessError: tried to access field net.minecraft.entity.Entity.field_70165_t from class me.marnic.extrabows.common.items.BasicBow
	at me.marnic.extrabows.common.items.BasicBow.func_77615_a(BasicBow.java:148) ~[?:v1.14.4 b6] {re:classloading}
	at net.minecraft.item.ItemStack.func_77974_b(ItemStack.java:433) ~[?:?] {re:classloading,pl:runtimedistcleaner:A}
	at net.minecraft.entity.LivingEntity.func_184597_cx(LivingEntity.java:2677) ~[?:?] {re:classloading,pl:runtimedistcleaner:A}
	at net.minecraft.network.play.ServerPlayNetHandler.func_147345_a(ServerPlayNetHandler.java:828) ~[?:?] {re:classloading}
	at net.minecraft.network.play.client.CPlayerDiggingPacket.func_148833_a(SourceFile:40) ~[?:?] {re:classloading}
	at net.minecraft.network.play.client.CPlayerDiggingPacket.func_148833_a(SourceFile:10) ~[?:?] {re:classloading}
	at net.minecraft.network.PacketThreadUtil.func_225383_a(SourceFile:21) ~[?:?] {re:classloading}
	at net.minecraft.util.concurrent.TickDelayedTask.run(SourceFile:18) ~[?:?] {re:classloading}
	at net.minecraft.util.concurrent.ThreadTaskExecutor.func_213166_h(SourceFile:144) ~[?:?] {re:classloading,pl:accesstransformer:B}
	at net.minecraft.util.concurrent.RecursiveEventLoop.func_213166_h(SourceFile:23) ~[?:?] {re:classloading}
	at net.minecraft.util.concurrent.ThreadTaskExecutor.func_213168_p(SourceFile:118) ~[?:?] {re:classloading,pl:accesstransformer:B}
	at net.minecraft.server.MinecraftServer.func_213205_aW(MinecraftServer.java:711) ~[?:?] {re:classloading,pl:accesstransformer:B,pl:runtimedistcleaner:A}
	at net.minecraft.server.MinecraftServer.func_213168_p(MinecraftServer.java:705) ~[?:?] {re:classloading,pl:accesstransformer:B,pl:runtimedistcleaner:A}
	at net.minecraft.util.concurrent.ThreadTaskExecutor.func_213160_bf(SourceFile:103) ~[?:?] {re:classloading,pl:accesstransformer:B}
	at net.minecraft.server.MinecraftServer.func_213202_o(MinecraftServer.java:690) ~[?:?] {re:classloading,pl:accesstransformer:B,pl:runtimedistcleaner:A}
	at net.minecraft.server.MinecraftServer.run(MinecraftServer.java:638) [?:?] {re:classloading,pl:accesstransformer:B,pl:runtimedistcleaner:A}
	at java.lang.Thread.run(Unknown Source) [?:1.8.0_221] {}
```
 
So what can we glean from this?
 
- `me.marnic.extrabows.common.items.BasicBow.func_77615_a` within BasicBow.java caused the error.
- It was trying to access a field from Vanilla minecraft, `net.minecraft.entity.Entity.field_70165_t` 
- With a bit of knowledge about .jar files we know the file we need to examine closer should be at `/me/marnic/extrabows/common/items/BasicBow.class` (.java source files get compiled to .class in this case)
 
Okay, looks like we're off to hunt down `func_77615_a` within `BasicBow.class` and fix whatever is causing the issue. To get a better idea of the situation let's check the .jar in one of my go-to quick Java decompilation tools - [JD-GUI][2]. Here's what We get from JD-GUI:
 
```java
  public void func_77615_a(ItemStack bowStack, World worldIn, LivingEntity entityLiving, int timeLeft) {
    if (entityLiving instanceof PlayerEntity) {
      PlayerEntity playerEntity = (PlayerEntity)entityLiving;
      boolean flag = (playerEntity.field_71075_bZ.field_75098_d || EnchantmentHelper.func_77506_a(Enchantments.field_185312_x, bowStack) > 0);
      ItemStack arrowStack = findAmmoNEW(playerEntity);
      int i = func_77626_a(bowStack) - timeLeft;
      i = ForgeEventFactory.onArrowLoose(bowStack, worldIn, playerEntity, i, (!arrowStack.func_190926_b() || flag));
      if (i < 0)
        return; 
      UpgradeList list = UpgradeUtil.getUpgradesFromStackNEW(bowStack);
      if (!arrowStack.func_190926_b() || flag) {
        if (arrowStack.func_190926_b())
          arrowStack = new ItemStack((IItemProvider)Items.field_151032_g); 
        float f = ArrowUtil.getArrowVelocity(i, this);
        if (f >= 0.1D) {
          boolean flag1 = (playerEntity.field_71075_bZ.field_75098_d || (arrowStack.func_77973_b() instanceof ArrowItem && ((ArrowItem)arrowStack.func_77973_b()).isInfinite(arrowStack, bowStack, playerEntity)));
          if (!worldIn.field_72995_K) {
            list.applyDamage(playerEntity);
            if (list.hasMul()) {
              list.getArrowMultiplier().handleAction(this, worldIn, bowStack, playerEntity, f, arrowStack, flag1, list);
            } else {
              AbstractArrowEntity entityarrow = ArrowUtil.createArrowComplete(worldIn, bowStack, playerEntity, this, f, arrowStack, flag1, 0.0F, 0.0F, list);
              worldIn.func_217376_c((Entity)entityarrow);
            } 
          } 
          worldIn.func_184148_a((PlayerEntity)null, playerEntity.field_70165_t, playerEntity.field_70163_u, playerEntity.field_70161_v, SoundEvents.field_187737_v, SoundCategory.PLAYERS, 1.0F, 1.0F / (RandomUtil.RANDOM.nextFloat() * 0.4F + 1.2F) + f * 0.5F);
          if (!flag1 && !playerEntity.field_71075_bZ.field_75098_d) {
            if (list.hasMul()) {
              list.getArrowMultiplier().shrinkStack(arrowStack);
            } else {
              arrowStack.func_190918_g(1);
              if (arrowStack.func_190926_b())
                playerEntity.field_71071_by.func_184437_d(arrowStack); 
            } 
            if (!worldIn.field_72995_K && 
              bowStack.func_77952_i() == bowStack.func_77958_k()) {
              list.dropItems(playerEntity);
              bowStack.func_222118_a(1, (LivingEntity)playerEntity, p -> p.func_213334_d(p.func_184600_cs()));
            } 
          } 
          playerEntity.func_71029_a(Stats.field_75929_E.func_199076_b(this));
        } 
      } 
    } 
  }
```
 
Okay, well that doesn't help nearly as much as it could. From this we're able to see we're checking that we have some arrows, and there are a few world-centric checks. Let's find that `playerEntity.field_70165_t` bit that caused the crash:
 
```java
worldIn.func_184148_a((PlayerEntity)null, playerEntity.field_70165_t, playerEntity.field_70163_u, playerEntity.field_70161_v, SoundEvents.field_187737_v, SoundCategory.PLAYERS, 1.0F, 1.0F / (RandomUtil.RANDOM.nextFloat() * 0.4F + 1.2F) + f * 0.5F);
```
 
And let's break it down a bit:
 
```java
worldIn.func_184148_a(
	(PlayerEntity)null, 
    playerEntity.field_70165_t, // <--- Problem 
    playerEntity.field_70163_u, // Seems closely related
    playerEntity.field_70161_v, // Seems closely related
    SoundEvents.field_187737_v, // Sound 
    SoundCategory.PLAYERS,		// Sound
    1.0F, 
    1.0F / (RandomUtil.RANDOM.nextFloat() * 0.4F + 1.2F) + f * 0.5F
);
```
 
At this point we don't have the hints we need from real source and I don't have any knowledge of native Minecraft playerEntity calls. I also tried searching online for code changes between 1.14 and 1.15, but there were no "Hey developers, `this` property is now `that`" docs available. What about seeing if someone else has been talking about `net.minecraft.entity.entity.field_70165_t`?
 
Gold! Deep in an old Bukkit repository on [someone's GitLab][3] I found a translation table that cleared things up instantly: 
 
```
FD: net/minecraft/server/Entity/locX net/minecraft/entity/Entity/field_70165_t
FD: net/minecraft/server/Entity/locY net/minecraft/entity/Entity/field_70163_u
FD: net/minecraft/server/Entity/locZ net/minecraft/entity/Entity/field_70161_v
```
 
Perfect! Those three variables are indeed closely related: They're the physical coordinates of our character. Now we know what exactly is causing the error: When a bow is fired a new arrow entity is created and launched from the current player location. We just need to take care of the permission error somehow now.
 
## Getting Our Hands Dirty
Enter: [DirtyJOE Java Overall Editor][4]. Okay, what the heck is DirtyJOE? I'm sure there are a ton of tools to edit .class files, but this tool happened to show up while I was on [StackOverflow][5] and it should do the trick perfectly. It doesn't seem to have been updated in several years but for now it will work.
 
### The Setup
Cutting to the chase here:
- Grab the DirtyJOE binary
- Grab the "Extra Bows" .jar file
- Use 7zip (I'm on Windows today) to pop open the .jar file (leave it open, we can add files back to a .jar with 7zip seamlessly!) 
- Rip out `/me/marnic/extrabows/common/items/BasicBow.class` and put it into a working directory
- Load `BasicBow.class` into DirtyJOE
 
If everything worked correctly something like this appears:
 
[DirtyJOE1.png]
 
From this screenshot we can surmise we'll be able to access and potentially edit the following with DirtyJOE:
 
- Constants
- Fields
- Methods
- Class metadata
 
Perfect! At this point we spend 10 or 15 minutes getting familiar with both DirtyJOE and our .class file. What we need next is to locate our `func_77615_a` function, and then find `playerEntity.field_70165_t` once again. We'll find this under the "Methods" tab.
 
[DirtyJOE2.png]
 
Here's what we can tell from this new information:
- We see a checkcast call to make sure there's a PlayerEntity involved, so that hints at this being a player property.
- We see three getfield commands close together. From context we surmise these are the X, Y, and Z properties.
- We see those SoundEvent items as well, so we know where our area of interest tapers off.
 
Great, this is helpful. So... What now? Let's think about why the newest version of Minecraft is having trouble with this straight-forward code. It's possible that the base game changed the way player coordinates are accessed. In fact that's just about the only answer since the code we're looking at is pretty much just a variable reference.
 
### The Switch
What if we got an example of a mod designed for 1.15 that *does* work? We just need to find a mod that collects player coordinates properly. Luckily there's a perfect mod out there for us: the alpha version of a 1.15 release of [TorchBowMod][6]. To be fair I checked out probably 5 other mods before encountering TorchBowMod. Either way, let's forge ahead.
 
- Grab the "TorchBowMod" .jar file
- Use 7zip to pop open the .jar file and extract `mod\torchbowmod\TorcchBow.class` to a working directory.
- Load `mod\torchbowmod\TorcchBow.class` in Dirty Joe
- Look around functionality that looks like coordinate gathering. 
- There are only 10 or so methods so examine each one, looking for the pattern of three fields being accessed.
- In "func_77615_a" of TorchBow.class (the same name of the broken method in our other mod!) we see a very similar pattern followed by the same sound event of the other mod.
 
[DirstyJOE3.png]
 
Aha! So what's different? Well we're using `invokevirtual` here instead of `getfield`. This means we're calling an instance of a method instead of fetching the field from an object. It seems the original mod was directly accessing the player coordinate variables, and in 1.15 perhaps the preferred way is to call a handler method that returns an equivalent. Okay so we have our original code and some working new code, now how do we port this? We can't just copy and paste source code. We're going to need to bootstrap the broken .class with some constants that will point the code in the right direction.
 
After some exploring it appears that 10 constants are needed to properly refer to the elements in question:
- 1 `Utf8` descriptor, `()D` in this case. Why `()D`? I have no idea. A-C were taken and I was  mostly just matching the pattern I was seeing.
- 3 `Utf8` descriptors for the methods that relate to x, y, and x (`func_226278_ct_`, `func_226278_cu_`, `func_226278_cx_`)
- 3 `NameAndType` constants linking the above x, y, and z functions to the `()D` descriptor
- 3 `Methodref` constants linking the `NameAndType` constants to the `PlayerEntity` Minecraft class
 
Here's what we end up with in order to wire up our hotfix: 
 
[PatchedConstants.png]
 
After we create these constants we need to take note of the pointers for the new items from the constant pool. We'll use these to replace the Java opcodes in the `func_77615_a` method. The original opcodes and those seen in "TorchBowMod" are detailed below.
 
[BasicBowOpCodes.png]
[TorchBowOpCode.png]
 
| BasicBow| TorchBow |
| ------------- |-------------| 
| `B4 00 FE` | `B6 00 D2` | 
| `B4 01 01` | `B6 00 D5`      |
| `B4 01 04` | `B6 00 D8`      | 
 
Fiddling with the first byte of these opcodes reveals that the type of call can easy be changed. `B4` is `getfield` and `B6` is `invokevirtual`. So we're 1/3rd of the way there. The other bytes are simply the constant pool table `MethodRef` entries we created a minute ago, which will vary a bit. 
 
[PatchedOpCodes.png]
 
| BasicBow| TorchBow | Patched  |
| ------------- |----------------| --- |
| `B4 00 FE` | `B6 00 D2` | `B6 01 F4` | 
| `B4 01 01` | `B6 00 D5`      | `B6 01 F5` | 
| `B4 01 04` | `B6 00 D8`      | `B6 00 F6` | 
 
### The Results
Once the patched code is applied we just save a new copy of the BasicBow.class file, drop it back in the mod's .jar file, and run the game. 
 
It works! 
 
[shoot gif?]
 
Here's what our newly hacked together class file like decompiled:
 
```java
worldIn.func_184148_a(
	(PlayerEntity)null, 
    playerEntity.func_226277_ct_(), 
    playerEntity.func_226278_cu_(), 
    playerEntity.func_226281_cx_(), 
    SoundEvents.field_187737_v, 
    SoundCategory.PLAYERS, 
    1.0F, 
    1.0F / (RandomUtil.RANDOM.nextFloat() * 0.4F + 1.2F) + f * 0.5F
);
```
 
A pretty simple change but super fun to make it without working directly from source.
 
## Lessons Learned
- Jar files are pretty easy to modify as regular archives. I used 7zip in this case since I was on my gaming computer (Windows).
- .class files are semi-readable in a text editor or hex editor but more abstraction is needed.
- Tools like DirtyJOE can let us edit methods directly without recompiling class files.
	- Methods, fields, and constants are all editable. This includes access flags.
- In the future while exploring .class files I'll probably hop into DirtyJOE if I might be able to leverage some code edits
- From a hacking perspective this opens up some opportunities for quickly patching .jar files. 
- This was a single error in a straight-forward mod. This kind of patching is definitely not reliable for more complex mods. 
- I'm pretty confident there's a better way to do this but since my Java interactions these days mostly consist of dropping as many .jar files as possible into a mods folder I'm not aware of better tools.
 
Overall this was an exciting little fix for a minor Minecraft mod annoyance.
 
[1]: https://www.curseforge.com/minecraft/mc-mods/extra-bows/files/2796951
[2]: https://java-decompiler.github.io/
[3]: https://gitlab.ultramine.ru/ultramine/um_bukkit/blob/ee0552a5a442dbc73e1ff1a37050113eab63e220/src/main/resources/mappings/v1_7_R1/cb2numpkg.srg?expanded=true&viewer=simple
[4]: http://dirty-joe.com/
[5]: https://stackoverflow.com/questions/38214462/java-how-can-i-edit-a-class-file
[6]: https://www.curseforge.com/minecraft/mc-mods/torchbowmod/files/2846878
