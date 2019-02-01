# Z-Probing
>There have been several bugs in the auto leveling and distortion correction code. This documentation assumes you have version 0.92.8 from 03/01/2016 or later installed!
## 工作原理

Before you start defining your auto leveling you should know what you are doing and what you can change and where you would change what error.

最重要的部件就是Z轴探针（Z probe，以下简称Z探针）。这个装置类似开关，在受到挤压时能够向IO口反馈一个信号。并且它与挤出头安装在一起，会在喷嘴触碰到热床前被优先触发。当它触发时就可以获得喷嘴距离热床的高度，我们可以用这种方法来测量热床在Z轴方向上的摆放状态。


The first thing we normally do is level out the bed. To do so we measure at least 3 points and compute the average plane trough this. The difference to the theoretically constant z show us how much rotation is required to be everywhere on same height. This can be done through a software rotation (if z moves have no backlash) or through a mechanical movement of the bed sides (through motorized modification of 2 from 3 fix points). This works quite well, but neglects some errors:

- 大多数热床都不会完全平整，尤其在加热的时候它会产生轻微形变。
- 测量并非100%准确
- 测量过程中可能会压弯你的打印平台

If that is a problem with your printer, you would measure a n x n grid to get a better approximation of the bed.

With z probes that require some considerable amount of force it may bend the bed depending on the position. For this we have a bending correction, that corrects measured values by bending. This is a bit limited especially if bending is not a linear function of x and y. Then you must limit measurements to 3 points for which you set the bending correction.

Even with an average plane for rotation, you still have bumps. With some luck the average bump is negligible and you can ignore it. In more extreme cases, especially with increasing bed sizes, it would be great to be able to follow the curvature of the bed for the first layers and remove that correction to zero with increasing z. That would help a big deal with correct bonding. This feature is called distortion correction and since 0.92.8 it is available for all printer types. It works by measuring a distortion map and storing it in eeprom (ideally) or ram (then you need to measure every time). Also since 0.92.8 we assume that distortion never changes, since it is a bed property. So recomputing the rotation will NOT reset the distortion map.

## Z探针 & Z轴归位
The best solution for printers having a z-probe is to home to z max. This allows rehoming during a print, e.g. if after a pause the motors were turned off and you might not trust your current position. Also you have no problems that a tilted bed hits the nozzle/probe while homing. You would simply home z first then xy and would never hit anything. Simple and save. The only drawback is that you need to home t z max and for printing go down again. This may take 30 seconds and some people thing that is not good to wait 30s also many prints take hours. So they are very sure they want to home to z min.

Well, first thing you must know is that a conventional z min end stop is not working here. Lets take some unlikely extremes to picture that. Lets say bed is rotated so that the left edge is 1cm higher the the right front and the 2 back edges are even 1cm higher. A conventional end stop is the lowest point and should prevent any moves going lower, so in this example it must match the right front edge. If we home Z first we will crash the nozzle into bed at any point except the right front edge. You see that is not a solution, so what you will need is a z sensor that is mounted on your extruder. To tell firmware we want to use z-probe also as z min end stop is by setting both to the same pin! That way we can measure everywhere the height to bed. But that is useless if we do not know where we are measuring, so we have to home x and y first. Ok, no problem if we are at a good height. Why? Well assuming our rotated bed again measuring at low z might cause extruder/probe to collide with the bed. So in version 1.0 newer then 14. January 2017 you can set homing to raise z before homing. Reason is simply to cause no collisions. Sounds simple, but what if are by accident near the top. If you have a z max end stop as well, that would catch the up move simply by triggering and you are safe. If you have no z max end stop just make sure never to move that much up. Ok, point taken, next issue. Now that xy homing is working safely we want to home z. So we have to think about a position where we can home z. Normally this is done at xy home position. But the z probe is in most cases not where the nozzle is plus it must be over the bed to do a valid measurement. So firmware will activate z probe adding offset and measure. SO if you are lucky it works since you home to min x and probe is on left side of nozzle. If not we HOME_ORDER_XYTZ which allows us to set a min. temperature for nozzle (only important if extruder is part of z probe) and more important we can set a probing coordinate. So we move first to that coordinate and then measure the height with offset. That way we can overcome the last problem with z min homing.

