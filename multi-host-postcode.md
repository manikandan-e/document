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


The below component diagram shows the present implementation for postcode and history at high-level overview
```ascii
+--------------------------------------------+
|                                       BMC  |
|                                            |
|  +-----------------+  +-----------------+  |
|  |  Host Discovery |  | OEM Specific    |  |
|  | (through        |  | Functions       |  |
|  |  inventory &    |  |                 |  |
|  |  hotplug        |  | (fb-ipmi-oem)   |  |
|  |  events)        |  |                 |  |                          +----+-------------+
|  |                 |  |                 |  |  +<-----+I2C/IPMI+------>+BIC |             |
|  +-----------+-----+  +--------+--------+  |  |                       |    |     Host1   |
|              |                 |           |  |                       +------------------+
|  +-----------v-----------------v--------+  |  |                          +------------------+
|  |phosphor-ipmi-host/phosphor-ipmi-ipmb +<-----<-------+I2C/IPMI+------->+BIC |             |
|  |         (interrupt handler)          |  |  |                          |    |     Host2   |
|  |xyz.openbmc_project.Misc.Ipmi.Update  |  |  |                          +------------------+
|  +-----------+--------------------------+  |  |                             +------------------+
|              +                             |  +<---------+I2C/IPMI+-------->+BIC  |            |
|            event                           |  |                             |     |    Host3   |
|              +                             |  |                             +------------------+
|              |                             |  |                                +-------------------+
|  +-----------v------------------------+    |  +<------------+I2C/IPMI+-------->+    |              |
|  |      fb-yv2-misc                   |    |                                   |BIC |     Host4    |
|  |                                    <-----------------------------------+    +----+--------------+
|  |   xyz.openbmc_project.State.       |    |                              |
|  |   HostX(0,1,2,3).Boot.Raw.Value    <--------------------+              |
|  +-+-------------+---------+--------+-+    |               |              |
|    |             |         |        |      |               |              |
|   event0         |         +        |      |               |              |
|    |          event1   event2       +      |  +------------v----------+   |
|    |             |         +     event3    |  | OCP Debug card        |   |
|    |             v         |        +      |  |(7 segment display &   |   |
|  +-v-----------------------v--------v---+  |  | Host selection switch)|   |
|  |         +--------+                   |  |  +-----------------------+   |
|  |          history1                    |  |                              |
|  |         +--------+        +--------+ |  |                              |
|  |                           | history3 |  |                              |  +---------------------+
|  +--------+                  +--------+ |  |                              +->+                     |
|  |history0|                             |  |                                 |    Command Line     |
|  +--------+       +--------+            <-----+xyz.openbmc_project.State.+-->+    Interface        |
|  |                 history2|            |  |   HostX(0,1,2,3).Boot.PostCode  |                     |
|  |                +--------+            |  |                                 +---------------------+
|  |                                      |  |
|  | Phosphor+post+code+manager           |  |
|  ++ +-----------------------------------+  |
+--------------------------------------------+

```

## Requirements

 - Read postcode from all servers.
 - Display given host postcode to 7 segment display based host position in debug card.
 - Provide a command interface for user to see any server current postcode .
 - Provide a command interface for user to see any server postcode history.

## Proposed Design

It is required to develop a new mechanism that would allow to read port 80 post code 
for multiple-host through BIC(Bridge IC) using IPMI protocol.

The below module involved on proposed design change.

 - fb-ipmi-oem.
 - fb-yv2-misc.
 - phosphor-post-code-manager.
 - phosphor-dbus-interfaces

