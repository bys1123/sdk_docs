# Example: VTOL Transitions

This example shows how you can use the SDK [Action](../api_reference/classdronecode__sdk_1_1_action.md) class to transition between VTOL copter and fixed-wing modes (and back).

![VTOL Transition QGC Screenshot](../../assets/examples/transition_vtol_fixed_wing/transition_vtol_fixed_wing_example_qgc.png)


## Running the Example {#run_example}

The example must be run against a VTOL aircraft (simulated or otherwise). 
Otherwise the example is built and run [in the standard way](../examples/README.md#trying_the_examples).

> **Tip** Instructions for running the Gazebo simulator for a standard VTOL can be found here: [PX4 Development Guide > Gazebo Simulation](https://dev.px4.io/en/simulation/gazebo.html#standard-vtol). 
  jMAVSim does not support VTOL simulation.

The example terminal output for a debug build of the SDK should be similar to that shown below (a release build will omit the "Debug" messages):

```
$ ./transition_vtol_fixed_wing udp://:14540
```
```
Waiting to discover system...
[10:24:42|Info ] New system on: 127.0.0.1:14557 (udp_connection.cpp:210)
[10:24:42|Debug] MAVLink: info: [logger] file: rootfs/fs/microsd/log/2017-11-21/0 (mavlink_system.cpp:286)
[10:24:43|Debug] Discovered 4294967298 (mavlink_system.cpp:483)
Discovered system with UUID: 4294967298
Arming...
Taking off...
[10:24:44|Debug] MAVLink: info: ARMED by arm/disarm component command (mavlink_system.cpp:286)
[10:24:44|Debug] MAVLink: info: Using minimum takeoff altitude: 10.00 m (mavlink_system.cpp:286)
[10:24:44|Debug] MAVLink: info: Takeoff detected (mavlink_system.cpp:286)
[10:24:44|Debug] MAVLink: critical: Using minimum takeoff altitude: 10.00 m (mavlink_system.cpp:286)
Altitude: 0.079 m
Altitude: 0.507 m
...
Altitude: 10.254 m
Transition to fixedwing...
Altitude: 10.263 m
...
Altitude: 20.72 m
Altitude: 24.616 m
Altitude: 22.262 m
Transition back to multicopter...
Altitude: 17.083 m
...
Return to launch...
Altitude: 11.884 m
[10:25:09|Debug] MAVLink: info: RTL: climb to 518 m (29 m above home) (mavlink_system.cpp:286)
Altitude: 13.61 m
...
Altitude: 27.489 m
Altitude: 28.892 m
[10:25:18|Debug] MAVLink: info: RTL: return at 517 m (29 m above home) (mavlink_system.cpp:286)
Altitude: 29.326 m
Altitude: 29.33 m
...
Altitude: 29.323 m
Altitude: 29.357 m
Landing...
[10:25:29|Debug] MAVLink: info: Landing at current position (mavlink_system.cpp:286)
Altitude: 29.199 m
Altitude: 28.722 m
Altitude: 28.189 m
Altitude: 27.62 m
Finished...
```


## How it works

The operation of the transition code is discussed in the guide: [Takeoff and Landing (and other actions)](../guide/taking_off_landing.md#transition_vtol).

## Source code {#source_code}

> **Tip** The full source code for the example [can be found on Github here](https://github.com/Dronecode/DronecodeSDK/tree/{{ book.github_branch }}/example/transition_vtol_fixed_wing).


[CMakeLists.txt](https://github.com/Dronecode/DronecodeSDK/blob/{{ book.github_branch }}/example/transition_vtol_fixed_wing/CMakeLists.txt)

```make
cmake_minimum_required(VERSION 2.8.12)

project(transition_vtol_fixed_wing)

if(NOT MSVC)
    add_definitions("-std=c++11 -Wall -Wextra -Werror")
    # Line below required if /usr/local/include is not in your default includes
    #include_directories(/usr/local/include)
    # Line below required if /usr/local/lib is not in your default linker path
    #link_directories(/usr/local/lib)
else()
    add_definitions("-std=c++11 -WX -W2")
    include_directories(${CMAKE_SOURCE_DIR}/../../install/include)
    link_directories(${CMAKE_SOURCE_DIR}/../../install/lib)
endif()

add_executable(transition_vtol_fixed_wing
    transition_vtol_fixed_wing.cpp
)

target_link_libraries(transition_vtol_fixed_wing
    dronecode_sdk
    dronecode_sdk_action
    dronecode_sdk_telemetry
)
```

[transition_vtol_fixed_wing.cpp](https://github.com/Dronecode/DronecodeSDK/blob/{{ book.github_branch }}/example/transition_vtol_fixed_wing/transition_vtol_fixed_wing.cpp)

```cpp
#include <chrono>
#include <cstdint>
#include <iostream>
#include <thread>
#include <cmath>
#include <dronecode_sdk/action.h>
#include <dronecode_sdk/dronecode_sdk.h>
#include <dronecode_sdk/telemetry.h>

using std::this_thread::sleep_for;
using std::chrono::seconds;
using std::chrono::milliseconds;
using namespace dronecode_sdk;

static constexpr auto ERROR_CONSOLE_TEXT = "\033[31m";
static constexpr auto TELEMETRY_CONSOLE_TEXT = "\033[34m";
static constexpr auto NORMAL_CONSOLE_TEXT = "\033[0m";

void usage(const std::string &bin_name);

int main(int argc, char **argv)
{
    if (argc != 2) {
        usage(argv[0]);
        return 1;
    }

    const std::string connection_url = argv[1];

    DronecodeSDK dc;

    // Add connection specified by CLI argument.
    const ConnectionResult connection_result = dc.add_any_connection(connection_url);
    if (connection_result != ConnectionResult::SUCCESS) {
        std::cout << ERROR_CONSOLE_TEXT
                  << "Connection failed: " << connection_result_str(connection_result)
                  << NORMAL_CONSOLE_TEXT << std::endl;
        return 1;
    }

    // We need an autopilot connected to start.
    while (!dc.system().has_autopilot()) {
        sleep_for(seconds(1));
        std::cout << "Waiting for system to connect." << std::endl;
    }

    // Get system and plugins.
    System &system = dc.system();
    auto telemetry = std::make_shared<Telemetry>(system);
    auto action = std::make_shared<Action>(system);

    // We want to listen to the altitude of the drone at 1 Hz.
    const Telemetry::Result set_rate_result = telemetry->set_rate_position(1.0);
    if (set_rate_result != Telemetry::Result::SUCCESS) {
        std::cout << ERROR_CONSOLE_TEXT
                  << "Setting rate failed: " << Telemetry::result_str(set_rate_result)
                  << NORMAL_CONSOLE_TEXT << std::endl;
        return 1;
    }

    // Set up callback to monitor altitude.
    telemetry->position_async([](Telemetry::Position position) {
        std::cout << TELEMETRY_CONSOLE_TEXT << "Altitude: " << position.relative_altitude_m << " m"
                  << NORMAL_CONSOLE_TEXT << std::endl;
    });

    // Wait until we are ready to arm.
    while (!telemetry->health_all_ok()) {
        std::cout << "Waiting for vehicle to be ready to arm..." << std::endl;
        sleep_for(seconds(1));
    }

    // Arm vehicle
    std::cout << "Arming." << std::endl;
    const ActionResult arm_result = action->arm();

    if (arm_result != ActionResult::SUCCESS) {
        std::cout << ERROR_CONSOLE_TEXT << "Arming failed: " << action_result_str(arm_result)
                  << NORMAL_CONSOLE_TEXT << std::endl;
        return 1;
    }

    // Take off
    std::cout << "Taking off." << std::endl;
    const ActionResult takeoff_result = action->takeoff();
    if (takeoff_result != ActionResult::SUCCESS) {
        std::cout << ERROR_CONSOLE_TEXT << "Takeoff failed:n" << action_result_str(takeoff_result)
                  << NORMAL_CONSOLE_TEXT << std::endl;
        return 1;
    }

    // Wait while it takes off.
    sleep_for(seconds(10));

    std::cout << "Transition to fixedwing." << std::endl;
    const ActionResult fw_result = action->transition_to_fixedwing();

    if (fw_result != ActionResult::SUCCESS) {
        std::cout << ERROR_CONSOLE_TEXT
                  << "Transition to fixed wing failed: " << action_result_str(fw_result)
                  << NORMAL_CONSOLE_TEXT << std::endl;
        return 1;
    }

    // Let it transition and start loitering.
    sleep_for(seconds(30));

    // Send it South.
    std::cout << "Sending it to location." << std::endl;
    // We pass latitude and longitude but leave altitude and yaw unset by passing NAN.
    const ActionResult goto_result = action->goto_location(47.3633001, 8.5428515, NAN, NAN);
    if (goto_result != ActionResult::SUCCESS) {
        std::cout << ERROR_CONSOLE_TEXT << "Goto command failed: " << action_result_str(goto_result)
                  << NORMAL_CONSOLE_TEXT << std::endl;
        return 1;
    }

    // Let it fly South for a bit.
    sleep_for(seconds(20));

    // Let's stop before reaching the goto point and go back to hover.
    std::cout << "Transition back to multicopter..." << std::endl;
    const ActionResult mc_result = action->transition_to_multicopter();
    if (mc_result != ActionResult::SUCCESS) {
        std::cout << ERROR_CONSOLE_TEXT
                  << "Transition to multi copter failed: " << action_result_str(mc_result)
                  << NORMAL_CONSOLE_TEXT << std::endl;
        return 1;
    }

    // Wait for the transition to be carried out.
    sleep_for(seconds(5));

    // Now just land here.
    std::cout << "Landing..." << std::endl;
    const ActionResult land_result = action->land();
    if (land_result != ActionResult::SUCCESS) {
        std::cout << ERROR_CONSOLE_TEXT << "Land failed: " << action_result_str(land_result)
                  << NORMAL_CONSOLE_TEXT << std::endl;
        return 1;
    }

    // Wait until disarmed.
    while (telemetry->armed()) {
        std::cout << "Waiting for vehicle to land and disarm." << std::endl;
        sleep_for(seconds(1));
    }

    std::cout << "Disarmed, exiting." << std::endl;

    return 0;
}

void usage(const std::string &bin_name)
{
    std::cout << NORMAL_CONSOLE_TEXT << "Usage : " << bin_name << " <connection_url>" << std::endl
              << "Connection URL format should be :" << std::endl
              << " For TCP : tcp://[server_host][:server_port]" << std::endl
              << " For UDP : udp://[bind_host][:bind_port]" << std::endl
              << " For Serial : serial:///path/to/serial/dev[:baudrate]" << std::endl
              << std::endl
              << "For example, to connect to the simulator use URL: udp://:14540" << std::endl;
}
```
