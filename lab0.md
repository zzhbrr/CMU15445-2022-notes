很早就想学数据库CMU15445课程了，在寒假空闲的时候刚好翻到了15445在2022年的课程，而且还提供线上的评测！马上开干！

所用的资料：

1. 课程官网：https://15445.courses.cs.cmu.edu/fall2022/
2. 课程视频：[(8) CMU Intro to Database Systems (15-445/645 - Fall 2022) - YouTube](https://www.youtube.com/playlist?list=PLSE8ODhjZXjaKScG3l0nuOiDTTqpfnWFf)
3. 评测网站：[Your Courses | Gradescope](https://www.gradescope.com/)，课程代码是PXWVR5，学校要选择CMU
4. github仓库：[cmu-db/bustub: The BusTub Relational Database Management System (Educational) (github.com)](https://github.com/cmu-db/bustub)
5. 交流平台：https://discord.gg/YF7dMCg

我将会同步课程笔记，和lab的题解和注意事项（**不会公开代码！**）。

### 配置环境

因为对c++不熟悉，配环境加摆烂就过了一整天。

我使用了win11里wsl2，ubuntu20.04（虽然课程说是不支持wsl，但我觉得问题不大），环境配置参考github仓库README即可。

除此之外按照其他人的推荐，我在vscode配置了clangd+lldb替代原先用的Microsoft C/C++ + gdb进行开发，过程可以参考[这篇博客]([几乎无痛的VSCode+clangd+lldb+cmake配置C/C++开发环境指南 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/566365173))。



### Project0

project0主要考验C++语法，让写一个并发Trie树，实现key-value存储。这个project让我意识到我还不会C++（从来没见过这么大的C++项目），于是边学边学用了一天半完成。

主要涉及到了以下C++知识：

* 类、继承、多态
* 模板
* 移动语义，如左值右值、std::move
* 智能指针，如unique_ptr
* stl基础，如string

代码需要满足编码规范（如必须使用尾置返回类型），不然提交会判0分，并且不能有内存泄露。

开发debug时不推荐一堆printf，而是用`common/logger.h`中的log宏函数，或者使用gdb或lldb进行debug。

具体实现没有什么好讲的，最大的困难就是在于不熟悉C++的语法，我反思。