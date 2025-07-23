# Model Context Protocol (MCP): A Complete Guide

## Table of Contents
- [What is MCP?](#what-is-mcp)
- [Architecture Overview](#architecture-overview)
- [The Complete Connection Flow](#the-complete-connection-flow)
- [Understanding the Endpoints](#understanding-the-endpoints)
- [Protocol Deep Dive](#protocol-deep-dive)
- [Code Implementation](#code-implementation)
- [Demo & Testing](#demo--testing)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

## What is MCP?

**Model Context Protocol (MCP)** is a standardized communication protocol that enables AI models like Claude to interact with external tools and services. Think of it as a universal API that allows AI systems to extend their capabilities by calling custom functions, accessing databases, or interacting with external systems.

### Key Benefits
- **Standardized**: One protocol works with all MCP-compatible AI systems
- **Type-safe**: JSON Schema validation ensures correct parameter types
- **Extensible**: Easy to add new tools without changing the core protocol
- **Secure**: Controlled execution environment with defined capabilities

## Architecture Overview

The MCP architecture uses a layered approach with multiple components:

```
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Claude    │    │   MCP Client    │    │  HTTP Transport │    │   Your Server   │
│     AI      │◄──►│  (mcp-remote)   │◄──►│   over TCP/IP   │◄──►│  (Express.js)   │
│             │    │                 │    │                 │    │                 │
│ - Reasoning │    │ - Protocol      │    │ - JSON-RPC 2.0  │    │ - Tool Logic    │
│ - Planning  │    │ - Transport     │    │ - HTTP POST     │    │ - Validation    │
│ - Execution │    │ - Error Handling│    │ - Request/Reply │    │ - Results       │
└─────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Component Responsibilities

1. **Claude AI**: Makes intelligent decisions about when and how to use tools
2. **MCP Client**: Handles protocol translation and connection management
3. **HTTP Transport**: Carries JSON-RPC messages over standard HTTP
4. **Your Server**: Implements the actual tool logic and handles requests

## The Complete Connection Flow

### Phase 1: Server Startup
Your Express.js server starts and listens on port 8000:

```javascript
app.listen(PORT, '0.0.0.0', () => {
  console.log(`▶ Remote MCP Server listening on http://0.0.0.0:${PORT}`);
});
```

### Phase 2: MCP Client Connection
When Claude needs to use MCP tools, the mcp-remote client initiates connection:

```bash
npx mcp-remote http://your-server:8000/mcp --allow-http --transport http-only
```

### Phase 3: Protocol Handshake

#### Step 1: Initialize Request
The client sends an initialization request:

```json
POST /mcp
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {},
    "clientInfo": {
      "name": "claude-ai",
      "version": "0.1.0"
    }
  }
}
```

#### Step 2: Server Response
Your server responds with its capabilities:

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": {}
    },
    "serverInfo": {
      "name": "Remote MCP Server",
      "version": "0.1.0"
    }
  }
}
```

#### Step 3: Initialized Notification
The client confirms initialization (this is a notification - no response expected):

```json
POST /mcp
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

**Important**: Your server should return `204 No Content` for notifications!

### Phase 4: Tool Discovery

#### Step 4: List Tools Request
The client discovers available tools:

```json
POST /mcp
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}
```

#### Step 5: Tools Response
Your server returns the available tools with their schemas:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "add",
        "description": "Return the sum of a and b",
        "inputSchema": {
          "type": "object",
          "properties": {
            "a": { "type": "number" },
            "b": { "type": "number" }
          },
          "required": ["a", "b"]
        }
      },
      {
        "name": "reverse",
        "description": "Return the input text reversed",
        "inputSchema": {
          "type": "object",
          "properties": {
            "text": { "type": "string" }
          },
          "required": ["text"]
        }
      }
    ]
  }
}
```

### Phase 5: Tool Execution

#### Step 6: Tool Call Request
When Claude decides to use a tool:

```json
POST /mcp
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "add",
    "arguments": {
      "a": 15,
      "b": 27
    }
  }
}
```

#### Step 7: Tool Result Response
Your server processes the request and returns the result:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Result: 42"
      }
    ]
  }
}
```

## Understanding the Endpoints

### `/mcp` - The Primary Protocol Endpoint

This is the heart of your MCP server. It handles all protocol communication:

```javascript
app.post('/mcp', (req, res) => {
  const { method, params, id, jsonrpc } = req.body;
  
  // Handle notifications (no id field)
  if (id === undefined && method) {
    console.log(`Handling notification: ${method}`);
    // CRITICAL: Don't send JSON for notifications
    res.status(204).end();
    return;
  }
  
  // Handle method calls (have id field)
  switch (method) {
    case 'initialize':
      // Return server capabilities
      break;
    case 'tools/list':
      // Return available tools
      break;
    case 'tools/call':
      // Execute the requested tool
      break;
  }
});
```

**Why this endpoint exists**: It's the standardized entry point for all MCP communication.

### `/health` - Health Check Endpoint

```javascript
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy', 
    server: 'Remote MCP Server',
    timestamp: new Date().toISOString()
  });
});
```

**Why this endpoint exists**:
- **Monitoring**: Load balancers and monitoring tools can check server health
- **Debugging**: Quick way to verify the server is running
- **DevOps**: Integration with health check systems

### `/mcp` GET Handler - Developer Aid

```javascript
app.get('/mcp', (req, res) => {
  res.json({ 
    message: 'MCP Server is running',
    note: 'Use POST requests for MCP protocol communication'
  });
});
```

**Why this endpoint exists**: When developers accidentally visit the MCP endpoint in a browser, they get a helpful message instead of an error.

## Protocol Deep Dive

### JSON-RPC 2.0 Foundation

MCP is built on JSON-RPC 2.0, which provides:

1. **Request Format**:
```json
{
  "jsonrpc": "2.0",           // Protocol version (required)
  "method": "method_name",    // Method to call (required)
  "params": { ... },          // Parameters (optional)
  "id": 123                   // Request ID (required for requests, absent for notifications)
}
```

2. **Response Format**:
```json
{
  "jsonrpc": "2.0",           // Protocol version (required)
  "id": 123,                  // Matches request ID (required)
  "result": { ... }           // Success result (OR error, not both)
}
```

3. **Error Format**:
```json
{
  "jsonrpc": "2.0",
  "id": 123,
  "error": {
    "code": -32601,           // Standard error code
    "message": "Method not found"
  }
}
```

### Standard Error Codes

```javascript
const MCP_ERRORS = {
  PARSE_ERROR: -32700,        // Invalid JSON
  INVALID_REQUEST: -32600,    // Invalid Request object
  METHOD_NOT_FOUND: -32601,   // Method doesn't exist
  INVALID_PARAMS: -32602,     // Invalid method parameters
  INTERNAL_ERROR: -32603,     // Server internal error
};
```

### Notifications vs Requests

**Requests** (have `id`):
- Expect a response
- Used for: `initialize`, `tools/list`, `tools/call`
- Must return JSON response

**Notifications** (no `id`):
- Fire-and-forget
- Used for: `notifications/initialized`, `notifications/cancelled`
- Must return `204 No Content` (no body)

## Code Implementation

### Complete Server Implementation

```javascript
import express from 'express';
import bodyParser from 'body-parser';
import cors from 'cors';

