---
title: "jSPINE – A Java library implementing EEBUS SPINE"
date: 2022-01-26T11:00:07+01:00
author: "Martin Eberle"
authorAvatar: "img/ada.jpg"
tags: [FraunhoferISE, EEBUS]
categories: []
image: "img/project-jspine/pexels-mike-110844.jpg"
draft: false
---

In a world with rising energy demands, power limitations by grid infrastructures
and fluctuating energy supply of renewable energy sources, the communication
between devices which consume electrical power is a key in distributing
electricity efficiently. The EEBus Initiative e.V. provides a free and open,
standardized language aiming to solve the barriers in communication
between devices of different manufacturers.

<!--more-->

*This project is the outcome of a cooperation between Fraunhofer ISE and the Algorithms & Data Structures Chair of the University of Freiburg.*

## Content

1. [What is EEBUS?](#what-is-eebus)
2. [What is SPINE?](#what-is-spine)
3. [What is jSPINE?](#what-is-jspine)
4. [The SPINE data model](#the-spine-data-model)
    - [The _NodeManagement Feature_](#the-_nodemanagement-feature_)
5. [The SPINE datagram](#the-spine-datagram)
6. [Message types](#message-types)
    - [Read messages](#read-messages)
    - [Reply messages](#reply-messages)
    - [Notify messages](#notify-messages)
    - [Write messages](#write-messages)
    - [Call messages](#call-messages)
    - [Result messages](#result-messages)
7. [DetailedDiscovery](#detaileddiscovery)
8. [UseCaseDiscovery](#usecasediscovery)
9. [Subscription](#subscription)
10. [Binding](#binding)
11. [Restricted Function Exchange](#restricted-function-exchange)
    - [Data Selector Filter](#data-selector-filter)
    - [Element Filter](#element-filter)
    - [Partial read](#partial-read)
    - [Partial write / notify](#partial-write--notify)
12. [Communication Modes](#communication-modes)
    - [Simple Communication Mode](#simple-communication-mode)
    - [Enhanced Communication Mode](#enhanced-communication-mode)
13. [Example Use Case](#example-evse-commissioning-and-configuration-use-case)
    - [Use Case Description](#use-case-description)
    - [EVSE](#evse)
        - [Functions](#functions)
        - [SPINE Device creation](#spine-_device_-creation)
    - [Energy Manager](#energy-manager)
        - [Client Functionality](#client-functionality)
14. [Future Work](#future-work)

## What is EEBUS?

EEBUS is an open, license free, standardized _language for energy_ enabling
devices in the energy sector to communicate with each other, regardless of the
manufacturer and technology, developed by the EEBus Initiative e.V. [^1]

The protocol is divided into three distinct layers: the
communication layer (SHIP [^2]) at the bottom, the function layer with Use Cases
at the top and the information layer (SPINE [^3]) in the middle. Implementing a
library which facilitates the use of the latter is the work of this project and
explained in more detail in the following.

## What is SPINE?

We can divide SPINE into two parts: SPINE Resources and the SPINE Protocol. The
SPINE Resources are described very well on the initiatives' website:

> SPINE is a toolbox of modular elements to enable the realization of any use
> case today and in the future. The toolbox contains a collection of data
> classes that can be exchanged on various technological platforms,
> communication and transmission channels. [^4]

The SPINE Protocol sets rules for the format of SPINE messages (a.k.a. SPINE
datagram) in which SPINE Resources can be used. As the usage of SPINE Resources
is dependent on the Use Case only the Data Classes can be provided by jSPINE
[^5] while most details of the SPINE Protocol can be abstracted and partly
automated.

A SPINE message exists usually in the XML format. Theoretically it can be in any
other format which allows the representation of the data classes, e.g. JSON. In
this article XML is used.

## What is jSPINE?

jSPINE is a software library written in Java providing the developer with all
_Data Classes_ defined in the SPINE specification and an easy to use API
abstracting operations and automating repeating processes of the SPINE protocol.
The API with its documentation can be accessed freely on the [OpenMUC Website](
https://openmuc.org/eebus/jspine/javadoc). With jSPINE the developer only needs
to care about the commands sent over SPINE as the payload but not metadata in
the header of a message.

The software architecture of jSPINE allows it to be used over any communication
protocol (via implementation of the abstract `Communication` class).

jSPINE provides the ability to easily build any SPINE data model and exchange
messages in an abstract manner. The following SPINE features can be used in
jSPINE fully automated (i.e. without developer interaction except of enabling
the feature in some cases):

- DetailedDiscovery (enabling required)
- Transparent usage of Devices over the Enhanced Communication Mode (enabling
  required)
- Acknowledgment delivery (enabled by default, disabling possible)
- Entity and Feature address assignment
- Notification of runtime changes (Entity and Feature additions / modifications
  / deletions)
- Subscription and Binding management (automatic registrations of active
  subscriptions and bindings, release on Feature deletions)
- Connection caching (reusing available open connections)

In general all _NodeManagement_ tasks are fully automated except for the
UseCaseDiscovery which requires a little more interaction by the developer.



## The SPINE data model

In SPINE data is contained in _Functions_. A _Function_ can be read returning
the contained data and written to, modifying or deleting existing data or adding
new data. A collection of _Functions_ is represented by a _Data Class_. A _Data
Class_ (partially or complete) is then provided to other SPINE _Devices_ in the
form of a _Feature_ with _Server_ role. _Features_ in turn can be grouped in
_Entities_ while an _Entity_ can contain also one or more _Entities_. This forms
a tree structure for each _Device_ with the _Device_ representing the root node
and a collection of _Entities_ as its children.

A _Device_ could look like the following illustration. The left-most _Entity_
and _Feature_ are always present with the given _Types_, explained in more
detail in the next chapter.

![Device tree example](/img/project-jspine/Device_tree_example.jpg)

For example the _Data Class_ `ActuatorSwitch` groups two _Functions_
`actuatorSwitchData` and `actuatorSwitchDescriptionData`. These could look like
the following for a simple lightbulb:

```xml
<actuatorSwitchData>
  <function>on</function>
</actuatorSwitchData>
<actuatorSwitchDescriptionData>
  <label>Smart Lightbulb</label>
  <description>SPINE enabled Smart Lightbulb</description>
</actuatorSwitchDescriptionData>
```

Each _Device_, _Entity_ and _Feature_ has a uniquely identifying address
containing the address of its parent. The _Device_ address is represented by a
unique string (containing the IANA PEN [^6]). Each _Entity_ has an ID
represented by an unsigned long. Its value is unique for all other _Entities_
at the same level in the tree with the same parent. Its address is a list of
all IDs of parent _Entites_ with its own at last. A _Feature_ address consists
of the the parent _Entity_ address and its own ID (also of type unsigned long
and unique inside of its parent _Entity_). An example address of a _Feature_
could look like this:

```xml
<device>d:_i:12345_ExampleDevice-0</device>
<entity>1</entity>
<entity>0</entity>
<feature>0</entity>
```

jSPINE assigns each _Entity_ and _Feature_ always the lowest ID possible
automatically.

### The _NodeManagement Feature_

A SPINE _Device_ always contains a _Feature_ with the type _NodeManagement_
providing information about the _Device_ and all its _Entities_ and _Features_.
The parent _Entity_ of the _NodeManagement Feature_ has the type
_DeviceInformation_. It always has the following address (ommitting the device
address):

```xml
<entity>0</entity>
<feature>0</entity>
```

The _NodeManagement Feature_ provides _functions_ which provide data about
_Subscriptions_, _Bindings_, _Use Cases_ and the _Device_ itself (see chapter
[DetailedDiscovery](#detaileddiscovery)).


## The SPINE datagram

The SPINE datagram is divided into Header and Payload.

The Header specifies metadata (e.g. routing details, timestamps, routing
details) and the [message type](#message-types). A header could look
like this:

```xml
<header>
  <specificationVersion>1.1.1</specificationVersion>
  <addressSource>
    <device>d:_i:12345_ExampleDevice-C</device>
    <entity>0</entity>
    <feature>0</feature>
  </addressSource>
  <addressDestination>
    <device>d:_i:12345_ExampleDevice-S</device>
    <entity>0</entity>
    <feature>0</feature>
  </addressDestination>
  <msgCounter>104</msgCounter>
  <cmdClassifier>read</cmdClassifier>
  <timestamp>2021-10-15T14:45:00.0Z</timestamp>
</header>
```

jSPINE sets the _Header_ automatically without any manual intervention of the
developer.

The payload contains a `cmd` tag which contains the Data Classes defined in
SPINE dependent on the message type. The `cmd` tag exists for the purpose of
later extensions of the protocol (so multiple commands could be sent in one
message).

## Message types

Five different messsage types are possible:
- read  
- reply
- notify
- write
- call
- result

### Read messages

A read message origins from a Feature behaving like a Client (i.e. with Client
or Special role) and requests data from a Feature behaving like a Server (i.e.
with Server or Special role).

### Reply messages

A reply message is the answer to a read message containing the requested data.

### Notify messages

A notify message contains updated data and origins from a Server Feature. The
recipient is a Client Feature subscribed to the sending Feature.

### Write messages

A write message origins from a Client Feature and requests a data change at the
recieving Server Feature. In the case of new data or modifications the message
contains the new / changed data.

### Call messages

Call messages can also contain data, but are used when not a simple data change
is requested. They usually trigger some procedure on the requested feature. E.g.
subscriptions and bindings are requested with a call message containing the
subscription / binding request.

### Result messages

Result messages are acknowledgements and used in the case of errors or if data
was successfully transmitted. Result messages are not transmitted for other
result messages or reply messages. Read messages are only acknowledged on
failure as a reply message already acknowledges the successful recieval of the
read request.

## DetailedDiscovery

The _DetailedDiscovery_ is a mechanism for a SPINE _Device_ to provide its
structure and information to other SPINE _Devices_. Essentially it is a SPINE
_Function_ contained in the _NodeManagement Feature_ providing information about
all its _Entities_ and _Features_. The _DetailedDiscovery_ of a minimal SPINE
_Device_ would look like this:

```xml
<nodeManagementDetailedDiscoveryData>
  <specificationVersionList>
    <specificationVersion>1.1.1</specificationVersion>
  </specificationVersionList>
  <deviceInformation>
    <description>
      <deviceAddress>
        <device>d:_i:12345_ExampleDevice-S</device>
      </deviceAddress>
    </description>
  </deviceInformation>
  <entityInformation>
    <description>
      <entityAddress>
        <entity>0</entity>
      </entityAddress>
      <entityType>DeviceInformation</entityType>
    </description>
  </entityInformation>
  <featureInformation>
    <description>
      <featureAddress>
        <entity>0</entity>
        <feature>0</feature>
      </featureAddress>
      <featureType>NodeManagement</featureType>
      <role>special</role>
      <supportedFunction>
        <function>nodeManagementDetailedDiscoveryData</function>
        <supportedOperations>
          <read/>
        </supportedOperations>
      </supportedFunction>
      <supportedFunction>
        <function>nodeManagementUseCaseData</function>
        <supportedOperations>
          <read/>
        </supportedOperations>
      </supportedFunction>
      <supportedFunction>
        <function>nodeManagementSubscriptionData</function>
        <supportedOperations>
          <read/>
        </supportedOperations>
      </supportedFunction>
      <supportedFunction>
        <function>nodeManagementBindingData</function>
        <supportedOperations>
          <read/>
        </supportedOperations>
      </supportedFunction>
    </description>
  </featureInformation>
</nodeManagementDetailedDiscoveryData>
```

As visible in the _DetailedDiscovery Function_ every _Function_ is contained.
Another SPINE _Device_ can _read_ this _Function_ to analyze if it
contains usable information and _subscribe_ to get notified about any changes in
the _Device_ structure (_Feature_/_Entity_ deletions/additions/modifications).
jSPINE offers to _read_ the _DetailedDiscovery Function_ automatically as soon
as another _Device_ is detected (either by the communication layer, e.g. by
SHIP, or by manual connection).

## UseCaseDiscovery

Due to the ability of a _Device_ to change its structure at runtime (by
deleting/adding or modifiying _Entites_ and/or _Features_) the
_DetailedDiscovery_ alone is not sufficient to derive supported _UseCases_ from
a _Device_. To indicate a certain _UseCase_ is implemented on a _Device_ the
information about the _UseCase_ is saved in the `nodeManagementUseCaseData`
_Function_. _UseCases_ are provided in a standardized manner by the EEBus
Initiative e.V.

If a _Device_ wants to find a matching communication partner for its own
_UseCase_ it can _read_ this _Function_ and determine the name, version and
actor name of the _UseCase_. Afterwards it knows where the required data of the
_UseCase_ can be obtained or sent to (with information of the
_DetailedDiscovery_).

## Subscription

To prevent the necessity of polling information, i.e. sending _read_ messages in
a periodic manner, SPINE supports the concept of _Subscriptions_. A _Client
Feature_ can _subscribe_ to a _Server Feature_ which then sends any changes in
any of its _Functions_ to the _subscribed Client Feature_ with a _notify_
message.

All subscriptions are maintained by the _NodeManagement Feature_ providing the
information in the `nodeManagementSubscriptionData` _Function_.

jSPINE supports a permission concept, where the developer can freely choose when
to allow _subscriptions_ from a _Client Feature_. This is for example useful to
limit the amount of _subscribers_ to a single _Feature_.

## Binding

The _Binding_ concept of SPINE is used to control the permission of _Client
Features_ to change data in a _Server Feature_, i.e. with a _write_ command.

A _Server Feature_ can require the previous _Binding_ of a _Client Feature_ to
accept _write_ messages from it. Similar to _Subscriptions_ all _Bindings_ are
managed by the _NodeManagement Feature_ providing the data in the
`nodeManagementBindingData` _Function_.

jSPINE supports the same permission concept for _Bindings_ as for
_Subscriptions_.

## Restricted Function Exchange

Often only part of the data contained in a _Function_ is needed or only a part
of it should be modified. SPINE provides the (optional) ability to perform a
_partial read_ command and a _restricted write_ command for that purpose.

### Data Selector Filter

A lot of data is contained in lists. To be able to reference a specific list
item, an appropriate _Data Selector_ can be set. For example to filter the
_DetailedDiscovery Function_ for _Features_ of type _NodeManagement_ the _read_
command would look like the following:

```xml
<cmd>
  <function>nodeManagementDetailedDiscoveryData</function>
  <filter>
    <cmdControl>
      <partial/>
    </cmdControl>
    <nodeManagementDetailedDiscoveryDataSelectors>
      <featureInformation>
        <featureType>NodeManagement</featureType>
      </featureInformation>
    </nodeManagementDetailedDiscoveryDataSelectors>
  </filter>
  <nodeManagementDetailedDiscoveryData/>
</cmd>
```

### Element Filter

In some scenarios not all information contained in a _Function_ is requested (or
to be changed). To only get the device address of the _DetailedDiscovery
Function_ a _partial read_ command would look like this:

```xml
<cmd>
  <function>nodeManagementDetailedDiscoveryData</function>
  <filter>
    <cmdControl>
      <partial/>
    </cmdControl>
    <nodeManagementDetailedDiscoveryDataElements>
      <deviceInformation>
        <description>
          <deviceAddress/>
        </description>
      </deviceInformation>
    </nodeManagementDetailedDiscoveryDataElements>
  </filter>
  <nodeManagementDetailedDiscoveryData/>
</cmd>
```

### Partial read

Partially reading a _Function_ requires the presence of a `filter`, the
`function` tag containing the name of the _Function_ on which filtering shall be
applied and the usual _read command_. The `filter` contains the following
`cmdControl` tag:

```xml
<cmdControl>
  <partial/>
</cmdControl>
```

The data is then filtered by the given _Data Selectors_ and / or _Element
Filters_, with the _Data Selectors_ being applied first.

### Partial write / notify

The `filter` elements of _write_ messages and _notify_ messages are identical
except for a small change in definition: the `filter` in _notify_ messages means
the data **was changed** while the `filter` in _write_ messages means the data
is **requested to change**. For simplicity this chapter uses _notify_ messages,
but can be applied to _write_ messages as well.

If data inside of a _Function_ was deleted the `cmdControl` tag contains the tag
`<delete/>` instead of `<partial/>`. _Data Selectors_ then specify which list
entry was deleted, while _Element Filters_ specify which element was deleted.
The following XML is an example:

```xml
<cmd>
  <function>electricalConnectionStateListData</function>
  <filter>
    <cmdControl>
      <delete/>
    </cmdControl>
    <electricalConnectionStateListDataSelectors>
      <electricalConnectionId>4</electricalConnectionId>
    </electricalConnectionStateListDataSelectors>
  </filter>
  <electricalConnectionStateListData/>
</cmd>
```

If data inside of a _Function_ was added or modified, the changed data is
contained in the _Function_ itself. This could look like the following:

```xml
<cmd>
  <function>electricalConnectionStateListData</function>
  <filter>
    <cmdControl>
      <partial/>
    </cmdControl>
  </filter>
  <electricalConnectionStateListData>
    <electricalConnectionId>5</electricalConnectionId>
    <timestamp>2021-11-23T17:34:54.0Z</timestamp>
    <currentEnergyMode>consume</currentEnergyMode>
    <consumptionTime>PT4H31M22S</consumptionTime>
  </electricalConnectionStateListData>
</cmd>
```

The _delete_ and _partial_ filter can be present at the same time with the
_delete_ filter always preceding the _partial_ filter both in the XML
representation and in the filter execution.

## Communication Modes

### Simple Communication Mode

The _Simple Communication Mode_ is the default communication mode in jSPINE.
Communication is always directly between two SPINE _Devices_. A message which is
sent to a SPINE _Device Address_ on a specific communication address (e.g. IP)
expects the SPINE _Device_ to be reachable on this communication address.

### Enhanced Communication Mode

With the _Enhanced Communication Mode_ it is possible to send SPINE messages to
SPINE _Devices_ via intermediate SPINE _Devices_. This makes it possible for
SPINE _Gateways_ to connect different communication protocols on which SPINE
messages are exchanged or for SPINE _Routers_ to route messages between
different networks.

A SPINE _Device_ which should forward messages offers the `destinationListData`
_Function_ on its _NodeManagement Feature_. It contains a list with all SPINE
_Devices_ not accessible directly from the calling _Feature_. A _Device_ can
therefor _read_ the _Function_ and send messages, which shall be received by the
distant _Device_, to the intermediate _Device_. All _Devices_ must have there
_networkFeatureSet_ set to a different value than `simple` (`smart`, `router` or
`gateway`).

jSPINE supports the automatic discovery of _Devices_ via an intermediate
_Device_.

## Example: EVSE Commissioning and Configuration Use Case

### Use Case Description

The purpose of the Use Case _EVSE Commissioning and Configuration_ (_EVSECC_) is
to specify the initial setup process between an electric vehicle supply
equipment (EVSE) and an energy manager. In the setup process the EVSE sends its
manufacturer information to the energy manager and can notify the energy manager
about possible operation failures.[^8]

### EVSE

An EVSE is a device (e.g. a charging station) connecting an electric vehicle
(EV) to a power grid. In SPINE an EVSE is represented by a SPINE _Entity_ with
type `EVSE`.

The EVSE provides data to communication partners in the form of two SPINE
_functions_: `deviceClassificationManufacturerData` and
`deviceDiagnosisStateData`. As the EVSE provides data it can be seen as a SPINE
Server.

The SPINE data model of a SPINE _Device_ representing an EVSE looks like this
(excluding the _NodeManagement Feature_):

![Device tree EVSE](/img/project-jspine/Device_tree_evse.jpg)

#### Functions

With jSPINE the developer extends the abstract
class `FeatureFunction`[^7] to represent each of these `functions`.

As this Use Case does not use _write messages_ only the `read` method needs to
be implemented and changes in data must be notified to any subscribers. Every
time a client sends a _read message_ to the EVSE jSPINE executes the `read`
method and replies with the returned `cmd`.

The `deviceDiagnosisStateData` _function_ could be represented as following in
jSPINE (omitting import statements):

```java
public class DeviceDiagnosisStateDataFunction extends FeatureFunction {
    private DeviceDiagnosisStateDataType stateData;  // holds the data

    public DeviceDiagnosisStateDataFunction() {
        super(FunctionEnumType.DEVICE_DIAGNOSIS_STATE_DATA.value());

        // marks function as readable in DetailedDiscovery, but not as partially readable
        setReadable(true, false);
    }

    // arguments can be ignored here
    // filter would be used for partial reads
    @Override
    public CmdType read(FilterType filter, FeatureAddressType sourceAddress) {
        CmdType replyCmd = new CmdType();
        replyCmd.setDeviceDiagnosisStateData(stateData);
        return replyCmd;
        /* replyCmd represents this XML (with example data):
        <cmd>
          <deviceDiagnosisStateData>
            <operatingState>failure</operatingState>
            <lastErrorCode>EV exploded</lastErrorCode>
          </deviceDiagnosisStateData>
        </cmd>
        */
    }

    @Override
    public SpineAcknowledgment write(CmdType cmd, FeatureAddressType sourceAddress) {
        throw new UnsupportedOperationException();  // function is not writable
    }

    @Override
    public SpineAcknowledgment call(FeatureAddressType sourceAddress) {
        throw new UnsupportedOperationException();  // function is not callable
    }

    // simplified; this would update the stateData even if no actual change occured
    public void updateStateData(DeviceDiagnosisStateDataType stateData) {
        this.stateData = stateData;
        // notifies any subscribers about the change
        feature.notifySubscribers(FunctionEnumType.DEVICE_DIAGNOSIS_STATE_DATA, null);
    }
}
```

The `deviceClassificationManufacturerData` _function_ would look very similar
with the difference of holding another data type and is omitted for the sake of
simplicity.

Note that the `UnsupportedOperationException` in the above code example is
actually never thrown as jSPINE would deny _write_ and _call_ requests before
executing the related methods. The default IDE behaviour of returning `null`
when overriding these methods is fine, but throwing the exception makes the
behaviour more obvious and helps readability.

An instance of the _FeatureFunction_ can then be added to a SPINE _Feature_ when
building the _Feature_ with jSPINE.

#### SPINE _Device_ creation

To build a SPINE _Device_ with jSPINE the `DeviceBuilder` class [^9] is used.
The _Device_ is firstly prepared by setting required information like a
communication implementation – e.g. ShipCommunication using jSHIP [^10], an ID
(i.e. the SPINE _Device_ address) and the _Device_ type. Afterwards the UseCase
specific data is set (UseCase metadata, UseCase specific data model).

```java
DeviceBuilder db = Device.getBuilder();

// required information
Communication comm = new ShipCommunication(...);  // constructor arguments omitted for simplicity
db.setCommunication(comm);  // implementation is protocol dependent, e.g. jSHIP
db.setId("d:_n:EVSECC-Demo_EVSE");  // should be unique and include the IANA PEN
db.setType(DeviceTypeEnumType.GENERIC);

// EVSECC UseCase specific data

// UseCase interface implementation to make the UseCase discoverable via the UseCaseDiscovery
// Details omitted for simplicity (provides simple data like the UseCase name)
UseCase evsecc = new UseCase(...);
db.addUseCase(evsecc);

// bindingAllowed override omitted here as no bindings are used
FeaturePermission subsAllowed = new FeaturePermission() {
    @Override
    boolean subscriptionAllowed(SubscriptionRequest request) {
        return true;
    }
};

// following could also be put into UseCase#setup(DeviceBuilder)
EntityBuilder eb = db.addEntity().setType(EntityTypeEnumType.EVSE);

FeatureBuilder deviceClassificationFb = eb.addFeature();
deviceClassificationFb.setRole(RoleType.SERVER);  // the Feature provides data
deviceClassificationFb.setType(FeatureTypeEnumType.DEVICE_CLASSIFICATION);
deviceClassificationFb.addFunction(new DeviceClassificationManufacturerDataFunction());
deviceClassificationFb.setFeaturePermission(subsAllowed);
deviceClassificationFb.apply();

FeatureBuilder deviceDiagnosisFb = eb.addFeature();
deviceDiagnosisFb.setRole(RoleType.SERVER);
deviceDiagnosisFb.setType(FeatureTypeEnumType.DEVICE_DIAGNOSIS);
deviceDiagnosisFb.addFunction(new DeviceDiagnosisStateDataFunction());
deviceDiagnosisFb.setFeaturePermission(subsAllowed);
deviceDiagnosisFb.apply();

// Finalize build
eb.applyToDevice();

db.build();  // Device connects and is reachable now
```

After `db.build();` was executed the SPINE _Device_ is started and listens for
messages on the communication protocol. Requests are handled by jSPINE and the
SPINE _Device_ can be discovered via the _DetailedDiscovery_ on the
_NodeManagement Feature_.

### Energy Manager

An energy manager in a smart grid has the purpose of intelligently controlling
devices to optimize energy flow and usage. In this UseCase the energy manager
requests information about the EVSE to integrate it into its manageable grid.

As the energy manager only accesses data of other SPINE _Devices_ in this
UseCase the energy manager can be seen as a SPINE Client.

The SPINE data model of a SPINE _Device_ representing an energy manager in the
EVSECC UseCase is very simple and only requires the presence of a SPINE
_Feature_ with `client` role to access data.

When building the SPINE _Device_ the DetailedDiscovery can be activated to run
automatically with calling `db.setDiscoverDevices(true)`[^11]. After verifying
(by executing the UseCaseDiscovery[^12]) the EVSECC UseCase is
supported on detected SPINE _Devices_ the UseCase can be executed.

#### Client Functionality

The energy manager requests data from the EVSE by sending _read messages_ to the
EVSE _functions_ and subscribes to the EVSE _Features_ to get notified about any
runtime changes:

```java
// cmd contains the function which shall be read on the Feature
CmdType dcReadCmd = new CmdType();
dcReadCmd.setDeviceClassificationManufacturerData(
    new DeviceClassificationManufacturerData());

// parse methods extract the Function data and process it
clientFeature.requestSubscription(evseDeviceClassificationAddress,
    FeatureTypeEnumType.DEVICE_CLASSIFICATION, this::parseDCUpdate);
clientFeature.requestRead(evseDeviceClassificationAddress, dcReadCmd)
    .whenComplete(this::parseDCReply);

CmdType ddReadCmd = new CmdType();
ddReadCmd.setDeviceDiagnosisStateData(new DeviceDiagnosisStateData());

clientFeature.requestSubscription(evseDeviceDiagnosisAddress,
    FeatureTypeEnumType.DEVICE_DIAGNOSIS, this::parseDDUpdate);
clientFeature.requestRead(evseDeviceDiagnosisAddress, ddReadCmd)
    .whenComplete(this::parseDDReply);
```

jSPINE then handles all messages, parses notifications and manages
acknowledgements.

## Future Work

As EEBUS SPINE is a relatively new protocol new versions will be released which
will be integrated into jSPINE. Different message formats than XML (e.g. JSON)
could also be integrated into jSPINE in the future. EEBUS Use Cases are
currently developed at the Fraunhofer ISE with the help of jSPINE.


[^1]: https://www.eebus.org/what-is-eebus/
[^2]: Smart Home IP
[^3]: Smart Premises Interoperable Neutral Message Exchange
[^4]: https://www.eebus.org/technology/
[^5]: https://www.openmuc.org/eebus/jspine
[^6]: https://pen.iana.org/pen/PenApplication.page
[^7]: https://www.openmuc.org/eebus/jspine/javadoc/org/openmuc/eebus/spine/spi/FeatureFunction.html
[^8]: The UseCase Specification can be downloaded at https://www.eebus.org/media-downloads
[^9]: https://www.openmuc.org/eebus/jspine/javadoc/org/openmuc/eebus/spine/api/Device.html#getBuilder()
[^10]: https://www.openmuc.org/eebus/jship
[^11]: https://www.openmuc.org/eebus/jspine/javadoc/org/openmuc/eebus/spine/impl/DeviceBuilder.html#setDiscoverDevices(boolean)
[^12]: https://www.openmuc.org/eebus/jspine/javadoc/org/openmuc/eebus/spine/api/NodeManagement.html#getFullUseCaseInformationRequest(java.lang.String)
