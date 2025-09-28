# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### XS1 EZcontrol Adapter Context

This is the **ioBroker.xs1** adapter that integrates with EZcontrol XS1 home automation gateways. The adapter communicates with XS1 devices through their RestAPI and provides real-time event listening capabilities.

**Key Characteristics:**
- **Target System**: EZcontrol XS1 home automation gateway
- **Communication Protocol**: HTTP RestAPI and WebSocket listening
- **Device Support**: FS20, FS10, and other 433MHz/868MHz home automation devices
- **Function**: Bidirectional communication with sensors (read-only) and actuators (read/write)
- **Real-time Updates**: Event listener for immediate state synchronization
- **Authentication**: Currently supports non-password protected XS1 gateways only
- **Special Features**: Direct device linking via copylist functionality

**Configuration Requirements:**
- XS1 gateway IP address or hostname
- Network accessibility to XS1 RestAPI
- Optional copylist configuration for device linking
- Battery status monitoring for compatible sensors

**Common API Patterns:**
- RestAPI calls for device control and status queries
- WebSocket or polling listener for real-time updates
- State management with proper ack flag handling (ack=false for commands, ack=true for confirmations)
- LOWBAT state generation for battery-powered sensors

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();

                        // Set adapter configuration
                        await harness.changeAdapterConfig('adapterName', {
                            enabled: true,
                            coordinates: TEST_COORDINATES
                        });

                        // Start adapter and wait for it to be ready
                        await harness.startAdapter();
                        await wait(10000); // Wait for adapter initialization

                        // Verify adapter is running
                        const isAlive = await harness.getAdapterAlive('adapterName');
                        if (!isAlive) {
                            throw new Error('Adapter did not start properly');
                        }

                        resolve();
                    } catch (error) {
                        reject(error);
                    }
                });
            });

            it('should create expected objects', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        // Wait for objects to be created
                        await wait(5000);

                        // Check for specific objects
                        const objects = await harness.getAllObjects();
                        // Add assertions for expected objects

                        resolve();
                    } catch (error) {
                        reject(error);
                    }
                });
            });

            after(async function() {
                // Cleanup
                if (harness) {
                    await harness.stopAdapter();
                }
            });
        });
    }
});
```

#### Test Execution Order
1. **Package Tests**: `npm run test:package` - Tests package.json structure
2. **Unit Tests**: `npm run test:js` - Tests individual functions  
3. **Integration Tests**: `npm run test:integration` - Tests adapter lifecycle and functionality

#### Common Testing Patterns
- Always use `defineAdditionalTests` callback pattern with `getHarness()`
- Include proper error handling with try-catch blocks
- Add appropriate wait times for async operations to complete
- Test both successful operations and error conditions
- Use `getAllObjects()` to verify created ioBroker objects
- Always include cleanup in `after()` hooks

## Development Patterns

### Adapter Structure
Follow the standard ioBroker adapter structure:
```
- main.js (or xs1.js)          # Main adapter code
- admin/                       # Admin interface files
- io-package.json             # ioBroker package definition
- package.json                # npm package definition
- lib/                        # Helper libraries
- test/                       # Test files
```

### State Management
```javascript
// Set states with appropriate acknowledgment
this.setState('device.sensor', { val: value, ack: false }); // Command from ioBroker
this.setState('device.sensor', { val: value, ack: true });  // Status from device

// Subscribe to state changes
this.subscribeStates('*');
```

### Error Handling
```javascript
try {
    // Risky operation
} catch (error) {
    this.log.error(`Operation failed: ${error.message}`);
    this.setState('info.connection', false, true);
}
```

### Connection Management
```javascript
onReady() {
    this.setState('info.connection', false, true);
    this.connectToDevice();
}

