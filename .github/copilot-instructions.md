# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

## Adapter-Specific Context

- **Adapter Name**: ioBroker.xs1 (EZcontrol XS1 Adapter)
- **Primary Function**: Integration with EZcontrol XS1 home automation gateway for controlling FS20, FS10, and other home automation devices
- **Target Device**: EZcontrol XS1 gateway (HTTP API based communication)
- **Key Technologies**: HTTP requests to XS1 gateway, sensor/actuator state management
- **Configuration Requirements**: XS1 gateway IP address/URL, device polling intervals
- **Repository**: iobroker-community-adapters/ioBroker.xs1

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
const TEST_XS1_URL = 'http://192.168.1.100'; // Mock XS1 gateway for testing
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test XS1 adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start XS1 adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.xs1.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('XS1 adapter object not found'));
                        }

                        // Configure XS1 adapter properties
                        Object.assign(obj.native, {
                            adresse: TEST_XS1_URL,
                            copylist: "{}",
                            // Add other XS1 configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: XS1 Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: XS1 Adapter started');

                        // Wait for adapter to process XS1 gateway data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking XS1 states after adapter run...');
                        
                        // Validate XS1 connection state
                        const connectionState = await harness.states.getStateAsync('xs1.0.info.connection');
                        
                        if (connectionState && connectionState.val === true) {
                            console.log('✅ SUCCESS: XS1 gateway connection established');
                            resolve();
                        } else {
                            throw new Error('XS1 Test Failed: Expected gateway connection to be established');
                        }

                    } catch (error) {
                        console.error('❌ XS1 Integration test failed:', error.message);
                        reject(error);
                    }
                });
            }).timeout(120000); // 2 minutes timeout for XS1 gateway communication
        });
    }
});
```

#### XS1-Specific Testing Patterns

For XS1 gateway testing, consider these patterns:

```javascript
// Test XS1 device discovery and state updates
it('should discover XS1 actuators and sensors', async function () {
    // Wait for device discovery
    await wait(30000);
    
    // Check for Actuators channel
    const actuatorsChannel = await harness.states.getStateAsync('xs1.0.Actuators');
    expect(actuatorsChannel).to.exist;
    
    // Check for Sensors channel  
    const sensorsChannel = await harness.states.getStateAsync('xs1.0.Sensors');
    expect(sensorsChannel).to.exist;
    
    console.log('✅ XS1 device channels created successfully');
});

// Test XS1 device control functionality
it('should control XS1 actuators', async function () {
    // Mock setting an actuator state
    await harness.states.setStateAsync('xs1.0.Actuators.device1.state', true);
    await wait(5000);
    
    // Verify state change was processed
    const deviceState = await harness.states.getStateAsync('xs1.0.Actuators.device1.state');
    expect(deviceState.val).to.be.true;
    
    console.log('✅ XS1 actuator control test passed');
});
```

### Test Environment Setup

```javascript
// Mock XS1 gateway responses for offline testing
const mockXS1Responses = {
    '/control?callback=x&cmd=get_list_actuators': {
        actuator: [
            { id: 1, name: 'Living Room Light', value: 0, type: 'switch' },
            { id: 2, name: 'Kitchen Light', value: 100, type: 'dimmer' }
        ]
    },
    '/control?callback=x&cmd=get_list_sensors': {
        sensor: [
            { id: 1, name: 'Temperature Sensor', value: 21.5, unit: '°C' },
            { id: 2, name: 'Motion Detector', value: 0, type: 'binary' }
        ]
    }
};

// Use in tests to simulate XS1 gateway responses
function mockXS1Gateway() {
    // Implementation to mock HTTP responses from XS1 gateway
}
```

## Error Handling

### XS1-Specific Error Patterns

Handle these common XS1 gateway scenarios:

```javascript
// Network connectivity issues
try {
    const response = await this.requestAsync(url);
    // Process XS1 response
} catch (error) {
    if (error.code === 'ENOTFOUND' || error.code === 'ECONNREFUSED') {
        this.log.warn(`XS1 gateway not reachable at ${this.config.adresse}: ${error.message}`);
        this.setState('info.connection', false, true);
        return;
    }
    throw error;
}

// XS1 API response validation
function validateXS1Response(response) {
    if (!response || typeof response !== 'object') {
        throw new Error('Invalid XS1 gateway response: no data received');
    }
    
    if (response.error) {
        throw new Error(`XS1 gateway error: ${response.error}`);
    }
    
    return response;
}

// Device state validation
function validateDeviceState(deviceId, value) {
    if (typeof deviceId !== 'number' || deviceId < 1) {
        throw new Error(`Invalid XS1 device ID: ${deviceId}`);
    }
    
    if (typeof value !== 'number' || value < 0 || value > 100) {
        throw new Error(`Invalid XS1 device value: ${value} (must be 0-100)`);
    }
}
```

## Adapter Structure

### Main Adapter Implementation (xs1.js)

```javascript
class XS1Adapter extends utils.Adapter {
    constructor(options = {}) {
        super({
            ...options,
            name: 'xs1',
        });
        
        this.on('ready', this.onReady.bind(this));
        this.on('stateChange', this.onStateChange.bind(this));
        this.on('unload', this.onUnload.bind(this));
    }

