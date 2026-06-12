# d3craft

## 题目简述

题目是 Minecraft PaperMC 1.19.4 服务端逻辑绕过。插件要求玩家走到 beacon 并左键拿 flag，但监听 `PlayerMoveEvent`，一旦检测到移动就踢出。解法需要逆向插件确认目标位置和事件逻辑，再结合给出的 Paper patch 分析 `ServerGamePacketListenerImpl` 对移动包、`lastPos`、teleport/collision 的处理差异，最终通过 Fabric mixin 发送“微小前进 + 轻微下移碰撞”移动包，绕过事件触发并逐步移动到目标。

## 解题过程

远端是一个 Minecraft PaperMC 服务器，版本为 1.19.4。提示要求玩家走到 beacon 并挥手（左键）获取 flag。尝试移动后会发现移动被检测到，并被踢出服务器。

附件包含 server jar 文件、配置文件、world、已安装插件，以及一个用于本地调试的 Paper patch 文件。

插件源码可以通过逆向直接得到：

```
package org.d3ctf.d3craft;
import io.papermc.paper.entity.LookAnchor;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import net.kyori.adventure.text.format.TextColor;
import net.kyori.adventure.text.format.TextDecoration;
import org.bukkit.Location;
import org.bukkit.World;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.block.Action;
import org.bukkit.event.player.*;
import org.bukkit.plugin.java.JavaPlugin;
import org.jetbrains.annotations.Contract;
import org.jetbrains.annotations.NotNull;
import java.util.Random;
import static java.lang.Math.sqrt;
public final class Main extends JavaPlugin implements Listener {
    public static final double RADIUS = 16;
    public static final double HORIZON = -60;
    private static final Random random = new Random();
    private static final String flag = "flag{this_is_a_test_flag}";
    private Player currentPlayer = null;
    @Override
    public void onEnable() {
        // Plugin startup logic
        getServer().getPluginManager().registerEvents(this, this);
    }
    @Override
    public void onDisable() {
        // Plugin shutdown logic
    }
    @Contract("_ -> new")
    private @NotNull Location randomPosition(World world) {
        double x = random.nextDouble(16);
        double z = sqrt(RADIUS * RADIUS - x * x);
        boolean sign = random.nextBoolean();
        x *= sign ? 1 : -1;
        sign = random.nextBoolean();
        z *= sign ? 1 : -1;
        return new Location(world, x, HORIZON, z);
    }
    public void sendHello(@NotNull Player player) {
        player.sendMessage("Welcome to");
        player.sendMessage(Component.text(" ____ ", NamedTextColor.RED) .append(Component.text("___  ", TextColor.color(0xFFA500))) .append(Component.text("___ ", NamedTextColor.YELLOW)) .append(Component.text("____   ", NamedTextColor.GREEN)) .append(Component.text("_ _   ", TextColor.color(0x00FFFF))) .append(Component.text("___ ", NamedTextColor.BLUE)) .append(Component.text("___ ", NamedTextColor.DARK_PURPLE))
        );
        player.sendMessage(Component.text("(   _  ", NamedTextColor.RED) .append(Component.text("(__ )", TextColor.color(0xFFA500))) .append(Component.text("/ __", NamedTextColor.YELLOW)) .append(Component.text("|  __ \\ ", NamedTextColor.GREEN)) .append(Component.text("/__\\ ", TextColor.color(0x00FFFF))) .append(Component.text("(___|", NamedTextColor.BLUE)) .append(Component.text("_   _)", NamedTextColor.DARK_PURPLE))
        );
        player.sendMessage(Component.text(" ) (_)  ", NamedTextColor.RED) .append(Component.text("|_  ", TextColor.color(0xFFA500))) .append(Component.text("( (__ ", NamedTextColor.YELLOW)) .append(Component.text(")     /", NamedTextColor.GREEN)) .append(Component.text("/(__)\\ ", TextColor.color(0x00FFFF))) .append(Component.text(")__)  ", NamedTextColor.BLUE)) .append(Component.text(") (", NamedTextColor.DARK_PURPLE))
        );
        player.sendMessage(Component.text("(____", NamedTextColor.RED) .append(Component.text("(___/", TextColor.color(0xFFA500))) .append(Component.text("\\___", NamedTextColor.YELLOW)) .append(Component.text("|_)\\_", NamedTextColor.GREEN)) .append(Component.text("|__) (__", TextColor.color(0x00FFFF))) .append(Component.text("|__)  ", NamedTextColor.BLUE)) .append(Component.text("(__)", NamedTextColor.DARK_PURPLE))
        );
        player.sendMessage(Component.text("Did you see the ") .append(Component.text("light", NamedTextColor.LIGHT_PURPLE)) .append(Component.text(" over there?"))
        );
        player.sendMessage(Component.text("Go there and ") .append(Component.text("wave your
hand.").decoration(TextDecoration.ITALIC, true))
        );
        player.sendMessage(Component.text("I'll give you the ") .append(Component.text("flag").decoration(TextDecoration.BOLD,
true))
        );
    }
    void preparePlayerLocation(@NotNull Player player) {
        Location location = randomPosition(player.getWorld());
        player.teleport(location);
        getLogger().info("set player " + player.getName() + " location to " +
location);
        player.lookAt(0.5, HORIZON, 0.5, LookAnchor.FEET);
        getLogger().info("set player " + player.getName() + " look at flag");
    }
    boolean checkLocation(@NotNull Location location) {
        int x = location.getBlockX();
        int z = location.getBlockZ();
        return x == 0 && z == 0;
    }
    @EventHandler
    public void onPlayerJoin(@NotNull PlayerJoinEvent event) {
        Player player = event.getPlayer();
        sendHello(player);
        preparePlayerLocation(player);
    }
    @EventHandler
    public void onPlayerMove(@NotNull PlayerMoveEvent event) {
        Player player = event.getPlayer();
        if (player == currentPlayer)
            player.kick(Component.text("Hold Still, HACKER!\nDon't MOVE"));
        else if (currentPlayer == null)
            currentPlayer = player;
        else
            player.kick(Component.text("HACKER!"));
    }
    @EventHandler
    public void onPlayerQuit(@NotNull PlayerQuitEvent event) {
        currentPlayer = null;
    }
    @EventHandler
    public void onPlayerInteract(@NotNull PlayerInteractEvent event) {
        Player player = event.getPlayer();
        if (event.getAction() == Action.LEFT_CLICK_AIR &&
checkLocation(player.getLocation())) {
            getLogger().info("player " + player.getName() + " get flag!");
            player.sendMessage(flag);
        }
    }
}
```

