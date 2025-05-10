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

I urge you to look at the documentation related to the SensorGps to see how these messages are usually formatted. We will use the following extract of the file to access the GPS coordinates of the drone:

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