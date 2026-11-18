Welcome to the drone_light_show wiki! Here you will find how Hubert and Lauren built their simulation. For this Capita Selecta project, we are building a drone light show using three autonomously driven drones running PX4 Autopilot software. We will do this using Gazebo Ignition as our simulation environment and ROS2 as our autonomous communicator between the drone and the companion computer. This wiki will bring you step-by-step through our procedure to create this simulation from start to finish.

Since the development belongs to a research group at TU Delft, the full repository cannot be shared. However, to rebuild our simulation, please follow the steps in our wiki. To run the simulation from start to finish, please refer to **Simulation Setup**.

Before trying to replicate the development process, please make sure you have followed the environment set-up guidelines below:

# Drone Light Show Set-up
This section will show you how to set up your environment so you have access to ROS2, Gazebo and PX4, along with creating your workspace for the single vehicle test example. It will also show you how to run the simulation provided. 

For more information about the development of this simulation, please refer to our wiki. 

*Note: This setup is for Windows!*
  
## Setting up the Development Environment 
A development environment is achieved through the use of devcontainers in VSCode. 

### What is this Development Environment and why use one?

This repository sets up a **ROS 2 (Robot Operating System 2) development environment** within a Docker container for this drone simulation project. It allows you to easily build, develop, and run ROS 2-based applications without the need to configure dependencies or worry about compatibility issues between different systems when collaborating with others. It is used to keep things consistent between machines so the setup is the same for everyone. Docker allows us to isolate the development environment from our local systems so we don't have any conflicts between other software on our laptops. 

### Prerequisites
Before developing the simulation, please make sure you have the following installed: 

