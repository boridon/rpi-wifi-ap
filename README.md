# Raspberry Pi Wifi Access Point #

USB Wifiアダプタを２つ挿してアクセスポイントを作ります

Raspbian jessie lite(2015-11-21 kernel 4.1.13)  
GW-900D (IEEE802.11ac 5GHz)  
GW-USEco300 (IEEE802.11n 2.4GHz)  
hostapd (アクセスポイントのソフトウェア)  

## raspi-configでSDカードの領域を最大に拡張 ##

    sudo raspi-config

## パッケージのアップデート ##

	sudo apt-get update
	sudo apt-get dist-upgrade
	sudo apt-get install rpi-update git gcc make bc screen ncurses-dev

## カーネル、ファームウェア更新 ##

	sudo rpi-update
	sudo reboot

## カーネルのリビルド ##

下記はRaspberry Pi 2用です。  
それ以外の場合は Module7 や kernel7 から 7 を抜いたファイル名で行います

	cd /usr/src
	sudo su
	git clone --depth 1 https://github.com/raspberrypi/linux.git -b rpi-4.1.y
	git clone --depth 1 https://github.com/raspberrypi/firmware.git
	cd linux
	modprobe configs
	zcat /proc/config.gz > .config
	cp ../firmware/extra/Module7.symvers Module.symvers
	make oldconfig
	make -j 4 zImage modules dtbs
	make modules_install
	cp arch/arm/boot/dts/*.dtb /boot/
	cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
	cp arch/arm/boot/dts/overlays/README /boot/overlays/
	cp /boot/kernel7.img /boot/kernel7.img.old
	scripts/mkknlimg arch/arm/boot/zImage /boot/kernel7.img
	reboot

## GW-900Dのドライバーをビルド ##

rtl8812au チップのドライバーをビルドして使用します  
いくつかRaspberry Pi用に設定を変えてビルドします

	git clone --depth 1 https://github.com/diederikdehaas/rtl8812AU.git -b driver-4.3.22-beta
	cd rtl8812AU

### Makefile を修正します ##

    --- a/Makefile
    +++ b/Makefile
    @@ -85,8 +85,8 @@
    ######### Notify SDIO Host Keep Power During Syspend ##########
    CONFIG_RTW_SDIO_PM_KEEP_POWER = y
    ###################### Platform Related #######################
    -CONFIG_PLATFORM_I386_PC = y
    -CONFIG_PLATFORM_ARM_RPI = n
    +CONFIG_PLATFORM_I386_PC = n
    +CONFIG_PLATFORM_ARM_RPI = y
    CONFIG_PLATFORM_ANDROID_X86 = n
    CONFIG_PLATFORM_ANDROID_INTEL_X86 = n
    CONFIG_PLATFORM_JB_X86 = n

### パッチ ##

上記のdiffを保存してパッチ実行する場合 (diff.patch)

	patch -p1 < diff.patch

### ビルド＆インストール ###

	make
	sudo make install
	sudo modprobe 8812au

## hostapdのインストール ##

### パッケージのインストール ###

	sudo apt-get install bridge-utils hostapd

### GW-USEco300 で動く hostapd をビルドして上書き ###

ビルドに必要なパッケージ

	sudo apt-get install libnl1 libnl-dev libssl-dev

hostapdのソースを取得

	wget http://w1.fi/releases/hostapd-2.5.tar.gz

展開

	tar xf hostapd-2.5.tar.gz

パッチを取得

	git clone --depth 1 https://github.com/pritambaral/hostapd-rtl871xdrv.git -b hostapd_2_5

パッチ

	cd hostapd-2.5
	patch -Np1 -i ../hostapd-rtl871xdrv/rtlxdrv.patch
	cd hostapd
	cp defconfig .config
	echo CONFIG_DRIVER_RTW=y >> .config
	echo CONFIG_IEEE80211N=y >> .config
	echo CONFIG_IEEE80211AC=y >> .config

ビルド

	make

パッケージバイナリを上書き

	sudo cp hostapd hostapd_cli /usr/sbin

## ネットワークインターフェースの設定 ##

### /etc/udev/rules.d/99-wifi.rules

デバイス名を接続順ではなく、任意になるよう設定します  
※MACアドレスを指定します

	SUBSYSTEM=="net", DRIVERS=="?*", ATTR{address}=="00:22:cf:xx:xx:xx", NAME="wlan0"
	SUBSYSTEM=="net", DRIVERS=="?*", ATTR{address}=="00:22:cf:xx:xx:xx", NAME="wlan1"

### /etc/network/interfaces ###

※eth0 のMACアドレスを指定します

	auto lo
	iface lo inet loopback

	iface eth0 inet manual

	allow-hotplug wlan0
	iface wlan0 inet manual

	allow-hotplug wlan1
	iface wlan1 inet manual

	auto br0
	iface br0 inet dhcp
	hwaddress b8:27:eb:xx:xx:xx
	bridge_ports eth0 wlan0 wlan1

## hostapdの設定 ##

２つのインターフェースそれぞれにconfを用意します

### /etc/hostapd/hostapd-wlan0.conf ###

ssid、wpa_passphrase の箇所は任意に書き換えます

	interface=wlan0
	bridge=br0
	driver=nl80211
	logger_syslog=-1
	logger_syslog_level=2
	logger_stdout=-1
	logger_stdout_level=2
	ctrl_interface=/var/run/hostapd
	ctrl_interface_group=0
	ssid=test0
	country_code=JP
	ieee80211d=1
	ieee80211h=1
	hw_mode=a
	channel=40
	beacon_int=100
	dtim_period=2
	max_num_sta=255
	rts_threshold=2347
	fragm_threshold=2346
	macaddr_acl=0
	auth_algs=3
	ignore_broadcast_ssid=0
	wmm_enabled=1
	wmm_ac_bk_cwmin=4
	wmm_ac_bk_cwmax=10
	wmm_ac_bk_aifs=7
	wmm_ac_bk_txop_limit=0
	wmm_ac_bk_acm=0
	wmm_ac_be_aifs=3
	wmm_ac_be_cwmin=4
	wmm_ac_be_cwmax=10
	wmm_ac_be_txop_limit=0
	wmm_ac_be_acm=0
	wmm_ac_vi_aifs=2
	wmm_ac_vi_cwmin=3
	wmm_ac_vi_cwmax=4
	wmm_ac_vi_txop_limit=94
	wmm_ac_vi_acm=0
	wmm_ac_vo_aifs=2
	wmm_ac_vo_cwmin=2
	wmm_ac_vo_cwmax=3
	wmm_ac_vo_txop_limit=47
	wmm_ac_vo_acm=0
	ieee80211n=1
	eapol_key_index_workaround=0
	eap_server=0
	own_ip_addr=127.0.0.1
	wpa=2
	wpa_passphrase=test0
	wpa_key_mgmt=WPA-PSK WPA-PSK-SHA256
	wpa_pairwise=CCMP
	rsn_pairwise=CCMP

### /etc/hostapd/hostapd-wlan1.conf ###

ssid、wpa_passphrase の箇所は任意に書き換えます

	interface=wlan1
	bridge=br0
	driver=rtl871xdrv
	logger_syslog=-1
	logger_syslog_level=2
	logger_stdout=-1
	logger_stdout_level=2
	ctrl_interface=/var/run/hostapd
	ctrl_interface_group=0
	ssid=test1
	country_code=JP
	ieee80211d=1
	ieee80211h=1
	hw_mode=g
	channel=1
	beacon_int=100
	dtim_period=2
	max_num_sta=255
	rts_threshold=2347
	fragm_threshold=2346
	macaddr_acl=0
	auth_algs=3
	ignore_broadcast_ssid=0
	wmm_enabled=1
	wmm_ac_bk_cwmin=4
	wmm_ac_bk_cwmax=10
	wmm_ac_bk_aifs=7
	wmm_ac_bk_txop_limit=0
	wmm_ac_bk_acm=0
	wmm_ac_be_aifs=3
	wmm_ac_be_cwmin=4
	wmm_ac_be_cwmax=10
	wmm_ac_be_txop_limit=0
	wmm_ac_be_acm=0
	wmm_ac_vi_aifs=2
	wmm_ac_vi_cwmin=3
	wmm_ac_vi_cwmax=4
	wmm_ac_vi_txop_limit=94
	wmm_ac_vi_acm=0
	wmm_ac_vo_aifs=2
	wmm_ac_vo_cwmin=2
	wmm_ac_vo_cwmax=3
	wmm_ac_vo_txop_limit=47
	wmm_ac_vo_acm=0
	ieee80211n=1
	eapol_key_index_workaround=0
	eap_server=0
	own_ip_addr=127.0.0.1
	wpa=2
	wpa_passphrase=test1
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=TKIP
	rsn_pairwise=CCMP

### /etc/default/hostapd

自動起動のパラメータで２つのconfを指定します

	DAEMON_CONF="/etc/hostapd/hostapd-wlan0.conf /etc/hostapd/hostapd-wlan1.conf"

## GW-USEco300 の省電力をオフ ##

8192cu チップは省電力モードになると通信が途切れたりすることがあるそうなのでオフに

### /etc/modprobe.d/8192cu-disable-power-management.conf

ファイルを新規作成します  

	options 8192cu rtw_power_mgnt=0 rtw_enusbss=0

## 再起動 ##

	sudo reboot