const app = express();
const PORT = 8000;

// Middleware
app.use(cors());
app.use(bodyParser.json());

// MCP Error Codes
const MCP_ERRORS = {
  PARSE_ERROR: -32700,
  INVALID_REQUEST: -32600,
  METHOD_NOT_FOUND: -32601,
  INVALID_PARAMS: -32602,
  INTERNAL_ERROR: -32603,
};

// Tool implementations
const tools = {
  add: (args) => {
    if (typeof args.a !== 'number' || typeof args.b !== 'number') {
      throw new Error('Parameters a and b must be numbers');
    }
    return `Result: ${args.a + args.b}`;
  },
  
  reverse: (args) => {
    if (typeof args.text !== 'string') {
      throw new Error('Parameter text must be a string');
    }
    return `Result: ${args.text.split('').reverse().join('')}`;
  },
  
  multiply: (args) => {
    if (typeof args.a !== 'number' || typeof args.b !== 'number') {
      throw new Error('Parameters a and b must be numbers');
    }
    return `Result: ${args.a * args.b}`;
  }
};

// Tool schemas
const toolSchemas = [
  {
    name: 'add',
    description: 'Add two numbers together',
    inputSchema: {
      type: 'object',
      properties: {
        a: { type: 'number', description: 'First number' },
        b: { type: 'number', description: 'Second number' }
      },
      required: ['a', 'b']
    }
  },
  {
    name: 'reverse',
    description: 'Reverse a string',
    inputSchema: {
      type: 'object',
      properties: {
        text: { type: 'string', description: 'Text to reverse' }
      },
      required: ['text']
    }
  },
  {
    name: 'multiply',
    description: 'Multiply two numbers',
    inputSchema: {
      type: 'object',
      properties: {
        a: { type: 'number', description: 'First number' },
        b: { type: 'number', description: 'Second number' }
      },
      required: ['a', 'b']
    }
  }
];

