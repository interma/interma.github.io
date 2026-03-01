---
layout: post
title:  "使用AI开发一个复古像素游戏！"
date:   2026-02-27 19:12:03 +0800
permalink: /game/pixel_game.html
tags: [game]
---

我和呦呦都很爱打游戏，在这个寒假快开学的这几天，突发灵感：是否能用Ai来开发我们自己的游戏呢？

于是问了一下GPT：
>  这事儿太棒了👏——用 AI + 像素横版游戏给儿子做个“爸爸牌 FC 游戏”，这本身就已经是满分项目了。而且你已经发现 Copilot 写代码很强，那我们可以把 ChatGPT 用在“架构 / 设计 /拆任务”，Copilot 用在“写具体代码”，这是目前最舒服的组合。

> 推荐技术路线：一个可运行、可玩的、像 FC 横版一样的小游戏 Demo。而非一个引擎深度的商业游戏。我强烈推荐这一套：

| 层   | 选择                         | 原因                           |
|------|------------------------------|--------------------------------|
| 语言 | TypeScript / JavaScript      | Copilot 最强，调试快            |
| 引擎 | Phaser 3                     | 2D 神器，像素友好，学习曲线低   |
| 运行 | 浏览器                       | Mac 上零门槛，给孩子直接点开    |
| IDE  | VS Code + Copilot            | 你已经验证好用                 |
| 资源 | 像素 PNG + tilemap           | 经典 FC 风格                  |


正好最近还买了Copilot会员，说干就干。
<img src="{{ site.url }}/assets/images/ai_game.jpg" alt="父子pair" style="display:block;max-width:600px;width:70%;height:auto;" />


于是我们的一个基于Phaser3和 TypeScript开发的复古像素游戏诞生了：
- [https://github.com/interma/yoyo-pixel-game](https://github.com/interma/yoyo-pixel-game)
- 部署到了vercal，直接可以访问：[https://github.com/interma/yoyo-pixel-game](https://yoyo-pixel.vercel.app/)

看着我们自己亲手开发的游戏，呦呦很高兴，希望他以后能喜欢上开发，专注的做一些事情。

最后自己关于AI Coding的一些感悟：
* 大多数情况下，可以跳出技术细节了：AI变成了一个需求编译器
* 人需要：
  * 知道能做什么
  * 给出约束（比如一次文件IO），控制边界（比如函数原型）
* AI开发的代码质量还是一般，一坨坨可work的代码更危险，所以体现出控制边界的重要性
