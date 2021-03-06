# 第四节 volatility 源码整体架构分析下
上一节我们看到了插件的初始化操作，这里同样以插件linux_pslist(源码文件为pslist)开讲，构造函数的第一个参数就是为配置文件，这个文件保存在哪呢，我们简单看linux_pslist类中没有这个字段啊，如图1所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/1cof.png)
我们继续看他调用的父类构造函数，如图2所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/2cof.png)
发现原来在父类，再在上面下断动态调试下，看下调用堆栈，我们分析没错如图3所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/3cof.png)
那插件是怎么调用起来的呢，我们回到最上面的入口，看如下代码：
```
 try:
        if module in cmds.keys():
            command = cmds[module](config)

            ## Register the help cb from the command itself
            config.set_help_hook(obj.Curry(command_help, command))
            config.parse_options()

            if not config.LOCATION:
                debug.error("Please specify a location (-l) or filename (-f)")

            command.execute()
```
我们发现它是找到对应的插件类执行execute函数，那这个execute函数不要想了肯定是一个父类的函数(多态)，我们翻command源码看看，如图3所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/comd1.png)
发现execute函数最终调用的是上面没有实现的calculate函数(虚拟函数),到这里肯定能猜到所有插件的入口就是calculate函数的实现，下断看下调用堆栈如图4所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/comd2.png)
到这里并没有结束，还有一个很重要的事情就是怎么获取profile的，猜想这个一般应该放在配置文件中，看看有没有如图5所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/5.png)
发现什么也没有，那放到哪呢，没什么好的分析思路，继续从插件入口下手，一开始就调用一个set_plugin_members函数，进去看看，调用了 obj_ref.addr_space = utils.load_as(obj_ref._config)，继续进入，又发现了熟悉的地方是遍历BaseAddressSpace子类操作，如图6所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/6class.png)
继续看BaseAddressSpace类发现了一行代码self._set_profile(config.PROFILE),如图7所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/7file.png)
但是还是有点迷，没关系下断动态调试下，发现就是set我们传进去(参数)的如图8所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/8set.png)
返回往上看看所有的子类，发现是一些镜像操作相关的子类，如图9所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/9.png)
现在又找到了一点线索，那么传的LinuxUbuntu1404x64是用来干嘛的呢，我们继续看_set_profile函数如图10所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/10set.png)
又是一个获取子类的操作，下断看看有哪些子类，如图11所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/11.png)
原来每个profile又是一个类，这里我又有疑问了这个类怎么生成的，因为LinuxUbuntu1404x64名字是我自己取的啊，好了又是到了怎么找这个类的问题，好办下断到找到的地方看看哪个类如图12所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/12.png)
发现FileAddressSpace类，进入standard源码看看，没什么线索，我找了好久,后面返回到最前面的main函数执行了一句registry.PluginImporter()，然后一路跟到了LinuxProfileFactory，原来他是一个工厂函数遍历所有的profile获取文件名创建一个类出来，下断到这个函数看堆栈如图13所示。
![avatar](https://github.com/haidragon/MemoryForensics/blob/master/pages/Chapter1/page4/images/13.png)
那他们是怎么联系起来的呢，我们捋一下，首先我们跟到utils.py文件的load_as函数，然后我们也知道找的是FileAddressSpace类，然而构造函数又调用了BaseAddressSpace类，这个类构造函数里面有个函数_set_profile，好了又回到了原点，我们看_set_profile函数实现，看到了调用registry.get_plugin_classes函数，咦我们刚刚不是回到main函数也看到了registry的PluginImporter函数吗，到这里应该一目了然了，volatility分析先到这里了。

