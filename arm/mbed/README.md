### About The connector to ARM mbed Device Connector :)
ARM provides the [ARM mbed™ Device Connector service](https://connector.mbed.com/), a cloud service that allows connecting IoT devices
to the cloud, notably to ARM's [mbed IoT device platform](https://www.mbed.com/en/) cloud platform. Using ARM's device connector, devices
operated by [ARM's mbed OS](https://www.mbed.com/en/platform/mbed-os/) become at the reach of REST APIs calls using the Device Connector
service's APIs. The pupose of our connector is to expose the latter to scriptr.io developers as first class JavaScript objects 
that can be easily invoked from within their scripts.

### Prerequisites
- You need to own an ARM mbed Device connector account 
- You need to have [created an application](https://connector.mbed.com/#accesskeys) on ARM's platform in order to obtain an access key
- You need to connect devices to your mbed Device connector account (you can real devices operated by ARM mbed OS or simulators)

### How to use

The connector provides different classes that are abstractions of the different APIs exposed by ARM's mbed Device Connector 
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
to pass an instance of the Client class, which encapsulate the technicalities on how to invoke ARM's APIs, 
- Or by asking the DirectoryService to give you such a reference. 

In the below example, we illustrate that second option. We first start by creating an instance of DeviceServer, then obtain from the latter
an instance of DirectoryService that we use to obtain your endpoint, passing its url to identify it:

```
var deviceServer = new deviceserverModule.DeviceServer({accessKey:"YOUR_ARM_ACCESS_KEY"});
var directoryService = deviceServer.getDirectoryService();
var endpoint = directoryService.getEndpoint("/actuator/0/speed");
```
**Notes:** 
- If you specified your access key in the confiuration file (see below), you can just invoke the constructor of DeviceServer without
passing any parameter,
- You can also use the directory service to list the available endpoints then pick an endpoint from the list.

### Adding adding a notification callback

Reading from a device or obtaining automatic notifications from devices is an asynchronous activity that requires you to register
a callback. Callbacks should be URL of scripts. 

The connector provides you with a default callback script "/arm/mbed/api/callback" that knows how to forwards the notification types 
to their specific handlers, based on the content of the "/arm/mbed/handlers/config" script, which we describe in more details below. 
You can reuse this callback or provide a URL to your own service/API.

Registering a callback is pretty easy and is done using the NotificationService

### Reading from an endpoint




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
