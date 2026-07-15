# SMUL GPSE — Community Library & Developer SDK Documentation

Welcome to the **SMUL General Purpose Simulation Engine (GPSE)** Community Library. This document serves as the master reference guide for prospective developers, researchers, and system engineers looking to extend the browser simulation IDE, build custom physics plugin containers, train RL policies, or interface directly with our state channels.

---

## 1. System Architecture & Topology

SMUL GPSE leverages a hybrid serverless model. Ephemeral APIs and pipeline orchestration run on short-lived micro-functions, while simulation loops run inside stateless container instances that scale to zero when idle.

### Infrastructure Blueprint
![AWS Serverless Blueprint](../assets/system_design/aws_serverless_blueprint.svg)

### Data Flow & Orchestration Topology
The platform follows a strict Input/Output (I/O) path linking user interactions, control validation structures, and active container simulation loops:

![Codebase I/O Flow Mapping](../assets/system_design/architectural_overview.svg)

### Data Plane vs. Control Plane
- **Control Plane**: Manages authorization (`auth.py`), user workspace persistence (`projects.py`), LLM code generation (`llm_parse.py`), and Fargate Task launching (`simulation_start.py`). Connects primarily to **Amazon DynamoDB** and **SQS FIFO queues**. Input code configurations undergo Abstract Syntax Tree (AST) validation and correctness/complexity audits (via the decoupled **In-App Harness**) before execution starts.
- **Data Plane**: Runs the continuous physics engines and training policies inside **AWS Fargate** containers. Communicates output state frames at ~60Hz via **ElastiCache Redis** pub/sub channels out through **API Gateway WebSocket** tunnels.

---

## 2. Integrated Development Environment (IDE) Layout

The browser-based interface is a single-page React workspace powered by WebGL (Three.js), Monaco, and local machine learning models (MediaPipe).

![IDE Layout Mockup](../assets/system_design/ide_layout_mockup.svg)

### Panel Directory
- **File Explorer**: Renders user workspaces mapped directly to project structures in DynamoDB.
- **3D Viewport**: Coordinates real-time canvas updates. Also overlays MediaPipe joint vectors to capture and classify human gestures.
- **Monaco Code Editor**: Python-centric editor exposing API hooks and supporting hot-reloads directly into the live container environment.
- **LLM Assistant**: A natural language compiler translating user prompts (e.g., *"double pendulum with damping"*) into code configurations, which are parsed and verified using an Abstract Syntax Tree (AST) validator before deployment.

---

## 3. WebSocket Micro-Data Protocol

Real-time operations are carried out using low-latency JSON data frames routed via AWS API Gateway WebSockets.

![WebSocket Frame Inspector](../assets/system_design/websocket_frame_inspector.svg)

### Telemetry Broadcast (`sim:state`)
Published by the running container every step ($dt \approx 16.6\text{ms}$):
```json
{
  "action": "sim:state",
  "simulationId": "uuid-string",
  "payload": {
    "time": 24.5826,
    "objects": [
      {
        "id": "bob_1",
        "position": [0.0, -1.0, 0.0],
        "rotation": [0.0, 0.0, 0.0, 1.0]
      }
    ],
    "energy": {
      "kinetic": 12.45,
      "potential": -4.12,
      "total": 8.33
    }
  }
}
```

### Interactive Gestures (`sim:command`)
Sent from the client browser when pinch and grab interactions are captured:
```json
{
  "action": "sim:command",
  "simulationId": "uuid-string",
  "payload": {
    "commandType": "APPLY_FORCE",
    "targetObjectId": "bob_1",
    "vector": [-15.2, 8.4, 0.0],
    "magnitude": 17.36
  }
}
```

---

## 4. Custom Plugin Development (Container SDK)

The SMUL Plugin SDK allows you to wrap arbitrary physics engines, fluid solvers, or game loops in a Docker container and plug them into the GPSE lifecycle.

![Plugin SDK Hooks](../assets/system_design/plugin_sdk_hooks.svg)

### Developer Walkthrough: Writing a Custom Physics Plugin

#### 1. Implement the Python SDK Class
Install `smul-gpse-sdk` and implement the `PluginEngine` abstract base class. You must define `on_init` and `on_step`:

```python
# vortex_plugin.py
import numpy as np
from smul_gpse_sdk import PluginEngine

class VortexPhysicsPlugin(PluginEngine):
    def on_init(self, config):
        """
        Called when the Fargate task boots.
        'config' is parsed from SQS startup configuration.
        """
        self.viscosity = float(config.get("viscosity", 0.1))
        self.density_grid = np.zeros((64, 64))
        self.objects_registry = []

    def on_step(self, dt):
        """
        Called 60 times a second.
        Perform vector math and return the updated frame properties.
        """
        # (Navier-Stokes fluid integration calculations here)
        self.density_grid = self.solve_fluid_dynamics(dt, self.viscosity)
        
        # Return state block matching the WebSocket Schema
        return {
            "objects": self.objects_registry,
            "custom_telemetry": {
                "max_vorticity": float(np.max(self.density_grid)),
                "grid_data": self.density_grid.tolist()
            }
        }

if __name__ == "__main__":
    # Boots the asyncio runner and establishes SQS/Redis loops
    plugin = VortexPhysicsPlugin()
    plugin.start_loop()
```

#### 2. Package inside a Container Image
Your plugin must be containerized and run as a non-root user for security hardening:
```dockerfile
FROM python:3.11-slim

RUN groupadd -g 1000 smuluser && \
    useradd -r -u 1000 -g smuluser smuluser

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY vortex_plugin.py .
USER smuluser

ENTRYPOINT ["python", "vortex_plugin.py"]
```

---

## 5. Machine Learning & Agent Training

Researchers can train policies inside Fargate Spot containers. Metric streams are cached and displayed on the real-time training panel.

![Agent Training Dashboard](../assets/system_design/agent_training_dashboard.svg)

### Model Checkpointing & Monitoring
- **Huber Loss Curves**: Monitored at 10-episode averages. If loss diverges, alerts are dispatched to the frontend logs.
- **Model Checkpoints**: Uploaded asynchronously to Amazon S3 every 100 episodes:
  `s3://smul-gpse-models/{simulationId}/cartpole_dqn_ep100.pt`

---

## 6. Local Workspace Setup & Emulators

To develop and test GPSE components locally without incurring cloud costs, use **LocalStack** and **Serverless Offline**.

### 1. Prerequisites
- Docker & Docker Compose
- Node.js >= 18
- Python >= 3.11

### 2. Configure Local Services
Create a `docker-compose.local.yml` to spin up local emulators:
```yaml
version: '3.8'

services:
  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      - SERVICES=dynamodb,s3,sqs
      - DEFAULT_REGION=us-east-1

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
```

### 3. Initialize Databases
```bash
# Create local DynamoDB Table
aws dynamodb create-table \
    --table-name smul-simulations \
    --attribute-definitions AttributeName=simulationId,AttributeType=S \
    --key-schema AttributeName=simulationId,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --endpoint-url http://localhost:4566

# Create local SQS Queue
aws sqs create-queue \
    --queue-name sim-queue.fifo \
    --attributes FifoQueue=true \
    --endpoint-url http://localhost:4566
```
