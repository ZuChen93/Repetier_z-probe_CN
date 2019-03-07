# Z-Probing
>原文：https://www.repetier.com/documentation/repetier-firmware/z-probing/
>
>旧版固件的自动调平和畸变校正代码存在问题，请使用版本0.92.8（2016年3月1日）之后的固件。
[TOC]

## 工作原理

在实际操作前请弄懂你所做的每一步，了解哪些配置可以修改，出现问题时需要调整哪个参数。

在机器调试过程中，最重要的部件就是Z轴探针（Z probe，以下简称Z探针）。这个装置类似限位开关，它在受到挤压时能够向IO口反馈一个信号。通常Z探针会与挤出头安装在一起，并会在喷嘴触碰到热床前被优先触发。当它触发时就可以计算出喷嘴距离热床的高度，我们可以用这种方法来测量热床在Z轴方向上的摆放状态。

那么我们通常先要做的是对热床进行调平。我们至少需要测量三个点的坐标，通过计算出一个平均值来获得热床表面的情况。与高度 *Z* 的差值显示了要达到同一高度的任意一处，电机需要转动多少次。这可以通过程序作出补偿（如果Z轴没有设置侧隙补偿*backlash*）或者手动机械调整热床的调平螺丝来修正。这个方法可行，但是忽视了一些问题：

- 大多数热床都不会完全平整，尤其在加热的时候它会产生轻微形变
- 测量并非100%准确
- 测量过程中可能会压弯热床

如果你的打印机存在以上问题，就需要测量一组n x n的网格点来获得热床更好的近似值。

有些Z探针需要受到很大的力才能触发，这可能会使热床当前测量位置的区域发生形变。对于此种情况，固件中有一个弯曲矫正功能（*bending correction*）可以对发生的弯曲作出补偿。但是这种方法有一定的局限性：如果弯曲不是关于x和y的线性函数关系，必须使用三点测量法调平。

即使对热床做了调平，你仍然会遇到平台上有小突起的情况。如果幸运的话突起微乎其微，那么你可以忽略。但是在更极端的场景下，尤其是扩大了打印平台的尺寸后，可以根据热床的曲率，通过提升Z轴坐标做出补偿来保证在打印第一层的时的准确性。这对打印材料与平台的贴合有十分大的帮助。这个功能叫做畸变校正（*distorsion correction*），Repetier固件从0.92.8版本以后对全类型打印机支持此功能。这个功能的原理是通过测量生成一个畸变凹凸图*distortion map*，并将数据存储在EEPROM（推荐）或者RAM（每次复位后都需要重新测量）。并且从0.92.8版本开始，程序会默认畸变参数恒定不变，因为这是热床的固有属性。所以调平**不会**重置畸变凹凸图的参数。

## Z探针 & Z轴归位
如果打印机有Z探针，最好设置机器的归零方向是Z-MAX，这样可以使打印机在打印前将位置重新归位。因为如果步进电机在暂停后关闭了，当前位置的坐标就不可信了；而且也避免了发生喷嘴/探针在归位时与翘起的热床发生磕碰。你需要先归位Z轴再归位XY轴，这样就保证不会发生磕碰。简单高效。唯一的缺点是每次打印前，机器为了归位不得不先移动到Z-MAX，再向下移动进行打印。这个归位过程可能会需要30秒的时间，即便打印过程长达数小时，有些人也不愿意等这30秒时间，所以他们就坚持把归位方向设置为Z-MIN。

然而，你不得不得接受一个现实：传统的Z-MIX限位开关在这个场景下是完全没有作用的。让我们用一个不太可能的极端情况来说明：假如热床左边缘比右前方高出1厘米，后半部分甚至高出了2厘米。传统的限位开关安装在机轴底部，以防止任何再往下的移动，在当前情况中，热床右前方区域是整个坐标系统的最低点，这样限位开关就应该探测右前方边缘位置。这时如果执行Z-MIN归位操作，那么任何向除了右前方区域的移动都会使喷头撞上热床。这并不是我们想要看到的情况，要想避免发生这个问题就要有一个捆绑在挤出头上的Z探针，并将信号输出连接到主板上的Z-MIN限位开关接口上，将其视作Z-MIN限位开关。这样就可以探测热床上每一点的高度了。但是探测高度时如果没有当前的位置信息是没有用的，所以我们应该先进行X、Y轴归位。好了，到这里如果高度合适就没有问题了。为什么？假设热床再次测量Z轴更低的高度，又可能会导致挤出机或者探针与热床相撞。所以Repetier从1.0版本（2017年1月14日）之后，你可以设置机器在Z轴归位前先将喷头抬升。原因很简单就是为了避免发生碰撞。听起来很简单，但是喷头会一直向机器顶部移动不停下来，于是又发生了碰撞。这时如果我们在机器顶部安装了Z-MAX限位开关就可以避免这一情况了。如果没有Z-MAX限位开关，要确保喷头在机器允许的高度范围内移动。好了，关键点来了，下一个问题。现在X、Y轴归位已经工作正常了，我们开始配置Z轴归位。这时我们要考虑到哪里可以进行Z轴归位。通常是X、Y轴归位的地方。的方向So we have to think about a position where we can home z. Normally this is done at xy home position. But the z probe is in most cases not where the nozzle is plus it must be over the bed to do a valid measurement. 

