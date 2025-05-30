# CoppeliaSim MCP Server

A Python-based MCP (Motion Control Protocol) server implementation for CoppeliaSim robotics simulation, supporting both FastAPI and FastMCP backends.

## Features
- JSON-RPC 2.0 protocol
- Joint control and robot/scene description tools
- Supports both HTTP/SSE and stdio transports
- Flexible, LLM-friendly output

## Requirements
- Python 3.x
- CoppeliaSim (formerly V-REP) **version 4.3 or newer** (ZeroMQ remote API required)
- Required Python packages (see requirements.txt)

## Installation

### 0. (Recommended) Create a Conda Environment

It is highly recommended to use a conda environment to avoid dependency conflicts:

```bash
conda create -n coppelia-mcp python=3.10
conda activate coppelia-mcp
```

You can then proceed with the steps below.

### 1. Clone the repository and install requirements

```bash
git clone https://github.com/alsak0de/CoppeliaSlim-mcpserver.git
cd CoppeliaSlim-mcpserver
pip install -r requirements.txt
```

## Usage

### 1. Start CoppeliaSim and load your robot model

### 2. Run the MCP server

NOTE: MCP Servers typically run using stdio transport assuming the tools provider runs in the same machine as the MCP server and, very often, also the MCP host and client.
Our project implements SSE transport and allows a complete distributed setup where CoppeliaSim, MCP Server and MCP Client can be running in different machines.


#### Which server should I use?
- **Option A (FastAPI-based, `coppelia_mcp.py`)** is recommended for most production deployments, especially if you want standard web server features, integration with web tooling, or custom endpoints.
- **Option B (FastMCP-based, `coppelia_fastmcp.py`)** is ideal for agent/LLM-native workflows, stdio transport, or pure MCP/agent integration.
- **Option C (Direct script, `coppelia_mcp.py`)** is for development, debugging, or when you need to specify `--coppeliaHost` via CLI.

#### Option A: FastAPI-based server (`coppelia_mcp.py`) [Recommended for production]
```bash
COPPELIASIM_HOST=<coppelia_host> uvicorn coppelia_mcp:app --host 0.0.0.0 --port 8000
```
- This is the standard way to run FastAPI apps in production.
- Set the CoppeliaSim host using the `COPPELIASIM_HOST` environment variable.
- **Precedence:**
  1. `COPPELIASIM_HOST` environment variable (if set)
  2. Default: `127.0.0.1`
- **Transports:**
  - HTTP POST (JSON-RPC)
  - SSE at `/sse` endpoint
  - Stdio (via npx mcp-remote bridge, see below)

#### Option B: FastMCP-based server (`coppelia_fastmcp.py`)
- **Default:**
  ```bash
  python coppelia_fastmcp.py --host 0.0.0.0 --port 8000 --coppeliaHost <coppelia_host>
  ```
  - You can specify the CoppeliaSim host with `--coppeliaHost` (default: 127.0.0.1).
- **Transports:**
  - HTTP POST (JSON-RPC)
  - SSE at `/sse` endpoint
  - Stdio (native, or via npx mcp-remote if needed)

#### Option C: Direct script (development/debug, `coppelia_mcp.py`)
```bash
python coppelia_mcp.py --host 0.0.0.0 --port 8000 --coppeliaHost <coppelia_host>
```
- Use this method for development, debugging, or when you need to specify `--coppeliaHost` via CLI.
- Not recommended for production use; prefer Option A for deployment.

### 3. Connecting Clients
- **SSE/HTTP:**
  - Use the `/sse` endpoint for SSE clients (recommended for modern LLM/agent clients)
  - Example: `http://localhost:8000/sse`
- **Stdio:**
  - Use a bridge if your client only supports stdio:
    ```bash
    npx mcp-remote http://localhost:8000
    ```
  - Or configure your client to use stdio transport if supported natively by FastMCP