    async onReady() {
        // Initialize XS1 connection
        this.setState('info.connection', false, true);
        
        try {
            await this.connectToXS1();
            await this.discoverDevices();
            this.setState('info.connection', true, true);
        } catch (error) {
            this.log.error(`Failed to initialize XS1 adapter: ${error.message}`);
            this.setState('info.connection', false, true);
        }
    }

    async connectToXS1() {
        const xs1Url = this.config.adresse;
        if (!xs1Url) {
            throw new Error('XS1 gateway address not configured');
        }
        
        // Test connection to XS1 gateway
        const testUrl = `${xs1Url}/control?callback=x&cmd=get_protocol_info`;
        const response = await this.requestAsync(testUrl);
        
        this.log.info(`Connected to XS1 gateway: ${response.protocol_info}`);
    }
}
```

## State Management

### XS1 Device States

```javascript
// Create actuator states
async createActuatorStates(actuator) {
    const devicePath = `Actuators.${actuator.name}`;
    
    await this.setObjectNotExistsAsync(`${devicePath}`, {
        type: 'device',
        common: {
            name: actuator.name,
            role: 'actuator'
        },
        native: {
            id: actuator.id,
            type: actuator.type
        }
    });
    
    await this.setObjectNotExistsAsync(`${devicePath}.state`, {
        type: 'state',
        common: {
            name: 'State',
            type: 'number',
            role: 'level',
            min: 0,
            max: 100,
            read: true,
            write: true
        },
        native: {}
    });
    
    // Set current state
    this.setState(`${devicePath}.state`, actuator.value, true);
}

// Handle state changes from ioBroker
async onStateChange(id, state) {
    if (!state || state.ack) return;
    
    const match = id.match(/^xs1\.0\.Actuators\.(.+)\.state$/);
    if (match) {
        const deviceName = match[1];
        await this.controlXS1Device(deviceName, state.val);
    }
}
```

## HTTP Communication

### XS1 API Patterns

```javascript
// Generic XS1 request method
async requestXS1(endpoint, params = {}) {
    const baseUrl = this.config.adresse;
    const queryParams = new URLSearchParams({
        callback: 'x',
        ...params
    }).toString();
    
    const url = `${baseUrl}/control?${queryParams}`;
    
    try {
        const response = await this.requestAsync(url);
        return this.parseXS1Response(response);
    } catch (error) {
        this.log.error(`XS1 request failed for ${endpoint}: ${error.message}`);
        throw error;
    }
}

// Parse XS1 JSONP response
parseXS1Response(responseText) {
    // XS1 returns JSONP format: x({"data": ...})
    const jsonMatch = responseText.match(/x\((.+)\)$/);
    if (!jsonMatch) {
        throw new Error('Invalid XS1 response format');
    }
    
    try {
        return JSON.parse(jsonMatch[1]);
    } catch (error) {
        throw new Error(`Failed to parse XS1 JSON response: ${error.message}`);
    }
}
```

## Logging

```javascript
// XS1-specific logging patterns
this.log.info(`XS1 gateway discovered ${actuators.length} actuators and ${sensors.length} sensors`);
this.log.debug(`XS1 device state update: ${deviceName} = ${value}`);
this.log.warn(`XS1 device ${deviceId} not found in gateway response`);
this.log.error(`XS1 communication failed: ${error.message}`);
```

## Configuration Management

### Admin Configuration (admin/index_m.html)

```javascript
// XS1 configuration validation
function validateXS1Config() {
    const address = $('#xs1_address').val();
    if (!address) {
        showError('XS1 gateway address is required');
        return false;
    }
    
    if (!address.startsWith('http://') && !address.startsWith('https://')) {
        showError('XS1 address must start with http:// or https://');
        return false;
    }
    
    return true;
}

// Test XS1 connection
async function testXS1Connection() {
    const address = $('#xs1_address').val();
    
    try {
        showMessage('Testing XS1 connection...', 'info');
        
        // Test connection via adapter
        const result = await sendToAdapter({
            command: 'testConnection',
            address: address
        });
        
        if (result.success) {
            showMessage('XS1 connection successful!', 'success');
        } else {
            showError(`XS1 connection failed: ${result.error}`);
        }
    } catch (error) {
        showError(`Connection test failed: ${error.message}`);
    }
}
```

## Code Style and Standards

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods
- Handle XS1 gateway communication errors gracefully
- Implement proper state management for devices
- Use consistent naming for XS1 devices and states

## Cleanup and Resource Management

```javascript
async onUnload(callback) {
    try {
        // Clear any polling intervals
        if (this.pollInterval) {
            clearInterval(this.pollInterval);
            this.pollInterval = undefined;
        }
        
        // Clear any connection timeouts
        if (this.connectionTimer) {
            clearTimeout(this.connectionTimer);
            this.connectionTimer = undefined;
        }
        
        // Close any open connections
        this.setState('info.connection', false, true);
        
        callback();
    } catch (e) {
        callback();
    }
}
```

## CI/CD and Testing Integration

### GitHub Actions for XS1 Testing
For XS1 adapter testing without physical hardware:

```yaml
# Tests with mock XS1 gateway (offline testing)
xs1-mock-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false
  
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run XS1 mock tests
      run: npm run test:integration-mock
```

### Package.json Script Integration
Add dedicated script for XS1 testing:
```json
{
  "scripts": {
    "test:integration-mock": "mocha test/integration-mock --exit"
  }
}
```