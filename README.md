## Dell7050 OpenCore配置
硬软件信息：Dell.OptiPlex.7050-OpenCore.0.6.4-macOS.Big.Sur.11.1

|   条目   |                                        内容                                         |
| -------- | ----------------------------------------------------------------------------------- |
| Model    | Dell OptiPlex 7050                                                                  |
| CPU      | i5-7500 CPU                                                                         |
| GPU      | Intel(R) HD Graphics 630                                                            |
| Audio    | ALC255                                                                              |
| Disk     |       PC401 NVMe SK hynix 256GB                                                                              |
| OpenCore | OpenCore 0.6.4 Debug                                                                |
| Dmg      | Install.macOS.Big.Sur.11.1(20C69).with.OpenCore.0.6.4.and.Clover.r5127.and.WEPE.dmg |

### 工作情况
- 正常工作
    - 网卡正常
    - 显卡UHD630正常，显存正常
    - USB可用（可以尝试USB端口定制）
- 不正常工作
    - 声卡无法读取，定制失败，**无法输出声音**
    - HDMI输出不支持2K，暂**只能使用1080P的显示器**
    - 如果~~系统安装界面是英文~~，安装好后进系统设置语言即可

### 处理问题
#### 集显HD630显存7MB问题
这种问题一般是显卡仿冒id设置错误，与自己的机型不符导致。安装系统可以将仿冒id `AAPL,ig-platform-id` 设置成`12345678`（仅使用这一个键值），可以提高你的系统安装成功率。

##### 准备工具
- `Hackintool`，下载地址：https://github.com/headkaze/Hackintool/releases
- OCC，下载地址：https://mackie100projects.altervista.org/download-opencore-configurator/ 主要这个OCC的版本一定要和OC的版本一致。

##### 找到显卡匹配的仿冒ID
[WhatwverGreenIntelHD](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.IntelHD.cn.md)
打开Hackintool工具，在 系统 -> 系统 -> IGPU 中查看**设备ID**，并将其复制，例如这儿是：`0x59128086`
![](_v_images/20201228114058948_17034.png)
参考 [通用Intel英特尔核显驱动完整教程（英特尔集成显卡与核显仿冒ID速查表）](http://imacos.top/2020/09/03/2216/) 搜索这个值，如果这个设备 ID为空，则根据显卡型号进行搜索
根据表中的**设备ID**，查到 **帧缓冲区** 的值，即为**仿冒 ID**的值，这儿有`0x59120000`和`0x59120003`这两个对应的值
![](_v_images/20201228091327015_26705.png =850x)

##### 启动参数尝试仿冒ID
打开 config.plist 文件，在随机访问储存器设置`boot-args`中添加`igfxframe=0x59120000`

![](_v_images/20201228091521166_8599.png =1000x)

保存重启，看显卡是否正常驱动，如果没有驱动上或进不去系统，再次更换**仿冒ID** 的值

![](_v_images/20201228114016007_15397.png)

##### 帧缓冲区打补丁
打开Hackintool工具

- 引导 -> 引导工具 中确认名称为OpenCore；
![](_v_images/20201228114123366_8335.png =600x)
- 应用补丁 -> 信息 ，将下方的CPU架构选择自己平台的，**平台ID**选择刚确定下来的显卡仿冒ID：`0x59120000`
![](_v_images/20201228114159261_9429.png =600x)
- 应用补丁 -> 应用补丁 ，点击通用，勾选部分值
![](_v_images/20201228114216708_10038.png =600x)
- 应用补丁 -> 应用补丁 ，点击高级，也勾选部分值，需要注意 仿冒图形卡ID这一项，选择这一项的前四位数的值，一定要与**平台ID**前四位数值一致，且后面的显卡型号与自己显卡型号也要一致的才可以。
![](_v_images/20201228114234885_29678.png =600x)
- 应用补丁 -> 生成补丁
- 文件 -> 导出 -> 引导工具 Config.plist ，会生成一个`config.plist`文件，使用ProperTree打开即可
![](_v_images/20201228120035750_3222.png =1000x)
- 将这两个段复制到正在使用EFI文件`config.plist`中替换即可
- 将`boot-args`中的`igfxframe=0x59120000`去掉，重启测试。

#### 修复声卡驱动
[Fixing audio with AppleALC官方文档](https://dortania.github.io/OpenCore-Post-Install/universal/audio.html)

##### 确认内核扩展
终端中输入 `kextstat | grep -E "AppleHDA|AppleALC|Lilu"`，确认三个是否都出现，否则无法正常打补丁。主要不要添加其他声卡驱动，否则将与AppleALC冲突。