// Main MCP endpoint
app.post('/mcp', (req, res) => {
  console.log('📨 Received MCP request:', JSON.stringify(req.body, null, 2));
  
  const { method, params, id, jsonrpc } = req.body;
  
  try {
    // Handle notifications (no response needed)
    if (id === undefined && method) {
      console.log(`🔔 Handling notification: ${method}`);
      switch (method) {
        case 'notifications/initialized':
          console.log('✅ Client initialized successfully');
          break;
        case 'notifications/cancelled':
          console.log('❌ Request cancelled');
          break;
        default:
          console.log(`❓ Unknown notification: ${method}`);
      }
      res.status(204).end();
      return;
    }

    // Validate required fields for requests
    if (!jsonrpc || jsonrpc !== '2.0') {
      return res.status(400).json({
        jsonrpc: '2.0',
        id: id || null,
        error: {
          code: MCP_ERRORS.INVALID_REQUEST,
          message: 'Invalid or missing jsonrpc field'
        }
      });
    }

    if (!method) {
      return res.status(400).json({
        jsonrpc: '2.0',
        id: id || null,
        error: {
          code: MCP_ERRORS.INVALID_REQUEST,
          message: 'Missing method field'
        }
      });
    }

    if (id === undefined) {
      return res.status(400).json({
        jsonrpc: '2.0',
        id: null,
        error: {
          code: MCP_ERRORS.INVALID_REQUEST,
          message: 'Missing id field for request'
        }
      });
    }

    // Handle method calls
    switch (method) {
      case 'initialize':
        console.log('🤝 Initializing MCP connection');
        res.json({
          jsonrpc: '2.0',
          id: id,
          result: {
            protocolVersion: '2025-06-18',
            capabilities: {
              tools: {}
            },
            serverInfo: {
              name: 'Demo MCP Server',
              version: '1.0.0'
            }
          }
        });
        break;
        
      case 'tools/list':
        console.log('📋 Listing available tools');
        res.json({
          jsonrpc: '2.0',
          id: id,
          result: {
            tools: toolSchemas
          }
        });
        break;
        
      case 'tools/call':
        if (!params || !params.name || !params.arguments) {
          return res.status(400).json({
            jsonrpc: '2.0',
            id: id,
            error: {
              code: MCP_ERRORS.INVALID_PARAMS,
              message: 'Missing required parameters: name and arguments'
            }
          });
        }

        const { name, arguments: args } = params;
        console.log(`🔧 Calling tool: ${name} with args:`, args);

        if (!tools[name]) {
          return res.status(400).json({
            jsonrpc: '2.0',
            id: id,
            error: {
              code: MCP_ERRORS.METHOD_NOT_FOUND,
              message: `Unknown tool: ${name}`
            }
          });
        }

        try {
          const result = tools[name](args);
          console.log(`✅ Tool ${name} result:`, result);
          
          res.json({
            jsonrpc: '2.0',
            id: id,
            result: {
              content: [
                { type: 'text', text: result }
              ]
            }
          });
        } catch (toolError) {
          console.error(`❌ Tool ${name} error:`, toolError.message);
          res.status(400).json({
            jsonrpc: '2.0',
            id: id,
            error: {
              code: MCP_ERRORS.INVALID_PARAMS,
              message: toolError.message
            }
          });
        }
        break;
        
      default:
        res.status(400).json({
          jsonrpc: '2.0',
          id: id,
          error: {
            code: MCP_ERRORS.METHOD_NOT_FOUND,
            message: `Method not found: ${method}`
          }
        });
    }
  } catch (error) {
    console.error('💥 Server error:', error);
    res.status(500).json({
      jsonrpc: '2.0',
      id: id || null,
      error: {
        code: MCP_ERRORS.INTERNAL_ERROR,
        message: error.message
      }
    });
  }
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy', 
    server: 'Demo MCP Server',
    version: '1.0.0',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

// Debug endpoint
app.get('/mcp', (req, res) => {
  res.json({ 
    message: 'MCP Server is running',
    note: 'Use POST requests for MCP protocol communication',
    endpoints: {
      '/mcp': 'POST - MCP protocol endpoint',
      '/health': 'GET - Health check',
      '/debug': 'GET - Server debug info'
    }
  });
});

// Debug info endpoint
app.get('/debug', (req, res) => {
  res.json({
    server: 'Demo MCP Server',
    version: '1.0.0',
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    availableTools: toolSchemas.map(tool => ({
      name: tool.name,
      description: tool.description
    }))
  });
});

// Start server
app.listen(PORT, '0.0.0.0', () => {
  console.log('🚀 =================================');
  console.log(`🚀 MCP Server running on port ${PORT}`);
  console.log('🚀 =================================');
  console.log(`📊 Health check: http://localhost:${PORT}/health`);
  console.log(`🔧 MCP endpoint: http://localhost:${PORT}/mcp`);
  console.log(`🐛 Debug info: http://localhost:${PORT}/debug`);
  console.log('🚀 =================================');
});

// Graceful shutdown
process.on('SIGINT', () => {
  console.log('\n🛑 Received SIGINT, shutting down gracefully...');
  process.exit(0);
});

process.on('SIGTERM', () => {
  console.log('\n🛑 Received SIGTERM, shutting down gracefully...');
  process.exit(0);
});
```

### Client Configuration

To connect Claude to your MCP server, you'll need a configuration like this:

```json
{
  "aiarchives": {
    "command": "npx",
    "args": [
      "-y", 
      "mcp-remote", 
      "http://your-server-url:8000/mcp",
      "--allow-http",
      "--transport",
      "http-only"
    ]
  }
}
```

## Demo & Testing

### 1. Manual Testing with curl

Test the health endpoint:
```bash
curl http://localhost:8000/health
```

Test tool listing:
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }'
```

Test tool execution:
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "add",
      "arguments": {"a": 15, "b": 27}
    }
  }'
