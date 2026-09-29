# 在 WebGAL 里加入连连看

## 结论

可以加。连连看应当做成一条会暂停剧本的命令，玩完把结果写进游戏变量，再让原来的视觉小说脚本继续走。

WebGAL 的运行时是「读一句脚本、改舞台、播演出、等玩家点下一步」。它没有第二种游戏主循环，也没有插件槽可以把一整款小游戏换进去。现成的暂停点只有选项（`choose`）和填空（`getUserInput`）。连连看沿用同一条路：命令挡住下一步，自己画棋盘，结束时写入变量并调用 `nextSentence()`。

Forge 的生成管线现在不会写出这种命令。引擎先能跑，校验和语法合同再认这条命令，生成出来的剧本才能用它。

## 引擎现在怎么跑

一条 WebGAL 语句的路径是：

1. `webgal-parser` 按 `SCRIPT_TAG_MAP` 把文本解析成 `commandType`。注册表在 `src/Core/parser/sceneParser.ts`，解析器构造时吃的就是这份配置。
2. `scriptExecutor` 执行当前句。后续句子要用的结果必须写进 `calculationStageState`，不能只留在动画回调里。
3. 命令返回 `IPerform`。`blockingNext` 挡住点击前进，`blockingAuto` 挡住自动播放，`blockingStateCalculation` 挡住快进把后面的句子先算掉。
4. 只有选项和填空把第三个开关打开。它们都把 React 界面挂到 `#chooseContainer`，结束时卸载并调用 `nextSentence()`。

点击前进的透明层是 `#FullScreenClick`，`z-index: 12`。立绘和背景所在的 Pixi 画布是 `z-index: 5`。选项界面自己的根节点是 `z-index: 13`，所以能盖住点击层。知识点热区 `HotspotLayer` 是 `z-index: 14`，但它不暂停脚本，点一下只弹说明。

因此小游戏界面必须盖在点击层上面，并自己吃掉点击。画在 Pixi 舞台上时，点击会先落到透明层上，棋盘收不到事件。

## 连连看怎么接

建议一条通用命令，第一种游戏是连连看：

```webgal
miniGame:lianliankan -cols=8 -rows=6 -time=60 -result=lk_result -score=lk_score;
changeScene:win.txt -when=lk_result=="clear";
changeScene:timeout.txt -when=lk_result=="timeout";
:时间到了，棋盘还没清完。;
```

`-when` 是这个仓库里可用的分支方式。`commandType.if` 还在枚举里，但 `sceneParser.ts` 里对应注册是注释掉的，不要依赖 `if` 语句。

命令执行时做四件事：

1. 返回一个非保持演出，三个阻塞开关都为真。快进预览时不要挂界面，直接写入默认结果，和 `getUserInput` 在 `isFastPreview` 时写入 `defaultValue` 一样。否则预览会停在这条命令上，或者后面的 `-when` 读不到结果。
2. 在高于 `z-index: 12` 的层里渲染棋盘。根节点要铺满舞台并阻止事件继续冒泡。进行中可以关掉对话框，避免文字框挡住格子。
3. 棋盘、配对、两折以内寻路、倒计时放在独立模块里。这些中间状态不要写进舞台演算态。
4. 清空、超时或放弃时，用 `stageStateManager.setStageVarAndCommit` 写入 `lk_result` 和 `lk_score`，卸载演出，再 `nextSentence()`。`GameVar` 的值是字符串、布尔、数字，或它们的数组。胜负和分数够用。

脚本侧只关心结果。背景、立绘、BGM 保持进入小游戏之前的状态，小游戏只是盖在上面的一层。

## 不要走的两条路

`pixiPerform` 用来放雨、雪这类保持型特效。它不阻塞下一步，也不把点击交给特效容器。连连看不是这种演出。

`HotspotLayer` 已经证明舞台上可以叠一块自己的 React。它按背景图显示热点，不读脚本参数，不写变量，也不暂停剧情。把它扩成连连看会和剧本时钟缠在一起。

## 三个必须单独处理的点

存档记的是当前句号和舞台快照，不记 React 组件里的棋盘。`choose` 执行后句号已经前进，进行中的演出留在 `PerformList` 里，读档时 `restorePerform` 会把那句再跑一遍。连连看会同样被重新打开，但格子布局会重新生成。第一版接受「读档重开这一局」。若要接着下，得把棋盘序列化进舞台状态，`ISetGameVar` 只接受标量，棋盘不适合塞进普通变量。

快进和实时预览会丢掉还没提交的普通演出，也不会跑 `startFunction`。默认结果必须在命令函数里同步写好，不能等界面关闭。

Forge 后端的 `_WEBGAL_CONTROL_COMMANDS`（`webgal_backend/scene_validation.py`）和 `webgal_backend/contracts/syntax.md` 只认识现在的视觉小说命令。引擎里新加的命令，生成剧本在校验阶段仍会被当成非法语句。要让生成流程写出连连看，还要改语法合同和校验白名单。

## 落地时要动的地方

引擎：

- `src/Core/controller/scene/sceneInterface.ts`：在 `commandType` 末尾加 `miniGame`。枚举是数字，插在中间会改掉已有命令的值。
- `src/Core/parser/sceneParser.ts`：注册 `miniGame`。解析器使用这份 `SCRIPT_CONFIG`，不需要另做一套命令表。
- 新目录 `src/Core/gameScripts/miniGame/`：命令、连连看规则、棋盘界面。
- 舞台样式：小游戏层高于 `#FullScreenClick`。

生成管线，等手写脚本验证通过后再改：

- `webgal_backend/contracts/syntax.md`
- `webgal_backend/scene_validation.py` 的命令白名单

第一版只做一种棋盘、倒计时、清空或超时、两个变量回写，以及预览时的默认结果。读档重开，不做中途续局，也不让模型自动编关卡。