### 4. Auto-Approve Tool Calls
- In most clients (e.g., Cursor), enable "Auto Approve" in the server settings to avoid confirmation prompts for each tool call.

## API Tools
- `rotate_joint`: Rotates a joint to a given angle
- `list_joints`: Lists all joints with their types and limits
- `describe_robot`: Returns a detailed, LLM-friendly description of all robot elements
- `describe_scene`: Returns a description of all scene objects (excluding robot joints)

**Note:** All tool logic is now centralized in `tools.py`. To add a new tool, define your function in `tools.py` (taking `sim` as the first argument), then register it in both `coppelia_mcp.py` and `coppelia_fastmcp.py` as needed. This ensures both servers share the same tool logic and remain consistent.

## Notes
- Both servers default to `0.0.0.0:8000` but you can override with `--host` and `--port`.
- Use the SSE endpoint for best compatibility with modern LLM/agent clients.
- For stdio-only clients, use the npx bridge or FastMCP's native stdio support.
- **Why/When uvicorn?**
  - For the FastAPI-based server (`coppelia_mcp.py`), you need `uvicorn` (or another ASGI server) to actually serve HTTP/SSE endpoints, because FastAPI is just a framework and does not include a web server.
  - For the FastMCP-based server (`coppelia_fastmcp.py`), you only need `uvicorn` if you want to serve HTTP/SSE endpoints. If you run FastMCP in stdio mode (for agent/CLI integration), you do **not** need `uvicorn` or any web server, since all communication is over stdin/stdout.

## Author and Credits

Created and maintained by **Albert Sagarra**.  
Community contributions are welcome and encouraged. Please refer to [`CONTRIBUTING.md`](./CONTRIBUTING.md) for details on how to participate.

## License

This project is licensed under the [MIT License](./LICENSE).  
You are free to use, modify, and distribute this project under the terms of that license.

## Running with Docker

This is the recommended way to ensure a clean, isolated environment for deployment or testing.

### 1. Build the Docker Image

```bash
docker build -t coppelia-mcp .
```

### 2. Run the MCP Server Container

**Important:**
- You **must** specify the CoppeliaSim host using the `--coppeliaHost` argument so the MCP server can connect to the CoppeliaSim instance running outside the container.
- The server always binds to `0.0.0.0` inside the container.
- The port can be set by passing it as the first argument to `docker run` (default: 8000). You **must** expose the server port to access the API from outside the container.
- The `--rm` flag is used so the container is automatically removed when stopped (no persistence is required).

#### Example (replace `<coppelia_host>` with your CoppeliaSim host IP):

```bash
docker run --rm -p 8000:8000 coppelia-mcp --coppeliaHost <coppelia_host>
```

- To use a different port, pass it as the first argument and map the port:

```bash
docker run --rm -p 9000:9000 coppelia-mcp 9000 --coppeliaHost <coppelia_host>
```

- Any additional arguments after the port are passed directly to the server script.
- If CoppeliaSim is running on the same machine (but not in Docker), use `--coppeliaHost host.docker.internal` (on Mac/Windows) or your host's IP address (on Linux).
- If CoppeliaSim is running in another container, ensure both containers are on the same Docker network and use the appropriate service/container name or IP.

### 3. Accessing the Server

- The MCP server will be available at `http://localhost:8000` (or the port you mapped).
- Use the `/sse` endpoint for SSE clients, or POST to `/` for JSON-RPC.

### Troubleshooting

- **Cannot connect to CoppeliaSim:**
  - Make sure the `--coppeliaHost` value is correct and reachable from inside the container.
  - Check firewall and network settings.
- **Port already in use:**
  - Change the host port in the `-p` flag (e.g., `-p 8080:8000`).
- **Persistent data:**
  - This setup does not persist any data. If you need to persist logs or configs, mount a volume.

For advanced usage (e.g., FastMCP server, custom ports, or volumes), adjust the Docker command and arguments as needed.
