## circuitpython調査
Adafruit Feather M0 Adaloggerをcircuitpythonで使うための作業履歴

### はじめに
購入して最初にusb接続したところ，LEDが2個点滅していた．
ひとつはD#13相当，もうひとつは電源用（と思われる）．
Arduino IDEからはアクセスできたがディスクマウントは
行われなかった．IDEにはadafruitのテンプレートを追加し，
2個のフォーマットを設定した．

最新のbossa 1.8を使ってファームアップロードを行おうとしたが
失敗．SAM-BA failedというメッセージがでた．1.7にダウングレード
したところファームウェアアップロード成功．使ったファームウェアは
[adafruit-circuitpython-feather_m0_adalogger-2.2.4.bin](https://github.com/adafruit/circuitpython/releases/tag/2.2.4)．
これにより，CIRCUITPYという名前のフォルダで見えるようになった．
このフォルダの中身は
`boot_out.txt` で中身は以下の通り
```
Adafruit CircuitPython 2.2.4 on 2018-03-07; Adafruit Feather M0 Adalogger with samd21g18
```
本来はこのフォルダ内に `boot.py` と `main.py` があるはずらしい．
デフォルトの `boot.py` は[こんな](http://micropython.org/resources/fresh-pyboard/boot.py)感じ
```
# boot.py -- run on boot-up
# can run arbitrary Python, but best to keep it minimal

import pyb
#pyb.main('main.py') # main script to run after this one
#pyb.usb_mode('CDC+MSC') # act as a serial and a storage device
#pyb.usb_mode('CDC+HID') # act as a serial device and a mouse
```
HIB keyboardを[使う](http://wiki.micropython.org/USB-HID-Keyboard-mode-example-a-password-dongle)のならこうなる
```
#boot.py
import machine  <-  このパッケージは見つからない
import pyb
#pyb.main('main.py') # main script to run after this one
#pyb.usb_mode('CDC+MSC') # act as a serial and a storage device
pyb.usb_mode('CDC+HID',hid=pyb.hid_keyboard)
```

ちなみにキー入力を実施する `main.py` は以下のようになる
```
import pyb
kb = pyb.USB_HID()
buf = bytearray(8)
# Sending T
# Do the key down
buf[0] = 0x02 # LEFT_SHIFT
buf[2] = 0x17 # keycode for 't/T'
kb.send(buf)
pyb.delay(5)
# Do the key up
buf[0] = 0x00
buf[2] = 0x00
kb.send(buf)
```

ampyを使うと，ローカルにあるpythonスクリプトをボード上で[実行できる](https://learn.adafruit.com/micropython-basics-load-files-and-run-code/run-code)．
```
$ cat test.py
print('Hello world! I can count to 10:')
for i in range(1,11):
    print(i)
$ ampy --port /dev/cu.usbmodem1411 run test.py
...
```

#### サイト情報
* [USB HID Keyboard mode example a password dongle](http://wiki.micropython.org/USB-HID-Keyboard-mode-example-a-password-dongle)
* [Turn your ProMicro into a USB Keyboard/Mouse](https://www.sparkfun.com/tutorials/337)


#### コマンド
* [ampy](https://github.com/adafruit/ampy)

```
$ ampy --help
Usage: ampy [OPTIONS] COMMAND [ARGS]...

  ampy - Adafruit MicroPython Tool

  Ampy is a tool to control MicroPython boards over a serial connection.
  Using ampy you can manipulate files on the board's internal filesystem and
  even run scripts.

Options:
  -p, --port PORT    Name of serial port for connected board.  Can optionally
                     specify with AMPY_PORT environemnt variable.  [required]
  -b, --baud BAUD    Baud rate for the serial connection (default 115200).
                     Can optionally specify with AMPY_BAUD environment
                     variable.
  -d, --delay DELAY  Delay in seconds before entering RAW MODE (default 0).
                     Can optionally specify with AMPY_DELAY environment
                     variable.
  --version          Show the version and exit.
  --help             Show this message and exit.

Commands:
  get    Retrieve a file from the board.
  ls     List contents of a directory on the board.
  mkdir  Create a directory on the board.
  put    Put a file or folder and its contents on the...
  reset  Perform soft reset/reboot of the board.
  rm     Remove a file from the board.
  rmdir  Forcefully remove a folder and all its...
  run    Run a script and print its output.
```

### CircuitPython
[ドキュメント](https://circuitpython.readthedocs.io/en/2.x/docs/index.html#)から調査