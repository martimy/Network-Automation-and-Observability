# Network Automation with Nokia SR Linux and Cumulus Linux

## Introduction

In the evolving landscape of network operations, automation has become essential for scaling, efficiency, and reliability in data centers, cloud environments, and enterprise networks. Modern network operating systems (NOS) provide programmatic interfaces that enable engineers to configure, monitor, and manage devices through scripts, orchestration tools, and APIs, reducing manual errors and accelerating deployments. 

This tutorial introduces two prominent examples of such interfaces: the gNMI (gRPC Network Management Interface) with YANG data models in Nokia's SR Linux, and the NVUE (NVIDIA User Experience) REST API in NVIDIA's Cumulus Linux[^others]. This tutorial illustrates practical approaches to automation using command-line tools like gnmic and curl, as well as Ansible for structured workflows. 

Both interfaces make automation practical with CLI tools, Ansible collections, and secure transport. gNMI/YANG emphasizes standards, interoperability, and scalability, making it ideal for heterogeneous or large networks. NVUE REST API, by contrast, provides a simpler, Linux-centric workflow that aligns well with programmable environments.

[^others]: Nokia SR Linux Network OS employs three management interfaces: gNMI, JSON-RPC, and CLI. Cumulus Linux can be managed through NVUE and CLI (Linux commands).

Here is a brief comparison of the two interfaces:


| Criteria | Nokia SR Linux (gNMI/YANG) | Cumulus Linux (NVUE REST API) |
|---|-----|-----|
| **Protocol**        | gRPC on TCP/57400 | REST over HTTP/HTTPS/8765 |
| **Data Models**     | IETF-standard YANG, OpenConfig for interoperability | Custom model via OpenAPI v3.0.2, tailored to Cumulus ecosystem |
| **Configuration**   | Fully model-driven, JSON/YAML payloads   | Declarative API that generates configs for subsystems |
| **Change Handling** | Atomic by default | Revision-based staging, apply/save process |
| **Monitoring**      | Native streaming telemetry | State retrieval via polling, telemetry available via gNMI add-ons |
| **Tools**         | CLI (gnmic), Ansible, OpenConfig collectors   | CLI (curl), Ansible, REST client libraries |
| **Ecosystem**   | Multi-vendor standards, widely adopted in industry | Linux/cloud-centric, less compatible with YANG-based tools |
| **Suitable For** | Large-scale orchestration, real-time telemetry | Simplicity for Linux admins, straightforward automation |


## Configuring Nokia SR Linux using gNMI/YANG

Nokia SR Linux is a modern network operating system that supports model-driven management interfaces, including gNMI (gRPC Network Management Interface) for configuration, state retrieval, and streaming telemetry. gNMI uses YANG data models to structure data, allowing you to interact with the device using either Nokia's native YANG models or OpenConfig models (for vendor-neutral automation). This enables programmatic control over network elements like interfaces, routing, and system settings.

gNMI operates over gRPC, typically on port 57400 with TLS encryption required. The protocol supports RPCs like Capabilities (discover supported models), Get (retrieve config/state), Set (update/delete/replace config), and Subscribe (stream data).

gNMI integrats with tools like gnmic (a gNMI CLI client) for quick scripting and Ansible for orchestrated playbooks. Examples here are based on SR Linux 25.3 documentation and common practices.

Prerequisites:
- SR Linux version 21.11 or later (examples assume 25.3+).
- Access to the device via management interface (e.g., mgmt VRF).
- For external access, ensure firewall rules allow TCP/57400.
- Install gnmic: Download from https://gnmic.openconfig.net/ or via `go install github.com/openconfig/gnmic/cmd/gnmic@latest`.
- YANG models: SR Linux supports OpenConfig (e.g., openconfig-interfaces.yang) and native models. Download from Nokia's GitHub or device via gNMI Capabilities.

The Nokia SRLinux Docker container used in this lab meets all Prerequisites.

### Using the gNMI from the Command Line with gnmic