So firmware will activate z probe adding offset and measure. SO if you are lucky it works since you home to min x and probe is on left side of nozzle. If not we HOME_ORDER_XYTZ which allows us to set a min. temperature for nozzle (only important if extruder is part of z probe) and more important we can set a probing coordinate. So we move first to that coordinate and then measure the height with offset. 

这样我们就解决了使用Z-MIN归位时遇到的最后一个问题了。

## 固件配置
### Z-Probe（Z探针）

从0.90版本开始Repetier固件开始支持自动调平（auto leveling）。如果使用这个功能需要一个Z探针装置，以自动或半自动的方式测距。

- 自动方式是指：Z探针与挤出头一起移动，不需要认为干预调平过程。

- 半自动方式是指：半自动的Z探针可能是数控机床上用来测量刀具高度的开关；因为它不与挤出头相连，用户需要手动将探针放置在挤出头下方并按压它一次，以激活测量。测量后，探针会自动移至下一个位置并等候新的触发信号。这样就能在使用Z探针的时候无需考虑如何固定和激活它。要想使用Z-Probe功能，需要先在configuration.h文件中进行配置。以下是配置文件中的相关参数:

```c++
/* Z-Probing */
#define FEATURE_Z_PROBE true
/* 机器归位后，Z轴坐标是准确的，可用于平台覆盖物的厚度补偿。如果启用了平台覆盖物功能，由于存储在EEPROM中覆盖物的厚度值可以修改，你可以在不同的平台覆盖物类型之间切换，而无需重新校正Z轴高度。
*/
#define Z_PROBE_Z_OFFSET 0 // 覆盖物表面到真正打印平台表面的厚度
/* 如何探测Z轴最低点
 0 = 无视覆盖物，以真实平台的高度触发
 1 = 将平台与当前覆盖物的高度算在一起触发
 
 对于第1种方式，当前的覆盖物厚度会叠加到测量的Z探针测量结果中，这样真正的平台高度永远都是相对高度。如果Z探针使用的是金属感应类传感器或者Z-MIX限位开关，那么平台覆盖物并不会影响探测结果，所以你需要使用模式0。
*/
#define Z_PROBE_Z_OFFSET_MODE 0
#define Z_PROBE_PIN 63
#define Z_PROBE_PULLUP true
#define Z_PROBE_ON_HIGH true
#define Z_PROBE_X_OFFSET -11.2625
#define Z_PROBE_Y_OFFSET -6.5

// 如果你有一个需要手动触发的探针，想收到触发信号再开始探测，需要配置下面的参数。手动触碰探针或者在操作屏幕上按下OK按钮都是有效的触发信号。
#define Z_PROBE_WAIT_BEFORE_TEST false
/** 探针在Z轴方向移动的速度（mm/s） */
#define Z_PROBE_SPEED 5
#define Z_PROBE_XY_SPEED 150
#define Z_PROBE_SWITCHING_DISTANCE 1.5 // 探针在触发后可以关闭的安全高度
#define Z_PROBE_REPETITIONS 5 // 对同一测量点的探测次数 
/** 探针在开始探测时探测头部距离喷嘴头部的距离 */
#define Z_PROBE_HEIGHT 39.91
/** Gap between probe and bed resp. extruder and z sensor. Must be greater then initial z height inaccuracy!  */
#define Z_PROBE_BED_DISTANCE 30.0
/** These scripts are run before resp. after the z-probe is done. Add here code to activate/deactivate probe if needed. */
#define Z_PROBE_START_SCRIPT ""
#define Z_PROBE_FINISHED_SCRIPT ""
```
```C++
/* 自调平功能可以让Z探针通过测量平台上3个点的坐标来计算倾角，以补偿打印时的误差。此功能需要有一个工作正常的Z探针做支持，并且在机器顶部（不是底部）需要安装上Z-MAX限位开关。这三点的坐标会在G33命令中使用到。
*/
#define FEATURE_AUTOLEVEL true
#define Z_PROBE_X1 -69.28
#define Z_PROBE_Y1 -40
#define Z_PROBE_X2 69.28
#define Z_PROBE_Y2 -40
#define Z_PROBE_X3 0
#define Z_PROBE_Y3 80
```
首先在固件中启用Z探针功能（**FEATURE_Z_PROBE**设为true）并定义对应的信号IO接口**Z_PROBE_PIN**。也可以和限位开关一样对接口设置上拉**Z_PROBE_PULLUP**和反转触发信号**Z_PROBE_ON_HIGH**。很多用户喜欢将Z探针的接口插在主板上的Z-MIN限位开关接口上。这是可行的，但是请注意，这并不意味着必须要在配置文件中定义出Z-MIN硬件限位开关是哪个接口，只需定义出与Z探针相连的那个接口就可以了。

