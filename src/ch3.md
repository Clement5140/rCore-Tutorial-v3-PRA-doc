# 页面置换算法的实现

## 局部页面置换算法

### FIFO页面置换算法

FIFO页面置换算法实现非常简单，只需要给每一个用户程序记录一个物理页面队列即可，在用户访问申请的内存并触发缺页异常分配物理页面时将对应PPN加入队列的队尾，在需要换出物理页面时将队首的物理页面换出即可。