# Introduction

An case that BIOS VBT table direct i915 driver to get EDID from an incorrect i2c port (for DDC protocal)

Refer to https://gitlab.freedesktop.org/drm/intel/-/issues/2577
That machine get dark screen in the first place, then force the i915 reading EDID from i2c-2 to i2c-3 workarounded it.


# Steps to debug:
## Check the connector

The operations are done on a machine which already corrected this issue.

```
u@ubuntu:/sys/class/drm$ for i in `ls | grep ^card0-`;do echo $i:;  ls $i ; echo "status=" "$(cat $i/status)"; done
card0-DP-1:
device  dpms  drm_dp_aux0  edid  enabled  i2c-9  modes  power  status  subsystem  uevent
status= disconnected
card0-HDMI-A-1:
ddc  device  dpms  edid  enabled  i2c-1  modes  power  status  subsystem  uevent
status= disconnected
card0-HDMI-A-2:
ddc  device  dpms  edid  enabled  i2c-3  modes  power  status  subsystem  uevent
status= connected
card0-HDMI-A-3:
ddc  device  dpms  edid  enabled  i2c-4  modes  power  status  subsystem  uevent
status= connected
```

It says:
 - Panel is connected to card0-HDMI-A-2
 - The DDC is connected to i2c-3

Or this information also shows on dmesg with enabled `drm.debug=0xf` 
```
i915 0000:00:02.0: [drm:intel_modeset_readout_hw_state [i915]] [CONNECTOR:183:HDMI-A-2] hw state readout: enabled
```

But things weird that card0-HDMI-A-2/edid shows nothing.
So, it could caused the currupted patch reading EDID from internal panel.

## Check if i2c slave replied as expected

```
u@ubuntu:~$ i2cdetect -l
i2c-3   unknown         i915 gmbus tc1                          N/A
i2c-1   unknown         i915 gmbus dpb                          N/A
i2c-8   unknown         i915 gmbus tc6                          N/A
i2c-6   unknown         i915 gmbus tc4                          N/A
i2c-4   unknown         i915 gmbus tc2                          N/A
i2c-2   unknown         i915 gmbus dpc                          N/A
i2c-0   unknown         i915 gmbus dpa                          N/A
i2c-9   unknown         AUX B/port B                            N/A
i2c-7   unknown         i915 gmbus tc5                          N/A
i2c-5   unknown         i915 gmbus tc3                          N/A
```

Check the incorrect i2c-2 which shows in card0-HDMI-A-2 originally. It shows no slaves there.
```
u@ubuntu:/sys/class/drm/card0-HDMI-A-2$ sudo i2cdetect -y -a -r 2
```
Check each i2c bus and find i2c-3 responses something.
So, it says there are 5 i2c slaves there which are 0x3A, 0x40, 0x49, 0x50 and 0x59
```
u@ubuntu:/sys/class/drm/card0-HDMI-A-2$ sudo i2cdetect -y -a -r 3
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- 3a -- -- -- -- -- 
40: 40 -- -- -- -- -- -- -- -- 49 -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- 59 -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
```

Check slaves one by one, then find slave 0x50 response EDID.
It matches [header format of EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) 

```
u@ubuntu:/sys/class/drm/card0-HDMI-A-2$ sudo i2cdump -f -y -a 3 0x50
No size specified (using byte-data access)
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f    0123456789abcdef
00: 00 ff ff ff ff ff ff 00 10 ac 09 94 96 21 21 04    ........?????!!?
10: 23 1b 01 04 a5 34 1d 78 0a c5 a5 a7 56 52 9e 26    #????4?x????VR?&
20: 10 50 54 21 08 00 d1 c0 81 c0 81 80 95 00 b3 00    ?PT!?.???????.?.
30: 01 01 01 01 01 01 d0 39 80 18 71 38 2d 40 70 38    ???????9??q8-@p8
.... [skip] ...
```

Now we know `i2c-2 i915 gmbus dpc` is the wrong one, and `i2c-3 i915 gmbus tc1` likly the correct one instead.

## Check VBT
Because VBT definds the internal panel be connected to which i2c port (for DDC protocal), so let's check VBT.
```
$ intel_vbt_decode /sys/kernel/dbug/dri/0/i915_vbt
```
from the [result](https://gitlab.freedesktop.org/drm/intel/uploads/95fcd181d7eaf71300bae8b4f236176c/vbt_rkl_decode.log)
It shows 3 child devices from DDC pin: 0x02, 0x03 and 0x04
And HDMI related words are associated to the child devices with DDC pin: 0x03 and 0x04, so likely 2 of them introduced i2c-2.  
And likely the incorrect pin mapping in VBT confused i915 driver to pick incorrect `i2c-2 i915 gmbus dpc` instead of `i2c-3 i915 gmbus tc1`.

### Reference
https://github.com/freedesktop/drm-tip/blob/drm-tip/drivers/gpu/drm/i915/display/intel_bios.c
```
[TGL_DDC_BUS_DDI_C] = GMBUS_PIN_3_BXT,
```

Then we check code under drm-tip/drivers/gpu/drm/i915/display
```
intel_gmbus.h:25:#define GMBUS_PIN_3_BXT                3
```

https://github.com/freedesktop/drm-tip/blob/drm-tip/drivers/gpu/drm/i915/display/intel_gmbus.h
```
#define GMBUS_PIN_DPC		4 /* HDMIC */
#define GMBUS_PIN_9_TC1_ICP	9
```

Then try to hard code change gmbus ping from 4 to 9 (because it's associted to `tc1`)

Then we can see the EDID content presents in /sys/class/drm/edid
