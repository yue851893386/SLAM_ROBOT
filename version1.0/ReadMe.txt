Version1.0:
��ORB_SLAM2�㷨�Ļ����ϣ��¿�һ���߳����ڽ���octomap�Ĺ�����ZED_Depth.yamlΪϵͳ���������ļ���ϵͳ������ROSϵͳ�£����ZED����ͷʹ�á������������£�
roscore
roslaunch zed_wrapper zed.launch
rosrun ORB_SLAM2 RGBD /home/nvidia/catkin_ws/src/ORB_SLAM2/Vocabulary/ORBvoc.txt /home/nvidia/catkin_ws/src/ORB_SLAM2/Examples/ROS/ORB_SLAM2/Zed_Depth.yaml
ʹ��ctrl+c�˳�ϵͳ��ʹ��octomap���е�octovis���ж�.bt��ͼ�ļ����й۲졣