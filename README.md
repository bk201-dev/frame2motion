<div align="center">

<img src="assets/hero/frame2motion-hero.png" width="92%" alt="frame2motion — ROS robotic arm"/>

<br>

# frame2motion

### **From coordinate frames to physical motion.**

A ROS Noetic robotic-arm project connecting  
**kinematics → robot description → simulation → control → real servo actuation**

<br>

<p align="center">
  <img src="https://img.shields.io/badge/ROS-Noetic-111827?style=for-the-badge&logo=ros&logoColor=white" />
  <img src="https://img.shields.io/badge/C++-Control_Core-111827?style=for-the-badge&logo=cplusplus&logoColor=white" />
  <img src="https://img.shields.io/badge/URDF%20%2F%20Xacro-Robot_Model-111827?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Gazebo-Simulation-111827?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Arduino-Real_Actuation-111827?style=for-the-badge&logo=arduino&logoColor=white" />
</p>

<br>

> **A robot arm is not one mechanism. It is a chain of frames, transforms, software layers, commands and actuators — all agreeing on what “move” means.**

</div>

---

## ◈ The Motion Chain

<table>
<tr>
<td width="20%" align="center"><h3>01</h3><b>FRAME</b><br><sub>Where is it?</sub></td>
<td width="20%" align="center"><h3>02</h3><b>TRANSFORM</b><br><sub>How does it move?</sub></td>
<td width="20%" align="center"><h3>03</h3><b>MODEL</b><br><sub>What is connected?</sub></td>
<td width="20%" align="center"><h3>04</h3><b>CONTROL</b><br><sub>What should move?</sub></td>
<td width="20%" align="center"><h3>05</h3><b>ACTUATE</b><br><sub>Make it real.</sub></td>
</tr>
</table>

```text
REFERENCE FRAME  ──►  KINEMATICS  ──►  URDF / XACRO  ──►  ROS CONTROL  ──►  ARDUINO  ──►  SERVO MOTION
```

---

## ◈ System Choreography

Instead of treating simulation, ROS control and hardware as separate topics, **frame2motion** follows one continuous motion path.

```mermaid
%%{init: {"theme":"base","themeVariables":{"background":"#0d1117","primaryTextColor":"#f8fafc","lineColor":"#64748b"}}}%%
flowchart LR
    subgraph MATH["MOTION MODEL"]
        F["Reference Frames"] --> K["Kinematics"] --> TF["TF / Transforms"]
    end
    subgraph DIGITAL["DIGITAL TWIN"]
        U["URDF / Xacro"] --> R["RViz"]
        U --> G["Gazebo"]
    end
    subgraph CONTROL["ROS CONTROL"]
        J["Joint Commands"] --> CM["controller_manager"] --> HI["Hardware Interface"]
    end
    subgraph REAL["PHYSICAL ARM"]
        C["Angle Conversion"] --> RS["rosserial"] --> A["Arduino"] --> S["Servo Motion"]
    end

    TF --> U
    R --> J
    G --> J
    HI --> C

    classDef math fill:#111827,stroke:#38bdf8,color:#f8fafc,stroke-width:2px;
    classDef digital fill:#111827,stroke:#a78bfa,color:#f8fafc,stroke-width:2px;
    classDef control fill:#111827,stroke:#f59e0b,color:#f8fafc,stroke-width:2px;
    classDef real fill:#111827,stroke:#34d399,color:#f8fafc,stroke-width:2px;
    class F,K,TF math;
    class U,R,G digital;
    class J,CM,HI control;
    class C,RS,A,S real;
```

---

# Kinematic Atlas

## 01 — One point, two frames

Everything begins with one question:

> **How can the same physical point have different coordinates depending on the frame observing it?**

The project starts in 2D to make the geometry intuitive, then extends the same reasoning to 3D rigid-body motion.

<table>
<tr>
<td width="33%" align="center"><h3>Translation</h3><b>Move the origin</b><br><br><code>pW = pL + t</code></td>
<td width="33%" align="center"><h3>Rotation</h3><b>Change orientation</b><br><br><code>pW = R pL</code></td>
<td width="33%" align="center"><h3>Rigid Transform</h3><b>Do both at once</b><br><br><code>pW = R pL + t</code></td>
</tr>
</table>

