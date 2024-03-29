# Multi-host Postcode Support

Author: Manikandan Elumalai, [manikandan.hcl.ers.epl@gmail.com]

Other contributors: None

Created: 2020-07-02

## Problem Description

The current implementation in the phosphor-host-postd supports only single host1
postcode access through LPC interface.

As the open BMC architecture is evolving, the single host support becomes
contingent and needs multiple-host post code access to be implemented.

## Background and References

**Existing postcode implementation for single host**

The below component diagram shows the present implementation for postcode and
history at high-level overview
```
+----------------------------------+              +--------------------+
|BMC                               |              |                    |
|  +-------------------------------+              |                    |
|  |Phosphor-host-postd            |              |                    |
|  |                    +----------+              +------------+       |
|  |                    | LPC      |              |            |       |
|  |                    |          +<-------------+            |       |
|  |                    +----------+              |  LPC       |       |
|  |                               |              |            |       |
|  |xyz.openbmc_project.State.     +<-------+     +------------+       |
|  |Boot.Raw.Value                 |        |     |                    |
|  +------+------------------------+        |     |         Host       |
|         |                        |        |     |                    |
|         +                        |        |     |                    |
|   postcode change event          |        +     +--------------------+
|         +                        |  xyz.openbmc_project.State.Boot.Raw
|         |                        |        +
|         v                        |        |      +------------------+
|  +------+------------------------+        +----->+                  |
|  |Phosphor-postcode-manager      |               |   CLI            |
|  |                 +-------------+               |                  |
|  |                 |   postcode  +<------------->+                  |
|  |                 |   history   |               |                  |
|  |                 +-------------+               +------------------+
|  +-------------------------------+  xyz.openbmc_project.State.Boot.PostCode
|                                  |
|                                  |
|  +-------------------------------+             +----------------------+
|  |                               |  8GPIOs     |                      |
|  |     SGPIO                     +-----------> |                      |
|  |                               |             |     7 segment        |
|  +-------------------------------+             |     Display          |
|                                  |             |                      |
+----------------------------------+             +----------------------+
```
## Requirements

 - Read postcode from all servers.
 - Display the host postcode to the 7 segment display based on host position
   selection.
 - Provide a command interface for user to see any server(multi-host) current
   postcode.
 - Provide a command interface for user to see any server(multi-host) postcode
   history.
 - Support for hot-plug-able host.

## Proposed Design

This document proposes a new design engaging the IPMB interface to read the
port-80 post code from multiple-host. The existing single host LPC interface
remains unaffected. This design also supports host discovery including the
hot-plug-able host connected in the slot.

Following modules will be updated for this implementation

 - phosphor-host-postd.
 - phosphor-post-code-manager.
 - platform specific OEM handler (fb-ipmi-oem).
 - phosphor-dbus-interfaces.
 - bmcweb(redfish logging service).

**Interface Diagram**

Provided below the post code interface diagram with flow sequence
```
+-------------------------------------------+
|                  BMC                      |
|                                           |
| +--------------+     +-----------------+  |    I2C/IPMI  +----+-------------+
| |              |     |                 |  |  +----------->BIC |             |
| |              |     |   ipmbbridged   <-----+           |    |     Host1   |
| |              |     |                 |  |  |           +------------------+
| | oem handlers |     +-------+---------+  |  | I2C/IPMI  +------------------+
| |              |             |            |  +----------->BIC |             |
| |              |             |            |  |           |    |     Host2   |
| |              |     +-------v---------+  |  |           +------------------+
| | (fb-ipmi-oem)|     |                 |  |  | I2C/IPMI  +------------------+
| |              <-----+     ipmid       |  |  +----------->BIC |             |
| |              |     |                 |  |  |           |    |     Host3   |
| +------+-------+     +-----------------+  |  |           +------------------+
|        |                                  |  | I2C/IPMI  +------------------+
|        | postcode    +-----------------+  |  +----------->BIC |             |
|        |             | Host position   |  |              |    |     HostN   |
|        |             |  from D-Bus     |  |              +----+-------------+
|        |             +-------+---------+  |
|        |                     |            |             +-----------------+
|        |                     |            |             |                 |
| +------v---------------------v---------+  |             | seven segment   |
| |phosphor-host-postd                   +----------------> display         |
| |(ipmisnoop)                           <----+           |                 |
| |xyz.openbmc_project.State.            |  | |           |                 |
| |Boot.RawX(0,1,2,..N).Value            |  | |           +-----------------+
| +--------------------------------------+  | |
|                 postcode event|           | |  xyz.openbmc_project.
|                               |           | |  State.Boot.
| +--------------------------------------+  | |  PostcodeX(0,1,2..N)  +-----+
| | +----------------+  +-------v------+ |  | |                       |     |
| | |                |  |              | |  | +----------------------->     |
| | |  Process1      |  |process N     | |  |                         | CLI |
| | |   (host1)      +-->  (hostN)     | |  |                         |     |
| | |                |  |              <------------------------------>     |
| | +----------------+  +--------------+ |  | /redfish/v1/Systems/    |     |
| |                                      |  | system/LogServices/     +-----+
| | Phosphor-post-code-manager**         |  | PostCodesX(0,1,2..N)
| +--------------------------------------+  |
+-------------------------------------------+

** Incase of the single host, process1 only run.

```

