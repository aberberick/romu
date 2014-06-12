romu
====

Android software for haptic navigation.

Overview
The Romu app is trying to combine the hardware power of the smartphone, the Google API, and an easy-to-use interface for users to view the surrounding map, find out where they are and set their destination to go. 
From a software engineering perspective, it consists of three main parts, including the User Interface, the Romu Service, which is a service that is in charge of background processing of all the services that are 
related navigation, and lastly, bluetooth service, which is used by Romu Service to utilize bluetooth low energy technology to communicate with another important part of Romu, the wearable.
The Romu software modularizes components to handle varying tasks with varying lifecycles. The major division in software is the division between firmware and android software.


Romu UI

The UI is handled in the first component, the Romu Activity. This component has a temporary lifecycle and is only functional as long as it is in the view of the user. This component provides the controls necessary 
to allow the user to specify the desire to pair with a given device, to search for a given address, and to stop/start the navigation. It is connected to the Google Maps API and handles the drawing of different map fragments,
routes, and the attachment of the “in navigation fragment” (i.e. the piece of the UI with the controls for play, pause, stop, etc.). Romu Activity uses three services provided by Google: Google Place Service, Google Directional
Service and Google Map service. Google Place Service is used when user input an incomplete address. The incomplete address will be sent to Google, then a list of potential addresses will be returned. Google Directional Service 
is used when user has chosen a destination to go. The origin and destination will be sent to Google via http request, then a route will be returned in JSON format. By parsing the response, we could get the route information to
start from origin and get the destination, what's more, we could use the information contained to draw the route on Map to visually show it to user and gives user navigation instruction on the screen. Google Map Service is used 
to draw the actual map. When the request is made, a corresponding request will be sent to Google.

Communication Between UI and Underlying Architecture

Romu UI interacts with Romu Service in a way that mostly modulizes the two. When user decides to start navigating by clicking the start button, start navigation will be sent to Romu service, nothing
￼else is needed to be done. When the navigation status is changed, for instance, the user has reached a end of segment, romu service will notify UI with Inter-Process Communication. And UI will update
corresponding UI elements and gives user feedback about the status changes.

RomuService

The next component, the Romu Service, has a lifecycle beyond the view of the application. It can only be stopped if the user explicitly quits the app or turns off the phone. The service communicates with the Bluetooth component 
while listening for location changes. It is the service that handles all navigational computation. It receives input for starting and stopping navigation or bluetooth connection from the UI but it can service beyond the life of the UI.
This service broadcasts updates to the UI to notify the user if the network connection, bluetooth connection, or navigation state has changed. The service actively communicates with google play services while a network connection is present 
to use Google’s provided location accuracy algorithms, but can function with or without a network connection. It is responsible to communicate with the wearable hardware, which is also abstracted into another service call BluetoothLE Service. 
The communication with the Underlying Bluetooth Service is abstracted into another module called BluetoothLE(Bluetooth Low Energy) service.When one location update is ready to be sent to the firmware, the program just calls sendCommand to delegate
actual communication to BluetoothLE Service. If the Romu Service gets a notifcation from the BluetoothLE service that the user has tapped on the wearable device, then the RomuService pauses or resumes navigation and updates the UI Romu Activity, accordingly.


Navigation Handling Within RomuService

Upon start of the Romu Application, a location client is established and location requests are made. A location client uses whatever means are available to get a specific reading on a user’s location. If internet is available, this client uses network providers, 
wifi, and GPS in combination with Google’s location algorithm’s to hone in on a user’s location while minimizing battery consumption. It generally takes a few minutes to gain truly accurate location readings, so location updates are made and tracked without Bluetooth
response until the user specifically starts navigation. When the user starts navigation with a specified route, the system “listens” to the navigation updates by parsing the route, and creating Bluetooth commands to be converted into haptic signals.
The system by default requests location updates every 5 seconds. If a location is different than the previous location, the system attempts to write the location update to Bluetooth. There are two algorithms that determine the frequency of Bluetooth writing, and updates cannot be written more
￼frequently than once every 500ms. This prevents a software serial overload on the firmware. The first algorithm watching to see if the user is lost takes priority over all others. If a user is lost (i.e. heading in a direction that is more than 50 degrees away from where they should be heading),
the system sends updates with frequency scaled to the degree of “lostness,” maximized at 180 degrees from the heading to the destination. If the user is not lost, then system sends updates that increase in frequency as the user approaches a given waypoint. This frequency is determined through a tier-based system. 
The tiers bucket the frequency of updates based on the proximity to the given location with buckets including 10m, 20m, 50m, 100m, etc. If a user resumes after pausing or starts navigation, the system uses the last known location of the user to begin navigation. If the frequency of updates increases beyond the default
frequency of updates (every 5s) the system uses the last known location from the location client.

A route consists of a series of linear segments and waypoints. When a user starts navigation, the Romu Service determines the first waypoint and the final destination. It then enters into the main function that converts the user’s current location into an instruction on how to get to the next waypoint. The system first 
determines the quality of the location reading by comparing it to previously known locations using parameters such as radius of accuracy, location provider, and time of location reading. Once the best location is determined, the system generates a heading angle between the given location the next waypoint. The system 
then attempts to find the user’s current orientation using a velocity vector. If this vector cannot be determined (i.e. the user is stationary), the system sends the Romu wearable the heading angle and a request to use the compass on the wearable for orientation. If the system is able to find the velocity vector, the android app
combines this orientation angle with the heading to destination and attempts to send it via Bluetooth to be converted into a haptic signal by the wearable. If a user is within a reasonable radius of accuracy to a given waypoint, the system updates to the next waypoint in the route. Reasonable accuracy depends on the user’s 
proximity to a given waypoint. With a default minimum of 35m, the system combines the estimated radius accuracy of a given location reading, with a tiered system for the radius of waypoint updates. For example, if the user is within 50 m of the waypoint the radius is set to 35m. If the user is 100m way, the radius is set to 75m, etc.
The allows the system to adapt to varying degrees of accuracy with fluctuating location providers. Once the user reaches the final destination, navigation stops.
If a user pauses navigation, the system keeps track of where the user last was and upon resume, guides the user to the last known waypoint.



BluetoothLEService

The BluetoothLE service is the interface between the Android software and the Romu firmware. It handles the bluetooth connection and the RomuService writes navigation updates to the device through this service. It’s lifecycle is tied to the romuservice. It uses the Bluetooth Low Engergy API provided by Android. When it is started, it will automatically try to connect the firmware for two ￼seconsd. Once connected, the disconnection and reconnection is handled. The interface between Romu Service and BluetoothLE is similiar with Romu UI and Romu Service.
In addition to writing location updates to the wearable device, the BluetoothLE service also listens for feedback from the device in the form of “tap-to-pause” and “tap-to-start.”



















