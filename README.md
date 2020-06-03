# Jetson Nano  


* [Jetson Nano](#Jetson-Nano)  
  - [硬體](#硬體)  
  - [系統設定](#系統設定)  
  - [系統架構](#系統架構)  
  - [程式語言](#程式語言)  
  - [開發環境](#開發環境)  
  - [下載整包開發專案](#下載整包開發專案)  
  - [可能會使用框架](#可能會使用框架)  
  - [辨識模型](#辨識模型)  
  - [轉送端(測試)](#轉送端(測試))  
 


硬體  
---
1. 網卡  
   - [Intel® Wi-Fi 6 AX200 160MHz ](https://www.intel.com.tw/content/www/tw/zh/support/articles/000005511/network-and-i-o/wireless-networking.html)  
### [網卡購買處](https://goods.ruten.com.tw/item/show?21917116648893)

2. 相機(i2c)  
   - [IMX219-77 Camera](http://www.waveshare.net/wiki/IMX219-77_Camera)  
   - [IMX219-77IR Camera (具有紅外線)](http://www.waveshare.net/wiki/IMX219-77IR_Camera)  
   - [IMX219-120 Camera](http://www.waveshare.net/wiki/IMX219-120_Camera)  
   - [IMX219-160 Camera](http://www.waveshare.net/wiki/IMX219-160_Camera)  
   - [IMX219-160IR Camera (具有紅外線)](http://www.waveshare.net/wiki/IMX219-160IR_Camera)  
   - [IMX219-200 Camera](http://www.waveshare.net/wiki/IMX219-200_Camera)  
3. 記憶卡
   - [SanDisk Extreme microSDXC UHS-I(V30)(A2) 64GB](https://24h.pchome.com.tw/prod/DGAG5E-1900AK5JP?fq=/S/DGAG0H)
      - 讀/寫高達 60MB/s and 160MB/s


系統設定
---
- 以下的設定皆Base on Kernel 4.9
- [x]   1. SD Card Image 下載 
     - [IMAGE 下載處](https://developer.nvidia.com/embedded/jetpack)
- [x]   2. SWAP 設定
     - ```console=
       sudo git clone https://github.com/JetsonHacksNano/resizeSwapMemory && cd resizeSwapMemory && cd sudo sh setSwapMemorySize.sh -g 8 
       sudo echo "vm.swappiness = 1" >> /etc/sysctl.conf
       ```
- [x]   3. Intel® Wi-Fi 6 AX200 驅動   
    - :bulb: 可使用**兩種**方法驅動該網卡  
      1. Kernel 4.9 更新至 Kernel 5.1 (此方法嘗試過,沒有成功) 
      2. 反向移植(Backport)  [參考網站1](https://forums.untangle.com/installation/41963-success-uefi-problem-wi-fi-6-intel-ax200.html)、[參考網站2](https://hant.kutu66.com/ubuntu/article_185392)  
    
     -  下載相關檔案
        :::info
        :bulb:至[Intel Linux Wireless](https://wireless.wiki.kernel.org/en/users/drivers/iwlwifi)下載如下.tgz檔案 
         Intel® Wi-Fi 6 AX200 160MHz 5.1+ **iwlwifi-cc-46.3cfab8da.0.tgz**
        :::
        - ```console=   
          cd /lib/firmware
          sudo wget https://git.kernel.org/cgit/linux/kernel/git/iwlwifi/linux-firmware.git/plain/iwlwifi-cc.a0.48.ucode
          sudo wget https://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/linux-firmware.git/plain/iwlwifi-cc-a0-46.ucode
          sudo wget https://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/linux-firmware.git/plain/iwlwifi-cc-a0-48.ucode
          sudo wget https://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/linux-firmware.git/plain/iwlwifi-cc-a0-50.ucode
          cd /temp
          tar iwlwifi-cc-46.3cfab8da.0.tgz
          cp iwlwifi-*.ucode /lib/firmware 
          git clone https://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/backport-iwlwifi.git 
          ``` 
    - 編譯
        :::info 
        進入下剛剛下載的backport-iwlwifi 
        :::
      - ```console=         
        cd backport-iwlwifi
        make defconfig-iwlwifi-public sed -i ‘s/CPTCFG_IWLMVM_VENDOR_CMDS=y/# CPTCFG_IWLMVM_VENDOR_CMDS is not set/’ .config make -j4
        sudo make install
        ```
        :::spoiler
        安裝後會將編譯的moudle(.ko)檔案放置如下資料夾
        /lib/modules/4.9.140-tegra/updates/drivers/net/wireless/intel/iwlwifi
        注意！並不會取代原先(下方)的moudle位置
        /lib/modules/4.9.140-tegra/kernel/drivers/net/wireless/intel/iwlwifi
        :::  
    
    - 測試掛載module
      - ```console=   
        lsmod
        ``` 
        尚未掛載前，並無wifi模組
        ```
        Module                 Size       Used      by
        zram                   26166      4     
        overlay                48691      0     
        spidev                 13282      0     
        nvgpu                  1575721    3     
        bluedroid_pm           13912      0     
        ip_tables              19441      0     
        x_tables               28951      1         ip_tables
        ```
    
      - ```console= 
        sudo modprobe iwlwifi
        lsmod
        ``` 
        手動掛載後，確認wifi模組是否有掛載成功
        ```
        Module                Size         Used     by
        iwlwifi               444497       0
        cfg80211              777921       1        iwlwifi
        compat                122316       2        iwlwifi,cfg80211
        zram                  26166        4
        overlay               48691        0
        spidev                13282        0
        nvgpu                 1575721      3
        bluedroid_pm          13912        0
        ip_tables             19441        0
        x_tables              28951        1        ip_tables
        ```

    - 取消其他wifi模組自動掛載
      - ```console= 
        echo "blacklist rtl8192cu" | sudo tee -a /etc/modprobe.d/blacklist.conf
        ```      
    - 自動掛載模組(重新開機會自動掛起 )
      - ```console=   
        cd /etc/modules-load.d
        sudo vim modules.conf
            #新增如下
        #wifi 6 ,intel AX200 
        iwlwifi
        ```  
      
    - 觀察wifi狀態與速度
      - ```console=
        狀態
        ifconfig
        關閉wifi 
        sudo ifconfig wlan0 down
        開啟wifi
        sudo ifconfig wlan0 up
        觀察速度        
        iw dev wlan0 link
        ```
- [x]   4. Jemalloc Source install 
     - ```console=
       wget https://github.com/jemalloc/jemalloc/archive/5.2.1.tar.gz
       tar zxvf 5.2.1.tar.gz && cd jemalloc-5.2.1/ 
       sudo sh autogen.sh && ./configure --prefix=/usr/local/jemalloc
       sudo make -j8 && sudo make install 
       ```
       
- [x]   5. CSI攝影機測試
     - ```console=
       gst-launch-1.0 nvarguscamerasrc ! 'video/x-raw(memory:NVMM),width=1920, height=1080, framerate=30/1, format=NV12' ! nvvidconv flip-method=2 ! nvoverlaysink
       ```
     
:::info 
- 裡面包含Jetpack等API Download the Jetson Nano Developer Kit SD Card image.
- Jemalloc 確保記憶體有效利用,現在裝的版本5.2.1(目前最新)，可在這找到更新或更舊的版本 https://github.com/jemalloc/jemalloc/releases  
:::  

架構
---
- 整體架構

![](https://github.com/gun123kk/IOT_jetson_nano_doc/blob/master/png/Pic1.png?raw=true)

- 辨識架構

![](https://github.com/gun123kk/IOT_jetson_nano_doc/blob/master/png/Pic2.png?raw=true)



程式語言
---
1. [Python](https://www.python.org/downloads/) 
2. [Node-js](https://nodejs.org/)
3. [React Native](https://reactnative.dev/)

開發環境
---
1. JupyterHub(Use JupyterLab by Default)
   - 安裝NodeJS & npm
     - ```console=
       curl -sL https://deb.nodesource.com/setup_13.x -o nodesource_setup.sh #setup_13.x 按版本所需做調整
       sudo bash nodesource_setup.sh
       sudo apt install nodejs npm 
       ``` 
   - 安裝configurable-http-proxy
     - ```console=
       npm install -g configurable-http-proxy 
       ```
   - 安裝Jupyter Lab & Jupyter Hub & SudosPawner
     - ```console=
       pip3 install jupyterlab jupyterhub sudospawner
       ```
   - 新增專用群組
     - ```console=
       sudo groupadd jupyterhub
       sudo groupadd shadow
       ``` 
   - 設定PAM
     - ```console=
       sudo chgrp shadow /etc/shadow
       sudo chmod g+r /etc/shadow
       sudo usermod -a -G shadow rhea
       sudo setcap 'cap_net_bind_service=+ep' /usr/bin/node
       ``` 
   - 設定 /etc/sudoers
     - ```console=
       sudo nano /etc/sudoers
       ``` 
     - ```
       alien           ALL=(ALL:ALL) ALL
       ivan_yu         ALL=(ALL:ALL) ALL
       philip_chiang   ALL=(ALL:ALL) ALL
       Cmnd_Alias JUPYTER_CMD = /usr/local/bin/sudospawner
       alien ALL=(%jupyterhub) NOPASSWD:JUPYTER_CMD  
       ```
   - 產生Jupyter Hub設置檔
     - ```console=
       sudo mkdir -p /etc/jupyterhub 
       cd /etc/jupyterhub
       sudo chown ${USER}
       sudo -u ${USER} LD_PRELOAD=/usr/local/jemalloc/lib/libjemalloc.so /usr/local/bin/jupyterhub --generate-config
       ```
   - 設定jupyterhub_config.py
     - ```console=
       sudo nano jupyterhub_config.py
       ``` 
     - ```
       c.Spawner.default_url = '/lab'
       c.JupyterHub.ip = '0.0.0.0'
       c.JupyterHub.spawner_class = 'sudospawner.SudoSpawner'
       c.JupyterHub.port = 8282
       c.Authenticator.whitelist =  {'alien','ivan_yu','philip_chiang'}
       ``` 
   - 新增用戶
     - ```console=
       sudo adduser alien
       sudo usermod -a -G jupyterhub alien
       sudo reboot
       ```
   - 啟動JupyterHub
     - ```console=
       sudo -u ${USER} LD_PRELOAD=/usr/local/jemalloc/lib/libjemalloc.so /usr/local/bin/jupyterhub -f /etc/jupyterhub/jupyterhub_config.py
       ``` 

下載整包開發專案
---
1. Download  
   - ```console=
     ssh 下載（需要在github有設定public ssh key才能使用）
     repo init -u git@github.com:gun123kk/IOT_jetson_nano_manifest.git -b master -q
     https下載（無設定key也能使用）
     repo init -u https://github.com/gun123kk/IOT_jetson_nano_manifest.git -b master
     repo sync -q
     repo start master --all
     ```
    
2. 若無法使用repo指令  
   - 請至[github repoitory 網址](https://github.com/gun123kk/IOT_jetson_nano_manifest)對要得資料夾做git clone
   - repoitories名單
     - [IOT_jetson_nano_doc](https://github.com/gun123kk/IOT_jetson_nano_doc)
     - [IOT_jetson_nano_OS_setting](https://github.com/gun123kk/IOT_jetson_nano_OS_setting)
     - [IOT_jetson_nano_ReactNative](https://github.com/gun123kk/IOT_jetson_nano_ReactNative)
     - [IOT_jetson_nano_NN](https://github.com/gun123kk/IOT_jetson_nano_NN)
     - [IOT_jetson_nano_redis](https://github.com/gun123kk/IOT_jetson_nano_redis)
     - [IOT_jetson_nano_test](https://github.com/gun123kk/IOT_jetson_nano_test)

3. 將各自管理的repoitory裡的commit推至github  
   - ```console=
     git add 修改的檔案
     git commit
     git push
     ```
4. 若要在repo上新增repoitory，請修改IOT_jetson_nano_manifest的default.xml文件  
    - ```console=
      git add default.xml
      git commit
      git push origin HEAD:master
      ``` 
      
轉送端(測試)
---
- Pub/Sub:
  - 測試主機：140.109.22.214:8282(2 GPU Host)
    - 測試檔案路徑：/home/alien/public/ivan_yu/pub_sub_example
  - 環境：Mqtt(Apache ActiveMQ)
    - 測試Lang： Python3.8
    - 測試IDE：Jupyterlab
    - 操作：照片reSize --> 照片轉Bytes
    - 狀態:Pub of Json (Ok! 但需要 reSize),Sub(無法動作) 
  - 環境：Redis(Version 6)
    - 測試Lang： Python3.8
    - 測試IDE：Jupyterlab
    - 操作：照片轉Bytes -->在轉Base64
    - 狀態：Pub/Sub 都正常
- Streaming:
  - 測試主機：140.109.22.214:8282(2 GPU Host) & Jetson nano 
    - 環境：Redis(Version 6)
    - 測試Lang： Python3.8
    - 測試IDE：Jupyterlab
    - 操作：照片轉Bytes --> Join Json Format --> xADD(傳送) --> xREVRANGE(接收) --> Bytes轉照片



     
使用框架(是否安裝完畢)
---
- [x] 1. [TensorFlow](https://www.tensorflow.org/api_docs/python/tf_overview) 
- [x] 2. [TensorFlow-Lite](https://www.tensorflow.org/api_docs/python/tf/lite)
- [x] 3. [Pytorch](https://forums.developer.nvidia.com/t/pytorch-for-jetson-nano-version-1-4-0-now-available/72048#5324123)
- [x] 4. [OpenCV4.1.1(CUDA)](https://www.jetsonhacks.com/2019/11/22/opencv-4-cuda-on-jetson-nano/)


辨識模型(是否測試完畢)
---
- [x] 1. [Yolo_V3-Tiny](https://pjreddie.com/darknet/yolo/) 不採納
- [x] 2. [EfficientNet(神經網絡)]不採納(https://github.com/tensorflow/tpu/tree/master/models/official/efficientnet/lite)  
- [x] 3. [EfficientDet(物件偵測器)]不採納(https://github.com/toandaominh1997/EfficientDet.Pytorch)