**Postcode Flow:**

 - BMC power-on the Host.
 - Host starts sending the postcode IPMI message continuously to the BMC
 - The ipmbbridged(phosphor-ipmi-ipmb) extracts the postcode from IPMI message
 - The ipmbd(phosphor-ipmi-host) appends host information with postcode and
   sends to the phosphor-host-postd.
 - phosphor-host-postd sends postcode events to the phosphor-post-code-manager
   and displays postcode in the seven segment display.
 - phosphor-post-code-manager stores the postcode as history in the /var
   directory.

##  Platform Specific OEM Handler (fb-ipmi-oem)

This library is part of  the [phosphor-ipmi-host]
(https://github.com/openbmc/phosphor-host-ipmid)
and get the postcode  from host through
[phosphor-ipmi-ipmb](https://github.com/openbmc/ipmbbridge).

 - Register IPMI OEM postcode callback handler.
 - Extract postcode from IPMI message (phosphor-ipmi-host/phosphor-ipmi-ipmb).
 - The phosphor-ipmi-host sends extracted postcode to the phosphor-host-postd.

## phosphor-host-postd

The implementation involves the following changes in the phosphor-host-postd.

 - Create D-bus names for hosts..
 - Read each hosts postcode from Platform Specific OEM ServicesIPMI OEM handler
   (fb-ipmi-oem, intel-ipmi-oem,etc).
 - Send event to post-code-manager based on which host's postcode received from
   IPMB interface (xyz.openbmc_project.State.Boot.RawX(0,1,2,3..N).Value)
 - phosphor-host-postd reads the host selection from the dbus property
 - Display the latest postcode of the selected host read through D-Bus.
 - unregister the D-Bus interface when host removed in the slot.

 **D-Bus interface**

  The below D-Bus names needs to be created for the multi-host post-code.

  Service   name    -- xyz.openbmc_project.State.Boot.Raw(0,1,2..N)
  
  Obj path  name    -- /xyz/openbmc_project/State/Boot/Raw
  
  Interface name    -- xyz.openbmc_project.State.Boot.Raw
  
  method                -- readPostcode(new method added in
                        phoshpor-dbus-interfaces)

## phosphor-post-code-manager

The phosphor-post-code-manager is the single process based on host discovery for
multi-host. This design shall not affect single host for post-code.

 - Create D-bus object for hosts.
 - Store/retrieve post-code from directory (/var/lib/phosphor-post-code-manager/
   hostX(0,1,2,3..N)) based on event received from phosphor-host-postd
 - unregister the D-bus interface when host removed in the slot.

 **D-Bus interface**

   The below D-Bus names needs to be created for multi-host post-code history.

   Service   name    -- xyz.openbmc_project.State.Boot.PostCodeX(0,1,2..N)
   
   Obj path  name    -- /xyz/openbmc_project/State/Boot/PostCode
   
   Interface name    -- xyz.openbmc_project.State.Boot.PostCode

## phosphor-dbus-interfaces

  The new method needs to create to interact between
  phosphor-host-postd and fb-ipmi-oem.

 [Raw.interface.yaml]( https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/
  xyz/openbmc_project/State/Boot/Raw.interface.yaml)

  methods:
  
    - name: readPostcode
      description: >
          Method to get the post codes .
      parameters:
        - name: postcode
          type: uint16
          description: >
              postcode indicates which host postcode.
        - name: host
          type: uint16
          description: >
              host indicates host number

## bmcweb
   The postcode history needs to be handled for the multi-host through redfish
   logging service.

## Alternate design

**phosphor-post-code-manager**

   The single process to handle the multi-host postcode history.

  **Platform specific service(fb-yv2-misc) alternate to phosphor-host-postd**

   Create and run the platfrom specific process daemon to
   handle IPMI postcode, seven segment display and
   host position specific feature.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3MzI4ODg0NzNdfQ==
-->