## 固件配置
### **Z-Probe**（Z探针）
 
从0.90版本开始Repetier固件开始支持自动调平（auto leveling）。如果使用这个功能需要一个Z探针装置，以自动或半自动的方式测距。

- 自动方式是指：Z探针与挤出头一起移动，不需要认为干预调平过程。

- 半自动方式是指：半自动的Z探针可能是数控机床上用来测量刀具高度的开关；因为它不与挤出头相连，用户需要 手动将探针放置在挤出头下方并按压它一次，以激活测量。测量后，探针会自动移至下一个位置并等候新的触发信号。 

That way you can have a z-probe, where you do not need to consider on how to fix and activate it. 

要想使用Z-Probe功能，需要先在configuration.h文件中进行配置。下面是配置在文件中所在的位置:

```c++
/* Z-Probing */
#define FEATURE_Z_PROBE true
/* After homing the z position is corrected to compensate
for a bed coating. Since you can change coatings the value is stored in
EEPROM if enabled, so you can switch between different coatings without needing
to recalibrate z.
*/
#define Z_PROBE_Z_OFFSET 0 // offset to coating form real bed level
/* How is z min measured
 0 = trigger is height of real bed neglecting coating
 1 = trigger is current coating
 
 For mode 1 the current coating thickness is added to measured z probe distances.
 That way the real bed is always the reference height. For inductive sensors
 or z min endstops the coating has no effect on the result, so you should use mode 0.
*/
#define Z_PROBE_Z_OFFSET_MODE 0
#define Z_PROBE_PIN 63
#define Z_PROBE_PULLUP true
#define Z_PROBE_ON_HIGH true
#define Z_PROBE_X_OFFSET -11.2625
#define Z_PROBE_Y_OFFSET -6.5

// 等待触发信号，探针触碰平台和OK按钮是有效的触发信号。如果你有一个需要手动触发的探针需要配置下面的参数。
#define Z_PROBE_WAIT_BEFORE_TEST false
/** 探针在Z轴方向移动的速度（mm/s） when probing */
#define Z_PROBE_SPEED 5
#define Z_PROBE_XY_SPEED 150
#define Z_PROBE_SWITCHING_DISTANCE 1.5 // 探针在触发后可以安全关闭的高度Distance to safely switch off probe after it was activated
#define Z_PROBE_REPETITIONS 5 // Repetitions for probing at one point. 
/** The height is the difference between activated probe position and nozzle height. */
#define Z_PROBE_HEIGHT 39.91
/** Gap between probe and bed resp. extruder and z sensor. Must be greater then initial z height inaccuracy!  */
#define Z_PROBE_BED_DISTANCE 30.0
/** These scripts are run before resp. after the z-probe is done. Add here code to activate/deactivate probe if needed. */
#define Z_PROBE_START_SCRIPT ""
#define Z_PROBE_FINISHED_SCRIPT ""
```
```C++
/* Auto leveling allows it to z-probe 3 points to compute the inclination and compensates the error for the print.
 This feature requires a working z-probe and you should have z-endstop at the top not at the bottom.
 The same 3 points are used for the G33 command.
*/
#define FEATURE_AUTOLEVEL true
#define Z_PROBE_X1 -69.28
#define Z_PROBE_Y1 -40
#define Z_PROBE_X2 69.28
#define Z_PROBE_Y2 -40
#define Z_PROBE_X3 0
#define Z_PROBE_Y3 80
```
首先在固件中启用Z探针功能（FEATURE_Z_PROBE设为true）并定义对应的信号IO接口（Z_PROBE_PIN）。也可以和限位开关一样对接口设置上拉（Z_PROBE_PULLUP）和反转触发信号（Z_PROBE_ON_HIGH）。很多用户喜欢用Z-min限位开关作为Z探针的接口。这是可行的，但是你要注意这并不意味着必须要在配置文件前面定义Z-min的That is ok, but you should be aware
that this does NOT mean you that you must define z min hardware endstop. It is only the z probe connected to the pin that could be used for this.
Next you define the offset relative to the extruder origin position (the basis you use to define extruder offsets). For single extruder setup this is the position of the nozzle. For multi-extruder setups the origin, from where you defined the extruder offsets.

