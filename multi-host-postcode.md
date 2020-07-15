# Multi-host postcode

Author:
  manikandan Elumalai, [manikandan.hcl.ers.epl@gmail.com](mailto:manikandan.hcl.ers.epl@gmail.com)

Primary assignee:
  
Other contributors:

Created:
  2020-06-18

## Problem Description

The current implementation in the phosphor-host-postd for postcode based on the LPC port 80 
connected b/w BMC and single host.

It is required to develop a new mechanism that would allow to read port 80 post code 
from multiple-host through BIC(Bridge IC) using IPMI protocol.

## Background and References

Facebook Yosemitev2 had an internal solution already for the problem and solution may to add 
into phosphor-host-postd and phosphor-post-code-manager. 

[OCP Debug Card with LCD Spec v1.0](http://files.opencompute.org/oc/public.php?service=files&t=4d86c4bcd365cd733ee1c4fa129bafca&download)

[fb-yv2-misc](https://github.com/HCLOpenBMC/fb-yv2-misc)

[fb-ipmi-oem](https://github.com/openbmc/fb-ipmi-oem)


 **phosphor-host-postd**

```ascii

+----------------------------------+                           +--------------------+
|  +-------------------------------+                           |                    |
|  |Phosphor-host-postd            |                           |                    |
|  |                    +----------+                           +------------+       |
|  |                    | LPC      |                           |            |       |
|  |                    |          +<--------------------------+            |       |
|  |                    +----------+                           |  LPC       |       |
|  |                               |                           |            |       |
|  |xyz.openbmc_project.State.     +<--------------------+     +------------+       |
|  |Boot.Raw.Value                 |                     |     |                    |
|  +------+------------------------+                     |     |         Host       |
|         |                        |                     |     |                    |
|         +                        |                     |     |                    |
|   postcode change event          |                     +     +--------------------+
|         +                        |  xyz.openbmc_project.State.Boot.Raw
|         |                        |                     +
|         v                        |                     |      +------------------+
|  +------+------------------------+                     +----->+                  |
|  |Phosphor-postcode-manager      |                            |   https          |
|  |                 +-------------+                            |                  |
|  |                 |   postcode  +<-------------------------->+                  |
|  |                 |   history   |                            |                  |
|  |                 +-------------+                            +------------------+
|  +-------------------------------+  xyz.openbmc_project.State.Boot.PostCode
|                                  |
|    BMC                           |
|  +-------------------------------+                           +----------------------+
|  |                               |                           |                      |
|  |     SGPIO                     +----GPIOs(8 line)  ------> |                      |
|  |                               |                           |     7 segment        |
|  +-------------------------------+                           |     Display          |
|                                  |                           |                      |
+----------------------------------+                           +----------------------+

```

The below device entry added in tiogapass DTS to create the LPC device(aspeed-lpc-snoop) in /dev

&lpc_snoop {
    status = "okay";
    snoop-ports = <0x80>;
};

GPIOs for 7 segment Display:
&gpio {
       status = "okay";
       gpio-line-names =
       /*H0-H7*/       "LED_POST_CODE_0","LED_POST_CODE_1","LED_POST_CODE_2",
                       "LED_POST_CODE_3","LED_POST_CODE_4","LED_POST_CODE_5",
                       "LED_POST_CODE_6","LED_POST_CODE_7";
}; 

snoopd daemon opens the lpc snoop character device( -lpc) and it owns the dbus object
whose value is the latest port 80h value.

**"/xyz/openbmc_project/state/boot/raw"**

These string constants are only used in this method within this object
and this object is the only object feeding into the final binary.

**"xyz.openbmc_project.State.Boot.Raw"**

If however, another object is added to this binary it would be proper
to move these declarations to be global and extern to the other object.

**root@tiogapass:~# busctl get-property xyz.openbmc_project.State.Boot.Raw /xyz/openbmc_project/state/boot/raw xyz.openbmc_project.State.Boot.Raw Value**

t 0

**phosphor-post-code-manager** 

This daemon [phosphor-post-code-manager](https://github.com/openbmc/phosphor-post-code-manager) monitors post code posted on dbus interface /xyz/openbmc_project/state/boot/raw by snoopd daemon [phosphor-host-postd](https://github.com/openbmc/phosphor-host-postd)
and save them in a file under /var/lib/phosphor-post-code-manager for history.

Every power cycle in host, post codes are saved as file in **/var/lib/phosphor-post-code-manager** on BMC.

The below files strored as Non-persistence storage in BMC.
 
  "1" file is saved all post codes for power cycle 1
  "2" file is saved all post codes for power cycle 2
  "3" file is saved all post codes for power cycle 3
   .                                         .
   .                                         .
  "100" file is saved all post codes for cycle 100

  "CurrentBootCycleIndex" file is saved the current boot cycle number
  "CurrentBootCycleCount" file is saved the current boot cycle count

BootCycleCount's max count is 100.

**root@tiogapass:~# ls -l  /var/lib/phosphor-post-code-manager/**

-rw-r--r--    1 root     root          1967 Jan  1 00:12 1
-rw-r--r--    1 root     root          2055 Jan  3 02:32 2
-rw-r--r--    1 root     root          1595 Jan  7 03:51 3
-rw-r--r--    1 root     root            19 Jan  7 03:51 CurrentBootCycleCount
-rw-r--r--    1 root     root            19 Jan  7 03:51 CurrentBootCycleIndex

**root@tiogapass:~# busctl call xyz.openbmc_project.State.Boot.PostCode /xyz/openbmc_project/State/Boot/PostCode xyz.openbmc_project.State.Boot.PostCode GetPostCodesWithTimeStamp q 1**

at 20 531933559797 5 531946666352 6 532003661083 183 532034668218 97 532036815235 154 532057582208 104 532068708677 121 532127904269 213 532132251059 151 532134938320 178 532138825772 156 532156771063 146 532157976249 192 532159246571 193 532190741768 173 532203686399 132 532244525662 132 532310008686 227 532310040008 0 532310095809 0

**root@tiogapass:~# busctl call xyz.openbmc_project.State.Boot.PostCode /xyz/openbmc_project/State/Boot/PostCode xyz.openbmc_project.State.Boot.PostCode GetPostCodes q 1**

at 20 5 6 183 97 154 104 121 213 151 178 156 146 192 193 173 132 132 227 0 0

**root@tiogapass:~# busctl call xyz.openbmc_project.State.Boot.PostCode /xyz/openbmc_project/State/Boot/PostCode xyz.openbmc_project.State.Boot.PostCode GetPostCodes q 2**

at 26 1 2 2 3 3 4 5 6 5 6 183 97 154 104 121 213 151 178 156 146 192 193 173 132 132 0

**root@tiogapass:~# busctl call xyz.openbmc_project.State.Boot.PostCode /xyz/openbmc_project/State/Boot/PostCode xyz.openbmc_project.State.Boot.PostCode GetPostCodes q 3**

at 26 2 1 2 3 4 5 6 4 5 6 183 97 154 104 121 213 151 178 156 146 192 193 173 132 132 0

## Requirements

 - Send POST code to 8-segment LED display on Debug Card.
 -  The POST code has to be from one of the server selected based on manual selection switch at front panel in debug card.
 - Provide a command for user to read current postcode.
 - POST code history.
 - Provide a command for user to see the all postcode for any given server.

## Proposed Design

The below module involved on proposed change.

fb-ipmi-oem.

fb-yv2-misc.

phosphor-post-code-manager.

```ascii

+----------------------------------------------+                       +------------------+
| +-----------------------------------+        +<-----+I2C/IPMI+------>+BIC||             |
| |       fb-ipmi-oem                 +<----+--+                       +----|     Host1   |
| |   (ipmi interrupt handler)        |     ^  |                       +------------------+
| +-----------------------------------+     |  |
| +-----------------------------------+     |  |                           +------------------+
| |     fb-yv2-misc                   |     <---<------+I2C/IPMI+--------->+BIC|--|           |
| |                                   |     |  |                           +------|   Host2   |
| | xyz.openbmc_project.              |     |  |                           +------------------+
| |  State.Hostx.Boot.Raw.Value       |     |  |
| +---------------+----------+--------+     |  |                                  +-------------------+
|  postcode       |          |        +     <---<------------+I2C/IPMI+---------->BIC|-|              |
|  event0         +          |   postcode   |  |                                  +----|  Host3       |
|    +         postcode      |   event3     |  |                                  +-------------------+
|    |         event1        +        +     |  |
|    |            +        postcode   |     |  |                                          +--------------------+
|    |            |        event2     |     |  |                                          +-----|   Host4      |
|    |            |          +        |     +---<---------------+I2C/IPMI+--------------->+BIC|-|              |
|    |            |          |        |        |                                          +--------------------+
|    v            v          v        v        |                                                        +--------------------+
|  +-+------------+----------+--------+---+    |    xyz.openbmc_project.State.Host0.Boot.Raw            |                    |
|  |                                      +<----<--+xyz.openbmc_project.State.Host1.Boot.Raw+---------->+                    |
|  |         +--------+                   |    |    xyz.openbmc_project.State.Host2.Boot.Raw            |                    |
|  |          history1                    |    |    xyz.openbmc_project.State.Host3.Boot.Raw            |                    |
|  |         +--------+        +--------+ |    |                                                        |  https             |
|  |                           | history3 |    |                                                        |                    |
|  +--------+                  +--------+ |    |    xyz.openbmc_project.State.Host0.Boot.PostCode       |                    |
|  |history0|                             +<----<--+xyz.openbmc_project.State.Host1.Boot.PostCode+----->+                    |
|  +--------+       +--------+            |    |    xyz.openbmc_project.State.Host2.Boot.PostCode       |                    |
|  |                 history2|            |    |    xyz.openbmc_project.State.Host3.Boot.PostCode       |                    |
|  |                +--------+            |    |                                                        +--------------------+
|  |                                      |    |           +--------------------+
|  | Phosphor-post-code-manager           |    ++GPIOs+--->+ 7 segment display  |
|  +--------------------------------------+    |           |                    |
+----------------------------------------------+           +--------------------+

```

##  fb-ipmi-oem

 - Register Bridge IC OEM callback interrupt handler for a postcode(cmd = 0x08, netfn=0x38, lun=00).
 - Extract port 80 data from IPMI response based on length.
 - Send extracted postcode to fb-yv2-misc.

## fb-yv2-misc

 - Get Bridge IC configuration(cmd = 0x0E, netfn=0x38, lun=00).
 - Set Bridge IC configuration(cmd = 0x10, netfn=0x38, lun=00).
 - Create, register and add dbus connection for "/xyz/openbmc_project/state/boot/raw".
 - Add "Value" property to store current postcode from hostX(X=1,2,3,4).
 - Read each hosts postcode data from fb-ipmi-oem postcode interrupt handler.
 - Read host position from debug card.
 - Display current post-code into the 7 segment display connected to GPIOs based on the host selection.
 - Generate postcode event to post-code-manager by update postcode into "Value" property.

## phosphor-post-code-manager
Change single process into a multi-process to handle multi-host postcode history.

## Alternatives Considered
Considered using to read post-code directly from Bridge IC under [fb-yv2-misc](https://github.com/HCLOpenBMC/fb-yv2-misc) instead of using [fb-ipmi-oem](https://github.com/openbmc/fb-ipmi-oem).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA3MTAwOTA5MiwtMTc4MTA4NjIzNV19
-->