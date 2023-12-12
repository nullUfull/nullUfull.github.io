# 踩坑日记


## 12.9

背景：有一个面板Fragment，内部有一个ViewPager2，ViewPager2下面又有3个Fragment
问题：当我打开这个面板，我想选中第二个Fragment，于是调用ViewPager的setCurrentItem(1)接口，会发现第三个Fragment的onCreateView被调用了，意味着Fragment被提前加载了。

解决方案：试着将接口调用改成setCurrentItem(1,false)，这意味着我不使用smoothScroll的方式，这时候前面说的问题消失了。但是并不知道原理是什么？