gnmic is a versatile CLI tool for sending gNMI RPCs. It supports targets with TLS and auth. Use `-a <ip>:<port>` for address, `-u <user> -p <pass>` for authentication, `--skip-verify` for testing, and `--encoding json` for output format.

#### Basic Capabilities: Discover Supported Models

Use to query the YANG models supported by the device.

```bash
gnmic -a 192.168.121.101:57400 -u admin -p NokiaSrl1! --skip-verify capabilities
```

The response includes model names like `ietf`, `nokia.com`, and versions.

#### Basic Get: Fetch Configuration or State

Retrieve specific paths. Use XPath-like notation for YANG paths.

- Get interface state (native YANG):

  ```bash
  gnmic -a 192.168.121.101:57400 -u admin -p NokiaSrl1! --skip-verify \
    --encoding json_ietf get --path /interface[name=ethernet-1/1]
  ```

Add `--type config` for config only, or `--type state` for operational data.


With lots of configuration options that `gnmic` supports it might get tedious to pass them all via CLI flags. Instead, `gnmic` can retrieve the configuration options from a file that includes all the command line flags. Options can also be passed to the `gnmic` by means of environment variables.

gnmic reads the configurtion file `.gnmic.yaml` automatically if it located in the working directory.

```
username: admin
password: NokiaSrl1!
skip-verify: true
encoding: json_ietf
```

Alternatively, the options can be passed as environment variables:

```
export GNMIC_USERNAME=admin
export GNMIC_PASSWORD=NokiaSrl1!
export GNMIC_SKIP_VERIFY=true
export GNMIC_ENCODING=json_ietf
```

#### Set: Configure the Device

Use `--update/--update-path`, `--replace/--replace-path`, or `--delete` flags. Provide paths and values inline or in YAML/JSON files as follows.

- Enable an interface using `--update`:

  ```bash
  gnmic -a 192.168.121.101:57400 set \
    --update interface[name=lo99]/admin-state:::json:::enable \
    --update interface[name=lo99]/description:::json:::"Test Loopback"
  ```

- Enable an interface using `--update-path`:

  ```bash
  gnmic -a 192.168.121.101:57400 set \
    --update-path interface[name=lo99]/admin-state \
    --update-value enable \
    --update-path interface[name=lo99]/description \
    --update-value "Test Loopback" 
  ```


- Set IP on loopback from file `config.yaml`:
  
  ```yaml
  - name: lo99
    subinterface:
      - index: 0
        ipv4:
          address:
            - ip-prefix: 10.10.10.1/32
  ```
  
  ```bash
  gnmic -a 192.168.121.101:57400 set \
    --update-path interface[name=lo99] \
    --update-file config.yaml
  ```
- Verify configuration:

  ```bash
  gnmic -a spine1 get --path interface[name=lo99] --type CONFIG --format flat
  srl_nokia-interfaces:interface[name=lo99]/admin-state: enable
  srl_nokia-interfaces:interface[name=lo99]/description: Test Loopback
  srl_nokia-interfaces:interface[name=lo99]/subinterface.0/index: 0
  srl_nokia-interfaces:interface[name=lo99]/subinterface.0/ipv4/address.0/ip-prefix: 10.10.10.1/32
  ```

- Delete config:
  
  ```bash
  gnmic -a 192.168.121.101:57400 set --delete /interface[name=lo99]
  ```

For bulk ops, use YAML files with multiple updates. SR Linux applies changes atomically; use dry-run with `--dry-run` to preview.

#### Other Examples

- Subscribe to telemetry: 
  ```bash
  gnmic -a 192.168.121.101:57400 subscribe \
    --path /interface[name=ethernet-1/1]/statistics --sample-interval 10s
  ```

- Multi-target: Use a targets file or `-a addr1,addr2`.

Refer to gnmic docs for advanced features like templates or Prometheus export.

### Using Ansible