如果没有 Minecraft 服务端插件开发经验，也可以结合函数名和进入游戏时的提示，大致猜出该插件的功能。

1. 监听玩家加入世界（`onPlayerJoin`）事件，随机生成一个距离 `(0, -60, 0)` 为 16 的点，并把玩家移动到那里。

2. 监听玩家移动（`onPlayerMove`）事件，并把玩家踢出游戏。

3. 监听玩家交互（`onPlayerInteract`）事件；如果玩家左键点击，并且位置在 `(0, x, 0)`，就给玩家 flag。

玩家加入游戏后，插件会使用 `player.teleport()` 设置玩家位置，这里也会触发 `onPlayerMove` 事件，所以设置了 `currentPlayer` 变量来避免这次被踢出游戏。有些师傅可能会被这里误导，以为进入游戏后需要直接移动过去，这里表示抱歉。

可以删除插件后进入世界，确认 `(0, x, 0)` 是 beacon 位置。

看起来似乎无法移动到 beacon。不过如果尝试非常小幅度地移动，会发现可以移动一小段距离而不被踢出。

思考为什么会这样。由于插件是通过监听事件实现检测的，这说明玩家移动事件没有被监测到。这里有两种可能：一种是客户端认为移动太小，没有向服务端发送数据；另一种是数据已经发送到服务端，但服务端做了某些处理，导致事件没有被触发。

附件给出了 Paper patch 文件，所以需要检查服务端逻辑。

