## m5stackで時計を作る
技術書展6で見つけた本を元に作ってみた

### 見つけた
m5stackで何か作ってみようと思っていたがなかなか手がつかなかった．
ふらっと行った技術書展6で見つけた
[PB Make部 ](https://techbookfest.org/event/tbf06/circle/71750007)
の本を元にNTPサーバと同期して時刻表示するプログラムを作ってみた．

デバッグする過程でwifiから割り当てられたdnsサーバの値を書き換えたり，
wait時の処理を追加したのが変更点．

```
from m5stack import lcd
import time
import machine
import network

SSID    = "SSID"
PASS    = "PASS"
DNS     = "8.8.8.8"
TIMEOUT = 10

def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    for ap in wlan.scan():
        ssid = ap[0].decode('utf-8')
        if ssid == SSID:
            lcd.println('Connecting to ' + SSID)
            wlan.connect(SSID, PASS)
            timer = time.time()
            while not wlan.isconnected():
                lcd.print('.')
                if time.time() - timer > TIMEOUT:
                    break
                time.sleep(1)
            if wlan.isconnected():
                lcd.println('\nSuccess to connect')
                config = list(wlan.ifconfig())
                config[-1] = DNS
                wlan.ifconfig(config)
                return True
            else:
                lcd.println('Failed to connect')
                return False
    return False

def setup_rtc():
    lcd.println("Syncing RTC")
    rtc = machine.RTC()
    rtc.ntp_sync(server = "ntp.jst.mfeed.ad.jp", tz = "JST-9")
    timer = time.time()
    while not rtc.synced():
        lcd.print('.')
        if time.time() - timer > TIMEOUT:
            break
        time.sleep(1)
    lcd.println("\nSynced RTC.")

connect_wifi()
setup_rtc()
lcd.clear()
lcd.setTextColor(lcd.ORANGE, lcd.BLACK)
lcd.font(lcd.FONT_7seg, fixedwidth = True, dist = 16, width = 2)
while True:
    d = time.strftime("%Y-%m-%d", time.localtime())
    t = time.strftime("%H:%M:%S", time.localtime())
    lcd.print(d, lcd.CENTER, 50)
    lcd.print(t, lcd.CENTER, 130)
    time.sleep(1)
```

