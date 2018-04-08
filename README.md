# ORB_SLAM2_Robot  
## Version1.0: 
在ORB_SLAM2算法的基础上，新开一个线程用于进行octomap的构建。ZED_Depth.yaml为系统参数配置文件。系统适用于ROS系统下，配合ZED摄像头使用。启动命令如下：    
>roscore     
>roslaunch zed_wrapper zed.launch   
>rosrun ORB_SLAM2 RGBD /home/nvidia/catkin_ws/src/ORB_SLAM2/Vocabulary/ORBvoc.txt /home/nvidia/catkin_ws/src/ORB_SLAM2/Examples/ROS/ORB_SLAM2/Zed_Depth.yaml   

使用ctrl+c退出系统。使用octomap带有的octovis进行对.bt地图文件进行观察。     
  
## Version1.1:  
这个版本新添加3个线程，分别是局部地图规划线程，全局地图规划线程，通信线程。  
#### 通信线程：  
>使用串口与下位机进行通信，下位机向上传输GPS信息以及实时机器人地磁传感器信息用于实时确定机器人的朝向。并将这些信息分发到局部路径规划线程以及全局路径规划线程中去。 

#### 全局路径规划：  
>这个线程的作用是通过获取的GPS定位信息，知道自己的全局坐标，然后根据我们人为输入的终点全局坐标，计算出实时的全局朝向。当然这个全局朝向比较粗略，因为暂时没有接入百度地图的API，我们没得全局大地图，全局朝向是以起点直指终点计算得出的。全局朝向是我们用于指示机器人运动大方向的。以正北方向为0度，然后以顺时针增加全局朝向角度，0~360度。 

#### 局部路径规划：  
>根据全局朝向角度，以及我们的octomap的数据，对机器人的正前方的三维地图进行二维压缩。然后根据机器人局部地图的初始建立角度，机器人实时地磁角度，全局朝向角度，对前方的二维地图进行A*的算法局部路径规划。不同的全局朝向和机器人实时朝向用于确定不同的局部规划终点。最终输出一个前方二维地图的矩阵，以及以地图的行进坐标点序列的方式输出局部路径规划的结果。 


## Verision2.0：  
从该版本开始，机器人代码分为2个分支。一个是基于阿栖机器人的AXi，一个是基于全向轮小车的Om。他们主要由于底盘不同，所以写的控制指令模式不同。该版本主要的更新在于修复各线程之间由于传感器不完全导致的不安全访问，修复路径规划系统中一些BUG，添加路径规划的平滑策略，添加机器人的相关控制逻辑。   
#### 修复传感器不完全导致错误的BUG：  
>由于系统需要使用地磁传感器，GPS，摄像头等所有传感器均能正常工作的时候才能正常运行。如果一旦其中一个传感器崩溃，系统理应进行相关处理，而不是仍由其跑飞。这里添加当某种传感器出错后的紧急应对措施，以及相关提醒，防止当错误情况发生时造成意外崩溃。   
而路径规划是其中最重要的部分。路径规划的开始，必须由点云线程和全局路径规划线程均就位才行。也就是必须当局部3D地图第一帧收录后，以及通过GPS定位将全局路径规划出来后才开始正式的局部路径规划以及发送控制指令。  

#### 修复路径规划中的BUG：  
>原来的路径规划代码，是最简单的A*算法。这里我们将A*的H计算公式，由曼哈顿距离，改为斜直距离，这样更加符合我们机器人的行动模式。  

#### 路径规划平滑策略：  
>由于A*算法的结果是得到一些列的前进点，这样不利于我们对于小车的控制。这里采用添加拐角惩罚的优化算法，对小车的形式轨迹进行优化，并进行直行检测后去掉中间点。使最后得到的小车轨迹是少量且少拐角的规划路径。

#### 机器人的相关控制策略：   
>由于有AXi和Om两种不同底盘的机器人，他们的行动模式各有不同。现在根据其具体行动模式，对其各个添加相应的机器人底盘控制代码，通过通信进程发送给机器人，使其进行运动。  

## Verision2.1：
该版本是基于Om小车也就是全向轮小车的框架进行的开发。主要修复了系统的路径规划以往需在第2个关键帧后才能进行的BUG，以及注释添加了无显示器模式，添加了跟踪丢失过后的应对措施。具体如下：    
#### 修复了octomap建图在第二个关键帧才启动的BUG:    
>原来pointcloud线程是通过创建新的关键帧时才获得帧，而没有获得初始化的第一帧。而局部路径规划是在pointcloud线程启动后才能运行的。现在在SLAM系统初始化的过程中，会把第一帧传递给pointcloud线程，达到一开机小车就能进行运动规划。    

#### 添加无显示屏模式:    
>这个需要在ros-rgbd文件里面，在初始化SLAM系统时，将viewer的输入改为false即可。

#### 跟踪丢失的相关操作:    
>在小车的运动过程中，会出现ORBSLAM跟踪失败的情况。现在当这种情况发生后，我们会令小车原地停止下来，也即不再发送路径规划指令。同时如果当30s后，小车都未重定位成功，表示此时小车已经彻底跟踪失败。此时我们清除掉原始SLAM系统中所有的数据，包括关键帧，以及OCTOMAP。并对系统进行重置，使小车以目前为坐标（0,0）重新开始路径规划。

## Verision2.2：
该版本是基于Om全向轮小车开发的一个过渡版本。它在2.1的基础上对于建图精度进行了优化。同时会作为AXi机器人的初始代码。
#### 建图精度的提升：
>该版本对于三维重建里面的滤波算法进行了改进。我们根据Zed相机110°广角，16:9成像原理，是滤波结果只接受0.5-5m深度范围内，左右视角90°，上下视角45°范围内的占据信息，防止深度畸变导致的建模错误。

## Verision2.3：
该版本是基于Om全向轮小车的基础上开发出的AXi机器人2.3版本。从该版本之后两个机器人的SLAM系统，路径规划等代码全部一致。区别只在于，局部路径导航的主函数中，对于机器人的指令函数名的改变。需要注意的是，该版本由于未拿到AXi机器人的具体数据，在控制方面还不够完善。同时这也是在进行实际运行之前的版本。
### 重大更新：
从这一版本开始，我们的机器人完全脱离ROS系统。也即现在是ZED-SLAM的结构，而不是ZED-ROS-SLAM的结构。我们使用ZED提供的SDK，通过直接操作ZED摄像头进行深度图像的测量，将其结果放入SLAM系统，进行后续工作。这样能够节省ROS中的大量额外开销，例如ZED_ROS_WARPPER里面会进行位姿估计，点云生成等不需要的操作。使用ZED与SLAM直连的形式，将GPU的占用和CPU的占用都降低了许多，为以后我们基于GPU优化，提供了便利。同时我们进行系统运行时，也不需要开启3个系统，只需要启动一条指令即可。
#### 现在的系统运行指令如下：
>/home/nvidia/ORB_SLAM2/Examples/RGB-D/rgbd_my /home/nvidia/ORB_SLAM2/Vocabulary/ORBvoc.txt /home/nvidia/ORB_SLAM2/Examples/RGB-D/Zed_Depth.yaml
#### 系统结束指令：
>在控制台下，按'q'键退出系统。
#### 基于新AXi机器人的控制指令更新：
>AXi新版有很多新的特性，例如转角有一定局限，转弯与直行模型之间需要等待等。同时由于路径规划算法的结果现在已经变为只有几个点了。所以几个控制指令全部进行更新。