<details>
<summary><b>▸ Open the 2D derivation</b></summary>

### Translation

```math
\boxed{
\mathbf{p}_W=\mathbf{p}_L+\mathbf{t}
}
```

### Rotation

```math
\mathbf{R}(\beta)=
\begin{bmatrix}
\cos\beta & -\sin\beta \\
\sin\beta & \cos\beta
\end{bmatrix}
```

```math
\boxed{
\mathbf{p}_W=\mathbf{R}(\beta)\mathbf{p}_L
}
```

### Homogeneous transformation

```math
\boxed{
{}^{W}\mathbf{T}_{L}=
\begin{bmatrix}
\cos\beta & -\sin\beta & L_x \\
\sin\beta &  \cos\beta & L_y \\
0 & 0 & 1
\end{bmatrix}
}
```

</details>

---

## 02 — From the plane to 3D

<details>
<summary><b>▸ Open the 3D rotation matrices</b></summary>

### Rotation around X

```math
\mathbf{R}_x(\gamma)=
\begin{bmatrix}
1 & 0 & 0 \\
0 & \cos\gamma & -\sin\gamma \\
0 & \sin\gamma & \cos\gamma
\end{bmatrix}
```

### Rotation around Y

```math
\mathbf{R}_y(\beta)=
\begin{bmatrix}
\cos\beta & 0 & \sin\beta \\
0 & 1 & 0 \\
-\sin\beta & 0 & \cos\beta
\end{bmatrix}
```

### Rotation around Z

```math
\mathbf{R}_z(\alpha)=
\begin{bmatrix}
\cos\alpha & -\sin\alpha & 0 \\
\sin\alpha & \cos\alpha & 0 \\
0 & 0 & 1
\end{bmatrix}
```

```math
\boxed{
\mathbf{R}
=
\mathbf{R}_z(\alpha)
\mathbf{R}_y(\beta)
\mathbf{R}_x(\gamma)
}
```

> Rotation order matters — 3D rotations are generally **not commutative**.

</details>

### 3D rigid-body transform

```math
{}^{W}\mathbf{T}_{L} =
\begin{bmatrix}
r_{11} & r_{12} & r_{13} & L_x \\
r_{21} & r_{22} & r_{23} & L_y \\
r_{31} & r_{32} & r_{33} & L_z \\
0 & 0 & 0 & 1
\end{bmatrix}
```

---

## 03 — A robot is a chain of frames

```mermaid
%%{init: {"theme":"base","themeVariables":{"background":"#0d1117","primaryTextColor":"#ffffff","lineColor":"#94a3b8"}}}%%
flowchart LR
    W(("WORLD")) --> B["base_link"] --> J1{"joint_1"} --> L1["link_1"] --> J2{"joint_2"} --> L2["link_2"] --> J3{"joint_3"} --> E(("END EFFECTOR"))

    classDef anchor fill:#0f172a,stroke:#38bdf8,color:#ffffff,stroke-width:3px;
    classDef link fill:#111827,stroke:#a78bfa,color:#ffffff,stroke-width:2px;
    classDef joint fill:#1f2937,stroke:#f59e0b,color:#ffffff,stroke-width:2px;
    classDef tip fill:#0f172a,stroke:#34d399,color:#ffffff,stroke-width:3px;

    class W anchor;
    class B,L1,L2 link;
    class J1,J2,J3 joint;
    class E tip;
```

```math
{}^{W}\mathbf{T}_{E}
=
{}^{W}\mathbf{T}_{1}
{}^{1}\mathbf{T}_{2}
{}^{2}\mathbf{T}_{3}
\cdots
{}^{n}\mathbf{T}_{E}
```

This is the mathematical backbone behind **forward kinematics**, **TF trees**, and pose propagation in ROS.

---

# Robot Description

## ◈ URDF / Xacro as the robot's structural language

