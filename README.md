# Build BusinessEvents (BE) Application in 10 minutes

## Overview

This tutorial implements a TIBCO BE application from scratch.  The application is a subscriber of TIBCO FTL messages.  When it receives a message, it will create a new Clock object, or reset the tick of a running Clock.  If no message is received, the Clock will increase its tick every 10 seconds. 

This introduces the most common scenario of an event processing agent that acts at realtime on incoming events from messaging or API services, as well as scenarios of no-events, i.e., missed messages that are expected within a given time period.

We first define a message subscriber by using an FTL channel destination.  We then implement a rule function that will create a clock when an FTL message is received.  In case that the clock is already running, we provide a "Reset" rule that will reset the tick to 0 in the clock that matches the received message.  If no message is received, we provide a state-machine that will increase the clock tick every 10 seconds.

Video for this tutorial can be found in [Wiki page](https://github.com/learn-tibco-cep/tutorials/wiki/Get-Started)

## Artifacts to be created

* Define `Clock` Concept with attibute - tick long
* Define `Demo` Event (for reset) with property - message string
* Define `FTL` Connection with server realm URL = http://$(hostname):8585
* Define `FTL` channel `demo` destination for FTL endpoint
* Implement function `onDemoEvent` to fetch or create a Clock upon receipt of an FTL message
* Implement rule `Reset` to reset a clock when the Clock concept matches a Demo event
* Define Clock state machine to increase its tick in 10-second interval
* Define `CDD` for applicatoin deployment and execution

## start docker container for FTL server

This tutorial uses TIBCO FTL messaging service.  To test the application, you must follow the instruction in [Prerequisites](https://github.com/learn-tibco-cep/tutorials/wiki/Prerequisites), install and start a FTL server.  For example, you may start FTL server in Docker:

```
docker run -d -p 8585:8585 ftl-tibftlserver:6.9.1 --name ftls1@$(hostname):8585
```

Publish FTL test message `hello world earth`:

```
docker run -it --rm ftl-client:6.9.1 tibftlsend $(hostname):8585
```

## `demo` FTL Destination

```
Name:               demo
Default Event:      /Events/Demo

Application Name:   default
Endpoint Name:      default
Instance Name:      default
Format Name:        helloworld
Content Matcher:    {"type": "hello"}
Durable Name:       demo
```

## `onDemoEvent` Rule Function

```
void rulefunction RuleFunctions.onDemoEvent {
    attribute {
        validity = ACTION;
    }
    scope {
        Events.Demo evt;
    }
    body {
        System.debugOut("Received message: " + evt.message);
        evt.message = String.split(evt.message, " ")[0];
        Concepts.Clock clock = Instance.getByExtIdByUri(evt.message, "/Concepts/Clock");
        if (clock == null) {
            clock = Concepts.Clock.Clock(evt.message /*extId String */, 0 /*tick long */);
            System.debugOut("Created new clock " + clock@extId);
        }
    }
}
```

## `Reset` Rule

```
rule Rules.Reset {
    attribute {
        priority = 5;
        forwardChain = true;
    }
    declare {
        Concepts.Clock clock;
        Events.Demo evt;
    }
    when {
        evt.message == clock@extId;
    }
    then {
        clock.tick = 0;
        System.debugOut("Reset clock " + clock@extId);
    }
}
```

## Clock State Machine: `ClockStateMain`

General: 
* Owner Concept: /Concepts/Clock
* Main: check the box

Add state `Ticking`:
* Transition in: `No Condition`
* Transition out: Rule Conditions=`false;`

`Ticking` state:
* Timeout: Units=`Seconds`; Timeout State=`Current`
* Timeout Expression: `return 10;`
* Timeout Action:
```
clock.tick += 1;
System.debugOut("Clock " + clock@extId + " added a tick " + clock.tick);
```

## Test the Application

* Build ear
* Create cdd
* Run configuration

To Run the application as implemented in this repository, clone this repository, and import it into BE studio workspace, `$WS`, as a `Existing TIBCO BE Studio Project`.  Build the enterprise archive, e.g., `$WS/Demo.ear`, then start it as follows:

```
$BE_HOME/bin/be-engine --propFile $BE_HOME/bin/be-engine.tra -n demo -u default -c ${WS}/Demo/Demo.cdd Demo.ear
```