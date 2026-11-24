MacBook Pro A1708（13寸2017无TouchBar）Ubuntu24.04 音频+蓝牙完整修复
硬件：声卡 Cirrus Logic CS8409（自带扬声器、麦克风）；无线蓝牙二合一芯片 BCM4350
Ubuntu24.04 默认内核6.8，自带PipeWire音频栈；原生内核蓝牙能识别、但容易断连；内置喇叭完全无声，必须装第三方DKMS声卡驱动
⚠️ 前置提醒：全程联网；建议先系统更新一遍
sudo apt update && sudo apt upgrade -y
sudo reboot
一、内置音频（喇叭+麦克风）驱动安装【重点】

使用 davidjo/snd_hda_macbookpro 这个成熟DKMS驱动，专门适配CS8409的MacBook 2016/2017（A1706/A1707/A1708）

1. 安装编译依赖、内核头文件
sudo apt install git dkms build-essential linux-headers-$(uname -r) linux-source -y
2. 拉取驱动源码
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro
3. 运行Cirrus声卡安装脚本
bash install.cirrus.driver.sh
如果提示缺少linux-source，按提示执行 sudo apt install linux-source-6.8.0（对应你当前内核版本），再重新运行脚本
4. DKMS注册模块
bash dkms.sh
5. 加载声卡驱动模块
sudo modprobe snd-hda-codec-cs8409
6. 重启音频服务（PipeWire）
systemctl --user stop pipewire wireplumber
systemctl --user start pipewire wireplumber
7. 重启电脑
sudo reboot
✅ 重启后：设置→声音里面会出现「MacBook Pro Speakers」，内置喇叭、内置麦克风可用
排查：如果仍然无声
# 查看声卡是否识别
aplay -l
# 查看模块是否加载
lsmod | grep cs8409
二、蓝牙驱动修复（BCM4350）

Ubuntu24.04主线内核自带brcmfmac固件，大部分情况蓝牙可以直接识别；问题是：配对耳机容易断、休眠唤醒蓝牙丢失、蓝牙音频A2DP不稳

方案1：基础修复（优先试）
# 安装蓝牙相关组件
sudo apt install bluez bluez-tools libspa-0.2-bluetooth -y
# 重启蓝牙服务
sudo systemctl restart bluetooth
# 重新加载模块
sudo modprobe -r btusb && sleep 2 && sudo modprobe btusb
打开设置-蓝牙，搜索配对设备；蓝牙音频默认PipeWire支持A2DP高清音乐模式。

方案2：如果蓝牙搜不到、固件报错（日志提示firmware load failed）

用macbook12-bluetooth-driver补全博通固件：
git clone https://github.com/leifliddy/macbook12-bluetooth-driver.git
cd macbook12-bluetooth-driver
sudo ./install_bluetooth.sh -i
sudo reboot
这个脚本自动拷贝BCM4350固件到 /lib/firmware/brcm，修复固件缺失问题
方案3：蓝牙耳机音质差、一直卡在通话HFP模式

1. 安装pavucontrol（音频配置面板）
sudo apt install pavucontrol
2. 终端运行 pavucontrol → 【配置】标签页，蓝牙耳机配置文件手动选择 A2DP Sink（高质量立体声）

3. 重启音频管道：
systemctl --user restart wireplumber pipewire
三、休眠唤醒蓝牙丢失 持久修复（A1708通病）

编辑蓝牙配置：
sudo nano /etc/bluetooth/main.conf
找到下面两行，修改/添加：
ControllerMode = le
AutoEnable=true
保存退出（nano：Ctrl+O回车保存，Ctrl+X退出）
sudo systemctl restart bluetooth
四、常用排查命令（出问题时用）
# 查看蓝牙日志
journalctl -u bluetooth -f
# 查看声卡日志
dmesg | grep -E 'snd|cs8409|audio'
# 查看pipewire状态
systemctl --user status pipewire
五、补充坑点说明（A1708专属）

1. 雷电3外接USB声卡、USB耳机不需要这个驱动，原生直接识别；上面只修机身自带喇叭麦

2. 升级内核之后，声卡DKMS模块可能需要重新跑一遍 bash dkms.sh 重新编译

3. Ubuntu24.04是PipeWire，不要强行装pulseaudio，会冲突

4. 无线WiFi（同BCM4350）：内核6.8+ brcmfmac原生驱动，一般不用额外驱动

六、一键卸载（以后升级系统或者不想用这个声卡驱动）
cd ~/snd_hda_macbookpro
bash uninstall.cirrus.driver.sh
sudo reboot
如果你想要，我可以顺便把A1708的键盘、触控板、屏幕亮度调节的Ubuntu24.04补丁一并给你。
