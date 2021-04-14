---
**费曼费曼费曼**  
- 笔记法：
    * 看一小段，复述，即费曼  

**费曼费曼费曼**
---


- qemu其他子系统可以注册地址空间变更事件，address space中的listeners把所有组测信息连接起来
- 平坦化过程，树-> 线性(如数组，也就是实际内存布局)
    * 核心`render_memory_region`
    * 平坦化过程针对叶子节点。region结构(节点表示啥等 p151)
- 内存分派：给定addressspace和地址，快速找到其所在section
- start: p172
- !! https://abelsu7.top/2019/07/07/kvm-memory-virtualization/

- 影子页


- 模电
    * 看课本，为何约等于
    * 看课本的对特性的总结


- 计网选课
- 计网笔记
- os笔记
- 计组笔记

- 计网
    * https://zhuanlan.zhihu.com/p/116436431
    * redo计算题，第二第三次作业

- 一阶微分方程
    * https://www.bilibili.com/video/BV1Xh411Z7gp?from=search&seid=6063876634422318946

- 三次多项式
- 欧拉公式!!!

- qemu
    * 初始化：https://blog.csdn.net/u011364612/article/details/53581501
    * 体系架构：https://blog.csdn.net/sungeshilaoda/article/details/97890633
    * https://blog.csdn.net/huang987246510/article/details/90738137
    * https://blog.csdn.net/bemind1/article/details/99674617?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control
    * https://binhack.readthedocs.io/zh/latest/virtual/qemu/event.html

- cpp面向对象的实现

- 文件描述符
    * https://zhuanlan.zhihu.com/p/109053744

- tty
    * https://segmentfault.com/a/1190000009082089

- qemu monitor使用
    * https://hhb584520.github.io/kvm_blog/2017/02/17/qemu-monitor.html


- 做个backtrace工具
    * 维护记录打开树
    * 退回
        + 仅历史退回
    * 前进
        + 对处理函数闭包，可以利用lsp
        + 具体目标前进，如果是新开，加入维护树
        + 历史前进，如果打开树只有1，且要开的和打开的一样，直接跳
    * 打印记录树，并提供树中任意文件跳转

- lua coroutine

- vmlinux
    * https://blog.csdn.net/nameofcsdn/article/details/78772645

- blog
    * qemu 
    * QOM 

- try
    * https://stackoverflow.com/questions/61537403/how-to-host-image-at-https-user-images-githubusercontent-com-path-filename


- economics
    * https://www.bilibili.com/video/BV1bK4y1576z?p=5

- book
    * anything you want
    * atomic habbit
    * make time
- movie
    * 怦然心动
    * 雨人
    * 未麻的部屋
    * 尽善尽美


- pandoc
    * https://www.youtube.com/watch?v=HllCrrXui2g&t=191s&ab_channel=BrodieRobertson
- fs
    * https://www.cnblogs.com/xiaojiang1025/p/6500557.html
    * BOOK
- 进程id管理
    * https://www.cnblogs.com/hazir/archive/2013/10/03/linux_kernel_pid.html
- 程进切换
    * http://www.wowotech.net/process_management/context-switch-arch.html
- !!!
    * https://www.cnblogs.com/LoyenWang/p/12116570.html
- filesystem
    * https://zhuanlan.zhihu.com/p/106459445

- 虚地址转换
    * https://zhuanlan.zhihu.com/p/65298260
- **mm**:https://zhuanlan.zhihu.com/p/73468738
    * **内存映射**
        + https://zhuanlan.zhihu.com/p/67894878
        + ! https://www.cnblogs.com/xelatex/p/3491305.html
        + ! https://blog.csdn.net/weixin_44277699/article/details/105727534
        + https://www.cnblogs.com/feng9exe/p/6379650.html
        + https://blog.csdn.net/liuyuanqing2010/article/details/6673178
        + https://blog.csdn.net/ludashei2/article/details/91126611
        + https://zhuanlan.zhihu.com/p/116896185
    * buddy in flash: https://zhuanlan.zhihu.com/p/105589621
    * 物理内存管理：https://zhuanlan.zhihu.com/p/68465952
- task struct
    * https://www.jianshu.com/p/691d02380312
- to note
    * https://www.youtube.com/watch?v=ctgheOhe1vE&ab_channel=%E6%9D%8E%E6%B0%B8%E4%B9%90%E8%80%81%E5%B8%88