```

### 2. Testing with Claude

Once connected, test these prompts:

**Tool Discovery:**
- "What MCP tools do you have available?"
- "List your available tools"

**Basic Tool Usage:**
- "Use the add tool to calculate 123 + 456"
- "Can you reverse the text 'Hello World' using your tools?"
- "Multiply 7 and 8 using the multiply tool"

**Error Handling:**
- "Use the add tool with 'hello' and 'world'" (should get type error)
- "Use a tool called 'divide' with 10 and 2" (should get tool not found error)

### 3. Debugging Tips

Enable detailed logging:
```javascript
// Add this middleware for request logging
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} ${req.method} ${req.path}`);
  if (req.method === 'POST' && req.path === '/mcp') {
    console.log('Request body:', JSON.stringify(req.body, null, 2));
  }
  next();
});
```

## Troubleshooting

### Common Issues

#### 1. "Connection Error" with Zod Validation
**Problem**: Server returning JSON for notifications
**Solution**: Return `204 No Content` for notifications (requests without `id`)

```javascript
// ❌ Wrong - returns JSON for notifications
if (!id && method) {
  res.json({});  // This breaks the protocol!
}

// ✅ Correct - returns no content for notifications
if (id === undefined && method) {
  res.status(204).end();
}
```

#### 2. "Method Not Found" Errors
**Problem**: Missing method handlers or typos
**Solution**: Ensure all required methods are implemented:
- `initialize`
- `tools/list` 
- `tools/call`
- `notifications/initialized`

#### 3. Parameter Validation Errors
**Problem**: Incorrect parameter types
**Solution**: Add comprehensive validation:

