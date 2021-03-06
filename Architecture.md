# Architecture

## Introduction

Over the years a lot of different *Demand Side Management* (DSM) approaches have been developed. Unfortunately these DSM approaches are not interoperable. A similar issue can be identified on the appliance level. Appliances provide the flexibility that is being exploited by DSM. To begin with there a lot of different appliances (washing machines, Combined Heat Power Systems, PV panels, fridges, etc.). They also use different protocols for communication (Zigbee, Z-wave, WiFi, PLC, etc.).

All this variety on both the DSM and appliance level presents *Energy Management Systems* (EMS) with a big challenge. This challenge is depicted below.

![](interaction.png)

Nowadays most EMS’es are tightly coupled to a particalar DSM approach. This results in a vendor lock-in for consumers. A switch to another DSM approach/service almost always requires the installation of another EMS (hardware box).

The Energy Flexibility Platform & Interface (EF-Pi) is a runtime environment where on one side smart grid applications can be deployed and on the other side appliances can be connected. The EF-Pi provides interfaces to interact with the environment and connect devices and smart grid apps. Part of the interface definitions are the Control Spaces and Allocations.

EF-Pi aims to create an interoperable platform that is able to connect to a variety of appliances and support a host of DSM approaches. This way the EMS hardware does not need to be changed when a consumers switches from one service to another. At the same time the Energy Flexibility Platform & Interface makes it easier for service providers to introduce new services, since they do not have to provide the EMS hardware to their consumers to go with it.

## High-level design
![](hourglass.png)

The Energy Flexibility Platform & Interface is the connecting part between appliances at the home of the consumers at one side, and the (smart) energy grid on the other side. Appliances of different vendors can implement functionality which can be used to choose when and maybe how to start and use certain energy consuming or energy providing appliances. Developers of EF-Pi are responsible for providing functionality which connects these parts. Protocols are used to communicate with appliances within appliance drivers. Smart Grid applications need a model of the appliance they are paired with, this is the Control Space  and is generated by the appliance driver. The appliance driver uses a self-chosen communication protocol to communicate with the appliance itself. At the physical level, this could be Zigbee, Z-Wave, PLC, WIFI, Ethernet, propriety, etc. It is provided by the manufacturer and allows for mix & matching of protocols.

With the current Control Space of a device, Smart Grid Applications can determine the usage profile of the devices, i.e. when a device should start or stop. The application then makes an Allocation and if there is any energy flexibility available, it should be listed within Allocation. The appliance driver receives the Allocation and based on it can make a decision what is the optimal way to control the device. In this decision the user preferences are taken into account.

It is not the device that is modelled but rather its energy flexibility. From this a real device can be connected and used to perform the communication between the derived class and its device. In our experience four models are sufficient enough to cover all device types. These classes are called Control Spaces and are described in Table 1 below.

Control Space | Description | Examples
--- | --- | ---
Uncontrollable | Has no flexibility, is measureable and may provide forecast. | Photo voltaic, Wind Turbine, TV, indoor lighting, etc.
Time Shiftable | Operation can be shifted in time, has a deadline.|  Washing machine, Dishwasher, etc.
Buffer/Storage | Flexible in operation for either production or consumption and operation is bound by a buffer. | Freezer, Heat Pump, CHP, Batteries, EV, etc.
Unconstrained | Flexible in operation for production. The operation is not bound by a buffer. | Gas Generators, Diesel Generator, etc.

** Table 1: Description of the four different Control Space categories **

In essence, a control space is a way to put the information that is contained within a device into a generic structure, such that energy apps are able to understand that device from a generic energy model.

Where Control Spaces form an abstract representation of a device, Allocations are used to express what a device is requested to do. For each Control Space, there is also one Allocation type.

Control Spaces and Allocations are further discussed in the next chapter.


The App Store and Remote Management Interface are still under development and thus not discussed in this tutorial.

For you as developer it is important to know how the infrastructure works:

* A Resource Manager receives a State from a Resource Driver.
* This State is obtained by the Resource Driver using information sent by or polled from the appliance(s).
* The State is converted by the Resource Manager into a Control Space.
* The Resource Manager sends this Control Space to the Energy App. A Control Space defines the freedom in which the appliance can be started, and how much energy is consumed or produced when started.
* The Energy App then provides an Allocation, which states when each appliance should be started, which is a time within the Control Space.
* A Resource Manager creates Control Parameters based on this, which define when the appliance should start. The Control Parameters are sent to the Resource Driver.
* The Resource Driver actually starts the appliance.

A dashboard can show controls and information about the current state of your appliances in the form of Widgets. Each Widget shows information for an appliance. It is possible to have multiple Widgets per appliance. When run the Dashboard is currently shown at [http://localhost:8080](http://localhost:8080).

![](dashboard.png)

**Figure: Example of the dashboard. The main page of the dashboard contains widgets.**

## Components and widgets
New functionality can be installed at run-time it the form of *Apps*. An App can for example contain drivers or smart grid applications. Apps consist of one or more OSGi components. We will discuss OSGi components later on in more detail. What you need to remember for now is that the a component can have multiple instances (just like a Java class can have multiple instances called objects) and that they can be configured.

Let's consider an example configuration to demonstrate how these components can interact. In this example the EF-Pi is used to control a Miele refrigerator and a Miele dishwasher. We have also added the PV panel simulation to make it a bit more interesting. The PowerMatcher has been added to control these three appliances.

[![](component_overview.png)](https://raw.githubusercontent.com/wiki/flexiblepower/EF-Pi-core/component_overview.png)

**Figure: Example EF-Pi configuration. Green blocks represent OSGi components. Click [here](https://raw.githubusercontent.com/wiki/flexiblepower/EF-Pi-core/component_overview.png) for the full version.**

The image above shows the example configuration. All the green blocks represent OSGi components. The first line in the green blocks state the name of the component. Under the name is the configuration of the component.

An important part is the *Energy Flexibility Interface* (EFI), for which we reserved the next chapter. This is the interface between an Energy App, like the PowerMatcher, and Resources, which makes the flexibility possible.

Below the *Energy Flexibility Interface* (EFI) we can see all the device specific components. We see that the *Miele App* provides two types of Resource Managers, two types of Resource Drivers and a Protocol Driver. The PV Panel Simulation does not have a Protocol Driver, since the simulation does not have to communicate with a device. The Pv Pavel Simulation is connected with a generic Resource Manager for uncontrolled appliances. Above the Resource Abstract Layer we can see the PowerMatcher App, which communicates with the three Resource Managers.

Communication wiring is done using a messaging abstraction, implemented in the `org.flexiblepower.messaging` package. This is implemented using annotations:
Any component, like an implementation of a ResourceDriver or ResourceManager, can have an annotation `@Ports` which can contain multiple `@Port` definitions, which have a `name`, `cardinality`, `sends`, and `accepts` parameter; the latter specify which objects are accepted by this object. They then must implement a MessageHandler that accepts the object(s). A cardinality can be given to specify wheter it support a single or multiple connections. Exact details are specified in the tutorial that follows later. Note that the usage of the `ResourceId` property is depricated since the 14.10 release.

You will also notice that there are several widgets. Every component in the system can provide a widget which is shown in the main page of the dashboard.