# M2M Quick Tour

   1. [Client-Server](#client-server)
   2. [Raspberry Pi Remote Control](#raspberry-pi-remote-control)
   3. [Capturing Data from Remote C/C++ Application through IPC (inter-process communication)](https://github.com/EdAlegrid/m2m-ipc-application-demo)
   4. [m2m integration with http web application](https://github.com/EdAlegrid/m2m-web-application-demo)
   5. [m2m integration with websocket  application](https://github.com/EdAlegrid/m2m-websocket-application-demo)

### Client-Server
![](https://raw.githubusercontent.com/EdoLabs/src2/master/quicktour.svg?sanitize=true)
[](quicktour.svg)

In this quick tour, we will create a simple server application that generates random numbers as its sole service and a client application that will access the random numbers using a *pull* and *push* method.

Using a *pull-method*, the client will capture the random numbers as one time function call.

Using a *push-method*, the client will watch the value of the random numbers. The remote device will check the random value every 5 seconds. If the value changes, it will send or *push* the new value to the client.   

Before you start, [create an account](https://www.node-m2m.com/m2m/account/create) and register your remote device. You also need a [node.js](https://nodejs.org/en/) installation on your client and device computers.

#### Remote Device Setup

##### 1. Create a device project directory and install m2m inside the directory.

```js
$ npm install m2m
```

##### 2. Save the code below as device.js within your device project directory.

```js
const m2m = require('m2m');

// create a device object with a device id of 100
// this id must must be registered with node-m2m
let device = new m2m.Device(100);

device.connect(() => {
  // set a channel data resourse named 'random-number'  
  device.setData('random-number', (data) => {
    let rn = Math.floor(Math.random() * 100);
    data.send(rn);
  });
});
```
##### 3. Start your device application.

```js
$ node device.js
```

The first time you run your application, it will ask for your full credentials.
```js
? Enter your userid (email):
? Enter your password:
? Enter your security code:

```
The next time you run your application, it will start automatically using a saved user token.

However, after a grace period of 15 minutes, you need to provide your *security code* to restart your application.

Also at anytime, you can re-authenticate with an *-r* flag with full credentials if you're having difficulty or issues restarting your application as shown below.
```js
$ node device.js -r
```

#### Remote Client Setup

##### 1. Create a client project directory and install m2m inside the directory.

```js
$ npm install m2m
```

##### 2. Save the code below as client.js within your client project directory.

**Method 1**

Create an *alias* object using the client's *accessDevice* method as shown in the code below.

```js
const m2m = require('m2m');

let client = new m2m.Client();

client.connect(() => {

  // access the remote device using an alias object
  let device = client.accessDevice(100);

  // capture 'random-number' data using a pull method
  device.getData('random-number', (data) => {
    console.log('random data', data); // 97
  });

  // capture 'random-number' data using a push method
  device.watch('random-number', (data) => {
    console.log('watch random data', data); // 81, 68, 115 ...
  });
});
```

**Method 2**

Instead of creating an alias, you can just provide the *device id* through the various data access methods from the client object.

```js
const m2m = require('m2m');

let client = new m2m.Client();

client.connect(() => {

  // capture 'random-number' data using a pull method
  client.getData(100, 'random-number', (data) => {
    console.log('random data', data); // 97
  });

  // capture 'random-number' data using a push method
  client.watch(100, 'random-number', (data) => {
    console.log('watch random data', data); // 81, 68, 115 ...
  });
});
```

##### 3. Start your application.
```js
$ node client.js
```
Similar with remote device setup, you will be prompted to enter your credentials.

You should get a similar output result as shown below.
```js
random data 97
watch random data 81
watch random data 68
watch random data 115
```

### Raspberry Pi Remote Control
![](https://raw.githubusercontent.com/EdoLabs/src2/master/quicktour2.svg?sanitize=true)
[](quicktour.svg)

In this quick tour, we will install two push-button switches ( GPIO pin 11 and 13 ) on the remote client, and an led actuator ( GPIO pin 33 ) on the remote device.

The client will attempt to turn *on* and *off* the remote device's actuator and receive a confirmation response of *true* to signify the actuator was indeed turned **on** and *false* when the actuator is turned **off**.

The client will also show an on/off response times providing some insight on the responsiveness of the remote control system.     

#### Remote Device Setup

##### 1. Create a device project directory and install m2m and array-gpio inside the directory.
```js
$ npm install m2m array-gpio
```
##### 2. Save the code below as device.js in your device project directory.

```js
const { Device } = require('m2m');

let device = new Device(200);

device.connect(() => {
  device.setGpio({mode:'output', pin:33});
});
```

##### 3. Start your device application.
```js
$ node device.js
```
#### Remote Client Setup

##### 1. Create a client project directory and install m2m and array-gpio.
```js
$ npm install m2m array-gpio
```
##### 2. Save the code below as client.js in your client project directory.

```js
const { Client } = require('m2m');
const { setInput } = require('array-gpio');

let sw1 = setInput(11); // ON switch
let sw2 = setInput(13); // OFF switch

let client = new Client();

client.connect(() => {

  let t1 = null;
  let device = client.accessDevice(200);

  sw1.watch(1, (state) => {
    if(state){
      t1 = new Date();
      console.log('turning ON remote actuator');
      device.output(33).on((data) => {
        let t2 = new Date();
        console.log('ON confirmation', data, 'response time', t2 - t1, 'ms');
      });
    }
  });

  sw2.watch(1, (state) => {
    if(state){
      t1 = new Date();
      console.log('turning OFF remote actuator');
      device.output(33).off((data) => {
        let t2 = new Date();
        console.log('OFF confirmation', data, 'response time', t2 - t1, 'ms');
      });
    }
  });
});
```
##### 3. Start your application.
```js
$ node client.js
```
The led actuator from remote device should toggle on and off as you press the corresponding ON/OFF switches from the client.

## Device Orchestration

### Remote Machine Monitoring

Install array-gpio for each remote machine.
```js
$ npm install array-gpio
```
#### Server setup
Configure each remote machine's rpi microcontroller with the following GPIO input/output and channel data resources
```js
const { Device } = require('m2m');
const { setInput, setOutput, watchInput } = require('array-gpio');

const sensor1 = setInput(11); // connected to switch sensor1
const sensor2 = setInput(13); // connected to switch sensor2

const actuator1 = setOutput(33); // connected to alarm actuator1
const actuator2 = setOutput(35); // connected to alarm actuator2

// assign 1001, 1002 and 1003 respectively for each remote machine
const device = new Device(1001);

let status = {};

// Local I/O machine control process
watchInput(() => {
  // monitor sensor1
  if(sensor1.isOn){
    actuator1.on();
  }
  else{
    actuator1.off();
  }
  // monitor sensor2
  if(sensor2.isOn){
    actuator2.on();
  }
  else{
    actuator2.off();
  }
});

// m2m device application
device.connect(() => {

  device.setData('machine-status', function(data){

    status.sensor1 = sensor1.state;
    status.sensor2 = sensor2.state;

    status.actuator1 = actuator1.state;
    status.actuator2 = actuator2.state;

    console.log('status', status);

    data.send(JSON.stringify(status));
  });
});
```