```mermaid
%%{init: {"theme":"base","themeVariables":{"background":"#0d1117","primaryTextColor":"#f8fafc","lineColor":"#64748b"}}}%%
flowchart TB
    ROBOT["robot"]
    ROBOT --> LINKS["LINKS"]
    ROBOT --> JOINTS["JOINTS"]
    ROBOT --> MACROS["XACRO"]
    ROBOT --> PHYSICS["PHYSICAL DATA"]

    LINKS --> VIS["visual"]
    LINKS --> COL["collision"]
    LINKS --> INE["inertial"]

    JOINTS --> AXIS["axis"]
    JOINTS --> LIM["limits"]
    JOINTS --> ORG["origin"]

    MACROS --> PROP["properties"]
    MACROS --> REP["reusable macros"]

    PHYSICS --> MASS["mass"]
    PHYSICS --> INERTIA["inertia tensor"]

    classDef root fill:#111827,stroke:#38bdf8,color:#fff,stroke-width:3px;
    classDef branch fill:#111827,stroke:#a78bfa,color:#fff,stroke-width:2px;
    classDef leaf fill:#0f172a,stroke:#475569,color:#e2e8f0,stroke-width:1px;
    class ROBOT root;
    class LINKS,JOINTS,MACROS,PHYSICS branch;
    class VIS,COL,INE,AXIS,LIM,ORG,PROP,REP,MASS,INERTIA leaf;
```

<p align="center">
  <img src="assets/gallery/urdf-structure.png" width="82%" alt="URDF structure"/>
</p>

---

# Simulation Layer

<table>
<tr>
<td width="50%" align="center">
<h2>RViz</h2>
<b>Geometry + TF + state visualization</b><br><br>
<img src="assets/gallery/rviz-view.png" width="100%" alt="RViz robot visualization"/>
</td>
<td width="50%" align="center">
<h2>Gazebo</h2>
<b>Physics + spawning + controller integration</b><br><br>
<img src="assets/gallery/gazebo-view.png" width="100%" alt="Gazebo simulation"/>
</td>
</tr>
</table>

---

# From ROS to Real Motion

> **The interesting moment is when software stops describing motion and starts causing it.**

```mermaid
%%{init: {"theme":"base","themeVariables":{"background":"#0d1117","primaryTextColor":"#ffffff","lineColor":"#64748b"}}}%%
flowchart LR
    CMD["Joint Trajectory<br/>Command"] --> MAN["controller_manager"] --> HW["ArduinobotInterface<br/>write()"] --> CONV["Radians → Degrees<br/>Service"] --> TOPIC["/arduino/arm_actuate"] --> SERIAL["rosserial"] --> MCU["Arduino"] --> SERVO["Servo Motion"]

    classDef input fill:#0f172a,stroke:#38bdf8,color:#fff,stroke-width:2px;
    classDef ros fill:#111827,stroke:#a78bfa,color:#fff,stroke-width:2px;
    classDef bridge fill:#111827,stroke:#f59e0b,color:#fff,stroke-width:2px;
    classDef hw fill:#0f172a,stroke:#34d399,color:#fff,stroke-width:2px;

    class CMD input;
    class MAN,HW ros;
    class CONV,TOPIC,SERIAL bridge;
    class MCU,SERVO hw;
```

<p align="center">
  <img src="assets/gallery/hardware-link.png" width="82%" alt="Software to hardware communication path"/>
</p>

---

# Control Core

## Hardware interface loop

```cpp
void ArduinobotInterface::update(const ros::TimerEvent& e)
{
    elapsed_time_ = ros::Duration(e.current_real - e.last_real);
    read();
    controller_manager_->update(ros::Time::now(), elapsed_time_);
    write(elapsed_time_);
}
```

```text
READ STATE  ──►  UPDATE CONTROLLERS  ──►  WRITE COMMANDS
```

## Angle conversion

```cpp
bool convert_radians_to_degrees(
    arduinobot_controller::AnglesConverter::Request  &req,
    arduinobot_controller::AnglesConverter::Response &res)
{
    res.base     = static_cast<int>(((req.base + (M_PI/2)) * 180) / M_PI);
    res.shoulder = 180 - static_cast<int>(((req.shoulder + (M_PI/2)) * 180) / M_PI);
    res.elbow    = static_cast<int>(((req.elbow + (M_PI/2)) * 180) / M_PI);
    res.gripper  = static_cast<int>(((-req.gripper) * 180) / (M_PI/2));
    return true;
}
```