接下来定义探针相对于挤出机原点位置的偏移量（这也是用来定义挤出机偏移量的基础）。对于单喷头来说可以直接将喷头的位置视为原点。对于多喷头来说，需要先定义每个喷头的偏移量。

如果需要手动移动Z探针，你需要将参数**Z_PROBE_WAIT_BEFORE_TEST**设置为true。在这种情况下，挤出机会悬停在测量点上，等待你再次激活开关。

参数**Z_PROBE_SPEED**设定了探针垂直方向移动的速度（单位mm/s），而**Z_PROBE_XY_SPEED**是探针在水平方向移动的速度。**Z_PROBE_HEIGHT**表示Z探针与打印平台的高度差。这个数值会叠加到打印高度上，以计算总高度。参数**Z_PROBE_BED_DISTANCE**定义了探针会在哪个高度开始探测（Z_PROBE_HEIGHT会叠加到这个高度上）。程序会从一个假想的平台高度开始测量，这个高度是你定义在固件中的高度。这个高度减去Z_PROBE_HEIGHT必会小于真正的打印高度。

如果启动/停止Z探针需要特殊的命令，你可以将所需的g代码分别填在参数**Z_PROBE_START_SCRIPT**和**Z_PROBE_FINISHED_SCRIPT**内，如有多行命令以`\n`分隔。

要想测量得更精确，可通过设置**Z_PROBE_REPETITIONS **参数来对一点多次探测。探针在触发后会抬升**Z_PROBE_SWITCHING_DISTANCE**毫米。根据探针类型，这个参数可以设置小于0.2毫米，以获得更快的重复探测速度。

## 校正Z探针
一开始最容易犯错的就是Z探针的校正，所以你需要在校正任何参数前，先校准Z探针。

首先检查触发信号是否正确。正常情况下，发送`M119`检查Z探针状态应该是“L”（低电平，表示未触发状态）；然后手动触发探针，再次发送`M119`，Z探针状态应该变为“H”（高电平，表示触发状态）。如果实际正相反，你就需要调整固件中的**Z_PROBE_ON_HIGH**参数。参数调整后如果情况没有改变，你可能插错了IO口，或者没有将对应IO口设置成**上拉**。修正错误后继续下面的步骤。

**Z_PROBE_HEIGHT**是最重要的参数，你需要在EEPROM中将它调整到正确的高度。这个参数定义了探针在触发那一刻，挤出头底部距离热床的高度。*大多数情况下这是个正数。但是如果Z探头使用的是压力传感器或是一个通过挤出头收到压力来触发的开关，则为0或一个小的负数*。测量这个距离的方法有很多，下面介绍一个最能准确测量高度的方法：

1- 发送命令`G28`将打印机归位，查看固件中已有的Z_PROBE_HEIGHT值，记作*ZH<sub>old</sub>*

2- 找一个表面平坦并且知道准确高度的金属块作为参照物。将金属块的高度记作*ref_height*。

3- 将Z探针移动到没有被触发的高度，发送命令`G30 P0`，将回显的Z轴高度记作*Z<sub>0</sub>*。

4- 加热挤出头并将喷嘴清理干净，保持高温状态，以避免喷嘴残留的硬质打印材料会对测量结果产生影响。

5- 发送命令`M114`，记录下回显的Z轴高度，记为*Z<sub>old</sub>*。