客户端和服务端通信的基本原理是双方持续收发数据包。这里重要的数据包是 `Set Player Position` / 玩家移动包：它携带玩家声明的目标位置和视角信息。如果能修改发出的数据包（例如通过代理），或者自行发送数据包，就能完成正常游玩中无法做到的操作。

事实上，即使玩家没有做任何操作，客户端每 tick 也会发送玩家位置包。

本题使用的服务端是 PaperMC，这是一个第三方开源 Minecraft 服务端。它的工作流基于 patch：先对上游 Minecraft 服务端反编译/去混淆，再应用 Paper/CraftBukkit patch，最后重新构建出 server jar。`CONTRIBUTING.md` 对本题有用的部分是源码生成流程：运行 `./gradlew applyPatches` 生成打 patch 后的源码，把题目 patch 放到 `./patches/server` 下，再 rebuild patches/source，就能检查修改后的 `ServerGamePacketListenerImpl.java`。

官网的下载页面可以找到版本对应的 commit 编号，492 对应 497b919。不过本题与版本本身关系不大，492 只是出题时的最新版。clone 仓库后切换到 commit 497b919，按照文档生成源码（`./gradlew applyPatches`），再把 d3craft patch 放入 `./patches/server`，重新生成源码（`./gradlew rebuildPatches`）。`./Paper-API`、`./Paper-MojangAPI`、`./Paper-Server` 是服务端源码。带有 d3craft patch 的文件是 `./Paper-Server/src/main/java/net/minecraft/server/network/ServerGamePacketListenerImpl.java`。

patch 如下：

```
From 0000000000000000000000000000000000000000 Mon Sep 17 00:00:00 2001
From: WingsZeng <wings.xiangyi.zeng@gmail.com>
Date: Thu, 6 Apr 2023 11:22:18 +0800
Subject: [PATCH] d3craft-patch
diff --git
a/src/main/java/net/minecraft/server/network/ServerGamePacketListenerImpl.java
b/src/main/java/net/minecraft/server/network/ServerGamePacketListenerImpl.java
index
177aac1ab10189bb5a52217e86ba5c8a535b4197..132494836fbb98f6676c3111c95c36b6826ccf
0d 100644
---
a/src/main/java/net/minecraft/server/network/ServerGamePacketListenerImpl.java
+++
b/src/main/java/net/minecraft/server/network/ServerGamePacketListenerImpl.java
@@ -1478,7 +1478,10 @@ public class ServerGamePacketListenerImpl implements
ServerPlayerConnection, Tic
                                 if (d11 - d10 > Math.max(f2, Math.pow((double)
(org.spigotmc.SpigotConfig.movedTooQuicklyMultiplier * (float) i * speed), 2)) &&
!this.isSingleplayerOwner()) {
                                 // CraftBukkit end
                                     ServerGamePacketListenerImpl.LOGGER.warn("{}
moved too quickly! {},{},{}", new Object[]{this.player.getName().getString(), d7,
d8, d9});
-                                    this.teleport(this.player.getX(),
this.player.getY(), this.player.getZ(), this.player.getYRot(),
this.player.getXRot());
+                                    // d3craft start
+                                    // this.teleport(this.player.getX(),
this.player.getY(), this.player.getZ(), this.player.getYRot(),
this.player.getXRot());
+                                    this.internalTeleport(this.lastPosX,
this.lastPosY, this.lastPosZ, this.lastYaw, this.lastPitch,
Collections.emptySet());
+                                    // d3craft end
                                     return;
                                 }
                             }
```

`teleport` 和 `internalTeleport` 有些相似，区别在于参数不同。不过这似乎不是短距离移动后不触发事件的原因。但从这里可以发现，玩家位置似乎有两份记录：一份是 `player` 的 `getX()`，另一份是 `this.lastPosX`。player 的记录位于 `Player` 类中，而 `lastPosX` 位于当前类

`ServerGamePacketListenerImpl` 中。

在 `handleMovePlayer` 函数中查找 PlayerMove 事件，可以看到如下代码（第 1576 行）：

