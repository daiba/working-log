## m5stack でmicropython
m5stackでmicropythonを使って見るための作業履歴

### 初期設定
手順は[ここ](https://github.com/m5stack/M5Cloud/blob/master/README_JP.md)を参照

1. [ダウンロードページ](https://github.com/m5stack/M5Cloud/tree/master/firmwares)から[OFFLINE用ファームウェア](https://github.com/m5stack/M5Cloud/blob/master/firmwares/OFF-LINE/m5stack-20180516-v0.4.0.bin)ダウンロード
2. mac用[ドライバ](https://jp.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers#mac) Silicon Labs CP210x VCP ダウンロード
3. MacOS

```
$ pip install --upgrade pip
$ pip search esptool
 esptool (2.4.1)  - A serial utility to communicate & flash code to Espressif
                   ESP8266 & ESP32 chips.
$ pip install esptool
$ esptool.py --chip esp32 --port /dev/tty.SLAB_USBtoUART erase_flash
esptool.py v2.4.1
Serial port /dev/tty.SLAB_USBtoUART
Connecting........_
Chip is ESP32D0WDQ6 (revision 1)
Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse
MAC: b4:e6:2d:8b:a3:79
Uploading stub...
Running stub...
Stub running...
Erasing flash (this may take a while)...
Chip erase completed successfully in 10.8s
Hard resetting via RTS pin...
$ esptool.py --chip esp32 --port /dev/tty.SLAB_USBtoUART write_flash --flash_mode dio -z 0x1000 m5stack-20180516-v0.4.0.bin 
esptool.py v2.4.1
Serial port /dev/tty.SLAB_USBtoUART
Connecting........___
Chip is ESP32D0WDQ6 (revision 1)
Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse
MAC: b4:e6:2d:8b:a3:79
Uploading stub...
Running stub...
Stub running...
Configuring flash size...
Auto-detected Flash size: 4MB
Compressed 1747296 bytes to 1119059...
Wrote 1747296 bytes (1119059 compressed) at 0x00001000 in 98.7 seconds (effective 141.6 kbit/s)...
Hash of data verified.

Leaving...
Hard resetting via RTS pin...
$ screen /devtty.SLAB_USBtoUART 115200
>>> print('hello world')
hello world
>>> from m5stack import lcd
>>> lcd.clear()
>>> lcd.print('hello')
>>>
 ```
 
   

### 参考
+ [M5Stack](https://github.com/m5stack)
+ [M5Stack_MicroPython](https://github.com/m5stack/M5Stack_MicroPython)
+ [M5Stack Arduino](http://www.m5stack.com/assets/docs/index.html)
+ [M5Cloud](https://github.com/m5stack/M5Cloud)
+ [M5Stackでセンサーデータを測定し，クラウドに送る（MicroPython編）](http://pages.switch-science.com/letsiot/m5stack_micropython/index.html)
+ [温度湿度気圧センサモジュールキット AE-BME280](https://qiita.com/n-yamanaka/items/7d6305a326e27158db2e)
