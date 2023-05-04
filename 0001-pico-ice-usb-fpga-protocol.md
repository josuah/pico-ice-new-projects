# USB-FPGA protocol


## Initial Situation

In the context of ROS2, there is a central protocol that makes everything
communicate together. Anything using a different protocol is "bridged" to it.

Pushing this limit further Micro-ROS is bringing the protocol onto the
firmware itself, allowing ROS2-formatted messages sent directly over USB.

But this requires knowing the internals of ROS2 to be able to do anything
which limits the adopters to FPGA developers who are also well used to ROS2.

```
... <--ros2--> [agent] <--ros2--> [RP2040] <--???--> [iCE40]
                          ^^^^     ^^^^^^
                        difficult to debug
```

## Solution Proposed

To avoid any dependency on ROS2 for developping/debugging/building new
pico-ice FPGA HDL code, it is possible to use a trivial protocol such as
[spibone](https://github.com/xobs/spibone) that interfaces the FPGA bus
data registers directly over USB.

```
Read protocol:
    Write: 01 | AA | AA | AA | AA
    [Wishbone Operation]
    Read:  01 | VV | VV | VV | VV
Write protocol:
    Write: 00 | AA | AA | AA | AA | VV | VV | VV | VV
    [Wishbone Operation]
    Read:  00
```

A protocol looking like `spibone` above would be looking familiar to FPGA
developers, who would be the one working with it, providing a reference
of address for use by the ROS2 developers (possibly the same person).

```
... <--ros2--> [script.py] <--spibone--> [RP2040] <--spibone--> [iCE40]
                ^^^^^^^^^     ^^^^^^^     ^^^^^^     ^^^^^^^
               easier to debug at every level link the chain
```


## ROS2 Integration

To allow ROS2 integration, a simple wrapper script in python would handle
the FPGA communication over USB on one side (`f = open('/dev/ttyACM0')`)
and the ROS2 communication on the other side
([`rclpy`](https://github.com/ros2/rclpy)).

```python
import pico_ice_ros2 as ice

ice.run([
    ice.Publish(0x1001, "/pico_ice_button", ice.trigger_if_not_0xff),
    ice.Publish(0x1002, "/pico_ice_ir_remote", ice.trigger_on_interrupt),
    ice.Publish(0x1003, "/pico_ice_bosch_bme280", ice.trigger_every_10ms),
    ice.Subscribe(0x2001, "/imu.orientation.x"),
    ice.Subscribe(0x2002, "/imu.orientation.y"),
    ice.Subscribe(0x2003, "/imu.orientation.z"),
    ice.Subscribe(0x2004, "/imu.orientation.w"),
])
```

This kind of Rosetta Stone would allow ROS2 litteracy to FPGA developers,
and FPGA litteracy to ROS2 developers around a same project.
It is also easier to implement and maintain.
