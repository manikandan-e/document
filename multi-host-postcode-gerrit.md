# Multi-host Postcode Support

Author: Manikandan Elumalai, [manikandan.hcl.ers.epl@gmail.com](mailto:manikandan.hcl.ers.epl@gmail.com)

Other contributors: None

Created: 2020-07-02

## Problem Description

The current implementation in the phosphor-host-postd supports only single host postcode access through LPC interface. Hence, multiple-host are not supported by mechanism.

## Background and References

[OCP Debug Card with LCD Spec v1.0](http://files.opencompute.org/oc/public.php?service=files&t=4d86c4bcd365cd733ee1c4fa129bafca&download)

[fb-ipmi-oem](https://github.com/openbmc/fb-ipmi-oem)

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
 - Display the host postcode to the 7 segment display based on host position selected in the debug card.
 - Provide a command interface for user to see any server(multi-host) current postcode .
 - Provide a command interface for user to see any server(multi-host) postcode history.
 - Support for hot-plug-able host.

## Proposed Design

This document proposes a new design engaging the IPMB interface to read port-80 post code from multiple-host. This design also supports host discovery including the hot-plug-able host connected in slot.

Following modules will updated for this implementation

 - phosphor-host-postd.
 - phosphor-post-code-manager.
 - fb-ipmi-oem.
 - fb-yv2-misc.
 - phosphor-dbus-interfaces.

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

 - Register postcode callback interrupt handler(cmd = 0x08, netfn=0x38, lun=00) to read postcode.
 - Extract postcode from IPMB  message (phosphor-ipm-host/phosphor-ipmi-ipmb).
 - Send extracted postcode to phosphor-host-postd.
 
## phosphor-host-postd

**Host discovery**
      This feature adds to detect when hot plug-able host connected in the slot.
      Postcode D-bus interface needs to create based on host present(Host Field replaceable Unit D-bus interface ).
      
 - Create, register and add dbus connection for "/xyz/openbmc_project/hostX/state/boot/raw" based on Host discovery as mentioned above..
 - Add "Value" property to store current postcode from hostX(0,1,2.N).
 - Read each hosts postcode from fb-ipmi-oem postcode interrupt handler.
 - Send event to post-code-manager based on which host's postcode received from IPMB interface(xyz.openbmc_project.State.HostX.Boot.Raw.Value) 
 - Read host position from dbus property (debug card).
 - Display current post-code into the 7 segment display connected to BMC's 8 GPIOs based on the host position.
 
 **D-Bus interface**
 - xyz.openbmc_project.State.Host0.Boot.Raw.Value
 - xyz.openbmc_project.State.Host1.Boot.Raw.Value
 - xyz.openbmc_project.State.Host2.Boot.Raw.Value
 - xyz.openbmc_project.State.HostN.Boot.Raw.Value

## phosphor-post-code-manager

phosphor-post-code-manager is the single process based on host discovery for multi-host. This design shall not affect single host for post-code.

- Create, register and add the dbus connection for "xyz.openbmc_project.State.Hostx(0,1,2.N).Boot.PostCode based on Host discovery.
- Store/retrieve post-code(/var/lib/phosphor-post-code-manager/hostX(0,1,2.N))  based on event received from phosphor-host-postd.
- 
The below D-Bus interface needs to created for multi-host post-code history.

**D-Bus interface**
 - xyz.openbmc_project.State.Host0.Boot.PostCode
 - xyz.openbmc_project.State.Host1.Boot.PostCode
 - xyz.openbmc_project.State.Host2.Boot.PostCode
 - xyz.openbmc_project.State.HostN.Boot.PostCode
 
## fb-yv2-misc

The below operation part of the fb-yv2-misc.

 - Enable the post code in each host connected through IPMB interface.
 - Detect and send the host position switch position to phosphor-post-code-manager through D-bus. 
 
## phosphor-dbus-interfaces

   All multi-host postcode related property and method need to create.

## Alternate design

 **phosphor-post-code-manager**
       Change single process into multi-process  on phosphor-post-code-manager.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTE0ODkwNjA0MiwtMTg0OTEyMTU1Myw1MD
QwODU4MzEsMTk0OTM2NjI1MCwtMTU1MzI5NzM5NSwtOTU4MDIy
MTcyLC03MzE1NjY1NjAsLTE1MDQwOTE3MTIsMjA3OTA4MTM5Ni
wxODk3MTM3ODQwLDE4MDA4NDM2NDcsOTE2MjEwMTMsLTQxMDYy
Nzg0MiwxMDk3NTYyMDMxLDg0NzQ2NTYyOSwtMTIxMDcyMTM0NS
wxNTgxMTAwMzE1LDIwNzQ5NDc1MjcsMTg5MTg1NDcyNCw1NTMw
ODE3NV19
-->