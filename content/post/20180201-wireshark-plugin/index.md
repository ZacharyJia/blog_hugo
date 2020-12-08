---
title: 自己动手编写Wireshark Lua插件解析自定义协议
slug: wireshark-plugin-using-lua
date: 2018-02-01
categories:
    - 开发
tags:
    - wireshark
    - 网络
---

搞网络的对于 Wireshark 这个抓包工具应该非常熟悉了，在抓包分析的时候非常好用，很大的一个原因就是 Wireshark 内置了大量的协议解析插件，基本上你叫得上来的协议，Wireshark都能给你解析出来。

但是啥事儿都有个万一，特别是像我们这种搞网络协议开发、修改的，经常就会遇到各种奇葩的网络协议，或者是自己拍脑瓜设计出来的网络协议，在调试的时候Wireshark不能正确解析，一个字节一个字节对着查那可真是看得眼都要瞎了。

<!--more-->

最近就遇上这么一个协议，其实也算不上协议，就是 Ethernet in UDP的封装：因为某些特殊的原因，需要将某个网口接收到的以太网数据帧，全部打包到UDP当中，作为UDP的PayLoad去解析。比较麻烦的就是，还非要把UDP PayLoad的前两个字节设置成自己的两个特殊字段，从第三个字段开始才是以太网数据帧。

因为要开发相应的程序，需要反复调试对比，所以肯定就得动用Wireshark去分析，但是Wireshark又默认不支持这样的协议，所以对比起来很麻烦。网上查了一下相关的资料，发现可以用C去写插件，然后编译成链接库给Wireshark用，但是尝试了一会儿之后，发现实在是太麻烦了，编译环境搞了半天都搞不定。遂弃之，研究使用Lua脚本语言进行解析。


# 0x01 基础知识

