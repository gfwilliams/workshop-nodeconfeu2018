## [Step 5 - Notifications](step5.md)

## Step 5 : Notifications

Ok, so what if we want to get data from a Bluetooth Peripheral?

Well, we could use `characteristic.readValue()` which behaves exactly like
you'd expect - it returns a promise which resolves with the value.

However in many cases we're not interested repeatedly making readings,
but in getting notified when a reading changes.

* Ensure everything (including the Web Bluetooth page) is disconnected from Espruino
* Close the tab with the Web Bluetooth Page (the Badge's services can end up being cached)

![](img/workshop4.png)

* Choose `Accelerometer` from the Badge's menu

![](img/workshop3.png)

* You should now see a menu saying the badge is connectable.

The Badge has now configured itself as the following:

* **Service:** UUID 7b340000-105b-2b38-3a74-2932f884e90e - Badge Service
  * **Characteristic:** UUID 7b340003-105b-2b38-3a74-2932f884e90e - Accelerometer - 3 bytes (notifyable)

So all we have to do is use `getPrimaryService` and `getCharacteristic` to find
the characteristic as before, but then `characteristic.startNotifications()` to
start us receiving `characteristicvaluechanged` events whenever the badge
sends a notification.

```HTML
<html>
<body>
<button onclick="connect()">Connect to Web Bluetooth</button>
<div id="data"></div>
<script>
function connect() {
  var options =  {
    filters: [
      {namePrefix: 'Pixl.js'},
    ],
    optionalServices: [ "7b340000-105b-2b38-3a74-2932f884e90e" ]
  };
  var busy = false;
  var gatt, service;
  navigator.bluetooth.requestDevice(options).then(function(device) {
    console.log('Device: ' + JSON.stringify(device));
    return device.gatt.connect();
  }).then(function(g) {
   gatt = g;
   // Get our custom service
   return gatt.getPrimaryService("7b340000-105b-2b38-3a74-2932f884e90e");
  }).then(function(s) {
    service = s;
    // Get the Acceleorometer characteristic
    return service.getCharacteristic("7b340003-105b-2b38-3a74-2932f884e90e");
  }).then(function(characteristic) {
    var dataDiv = document.querySelector("#data");
    // When we get a notification, write the data to a div below the button
    characteristic.addEventListener('characteristicvaluechanged', function(event) {
      var value = event.target.value.buffer; // an arraybuffer
      dataDiv.textContent = (new Int8Array(value)).toString();
    });
    // Now start getting notifications
    return characteristic.startNotifications();
  }).then(function() {
    //gatt.disconnect();
    console.log("Done!");
  }).catch(function(error) {
    console.log("Something went wrong. " + error);
  });
}
</script>
</body>
</html>
```

### How do you do this on the Bluetooth Peripheral?

The code needed to handle this on the peripheral (which is being called when
you choose the menu item) is simply:

```JS
NRF.setServices({
 "7b340000-105b-2b38-3a74-2932f884e90e" : {
   "7b340003-105b-2b38-3a74-2932f884e90e" : {
     readable : true,
     notify : true,
     value : [0,0,0]
   }
 }
},{uart:false});
// Only start sending notifications a few seconds
// after we're connected to.
var started = false;
NRF.on('connect',()=>{
 if (started) return;
 started = true;
 setTimeout(()=>{
   // Update the service with new accelerometer data
   setInterval(()=>{
     var accel = NC.accel();
     NRF.updateServices({
       "7b340000-105b-2b38-3a74-2932f884e90e" : {
         "7b340003-105b-2b38-3a74-2932f884e90e" : {
           value : [accel.x*63,accel.y*63,accel.z*63],
           notify: true
         }
       }
     });
   },200);
 }, 2000);
});
```


## That's it!
