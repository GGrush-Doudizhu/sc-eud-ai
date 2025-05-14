# 使用纯EUD技术开发星际争霸AI

## 将电脑玩家类型修改为人类
星际争霸的原生电脑AI拥有自主的单位控制逻辑，这些内置行为会严重干扰EUD脚本对单位的精确控制。为了实现EUD对单位精确地操作，​**必须尽可能消除星际争霸原生程序对电脑单位的控制干扰**。解决方案是在游戏开始的时候将目标电脑玩家的player type改为human，使星际争霸将电脑玩家识别为人类玩家。
参考资料：https://armoha.github.io/eud-book/offsets/ActivePlayerStructures.html
```js
function onPluginStart() {
    bwrite_epd(EPD(0X57EEE0) + 9 * player + 0x02, 0, 2);
}
```
