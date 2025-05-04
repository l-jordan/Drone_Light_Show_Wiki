In this section, we will bring you through how we set everything up to achieve autonomous operation of the drone. We will show you how we created our files for a simple test case where we took off, flew in a circle and landed for a singular drone.

## Creating a Package 
First we created a package which will house our autonomous operating files. 

To do this, we used the following commands: 

```powershell
cd ros2_ws
ros2 pkg create --build-type ament_python <package_name_pkg> --dependencies <dependencies>
```

Once the package is created, we executed it and sourced the workspace to make sure all the latest updates are there. Make sure you 'colcon build' in ros2_ws

```powershell
colcon build --packages - select <package_name>
cd ..
source ~/ros2_ws/install/setup.bash
```

To check that your package is indeed there, go into the `ros2_ws` directory and use `ros2 pkg list`

## Offboard Control 
When creating an offboard control file that will be used to fly the drone autonomously, there are a few steps that need to be taken. 

The first step is creating the python file that will be used for the offboard control. This is done within the package. 

```bash
cd <package_name>
touch <file_name>

```

### QoS Settings Compatibility
Next, the QoS settings need to be established. QoS stands for Quality of Service (QoS) settings. These are like rules you set when subscribing or publishing to a topic to control how reliable, fast and resilient your messages are. These settings determine your:
* Storage history 
* Depth (Only applicable if you're storing last samples and not all)
* Reliability 
* Durability
* Deadline 
* Lifespan
* Liveliness

For a full explanation of what each of these elements mean please refer to the [ROS2 Documentation (QoS settings)](https://docs.ros.org/en/rolling/Concepts/Intermediate/About-Quality-of-Service-Settings.html) 

Now that we understand how QoS settings work for ROS2, we need to make sure that these settings hold for PX4. To subscribe to topics published by PX4, an appropriate QoS setting is needed. The default ROS 2 QoS settings are different from the settings used by PX4 so not all combinations of publisher - subscriber QoS settings are possible. More specifically, the default ROS2 subscriber settings for subscribing are not compatible but for publishing they are. Therefore at the beginning of every node where you will subscribe to a topic, you have to specify the specific QoS settings. 

We didn't find much concrete information about the QoS settings between PX4 and ROS2 but after asking ChatGPT, they gave us the following rule-of-thumb:

1. Best Effort for reliability 
2. Volatile durability as PX4 won't store past messages for us
3. Keep last history
4. Depth is either 1 or 10 so that there is enough buffer for minor delays. 

This transforms to the following in Python code:

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy

qos_profile = QoSProfile(
    reliability=ReliabilityPolicy.BEST_EFFORT,
    history=HistoryPolicy.KEEP_LAST,
    depth=10
)

```

It is important to note that since we are using MICRO XRCE-DDS middleware, we can only use reliability, durability, history and depth QoS settings and not deadline, lifespan and liveliness. 

### Subscribing & Publishing to Topics
PX4 has specific topics that you can subscribe and publish to. These can be found [here.](https://github.com/PX4/PX4-Autopilot/blob/main/src/modules/uxrce_dds_client/dds_topics.yaml). Inside the PX4 flight controller, these topics are referred to as [uORB messages.](https://docs.px4.io/main/en/msg_docs/) which will be used when subscribing to these topics. 

For this example, we want to put the drone into Offboard Mode so it can operate autonomously. Then we want the drone to take-off to a specific height. After we want to circle around and land. Let's break it down step-by-step.

**1. Creating a Node** \
This is quite straight forward. First import the ROS2 libraries to create a node then create the class and the initialiser definition. Make sure to add your QoS settings here. An example of this can be seen below:

```python
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy

# Node to start autonomous flight
class OffboardControl(Node):
    def __init__(self):
        super().__init__('NODE_NAME')
        qos_profile = QoSProfile(
            reliability=ReliabilityPolicy.BEST_EFFORT,
            history=HistoryPolicy.KEEP_LAST,
            depth=10
            )
```

**2. Import the relevant uORB messages from px4_msgs.msg** \
Refer to the documentation and choose your relevant uORB messages. For this example, we used OffboardControlMode (To set to Offboard Mode), TrajectorySetpoint (To give setpoints for autonomous flight), VehicleStatus (to check if the drone is in offboard mode and is armed) and VehicleCommand (to take-off). Once you import them, you can subscribe to these topics.

```python
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy




