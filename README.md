# Design Proposal

## Setup

Edukit rotary inverted pendulum.

Components:
  - Stepper motor for rotating the pendulum base.
  - Encoder for reading the position of the pendulum.
  - Nucleo expansion card (X-NUCLEO-IHM01A1) with a L6474PD motor driver. Motor and encoder are connected to the expansion card.
  - STM32F041 Nucleo board on which the expansion card is installed.
  - A computer running a MARTe2 application connected via a USB to the Nucleo board. (Not part of the Edukit).

## Existing Solutions

1) UCLA's Edukit comes with a program running on STM32 that is already capable of balancing the pendulum.
2) A MARTe2 port of the UCLA solution.

## Goal

Existing MARTe2 solution simply moved part of the UCLA's code into a single custom GAM. To better demonstrate the use of functionalities provided by MARTe2 a new solution is needed.

## New Solution

This section described not only the proposed solution, but also the thought process behind it.

### DataSource

Since a general DataSource for controlling an L6474 motor driver connected to STM32 does not yet exist, a new DataSource must be written.

First, we have to decide what functionality belongs on the SMT32 and what functionality belongs in MARTe2. For example, in UCLA's solution, all the logic resides on the STM32. In the existing MARTe2 solution, majority of the logic was transferred to MARTe2, while some parts were left on the STM32 (configuration of the motor, moving the motor at initialization, waiting for the move to finish...).

If we want to create a general DataSource, which could be used for other projects using similar hardware, we must treat the STM32 as a simple motor controller. It would only expose the functionalities of the motor (move to position, read status, set speed, set acceleration, read position...), while the project specific code would be put into MARTe2.

The existing software drivers provided by ST could help us develop a general DataSource. The ST drivers already have an abstract concept of a motor with specific implementation depending on the motor driver type (in our case, that is L6474). So, in theory, we could expose this common motor interface to our DataSource, which would allow it to communicate with different motors. However, the interface provided by ST does not allow us to move the motor or change its speed if it is already moving. Waiting for the motor to stop in order to change its speed would make it more difficult or even impossible to balance the pendulum. This is most likely why the UCLA code bypasses the high-level motor interface, and controls the speed of the motor using the L6474 specific functions. Unfortunately, this means that for every different hardware setup, a custom STM32 code would need to be written.

Proposed functionalities of the DataSource:
  - Support for full motor configuration and control at startup. Configuration parameters are defined in MARTe2 configuration file.
  - Optionaly support for changing the configuration during runtime via MARTe2 messages.
  - Custom input broker, that reads an array of values from the STM32. It is up to the STM32 code to define the number and meaning of values, and up to GAMs to interpret these values correctly. In our case, these values would be MotorState, MotorPosition, MotorSpeed, EncoderPosition.
  - Custom output broker, that sends one of multiple available move commands. Examples would be: moveRelative, moveAbsolute, setRunSpeed.

Detailed description:
  - Motor is configured at startup with the values provided in the MARTe2 configuration file. Name of the parameter determines which API call is to be executed. Order of parameters defines the order of API calls. The whole interface of an abstract motor (as provided by ST) is exposed.
  - Output broker sends a command to STM32 depending on the value of the first three signals (Command, Param1, Param2). Possible commands are: move absolute, move relative, set speed and direction, no command.
  - Input broker sends a readout status command every time it is executed. Response consists of an array of bytes. The response is interpreted according to provided input signals starting at 4th signal (first three signals are used for sending commands). Order of input signals is important and must match that of the STM32 code.

Example of DataSource configuration:
```
+Motor = {
    Class = STM32Motor
    // Communication channel configuration
    Port = "/dev/ttyACM0"
    BaudRate = 230400
    // Motor configuration
    SetMaxSpeed = 800
    SetMinSpeed = 200
    ...
    Signals = {
        // Signals for moving the motor. Names, order and types must be as defined here.
        Command = {
            Type = uint8
        }
        Param1 = {
            Type = uint32
        }
        Param2 = {
            Type = uint32
        }
        // Signals defining the layout of status response. In this case response
        // consists of 14 bytes starting with a 32-bit unsigned integer and ending with
        // an 8-bit unsigned integer.
        EncoderPosition = {
            Type = uint32
        }
        MotorPosition = {
            Type = uint32
        }
        MotorSpeed = {
            Type = uint32
        }
        MotorDiretion = {
            Type = uint8
        }
        MotorStatus = {
            Type = uint8
        }
    }
}
```

### State Machine

Balancing of the pendulum can be divided into four stages:
  - Startup - move the motor back and forth so that the pendulum "falls" if it is currently balanced.
  - Homing - wait for the pendulum to come to rest and determine the lowest pendulum point.
  - Swing up - swing the pendulum from the lowest point towards the highest position.
  - Balance - keep the pendulum balanced near the highest point.

MARTe2 offers RealTimeApplication states, and at first glance, we could implement each stage in its own RealTimeApplication state. The problem however is that transition between states is not deterministic. This is not a problem for transition between startup, homing, and swing up stages. However, for the transition between swing up and balance stages, this might present a problem. If RealTimeApplication state switch takes too long, the pendulum might move too far away from the highest position, and we are no longer able to balance it. The proposed approach is therefore, to have three RealTimeApplication states with the following tasks:
  1) Move the motor to make sure the pendulum is not balanced (startup stage).
  2) Execute the homing procedure (homing stage).
  3) Swing up the pendulum and then balance it (swing up and balance stage).
Since the swing up and the balancing of the pendulum is done within the same RealTimeApplication state, we no longer have to worry about pendulum moving too far away from the highest point before the balancing code starts executing.

Non-real-time states (1 and 2) would be synchronized to LinuxTimer cycle, while the hard real-time thread (3) could run at maximum frequency (limited only by the execution time of GAMs and STM32 communication latency).

### GAMs

Most likely a custom GAM for each state would be the easiest way.

First state is the only simple enough that it might make sense to implement it using existing GAMs. Even if the solution with existing GAMs is a bit more complicated compared to a custom GAM, it might make sense to do it with existing GAMs, to demonstrate how one would create a MARTe2 application without the need of custom GAMs.

For the last state it might make sense to have two custom GAMs. One for the swing up and one for the balancing. This way the state can later be split into two RealTImeApplication states with minimal changes to the code, which would allow us to compare the two different approches (internal states vs. RealTimeApplication states).

Extra IOGAMs could be used in each state to write the motor position and encoder position to a GAMDataSource. This would allow us to visualize the position in a browser using the HttpObjectBrowser and HttpService, without writing any extra code for exposing these values from custom GAMs and DataSource.

We could also use ConversionGAMs to convert between units understandable to the motor and units understandable to the algorithm for balancing the pendulum.