6- 通过软件调整Z轴高度直至让挤出头底部抵住金属块。最佳的状态是，当有一丝间隙的时候就可以轻易移动金属块。即便太低了，挤出头也会磕在金属块上而不会损坏热床。而且这个方法比用一页纸垫在热床上测试距离准确多了。一旦你找准了合适的位置，再次发送命令`M114`，记下高度*Z<sub>new</sub>*。

7- 至此我们就可以计算出准确的Z_PROBE_HEIGHT值（*ZH<sub>new</sub>*）：
    *ZH<sub>new</sub> = ZH<sub>old</sub> - Z<sub>0</sub> - Z<sub>new</sub> + Z<sub>old</sub> + ref_height*

将这个结果保存至EEPROM中，这将是你可以能获得的最精确的测量值。

**提示**：如果你的热床和平台覆盖物一直不更换的话，测量数值一直有效。对于感应传感器的Z探针，它只会测量到金属热床的距离，这样它就可以纠正不同的平台覆盖物的厚度了。当前的覆盖物厚度存储在EEPROM中的Z_PROBE_Z_OFFSET参数里，对于大多数打印机这个值是0。

## 自动调平
### 1、在固件中启用自调平功能
现在我们就有了工作正常的Z探针了，要想开启这个功能，在固件中的相应位置做如下定义`#define FEATURE_AUTOLEVEL 1`。

### 2、定义自动调平的策略
首先我们定义如何测量平面旋转（plane rotation）。目前固件里内置了3种调平策略，可以通过`BED_LEVELING_METHOD`参数设定。