- ffmpeg
    * https://www.youtube.com/watch?v=HO6oU5oT6uU&ab_channel=MentalOutlaw
- qemu
    * https://www.youtube.com/watch?v=jLRmVNWOrgo&ab_channel=ChrisTitusTech
- learn sed
- Kruskal
- dijkstra, floyd
- OS
    * 动态库
        + https://www.cnblogs.com/aaronLinux/p/6970590.html
        + https://baike.baidu.com/item/.dll/2133451
        + https://blog.csdn.net/eibo51/article/details/50479851
    * scheduling
        + https://zhuanlan.zhihu.com/p/112203100
- 页表
    * https://www.bilibili.com/video/BV1jE411W7e8?p=5
- https://labuladong.gitbook.io/algo/
- KMP!!!!!!
    * **remake** /home/ring/Documents/Note/Major/algorithm/arithmetic.md
    * next 和 nextval
    * 需要好好反思
        + https://www.luogu.com.cn/problem/P3375
- 数据结构图好好耍耍
    * https://www.luogu.com.cn/problem/P4779
        + 小顶堆的创建以及动态调整
    * try: Floyd
- master four
    * https://www.bilibili.com/video/BV1qJ411w7du
    * https://www.bilibili.com/video/BV1u7411z7Sv
    * https://www.bilibili.com/video/BV1454y1X7rk
    * https://www.bilibili.com/video/BV17V411S71E
- 两种跨域的姿势
    * https://www.bilibili.com/video/BV1Kt411E76z
- AQS
    * https://www.bilibili.com/video/BV12K411G7Fg
- 深入浅出写代理
    * https://zhuanlan.zhihu.com/p/28645724
- C模块化开发实例(gcc编译)
    * https://blog.csdn.net/u010476739/article/details/84338147
- 看爆
    * https://space.bilibili.com/31273057/video?tid=0&page=2&keyword=&order=pubdate
    * https://space.bilibili.com/31273057/video?tid=0&page=3&keyword=&order=pubdate
- 看爆！
    * https://zorrozou.github.io/
    * https://zorro.gitbooks.io/poor-zorro-s-linux-book/content/
    * https://zorro.gitbooks.io/poor-zorro-s-linux-book/content/cgroup_linux_cpu_control_group.html
- 计算机网络
    * https://www.bilibili.com/video/BV124411k7uV
- zephyr-nvim nvim-telescope/telescope.nvim
- 防火墙
    * https://www.cnblogs.com/f-ck-need-u/p/7397146.html
- 骏马金龙！！看爆!!
    * https://www.junmajinlong.com/linux/index/
- blog to travel 
    * https://gohalo.me/categories.html#Linux
- 树同构
    * https://pintia.cn/problem-sets/15/problems/711
- 陈海波操作系统
- NOTE about linux创建虚拟硬盘
- epoll和kqueue
    * https://www.cnblogs.com/FG123/p/5256553.html
    * https://www.zhihu.com/question/20122137
- linux内核代码情景分析
- git多人协作笔记
    * rebase


- learn QEMU (kvm)
    * https://zhuanlan.zhihu.com/p/48664113
    * https://wiki.archlinux.org/index.php/QEMU
- 算法
    * 插入区间
        + https://leetcode-cn.com/problems/insert-interval/submissions/
    * 单词拆分 II
        + https://leetcode-cn.com/problems/word-break-ii/
    * O(1) 时间插入、删除和获取随机元素 - 允许重复
        + https://leetcode-cn.com/problems/insert-delete-getrandom-o1-duplicates-allowed/solution/
    * 求叶子到节点数字之和
        + https://leetcode-cn.com/problems/sum-root-to-leaf-numbers/submissions/
    * 树中距离之和
        + https://leetcode-cn.com/problems/sum-of-distances-in-tree/
    * 秋叶收藏集
        * https://leetcode-cn.com/problems/UlBDOe/submissions/
    * 冗余连接II
        + https://leetcode-cn.com/problems/redundant-connection-ii/
    * 解数独
        + https://leetcode-cn.com/problems/sudoku-solver/
    * 有效状态自动机
        + https://leetcode-cn.com/problems/n-queens/submissions/
    * 监控二叉树
       + https://leetcode-cn.com/problems/binary-tree-cameras/
- vimspector
    * https://github.com/puremourning/vimspector
- linux from scratch
    * http://linuxfromscratch.org/lfs/view/stable/chapter04/settingenvironment.html