```
      // CraftBukkit start - fire PlayerMoveEvent
      // Rest to old location first
      this.player.absMoveTo(prevX, prevY, prevZ, prevYaw, prevPitch);
      Player player = this.getCraftPlayer();
      Location from = new Location(player.getWorld(), this.lastPosX,
this.lastPosY, this.lastPosZ, this.lastYaw, this.lastPitch); // Get the Players
previous Event location.
      Location to = player.getLocation().clone(); // Start off the To location
as the Players current location.
      // If the packet contains movement information then we update the To
location with the correct XYZ.
      if (packet.hasPos) {
          to.setX(packet.x);
          to.setY(packet.y);
          to.setZ(packet.z);
      }
      // If the packet contains look information then we update the To location
with the correct Yaw & Pitch.
      if (packet.hasRot) {
          to.setYaw(packet.yRot);
          to.setPitch(packet.xRot);
      }
      // Prevent 40 event-calls for less than a single pixel of movement >.>
      double delta = Math.pow(this.lastPosX - to.getX(), 2) +
Math.pow(this.lastPosY - to.getY(), 2) + Math.pow(this.lastPosZ - to.getZ(), 2);
      float deltaAngle = Math.abs(this.lastYaw - to.getYaw()) +
Math.abs(this.lastPitch - to.getPitch());
      if ((delta > 1f / 256 || deltaAngle > 10f) && !this.player.isImmobile()) {
          this.lastPosX = to.getX();
          this.lastPosY = to.getY();
          this.lastPosZ = to.getZ();
          this.lastYaw = to.getYaw();
          this.lastPitch = to.getPitch();
          // Skip the first time we do this
          if (from.getX() != Double.MAX_VALUE) {
              Location oldTo = to.clone();
              PlayerMoveEvent event = new PlayerMoveEvent(player, from, to);
              this.cserver.getPluginManager().callEvent(event);
              // If the event is cancelled we move the player back to their old
location.
              if (event.isCancelled()) {
                  this.teleport(from);
                  return;
              }
              // If a Plugin has changed the To destination then we teleport the
Player
              // there to avoid any 'Moved wrongly' or 'Moved too quickly'
errors.
              // We only do this if the Event was not cancelled.
              if (!oldTo.equals(event.getTo()) && !event.isCancelled()) {
                  this.player.getBukkitEntity().teleport(event.getTo(),
PlayerTeleportEvent.TeleportCause.PLUGIN);
                  return;
              }
              // Check to see if the Players Location has some how changed
during the call of the event.
              // This can happen due to a plugin teleporting the player instead
of using .setTo()
              if (!from.equals(this.getCraftPlayer().getLocation()) &&
this.justTeleported) {
                  this.justTeleported = false;
                  return;
              }
          }
      }
      this.player.absMoveTo(d0, d1, d2, f, f1); // Copied from above
      // CraftBukkit end
```

这段代码做了以下事情：

1. 先把玩家移回 prev 位置（`prevX` 等变量在 `handleMovePlayer` 开头设置，第 1411 行有 `preX = this.player.getX()`）。此时 `player` 的位置此前已经

被修改过，这段代码是 CraftBukkit 加入的 patch，因此会把玩家移动回去。

2. 计算接收到的数据包位置 `to` 与 lastPos 位置（`from`）之间的距离变化（`delta`）和角度变化（`deltaAngle`）。如果位置或视角变化超过阈值（`delta > 1f / 256 || deltaAngle > 10f`），就把 lastPos 更新为 `to`，并触发 `PlayerMoveEvent`；如果位置和视角变化都不大，

则既不更新 lastPos，也不触发 `PlayerMoveEvent`。

3. 移动到 `(d0, d1, d2, f, f1)`。快速阅读代码可以看出，这就是玩家实际需要移动到的位置。

这就是为什么可以小距离移动而不触发事件。不过由于 lastPos 没有更新，如果连续多次小距离移动，差值会不断“累积”，直到超过阈值并触发事件。

可以尝试修改阈值、重新编译服务端并测试。

### 接着思考如何利用