Nokia provides the `nokia.grpc` Ansible collection for gNMI operations. However, this collection is not widely used in Ansible due to some limitations. Instead, Nokia offers `nokia.srlinux` collection that uses HTTP API and integrates well with other collections. The collection is available on Ansible Galaxy and can be installed using the ansible-galaxy command.

```
ansible-galaxy collection install nokia.srlinux
```

**Resources:**

- [SRLinux Collection](https://learn.srlinux.dev/ansible/collection/)
- [SRLinux Tutorials](https://learn.srlinux.dev/tutorials/programmability/ansible/using-nokia-srlinux-collection/)

*Note: the following examples assume the use of the Ansible files `ansible.cfg` and `inventory.yaml` included in this repository.*

The direct use of `gnmic` using shell commands is also possible:

```yaml
# Option 3: Direct gRPC using raw connection
- name: Direct gNMI connection
  hosts: nokia_switches
  vars:
    gnmi_host: "{{ ansible_host }}"
    gnmi_port: 57400
    gnmi_username: admin
    gnmi_password: NokiaSrl1!
  tasks:
    - name: Test gNMI connectivity with gnmic
      shell: |
        gnmic -a {{ gnmi_host }}:{{ gnmi_port }} \
              -u {{ gnmi_username }} \
              -p {{ gnmi_password }} \
              --skip-verify \
              capabilities
      register: gnmi_caps
      delegate_to: localhost

    - name: Display capabilities
      debug:
        var: gnmi_caps.stdout_lines
```

#### Using `get` Module

Fetch supported models.

```yaml
- name: Get state and config data from SR Linux
  hosts: all
  gather_facts: false
  tasks:
    - name: Get hostname and version
      nokia.srlinux.get:
        paths:
          - path: /system/name/host-name
          - path: /system/information/version
      register: get_result

    - name: Show Results
      ansible.builtin.debug:
        msg: "Host {{get_result.result[0]}} runs {{get_result.result[1]}} version."
```

#### Playbook for Configuration Updates

Use `update` and `replace` to change configuration.

```yaml
---
- name: Configure SR Linux Interface using nokia.srlinux collection
  hosts: nokia_switches

  tasks:
    - name: Configure loopback interface lo99
      nokia.srlinux.config:
        update:
          - path: /interface[name=lo99]/admin-state
            value: enable
          - path: /interface[name=lo99]/description
            value: Test Loopback
        replace:
          - path: /interface[name=lo99]/subinterface[index=0]
            value:
              index: 0
              admin-state: enable
          - path: /interface[name=lo99]/subinterface[index=0]
            value:
              ipv4:
                address:
                  ip-prefix: 10.10.10.1/32

    - name: Verify Configuration
      nokia.srlinux.get:
        paths:
          - path: /interface[name=lo99]
      register: get_result

    - name: Show Configuration
      debug:
        var: get_result.result[0]
```

Ansible also supports pushing configuration saved on on disk using Jinja2 templates.


## Introduction to Cumulus Linux NVUE REST API

Cumulus Linux, now under NVIDIA, provides the NVIDIA User Experience (NVUE) as a unified management interface for configuring and monitoring network switches. NVUE includes a CLI and a REST API that share the same underlying object model, making them functionally equivalent. The REST API allows programmatic access for automation, using standard HTTP methods to interact with the switch's configuration and operational state. It's particularly useful for network automation, as it supports integration with tools like curl for quick scripting or Ansible for more structured playbooks.

The API follows a tree-like structure based on the NVUE object model, with top-level objects such as `acl`, `bridge`, `evpn`, `interface`, `mlag`, `nve`, `platform`, `qos`, `router`, `service`, `system`, and `vrf`. Operations map to NVUE CLI commands:
- GET ~= `nv show`
- PATCH/POST/DELETE ~= `nv set`, `nv unset`, `nv config apply`

Prerequisites:
- Cumulus Linux version 5.0 or later (examples here based on 5.5 documentation).
- The `nvued` service must be running (`sudo systemctl status nvued`).
- For external access, configure NGINX as a reverse proxy for HTTPS.

In this lab, the Cumulus Docker container meets all prerequisites so no additional setup is needed.

#### Basic GET: Fetch Running Configuration

Use the following example to obtain the current applied configuration on the switch. Change the `rev` argument to view any revision (e.g. '1'). Possible options for the `rev` argument include `startup`, `pending`, `operational`, and `applied`.

```bash
curl -u 'cumulus:cumulus' -k -X GET https://192.168.121.111:8765/nvue_v1/?rev=applied
```

#### Managing Revisions (Staging Changes)

NVUE uses revisions to stage configs without immediate application:

1. Create a revision (POST):
   
   ```bash
   curl -u 'cumulus:cumulus' -k -X POST https://192.168.121.111:8765/nvue_v1/revision
   ```
  
   Response: `{"1": {...}}` (or similar ID).


2. Apply changes to the revision (PATCH with `?rev=<id>`):
   Example: Set loopback interface.
   
   ```bash
   curl -u 'cumulus:cumulus' -k -X PATCH -H 'Content-Type:application/json' \
    -d '{"lo": {"ip": {"address": {"10.10.10.1/32":{}}}}}' \
    https://192.168.121.111:8765/nvue_v1/interface?rev=1
   ```

3. Review pending changes (GET):
   
   ```bash
   curl -u 'cumulus:cumulus' -k -X GET https://192.168.121.111:8765/nvue_v1/interface?rev=1
   ```

4. Apply the revision (PATCH):
   
   ```bash
   curl -u 'cumulus:cumulus' -H 'Content-Type:application/json' \
     -d '{"state": "apply", "auto-prompt": {"ays": "ays_yes"}}' \
     -k -X PATCH https://192.168.121.111:8765/nvue_v1/revision/1
   ```


5. Review the status of the apply and the configuration:

   ```bash
   curl -u 'cumulus:cumulus' -k -X GET https://192.168.121.111:8765/nvue_v1/revision/1
   ```

6. Save to startup (PATCH on applied revision):
   
   ```bash
   curl -u 'cumulus:cumulus' -H 'Content-Type:application/json' \
     -d '{"state": "save", "auto-prompt": {"ays": "ays_yes"}}' \
     -k -X PATCH https://192.168.121.111:8765/nvue_v1/revision/applied
   ```

#### Other Examples

- Unset a config (PATCH with unset):

  ```bash
  curl -u 'cumulus:cumulus' -H 'Content-Type:application/json' \
    -d '{"unset": {"interface": {"swp1": {"link": {"speed": {}}}}}}' \
    -k -X PATCH https://192.168.121.111:8765/nvue_v1?rev=1
  ```

- Delete an object (DELETE):

  ```bash
  curl -u 'cumulus:cumulus' -k -X DELETE \
    https://192.168.121.111:8765/nvue_v1/interface/swp1?rev=1
  ```

- Bulk patch at root:
  Use `/nvue_v1` endpoint for multiple changes in one request.

For information about using the NVUE REST API, refer to the [NVUE API Swagger documentation](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-53/api/index.html#/interface/deleteInterfaceIp).


### Using the API with Ansible

NVIDIA provides an official Ansible collection `nvidia.nvue` for interacting with the NVUE REST API, abstracting curl-like operations into modules. Install it via Ansible Galaxy:

```bash
ansible-galaxy collection install nvidia.nvue
```

Available modules include:

- `nvidia.nvue.bridge`: Manages bridge configurations.
- `nvidia.nvue.config`: Handles revisions (create, apply, save).
- `nvidia.nvue.command`: Runs arbitrary NVUE commands (like CLI equivalents via API).
- Others: `nvidia.nvue.interface`, `nvidia.nvue.router`, etc., for specific features.

Connection uses `network_os: nvidia.nvue` in inventory or playbooks, with vars for host, username, password, and `validate_certs: false` for testing.

*Note: the following examples assume the use of the Ansible files `ansible.cfg` and `inventory.yaml` included in this repository.*

#### Example: Using `nvidia.nvue.command` Module

This module runs NVUE commands remotely.

```yaml
---
- name: Set system pre-login message
  hosts: switches
  tasks:
    - name: Set message
      nvidia.nvue.command:
        commands:
          - set system message pre-login "Welcome to the switch!"
```

#### Example Playbook for Configuration Management

From community examples, here's a basic playbook to fetch config, stage changes, and apply (adapted from blogs using raw HTTP modules, but align with collection):

```yaml
---
- name: Test playbook to update interface settings
  hosts: cumulus_switches

  tasks:
    - name: Create new revision
      nvidia.nvue.config:
        state: new
      register: revision

    - name: Show revision
      ansible.builtin.debug:
        msg: '{{ revision.revid }}'

    - name: Set interface
      nvidia.nvue.interface:
        state: merged
        revid: '{{ revision.revid }}'
        data:
          - id: lo
            ip:
              address:
                - id: '10.10.10.1/32'
            type: 'loopback'
      register: interface

    - name: Show previous output
      ansible.builtin.debug:
        msg: '{{ interface }}'

    - name: Apply new revision
      nvidia.nvue.config:
        state: apply
        revid: '{{ revision.revid }}'
        force: true
        wait: 10
      register: revision

    - name: Show previous output
      ansible.builtin.debug:
        msg: '{{ revision }}'

    - name: Fetch interface config
      nvidia.nvue.interface:
        state: gathered
      register: interface

    - name: Display interface config
      ansible.builtin.debug:
        msg: '{{ interface }}'
```

For raw REST without the collection, use Ansible's `uri` module:

```yaml
---
- name: Configure Cumulus Linux Leaf Switches using NVUE
  hosts: cumulus_switches
  gather_facts: no

  vars:
    nvue_common_params: &nvue_common
      headers: "{{ nvue_headers }}"
      user: "{{ nvue_username }}"
      password: "{{ nvue_password }}"
      force_basic_auth: true
      validate_certs: false

  tasks:
    - name: Wait for NVUE API to be ready
      uri:
        url: "{{ nvue_api_base_url }}/system"
        method: GET
        <<: *nvue_common
        status_code: [200, 404]
      register: nvue_ready
      until: nvue_ready.status == 200
      retries: 30
      delay: 2

    - name: Create NVUE revision
      uri:
        url: "{{ nvue_api_base_url }}/revision"
        method: POST
        body_format: json
        body: {}
        status_code: [200, 201]
        <<: *nvue_common
      register: revision_response

    - name: Extract revision ID from POST response
      set_fact:
        revision_id: "{{ revision_response.json.keys() | first }}"
      when: revision_response.json is defined

    - name: Debug revision ID
      debug:
        msg: "Created revision: {{ revision_id }} for {{ loopback_ip }}"

    - name: Configure interfaces
      uri:
        url: "{{ nvue_api_base_url }}/?rev={{ revision_id }}"
        method: PATCH
        body_format: json
        body:
          interface:
            lo:
              ip:
               address: "{{ { loopback_ip: {} } }}"
        status_code: [200, 204]
        <<: *nvue_common
      register: interface_config_result
      when: revision_id is defined

    - name: Apply configuration changes
      uri:
        url: "{{ nvue_api_base_url }}/revision/{{ revision_id }}"
        method: PATCH
        body_format: json
        body:
          state: apply
        status_code: [200, 202, 204]
        <<: *nvue_common
      register: apply_result
      when: revision_id is defined

    - name: Display apply result
      debug:
        msg: "Configuration apply status: {{ apply_result.status }}"
      when: apply_result is defined
```


These integrate well with existing Ansible setups for idempotent network automation. Check the full collection docs on Galaxy for module-specific params.

## Author

Created by Maen Artimy - [Personal Blog](http://adhocnode.com)

