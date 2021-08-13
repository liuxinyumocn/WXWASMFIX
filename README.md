# 手动修复微信8.0.9iOS客户端中WASM能力异常指引

## 概述

------

在微信客户端(iOS 8.0.9)中小游戏使用WASM时有概率对部分的WASM库调用发生报错，例如目前Cocos Creator3中使用WASM版本的Ammo物理引擎是将会发生异常。

该问题在微信开发者社区中其他人的提问地址为：https://developers.weixin.qq.com/community/minigame/doc/0002a0271c4d982fa07cdb3795b400

根据官方的说明是指JS胶水层代码在对同一个函数的导出做了次数限制，超过3次函数将没有实际的指针，官方将在后续的客户端版本修复这一问题，但修复前需要进行手动适配，经过本人的实际操作，已成功修复，因此本文将总结出这一问题的修复指引。



## 修复说明

------

下图是官方给出的解决思路

<img src="https://raw.githubusercontent.com/liuxinyumocn/WXWASMFIX/master/img/WX20210812-154119%402x.png" alt="image-20210410210434454" />



简单地说，我们解决过程是：

1. 在模拟器中打印出当前的WASM胶水层代码的句柄列表 **list1**；
2. 再试图导出iOS微信客户端中的WASM胶水层代码的句柄列表 **list2**；
3. 此时统计list2中缺失的（不在list1）中的函数名列表 **list3**；
4. 回到list1中可以看到不同函数名其对应的实际指针ID（函数），将list3的Key健替换成list2中存在且指针相同的导出名，形成替换 **map1**；
5. 根据map1完成对胶水层的更替，至此问题解决。

通常来说list的数量很大，我们需要写一个简单的脚本对其进行批量处理，为了方便大家的使用，我写了一个在线的替换工具，需要大家根据指引填入相应的信息，在线工具源代码在本仓库目录 /tool/ 内。**由于修复是根据WASM在iOS真机上的实际表现有关，因此工具并不能一键完成转换，请参考下方实践完成这一修复工作。**



## 实践修复

------

本文给出Ammo.wasm的手动修复实践案例。

修复前的 Demo 源代码在本仓库目录 [/demo/prev/](https://github.com/liuxinyumocn/WXWASMFIX/tree/master/demo/prev) 内

修复后的 Demo 源代码在本仓库目录 [/demo/fixed/](https://github.com/liuxinyumocn/WXWASMFIX/tree/master/demo/fixed) 内

1. 使用微信开发者工具打开 prev 代码包，模拟器与Android真机运行正常，iOS8.0.9客户端运行时控制台报如下错误：

   <img src="/Users/nebulaliu/Library/Application Support/typora-user-images/image-20210812170326564.png" alt="image-20210812170326564" width="200" />

   报错内容为：未定义的对象

2. 打开wasm的胶水层代码，本案例中为 wx.ammo.wasm.js ，在 [17行](https://github.com/liuxinyumocn/WXWASMFIX/blob/master/demo/prev/ammo_wasm/wx.ammo.wasm.js#L17) 位置进行对 b.asm 的结构进行打印。

   ```javascript
   var b;
   	setTimeout(()=>{	//使用setTimeout 是确保对象b被实例并完全初始化
   		console.log(b.asm);	//打印 b.asm 本案例是 var b 其他案例自行观察代码
   		var description = "";
   		for (var i in b.asm) {	//将对象的函数已经对应的实际指针ID转换成字符串
   			var property = b.asm[i];
   			description += i + " = " + property + "\n";
   		}
   		wx.setClipboardData({	//利用剪贴板完整导出
   			data: description,
   		})
   	},1000);
   b||(b=typeof Ammo !== 'undefined' ? Ammo : {}); // others...
   ```

   打印结果如下：

   <img src="/Users/nebulaliu/Library/Application Support/typora-user-images/image-20210812171431605.png" alt="image-20210812171431605" width="300" />

   其中 $、$a...就是导出函数名，后面的 f 27() 、f 49() 是导出的函数指向的实际指针，报错的原因就是同一个实际指针被导出多个函数名（意味着右边存在超过3个以上相同的实际指针），多出的导出名在iOS中被忽略掉，因此导致了报错。

3. 分别在模拟器与iOS真机中运行该脚本，获得粘贴板内的文本内容。可以看到模拟器中有855个导出函数，而iOS真机只有600个导出函数。接下来我们需要借助脚本进行批量扫描，得到需要替换的序列。使用浏览器打开本仓库 /tool/index.html 替换工具，将模拟器给出的粘贴板内容复制到第一个文本框内，将真机给出的粘贴板内容复制到第二个文本框内，点击识别。如下图所示。

   <img src="/Users/nebulaliu/Library/Application Support/typora-user-images/image-20210813103450059.png" alt="image-20210813103450059" width="400" />

4. 将胶水层代码全部复制粘贴至替换工具中的第三个文本框内，本文中也就是 wx.ammo.wasm.js 文件的源代码。

5. 观察一下替换的部分的代码规则，以本文为例，需要将

   ```javascript
   =function(){return b.asm.XXXX.apply(null,arguments)}
   ```

   更替为

   ```javascript
   =function(){return b.asm.YYYY.apply(null,arguments)}
   ```

   因此在工具的第四个文本框定义替换规则：

   使用 ${Fun} 来声明更替的函数部位。

   以上述更替案例为例，则第四个文本框应该输入：

   =function(){return b.asm.**${fun}**.apply(null,arguments)}

6. 最后点击替换，最后一个文本框即是替换的源代码，覆盖原胶水层代码，删除调试时的注释，问题到此解决。





