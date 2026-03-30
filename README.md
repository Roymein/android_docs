# Framework 关键流程（基于 frameworks/base 当前代码）

面向刚开始学习 Framework 开发的新手，这里把三个最常见的“跨进程 + system_server 调度 + app 进程回调”的流程，按源码主线梳理成可对照的流程图与要点：

- startActivity：应用启动/界面启动链路（Activity 启动、进程冷启动/热启动、客户端事务）
- startService / startForegroundService：Service 启动与进入 app 进程回调链路（含进程拉起）
- Broadcast：广播发送、队列调度、分发到动态/静态接收者（含超时/ANR 的关键点）

目录结构：

- 01_startActivity.md
- 02_startService.md
- 03_broadcast.md

阅读建议（新手路线）：

1. 先看每篇顶部的“你需要先记住的 5 个概念”
2. 再看“主流程图（Mermaid）”，先抓住大骨架（跨进程边界与关键类）
3. 最后顺着“源码定位清单”在 IDE 里点进，把每一步对应到真实方法
4. 进阶：看“逐函数状态机”小节，把关键函数内部的子阶段按 if/return/消息流逐段对上源码

