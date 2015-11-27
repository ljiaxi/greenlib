# GreenRouter
## Purpose

The purpose of this documentation is to explain the concept and usage of GenericRouter. An example will be shown for its efficient use in modelling.

## Scope

This document will cover the usage and APIs of GenericRouter and not go into the implementation details.

## Motivation

**GreenRouter** is used to connect the IPs using the GreenSocket infrastructure.




## Overview

The TLM technology provides base protocol, sockets and payload to be used for communication between various IPs. These are commonly used for memory mapped bus communication, where the Target has certain registers with known addresses and Initiator can send transactions to read or write these registers.



![Interconnect two models](img/2-models.png)

When `model1` is connected to more than one models, it may require more than one init_socket, as shown in the below diagram. This can also be achieved by using multi-socket versions but still the user has to take care of which transport call to make on which port.

![Interconnect three models](img/3-models.png)

In the case shown above we can assume that `model2` and `model3` have disjoint address spaces. Then within 'model1' it can be decoded which payload should go to which socket depending upon the address configured in the payload. Further, when there are multiple initiators and targets all talking to each other, then making connections between each of them and doing decoding in each of them becomes a tedious and error-prone task.


![Generic Router](img/genericrouter.png)

A `GenericRouter` handles the routing information within itself. And using this information, it appropriately forwards a transaction request to the desired target. By using a `GenericRouter`, making connections becomes easy. So now the Initiator need to contain only one init_socket for all the Targets it wishes to communicate to. And similarly, a target contain a single `target_socket` to receive transactions from any of the initiators.



```cplusplus
T_1.target_socket.base_addr = 0x40; T_1.target_socket.high_addr = 0x60
T_2.target_socket.base_addr = 0x80; T_2.target_socket.high_addr = 0x90
```
The above statements are telling that `T_1` expects to recieve payloads whose address lies in the range from 0x40 to 0x60. Similarly `T_2` expects to recieve payloads whose address lies in the range from 0x80 to 0x90. When the init_socket of GenericRouter is connected to `T_1` and `T_2`, it extracts this information from them and stores it internally. This information is then used at runtime to appropriately route the payload. So if the Router recieves two payloads, `P1` and `P2`, having their address field as 0x45 and 0x81 respectively, then `P1` is sent to `T_1` and `P2` is sent to `T_2`.

The address information is used by GenericRouter to generate an internal map of address ranges for each target. This map is used to direct the transactions to their desired targets. The basis on which user wishes to route transactions can be supplied as an ADDR_MAP template parameter at the time of instantiation. It gives complete freedom to the designer to use this GenericRouter for memory mapped communication, signal level communication etc. Its details are discussed below in the section on [Routing Algortihm](routing-algorithm.md).


### Internals

The GenericRouter is defined in gs::gp namespace. To use it, one has to make following includes,

```cplusplus
#include “greenrouter/genericRouter.h”
```

The GenericRouter accepts some template parameters, which allows user to work on different protocol types and different address decoding mechanism and some other related parameters(which are required when the user is extending base protocol). The template arguments have default values also, so that GenericRouter can be readily used on base protocol and a simple address map.

```cplusplus
template <BUSWIDTH, PORTMAX, TRAITS, RESP_TYPE, ADDR_ERR_RESP, SET_RESP_CALL, ADDR_MAP> GenericRouter
```

**Default value** 255.

* `TRAITS` : the protocol type for the transactions that are going to use this bus. 

**Default value** `tlm::tlm_base_protocol_types`.


**Default Value** `tlm::tlm_response_status`.


**Default Value** `tlm::TLM_ADDRESS_ERROR_RESPONSE`.


**Default Value** `&TRAITS::tlm_payload_type::set_response_status`.


**Default Value** `SimpleAddressMap<TRAITS, PORTMAX>`.

#### Routing Algorithm



