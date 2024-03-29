@[TOC](开关CentOS CPU超线程的脚本分析)

脚本来自网络，closeHyperThread.sh为入口，调用toggleHyperThreading.sh。本文为部分关键逻辑添加注释，笔者还未深入了解超线程，在siblings处可能有误解，welcome chat.
# closeHyperThread.sh
```bash
cpuinfo1=$(cat /proc/cpuinfo | grep siblings | uniq | awk '{print $3}' | tr -d ' ')
cpuinfo2=$(cat /proc/cpuinfo | grep "cpu cores" | uniq | awk '{print $4}' | tr -d ' ')
echo siblings = $cpuinfo1
echo cpu cores = $cpuinfo2
if [ $cpuinfo1 -eq $cpuinfo2 ]  ;then
echo "超线程已关闭"
elif [ $[$cpuinfo1/2] -eq $cpuinfo2 ];then
        echo "超线程未关闭"
#       read -p "是否脚本关闭超线程(y/n)" answer
#       [ $answer == "y" ]
 echo -e '\nd' | /masked_route/toggleHyperThreading.sh
else
         echo “请查看超线程情况”
fi
```

# toggleHyperThreading.sh
```bash
HYPERTHREADING=1

function toggleHyperThreading() {
 for CPU in /sys/devices/system/cpu/cpu[0-9]*; do
	#将目录名截断，只留编号作为CPUID。比如./bpu6,则CPUID=6
   CPUID=`basename $CPU | cut -b4-`
	#打印CPUID
   echo -en "CPU: $CPUID\t"
	#  -e filename   如果file存在则为true
	#    command1 && command2        and list    只有当command1返回为0时才会执行command2
	#判断一下当前cpu是否为online状态，如果不在，则重置为online状态，便于后续统一更改
   [ -e $CPU/online ] && echo "1" > $CPU/online
	#thread_siblings_list中存储了虚拟cpu与真实cpu的siblings的关系
	#如  6，22   表示cpu6为真实cpu，cpu22为其对应的虚拟cpu
	#相应的，cat cpu6/topology/thread_siblings_list与cpu22/topology/thread_siblings_list会得到完全相同的结果， 即6，22
	#截取siblings关系中前一个cpu编号赋值给THREAD1变量
   THREAD1=`cat $CPU/topology/thread_siblings_list | cut -f1 -d,`
	#打开一半关闭一半，siblings中前面的打开，后面的关闭
   if [ $CPUID = $THREAD1 ]; then
     echo "-> enable"
     [ -e $CPU/online ] && echo "1" > $CPU/online
   else
	#如果HYPERTHREADING flag=0，意味着当前在执行关闭操作，后一半cpu关闭
    if [ "$HYPERTHREADING" -eq "0" ]; then echo "-> disabled"; else echo "-> enabled"; fi
     echo "$HYPERTHREADING" > $CPU/online
   fi
 done
}

function enabled() {
    echo -en "Enabling HyperThreading\n"
    HYPERTHREADING=1
    toggleHyperThreading
}

function disabled() {
    echo -en "Disabling HyperThreading\n"
    HYPERTHREADING=0
    toggleHyperThreading
}

#
ONLINE=$(cat /sys/devices/system/cpu/online)
OFFLINE=$(cat /sys/devices/system/cpu/offline)
echo "---------------------------------------------------"
echo -en "CPU's online: $ONLINE\t CPU's offline: $OFFLINE\n"
echo "---------------------------------------------------"
while true; do
  read -p "Type in e to enable or d disable hyperThreading or q to quit [e/d/q] ?" ed
  case $ed in
    [Ee]* ) enabled; break;;
    [Dd]* ) disabled;exit;;
    [Qq]* ) exit;;
    * ) echo "Please answer e for enable or d for disable hyperThreading.";;
  esac
done
```

bash AND list执行逻辑: ![在这里插入图片描述](https://img-blog.csdnimg.cn/1bae1816ca6f49259e090047eb65c908.png#pic_center)
bash  -e 参数逻辑：
![在这里插入图片描述](https://img-blog.csdnimg.cn/c386b8275f54482db1a8fbe5edc3974f.png#pic_center)

