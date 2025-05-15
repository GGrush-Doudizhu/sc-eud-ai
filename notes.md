# 使用纯EUD技术开发星际争霸AI

## 将电脑玩家类型修改为人类
星际争霸的原生电脑AI拥有自主的单位控制逻辑，这些内置行为会严重干扰EUD脚本对单位的控制。为了精准有效地实现EUD对单位的操作，​**必须尽可能消除星际争霸原生程序对电脑单位的控制干扰**。解决方案是将目标电脑玩家的player type改为human，使星际争霸将电脑玩家识别为人类玩家。

修改地址：https://armoha.github.io/eud-book/offsets/ActivePlayerStructures.html
```js
function onPluginStart() {
    bwrite_epd(EPD(0X57EEE0) + 9 * player + 0x02, 0, 2);
}
```

## 矿区资源初始化
星际争霸AI一个非常首要的任务就是命令工人采集资源和扩张，现阶段为了简单起见，我固定AI在地图上的出生点位，这样我就能够简单高效地进行初始化。

首先需要利用SCMD在地图的矿区的基地处放置location, 在示例中，我按照以下命名格式放置了4个location, 意味着示例AI最多支持4片矿。
```python
# config.py
BASE_NAMES = [f"base_{i}" for i in range(4)]
```
以下获取资源块epd地址的方法是通过读取单位在chk结构中的索引计算获得，这样做的好处是在编译期完成，使得输出地图的触发数量和文件大小都比较小。缺点是可能会由于空单位导致所有资源块的epd地址产生系统偏移，所以更保险的做法是在游戏开始的时候再进行初始化。
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