```
    public void internalTeleport(double d0, double d1, double d2, float f, float
f1, Set<RelativeMovement> set) {
        // ...
        this.awaitingPositionFromClient = new Vec3(d0, d1, d2);
        if (++this.awaitingTeleport == Integer.MAX_VALUE) {
            this.awaitingTeleport = 0;
        }
        // CraftBukkit start - update last location
        this.lastPosX = this.awaitingPositionFromClient.x;
        this.lastPosY = this.awaitingPositionFromClient.y;
        this.lastPosZ = this.awaitingPositionFromClient.z;
        this.lastYaw = f;
        this.lastPitch = f1;
        // CraftBukkit end
        this.awaitingTeleportTime = this.tickCount;
        this.player.moveTo(d0, d1, d2, f, f1); // Paper - use proper moveTo for
teleportation
        this.player.connection.send(new ClientboundPlayerPositionPacket(d0 - d3,
d1 - d4, d2 - d5, f - f2, f1 - f3, set, this.awaitingTeleport));
    }
```

在 `internalTeleport` 中可以发现，lastPos 会被设置成参数值。这可以作为绕过 `PlayerMoveEvent` 的利用点。

被 patch 的部分会检查玩家是否移动过快，协议文档中也提到了这一点；如果移动过快，就会把玩家 `teleport`（`internalTeleport`）回去。

先看原始代码，传入的是 `this.player.getX()`、`this.player.getY()`、`this.player.getZ()`，因此 lastPos 最终会被设置成这个值。所以绕过方法如下：

1. 先移动一小步，不触发 `PlayerMoveEvent`，不修改 lastPos，但修改 player location。

2. 再移动很远（修改移动数据包中的位置），进入该 if 分支触发 teleport，并把 lastPos 修改为 player location。

3. 重复上述过程，即可在不触发 `PlayerMoveEvent` 的情况下移动玩家位置。

这个思路很巧妙，但很可惜已经被 patch。

不过，在其他位置也能找到把玩家 teleport 回去的代码（第 1567 行）：

```
    if (!this.player.isChangingDimension() && d11 >
org.spigotmc.SpigotConfig.movedWronglyThreshold && !this.player.isSleeping() &&
!this.player.gameMode.isCreative() &&
this.player.gameMode.getGameModeForPlayer() != GameType.SPECTATOR) { // Spigot
        flag2 = true; // Paper - diff on change, this should be moved wrongly
        ServerGamePacketListenerImpl.LOGGER.warn("{} moved wrongly!",
this.player.getName().getString());
    }
    this.player.absMoveTo(d0, d1, d2, f, f1);
    // Paper start - optimise out extra getCubes
    // Original for reference:
    // boolean teleportBack = flag2 && worldserver.getCubes(this.player,
axisalignedbb) || (didCollide && this.a((IWorldReader) worldserver,
axisalignedbb));
    boolean teleportBack = flag2; // violating this is always a fail
    if (!this.player.noPhysics && !this.player.isSleeping() && !teleportBack) {
        AABB newBox = this.player.getBoundingBox();
        if (didCollide || !axisalignedbb.equals(newBox)) {
            // note: only call after setLocation, or else getBoundingBox is
wrong
            teleportBack = this.hasNewCollision(worldserver, this.player,
axisalignedbb, newBox);
        } // else: no collision at all detected, why do we care?
    }
    if (!this.player.noPhysics && !this.player.isSleeping() && teleportBack) {
// Paper end - optimise out extra getCubes
        this.internalTeleport(d3, d4, d5, f, f1, Collections.emptySet()); //
CraftBukkit - SPIGOT-1807: Don't call teleport event, when the client thinks the
player is falling, because the chunks are not loaded on the client yet.
        this.player.doCheckFallDamage(this.player.getY() - d6,
packet.isOnGround());
    }
```

`d3` / `d4` / `d5` 与 `player.getX()`、`player.getY()`、`player.getZ()` 相同。我无法触发 “move wrong”，但可以在 bounding box 上做文章。

`didCollide` 定义如下：

