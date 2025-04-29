In this section, we will bring you through how we set everything up to achieve autonomous operation of the drone. We will show you how we created our files for a simple test case where we took off and flew in a circle for a singular drone.

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

Now that we understand how QoS settings work for ROS2, we need to make sure that these settings hold for PX4. To subscribe to topics published by PX4, an appropriate QoS setting is needed. The default ROS 2 QoS settings are different from the settings used by PX4 so not all combinations of publisher - subscriber QoS settings are possible. More specifically, the default ROS2 subscriber settings for subscribing are not compatible but for publishing they are. Therefore at the beginning of every file where you will subscribe to a topic, you have to specify the specific QoS settings. 

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

It is important to note that since we are using MICRO XRCE-DDS middleware that we can only use reliability, durability, history and depth QoS settings and not deadline, lifespan and liveliness. 

### Subscribing & Publishing to Topics
PX4 has specific topics that you can subscribe and publish to. These can be found [here.](https://github.com/PX4/PX4-Autopilot/blob/main/src/modules/uxrce_dds_client/dds_topics.yaml). Inside the PX4 flight controller, these topics are referred to as [uORB messages.](https://docs.px4.io/main/en/msg_docs/) which will be used when subscribing to these topics. 

For this example, we want to put the drone into Offboard Mode so it can operate autonomously. Then we want the drone to take-off to a specific height. After we want to circle around. Let's break it down step-by-step.

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

```python
# Importing the ROS2 topics
import rclpy
import numpy as np
from rclpy.clock import Clock
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy

# Importing the PX4 topics 
from px4_msgs.msg import OffboardControlMode
from px4_msgs.msg import TrajectorySetpoint
from px4_msgs.msg import VehicleStatus
from px4_msgs.msg import VehicleControl 

# Node to start autonomous flight
class OffboardControl(Node):
    def __init__(self):
        super().__init__('Testing_publisher')
        qos_profile = QoSProfile(
            reliability=ReliabilityPolicy.BEST_EFFORT,
            history=HistoryPolicy.KEEP_LAST,
            depth=10
            )


        # SUBSCRIPTIONS
        # Vehicle Status
        self.subscriber_status = self.create_subscription(VehicleStatus,
            '/fmu/out/vehicle_status_v1',
            self.vehicle_status_callback,
            qos_profile)
        
        # PUBLISHERS
        self.publisher_offboard_mode = self.create_publisher(OffboardControlMode, '/fmu/in/offboard_control_mode', qos_profile)
        self.publisher_trajectory = self.create_publisher(TrajectorySetpoint, '/fmu/in/trajectory_setpoint', qos_profile)
        self.publisher_vehicle_command = self.create_publisher(VehicleCommand, '/fmu/in/vehicle_command', qos_profile)

        # CREATING A TIMER 
        timer_period = 0.02 # seconds 
        self.timer = self.create_timer(timer_period, self.cmdloop_callback)
        self.dt = timer_period

        # USED TO GO IN A CIRCLE
        self.declare_parameter('radius', 10.0)
        self.declare_parameter('omega', 5.0)
        self.declare_parameter('altitude', 5.0)

        # DEFINING THE NAVIGATION STATE OF THE DRONE AND IF IT IS ARMED
        self.nav_state = VehicleStatus.NAVIGATION_STATE_MAX
        self.arming_state = VehicleStatus.ARMING_STATE_DISARMED
        
        # USED TO PREVENT SPAM 
        self.takeoff_sent = False 

        # INITIAL SETTINGS FOR MANEOUVRE
        self.theta = 0.0
        self.radius = self.get_parameter('radius').value
        self.omega = self.get_parameter('omega').value
        self.altitude = self.get_parameter('altitude').value
 
    # SUBSCRIBER WILL INITIATE THIS FUNCTION RESULTING IN A PRINT STATEMENT
    def vehicle_status_callback(self, msg):
        print("NAV_STATUS: ", msg.nav_state)
        print("  - offboard status: ", VehicleStatus.NAVIGATION_STATE_OFFBOARD)
        self.nav_state = msg.nav_state
        self.arming_state = msg.arming_state


    def cmdloop_callback(self):
        # Publish offboard control modes
        offboard_msg = OffboardControlMode()
        offboard_msg.timestamp = int(Clock().now().nanoseconds / 1000)

        # USING POSITION SET POINTS
        offboard_msg.position = True
        offboard_msg.velocity = False
        offboard_msg.acceleration = False
        self.publisher_offboard_mode.publish(offboard_msg)

        # ONCE THE DRONE IS ARMED ALLOW IT TO TAKE OFF
        if (self.arming_state == VehicleStatus.ARMING_STATE_ARMED) and not self.takeoff_sent:
            self.publish_takeoff() # CALLING PUBLISH TAKEOFF DEFINITION
            self.takeoff_sent = True

        # Normal trajectory loop
        # If the drone is 
        if (self.nav_state == VehicleStatus.NAVIGATION_STATE_OFFBOARD and self.arming_state == VehicleStatus.ARMING_STATE_ARMED):
            trajectory_msg = TrajectorySetpoint()
            trajectory_msg.position[0] = self.radius * np.cos(self.theta)
            trajectory_msg.position[1] = self.radius * np.sin(self.theta)
            trajectory_msg.position[2] = -self.altitude
            self.publisher_trajectory.publish(trajectory_msg)

            self.theta = self.theta + self.omega * self.dt

    def publish_takeoff(self):
        takeoff_cmd = VehicleCommand()
        takeoff_cmd.timestamp = int(Clock().now().nanoseconds / 1000)
        takeoff_cmd.param1 = 2.5  # minimum pitch (not too important here)
        takeoff_cmd.param7 = -self.altitude  # Altitude in meters
        takeoff_cmd.command = VehicleCommand.VEHICLE_CMD_NAV_TAKEOFF

        takeoff_cmd.target_system = 1 # Drone 1
        takeoff_cmd.target_component = 1 # Sent To Autopilot
        takeoff_cmd.source_system = 1 # Companion Computer Sending the Command
        takeoff_cmd.source_component = 1 # Default Component
        takeoff_cmd.from_external = True # External source providing the command (ME)

        self.get_logger().info('Sending takeoff command')
        self.publisher_vehicle_command.publish(takeoff_cmd)

def main(args=None):
    rclpy.init(args=args)

    offboard_control = OffboardControl()

    rclpy.spin(offboard_control)

    offboard_control.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```
## Launch file 


## Testing 

