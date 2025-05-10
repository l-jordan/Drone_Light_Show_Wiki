PX4 is an open source flight control software for drones. It also has compatibility with ROS2, the robotics software used to interface with the drone. To use PX4 and ROS2 together, it is important to understand what topics are available and how they work. 

It is first recommended that you read the README.md file so you understand how the PX4-ROS2 MICRO-XCRE DDS bridge works. 

The two sources of documentation are the topics that can be used in the ROS2 node and the uORB messages and how they work: 
1. [ROS2 Topics](https://github.com/PX4/PX4-Autopilot/blob/main/src/modules/uxrce_dds_client/dds_topics.yaml) 
2. [uORB Messages](https://docs.px4.io/main/en/msg_docs/) (The topics are usually mentioned at the bottom of the file but it can be easier to use source  [[1]](https://github.com/PX4/PX4-Autopilot/blob/main/src/modules/uxrce_dds_client/dds_topics.yaml) as a quick 'dictionary')

The uORB messages are written in a very similar, if not the same, format as an older messaging protocol for communicating with drones called [MAVLink](https://mavlink.io/en/). The uORB messages tell us what parameters can be changed to execute the desired function when we publish to a topic. In this documentation, we will show you how to use the topics in general but also specific examples of some of the topics and uORB messages we used to establish communication between drones and to achieve the desired drone function in our offboard_control.py example.

## General Instructions 

1. Determine if you want to subscribe or publish to a topic
2. Determine what parameters you need to subscribe/publish to by referring to the documentation [[2]](https://docs.px4.io/main/en/msg_docs/)
3. In your Python file, import the uORB messages associated with the topics you want to use from px4_msgs.msg as seen below:
``` python 
from px4_msgs.msg import <Insert uORB message>
```
4. Create your subscriber or publish under def __init__. 
 A subscriber and publisher need the following information: the uORB name, the topic name, the QoS profile and for subscribers only, the corresponding function that can print/define the subscriber information. 

``` python 
def __init__(self) -> None:
        super().__init__('offboard_control_node')

        # Configure QoS profile for publishing and subscribing
        qos_profile = QoSProfile(
            reliability=ReliabilityPolicy.BEST_EFFORT,
            durability=DurabilityPolicy.TRANSIENT_LOCAL,
            history=HistoryPolicy.KEEP_LAST,
            depth=1
        )
                
        # Create subscribers
        self.subscriber = self.create_subscription(
            <uORB message name>, '/fmu/out/<topic_name>', self.<subscriber_function>, qos_profile)

        # Create publishers
        self.publisher = self.create_publisher(
            <uORB message name>, '/fmu/out/<topic_name>', qos_profile)

```
5. (SUBSCRIBER ONLY) Create the definition used to access the topic information. To know what parameters you want to access, visit the second source to see what parameters are associated with the uORB message associated with the topic you are subscribing to [[2]](https://docs.px4.io/main/en/msg_docs/).

```python
def subscriber_function(self, msg):
    # Example
    x = msg.x 
    y = msg.y
    z = msg.z

    # Print the message outputs 
    self.get_logger().info(f"x = {x}, y = {y}, z = {z}")
```
6. (PUBLISHER ONLY) Create the definition used to publish information to the desired parameters. To know what parameters you need to access, visit the second source to see what parameters are associated with the uORB message with the topic you are publishing to [[2]](https://docs.px4.io/main/en/msg_docs/).

```python
def publisher_function(self, x: float, y: float, z: float):
    msg = <Insert uORB message>
    # Example
    msg.<param_name1> = x 
    msg.<param_name1> = y
    msg.<param_name1> = z
   
    # The name of the publisher you defined in __init__ for your uORB message
    self.publisher.publish(msg)

```

## Subscriber Example
For this example, I will be showing you how to access the GPS information of the drone. 

To access this information you have to use the uORB message: [**SensorGps**](https://docs.px4.io/main/en/msg_docs/SensorGps.html) \
This is associated with the topic: `/fmu/out/vehicle_gps_position` \

I urge you to look at the documentation related to the SensorGps uORB message to see how these messages are usually formatted. We will use the following extract of the file to access the GPS coordinates of the drone:

``` C
float64 latitude_deg		# Latitude in degrees, allows centimeter level RTK precision
float64 longitude_deg		# Longitude in degrees, allows centimeter level RTK precision
float64 altitude_msl_m		# Altitude above MSL, meters
```

With this information, the subscriber can be built. 

1. Following the aforementioned steps, import the uORB message from px4_msgs.msg as `from px4_msgs.msg import SensorGps`.

2. In the __init__ definition, define the subscriber. In this example, we will use the a function called `gps_position_callback` to access the contents of the SensorGps topic.
``` python
def __init__(self) -> None:
        super().__init__('offboard_control_node')

        # Configure QoS profile for publishing and subscribing
        qos_profile = QoSProfile(
            reliability=ReliabilityPolicy.BEST_EFFORT,
            durability=DurabilityPolicy.TRANSIENT_LOCAL,
            history=HistoryPolicy.KEEP_LAST,
            depth=1
        )
                
        # Create subscribers
        self.subscriber = self.create_subscription(
            SensorGps, '/fmu/out/vehicle_gps_position', self.gps_position_callback, qos_profile)
```
3. Create the gps_position_callback. Within this definition, we want to access the **latitude_deg**, **longitude_deg** and **altitude_msl_m** data. By following the format shown in step 5 in the previous section, this can be done in the following way: 

``` python
    def gps_position_callback(self, msg):
        # Print GPS position data
        lat = msg.latitude_deg
        lon = msg.longitude_deg
        alt = msg.altitude_msl_m
        self.get_logger().info(f'Lat: {lat:.6f}, Lon: {lon:.6f}, Alt (MSL): {alt:.2f} m')
```
4. Launch the node and monitor the terminal. You should see an output similar to this (depending on your trajectory):

``` python
[offboard_control_node-1] [INFO] [1746907973.417738788] [control]: Lat: 47.397959, Lon: 8.546141, Alt (MSL): 0.22 m
[offboard_control_node-1] [INFO] [1746907973.555814720] [control]: Lat: 47.397959, Lon: 8.546141, Alt (MSL): 0.20 m
[offboard_control_node-1] [INFO] [1746907973.671510204] [control]: Lat: 47.397959, Lon: 8.546142, Alt (MSL): 0.22 m
[offboard_control_node-1] [INFO] [1746907973.733416409] [control]: Lat: 47.397959, Lon: 8.546142, Alt (MSL): 0.21 m
```

## Publisher Example
For this example, we will show you a standard template we have used for most vehicle commands but more specifically for an arming command. This means that we are publishing the parameter that will change the drone from disarmed to armed. 

To access this information you have to use the uORB message: [**VehicleCommand**](https://docs.px4.io/main/en/msg_docs/VehicleCommand.html) \
This is associated with the topic: `/fmu/in/vehicle_command` \

I urge you to look at the documentation related to the VehicleCommand uORB message to see how these messages are usually formatted. We will use the following extract of the file to access the arming parameters of the drone:

``` C
# used as param1 in ARM_DISARM command
int8 ARMING_ACTION_DISARM = 0
int8 ARMING_ACTION_ARM = 1
```

1. Following the aforementioned steps, import the uORB message from px4_msgs.msg as `from px4_msgs.msg import VehicleCommand`.

2. In the __init__ definition, define the publisher. 

``` python
self.vehicle_command_publisher = self.create_publisher(VehicleCommand, '/fmu/in/vehicle_command', qos_profile)
```

3. This is a unique step that we used. We created a multifunctional publishing vehicle command definition as multiple functions require the VehicleCommand uORb message such as arming, disarming, engaging offboard mode, landing etc. By using the `**params`, it unpacks all parameters that you specify that are related to your specific action. So if you only specify param1, then it will update param 1 and surpass the other param statements within the definition.

``` python
    def publish_vehicle_command(self, command, **params) -> None:
        msg = VehicleCommand()
        msg.command = command
        msg.param1 = params.get("param1", 0.0)
        msg.param2 = params.get("param2", 0.0)
        msg.param3 = params.get("param3", 0.0)
        msg.param4 = params.get("param4", 0.0)
        msg.param5 = params.get("param5", 0.0)
        msg.param6 = params.get("param6", 0.0)
        msg.param7 = params.get("param7", 0.0)
        msg.target_system = 1
        msg.target_component = 1
        msg.source_system = 1
        msg.source_component = 1
        msg.from_external = True
        msg.timestamp = int(self.get_clock().now().nanoseconds / 1000)
        self.vehicle_command_publisher.publish(msg)
``` 

As we are looking specifically at arming the drone, we must understand what each parameter is and what message we are updating for the arming command. As this uses old MAVLink protocol, the following steps need to be taken:

a) Find the associated vehicle command message in your uORB message. In this case, it is `VEHICLE_CMD_COMPONENT_ARM_DISARM`
b) In the uORB documentation presented by PX4, you get this description of the parameters:

``` python
float32 param1			# Parameter 1, as defined by MAVLink uint16 VEHICLE_CMD enum.
float32 param2			# Parameter 2, as defined by MAVLink uint16 VEHICLE_CMD enum.
float32 param3			# Parameter 3, as defined by MAVLink uint16 VEHICLE_CMD enum.
float32 param4			# Parameter 4, as defined by MAVLink uint16 VEHICLE_CMD enum.
float64 param5			# Parameter 5, as defined by MAVLink uint16 VEHICLE_CMD enum.
float64 param6			# Parameter 6, as defined by MAVLink uint16 VEHICLE_CMD enum.
``` 
This is essentially useless unless you know that you need to refer to **MAVLink documentation**. The old MAVLink protocol defines this function in a slightly different manner but if you want to arm the drone, you will need to set the param1 to 1. The specific parameters that are associated with the vehicle command message related to arming and disarming the drone can be found [here](https://mavlink.io/en/messages/common.html#MAV_CMD_COMPONENT_ARM_DISARM). 

4. To use the publish_vehicle_command definition, create a new definition specifically for arming the drone and call the publish_vehicle_command within in by using the message `VEHICLE_CMD_COMPONENT_ARM_DISARM` and setting param1 to 1 (arming the drone).


``` python
# Arming the vehicle by sending the command
def arm(self):
   self.publish_vehicle_command(VehicleCommand.VEHICLE_CMD_COMPONENT_ARM_DISARM, param1=1.0)

   # Printing an update
   self.get_logger().info('Arm command sent')

``` 

5. To use this function, just call the arming function in your main control loop like so *:

``` python

    def control_loop(self) -> None:
        self.arm()
        self.get_logger().info("Arm command sent")

``` 

* Please keep in mind, if you want to use the arming command, you will have to enter offboard mode. For this you need to publish the offboard command mode message to OffboardControlMode() uORB message and have to send setpoint command for at least before the arming command will initiate in Offboard Mode [[3]](https://docs.px4.io/main/en/flight_modes/offboard.html#description)