# Lab Setup

Before starting this project, ensure your host machine is ready to run the containerlab-based lab. This tutorial focuses on setting up the tools and deploying the spine-leaf topology.

## Install Required Software:

- Use a Linux host (Ubuntu 22.04+ or similar recommended for compatibility).
- Install Docker: Follow the official [Docker docs](https://docs.docker.com/engine/install/ubuntu/) for installation instructions.
- Install containerlab following the [containerlab documentation](https://containerlab.dev/install/).
- Install Git: `sudo apt install git`.
- (Optional): Install tools like [net-snmp](https://www.net-snmp.org/) (more tools will be listed in different tutorials).


## Clone the Repository:

- Run `git clone https://github.com/martimy/Network-Automation-and-Observability.git [directory]`.
- Navigate to the relevant directory.


## Understand the Lab Topology:

The setup uses containerlab to emulate a data center fabric. The topology is defined in the file `topology.clab.yml':

- Spines: 2 x Nokia SR Linux (version 25.x) routers.
- Leaves: 4 x Cumulus Linux (version 5.3) routers.
- Connections: Each leaf connects to both spines.
- Management: Devices will have management interfaces accessible via the host..

The topology file includes also startup configs.

## Deploy the Lab:

- Pull required container images if not automatic (containerlab will do this automatically for the routers).
- Deploy: Run `sudo containerlab deploy [-t topology.clab.yml]`.
- Verify: Use `sudo containerlab inspect` to check running nodes, IPs, and statuses. Access devices via `docker exec -it <container-name> <cli-command>` or `ssh <user>@<management-ip>.
- Spend time exploring: Use SR Linux CLI (sr_cli) on spines and Cumulus commands (net show) on leaves to familiarize yourself with the platforms.
- Teardown when done: `sudo containerlab destroy [-t topology.clab.yml]`.

Note: use `clab` as a shorthand for `containerlab` command.

## Author

Created by Maen Artimy - [Personal Blog](http://adhocnode.com)
