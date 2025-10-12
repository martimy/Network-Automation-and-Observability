# Suggested Hardware and Roles


Although this lab is virtual and does not require any phsyical routers, I will use the following hardware models to be able to populate NetBox databases and provide other information.

| Device | Image | Suggested Hardware Model | Role | Key Ports |
| :--- | :--- | :--- | :--- | :--- |
| Spine (x2) | Nokia SR Linux | 7220 IXR-D3L | Spine | 32x 100G QSFP28, 2x 10G SFP+ |
| Leaf (x4) | Cumulus Linux v5.3 | NVIDIA Spectrum MSN2201-CB2FC | Leaf | 48x 1G RJ45 access, 4x 100G QSFP28 uplinks |


### Nokia 7220 IXR-D1 as Spine (SR Linux)

The[Nokia 7220 IXR-D3L](https://www.nokia.com/asset/207599/) is optimized for leaf-spine designs. Its 32 QSFP28 ports with native 100GE hardware support are ideal for the spine role to connect to the four leaf switches.

In a typical spine-leaf topology, each leaf switch connect to all spine switches. With two spines and four leaves, each leaf will need two uplinks (one to each spine), totaling 8 spine-to-leaf connections. The 7220 IXR-D3L's 4x QSFP28 ports are suitable for this lab topology where the two spine switches together provide $2 \times 4 = 8$ uplinks.

### NVIDIA Spectrum MSN2201-CB2FC as Leaf (Cumulus Linux)

NVIDIA (which acquired Cumulus Networks) sells switches optimized for Cumulus Linux. The [NVIDIA Spectrum MSN2201-CB2FC](https://marketplace.nvidia.com/en-us/enterprise/networking/sn2201/) is a suitable choice for the leaf role, as it is a Top-of-Rack (ToR) switch designed for server connectivity. The switch features 48x RJ45 1GbT ports, which are suitable for connecting servers at 1G speed. The switch also has 4x QSFP28 100GbE ports. In a typical DC fabric, these 100G ports are used as uplinks to the spine.
    

