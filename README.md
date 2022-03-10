
# RAK7204 - Home Environmental Sensor 

- [RAK7204 - Home Environmental Sensor](#rak7204---home-environmental-sensor)
  - [TagoIO Integration](#tagoio-integration)
  - [Payload Parser](#payload-parser)
  - [TagoIO Dashboard](#tagoio-dashboard)


## TagoIO Integration

In order to send data from Helium to TagoIO, you will have to create an Helium integration with TagoIO. Follow [this article](https://docs.helium.com/use-the-network/console/integrations/tago/) to achieve that. 

## Payload Parser

Down below, the JavaScript ["Payload Parser"](https://gist.github.com/vhuynen/2bc0bdeec4b2c718508d8c2bd0bb4892) that decodes the Cayenne LPP payload into JSON format expected by TagoIO API:

``` js
// convert string to short integer
function parseShort(str, base) {
  var n = parseInt(str, base);
  return (n << 16) >> 16;
}

// Search the payload variable in the payload global variable. It's contents is always [ { variable, value...}, {variable, value...} ...]
const payload_raw = payload.find(x => x.variable === 'payload_raw' || x.variable === 'payload' || x.variable === 'data');
const data = [];
// Set Hex value into buffer
var buffer = payload_raw.value;

if (payload_raw) {
  try {
    while (buffer.length > 4) {
      var flag = parseInt(buffer.substring(0, 4), 16);
      switch (flag) {
        case 0x0768:// Humidity
          data.push({ variable: 'humidity', value: parseFloat(((parseShort(buffer.substring(4, 6), 16) * 0.01 / 2) * 100).toFixed(1)), unit: '%' });
          buffer = buffer.substring(6);
          break;
        case 0x0673:// Atmospheric pressure
          data.push({ variable: 'barometer', value: parseFloat((parseShort(buffer.substring(4, 8), 16) * 0.1).toFixed(2)), unit: 'hPa' });
          buffer = buffer.substring(8);
          break;
        case 0x0267:// Temperature
          data.push({ variable: 'temperature', value: parseFloat((parseShort(buffer.substring(4, 8), 16) * 0.1).toFixed(2)), unit: '°C' });
          buffer = buffer.substring(8);
          break;
        case 0x0402:// Gas Resistance
          data.push({ variable: 'gasResistance', value: parseFloat((parseShort(buffer.substring(4, 8), 16) * 0.01).toFixed(2)), unit: 'KΩ' });
          buffer = buffer.substring(8);
          break;
        case 0x0802:// Battery Voltage
          data.push({ variable: 'voltage', value: parseFloat((parseShort(buffer.substring(4, 8), 16) * 0.01).toFixed(2)), unit: 'Volts' });
          buffer = buffer.substring(8);
          break;
        case 0x0c65:// Luminosity Sensor
          data.push({ variable: 'luminosity', value: parseFloat((parseShort(buffer.substring(4, 8), 16)).toFixed(4)), unit: 'Lux' });
          buffer = buffer.substring(8);
          break;
        default:
          buffer = buffer.substring(7);
          break;
      }
    }

    // This will concat the content sent by your device with the content generated in this payload parser.
    // It also add the field "serie" and "time" to it, copying from your sensor data.
    payload = payload.concat(data.map(x => ({ ...x, serie: payload_raw.serie, time: payload_raw.time })));
  } catch (e) {
    // Print the error to the Live Inspector.
    console.error(e);
    // Return the variable parse_error for debugging.
    payload = [{ variable: 'parse_error', value: e.message }];
  }
}

```

## TagoIO Dashboard

![dashboard](./docs/gallery/dashboard_tagoio.png)