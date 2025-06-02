# 使用纯EUD技术开发星际争霸AI

## 将电脑玩家类型修改为人类
星际争霸的原生电脑AI拥有自主的单位控制逻辑，这些内置行为会严重干扰EUD脚本对单位的控制。为了精准有效地实现EUD对单位的操作，​**必须尽可能消除星际争霸原生程序对电脑单位的控制干扰**。解决方案是将目标电脑玩家的player type改为human，使星际争霸将电脑玩家识别为人类玩家。

内存地址：https://armoha.github.io/eud-book/offsets/ActivePlayerStructures.html

最简用法：
```js
// main.eps
function onPluginStart() {
    bwrite_epd(EPD(0X57EEE0) + 9 * player + 0x2, 0, 2);
}
```
典型使用案例, 1v7 AI:
```py
# config.py
AI_PLAYER_IDS = list(range(1, 8))


# core.py
from eudplib import *
from config import *

def set_player_type_to_human():
    actions = []

    for player in AI_PLAYER_IDS:
        actions.append(SetMemoryXEPD(EPD(0X57EEE0) + 9 * player + 0x2, SetTo, 2, 0xFF))

    DoActions(actions)


# main.py
import core as c

def onPluginStart():
    c.set_player_type_to_human()
```
以上案例中，使用了技巧，将7个修改动作合并为了一个动作，效率比以下更高：
```py
# low-efficient solutions, consume more triggers
def set_player_type_to_human():
    for player in AI_PLAYER_IDS:
        f_bwrite_epd(EPD(0X57EEE0) + 9 * player + 0x2, 0, 2)
```

## 禁用单人模式
玩家输入单机作弊码进行游戏可能不是作者们所愿意见到的，于是我们可以禁用单人模式。

内存地址：https://armoha.github.io/eud-book/offsets/MultiplayerMode.html

最简用法：
```js
// main.eps
function onPluginStart() {
    if (Memory(0x57F0B4, Exactly, 0)) {
        setcurpl(P1);
        Defeat();
    }
}
```
典型使用案例：
```py
# config.py
HUMAN_PLAYER = 0


# core.py
from eudplib import *
from config import *

def is_single_player_Mode():
    return Memory(0x57F0B4, Exactly, 0)


def defeat_if_single_player_mode():
    if EUDIf()(is_single_player_Mode()):
        f_setcurpl(HUMAN_PLAYER)
        f_eprintln("\x06请在局域网或者战网玩！\x04为了防止输入单机作弊码，禁止使用单人模式游玩。")
        DoActions(Defeat())
    EUDEndIf()


# main.py
import core as c

def onPluginStart():
    c.defeat_if_single_player_mode()
```

## 取消部分单位的Men属性
在游戏中，我们经常需要Order所有Men发起进攻，但是电脑的农民等非战斗单位也被命令了，这是因为那些单位在星际争霸里都属于Men. 我们可以提前修改这些单位的groupFlags, 把Men属性去除，这样就可以安心地使用Order命令让电脑对玩家发起进攻了。

特殊单位，如魔法兵种、农民等单位，必须使用EUD AI单独微操。

最简用法：
```js
function onPluginStart() {
    TrgUnit("Protoss Probe").groupFlags.Men = 0;
}
```
典型使用案例：
```py
# config.py
UNSET_MEN_GROUPFLAGS_LIST = [
    "Protoss Probe", 
    "Protoss Dark Archon", 
    "Protoss High Templar", 
    "Protoss Observer", 
    "Protoss Shuttle", 
    "Protoss Arbiter", 
]


# core.py
from eudplib import *
from config import *

def unset_men_groupFlags():
    for unit in UNSET_MEN_GROUPFLAGS_LIST:
        TrgUnit(unit).groupFlags.Men = 0


# main.py
import core as c

def onPluginStart():
    c.unset_men_groupFlags()
```

## 禁止AI的虫族房子开局乱动
开局AI的overlord会随机运动，为了禁止这个行为，最直观的方法是修改cunit.order, 这里我们使用一个功能等价，但是效率远高得多的方案。