If you need to manually move the z-probe, set Z_PROBE_WAIT_BEFORE_TEST to true. In that case, the extruder will hover over the measure points and wait for you to activate the switch once.

Z_PROBE_SPEED sets the probing speed in mm/s, while Z_PROBE_XY_SPEED is the speed in the xy plane. Z_PROBE_HEIGHT is the height difference between z-probe trigger and bed. This value is always added to the height, to compute the total height. Next comes Z_PROBE_BED_DISTANCE, which determines, at which height the probe will start measurements (probe height is added to this). You start with a guessed bed height, which you have set in the firmware. That height minus gap size must always be smaller then the real print area height. If you have some trick to enable/disable a z-probe, you can write the required g-code commands in Z_PROBE_START_SCRIPT and Z_PROBE_FINISHED_SCRIPT. Multiple commands are separated by a \n.

For more precise measurement you can repeat each point Z_PROBE_REPETITIONS times. After a probe is hit, the probe only lifts Z_PROBE_SWITCHING_DISTANCE mm. Depending on your probe type, this can be as less then 0.2 mm allowing very fast repetitions.

## 校正Z探针
一开始最容易犯错的就是Z探针的校正，所以你需要在校正任何参数前，先校准Z探针。

首先检查触发信号是否正确。正常情况下，发送`M119`检查Z探针状态“L”（低电平，表示未触发状态）；然后人为触发探针，再次发送`M119`，Z探针状态变为“H”（高电平，表示触发状态）。如果实际正相反，你就需要调整固件中的**Z_PROBE_ON_HIGH**参数。参数调整后如果情况没有改变，你可能插错了IO口，或者没有将对应IO口设置成**上拉**。修正错误后继续下面的步骤。

**Z_PROBE_HEIGHT**是最重要的参数，你需要在EEPROM中将它调整到正确的高度。这个参数定义了探针在触发那一刻，挤出头底部距离热床的高度。*大多数情况下这是个正数。但是如果Z探头使用的是压力传感器或是一个通过挤出头收到压力来触发的开关，则为0或一个小的负数*。测量这个距离的方法有很多，下面介绍一个最能准确测量高度的方法：

1- 找一个表面平坦并且知道准确高度的金属块。金属块的高度就是我们的参照物高度*ref_height*。

2- 讲Z探针移动到没有被触发的高度，发送命令`G30 P0`，将回显的Z轴高度记作z<sub>0</sub>。

4- 加热挤出头并将喷嘴清理干净，保持高温状态，以避免喷嘴残留的硬质打印材料会对测量结果产生影响。

5- 发送命令`M114`，记录下回显的Z轴高度，记为*z<sub>old</sub>*。

6- 通过软件调整Z轴高度直至让挤出头底部抵住金属块。最佳的状态是，当有一丝间隙的时候就可以轻易移动金属块。即便太低了，挤出头也会磕在金属块上而不会损坏热床。而且这个方法比用一页纸垫在热床上测试距离准确多了。一旦你找准了合适的位置，再次发送命令`M114`，记下高度*z<sub>new</sub>*。

So what we now have is starting height as firmware would assume it from first probe Z0 using our initial Z_PROBE_HEIGHT (ZHold). We also know that we needed to go up Znew-Zold to reach ref_height. From this we can compute the exact real value required from Z_PROBE_HEIGHT (ZHnew):

ZHnew = ZHold - Z0 - Znew + Zold + ref_height
Enter this value in eeprom and your measurements will be as precise as you can get it.

Hints: This works fine if you have always the same bed and bed coating. Inductive sensors measure distance to the metal bed part, so it would be possible to correct different bed coatings that you can but on it. The current thickness is stored in Z_PROBE_Z_OFFSET (bed coating in eeprom) and should be 0 for most printers.

Autoleveling
Now that we have a working z-probe, we enable auto leveling. For that you need to add “#define FEATURE_AUTOLEVEL 1”.
Then we define how we want the auto leveling to work. First we define how to measure the plane rotation. Currentyl 3 methods can be defined and set with BED_LEVELING_METHOD.

BED_LEVELING_METHOD 0

This method measures at the 3 probe points and creates a plane through these points. If you have
a really planar bed this gives the optimum result. The 3 points must not be in one line and have
a long distance to increase numerical stability. Delta printers should have them as close to the columns as possible.