```javascript
// Validate parameters match schema
const validateParams = (toolName, args) => {
  const schema = toolSchemas.find(t => t.name === toolName);
  if (!schema) return false;
  
  for (const required of schema.inputSchema.required) {
    if (!(required in args)) {
      throw new Error(`Missing required parameter: ${required}`);
    }
  }
  
  for (const [key, value] of Object.entries(args)) {
    const expectedType = schema.inputSchema.properties[key]?.type;
    if (expectedType && typeof value !== expectedType) {
      throw new Error(`Parameter ${key} must be ${expectedType}, got ${typeof value}`);
    }
  }
  
  return true;
};
```

#### 4. CORS Issues
**Problem**: Browser blocking requests
**Solution**: Proper CORS configuration:

```javascript
app.use(cors({
  origin: ['https://claude.ai', 'http://localhost:3000'],
  methods: ['GET', 'POST'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Debugging Checklist

- [ ] Server starts without errors
- [ ] Health endpoint returns 200
- [ ] MCP endpoint accepts POST requests
- [ ] All required methods implemented
- [ ] Notifications return 204 (no JSON)
- [ ] Tool schemas are valid JSON Schema
- [ ] Error responses include proper error codes
- [ ] Parameters are validated correctly

## Best Practices

### 1. Error Handling
Always return proper JSON-RPC error responses:

```javascript
const sendError = (res, id, code, message) => {
  res.status(400).json({
    jsonrpc: '2.0',
    id: id || null,
    error: { code, message }
  });
};
```

### 2. Input Validation
Validate all inputs against your schemas:

```javascript
const validateToolInput = (toolName, args) => {
  const tool = toolSchemas.find(t => t.name === toolName);
  if (!tool) throw new Error(`Unknown tool: ${toolName}`);
  
  // Use a proper JSON Schema validator in production
  // This is a simplified example
  const { required, properties } = tool.inputSchema;
  
  for (const field of required) {
    if (!(field in args)) {
      throw new Error(`Missing required field: ${field}`);
    }
  }
  
  for (const [key, value] of Object.entries(args)) {
    const fieldSchema = properties[key];
    if (fieldSchema && typeof value !== fieldSchema.type) {
      throw new Error(`Field ${key} must be ${fieldSchema.type}`);
    }
  }
};
```

### 3. Logging and Monitoring
Implement comprehensive logging:

```javascript
const logger = {
  info: (message, data) => console.log(`ℹ️  ${message}`, data || ''),
  error: (message, error) => console.error(`❌ ${message}`, error),
  warn: (message, data) => console.warn(`⚠️  ${message}`, data || ''),
  debug: (message, data) => console.log(`🐛 ${message}`, data || '')
};
```

### 4. Security Considerations
- Validate all inputs thoroughly
- Implement rate limiting for production use
- Use HTTPS in production
- Don't expose internal errors to clients
- Implement proper authentication if needed

### 5. Performance Tips
- Keep tool implementations fast (< 5 seconds)
- Cache results when appropriate
- Use connection pooling for database access
- Implement proper error timeouts

### 6. Documentation
Document your tools clearly:

```javascript
{
  name: 'weather',
  description: 'Get current weather for a city. Returns temperature, humidity, and conditions.',
  inputSchema: {
    type: 'object',
    properties: {
      city: { 
        type: 'string', 
        description: 'City name (e.g., "New York" or "London, UK")'
      },
      units: { 
        type: 'string', 
        enum: ['celsius', 'fahrenheit'],
        default: 'celsius',
        description: 'Temperature units'
      }
    },
    required: ['city']
  }
}
```

## Conclusion

The Model Context Protocol provides a powerful, standardized way to extend AI capabilities. By understanding the protocol flow, implementing proper error handling, and following best practices, you can create robust tools that integrate seamlessly with AI systems like Claude.

Key takeaways:
- MCP uses JSON-RPC 2.0 over HTTP for communication
- The protocol has a clear handshake and discovery process
- Proper error handling and validation are crucial
- Notifications require special handling (204 No Content)
- Tool schemas enable type-safe parameter passing

This foundation enables you to build sophisticated AI-powered applications with custom functionality tailored to your specific needs.
