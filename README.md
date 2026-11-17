<div align="center">

# DeepWheat Drones

**Precision Agriculture Multi-Agent System**

A fleet of autonomous drones that looks after wheat fields. Scout drones photograph every plant and spot disease with the DeepWheat image model, payload drones spray fungicide or fertiliser where it's needed, and SPADE agents decide which drone does what based on battery, wind, weather and the crop calendar.

![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=for-the-badge&logo=python&logoColor=white)
![SPADE](https://img.shields.io/badge/SPADE-4.0-5C2D91?style=for-the-badge&logoColor=white)
![ROS 2](https://img.shields.io/badge/ROS_2-22314E?style=for-the-badge&logo=ros&logoColor=white)
![PX4](https://img.shields.io/badge/PX4-0C1E3C?style=for-the-badge&logoColor=white)
![TensorFlow](https://img.shields.io/badge/TensorFlow-FF6F00?style=for-the-badge&logo=tensorflow&logoColor=white)

[Objectives](#objectives) · [Features](#features) · [Architecture](#system-architecture) · [Agents](#agent-specifications) · [Setup](#environment-setup) · [Behaviour](#system-behavior) · [Team](#team)

<sub>Challenge 4 · Master's in Artificial Intelligence Engineering · <a href="https://www.isep.ipp.pt">ISEP</a>, Porto · 2024–25</sub>

</div>


## Objectives

The challenge followed on from [DeepWheat](https://github.com/Marco-Nobrega-Andrade/deepwheat): take its disease-detection model off the web page and put it on drones that work a field on their own. These were the goals:

- [x] Coordinate a mixed drone fleet with a multi-agent system, using the Contract Net Protocol to hand out tasks
- [x] Let field agents decide when a field needs monitoring, fertiliser or treatment, using live weather and the growth season
- [x] Fly every drone as a simulated PX4 quadcopter through ROS 2, with a camera and gimbal on each one
- [x] Detect disease from the drones' own camera images with the CNN + Vision Transformer model from DeepWheat
- [x] Plan short routes through every plant, and manage battery, payload and a shared charging pad



## Features

| | Feature | What it does |
|:-:| --- | --- |
| 🤝 | **Multi-agent coordination** | Contract Net Protocol for task allocation, drones competing on battery level and wind, parallel field operations with request queuing, and real-time proposal evaluation |
| 🌾 | **Smart agriculture operations** | Autonomous field monitoring with disease detection, weekly preventive monitoring, fertilisation during the growth season, and targeted fungicide for diseased plants, all weather-aware through the OpenWeatherMap API |
| 🔋 | **Resource management** | Battery and payload tracking, a single charging pad that drones queue for (lowest battery first), wind assessment for flight planning, seasonal behaviour, and load balancing across the fleet |
| 🚁 | **Drone simulation** | Each drone is a PX4 x500 quadcopter with a gimbal camera in Gazebo, flown in offboard mode through ROS 2, at its own altitude per task so drones never collide or land on each other |
| 🦠 | **Disease detection** | Scout drones point the camera down at each plant and classify the image with the DeepWheat CNN + ViT model |
| 🗺️ | **Route planning** | Google OR-Tools solves the shortest path through every plant (a travelling-salesman problem); fertilising and fungicide runs get their own paths |

---

## System Architecture

### Agent Hierarchy
```
CentralAgent (Coordinator)
├── Contract Net Protocol Manager
├── Request Queue Management
├── Charging Pad Queue (lowest battery first)
└── Drone Fleet Coordination
    ├── PayloadDrone Fleet (2 drones)
    │   ├── Fertilization Operations
    │   ├── Fungicide Application
    │   └── Battery and Payload Management
    ├── VigilantDrone Fleet (2 drones)
    │   ├── Field Monitoring
    │   ├── Disease Detection (CNN + ViT)
    │   └── Plant Scanning
    └── FieldAgent Network (3 fields)
        ├── Weather Monitoring
        ├── Crop Status Tracking
        └── Request Generation
```

### From agents to drones
```
SPADE agents (XMPP, Prosody)  →  OffboardControl  →  ROS 2 topics  →  PX4 drones in Gazebo
                                       ↑
              camera images  →  CNN + ViT disease model
```

Each drone agent owns an `OffboardControl` object. It arms the drone, switches it to offboard mode and streams position setpoints to `/px4_<id>/fmu/in/trajectory_setpoint`. It reads the drone's position from `/px4_<id>/fmu/out/vehicle_local_position` and its camera from the Gazebo image topic of `x500_gimbal_<id>`, and points the gimbal down while scanning. The SPADE agents and the ROS 2 node run side by side, so a drone can fly while its agent keeps talking to the others.

### Communication Flow
1. **Field Agents** monitor weather and crop conditions
2. **Weekly monitoring requests** are sent to keep the crop healthy
3. **Fertilization requests** are sent during the growth season
4. **CentralAgent** broadcasts a CFP (Call for Proposals)
5. **All available drones** submit competitive proposals
6. **The best proposal wins**, based on battery and wind scoring
7. **Parallel operations** with automatic request queuing; requests nobody can take go back to the end of the queue
8. **Disease alerts** trigger targeted treatment requests
9. **Low battery or empty payload** sends a drone to the charging pad, where the CentralAgent lets one drone in at a time

---

## Project Structure
```
deepWheat/
├── agents/
│   ├── central_agent.py         # Contract Net Protocol coordinator and charging queue
│   ├── field_agent.py           # Field monitoring and requests
│   ├── payload_drone.py         # Fertilization/fungicide operations
│   └── vigilant_drone.py        # Disease detection and monitoring
├── offboard/
│   ├── offboard_control.py      # Flies a drone: takeoff, paths, scanning, charging
│   ├── offboard_ros_messenger.py# ROS 2 publishers and subscribers for one PX4 drone
│   ├── offboard_signal.py       # Keeps the offboard heartbeat going
│   ├── routing_service.py       # Shortest path through the plants (OR-Tools)
│   ├── disease_classification_service.py  # Runs the CNN + ViT model on camera images
│   ├── diseases.py              # Disease classes
│   └── cnn_vit_saved_model/     # Trained model from DeepWheat
├── utils/
│   ├── battery.py               # Battery usage calculations
│   ├── field_map.py             # Shared field state, loaded from CSV
│   ├── logger.py                # Color-coded logging system
│   ├── negotiation.py           # Proposal evaluation logic
│   ├── season.py                # Growth season detection
│   └── weather.py               # OpenWeatherMap integration
├── data/
│   └── field_map.csv            # Plant positions for each field
├── scripts/
│   └── setup-start.sh           # Creates the XMPP accounts for every agent
├── ws_sensor_combined_deepwheat/  # ROS 2 workspace with the offboard_control package
├── config.py                    # System configuration
└── main.py                      # Simulation launcher (agents + ROS 2 node)
requirements.txt                 # Python dependencies
```

---

## Agent Specifications

| Agent | Role | Protocol and features | Fleet |
| --- | --- | --- | --- |
| **CentralAgent** | System coordinator: task assignment, drone coordination, request queuing | Contract Net Protocol with FSM-based state management, dynamic proposal evaluation, parallel request processing, and the queue for the charging pad | Manages 2 payload drones and 2 vigilant drones |
| **FieldAgent** | Field management: weather monitoring, crop status, fertilization requests, weekly monitoring | Uses the OpenWeatherMap API and the growth season calendar; daily initialization, weekly preventive monitoring, treatment tracking and completion monitoring | One agent per field, 3 fields with 28 plant positions each |
| **PayloadDroneAgent** | Operations: fertilization, fungicide application, battery and payload management | Proposal generation, task execution, automatic recharging and payload refills; full-field fertilization or targeted plant treatment | 2 drones competing for tasks |
| **VigilantDroneAgent** | Monitoring and detection: field scanning, disease detection, alert generation | Plant-by-plant scanning with position tracking, camera images classified by the CNN + ViT model, disease reporting, battery management | 2 drones for field surveillance |

---

## Environment Setup

### System Requirements
- Ubuntu 22.04 LTS (recommended)
- Python 3.10
- Prosody XMPP Server
- OpenWeatherMap API key
- ROS 2 with `px4_msgs` and `cv_bridge`
- PX4 SITL with Gazebo, running four `x500_gimbal` drones (namespaces `px4_1` to `px4_4`) and the Micro XRCE-DDS agent

### 1. Install System Dependencies
```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv prosody -y
```

### 2. Setup Virtual Environment
```bash
python3 -m venv spade_env
source spade_env/bin/activate
```

### 3. Install Python Dependencies
```bash
pip install -r requirements.txt
pip install ortools
```
`requirements.txt` covers SPADE 4.0.3, aioxmpp, slixmpp, TensorFlow, Keras, Pillow, NumPy, requests and termcolor. OR-Tools is used by the route planner but isn't listed there yet.

### 4. Configure Prosody XMPP Server
```bash
# Start Prosody service
sudo systemctl start prosody
sudo systemctl enable prosody

# Create agent accounts (or run deepWheat/scripts/setup-start.sh)
sudo prosodyctl adduser central@localhost
sudo prosodyctl adduser field1@localhost
sudo prosodyctl adduser field2@localhost
sudo prosodyctl adduser field3@localhost
sudo prosodyctl adduser payload1@localhost
sudo prosodyctl adduser payload2@localhost
sudo prosodyctl adduser vigilant1@localhost
sudo prosodyctl adduser vigilant2@localhost
```
The script creates all eight accounts with the password `admin1234`, which is what `main.py` uses.

### 5. Configure Weather API
Update `config.py` with your OpenWeatherMap API key:
```python
WEATHER_API_KEY = "your_api_key_here"
WEATHER_LAT = 41.1496  # Your latitude
WEATHER_LON = -8.6109  # Your longitude
WEATHER_UPDATE_INTERVAL = 600  # seconds between weather updates
```

---

## Running the Simulation

### Start the System
Start PX4 and Gazebo with the four drones first, then:
```bash
cd deepWheat
python main.py
```

### Expected Output
A sample from the first, SPADE-only version of the system:
```
🚁 Managing 2 vigilant drones and 3 payload drones
🔍 First monitoring request for field_1
📤 Weekly monitoring request sent for field_1
📤 Sending CFP for fertilization_request: field_1|3.09 to 3 payload drones
📤 Sending CFP for monitoring_request: field_1|3.09 to 2 vigilant drones
📊 Proposal from payload1: Battery=100.0%, Wind=10.58km/h, Score=0.65
📊 Proposal from vigilant1: Battery=100.0%, Wind=12.44km/h, Score=0.59
🏆 Best proposal: payload3 (score: 0.70)
✅ Assigned fertilization_request to payload3
🦠 Disease (white_stripe) detected at field_1 (0, 1)
🎉 Treatment COMPLETE @ field_1 (0,1) - plant is healthy!
```

### Stop the Simulation
Press **Q** and hit **Enter** for graceful shutdown.

---

## Configuration Options

### Battery Management
```python
BATTERY_LOW_THRESHOLD = 45       # Recharge below 45%
BATTERY_CRITICAL_THRESHOLD = 25  # Critical battery level
BATTERY_RECHARGE_STEP = 20       # Recharge 20% per interval
RECHARGE_INTERVAL = 1            # 1 second per recharge step
```
In flight, each `OffboardControl` also tracks its own battery and payload: scout drones use less battery than payload drones, a drone heads for the charging pad below 30%, and both battery and payload refill there in steps of 20%.

### Seasonal Behavior
```python
GROWTH_SEASON_START = (3, 15)   # March 15
GROWTH_SEASON_END = (7, 5)      # July 5
DEFAULT_CROP_TYPE = "spring"    # Spring wheat
```

### Environmental Conditions
```python
WIND_MIN = 5                    # Minimum wind speed (km/h)
WIND_MAX = 15                   # Maximum wind speed (km/h)
FLIGHT_TIME = 2                 # Flight time between positions (seconds)
APPLICATION_TIME = 2            # Application time per plant (seconds)
PROPOSAL_TIMEOUT = 0.5          # Seconds to wait for proposals
RECEIVE_TIMEOUT = 1             # Seconds to wait for a message
```

### Field Configuration
Each field's plants are listed in `data/field_map.csv` as `field_id, x, y, z`, and the field agents and drones build their maps and waypoints from it:
```
field_1,1.59,-4.5,-1.3
field_1,1.59,-3,-1.3
field_1,1.59,-1.5,-1.3
```
`config.py` still keeps a grid size and the list of field agents:
```python
FIELD_ROWS = 2                  # Grid rows per field
FIELD_COLS = 5                  # Grid columns per field

FIELD_AGENTS = [
    {"agent_jid": "field1@localhost", "field_id": "field_1"},
    {"agent_jid": "field2@localhost", "field_id": "field_2"},
    {"agent_jid": "field3@localhost", "field_id": "field_3"}
]
```
The first version placed diseases by hand with a `DISEASED_PLANTS` setting (for example white stripe and brown rust on field 1, yellow rust and septoria on field 2). Diseases now come from what the scout drones actually see.

### Drone Positions
Each drone has its own parking spot, offset from the charging pad, so they never take off or land on top of each other:
```python
OFF_SET_DRONES = [
    {"drone_id": "1", "offset": [-3.0, 7.0]},
    {"drone_id": "2", "offset": [-3.0, -7.0]},
    {"drone_id": "3", "offset": [-6.0, 7.0]},
    {"drone_id": "4", "offset": [-6.0, -7.0]},
]
```

---

## System Behavior

### Weekly Monitoring System
- **First Run**: Always requests monitoring for immediate crop assessment
- **Subsequent Runs**: Automatically requests monitoring every 7 days
- **Preventive Care**: Ensures regular field inspection regardless of detected conditions
- **Simple Logic**: Uses date comparison to track the last monitoring request

### Contract Net Protocol Flow
1. **Field Agent** detects a need (monitoring, fertilization or treatment)
2. **Central Agent** broadcasts a CFP to the matching drones
3. **Available drones** submit proposals with battery/wind data
4. **Proposals are evaluated** using the scoring algorithm: `battery_score - wind_penalty`
5. **The best drone is selected** and assigned the task
6. **Other drones are rejected** and remain available for future tasks

### Proposal Evaluation Algorithm
```python
def evaluate_proposal(battery_level, wind_speed):
    battery_score = battery_level / 100.0  # Normalize to 0-1
    wind_penalty = wind_speed / 30.0       # Assume max wind 30km/h
    return battery_score - wind_penalty    # Higher score = better drone
```

### Disease Detection Workflow
1. **Vigilant Drone** takes off, points its camera down and flies the shortest path through every plant (weekly + on-demand)
2. **At each plant**, the camera image is classified by the CNN + ViT model
3. **Disease detected** at specific plant coordinates
4. **Field Agent** receives a disease alert with the position
5. **Treatment request** sent if the plant is not already being treated
6. **Payload Drone** flies to that plant and applies fungicide
7. **Treatment completion** is reported back to the Field Agent

The model knows brown rust, yellow rust, septoria, mildew and healthy leaves. In the final version, septoria and mildew results are counted as healthy, so the drones act on brown rust and yellow rust.

### Charging Pad
There is one charging pad for the whole fleet. When a drone runs low on battery, or a payload drone runs out of fertiliser or fungicide, it tells the CentralAgent. The CentralAgent lets one drone land at a time, queues the rest with the lowest battery first, and calls the next one in when the pad is free. A drone that was interrupted picks its task back up after charging.

---

## Monitoring & Logging

### Color-Coded Logging
- **🔵 Central Agent**: Blue - Coordination and task assignment
- **🟢 Vigilant Drones**: Green - Monitoring and detection
- **🟡 Payload Drones**: Yellow - Operations, battery and payload status
- **🔴 Field Agents**: Red - Field conditions and requests

Each run also writes the logs to a `logs/` folder, cleared when the simulation starts.

### Key Metrics Tracked
- Drone battery levels and recharge cycles
- Payload levels and refills
- Wind conditions and flight efficiency
- Weekly monitoring schedule compliance
- Task completion rates and timing
- Proposal scores and selection rationale
- Disease detection and treatment effectiveness

---

## Extensibility

### Adding More Drones
Update the range in `main.py`, add a PX4 drone with the matching id in the simulation, and give it an offset in `OFF_SET_DRONES`:
```python
payload_drones = [
    PayloadDroneAgent(f"payload{i}@localhost", "admin1234", ros_thread.node, i + NUM_NODES_VIGILANT)
    for i in range(1, 4)  # Creates payload1 through payload3
]
```

### Adding More Fields
Add the field's plant positions to `data/field_map.csv` and its agent to `config.py`:
```python
FIELD_AGENTS = [
    {"agent_jid": "field1@localhost", "field_id": "field_1"},
    {"agent_jid": "field2@localhost", "field_id": "field_2"},
    {"agent_jid": "field3@localhost", "field_id": "field_3"},
    {"agent_jid": "field4@localhost", "field_id": "field_4"}  # New field
]
```

### Custom Disease Types
The disease classes live in `offboard/diseases.py` and come from the trained model. To add a disease, retrain the model with images of it and add the new class there and in `disease_classification_service.py`.

---

## Future Enhancements

### Planned Features
- [x] **ROS2 Integration** for physical drone simulation
- [x] **Gazebo 3D Environment** for realistic field visualization
- [x] **Machine Learning** for disease detection (CNN + ViT from DeepWheat)
- [ ] **Machine Learning** for disease prediction
- [ ] **IoT Sensor Integration** for real-time field data
- [ ] **Mobile Dashboard** for monitoring and control

### Research Applications
- Multi-agent coordination algorithms
- Precision agriculture optimization
- Autonomous vehicle fleet management
- Environmental monitoring systems
- Supply chain and logistics optimization

---

## License

This project is licensed under the GNU General Public License v3.0 (see [LICENSE](LICENSE)). It was developed for research and educational purposes in precision agriculture and multi-agent systems.

## Contributing

Contributions are welcome! Please feel free to submit issues, feature requests, or pull requests to improve the DeepWheat Drones system.

## Team

<table align="center">
  <tr>
    <td align="center" width="150">
      <a href="https://github.com/Marco-Nobrega-Andrade"><img src="https://github.com/Marco-Nobrega-Andrade.png?size=200" width="100" alt="Marco Andrade"/></a><br>
      <b>Marco Andrade</b><br>
      <a href="https://github.com/Marco-Nobrega-Andrade">@Marco-Nobrega-Andrade</a>
    </td>
    <td align="center" width="150">
      <a href="https://github.com/SrMasterAD"><img src="https://github.com/SrMasterAD.png?size=200" width="100" alt="David Dias"/></a><br>
      <b>David Dias</b><br>
      <a href="https://github.com/SrMasterAD">@SrMasterAD</a>
    </td>
    <td align="center" width="150">
      <a href="https://github.com/nelsonnovais"><img src="https://github.com/nelsonnovais.png?size=200" width="100" alt="Nelson Novais"/></a><br>
      <b>Nelson Novais</b><br>
      <a href="https://github.com/nelsonnovais">@nelsonnovais</a>
    </td>
    <td align="center" width="150">
      <a href="https://github.com/ricardoham"><img src="https://github.com/ricardoham.png?size=200" width="100" alt="Ricardo Manoel"/></a><br>
      <b>Ricardo Manoel</b><br>
      <a href="https://github.com/ricardoham">@ricardoham</a>
    </td>
    <td align="center" width="150">
      <a href="https://github.com/sabrina-003"><img src="https://github.com/sabrina-003.png?size=200" width="100" alt="Sabrina"/></a><br>
      <b>Sabrina</b><br>
      <a href="https://github.com/sabrina-003">@sabrina-003</a>
    </td>
    <td align="center" width="150">
      <a href="https://github.com/binaps"><img src="https://github.com/binaps.png?size=200" width="100" alt="binaps"/></a><br>
      <b>binaps</b><br>
      <a href="https://github.com/binaps">@binaps</a>
    </td>
  </tr>
</table>

---

<div align="center">

**DeepWheat Drones** · *Autonomous Multi-Agent Precision Agriculture* 🌾🚁

</div>