```
    this.player.move(MoverType.PLAYER, new Vec3(d7, d8, d9));
    boolean didCollide = toX != this.player.getX() || toY != this.player.getY()
|| toZ != this.player.getZ(); // Paper - needed here as the difference in Y can
be reset - also note: this is only a guess at whether collisions took place,
floating point errors can make this true when it shouldn't be...
```

注意，定义后面甚至有一段注释：这里需要该变量，因为 Y 方向差异可能被 reset；同时这只是对是否发生碰撞的猜测，浮点误差可能让它在不该为 true 时变成 true。因此可以尝试制造碰撞。

`hasNewCollision` 定义如下：

```
    // Paper start - optimise out extra getCubes
    private boolean hasNewCollision(final ServerLevel world, final Entity
entity, final AABB oldBox, final AABB newBox) {
        final List<AABB> collisions =
io.papermc.paper.util.CachedLists.getTempCollisionList();
        try {
            io.papermc.paper.util.CollisionUtil.getCollisions(world, entity,
newBox, collisions, false, true,
                true, false, null, null);
            for (int i = 0, len = collisions.size(); i < len; ++i) {
                final AABB box = collisions.get(i);
                if
(!io.papermc.paper.util.CollisionUtil.voxelShapeIntersect(box, oldBox)) {
                    return true;
                }
            }
            return false;
        } finally {
 io.papermc.paper.util.CachedLists.returnTempCollisionList(collisions);
        }
    }
    // Paper end - optimise out extra getCubes
```

它大概是从可能产生碰撞的 collision box cache 中取出候选碰撞盒，然后逐个判断；如果发生碰撞就返回 true。在 collision box 中制造碰撞相对简单：发送一个 Y 轴位置略微降低的 `set player position` 数据包，使其与地面方块的 collision box 相交。可以在服务端代码中打印 `didCollide` 和 `teleportBack` 来确认是否有效。

最终绕过方式很简单：

1. 发送一个数据包，让玩家向前移动极小一步。

2. 发送一个数据包制造碰撞。

3. 重复上述两步，即可在不触发事件的情况下在游戏中移动。

可以实现一个 Fabric mod 来利用它，mixin 如下：

```
package org.d3ctf.d3craftexp.mixin;
import net.minecraft.client.network.ClientPlayerEntity;
import net.minecraft.network.packet.c2s.play.PlayerMoveC2SPacket;
import net.minecraft.util.math.Vec3d;
import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.injection.At;
import org.spongepowered.asm.mixin.injection.Redirect;
@Mixin(ClientPlayerEntity.class)
public abstract class ClientPlayerEntityMixin {
    @Redirect(method = "tick", at = @At(value = "INVOKE", target =
"Lnet/minecraft/client/network/ClientPlayerEntity;sendMovementPackets()V"))
    private void injected(ClientPlayerEntity player) {
        Vec3d pos = player.getPos();
        Vec3d rot = player.getRotationVector();
        player.setPosition(pos.add(rot.multiply(0.06)));
        player.networkHandler.sendPacket(new
PlayerMoveC2SPacket.Full(player.getX(), player.getY(), player.getZ(),
player.getYaw(), player.getPitch(), true));
        player.setPosition(pos.add(new Vec3d(0, -0.01, 0)));
        player.networkHandler.sendPacket(new
PlayerMoveC2SPacket.Full(player.getX(), player.getY(), player.getZ(),
player.getYaw(), player.getPitch(), true));
    }
}
```

## 方法总结

- 核心技巧：Minecraft 插件逆向、PaperMC patch 源码还原、移动包处理逻辑分析、`PlayerMoveEvent` 阈值绕过、collision/teleportBack 利用、Fabric mixin 自动发包。
- 识别信号：服务端插件只监听 Bukkit/Paper 事件而不是原始网络包时，应检查底层 packet handler 是否存在不会触发事件但会更新位置的分支。
- 复用要点：PaperMC 外部文档的关键是如何还原对应版本源码；真正 bypass 来自两个状态差异：小位移不更新 `lastPos`/不触发事件，碰撞/teleportBack 又能把内部位置同步到可继续前进的状态。