## Simulation / hardware split

```xml
<group unless="$(arg is_sim)">
    <node pkg="arduinobot_controller"
          type="angles_converter_cpp"
          name="angles_converter"
          output="screen"/>

    <node pkg="arduinobot_controller"
          type="arduinobot_interface_node"
          name="arduinobot_interface_node"
          ns="/arduinobot"
          output="screen"/>
</group>
```

---

# Repository Anatomy

```text
frame2motion/
│
├── assets/
│   ├── hero/
│   ├── kinematics/
│   ├── diagrams/
│   └── gallery/
│
├── arduinobot_description/
│   ├── launch/
│   ├── mesh/
│   ├── rviz/
│   └── urdf/
│
├── arduinobot_controller/
│   ├── config/
│   ├── include/
│   ├── launch/
│   ├── src/
│   └── srv/
│
├── arduinobot_test/
│
└── README.md
```

---

# Engineering Map

<table>
<tr>
<td width="50%" valign="top">
<h3>🧭 Motion Mathematics</h3>
Reference frames · 2D/3D transforms · Rotation matrices · Homogeneous transforms · Frame chaining
</td>
<td width="50%" valign="top">
<h3>🤖 Robot Modeling</h3>
URDF links/joints · Xacro macros · Visual/collision geometry · Joint limits · Inertial data
</td>
</tr>
<tr>
<td width="50%" valign="top">
<h3>⚙️ ROS Control</h3>
RobotHW · JointStateInterface · PositionJointInterface · controller_manager · Timer-driven update loop
</td>
<td width="50%" valign="top">
<h3>🔌 Hardware Bridge</h3>
Radian-to-degree mapping · Custom service · rosserial · Arduino actuation · Servo command path
</td>
</tr>
</table>

---

# Technology Strip

<p align="center">
  <img src="https://img.shields.io/badge/ROS%20Noetic-111827?style=for-the-badge&logo=ros&logoColor=white" />
  <img src="https://img.shields.io/badge/C++-111827?style=for-the-badge&logo=cplusplus&logoColor=white" />
  <img src="https://img.shields.io/badge/URDF%20%2F%20Xacro-111827?style=for-the-badge" />
  <img src="https://img.shields.io/badge/RViz-111827?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Gazebo-111827?style=for-the-badge" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/ros__control-111827?style=for-the-badge" />
  <img src="https://img.shields.io/badge/controller__manager-111827?style=for-the-badge" />
  <img src="https://img.shields.io/badge/hardware__interface-111827?style=for-the-badge" />
  <img src="https://img.shields.io/badge/rosserial-111827?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Arduino-111827?style=for-the-badge&logo=arduino&logoColor=white" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Linux-111827?style=for-the-badge&logo=linux&logoColor=white" />
  <img src="https://img.shields.io/badge/Docker-111827?style=for-the-badge&logo=docker&logoColor=white" />
  <img src="https://img.shields.io/badge/YAML-111827?style=for-the-badge&logo=yaml&logoColor=white" />
</p>

---

# Visual Gallery

<table>
<tr>
<td width="50%" align="center">
<img src="assets/gallery/rviz-view.png" width="100%" alt="RViz View"/><br>
<b>RViz · Robot State & TF</b>
</td>
<td width="50%" align="center">
<img src="assets/gallery/gazebo-view.png" width="100%" alt="Gazebo View"/><br>
<b>Gazebo · Physics Simulation</b>
</td>
</tr>
<tr>
<td width="50%" align="center">
<img src="assets/gallery/urdf-structure.png" width="100%" alt="URDF Structure"/><br>
<b>URDF · Structural Model</b>
</td>
<td width="50%" align="center">
<img src="assets/gallery/hardware-link.png" width="100%" alt="Hardware Link"/><br>
<b>ROS ↔ Hardware Bridge</b>
</td>
</tr>
</table>

---

<div align="center">

## frame2motion

**Frames define motion. Software organizes it. Hardware makes it real.**

<br>

**Mohamed Reda**  
Embedded Systems & Control Engineering Student

`Robotics` · `Embedded Software` · `ROS` · `Simulation` · `Control` · `Hardware Integration`

</div>
