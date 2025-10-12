# Suggested Hardware and Roles


Although this lab is virtual and does not require any phsyical routers, I will use the following hardware models to be able to populate NetBox databases and provide other information.

| Device | Image | Suggested Hardware Model | Role | Key Ports |
| :--- | :--- | :--- | :--- | :--- |
| Spine (x2) | Nokia SR Linux | Nokia 7220 IXR-D1 | Spine | 4x 10G SFP+ uplinks |
| Leaf (x4) | Cumulus Linux v5.3 | NVIDIA Spectrum MSN2201-CB2FC | Leaf | 48x 1G RJ45 access, 4x 100G QSFP28 uplinks |


### Nokia 7220 IXR-D1 as Spine (SR Linux)

The [Nokia 7220 IXR-D1](https://www.nokia.com/asset/207599/) is described as being optimized for leaf-spine designs. While it supports 1GE on the copper ports, its 4x SFP+ ports with native 10GE hardware support are ideal for the spine role to connect to the four leaf switches.

In a typical spine-leaf topology, each leaf switch connect to all spine switches. With two spines and four leaves, each leaf will need two uplinks (one to each spine), totaling 8 spine-to-leaf connections. The 7220 IXR-D1's 4x 10GE SFP+ ports are suitable for this lab topology where the two spine switches together provide $2 \times 4 = 8$ uplinks.

### NVIDIA Spectrum MSN2201-CB2FC as Leaf (Cumulus Linux)

NVIDIA (which acquired Cumulus Networks) sells switches optimized for Cumulus Linux. The [NVIDIA Spectrum MSN2201-CB2FC](https://marketplace.nvidia.com/en-us/enterprise/networking/sn2201/) is a suitable choice for the leaf role, as it is a Top-of-Rack (ToR) switch designed for server connectivity. The switch features 48x RJ45 1GbT ports, which are suitable for connecting servers at 1G speed. The switch also has 4x QSFP28 100GbE ports. In a typical DC fabric, these 100G ports are used as uplinks to the spine. However, many modern open networking switches (which run Cumulus Linux) allow the use of breakout cables or specific transceiver modules in their high-speed ports to provide 10G connectivity, allowing connectivity to the spine switches.

    
**Note:** In a lab setting, using a virtual switch image for Cumulus Linux (like Cumulus VX in Containerlab) will abstract the specific port speeds and allow you to model 1G/10G connectivity from the leaf to a host, irrespective of the physical hardware's primary port speeds. However, for a realistic hardware model, this NVIDIA switch provides the necessary combination of density and speed for a leaf.

