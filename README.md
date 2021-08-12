# 手动修复微信8.0.9iOS客户端中WASM能力异常指引

## 概述

------

在微信客户端(iOS 8.0.9)中小游戏使用WASM时有概率对部分的WASM库调用发生报错，例如目前Cocos Creator3中使用WASM版本的Ammo物理引擎是将会发生异常。

该问题在微信开发者社区中其他人的提问地址为：https://developers.weixin.qq.com/community/minigame/doc/0002a0271c4d982fa07cdb3795b400

根据官方的说明是指JS胶水层代码在对同一个函数的导出做了次数限制，超过3次函数将没有实际的指针，官方将在后续的客户端版本修复这一问题，但修复前需要进行手动适配，经过本人的实际操作，已成功修复，因此本文将总结出这一问题的修复指引。



## 修复说明

------



