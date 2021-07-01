# Introduction
The scope of this document is to provide the requirements and a high-level design proposal for VXLAN src port support.

# Requirements
When we encapsulate the VxLAN packet, in the outer packet the L4 src port field (16 bits) is currently 0-255. 
The top 8 bits are hard coded 0, the bottom 8 bits are hash based value

At a high level the following should be supported:
- Support user defined range of UDP source ports for VXLAN tunnel.
- Attributes valid only if:
  SAI_TUNNEL_ATTR_TYPE == SAI_TUNNEL_TYPE_VXLAN and SAI_TUNNEL_ATTR_VXLAN_UDP_SPORT_MODE == SAI_TUNNEL_VXLAN_UDP_SPORT_MODE_USER_DEFINED
- Configure range with 2 parameters:
	- SAI_SWITCH_TUNNEL_ATTR_VXLAN_UDP_SPORT - defines the fixed base (default 0)
	- SAI_SWITCH_TUNNEL_ATTR_VXLAN_UDP_SPORT_MASK - mask defining the number of least significant bits reserved for the calculated hash value. 
	0 means a fixed value  

# Design Proposal
This document is intended to describe details about adding support of VxLAN source port configuration to SONiC. 
A user can set an attribute "vxlan_sport" and "vxlan_mask" to AppDB via swssconfig. The default values if not specified would be "0".

swssconfig - is application in swss docker which loads db settings from json file.
Example: ```swssconfig /etc/swss/config.d/switch.json``` 

```
switch.json:
[
    {
        "SWITCH_TABLE:switch": {
			"vxlan_sport": "0xFFA0",
			"vxlan_mask": "3"
        },
        "OP": "SET"
    }
]
```

```
"SWITCH_TABLE:switch": {
	"vxlan_sport": {{vxlan_sport}},
	"vxlan_mask": {{vxlan_mask}}
},
```

```
; Defines Switch table schema

key             = INTERFACE:name        ; Same as existing
; field
vxlan_sport     = vxlan sport    	; Default "0" 
vxlan_mask      = vxlan sport mask  	; Default "0"
```
Updating data in SWITCH_TABLE will trigger switchorch (orcagent). It will handle new value and execute: 
```set_switch_tunnel_attribute(gSwitchId, &attr)``` function, where:
- gSwitchId - pointer to the SAI switch OID
- attr.id = we need to set 2 attributes with following id: ```SAI_SWITCH_TUNNEL_ATTR_VXLAN_UDP_SPORT``` and ```SAI_SWITCH_TUNNEL_ATTR_VXLAN_UDP_SPORT_MASK```.
But they will be valid only if two other attributes are set: ```SAI_TUNNEL_ATTR_VXLAN_UDP_SPORT_MODE == SAI_TUNNEL_VXLAN_UDP_SPORT_MODE_USER_DEFINED```
and ```SAI_TUNNEL_ATTR_TYPE == SAI_TUNNEL_TYPE_VXLAN```
- attr.val = new value from swssconfig : UDP src port (for ```SAI_SWITCH_TUNNEL_ATTR_VXLAN_UDP_SPORT```) or mask that defines the number of bits (for ```SAI_SWITCH_TUNNEL_ATTR_VXLAN_UDP_SPORT_MASK```)


# Flows
![](https://github.com/ayurkiv-nvda/SONiC/blob/ayurkiv-design-ver-1/images/vxlan_hld/vnet_vxlan_src_port_range_flow.png)
