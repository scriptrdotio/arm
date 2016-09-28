### About The connector to ARM mbed Device Connector :)
ARM provides the [ARM mbed™ Device Connector service](https://connector.mbed.com/), a cloud service that allows connecting IoT devices
to the cloud, notably to ARM's [mbed IoT device platform](https://www.mbed.com/en/) cloud platform. Using ARM's device connector, devices
operated by [ARM's mbed OS](https://www.mbed.com/en/platform/mbed-os/) become at the reach of REST APIs calls using the Device Connector
service's APIs. The purpose of our connector is to expose the latter to scriptr.io developers as first class JavaScript objects 
that can be easily invoked from within their scripts.

### Prerequisites
- You need to own an ARM mbed Device connector account 
- You need to have [created an application](https://connector.mbed.com/#accesskeys) on ARM's platform in order to obtain an access key
- You need to connect devices to your mbed Device connector account (you can real devices operated by ARM mbed OS or simulators)

### How to use

The connector provides classes that are abstractions of the different APIs exposed by ARM's mbed Device Connector 
(aka Device Server):

- DeviceServer: is a kind of factory that gives you access to the other services. It is the main entry point of your application,
- DirectoryService: wraps the APIs that allow you to manipulate endpoints (i.e. your exposed devices),
- NotificationService: wraps the APIs that allow you to deal with notifications (e.g. subscribe a callback to receive notifications and asynchronous responses),
- SubscriptionService: wraps the APIs that allow you to deal with subscriptions, e.g. subscribe to automatic notifications from devices

In addition to the above, the main construct you will be dealing with is the endpoint, which notably allows you to read from your devices 
and write to them, i.e. send them values, instruct them to execute functions, etc.

### Obtaining a reference to an endpoint

There are two ways of doing that:

- Either by invoking the constructor of the Endpoint class, which is straightforward but requires you
to pass an instance of the Client class, which encapsulates the technicalities on how to invoke ARM's APIs, 
- Or by asking the DirectoryService to give you such a reference. 

In the below example, we illustrate that second option. We first start by creating an instance of DeviceServer, then obtain from the latter
an instance of DirectoryService that we use to obtain your endpoint, passing its url to identify it:

```
var deviceserverModule = require("/arm/mbed/deviceserver");
var deviceServer = new deviceserverModule.DeviceServer({accessKey:"YOUR_ARM_ACCESS_KEY"});
var directoryService = deviceServer.getDirectoryService();
var endpoint = directoryService.getEndpoint("/actuator/0/speed");
```
**Notes:** 
- If you specified your access key in the confiuration file (see below), you can just invoke the constructor of DeviceServer without
passing any parameter,
- You can also use the directory service to list the available endpoints then pick an endpoint from the list.

### Adding a notification callback

Reading from a device or obtaining automatic notifications from devices is an asynchronous activity that requires you to register
a callback. Callbacks should be URLs of scripts. 

The connector provides you with a default callback script "/arm/mbed/api/callback" that knows how to forward the notification types 
to their specific handlers, based on the content of the "/arm/mbed/handlers/config" script, which we describe in more details below. 
You can reuse this callback or provide a URL to your own service/API.

Registering a callback is pretty easy and is done using the NotificationService. In the example herefater, we subscribe the default callback script:
```
var notificationService = deviceServer.getNotificationService();
notificationService.registerCallback({url:"https://yoursubdomain.scriptrapps.io/arm/mbed/api/callback"});
```
Removing (unregistering) the callback is easy as well:
```
notificationService.removeCallback(); 
```

### Configuring notification handlers

Handlers are functions in scripts that know how to handle a given notification type. As a reminder, the following are the notification types that exist:

- notifications: contains resource notifications. Note that the payload is base64-encoded.
- registrations: list of new endpoints that have registered with mbed Device Connector (with resources).
- reg-updates:	list of endpoints that have updated registration.
- de-registrations: list of endpoints that were removed in a controlled manner.
- registrations-expired: list of endpoints that were removed because the registration has expired.
- async-responses:	responses to asynchronous proxy request. Note that the payload is base64-encoded.

You can associate a specific handler to a given notification type using the "/arm/mbed/handlers/config" configuration file. This latter
comes pre-configured to use the "/arm/mbed/handlers/defaulthandler" and its "defaultHandle" and "asyncHandler" functions. More on them further on. You can modify the content of the configuration file to point to your own scripts/functions:

```
var handlers = {  
   "notifications": {     
     path: "/arm/mbed/handlers/mynotificationhandlerscript", // the path to the script that contains the handler of "notifications"
     name: "handleNotifications" // the name of the function to invoke in the script
   },  
  "registrations": {     
     path: "/arm/mbed/handlers/defaulthandler",
     name: "defaultHandle"
   },  
  "reg-updates": {     
     path: "/arm/mbed/handlers/defaulthandler",
     name: "defaultHandle"
   },  
  "de-registrations": {     
     path: "/arm/mbed/handlers/defaulthandler",
     name: "defaultHandle"
   },  
  "registrations-expired":{    
    path: "/arm/mbed/handlers/defaulthandler",
    name: "defaultHandle"
   },  
  "async-responses":{    
    path: "/arm/mbed/handlers/defaulthandler",
    name: "asyncHandler"
   }
};
```
Note that you might want to keep the default handler that is currently associated to notifications of type "async-responses" ("asyncHandler") as it contains a generic implementation that will automatically invoke any handler you might have specified when reading from an endpoint (next paragraph).

### Creating subscriptions

Subcriptions are means to stay permanently informed of the occurrence of events that occur on a given endpoint, instead of explicitly sending read intructions. You can subscribe to an individual resource of an endpoint or ask to subscribe to all event occurrences on some  resources of a list of endpoints. Both cases are illustrated in the below code sample:

```
endpoint.createSubscription("/speed"); // subscribe to the "speed" resource of the current endpoint

var subscriptionService = deviceServer.getSubscriptionService(); // get an instance of SubscriptionService    
var subscriptionList = [ {endpoint-name: "node-001", resource-path: ["/dev"]}, {endpoint-type: "Light", resource-path: ["/sen/*"]} ];
subscriptionService.createAutoSubscriptions(subscriptionList);
```

### Reading from an endpoint

Reading data from an endpoint is mainly about instructing an endpoint to send data whenever possible to a callback. Reading is done uby invoking the "readFromResource" method of an endpoint instance and passing the URL of the target resource (a property of the endpoint, such as the temperature). Optionally you can also pass a handler configuration, i.e. the path to a script that contains a handler function and the name of that latter. 

**Note**: if you are still using the default notifications handlers configuration as described above, any response sent by the device will be caught by the callback script that will convey it to the "asyncHandler" function, which will further invoke any specific handler you might have specified when calling the "readFromResource" method. 

```
// list all resources of our endpoint
var resourceList = endpoint.listResources();

// we instruct our connector to forward any response to the below read request to the "handler1" function 
// defined in the "/arm/mbed/test/notificationHandler" script
var handlerConfig = {
 path: "/arm/mbed/test/notificationHandler",
 name: "handler1"
};

// read from one of the first resource 
var readFromResource = endpoint.readFromResource(resourceList[0], null, handlerConfig);
```

### Writing to a resource

When writing to a resource of an endpoint, you need to specify the type of content you are sending:

```
// list all resources of our endpoint
var resourceList = endpoint.listResources();

// let's say we need to send the target temperature to a thermostat
var writeParams = {
  resourcePath: resourceList[0].uri,
  contentType: "text/plain",
  data: "22"
};

endpoint.writeToResource(writeParams);
```

## Configuring access to ARM mbed Device connector service

- Open the "/arm/mbed/config" file and add a new key to the "apps" variable. This should reflect the name of your application for example.
- Add the "accessKey" property to your application and set it to the value of the access Key you obtained from ARM.

Example:
```
// Configuration of your applications on mbed device connector service
var apps = {
  
  "myiotapp": {
    accessKey: "qry7ayTu4pMlMIVnXaYkeSp6LCtYDQQa4MGVuzLJmaoiwK9mdZgSBmqyy2NB09GPMnse7swpy0bcZkLzei1nAq7YtNuJM6TrJwqw" // access key for app xxx
  }
};		
```

**Note**: there are many more available features in the connector. You can review and test them in the "/arm/mbed/test/test" script.
