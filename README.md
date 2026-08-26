# Documentation

## 1. Introduction

### 1.1. Background and Motivation
Controlling unstable systems that tend to fall over is a fundamental challenge in robotics and control theory. A two-wheeled self-balancing robot is a clear example of this: it is a moving system that is naturally unstable and requires constant adjustment. By developing such systems, we can apply control systems theory about PID regulation.

### 1.2. Project Objectives
The primary objective of this project is to design, develop, and implement embedded firmware for a two-wheeled self-balancing robot that achieves:
* Stable vertical posture retention in both static and dynamic operating conditions within a real-time execution framework.
* Acquisition and processing of data from a 6-axis inertial measurement unit (IMU) with high-frequency noise suppression and gyro drift mitigation.
* Reception and execution of wireless movement commands transmitted from a remote interface (such as a gesture glove) via an RF radio link.
* Integration of hardware quadrature encoders for precise velocity feedback and accurate motor torque management.

### 1.4. Key Tasks
To accomplish the stated objective, the following tasks were executed:
* Configured and mapped the peripheral modules of the STM32F030C8Tx microcontroller, including I2C1, SPI1, encoder timer interfaces (TIM1, TIM3), and PWM output channels (TIM15).
* Integrated the MPU6050 IMU and implemented a pitch/roll attitude estimation algorithm using a complementary filter with a deterministic loop interval of $\Delta t = 4\text{ ms}$ ($250\text{ Hz}$).
* Designed a cascaded PID control architecture featuring an inner loop for tilt angle stabilization and an outer loop for wheel speed regulation combined with steering command blending.
* Established a wireless telemetry subsystem using the NRF24L01 module, incorporating a safety failsafe mechanism to trigger emergency shutdowns upon packet loss.
* Implemented the L298N motor driver control interface with deadband compensation to overcome motor static friction based on the defined `MIN_PWM` threshold.

---

## 2. System Architecture & Mechanical Design

### 2.1. Overview of the Development Lifecycle
The development of the self-balancing robotic platform followed a structured design approach, combining 3D computer modeling, 3D printing, electronic component assembly, and writing the microcontroller program. The design process focused on building a strong structure, compact parts layout, and precise sensor placement to ensure the best possible control performance.

### 2.3. Physical Layout & Component Integration
The internal layout of the chassis was organized in layers to manage weight distribution and electrical safety. DC motors and motor drivers (L298N) were placed at the lower section of the chassis to keep the center of gravity as low as possible. Additionally, the battery pack was integrated into the lower tier, serving both as the main power source and as heavy ballast to lower the center of mass, which makes it easier for the control system to keep the robot balanced. Intercard connections used custom wiring and precise soldering to route signals cleanly between tiers while preventing wire strain during movement.

### 2.4. IMU Sensor Alignment & Placement
A critical factor in keeping the robot balanced is the accurate placement of the sensor module. The 6-axis MPU6050 sensor (which combines a 3-axis accelerometer and a 3-axis gyroscope) was mounted precisely along the central Y-axis of the robot's symmetry plane. Placing the sensor centrally ensures that movement and acceleration do not create measuring errors, allowing the system to accurately calculate the tilt angles needed by the control loops.

---

## 3. List of Components and Hardware Architecture
This section outlines the primary electronic components integrated into the self-balancing robot, summarized in the table below:

| Component | Specifications / Type |
| :--- | :--- |
| **STM32F030C8Tx Microcontroller** | ARM Cortex-M0 32-bit RISC core (LQFP48 package) |
| **MPU6050 Inertial Measurement Unit** | 6-axis MotionTracking device (3-axis accelerometer & 3-axis gyroscope) |
| **NRF24L01+ Wireless Transceiver Module** | 2.4 GHz RF wireless transceiver module (2 pieces) |
| **L298N Dual H-Bridge Motor Drive Controller Board** | Stepper Motor DC Dual H-Bridge driver module |
| **DC12V Encoder Gear Motor with Mounting Bracket** | 65mm Magnetic Motor for Smart Car Robot (130 RPM) with integrated quadrature encoders |
| **Multicolored Dupont Wire Breadboard Jumper Ribbon Cables Kit** | 120 pcs (40pin M/F, 40pin M/M, 40pin F/F, 20cm/8inch) |

---

## 4. Control System Design & PID Implementation
To achieve steady balance and smooth movement, the control system uses a cascaded PID architecture that runs in a strict real-time loop every $4\text{ ms}$ ($250\text{ Hz}$).

### 4.1. Inner Loop: Tilt Angle Stabilization (Angle PID)
The main goal of the inner loop is to keep the robot standing straight by fighting gravity and catching it before it falls. It compares the current filtered tilt angle (roll, derived from the MPU6050 via a complementary filter) against a dynamic target angle:
$$\text{current\_target\_angle} = \text{pid\_roll.target\_angle} + \text{rxData.channel1}$$

The system computes the proportional, integral, and derivative terms:
$$u_{\text{angle}} = K_p e_t + K_i \int e_t dt + K_d \frac{de_t}{dt}$$

To prevent integral saturation during prolonged tilts or external disturbances, anti-windup protection is applied to strictly clamp the integral term. The final output generates the baseline balance speed required to drive the wheels underneath the shifting center of gravity.

### 4.2. Outer Loop: Motor Speed & Steering Control (Motor PID)
While the inner loop keeps the robot upright, the outer loop controls individual wheel speeds and turning trajectories. Movement commands received from the gesture glove (`rxData.channel2` for turning offsets) are directly blended with the baseline balance output to compute individual wheel target speeds:
$$\text{TargetLeft} = \text{BalanceOutput} + \text{SteeringOffset}$$
$$\text{TargetRight} = \text{BalanceOutput} - \text{SteeringOffset}$$

The system then compares these target speeds against actual velocities measured via hardware quadrature encoders (TIM1 and TIM3), computing individual motor PID corrections to adjust the final PWM duty cycles applied to each wheel.

### 4.3. Safety Interlocks & Deadband Compensation
To keep the robot safe and operational, the firmware includes critical safety checks. If the measured robot tilt angle exceeds $\pm 30^\circ$, an emergency cutoff protocol (`Motor_Emergency_Stop`) instantly zeroes out all PWM outputs, resets PID integral accumulators, and halts actuation to prevent physical damage. Additionally, to overcome static friction in the DC motors, a minimum threshold (`MIN_PWM = 1200`) is injected into non-zero PWM calculations before updating the TIM15 compare registers (capped at a maximum period of 2399).




