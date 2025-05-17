Welcome to the drone_light_show wiki! Here you will find how Hubert and Lauren built their simulation. For this Capita Selecta project, we are building a drone light show using three autonomously driven drones running PX4 Autopilot software. We will do this using Gazebo Ignition as our simulation environment and ROS2 as our autonomous communicator between the drone and the companion computer (how it would be done in reality). This wiki will bring you step-by-step through our procedure to create this simulation from start to finish.

Before trying to replicate the development process, please make sure you have followed the environment set-up guidelines in our README.md file. 

**Table of Contents**
[Building the Docker Container](https://github.com/p4mars/drone_light_show/wiki/0.-Building-the-Docker-Container)
[Creating the Test File for a Single Drone](https://github.com/p4mars/drone_light_show/wiki/1.-Creating-Test-File-for-a-Single-Drone)
[(Appendix) PX4 Topics: How to Use Them](https://github.com/p4mars/drone_light_show/wiki/1.A.-PX4-Topics:-How-to-Use-Them)
[Expanding Test File to Multi-Vehicle Application](https://github.com/p4mars/drone_light_show/wiki/2.-Expanding-Test-File-to-Multi-Vehicle-Application)
[(Appendix) Frame Transformation](https://github.com/p4mars/drone_light_show/wiki/2.A.-Frame-Transformation)
[Path Planning](https://github.com/p4mars/drone_light_show/wiki/3.-Path-Planning)
[Building the Gazebo World](https://github.com/p4mars/drone_light_show/wiki/4.-Building-Gazebo-World)
[Implementing Path Planning](https://github.com/p4mars/drone_light_show/wiki/5.-Implementing-Path-Planning)
[Final Remarks](https://github.com/p4mars/drone_light_show/wiki/6.-Final-Remarks)