[![npm](https://img.shields.io/npm/v/solar-manager-virtual-device.svg?style=flat-square)](https://www.npmjs.com/package/solar-manager-virtual-device)
[![build](https://img.shields.io/github/workflow/status/claudiospizzi/solar-manager-virtual-device/CI?style=flat-square)](https://github.com/claudiospizzi/solar-manager-virtual-device/actions/workflows/ci.yml)

# solar-manager-virtual-device

Solar Manager Virtual Device to integrate unsupported devices into a smart home power management.

## Description

This node module is able to mock a MyStrom Switch device to be used with the Solar Manager. This enables the usage of unsupported devices in a smart home power management. The whole solutions consists of three parts:

- **Virtual Device**  
  Run the virtual device, for example within a Docker container.

- **Solar Manager**  
  Configure a MyStrom Switch within the [Solar Manager configuration](https://web.solar-manager.ch/my-devices/).

- **Integration**  
  Integrate the virtual device with the 3rd party device, for example by using Node-RED and the [node-red-contrib-mystrom-switch](https://flows.nodered.org/node/node-red-contrib-mystrom-switch) node.

### Virtual Device Setup (this module)

This module can be used within a Docker container. The pre-built container image is published to the GitHub Container Registry as `ghcr.io/neuhausf/solar-manager-virtual-device`.

Because the Solar Manager communicates with the virtual device over port 80, each virtual device must have its own dedicated IP address. The recommended approach is to attach the containers to a `macvlan` network so that every container appears as an independent device on the local network.

Adjust the `parent` interface, the `subnet`, the `gateway`, and the `ip_range` (the range reserved for containers, must not overlap with addresses already used by other hosts) to match your network configuration, then assign a unique `ipv4_address` to each virtual device.

```yaml
services:
  virtual-device-1:
    image: ghcr.io/neuhausf/solar-manager-virtual-device:1
    container_name: virtual-device-1
    restart: unless-stopped
    networks:
      macvlan:
        ipv4_address: 192.168.1.201

  virtual-device-2:
    image: ghcr.io/neuhausf/solar-manager-virtual-device:1
    container_name: virtual-device-2
    restart: unless-stopped
    networks:
      macvlan:
        ipv4_address: 192.168.1.202

networks:
  macvlan:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1
          ip_range: 192.168.1.200/29
```

### Solar Manager Configuration

Within the Solar Manager, a new device can be added with the type `Smart Plug` and the name `MyStrom Energy Control Switch`. Specify the IP address of the virtual device as the `IP Address` field. The temperature field reported by the virtual device can be updated via the `/temperature` endpoint and will be usable in the Solar Manager.

### Node-RED Integration

Finally, the integration to the actual physical device needs to be implemented. The implementation must scan the report for the relay state to turn on or off. In addition, the current power usage should be reported every 5 seconds, as the Solar Manager will scan the device status every 5 seconds. This can be done by using the [node-red-contrib-mystrom-switch](https://flows.nodered.org/node/node-red-contrib-mystrom-switch) Node-RED module.

### Home Assistant Integration

The virtual device can also be integrated with [Home Assistant](https://www.home-assistant.io/). Home Assistant can post the current power consumption to the `/power` endpoint and the current temperature to the `/temperature` endpoint, while reading the relay state from the `/report` endpoint to control connected devices accordingly.

A ready-to-use bridge between the Solar Manager virtual device and MyStrom devices is available at [solarmanager-mystrom-bridge](https://github.com/neuhausf/solarmanager-mystrom-bridge). It demonstrates how to wire up the virtual device API with real hardware and can serve as a reference for building custom Home Assistant integrations.

## API

The following methods are available. All follow the exact API implementation as the official [MyStrom Switch](https://api.mystrom.ch/#982cf1bb-c873-4f62-b3c2-1cdfa51e1afe) API. Except the `/power` and `/temperature` resources, these are used to set the current power consumption and temperature of the device.

| Method | Resource         | Body                      | Response        |
| ------ | ---------------- | ------------------------- | --------------- |
| `GET`  | `/report`        |                           | Device state    |
| `GET`  | `/relay?state=1` |                           |                 |
| `GET`  | `/relay?state=0` |                           |                 |
| `GET`  | `/toggle`        |                           | The relay state |
| `POST` | `/power`         | `{ "power": 42 }`         |                 |
| `POST` | `/temperature`   | `{ "temperature": 21.5 }` |                 |

## Constraint

This module is not associated with the [Solar Manager AG](https://www.solarmanager.ch/). In case of problems with the integration between this module and Solar Manager, open a GitHub issue.

This software is provided "as is", without any guarantees on the function and operation of the Solar Manager device and account. You use it at your own risk. For more details, check the license terms.
