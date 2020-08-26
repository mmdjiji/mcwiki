# 如何编写客户端

创建本教程是为了记录编写一个独立客户机与Notchian服务器交互所需的操作。本教程不完整，但一旦发现更多信息，将进行更新。

**请注意，这里的许多信息都已过时（<2013年4月）。检查 [页面历史记录](http://wiki.vg/index.php?title=How_to_Write_a_Client&action=history) 查看最近更新的内容**

## 在你开始之前

- 确保您不想fork或加入 [现有项目](https://wiki.vg/Server_List) 。
- 想想这会有多少工作量。
- 下载最新的官方Minecraft服务器并在关闭身份验证的情况下运行它。如果需要，可以将其绑定到本地主机。

## 解析消息

*主要文章: [协议](Protocol.md)*

您的客户机必须能够*处理* 所有消息，但不一定能够解析所有数据。可以跳过你不感兴趣的数据包。因为数据包的长度是在每个数据包的开头发送的(参见 [数据包格式](Protocol.md#数据包格式) )，您的客户端只需跳过正确的字节数就可以保持同步。你将会需要库来进行加密(openssl或crypto)和压缩(zlib)。

## 登录

连接到位于 `localhost` 的服务器，端口号为 `25565` 。

*主要文章: [协议常见问题解答](Protocol_FAQ.md)*

解释(不带身份验证和加密): 发送 `0x02` ，获取 `0xFD` ，发送 `0xCD` ，获取 `0x01` ，获取 `0x06` 。然后你最终会得到一个 `0x0D` ，这就是游戏真正开始的时候。

要测试这是否有效，请使用Notchian客户端连接到本地主机服务端，看看是否可以看到自定义客户端出现并漂浮在空中。

## 获取地图区块

一旦你得到了第一个 `0x0D` ，你必须让服务器相信你应该知道你周围的所有地图区块。在接收到第一个 `0x0D` 包之前，Vanilla客户端实际上将开始发送 `0x0D` 包（同时接收 `0x32` 预分区块）

- 每50毫秒发送一个 `0x0A` ~ `0x0D` 。

一开始你会得到一些远离你所在位置的碎片。忽略这些，因为它们所在的块稍后将被完全加载。完整的块以一个圆形(不是正方形)的模式从您所在的位置螺旋形向外加载，最终为总共400个块创建一个20x20的正方形。

要测试这是否有效，请计算收到的全尺寸区块的 `0x33` 映射区块消息数 `(16,128,16)` 。如果您没有收到任何消息，或者您只收到一对，那么您可能没有正确地发送 `0x0A` ~ `0x0D` 。

## 四处走动

每隔50毫秒发送一次您所在位置的服务器更新( `0x0A` ~ `0x0D` )。这一部分是在尝试只发送 `0x0D` 消息的情况下编写的。您可以较慢发送位置更新，但这会导致运行状况更新变慢。

服务器通常会对你的位置保持沉默，这意味着你的动作是可以接受的。如果服务器把你的位置发给你，就意味着你做错了什么。在这种情况下，您必须通过发回相同的数据来道歉，否则服务器将开始忽略任何将来的位置更新。在这个事件中发生的抽搐效应可以在Notchian客户机上看到，当他们试图穿过缺失的区块时。

当您在地面上移动时，行走动画将自动显示在Notchian客户端中。当你直接上下移动时，没有行走动画。

### 允许超人能力

- **飞行** - 在空中漂浮和移动。必须将 `on-ground` 设置为 `false` 才能垂直移动。重力只有在你说它存在的时候才存在。如果服务器在配置上有 `allow-flight=false` ，在一定时间(5秒)后，玩家将被踢出。
- **在空气中行走** - 将 `on-ground` 设置为 `true` 。玩家将可以在空气中行走，好像有地面在他下面支撑着。也许服务器甚至没有检查地面和接近地面之间的关联。同样，要做到这一点，服务器需要启用飞行。
- **任意快速移动** - 更新你的位置，移动到你喜欢的任何地方，每一步移动任意快(只要你有一个明确的路径)。一步测试可达1000米。(这不起作用了，服务器会因为 `Kicked for moving too fast. Hacking? :(` 踢出移动过快的玩家)

### 限制

- 不能与实心块相交。边界框在水平轴上是一个0.3米的远距线，从脚到头顶的距离为1.74米。
- 你不能穿墙。如果路上有坚固的障碍物，不管你走多快，你都得绕过它们。

如果您违反以上任何规则，服务器将向您发送正确的位置。修正后的位置可以保持你在地面上的状态，即使你试图从一个坚实的街区的顶部坠落。请注意，有时增加位置更新消息的速率将有助于服务器跟踪而不是更正您的位置。50毫秒似乎是一个很好的更新间隔。

### 表情和姿势

外观由偏航和俯仰定义。向服务器提供偏航组件可使化身的头部绕垂直轴旋转。玩家身体定向的概念似乎只存在于客户端，当玩家旋转他们的横摆和移动时，Notchian客户端会自动更新。提供俯仰组件可以使你的头像上下旋转。似乎没有对俯仰的限制，所以提供180度的俯仰会导致你的化身的头朝下。见 [协议](Protocol.md) 查看如何校准偏航和俯仰值。

改变你的姿态似乎没有明显的效果。其目的是修改边界框(例如，当玩家蹲伏时，边界框应该更短)。

### 跌落伤害

当地面由假变真，且坠落高度超过4米左右的某个临界值时，会发生坠落伤害。坠落高度似乎是根据地面上变为真的点和地面上为假的跳跃/坠落/飞行的最高点之间的差值来计算的。

如果你以慢速匀速(而不是恒定加速度)朝地板飞行，可以避免坠落伤害。

### 测试/调试

要测试移动是否有效，请从Notchian客户端观察您的化身移动。

如果玩家仍然保持不动，尽管你的位置更新信息，那么也许你没有为服务器更正你的位置而道歉。重新连接给自己一次机会。

如果玩家在Notchian客户端中不可见，尽管服务器给你合理的重生坐标，那么可能你死了。观察 `0x08` 运行状况更新以验证此原因。尝试重新生成或删除服务端的玩家文件( `world/players/<username>.dat` )以从这种情况中恢复。

## 挖掘

If you don't want to be able to abort digging:

1. send a start digging (0) packet and an finish digging (2) packet.
2. server sends you an update block telling you of your success or failure after some amount of time has passed.

If you do want to be able to abort digging:

1. send a start digging (0).
2. wait appropriate amount of time (see below).
3. if you haven't aborted yet, send a finish digging (2).

If you go with the first option, you must wait until the block has been dug before you can start digging another block. In older versions of the server (1.4 and previous) you could bypass the time and destroy lots of blocks at once (instant mining) by sending "start digging" and "finish digging" packets without an interval in between.

### 等待多长时间

The server requires you to wait for a certain amount of time between sending a StartDiggingPacket and a StopDiggingPacket. Below is some old pseudocode; you can find an updated one [here](https://gist.github.com/phase/85d3081047cbf3d2f5ae).

```pseudocode
20 ticks per second

sum = 0
every tick, sum += strengthVsBlock
when sum >= 1.0, block is broken

function strengthVsBlock(tool, block, underwater, on_ground) {
    if block.hardness < 0:
        return 0
    if not canHarvestBlock(tool, block):
        return 1.0 / block.hardness / 100
        
    mult = 1
    
    if tool effective against block:
        mult *= tool.effectiveness
    if underwater:
        mult /= 5
    if not on_ground:
        mult /= 5
    
    return mult / block.hardness / 30
}

function canHarvestBlock(block) {

    if block material is not in (rock, iron, snow, snow_block):
        return true
    else
        return whether equipped item can harvest block

}

effectiveness against proper material
wood: 2.0
stone: 4.0
iron: 6.0
diamond: 8.0
gold: 12.0


Shovel effective against:
Block.grass, Block.dirt, Block.sand, Block.gravel, Block.snow, Block.blockSnow, Block.blockClay


axe effective against:

Block.planks, Block.bookShelf, Block.wood, Block.crate


pick effective against:
            Block.cobblestone, Block.stairDouble, Block.stairSingle, Block.stone, Block.sandStone, Block.cobblestoneMossy, Block.oreIron, Block.blockSteel, Block.oreCoal, Block.blockGold, 

            Block.oreGold, Block.oreDiamond, Block.blockDiamond, Block.ice, Block.bloodStone, Block.oreLapis, Block.blockLapis
            
pick can harvest:
    switch(harvest level of tool):
        case 3:
            obsidian
        case 2:
            diamond
            diamondore
            gold
            goldore
            redstone
            redstoneglowing
        case 1:
            iron
            ironore
            lapis
            lapisore
            
    if block.material is rock
        return true
    if block.material is iron
        return true
    
    
    return false

shovel can harvest:
    both types of snow = true
    else false

axe can harvest:
    false


harvest level:
wood: 0
stone: 1
iron: 2
diamond: 3
gold: 0

# 对于下列字段，参见 https://github.com/PrismarineJS/minecraft-data/blob/master/data/1.8/blocks.json
Block.hardness
Block.material
```

### 在墙后挖掘/放置方块

- Neither the Notchian server nor Bukkit check whether you actually have line-of-sight to a block when you dig or place blocks. Thus, if you noclip through a wall and mine the blocks behind it, the server will correct your position and ignore position updates, but still break those blocks which were within four meters of your corrected position. Likewise, any chests, buttons or levers can be used as long as they are within that critical 4-m distance.

## 背包与合成

When you log in, before you get a 0x0D, you will get a [Window Items](https://wiki.vg/Protocol#Window_Items) with all inventory slots including armor. You will then get a series of [Set Slot](https://wiki.vg/Protocol#Set_Slot) repeating the slot information, one for every non-empty slot in the opened window, plus the cursor slot (which is empty by default, but changes when you click on a slot). The slot indexes of these messages are for an inventory window with the crafting zone empty.

By clicking on a block or entity, the client can open a window. After sending the [Player Block Placement](https://wiki.vg/Protocol#Player_Block_Placement) or [Use Entity](https://wiki.vg/Protocol#Use_Entity) packet, the server responds with [Open Window](https://wiki.vg/Protocol#Open_Window), [Window Items](https://wiki.vg/Protocol#Window_Items), several [Set Slot](https://wiki.vg/Protocol#Set_Slot) (one for every non-empty slot in the opened window, plus the cursor slot), and optionally [Window Property](https://wiki.vg/Protocol#Window_Property). It seems that only the most recently opened window can be interacted with. For more info, look at [Inventory](https://wiki.vg/Inventory).

Clicking happens by sending [Click Window](https://wiki.vg/Protocol#Click_Window), which the server will answer with a [Confirm Transaction](https://wiki.vg/Protocol#Confirm_Transaction). If the "Clicked item" is wrong, the click will still be *executed* but not *accepted*.

If the click was not accepted by the server (accepted = false), the client must answer with [Confirm Transaction (serverbound)](https://wiki.vg/Protocol#Confirm_Transaction_2) to apologize (like with movement), otherwise the server ignores any successive clicks. Additionally, the server answers erroneous clicks with one [Window Items](https://wiki.vg/Protocol#Window_Items) and several [Set Slot](https://wiki.vg/Protocol#Set_Slot) as above, to resynchronize the client with the server.

The client has to know all clicking behaviour (including crafting recipes) and change the slots accordingly after each click, because the server does not send any [Set Slot](https://wiki.vg/Protocol#Set_Slot), just [Confirm Transaction](https://wiki.vg/Protocol#Confirm_Transaction). But you can cheat and always send the wrong "Clicked item" in [Click Window](https://wiki.vg/Protocol#Click_Window) to force the server to send you all slots again while still applying your click.

Crafting is done by placing the correct materials into the crafting grid and clicking on the result slot (always slot nr. 0) to pick up the crafted item. Make sure to correctly update all slots after clicking the result slot, as some recipes (e.g. cake) leave back items in the crafting grid/inventory (e.g. empty buckets).

## 攻击怪物/玩家

To attack another player or mob, send a 0x07 with the entity's id. The range seems to be limited to 4 meters or so and there must be a clear line of sight. If these conditions are not met, nothings happens. Damage is calculated on the server side.

## 聊天

*主要文章: [Chat](https://wiki.vg/Chat)*

## 血量与重生

The server sends [0x08 Health updates](https://wiki.vg/Protocol#0x08) whenever your health changes. If the server sends a health value less than 1, you are dead. Your avatar will disappear a second or two after dying only if you keep sending position updates (packets 0x0A through 0x0D). The "fall over" animation is not automatic.

After you get a death notice, you must still update your position while being dead. (Otherwise your dead body will be visible by notchian clients of the server, and you will be invisible after respawning.) Then, you can send a [0xCD Client Status](https://wiki.vg/Protocol#0xCD) to respawn. You'll be given a 0x0D with your new position, and possibly other messages.

If your health updates come much slower than on the notchian client, make sure you are sending position updates fast enough.



From https://wiki.vg/How_to_Write_a_Client