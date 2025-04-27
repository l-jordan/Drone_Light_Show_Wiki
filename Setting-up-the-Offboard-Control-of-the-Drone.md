In this section, we will bring you through how we set everything up to achieve autonomous operation of the drone. 

### Creating a Package 
First we created a package which will house our autonomous operating files. 

To do this, we used the following commands: \

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

### Offboard Control 


### Launch file 


### Testing 