```ascii
+--------------------------------------------+
|                                       BMC  |
|                                            |
|  +-----------------+  +-----------------+  |
|  |  Host Discovery |  | OEM Specific    |  |
|  | (through        |  | Functions       |  |
|  |  inventory &    |  |                 |  |
|  |  hotplug        |  | (fb-ipmi-oem)   |  |
|  |  events)        |  |                 |  |                          +----+-------------+
|  |                 |  |                 |  |  +<-----+I2C/IPMI+------>+BIC |             |
|  +-----------+-----+  +--------+--------+  |  |                       |    |     Host1   |
|              |                 |           |  |                       +------------------+
|  +-----------v-----------------v--------+  |  |                          +------------------+
|  |phosphor-ipmi-host/phosphor-ipmi-ipmb +<-----<-------+I2C/IPMI+------->+BIC |             |
|  |         (interrupt handler)          |  |  |                          |    |     Host2   |
|  |xyz.openbmc_project.Misc.Ipmi.Update  |  |  |                          +------------------+
|  +-----------+--------------------------+  |  |                             +------------------+
|              +                             |  +<---------+I2C/IPMI+-------->+BIC  |            |
|            event                           |  |                             |     |    Host3   |
|              +                             |  |                             +------------------+
|              |                             |  |                                +-------------------+
|  +-----------v------------------------+    |  +<------------+I2C/IPMI+-------->+    |              |
|  |      fb-yv2-misc                   |    |                                   |BIC |     Host4    |
|  |                                    +<----------------------------------+    +----+--------------+
|  |   xyz.openbmc_project.State.       |    |                              |
|  |   HostX(0,1,2,3).Boot.Raw.Value    +<-------------------+              |
|  +-------------+-----------+--------+-+    |               +              |
|   event0       +           |        |      |             GPIOs            |
|    |        event1         +        |      |               +              |
|    |           +       event2       +      |  +-----------+V+---------+   |
|    |           |           +     event3    |  | OCP Debug card        |   |
|    |           |           |        +      |  |(7 segment display &   |   |
|  +-v-----------------------v--------v---+  |  | Host selection switch)|   |
|  |         +--------+                   |  |  +-----------------------+   |
|  |          history1                    |  |                              |
|  |         +--------+        +--------+ |  |                              |
|  |                           | history3 |  |                              |  +---------------------+
|  +--------+                  +--------+ |  |                              +->+                     |
|  |history0|                             |  |                                 |    Command Line     |
|  +--------+       +--------+            <-----+xyz.openbmc_project.State.+-->+    Interface        |
|  |                 history2|            |  |   HostX(0,1,2,3).Boot.PostCode  |                     |
|  |                +--------+            |  |                                 +---------------------+
|  |                                      |  |
|  | phosphor-post-code-manager           |  |
|  ++ +-----------------------------------+  |
+--------------------------------------------+


```

##  fb-ipmi-oem

This library is part of [phosphor-ipmi-host](https://github.com/openbmc/phosphor-host-ipmid) and get the postcode  from host through [phosphor-ipmi-ipmb](https://github.com/openbmc/ipmbbridge).

 - Register new Bridge IC(BIC) OEM callback interrupt handler for a postcode(cmd = 0x08, netfn=0x38, lun=00).
 - Extract port 80 data from IPMI response based on length.
 - Send extracted postcode to fb-yv2-misc by D-bus callback method registered in the fb-yv2-misc(xyz.openbmc_project.Misc.Ipmi.Update).
 
## fb-yv2-misc

 This is new process going create as part of the openbmc/meta-facebook to handle Facebook platform specific feature.
 
 - Get Bridge IC(BIC) configuration(cmd = 0x0E, netfn=0x38, lun=00).
 - Set Bridge IC(BIC) configuration(cmd = 0x10, netfn=0x38, lun=00).
 - Create, register and add dbus connection for "/xyz/openbmc_project/hostX/state/boot/raw".
 - Add "Value" property to store current postcode from hostX(X=0,1,2,3).
 - Read each hosts postcode data from fb-ipmi-oem postcode interrupt handler.
 - Generate postcode event to post-code-manager by update postcode into "Value" D-bus property(xyz.openbmc_project.State.HostX.Boot.Raw.Value).
 - Read host position from debug card.
 - Display current post-code into the 7 segment display connected to GPIOs based on the host selection in the plug-able debug card.
 
 **D-Bus interface**
 - xyz.openbmc_project.State.Host0.Boot.Raw.Value
 - xyz.openbmc_project.State.Host1.Boot.Raw.Value
 - xyz.openbmc_project.State.Host2.Boot.Raw.Value
 - xyz.openbmc_project.State.Host3.Boot.Raw.Value

## phosphor-post-code-manager
The design shall handle the hot plugged multi-host in the single process phosphor-post-code-manager based on host discovery.

The below D-Bus interface needs to created for multi-host post-code history.

**D-Bus interface**
 - xyz.openbmc_project.State.Host0.Boot.PostCode
 - xyz.openbmc_project.State.Host1.Boot.PostCode
 - xyz.openbmc_project.State.Host2.Boot.PostCode
 - xyz.openbmc_project.State.Host3.Boot.PostCode
 
## phosphor-dbus-interfaces
The new YAML file needs to create to handle the Facebook platform specific implementation .

The example as below,

description: >

    Implement to provide D-bus interface for Facebook specific implementation.
    
properties:

    - name: hostPosition
    
      type: uint16
      
      description: >
      
          hostPosition indicates the current host position selected in OCP debug card.
          
methods:

    - name: update
    
      description: >
      
          Method to get the cached post codes.
          
      parameters:
      
        - name: postcode
        
          type: uint16
          
          description: >
          
              postcode indicates which host postcode.
              
        - name: host
        
          type: uint16
          
          description: >
          
              postcode indicates which host postcode. 

**D-Bus interface**

- xyz.openbmc_project.Misc.Ipmi.Update
- xyz.openbmc_project.Misc.Ipmi.Postcode
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTc3NDMyNjU4XX0=
-->