这个方案看起来反直觉，但是功能上是完美的，应用这个函数后，进游戏后overlord是完全静止的，朝向等一切都没变。

最简用法：
```js
function onPluginStart() {
    MoveLocation("loc", "Zerg Overlord", P1, "Anywhere");
    MoveUnit(1, "Zerg Overlord", P1, "loc", "loc");
}
```
典型使用案例：
```py
# config.py
AI_PLAYER_IDS = list(range(1, 8))


# core.py
from eudplib import *
from config import *

def stop_overlord():
    actions = []

    for player in AI_PLAYER_IDS:
        actions.append(MoveLocation("loc", "Zerg Overlord", player, "Anywhere"))
        actions.append(MoveUnit(1, "Zerg Overlord", player, "loc", "loc"))
    
    DoActions(actions)


# main.py
import core as c

def onPluginStart():
    c.stop_overlord()
```

## 新农民自动采水晶（通用）
以下是在游戏的绝大多数时候，新造出来的农民默认自动采水晶的方案。该方案暂时未考虑贫矿分农民以及虫族的新单位的特殊性这两个细节，之后再更新，现在先把已完成的工作记录上来。

最简用法：
```js
function afterTriggerExec() {
    foreach (cunit: EUDLoopNewCUnit()) {
        if (cunit.unitType == "Protoss Probe") {
            cunit.order = "Move to Harvest Minerals";
        }
    }
}
```
典型使用案例：
```py
# config.py
HUMAN_PLAYER = 0


# unit.py
from eudplib import *
from config import *

new_unit_queue_list = [
    "Zerg Drone", 
    "Terran SCV", 
    "Protoss Probe", 
]

new_unit_queue = {}

for unit in new_unit_queue_list:
    new_unit_queue[unit] = EUDQueue(8)()


def process_new_units():
    for cunit in EUDLoopNewCUnit():
        EUDContinueIfNot(cunit.playerID == HUMAN_PLAYER) # Not AI player
        EUDContinueIfNot(cunit.unknown0x106 == 0)  # Not specially marked

        EPDSwitch(cunit + 0x64 // 4, 0xFF)
        for unit in new_unit_queue_list:
            if EUDSwitchCase()(EncodeUnit(unit)):
                new_unit_queue[unit].append(cunit)
                EUDBreak()
        EUDEndSwitch()


# resource.py
from eudplib import *
import unit as u

WORKER_TYPES = ("Zerg Drone", "Terran SCV", "Protoss Probe")

class Orders:
    Harvest_Minerals = EncodeUnitOrder("Move to Harvest Minerals")


def new_worker_harvest():
    for unit in WORKER_TYPES:
        queue = u.new_unit_queue[unit]
        if EUDWhileNot()(queue.empty()):
            cunit = CUnit.cast(queue.popleft())
            cunit.order = Orders.Harvest_Minerals
        EUDEndWhile()


# main.py
import resource as r
import unit as u

def beforeTriggerExec():
    u.process_new_units()

def afterTriggerExec():
    r.new_worker_harvest()
```

## 矿区资源初始化
星际争霸AI一个非常首要的任务就是命令工人采集资源和扩张，现阶段为了简单起见，我固定AI在地图上的出生点位，这样我就能够简单高效地进行初始化。