onUnload(callback) {
    this.disconnectFromDevice();
    callback && callback();
}
```

## Code Style Guidelines

### ESLint Configuration
Use the provided `.eslintrc.json` configuration:
- Extends `@iobroker/adapter-dev/eslint-config`
- JavaScript ES2020 features enabled
- Node.js 18+ environment

### Logging
Use appropriate log levels:
```javascript
this.log.debug('Detailed debugging information');
this.log.info('General information');
this.log.warn('Warning message');
this.log.error('Error message');
```

### Async/Await Patterns
Prefer async/await over Promise chains:
```javascript
async onReady() {
    try {
        await this.initializeConnection();
        await this.loadConfiguration();
        this.log.info('Adapter initialized successfully');
    } catch (error) {
        this.log.error(`Initialization failed: ${error.message}`);
    }
}
```

## ioBroker-specific Patterns

### Object Creation
```javascript
await this.setObjectNotExists('sensors.temperature', {
    type: 'state',
    common: {
        name: 'Temperature Sensor',
        type: 'number',
        role: 'value.temperature',
        unit: '°C',
        read: true,
        write: false
    },
    native: {}
});
```

### State Subscription
```javascript
onStateChange(id, state) {
    if (!state || state.ack) return; // Ignore acknowledged states
    
    const command = id.split('.').pop();
    this.handleCommand(command, state.val);
}
```

### Configuration Handling
```javascript
onReady() {
    const config = this.config;
    if (!config.hostname || !config.port) {
        this.log.error('Missing required configuration');
        return;
    }
}
```

## Device Integration

### RestAPI Communication
```javascript
async makeRequest(endpoint, method = 'GET', data = null) {
    const url = `http://${this.config.hostname}:${this.config.port}${endpoint}`;
    
    try {
        const response = await fetch(url, {
            method,
            headers: { 'Content-Type': 'application/json' },
            body: data ? JSON.stringify(data) : undefined
        });
        
        if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }
        
        return await response.json();
    } catch (error) {
        this.log.error(`API request failed: ${error.message}`);
        throw error;
    }
}
```

### Event Listening
```javascript
setupEventListener() {
    // WebSocket or polling implementation
    this.listener = new WebSocket(`ws://${this.config.hostname}:${this.config.port}/events`);
    
    this.listener.on('message', (data) => {
        const event = JSON.parse(data);
        this.handleDeviceEvent(event);
    });
    
    this.listener.on('error', (error) => {
        this.log.error(`Event listener error: ${error.message}`);
    });
}
```

### Device State Mapping
```javascript
mapDeviceState(deviceData) {
    const states = {};
    
    if (deviceData.sensors) {
        deviceData.sensors.forEach(sensor => {
            states[`sensors.${sensor.name}`] = {
                val: sensor.value,
                ack: true
            };
        });
    }
    
    return states;
}
```

## Common Patterns

### Initialization Sequence
1. Validate configuration
2. Establish connection to external system  
3. Create ioBroker objects
4. Set up event listeners
5. Start polling/monitoring
6. Set connection state to true

### Error Recovery
- Implement connection retry logic
- Handle network timeouts gracefully  
- Provide meaningful error messages
- Reset connection state on failures

### Resource Cleanup
Always clean up resources in `onUnload()`:
```javascript
onUnload(callback) {
    if (this.pollInterval) {
        clearInterval(this.pollInterval);
    }
    
    if (this.connection) {
        this.connection.close();
    }
    
    callback && callback();
}
```

### TypeScript Support
If using TypeScript:
- Use proper type definitions for ioBroker
- Define interfaces for external API responses
- Implement strict null checking
- Use proper error type handling

## Security Considerations

- Validate all user inputs from admin interface
- Sanitize data before API requests
- Handle authentication securely
- Don't log sensitive information (passwords, API keys)
- Use HTTPS when possible for external communications

## Performance Optimization  

- Implement connection pooling for HTTP requests
- Use appropriate polling intervals
- Batch multiple state updates when possible
- Cache frequently accessed data
- Implement proper memory cleanup

## Documentation Standards

- Update README.md with clear setup instructions
- Document configuration parameters in io-package.json
- Provide example configurations
- Include troubleshooting section
- Document API endpoints and data formats

## Common Gotchas

### State Updates
- Always check `state.ack` before processing commands
- Use `setState` with correct acknowledgment flags
- Don't process states during adapter startup (wait for `onReady`)

### Object Management
- Use `setObjectNotExists` for initial object creation
- Don't delete/recreate objects unnecessarily  
- Handle object changes properly in admin interface

### Memory Management
- Clean up intervals and timeouts
- Remove event listeners on unload
- Close database/file connections
- Clear large data structures

## Release Management

Use the standard ioBroker release process:
1. Update CHANGELOG.md
2. Update version in io-package.json AND package.json
3. Use `npm run release` command
4. Tag releases appropriately
5. Update adapter in ioBroker repository

## Additional Resources

- [ioBroker Adapter Development Documentation](https://github.com/ioBroker/ioBroker.docs)
- [ioBroker Adapter Template](https://github.com/ioBroker/ioBroker.template)
- [ioBroker Testing Framework](https://github.com/ioBroker/testing)
- [ioBroker Developer Portal](https://www.iobroker.net/#en/developer)