Lua是一种轻量级的脚本语言，解释执行，不需要编译器之类的。 Lua的基本语法可以参考 [官网](http://www.lua.org/start.html) 或者 [菜鸟教程](http://www.runoob.com/lua/lua-tutorial.html)。
Wireshark内置了对Lua脚本的支持，可以直接编写Lua脚本，无需配置额外的环境，使用起来还是非常方便的。 [Wireshark Developer's Guide]里的第10章和第11章都是关于Lua支持的文档，有需要的话可以详细查阅。

使用Lua编写Wireshark协议解析插件，有几个比较重要的概念:
1. Dissector，中文直译是解剖器，就是用来解析包的类，我们最终要编写的，也是一个Dissector。
2. DissectorTable，解析器表是Wireshark中解析器的组织形式，是某一种协议的子解析器的一个列表，其作用是把所有的解析器组织成一种树状结构，便于Wireshark在解析包的时候自动选择对应的解析器。例如TCP协议的子解析器 http, smtp, sip等都被加在了"tcp.port"这个解析器表中，可以根据抓到的包的不同的tcp端口号，自动选择对应的解析器。


# 0x02 一个例子
我们先来看一下上面说的那个封装格式的脚本例子：（`--`后面的是注释）
```lua
do
    --协议名称为DT，在Packet Details窗格显示为Nselab.Zachary DT
    local p_DT = Proto("DT","Nselab.Zachary DT")
    --协议的各个字段
    local f_identifier = ProtoField.uint8("DT.identifier","Identifier", base.HEX)
    --这里的base是显示的时候的进制，详细可参考https://www.wireshark.org/docs/wsdg_html_chunked/lua_module_Proto.html#lua_class_ProtoField
    local f_speed = ProtoField.uint8("DT.speed", "Speed", base.HEX)

    --这里把DT协议的全部字段都加到p_DT这个变量的fields字段里
    p_DT.fields = {f_identifier, f_speed}
    
    --这里是获取data这个解析器
    local data_dis = Dissector.get("data")
    
    local function DT_dissector(buf,pkt,root)
        local buf_len = buf:len();
        --先检查报文长度，太短的不是我的协议
        if buf_len < 16 then return false end

        --验证一下identifier这个字段是不是0x12,如果不是的话，认为不是我要解析的packet
        local v_identifier = buf(0, 1)
        if (v_identifier:uint() ~= 0x12)
        then return false end

        --取出其他字段的值
        local v_speed = buf(1, 1)
        
        --现在知道是我的协议了，放心大胆添加Packet Details
        local t = root:add(p_DT,buf)
        --在Packet List窗格的Protocol列可以展示出协议的名称
        pkt.cols.protocol = "DT"
        --这里是把对应的字段的值填写正确，只有t:add过的才会显示在Packet Details信息里. 所以在之前定义fields的时候要把所有可能出现的都写上，但是实际解析的时候，如果某些字段没出现，就不要在这里add
        t:add(f_identifier,v_identifier)
        t:add(f_speed,v_speed)
        
        return true
    end
    
    --这段代码是目的Packet符合条件时，被Wireshark自动调用的，是p_DT的成员方法
    function p_DT.dissector(buf,pkt,root) 
        if DT_dissector(buf,pkt,root) then
            --valid DT diagram
        else
            --data这个dissector几乎是必不可少的；当发现不是我的协议时，就应该调用data
            data_dis:call(buf,pkt,root)
        end
    end
    
    local udp_encap_table = DissectorTable.get("udp.port")
    --因为我们的DT协议的接受端口肯定是50002，所以这里只需要添加到"udp.port"这个DissectorTable里，并且指定值为50002即可。
    udp_encap_table:add(50002, p_DT)
end
```

将其保存为 `packet-dt.lua` 文件
上面这段代码已经看起来非常清楚了，如果是解析一般的自定义协议，上边的代码基本上够用了。

0x03 Lua插件的启用
想要启用Lua插件，首先要确认你的Wireshark版本是支持Lua的（Windows版本默认应该都是启用支持了的）。可以通过【帮助】-【关于】窗口确认：
![lua支持.png][1]

如果是这种With Lua的，应该就是可以的了。

然后去文件夹选项卡，找到Global Configuration文件夹的位置:

![global-configuration.png][2]

在这个文件夹里找到init.lua文件，使用文本文件编辑器打开它，在文件的最后添加：
```
dofile("c:\\path\\to\\packet-dt.lua")
```
填写好正确的packet-dt.lua所在的位置，保存文件就可以了。
然后重新启动Wireshark或者点击【分析】-【重新载入Lua插件】，就可以启用你自己的lua插件了。

# 0x04 测试与调试

测试的话，直接抓包就可以看到对应的包的协议列变成了DT，并且Packet详情窗口里可以看到对应的协议行了。
如果出现问题，Wireshark会直接在对应位置报错，按照报错信息修改packet-dt.lua文件，保存后重新载入Lua插件就可以。

# 0x05 高级一点的玩法
虽然我们实现了基本的包解析功能，但是其实我之前说过，我们的UDP的PayLoad里封装的其实是以太网包，能不能让Wireshark在我们的插件执行完之后，继续按照以太网格式解析其他部分呢？肯定是可以的。

这里，我们只需要重新构造一下需要继续解析的数据，然后获取出一个以太网解析器就可以继续做下去了：
```lua
        local raw_data = buf(2, buf:len() - 2)
        Dissector.get("eth_maybefcs"):call(raw_data:tvb(), pkt, root)
```

把这段添加在刚才的 `t:add(f_speed, v_speed)`之后，就可以了。
这里要注意两点，第一点是获取的解析器名称应该是 `eth_maybefcs`，这个坑了我很久，因为DissectorTable里写的也是eth，但是提示找不到。网上查了很久之后才发现应该用这个名字去获取，意思是可能带有fcs的eth帧。。。

第二点是raw_data需要调用一下tvb()函数，不然会提示你这个是userdata，不能使用。tvb的全称应该是Testy Virtual Buffer，用来存储Packet buffer的，要处理必须先转成这个。

这样你测试的时候，就可以看到，Packet Details窗口里的"Nselab.Zachary DT"栏的下面，又出现了Ethernet、IP等，这就是内部的数据解析出来的结果。

当然，你也会发现列表的协议栏又被改成了ARP、ICMP等内部协议的名称了，这是因为调用`eth_maybefcs`解析器的时候，这些解析器又会给协议栏赋值，覆盖掉我们之前写的`DT`。为了和其他的区分，我们还可以玩得更骚气一点，在上面的代码之后加上：
```
        pkt.cols.protocol:append("-DT")
```
这句话的意思就是不管协议栏被改成了啥，我都在后面加上`-DT`，这样ARP、ICMP等就会变成 `ARP-DT`、`ICMP-DT`了，一眼就可以跟那些平淡无奇的ARP和ICMP区分出来。


# 0x06 结束语
总的来说，使用Lua来编写Wireshark的协议解析插件还是比较简单的，相对于使用C语言，配置、开发、调试应该都方便了不少。当然，如果要详细开发，肯定还是要多看看官方的开发文档：[Wireshark Developer's Guide](https://www.wireshark.org/docs/wsdg_html_chunked/).

  [1]: https://www.zacharyjia.me/usr/uploads/2018/02/3733043128.png
  [2]: https://www.zacharyjia.me/usr/uploads/2018/02/569570693.png