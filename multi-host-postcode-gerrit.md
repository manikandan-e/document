# Multi-host Postcode Support

Author: Manikandan Elumalai, [manikandan.hcl.ers.epl@gmail.com](mailto:manikandan.hcl.ers.epl@gmail.com)

Other contributors: None

Created: 2020-07-02

## Problem Description

The current implementation in the phosphor-host-postd supports only single host postcode access through LPC interface. Hence, multiple-host are not supported by mechanism.

## Background and References

Facebook Yosemitev2 had an internal solution already for the problem and solution may to add 
into phosphor-host-postd and phosphor-post-code-manager. 

[OCP Debug Card with LCD Spec v1.0](http://files.opencompute.org/oc/public.php?service=files&t=4d86c4bcd365cd733ee1c4fa129bafca&download)

[fb-yv2-misc](https://github.com/HCLOpenBMC/fb-yv2-misc)

[fb-ipmi-oem](https://github.com/openbmc/fb-ipmi-oem)

BIC(Bridge IC)
  The Bridge IC plays as a bridge between BMC and the host systemo which is part removable host. The host system can be managed by the BMC through Bridge IC even if host hardware or OS hang or goes down


The below component diagram shows the present implementation for postcode and history at high-level overview
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
|  |Phosphor-postcode-manager      |                            |   CLI            |
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

## Requirements

 - Read postcode from all servers.
 - Display given host postcode to 7 segment display based host position in debug card.
 - Provide a command interface for user to see any server current postcode .
 - Provide a command interface for user to see any server postcode history.
 - Support for hotplug host
 - Dynamic host management

## Proposed Design

This document proposes a new design engaging the IPMB interface to read port-80 post code from multiple-host through BIC (Bridge IC). This design also supports host discovery including the hot-plug feature.

Following modules will updated for this implementation

 - phosphor-host-postd.
 - phosphor-post-code-manager.
 - phosphor-dbus-interfaces.
 - fb-ipmi-oem.
 - fb-yv2-misc.

**Interface Diagram**
```ascii
+-------------------------------------------+
|                      +-----------------+  |
|                      | (fb-ipmi-oem)   |  |
|     BMC              |                 |  |                          +----+-------------+
|                      +--------+--------+  |       +-+I2C/IPMI+------>+BIC |             |
|                               |           |       |                  |    |     Host1   |
| +-----------------------------v--------+  |       |                  +------------------+
| |                                      |  |       |                     +------------------+
| | phosphor-ipm-host/phosphor-ipmi-ipmb +<-------------+I2C/IPMI+------->+BIC |             |
| |         (interrupt handler           |  |       |                     |    |     Host2   |
| |xyz.openbmc_project.Misc.Ipmi.Update  |  |       |                     +------------------+
| +--+-----------------------------------+  |       |                        +------------------+
|    |            +----------------------+  |       +-----+I2C/IPMI+-------->+BIC  |            |
|    |       +----+    fb-yv2-misc       <---------->                        |     |    Host3   |
|  event     |    |(postcode enable)     |  |       |                        +------------------+
|    +       |    |(platfrom specific)   |  |       |                           +-------------------+
|    |       |    |                      <-------+  +--------+I2C/IPMI+-------->+    |              |
|    |       |    +----------------------+  |    |                              |BIC |     HostN    |
|    |    host pos                          |    |                              +----+--------------+
|    |       |                              |    |        +-------------------+
| +--v-------v-------------------------+    |    +GPIOs---+ host postcode     |
| |phosphor-post-code-manager          |    |             | selection switch  |
| |   (ipmi snoop)                     +---------+        |                   |
| | xyz.openbmc_project.State.         <------+  |        +-------------------+
| | HostX(0,1,2.N).Boot.Raw.Value      |    | |  |
| +-----------------------------+------+    | |  |
|                               |           | |  |        +--------------------+
|                          postcode event   | |  |        |  7 segment         |
|                               |           | |  +GPIOs-->+   Display          |
| +--------------------------------------+  | |           +--------------------+
| | +-------------+     +-------v------+ |  | |
| | |Inventory &  |     |              | |  | |                               +---------------------+
| | |hotplug      |     |Histroy(1,2,3.N)|  | +------------------------------->                     |
| | |             |     |              + |  |                                 |    Command Line     |
| | |             |     |              +<------+xyz.openbmc_project.State.+--->    Interface        |
| | +-------------+     +--------------+ |  |   HostX(0,1,2..N).Boot.PostCode |                     |
| |                                      |  |                                 +---------------------+
| | Phosphor-post-code-manager           |  |
| +--------------------------------------+  |
+-------------------------------------------+

```

##  fb-ipmi-oem

This library is part of [phosphor-ipmi-host](https://github.com/openbmc/phosphor-host-ipmid) and get the postcode  from host through [phosphor-ipmi-ipmb](https://github.com/openbmc/ipmbbridge).

 - Register new Bridge IC(BIC) OEM callback interrupt handler for a postcode(cmd = 0x08, netfn=0x38, lun=00).
 - Extract port 80 data from IPMIBresponse based on length.
 - Send extracted postcode to phosphor-host-postd by D-bus callback method registered in the phosphor-host-postd(xyz.openbmc_project.Misc.Ipmi.Update).
 
## phosphor-host-postd

 This is new process going create as part of the openbmc/meta-facebook to handle Facebook platform specific feature.
 
 - Create, register and add dbus connection for "/xyz/openbmc_project/hostX/state/boot/raw".
 - Add "Value" property to store current postcode from hostX(X=0,1,2.N).
 - Read each hosts postcode data from fb-ipmi-oem postcode interrupt handler.
 - Send event to post-code-manager based on which host's postcode received from IPMB interface(xyz.openbmc_project.State.HostX.Boot.Raw.Value) 
 - Read host position from dbus property (debug card).
 - Display current post-code into the 7 segment display connected to BMC's 8 GPIOs based on the host position.
 
 **D-Bus interface**
 - xyz.openbmc_project.State.Host0.Boot.Raw.Value
 - xyz.openbmc_project.State.Host1.Boot.Raw.Value
 - xyz.openbmc_project.State.Host2.Boot.Raw.Value
 - xyz.openbmc_project.State.HostN.Boot.Raw.Value

## phosphor-post-code-manager
The design shall handle the hot plugged multi-host in the single process phosphor-post-code-manager based on host discovery.

- Create, register and add dbus connection for "xyz.openbmc_project.State.Hostx(0,1,2.N).Boot.PostCode.
- store 

**Host discovery**

The below D-Bus interface needs to created for multi-host post-code history.

**D-Bus interface**
 - xyz.openbmc_project.State.Host0.Boot.PostCode
 - xyz.openbmc_project.State.Host1.Boot.PostCode
 - xyz.openbmc_project.State.Host2.Boot.PostCode
 - xyz.openbmc_project.State.HostN.Boot.PostCode
 - 
## fb-yv2-misc

The below operation part of the fb-yv2-misc.

 - Enable post code in each host connected through IPMB interface.
 - Detect and send the host position switch position to phosphor-post-code-manager through D-bus. 
 
## phosphor-dbus-interfaces

   D-bus interface need to create to support for multi-host postcode.

## Alternate design

 **phosphor-post-code-manager**
       change single process into multi-process on phosphor-post-code-manager.
 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2ODk0Nzc4NiwtMTU1MzI5NzM5NSwtOT
U4MDIyMTcyLC03MzE1NjY1NjAsLTE1MDQwOTE3MTIsMjA3OTA4
MTM5NiwxODk3MTM3ODQwLDE4MDA4NDM2NDcsOTE2MjEwMTMsLT
QxMDYyNzg0MiwxMDk3NTYyMDMxLDg0NzQ2NTYyOSwtMTIxMDcy
MTM0NSwxNTgxMTAwMzE1LDIwNzQ5NDc1MjcsMTg5MTg1NDcyNC
w1NTMwODE3NSw1Nzc0MzI2NTgsODc5OTY0NzI5LDEyNTUxOTA5
ODFdfQ==
-->