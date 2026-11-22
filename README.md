# Hacking the ReoLink Argus Eco camera

I had an Argus Eco camera from 2020. It had fallen down and stopped charging via the solar panel.

When I brought it back inside it wouldn't connect to the network, so I did what anyone would do and factory reset it. 

Mistake 1 - it had actually connected to the network.

I deleted it from the app also, because clearly I wouldn't need to talk to the current setup any more.

Mistake 2 - theres useful info about your camera in the app.

Following the bouncing ball on the app, it asked to find it on the network. It wasn't on the network so I looked for other options.

Most were network related, but one had UID.

UID was now missing from the app, so I had a look on the camera body. The sticker had completely faded in the sun and it turns out that's what the UID is.

I remembered the camera used to go into a setup mode where you could show it a QR code but I couldn't find out where the QR code would come from in the app.

Turns out, it will show you the QR code, if you give it the UID.

Hunting online I found someone had already described what the format of the QR code had to be.

```
<QR><S>ssid</S><P>password</P><C>last4digitsofyourUID</C></QR>
```

if you encode that in ISO-8859-1 using something like https://qr.pratt.edu/index.php the camera will detect it and jump onto your network.
Except for the fact I had no UID.
So I did what anyone would do and tried 0000, 1234 etc. It would tell me it was invalid every time.

Ok now we dismantle the camera and look for the UID somewhere. It's not printed on the board, nor are any of the stickers related to a UID.

But, I found tx/rx pads next to the SOC. So I thought I'd hook it up to an ESP32 and read the console and obviously it would print out the UID for me.
You can do this with some code like this:

```
import sys
import uselect
import time
from machine import UART

uart = UART(2, baudrate=115200, tx=32, rx=33)

poller = uselect.poll()
poller.register(sys.stdin.buffer, uselect.POLLIN)

def flush_output():
    try:
        sys.stdout.buffer.flush()
    except:
        pass

print("UART console ready.")
flush_output()

while True:
    # Camera -> PC
    if uart.any():
        data = uart.read()
        if data:
            sys.stdout.buffer.write(data)
            flush_output()

    # PC -> Camera
    if poller.poll(0):
        data = sys.stdin.buffer.read(1)
        if data:
            uart.write(data)

    time.sleep_ms(1)
```

Pic of the console location here:


I bashed at the console for a long time. In setup mode you get a command line, where you can run stuff like `ls`, `cat` etc.
You'd get things like the following:
```
cat version_file
cat version_file1202_491_352_19

~# cat service.cfg

<service_config_t>
    <mod0 mod0="0" />
    <mod1 mod1="0" />
    <mod2 mod2="1" />
    <mod3 mod3="1" />
    <mod4 mod4="1" />
    <mod5 mod5="0" />
    <mod6 mod6="1" />
    <web port="80" https_port="443" />
    <upnp enable="0" />
    <ipfilter type="0" white0="" black0="" white1="" black1="" white2="" black2="" white3="" black3="" white4="" black4="" white5="" black5="" white6="" black6="" white7="" black7="" white8="" black8="" white9="" black9="" white10="" black10="" white11="" black11="" white12="" black12="" white13="" black13="" white14="" black14="" white15="" black15="" white16="" black16="" white17="" black17="" white18="" black18="" white19="" black19="" white20="" black20="" white21="" black21="" white22="" black22="" white23="" black23="" white24="" black24="" white25="" black25="" white26="" black26="" white27="" black27="" white28="" black28="" white29="" black29="" white30="" black30="" white31="" black31="" white32="" black32="" white33="" black33="" white34="" black34="" white35="" black35="" white36="" black36="" white37="" black37="" white38="" black38="" white39="" black39="" white40="" black40="" white41="" black41="" white42="" black42="" white43="" black43="" white44="" black44="" white45="" black45="" white46="" black46="" white47="" black47="" white48="" black48="" white49="" black49="" white50="" black50="" white51="" black51="" white52="" black52="" white53="" black53="" white54="" black54="" white55="" black55="" white56="" black56="" white57="" black57="" white58="" black58="" white59="" black59="" white60="" black60="" white61="" black61="" white62="" black62="" white63="" black63="" />
    <surv port="9000" />
    <ddns isp="3" url="" name="" pass="" />
    <ntp auto="0" timeout="43200" servername="pool.ntp.org" port="123" />
    <email battachment="1" bssl="1" name="" pass="" smtp="smtp.gmail.com" smtpport="465" sender="" recver0="" recver1="" recver2="" />
    <mobile quality="5" framerate="1" bitrate="16" />
    <onvif port="8000" />
    <rtsp port="554" />
    <bcp2p domain="p2p.reolink.com" port="9999" />
</service_config_t>
```

But no UID in any file I could find. I started looking for a way to dump the firmware to see if I could extract it, but it only offered ways to do things like tftp etc but once it was actually connected to a network. I couldn't manually set SSID etc from here either, so I was stuck.

I had pretty much given up at this point, barring dumping the firmware straight from the flash. But the last thing that came to mind was maybe it would print something useful when I tried scanning a QR code.
So I gave it a crappy QR code in the same format as above, that was known incorrect. Console showed the following:
```
**************2d code xml string*******************
<QR><S>ssid</S><P>password</P><C>3513</C></QR>
***************************************************
uid_check err uid_check = 1
m_stoppts[channel][STREAM_TYPE_MAIN] = 5002560000
m_stoppts[channel][STREAM_TYPE_SUB] = 5002426000
flash_fs_deinit succ!
system shutdown !
```

Turns out next time it boots, it prints the damn UID straight to the screen.

```
libstaticparameter Version[GM8136S_FREERTOS]:2018-03-30 16:20:06 by yzt
ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ
<?xml version="1.0" encoding="utf-8"?>
<StaticParameter>
<RfType>0</RfType>
<TmpMacAddr>ec:71:ff:ff:ff:ff</TmpMacAddr>
<Uid>95270001XXXXXXXX</Uid>
<P2Pkey>XXXX</P2Pkey>
</StaticParameter>
TTTTTTTTTTTTTTTTTTTTTTTTTTTTT
Sp_init[RfType] = [0]
Sp_init[TmpMacAddr] = [ec:71:ff:ff:ff:ff]
Sp_init[Uid] = [95270001XXXXXXXX]
Sp_init[P2Pkey] = [XXXX]
sar_adc_init
```

Making a new QR code with the same format, subbing in my last 4 chars of the UID, and my correct wifi details, worked first time and I was back in!

Next step is dumping the firmware, because stuffed if I'm letting someone else with the same worn off sticker problem and a lack of circuit board probe kit have a useless camera.