首先需要利用SCMD在地图的矿区的基地处放置location, 在示例中，我按照以下命名格式放置了4个location, 意味着示例AI最多支持4片矿。
```python
# config.py
BASE_NAMES = [f"base_{i}" for i in range(4)]
```
以下获取资源块epd地址的方法是通过读取单位在chk结构中的索引计算获得，这样做的好处是在编译期完成，使得输出地图的触发数量和文件大小都比较小。缺点是可能会由于某些无效单位导致所有资源块的epd地址产生系统偏移，所以更保险的做法是在游戏开始的时候再进行初始化。
```python
# resource.py
import config
from eudplib import *
import struct


def get_resource_epd(proximity_threshold: int = 256) -> tuple[list[list[int]], list[list[int]]]:
    """Generate grouped Mineral Field/Vespene Geysers epd lists by base.

    Args:
        proximity_threshold: Maximum bounding box distance in pixels (default: 256)

    Returns:
        Tuple containing:
        - minerals_by_base: List of mineral EPD lists for each base
        - vespenes_by_base: List of vespene EPD lists for each base
    """
    # Load CHK sections. https://www.starcraftai.com/wiki/CHK_Format
    chkt = GetChkTokenized()
    unit_data = chkt.getsection("UNIT")
    thg2_data = chkt.getsection("THG2")
    mrgn_data = chkt.getsection("MRGN")

    # region epd Calculation Logic
    # Count unit sprites from THG2
    unit_sprite_count = sum(
        1 for entry in struct.iter_unpack('<HHHBBH', thg2_data)
        if not (entry[5] & 0x1000)
    )

    # Process UNIT entries
    all_resource_in_map = []
    invalid_unit_count = 0
    RESOURCE_IDS = {
        EncodeUnit("Mineral Field (Type 1)"),
        EncodeUnit("Mineral Field (Type 2)"),
        EncodeUnit("Mineral Field (Type 3)"),
        EncodeUnit("Vespene Geyser"),
    }

    for offset in range(0, len(unit_data), 36):
        x, y, unit_id = struct.unpack_from('<4xHHH26x', unit_data, offset)

        # Track invalid units
        # Only consider Start location, without considering unit un-placeable and vacant player's pre-placed unit
        if unit_id == EncodeUnit("Start Location"):
            invalid_unit_count += 1
            continue

        if unit_id not in RESOURCE_IDS:
            continue

        # Memory address calculation
        runtime_index = (offset // 36) - invalid_unit_count + unit_sprite_count
        ptr = 0x59CCA8 + (1700 - runtime_index) * 336
        epd = (ptr - 0x58A364) // 4

        all_resource_in_map.append((x, y, epd, unit_id))
    # endregion

    # region Base Processing
    # Parse base centers from MRGN
    base_centers = []
    for idx in [EncodeLocation(name) for name in config.BASE_NAMES]:
        x1, y1, x2, y2 = struct.unpack_from('<4I', mrgn_data, (idx - 1) * 20)
        base_centers.append(((x1 + x2) // 2, (y1 + y2) // 2))

    # Initialize result containers
    minerals = [[] for _ in base_centers]
    vespenes = [[] for _ in base_centers]
    # endregion

    # region Resource Distribution
    for x, y, epd, unit_id in all_resource_in_map:
        closest_base = -1
        min_distance = float('inf')

        for base_idx, (base_x, base_y) in enumerate(base_centers):
            # Fast bounding box check
            dx, dy = abs(x - base_x), abs(y - base_y)
            if dx > proximity_threshold or dy > proximity_threshold:
                continue

            # Precise distance calculation
            distance_sq = dx ** 2 + dy ** 2
            if distance_sq < min_distance:
                min_distance = distance_sq
                closest_base = base_idx

        # Assign to nearest base
        if closest_base != -1:
            target = vespenes if unit_id == EncodeUnit("Vespene Geyser") else minerals
            target[closest_base].append(epd)
    # endregion

    return minerals, vespenes


# Index of the base location
BASE_LOC = [EncodeLocation(name) for name in config.BASE_NAMES]

# epd of resource patch
MINERALS, VESPENES = get_resource_epd()

# Maximum number of Mineral Field at a single base location, basically it is 9
MINERAL_NUM = max(len(base_minerals) for base_minerals in MINERALS)
```

