[![MseeP.ai Security Assessment Badge](https://mseep.net/pr/mcp-mirror-doronaviguy-mpc-0x-badge.png)](https://mseep.ai/app/mcp-mirror-doronaviguy-mpc-0x)

# MCP Ethereum Address Info Server

This server provides information about Ethereum addresses across multiple chains using the Model Context Protocol (MCP). It includes a Server-Sent Events (SSE) endpoint for real-time updates.

## Table of Contents

- [Setup](#setup)
- [Running the Server](#running-the-server)
- [Available Endpoints](#available-endpoints)
- [Using the SSE Endpoint](#using-the-sse-endpoint)
- [Testing with Curl](#testing-with-curl)
- [Example Workflow](#example-workflow)

## Setup

1. Clone the repository:
   ```bash
   git clone <repository-url>
   cd mcp-0x-address
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Create a `.env` file with the following variables:
   ```
   MCP_PORT=3002
   ```

## Running the Server

Start the HTTP MCP server:

```bash
npm run start:http
```

This will start the server on port 3002 (or the port specified in your `.env` file).

## Available Endpoints

The server provides the following endpoints:

- `GET /health` - Server health check
- `POST /mcp` - MCP endpoint for tool calls
- `GET /sse` - Server-Sent Events endpoint for real-time updates
- `GET /sse/clients` - Get information about connected SSE clients
- `POST /sse/subscribe/:clientId` - Subscribe to address updates
- `POST /sse/unsubscribe/:clientId` - Unsubscribe from address updates

## Using the SSE Endpoint

The SSE endpoint allows clients to receive real-time updates from the server. Here's how to use it:

1. Connect to the SSE endpoint
2. Get your client ID from the connection response
3. Subscribe to specific addresses
4. Receive real-time updates for those addresses

## Testing with Curl

### 1. Connect to the SSE Endpoint

```bash
curl -N http://localhost:3002/sse
```

This will establish a connection to the SSE endpoint and start receiving events. The connection will remain open until you manually terminate it.

### 2. Check Connected Clients

```bash
curl http://localhost:3002/sse/clients
```

### 3. Subscribe to Address Updates

After connecting to the SSE endpoint, you'll receive a client ID. Use that ID to subscribe to address updates:

```bash
curl -X POST \
  http://localhost:3002/sse/subscribe/YOUR_CLIENT_ID \
  -H "Content-Type: application/json" \
  -d '{"addresses": ["0x742d35Cc6634C0532925a3b844Bc454e4438f44e", "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"]}'
```

Replace `YOUR_CLIENT_ID` with the client ID you received when connecting to the SSE endpoint.

### 4. Unsubscribe from Address Updates

```bash
curl -X POST \
  http://localhost:3002/sse/unsubscribe/YOUR_CLIENT_ID \
  -H "Content-Type: application/json" \
  -d '{"addresses": ["0x742d35Cc6634C0532925a3b844Bc454e4438f44e"]}'
```

### 5. Trigger an Address Update

To trigger an address update (which will be sent to subscribed clients), call the `get-address-info` tool:

```bash
curl -X POST \
  http://localhost:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "get-address-info",
      "arguments": {
        "address": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e"
      }
    }
  }'
```

### 6. Check Server Health

```bash
curl http://localhost:3002/health
```

### 7. Test the Ping Tool

```bash
curl -X POST \
  http://localhost:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "ping",
      "arguments": {}
    }
  }'
```

## Example Workflow

Here's a complete workflow for testing the SSE functionality:

1. Start the server:
   ```bash
   npm run start:http
   ```

2. In a new terminal, connect to the SSE endpoint:
   ```bash
   curl -N http://localhost:3002/sse
   ```

   You'll receive a response like:
   ```
   data: {"type":"connection","clientId":"client-1234567890abcdef","message":"Connected to MCP SSE endpoint","timestamp":"2023-01-01T00:00:00.000Z"}
   ```

3. Note the `clientId` from the response.

4. In another terminal, subscribe to address updates:
   ```bash
   curl -X POST \
     http://localhost:3002/sse/subscribe/client-1234567890abcdef \
     -H "Content-Type: application/json" \
     -d '{"addresses": ["0x742d35Cc6634C0532925a3b844Bc454e4438f44e"]}'
   ```

5. Trigger an address update:
   ```bash
   curl -X POST \
     http://localhost:3002/mcp \
     -H "Content-Type: application/json" \
     -d '{
       "jsonrpc": "2.0",
       "id": 1,
       "method": "tools/call",
       "params": {
         "name": "get-address-info",
         "arguments": {
           "address": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e"
         }
       }
     }'
   ```

6. In the terminal where you're connected to the SSE endpoint, you'll see updates for the address.

## Automated Testing Script

For a more automated test, you can use this bash script:

```bash
#!/bin/bash

# Start SSE connection in the background and capture the output
curl -N http://localhost:3002/sse > sse_output.txt &
SSE_PID=$!

# Wait a moment for the connection to establish
sleep 2

# Extract the client ID from the output
CLIENT_ID=$(grep -o '"clientId":"[^"]*"' sse_output.txt | head -1 | cut -d'"' -f4)

if [ -z "$CLIENT_ID" ]; then
  echo "Failed to get client ID"
  kill $SSE_PID
  exit 1
fi

echo "Connected with client ID: $CLIENT_ID"

# Subscribe to an address
curl -X POST \
  http://localhost:3002/sse/subscribe/$CLIENT_ID \
  -H "Content-Type: application/json" \
  -d '{"addresses": ["0x742d35Cc6634C0532925a3b844Bc454e4438f44e"]}'

echo "Subscribed to address. Waiting for updates..."
echo "Press Ctrl+C to stop"

# Keep the script running to see updates
tail -f sse_output.txt

# Clean up on exit
trap "kill $SSE_PID; rm sse_output.txt" EXIT
```

Save this as `test_sse.sh`, make it executable with `chmod +x test_sse.sh`, and run it with `./test_sse.sh`.