- **Docker**: Ensure that Docker is installed and running on your system. You can follow the [official Docker installation guide](https://docs.docker.com/get-docker/) for your operating system.
- **Docker Desktop**: Make sure Docker Desktop is installed and running on your system. Here is the [installation guide](https://www.docker.com/products/docker-desktop/).
- **VS Code**: For editing and working with the ROS 2 workspace, it is recommended to use Visual Studio Code with the [Remote - Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers).

Make sure you also have the following dependencies set up so you can work with development containers:
- vscode, setup for [dev containers](https://code.visualstudio.com/docs/devcontainers/containers)
- [docker engine](https://docs.docker.com/engine/install/ubuntu/)

# Table of Contents - Wiki
1. [Building the Docker Container]([0.-Building-the-Docker-Container.md](0.-Building-the-Docker-Container.md))
2. [Creating the Test File for a Single Drone]([1.-Creating-Test-File-for-a-Single-Drone.md](1.-Creating-Test-File-for-a-Single-Drone.md)) for a simple circle trajectory \
2.A. [PX4 Topics: How to Use Them]([1.A.-PX4-Topics:-How-to-Use-Them.md](1.A.-PX4-Topics:-How-to-Use-Them.md))
3. [Expanding Test File to Multi-Vehicle Application]([2.-Expanding-Test-File-to-Multi-Vehicle-Application.md](2.-Expanding-Test-File-to-Multi-Vehicle-Application.md)) for a simple triangle configuration \
3.A. [Frame Transformation]([2.A.-Frame-Transformation.md](2.A.-Frame-Transformation.md))
4. [Path Planning]([3.-Path-Planning.md](3.-Path-Planning.md))
5. [Building the Gazebo World]([4.-Building-Gazebo-World.md](4.-Building-Gazebo-World.md))
6. [Implementing Path Planning to Multi Node Simulation]([5.-Implementing-Path-Planning-to-Multi‐Node-Simulation.md](5.-Implementing-Path-Planning-to-Multi‐Node-Simulation.md))
7. [Final Remarks]([6.-Final-Remarks.md](6.-Final-Remarks.md))

# Simulation Setup

## ROS2 Workspace 
Once you've built the simulation, here are the instructions to run the simulation from start to finish. 

1. Source the ROS 2 development environment in the current terminal and compile the workspace using colcon. Please keep in mind that this may take up to 20 minutes. 
```
  cd ..
  source /opt/ros/humble/setup.bash
  colcon build
```
1. a. If you have already built the package using colcon build and need to implement changes to a file, rebuild the changed file by doing:
```
colcon build --packages-select <NAME_OF_PACKAGE>
```

2. Source your directory to access the newly built (or rebuilt) package by typing:
```
cd ..
source ros2_ws/install/setup.bash
```

This builds all the folders under /src using the sourced toolchain. To run the example to ensure your setup was done correctly, please refer to the [official PX4 documentation](https://docs.px4.io/main/en/ros2/user_guide.html#running-the-example). 

## Establishing Communication between the Agents and the Client 
PX4 is the autopilot firmware which acts as the flight control for the drone. This firmware includes many flight modes that help with the control of the drone, such as holding attitude, acrobatic mode for drone racing but also autonomous modes such as return-to-launch and an offboard mode. This offboard mode will be used for this simulation. The [offboard mode](https://docs.px4.io/main/en/flight_modes/offboard.html) allows the companion computer on the drone to take over its operation autonomously by giving the controller control inputs. For us, this is the ROS2 framework that we develop. On the drone, there is a client which allows communication with our ROS2 network. The client communicates with [uORB](https://docs.px4.io/main/en/msg_docs/) topics in the PX4 firmware to turn the commands into code. Our ROS2 network communicates with this client through the agent. This communication link can be seen in the image below. The set-up of the client and agent has already been done when building the development environment. 

![image](https://github.com/user-attachments/assets/1a828b53-ca2c-4274-822d-ad8c367571f2)

To establish this communication, the following should be done:

1. Start the agent in a designated terminal (2nd terminal)
```bash
MicroXRCEAgent udp4 -p 8888
```

2. In a new (third) terminal, start a PX4-Gazebo simulation with the drone in the PX4-Autopilot directory.
```bash
cd ..
cd PX4-Autopilot
PX4_SYS_AUTOSTART=4001 PX4_SIM_MODEL=gz_x500 PX4_GZ_WORLD=sim_world ./build/px4_sitl_default/bin/px4 -i 1
```
  
  - PX4_SYS_AUTOSTART (Mandatory): Sets the airframe autostart id of the PX4 airframe to start.
  
  - PX4_GZ_MODEL_POSE: Sets the spawning position and orientation of the model when PX4_SIM_MODEL is adopted. If provided, the startup script spawns the model at a pose following the syntax "x,y,z,roll,pitch,yaw", where the positions are given in metres and the angles are in radians. (used for the next two drones to make sure they are not placed on top of the previously instantiated drones)
    - If omitted, the zero pose [0,0,0,0,0,0] is used.
    - If less then 6 values are provided, the missing ones are fixed to zero. (this is used by only specifying the x and y positions of the two extra drones later on)
  
  - PX4_GZ_STANDALONE: Lets PX4 know that it should not launch an instance of Gazebo (used when launching the other two drones).

  - PX4_GZ_WORLD=sim_world: tells the simulation to start in the custom-made gazebo world instead of the default blank one
  
  - The last "item" in the command is "-i 1"; this specifies the instance of the launched drone, which are 1, 2, and 3 for the three launched drones in this simulation.
      - The simulation then automatically generates a name for the drones, which is based on the instance number - for drone 1 with instance 1, it is "px4_1". Thus, it is important to only use instances 1, 2, and 3 in this simulation as this sets up the relationships and the publisher/subscriber relationships in the background.

3. The PX4 terminal should display:
```
   ...
  INFO  [uxrce_dds_client] synchronized with time offset 1675929429203524us
  INFO  [uxrce_dds_client] successfully created rt/fmu/out/failsafe_flags data writer, topic id: 83
  INFO  [uxrce_dds_client] successfully created rt/fmu/out/sensor_combined data writer, topic id: 168
  INFO  [uxrce_dds_client] successfully created rt/fmu/out/timesync_status data writer, topic id: 188
...
```
The Micro XCRE-DDS agent terminal will show the equivalent topics that are created in the DDS network.
```
...
  [1675929445.268957] info     | ProxyClient.cpp    | create_publisher         | publisher created      | client_key: 0x00000001,   publisher_id: 0x0DA(3), participant_id: 0x001(1)
  [1675929445.269521] info     | ProxyClient.cpp    | create_datawriter        | datawriter created     | client_key: 0x00000001,   datawriter_id: 0x0DA(5), publisher_id: 0x0DA(3)
  [1675929445.270412] info     | ProxyClient.cpp    | create_topic             | topic created          | client_key: 0x00000001, topic_id: 0x0DF(2), participant_id: 0x001(1)
  ...
```

2.a. Once the terminal displays the following message after launching the drone...:
```
INFO  [px4] Startup script returned successfully
pxh> ...
```
2.b. ...On a new line, input the command below to bypass certain pre-arming checks that are not applicable for this simulation.
```
param import /home/ros/ros2_ws/working.params
```

Continue by launching the other two drones analogously, but pay attention to things like the model poses (their starting coordinates) and the instance numbers at the end of the command lines (-i 2 and -i 3 respectively):

3. In a new (fourth) terminal, launch the **SECOND** drone in the PX4-Autopilot directory.
```bash
cd ..
cd PX4-Autopilot
PX4_GZ_STANDALONE=1 PX4_SYS_AUTOSTART=4001 PX4_GZ_MODEL_POSE="0,1" PX4_GZ_WORLD=sim_world PX4_SIM_MODEL=gz_x500 ./build/px4_sitl_default/bin/px4 -i 2
```
3.a. **!** Don't forget to specify the simulation parameters by inserting the following line after the same terminal message as before:
```
param import /home/ros/ros2_ws/working.params
```
4. In a new  (fifth) terminal, launch the **THIRD** drone in the PX4-Autopilot directory.
```bash
cd ..
cd PX4-Autopilot
PX4_GZ_STANDALONE=1 PX4_SYS_AUTOSTART=4001 PX4_GZ_MODEL_POSE="0,2" PX4_GZ_WORLD=sim_world PX4_SIM_MODEL=gz_x500 ./build/px4_sitl_default/bin/px4 -i 3
```

4.a. **!** Don't forget to specify the simulation parameters by inserting the following line:
```
param import /home/ros/ros2_ws/working.params
```

## Running the Simulation

1. Open a new terminal and source ROS2 and the ROS2 workspace
```
cd ..
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
```

2. Once everything has been sourced, you can now launch the simulation. 
```
cd ros2_ws
ros2 launch offboard_control_pkg offboard_control_launch_file.launch.py
```
If everything went well, the simulation should be up and running, and after a short while, the drones should start flying.

For more information about our development and how to run our simulation, please refer to our [wiki](https://github.com/p4mars/drone_light_show/wiki).