## 建造功能的实现
```py
from enum import IntEnum
from eudplib import *
from eud_helper import *


class TaskState(IntEnum):
    CHECK_COND = 0
    ASSIGN_WORKER = 1
    MONITOR_BUILD = 2
    RESET_COLLISION = 3
    RELEASE_WORKER = 4
    BEING_BUILT = 5
    FINISHED = 6


class BuildTask:
    def __init__(self, building, loc):
        self.building = TrgUnit(building)
        self.loc = EncodeLocation(loc)
        self.state = EUDVariable()
        self.cond = EUDLightBool()
        self.cworker = CUnit(EUDVariable(0))
        self.cbuilding = CUnit(EUDVariable(0))
        self.loc_x, self.loc_y = ut.get_loc_xy_static(loc)
        self.loc_pos = ut.get_loc_pos_static(loc)

    def find_worker(self):
        dmin, nearest_worker = set_new_var(0xFFFF, 0)
        for unit in u.unit_group["Protoss Probe"].cploop:
            # order
            unit.move_cp(0x4D // 4)
            EUDContinueIfNot(DeathsX(CurrentPlayer, Exactly, EncodeUnitOrder("Move to Harvest Minerals") << 8, 0, 0xFF00))
            # special mark: unknown0x106
            unit.move_cp(0x106 // 4)
            EUDContinueIfNot(DeathsX(CurrentPlayer, Exactly, 0, 0, 0xFF0000))
            # pos
            unit.move_cp(0x28 // 4)
            x, y = f_posread_cp(0)
            dis = (x - self.loc_x).iabs() + (y - self.loc_y).iabs()
            EUDContinueIf(dis >= dmin)
            nearest_worker << unit.epd
            dmin << dis

        return nearest_worker

    def is_worker_alive(self):
        return self.cworker.order != EncodeUnitOrder("Die")
    
    def is_building_alive(self):
        return self.cbuilding.order != EncodeUnitOrder("Die")

    def reset_worker_status_and_mark(self):
        VProc(self.cworker, [
            SetMemory(0x6509B0, SetTo, 0x4D // 4),  # order
            self.cworker.QueueAddTo(EPD(0x6509B0)), 
        ])
        DoActions([
            # self.cworker.order = "Guard"
            SetDeathsX(CurrentPlayer, SetTo, EncodeUnitOrder("Guard") << 8, 0, 0xFF00), 
            # self.cworker.orderState = 0
            SetMemory(0x6509B0, Add, 0x4E // 4 - 0x4D // 4), 
            SetDeathsX(CurrentPlayer, SetTo, 0, 0, 0xFF0000), 
            # self.cworker.orderQueueTimer = 0
            SetMemory(0x6509B0, Add, 0x85 // 4 - 0x4E // 4), 
            SetDeathsX(CurrentPlayer, SetTo, 0, 0, 0xFF00), 
            # self.cworker.unknown0x106 = True
            SetMemory(0x6509B0, Add, 0x106 // 4 -  0x85 // 4), 
            SetDeathsX(CurrentPlayer, SetTo, 1 << 24, 0, 0xFF0000), 
        ])
        f_setcurpl2cpcache()

    def build_probe(self):
        VProc(self.cworker, [
            SetMemory(0x6509B0, SetTo, 0x98 // 4),  # buildQueue
            self.cworker.QueueAddTo(EPD(0x6509B0)), 
        ])
        DoActions([
            # self.cworker.buildQueue1 = self.building
            SetDeathsX(CurrentPlayer, SetTo, self.building, 0, 0xFFFF), 
            # self.cworker.order = "Build (Probe)"
            SetMemory(0x6509B0, Add, 0x4D // 4 - 0x98 // 4), 
            SetDeathsX(CurrentPlayer, SetTo, EncodeUnitOrder("Build (Probe)") << 8, 0, 0xFF00), 
            # self.cworker.orderTargetPos = self.loc_pos
            SetMemory(0x6509B0, Add, 0x58 // 4 - 0x4D // 4), 
            SetDeaths(CurrentPlayer, SetTo, self.loc_pos, 0), 
            # self.cworker.set_gathering()
            SetMemory(0x6509B0, Add, 0xDC // 4 -  0x58 // 4), 
            SetDeathsX(CurrentPlayer, SetTo, 0x00800000, 0, 0x00800000), 
        ])
        f_setcurpl2cpcache()


    def free_cworker_and_unmark(self):
        self.cworker.unknown0x106 = False
        self.cworker << 0

    def reset_collision(self):
        VProc(self.cworker, [
            SetMemory(0x6509B0, SetTo, 0x4D // 4),  # order
            self.cworker.QueueAddTo(EPD(0x6509B0)), 
        ])
        DoActions([
            # self.cworker.order = "Reset Collision (2 Units)"
            SetDeathsX(CurrentPlayer, SetTo, EncodeUnitOrder("Reset Collision (2 Units)") << 8, 0, 0xFF00), 
            # self.cworker.orderState = 0
            SetMemory(0x6509B0, Add, 0x4E // 4 - 0x4D // 4), 
            SetDeathsX(CurrentPlayer, SetTo, 0, 0, 0xFF0000), 
            # self.cworker.orderQueueTimer = 0
            SetMemory(0x6509B0, Add, 0x85 // 4 - 0x4E // 4), 
            SetDeathsX(CurrentPlayer, SetTo, 0, 0, 0xFF00), 
        ])
        f_setcurpl2cpcache()

    def exec(self):
        EUDSwitch(self.state)
        case_state = EUDSwitchCase()
        if case_state(TaskState.CHECK_COND):
            EUDBreakIfNot(self.cond)
            maybe_unit = self.find_worker()
            if EUDIf()(maybe_unit != 0):
                self.cworker = CUnit.cast(maybe_unit)
                self.state += 1
                self.reset_worker_status_and_mark()
            EUDEndIf()
            EUDBreak()
        if case_state(TaskState.ASSIGN_WORKER):
            if EUDIf()(self.is_worker_alive()):
                self.state += 1
                self.build_probe()
            if EUDElse()():
                self.state << TaskState.CHECK_COND
                self.free_cworker_and_unmark()
            EUDEndIf()
            EUDBreak()
        if case_state(TaskState.MONITOR_BUILD):
            EPDSwitch(self.cworker + 0x4D // 4, 0xFF00)  # order
            case_order = EUDSwitchCase()
            if case_order(EncodeUnitOrder("Build (Probe)") << 8):
                ctarget = self.cworker.orderTarget
                EUDBreakIfNot(ctarget.unitType == self.building)
                self.state += 1
                self.cbuilding = ctarget
                EUDBreak()
            if case_order(EncodeUnitOrder("Die") << 8):
                self.state << TaskState.CHECK_COND
                self.free_cworker_and_unmark()
                EUDBreak()
            if EUDSwitchDefault()():
                self.state << TaskState.ASSIGN_WORKER
                EUDBreak()
            EUDEndSwitch()
            EUDBreak()
        if case_state(TaskState.RESET_COLLISION):
            if EUDIf()(self.is_worker_alive()):
                self.reset_collision()
            if EUDElse()():
                self.free_cworker_and_unmark()
            EUDEndIf()
            self.state += 1
            EUDBreak()
        if case_state(TaskState.RELEASE_WORKER):
            if EUDIf()(self.is_worker_alive()):
                self.cworker.order = "Move to Harvest Minerals"
                cresource = self.cworker.targetResourceUnit
                if EUDIfNot()(cresource.order == EncodeUnitOrder("Die")):
                    self.cworker.orderTarget = cresource
                    self.cworker.orderTargetPos = cresource.pos
                EUDEndIf()
                self.free_cworker_and_unmark()
            EUDEndIf()
            self.state += 1
            EUDBreak()
        if case_state(TaskState.BEING_BUILT):
            if EUDIf()(self.is_building_alive()):
                if EUDIf()(self.cbuilding.is_completed()):
                     self.state += 1
                EUDEndIf()
            if EUDElse()():
                self.state << TaskState.CHECK_COND
                self.cbuilding << 0
            EUDEndIf()
            EUDBreak()
        if case_state(TaskState.FINISHED):
            if EUDIfNot()(self.is_building_alive()):
                self.state << TaskState.CHECK_COND
                self.cbuilding << 0
            EUDEndIf()
            EUDBreak()
        EUDEndSwitch()
```