BED_LEVELING_METHOD 1

This measures a grid. Probe point 1 is the origin and points 2 and 3 span a grid. We measure
BED_LEVELING_GRID_SIZE points in each direction and compute a regression plane through all
points. This gives a good overall plane if you have small bumps measuring inaccuracies.

BED_LEVELING_METHOD 2

Bending correcting 4 point measurement. This is for cantilevered beds that have the rotation axis
not at the side but inside the bed. Here we can assume no bending on the axis and a symmetric
bending to both sides of the axis. So probe points 2 and 3 build the symmetric axis and
point 1 is mirrored to 1m across the axis. Using the symmetry we then remove the bending
from 1 and use that as plane.

Next you should decide on the correction method. This correction is defined by BED_CORRECTION_METHOD and allows the following values:

BED_CORRECTION_METHOD 0
Use a rotation matrix. This will make z axis go up/down while moving in x/y direction to compensate
the tilt. For multiple extruders make sure the height match the tilt of the bed or one will scratch.

BED_CORRECTION_METHOD 1
Motorized correction. This method needs a bed that is fixed on 3 points from which 2 have a motor
to change the height. The positions are defined by
BED_MOTOR_1_X, BED_MOTOR_1_Y, BED_MOTOR_2_X, BED_MOTOR_2_Y, BED_MOTOR_3_X, BED_MOTOR_3_Y
Motor 2 and 3 are the one driven by motor driver 0 and 1. These can be extra motors like Felix Pro 1
uses them or a system with 3 z axis where motors can be controlled individually like the Sparkcube
does.

You need also to set the following parameter:

#define ENDSTOP_Z_BACK_MOVE 5
#define Z_HOME_DIR 1

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

它的原理是：通过测量n x n的网格上的每个点，将理论和实际高度差记录成一张凹凸贴图（bump map)，并将结果存储在EEPROM上（或者RAM上，但是机器重启后会失效）。这个功能用`G33`触发。程序会根据你预先在固件中定义好的网格点进行测量。**在使用这个功能前要确保Z轴d打印高度已经设置正确**。畸变校正后，程序将自动开启畸变校正补偿。下次打印时，挤出头在移动中就会按照各个位置的凹凸情况作出相应补偿。补偿值将随着打印高度的增加越来越小，直至停止补偿。

**注意**：畸变校正使用32位整型变量。如果Z轴变化值乘以网格点距离值的结果超过了2<sup>31</sup>，会导致内存溢出，进而计算结果会出现问题。通常大部分打印机不会遇到这个问题，但是在一些高精度打印机上如果Z轴变化值过大就会发生溢出！

G33命令有三种用法，分别是：

- `G33 L0`（输出凹凸贴图）：使用此命令你可以得到EEPROM中保存的全部突起补偿值。这些值会在相应的位置对Z轴坐标进行补偿，点与点之间会进行插值处理。

- `G33 X<xpos> Y<ypos> Z<newCorrection>`（调整凹凸贴图）：在最靠近输入坐标的位置设定一个新的补偿值。所以这个命令中的X、Y轴坐标不需要完全遵照G33 L0结果中的坐标输入。

- `G33 R0`（重置凹凸贴图）：将所有补偿值重置为0。

### 开启/禁用畸变校正
```
M323 ;          查看校正是否开启
M323 S0 ;       临时禁用校正
M323 S0 P1 ;    永久禁用校正
M323 S1 ;       临时启用校正
M323 S1 P1 ;    永久启用校正
```
## 手动调整热床
最后，还有一种替代方法是热床进行调平，这样就能保证做x-y的移动。这需要较少的计算，也不像自动调平那样磨损z轴（这相当于机械自动调平）。唯一的缺点是，要把它调整好的需要做更多的工作。最简单的方法是只用3颗螺丝固定热床，并设置自调平的3个测量点靠近热床的3个固定点。发送`G29`，固件程序将测量并输出3个点的高度。然后你在手动调整热床某一固定点的高度以确保自调平测量的高度一致，并反复重直至满意位置。如果打印机有z-max限位开关，发送`G29 S2`可以测量打印区域的高度，并将其永久存储在EEPROM中(S1将只测量高度)。