Again, later during simulation when the payloads are routed through this GenericRouter, the decode functionality of `ExtensionBased` `ADDR_MAP` determines the exact target to which the payload should be forwarded.

Following diagrams help explain the use as `SignalBus`.

![Signal Bus](img/signalbus.png)

The adjacent diagram shows the case of two initiators driving signals for two Targets in the system. As shown, `T_1` has an in signal named `X`, for which it has an `sc_in` port that is driven by an `sc_out` of `I_1`. Similary, `T_2` has two `sc_in` ports to recieve `Y` from `I_1` and `X` from `I_2`.

The diagram below shows the same scenario when a GenericRouter is used as a SignalBus.

![GenericRouter used as a Signal Bus](img/genericrouter-signalbus.png)

The adjacent diagram shows the case when a signal bus is placed between the initiators and targets for communicating signals. Each initiator contains an `initiator_signal_socket` and each target contains a `target_signal_socket`. These are connected to sockets of signal bus.

Please note that in this case the user does not configure any address ranges on the target_sockets of `T_1` and `T_2`. Instead it has to configure extensions on the target_sockets. These extensions correspond to the signal that the target is expecting. The user also has to specify the exact initiator from which a particular extension is expected. So for the above example we have:

```cplusplus
T_1.tar_sig_socket.set_source<X>(I_1.init_sig_socket.name());
T_2.tar_sig_socket.set_source<Y>(I_1.init_sig_socket.name());
T_2.tar_sig_socket.set_source<X>(I_2.init_sig_socket.name());
```

The signal socket documentation covers these APIs in more details.


```cplusplus
T_1.tar_sig_socket.set_source<Y>(I_1.init_sig_socket.name());
```

Now when the **SignalBus** recieves a payload with extension corresponding signal `Y` validated, it

#### Bus Protocol(scheduling)

The **GenericRouter** also has protocol ports. The protcol ports can be used to connect to a bus protocol class, by binding them to ports of `Protocol` class. The `Protocol` class acts as an arbiter and can be used to provide the scheduling algorithm for the transactions on the bus. The code package contains some example protocol (`simpleBus` and `dummy`) and scheduler(`fixedPriority` and `dynamicPriority`) classes.


### APIs

#### Assign an address

```cplusplus
void assign_address(baseAddr_, highAddr_, portNumber_)
``` 

Parameters:

* `baseAddr_` : base address
* `highAddr_` : high address
* `portNumber_` : port number

This is used when address based routing is being done. The target sockets can be configured with base and high addresses, or alternatively this API can be used to provide information to the GenericRouter.

#### Get number of routers

unsigned int getCurrentNumRouters()
```

Return:

* The total number of routers instantiated in the system.

```cplusplus
```

Return: 

* The id of this GenericRouter.

## Example
### Usage

To use greenrouter include the following files:

```cplusplus
#include <greenrouter/genericRouter.h>
#include <greenrouter/scheduler/fixedPriorityScheduler.h>
```

Create the following instances

```cplusplus
gs::gp::SimpleBusProtocol<32> p("Protocol", 10);
gs::gp::fixedPriorityScheduler s("Scheduler");
```

The second argument of SimpleBusProtocol is the clock period duration.

Bind the protocol, router and scheduler ports as shown.

```cplusplus
r.protocol_port(p);
p.router_port(r);
p.scheduler_port(s);
```

Instantiate the target and define the target socket's base address and high address.

```cplusplus
target.target_port.base_addr = 0x0;
```

Now bind the Master's initiator port with the router target port and router's initiator port with slave's target port.

```cplusplus
master.init_port(router.target_socket); router.init_socket(slave.target_port);
```

Note that there can be many initiators and many targets, bind the ports of initiators and targets in the similar fashion.


### Simple example

![Simple example](img/simple-example.png)

A small example with the name **simple** is kept in the `example` directory. In this example, sillysort is a master which reads/writes on the memory and sorts the string kept in the memory.