- BED_LEVELING_METHOD 0：

  ![img](https://www.repetier.com/w/wp-content/uploads/2016/01/bed-leveling-method-0.png) 

  此策略会通过探测3个点创建一个平面。如果你可以确保热床表面非常平整，用这个策略会获得最佳效果。这三个点**必须**不能处在同一直线上，而且需要相距很远以确保数值稳定性（numerical stability）。对于Delta型打印机，这三点的坐标应该尽可能的靠近三根立柱。

- BED_LEVELING_METHOD 1

    ![img](https://www.repetier.com/w/wp-content/uploads/2016/01/bed-leveling-method-1.png)

    此策略会探测一张网格。如图所示以P1为原点，通过P2、P3两点扩展成一组网格点。程序会分别在每一个边探测`BED_LEVELING_GRID_SIZE`个点，最后计算出一张回归平面。如果热床局部有小的突起，用此策略可以获得一个整体不错的平面。 points in each direction and compute a regression plane through all points. This gives a good overall plane if you have small bumps measuring inaccuracies.

- BED_LEVELING_METHOD 2

    ![img](https://www.repetier.com/w/wp-content/uploads/2016/01/bed-leveling-method-2.png)

    4点测量法以修正平台弯曲。此策略适用于悬臂式平台（cantilevered bed，译注：见下图，Google搜到的不确定……）![Image result for 3d printing cantilevered beds](http://mark.rehorst.com/misc/CubeX_Duo/CubeX%20Duo%20Z%20axis%20drawing.png) 

    此类平台的旋转轴不在平台边缘而是在平台内部。这里假设一种平台中轴线没有弯曲，而中轴线的两侧呈对称性弯曲的情况。如图，P2和P3点连线构成一条对称轴，P1和P1<sub>m</sub>点关于直线P2P3轴对称。通过这种对称性，我们就可以消除从P1点产生的弯曲并形成一个平面。

下面还需要定义一个校正方法。通过参数'BED_CORRECTION_METHOD'设置：

- BED_CORRECTION_METHOD 0
  使用旋转矩阵（rotation matrix）。挤出头在x/y轴平面运动的时候，z轴坐标会根据矩阵中相应位置的数据进行补偿。对于多挤出头，请确保它们的高度与打印平台的倾斜度相匹配，否则会发生剐蹭。

- BED_CORRECTION_METHOD 1
  机械化修正。这个方法需要平台由3点固定，其中2点可以通过电机改变高度。这三个固定点的坐标由参数`BED_MOTOR_1_X, BED_MOTOR_1_Y, BED_MOTOR_2_X, BED_MOTOR_2_Y, BED_MOTOR_3_X, BED_MOTOR_3_Y`决定。其中电机2和3分别由驱动模块0和1驱动。These can be extra motors like Felix Pro 1 uses them or a system with 3 z axis where motors can be controlled individually like the Sparkcube
  does.

You need also to set the following parameter:

```c++
#define ENDSTOP_Z_BACK_MOVE 5
#define Z_HOME_DIR 1
```
The first makes the z-axis go down a few mm after hitting the endstop. This is needed, because we assume a not even bed and correct z height during x-y-travel. It is important not to hit the z-endstop during these travels or your coordinate system gets wrong. The value must be larger then the maximum height difference of your bed. Delta printer user have a special problem at z max. There it is not possible to move the extruder without hitting an endstop. Printing at that height is also not possible for the same reason, so it is a good idea to set ENDSTOP_Z_BACK_MOVE to even higher values where some movements are possible like 50mm. Remember that a slight bed tilt also needs a side move! The second sets homing to z-max. This again is required, because the bed is not planar. If we had a z-min endstop, it should only get triggered where the bed has it’s lowest point. Of course this would mean, that for all other positions homing to z-min would crash the extruder head into the bed.

Verify z-probe
Before you risk any damage, you should test, if you have configured the z-probe correctly. Run a

G31
to check the signal of the z-probe. It should be low/off. Now trigger it by hand and send the command again. Make sure the signal is now inverted.

Next home and measure the distance between extruder and bed. Some commands like G30/G32 will go to a lower position when started, namely Z_PROBE_BED_DISTANCE + Z_PROBE_HEIGHT, so make sure that is everywhere above the bed!

Now that this works, make your first real probe. Move your extruder to a position you want to measure and send:

G30
The extruder will now descent, until the z-probe gets triggered and then go back to the starting position. In your log, you should then find the distance needed to trigger the z-probe. For a first test you may start with plenty of room and trigger the switch manually, just to test if it reverse, which should be the case if you set up the probe correctly. If you have a semi-automatic z-probe hit the probe first to start probing.

Automatic correction
Now that everything is configured and working we can start adjusting the bed. Before using the leveling commands you need to at least home x and y axis. Deltas will home on G32, so they can skip this. Depending on the measurement method you have more or less testing points and time can be greatly reduced if we have to move less in z direction. When we have homed also in z axis the probe gets positioned at Z_PROBE_BED_DISTANCE + Z_PROBE_HEIGHT (if positive). You should select
Z_PROBE_BED_DISTANCE such that it is higher then every expected tilt. If you have only homed x and y as you do not want to home to max z for the time it takes, you should know that the height you enabled the printer is 0 internally until you home. So positioning will be normally higher.

One simple method is G29. It will measure 3 heights (at probing points) and use the average as printer height. This requires homing to z max before starting it.

The much better solution is to correct the possible rotation. We have already defined how we measure and how we correct, so there is no real difference any more. The command to start autoleveling is G32 with optional Sx parameter. What then happen is the following procedure:

Disabling distortion correction and autolevelig. Autoleveling rotation matrix gets reset.
Delta printers will home and go down again to starting height.
If you are higher then max. starting height the probe will lower to that height.
Measuring and correcting rotation.
If you have a max z endstop and parameter S was not 0, we update the printer height so later homing gives correct results. Make sure to do this only IF you had homed to z max before or you get completely wrong heights.
Update current position.
Store new matrix in EEPROm if parameter S is 2 or higher.
Enable automatic correction.
Enable distortion correction if it was enabled before.
Nonlinear printers like deltas will home again to get right positioning.
Older firmware versions (< 0.92.8) had S1 for measuring and update z length. This is now done always for z max homing as long as S is not 0.

## Delta型打印机

<iframe width="560" height="315" src="https://www.youtube.com/embed/L9URtv2LqKc" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

> 自0.92版本以后，触发畸变校正（distorsion correction）功能的命令是`G33`（取代原来的G29）。

校正Delta型打印机比XYZ笛卡尔型打印机需要额外的步骤——**需要先校正限位开关的位置**。此操作会更改EEPROM配置中的“Tower X(Y/Z) endstop offset [steps].”参数。其思路很简单：打印机在归位到最大限位开关以后，固件程序会控制打印机根据这个参数向下移动一段补偿距离，确保将挤出头准确地定位在中心位置。确定打印机的中心位置很重要，否则delta机构的非线性动作将产生错误的几何形状。人为测量这个补偿值几乎不可能，但是用下面的方法可以达到同样目的：
- 将打印机归位`G28`，清空限位开关全部补偿值（extruder offset）`G131`

- 将3个滑轨上的滑块定位在同一高度。发送命令`M84`关闭步进电机，找一根棍子贴住滑轨，棍子下端抵住打印机底部（不要抵住热床，可能会有些许形变），移动滑块来压住棍子上端。**注意用手移动滑块时不要太快**，因为这样做步进电机会产生电流反向流进电子元件中。 
- 由于有的打印机在关闭电机的时候会向下滑动，如果使用新的固件就可以使用M99命令单独关闭一个电机。首先发送`M99 X0`关闭X(A)轴电机10秒钟，在这10秒钟里，用棍子定位好A轴滑块的位置（可以在中间追加参数S<秒>来改变关闭的时间）。然后再换成Y0、Z0来对另外两个轴进行相同操作。
- 一旦所有滑块都调整到位，发送命令`G132 S1`。随后打印机会开始测量补偿值。参数S1代表在测量后将结果存储在EEPROM中，这样以后就不用每次都重复测量了。

一旦限位开关校正完成，你就可以和笛卡尔型打印机一样继续执行调平程序了。

### 开启/禁用自动调平（*autolevel*）

一旦你发送了`G32`命令，固件就切换到了自动调平模式。如果你添加了`S2`参数将参数保存在EEPROM中，设置会在机器重启后依然存在。如果更换打印平台导致打印高度发生了改变，你就得重新调平。可以使用下面的命令：
```
M320 ;      激活自动调平，调平结果临时生效
M320 S2 ;   激活自动调平，调平结果永久生效
M321 ;      禁用自动调平，调平结果临时生效
M321 S2 ;   禁用自动调平，调平结果永久生效
M322 ;      临时清空自动调平参数矩阵
M322 S3 ;   永久清空自动调平参数矩阵
```
如果你重置了自动调平参数，结果和激活/禁用自动调平一样。

## 畸变校正（*distorsion correction*）
幸运的话你已经完成了机器的调试。然而如果你的热床不是100%平整，或者delta型打印机的几何移动不是完全正确，那么打印机热床和挤出头之间的高度仍然是不均衡的。Repetier固件从0.92.8版本开始，可以通过畸变校正功能来修正这个问题。

它的原理是：通过测量n x n的网格上的每个点，将理论和实际高度差记录成一张凹凸图（bump map)，并将结果存储在EEPROM上（或者RAM上，但是机器重启后会失效）。这个功能用`G33`触发。程序会根据你预先在固件中定义好的网格点进行测量。**在使用这个功能前要确保Z轴d打印高度已经设置正确**。畸变校正后，程序将自动开启畸变校正补偿。下次打印时，挤出头在移动中就会按照各个位置的凹凸情况作出相应补偿。补偿值将随着打印高度的增加越来越小，直至停止补偿。

**注意**：畸变校正的相关参数使用32位整型变量。如果Z轴变化值乘以网格点距离值的结果超过了2<sup>31</sup>，会导致内存溢出，进而计算结果会出现问题。通常大部分打印机不会遇到这个问题，但是在一些高精度打印机上如果Z轴变化值过大就会发生溢出！

G33命令有三种用法，分别是：

- `G33 L0`（输出凹凸图）：使用此命令你可以得到EEPROM中保存的全部突起补偿值。这些值会在相应的位置对Z轴坐标进行补偿，点与点之间会进行插值处理。

- `G33 X<xpos> Y<ypos> Z<newCorrection>`（调整凹凸图）：在最靠近输入坐标的位置设定一个新的补偿值。所以这个命令中的X、Y轴坐标不需要完全遵照G33 L0结果中的坐标输入。

- `G33 R0`（重置凹凸图）：将所有补偿值重置为0。

### 开启/禁用畸变校正
```
M323 ;          查看校正是否开启
M323 S0 ;       临时禁用校正
M323 S0 P1 ;    永久禁用校正
M323 S1 ;       临时启用校正
M323 S1 P1 ;    永久启用校正
```
## 手动调整热床
最后，还有一种替代方法是热床进行调平，这样就能保证做x-y的移动。这需要较少的计算，也不像自动调平那样磨损z轴（这相当于机械自动调平）。唯一的缺点是，要把它调整好的需要做更多的工作。最简单的方法是只用3颗螺丝固定热床，并设置自调平的3个测量点靠近热床的3个固定点。发送`G29`，固件程序将测量并输出3个点的高度。然后你在手动调整热床某一固定点的高度以确保自调平测量的高度一致，并反复重直至满意位置。如果打印机有z-max限位开关，发送`G29 S2`可以测量打印区域的高度，并将其永久存储在EEPROM中（`G29 S1`将只测量高度不储存）。