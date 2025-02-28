---
title: "Philatelist, YANG-based Network Controller collection and aggregation framework integrating Telemetry data and Time Series Databases"
abbrev: "Philatelist"
category: std

docname: draft-lindblad-tlm-philatelist-latest
submissiontype: IETF
number:
date: 2025-02-28
consensus: true
v: 3
area: OPS
workgroup: GREEN WG
keyword:
 - YANG
 - telemetry
 - collection
 - aggregation
 - time series database
 - TSDB
venue:
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "janlindblad/draft-tlm-philatelist"
  latest: "https://janlindblad.github.io/draft-tlm-philatelist/draft-lindblad-tlm-philatelist.html"

author:
 -
    fullname: "Jan Lindblad"
    organization: All For Eco
    email: "jan.lindblad+ietf@for.eco"

normative:
  RFC6570:
  RFC7950:
  RFC9195:
  I-D.draft-kll-yang-label-tsdb-00:
  I-D.draft-palmero-ivy-ps-almo-01:

informative:
  I-D.draft-ietf-opsawg-collected-data-manifest-05:
  I-D.draft-claise-netconf-metadata-for-collection-03:
  I-D.draft-ietf-nmop-yang-message-broker-integration-06:
  RFC5424:
  RFC7603:
  RFC8641:
  RFC9232:
  Redfish:
    title: DMTF Redfish
    target: https://www.dmtf.org/standards/redfish
    date: 2025-01-13
  Sensor_Service_Methods:
    title: Schneider Electric Sensor Service Methods
    target: https://community.se.com/t5/DCE-web-services-API/Sensor-Service-Methods/ta-p/446584
    date: 2024-06-20

--- abstract

Timestamped telemetry data is collected en masse today.  Mature tools are typically used, but the data is often collected in an ad hoc manner.  While the dashboard graphs look great, the resulting data is often of questionable quality, not well defined, and hard to compare with seemingly similar data from other organizations.

This document proposes a standard, extensible, cross domain framework for collecting and aggregating timestamped telemetry data in a way that combines YANG, metadata and Time Series Databases to produce more transparent, dependable and comparable results.  This framework is implemented in the Network Controller layer, but is rooted in data that is collected from all kinds of Network Elements and related systems.

--- middle

# Introduction

## The Problem

Many organizations today are collecting large amounts of telemetry data from their networks and data centers for a variety of purposes.  Much (most?) of this data is funneled into a Time Series Database (TSDB) for display in a graphical dashboard or further (AI-backed) processing and decision making.

While this data collection is often handled using existing and stable tools, there generally seems to be little commonality when it comes to what is meaured, how the data is aggregated, or definitions of the measured quantities (if any).

Data science issues like adding overlapping quantities, adding quantities of different units of measurement, or quantities with different scopes, are likely common.  Such errors are hard to detect given the ad hoc nature of the collection.  This often leads to uncertainty regarding the quality of the conclusions drawn from the collected data.

## The Solution

The Philatelist framework proposes to standardize the collection, definitions of the quantities measured and meta data handling to provide a robust ground layer for telemetry collection on the Controller side.  The architecture defines a few interfaces, but allows great freedom in the implementations with its plug-in architecture.  This allows flexibility enough that any kind of quantity can be measured, any kind of collection protocol and mechanism employed, and the telemetry data flows aggregated using any kind of operation.

To do this, YANG is used both to describe the quantities being measured, as well as act as the framework for the metadata management.  Note that the use of YANG here does not limit the architecture to devices or sources supporting traditional YANG-based transport protocols.  YANG is used to describe the data, regardless of which format, protocol or source it arrives from.

Initially developed in context of the Power and Energy Efficiency work (POWEFF), the authors realized both the potential and the need for this collection and aggregation architecture to become a general framework for collection of a variety of metrics.

There is not much point in knowing the "cost side" of a running system (as in energy consumption or CO2e-emissions) if that cannot be weighed against the "value side" delivered by the system (as in transported bytes, VPN connections, music streaming hours, or number of cat videos, etc.), which means traditional performance metrics will play an equally important role in the collection.

## YANG-based Telemetry Outlook

Much work has already gone into the area of telemetry, YANG, and their intersection.  E.g.
{{I-D.draft-ietf-opsawg-collected-data-manifest-05}},
{{I-D.draft-claise-netconf-metadata-for-collection-03}} and
{{I-D.draft-ietf-nmop-yang-message-broker-integration-06}} come to mind. We (the POWEFF authoring team) would like to work with the authoring teams of these drafts to align our joint work.  We believe this work generally fits well with the principles outlined in the Network Telemetry Framework {{RFC9232}}.

Many essential data sources in real world deployments do not support any YANG-based interfaces, and that situation is expected to remain for the forseable future, which is why we find it important to be able to ingest data from free form (often REST-based) interfaces, and then add the necessary rigor on the Collector level.  Then output the datastreams in formats that existing, mature tools can consume directly.

A couple of collection source examples, just to stimulate the imagination:

* SYSLOG {{RFC5424}}

* EMAN MIBs over SNMP {{RFC7603}}

* DMTF's Redfish REST API {{Redfish}} (no endorsement)

* Proprietary REST APIs, e.g. {{Sensor_Service_Methods}} (no endorsement)

* YANG-Push {{RFC8641}}

In particular, this draft depends on the mapping of YANG-based structures to the typical TSDB tag-based formats described in {{I-D.draft-kll-yang-label-tsdb-00}}.

For the evolution of the YANG-based telemetry area, we believe this approach, combining pragmatism in the telemetry data stream interfaces with rigor and transparency regarding the data content, is key.  We would like to make this work fit in with the works of others in the field.

## The Philatelist Name

This specification is about a framework for collection, aggregation and interpretation of timestamped telemetry data.  The definition of "philatelist" seems close enough.

~~~

1. philatelist

noun. ['fɪˈlætəlɪst'] [fih-LAT-uh-list]
      A collector and student of postage stamps.

Synonyms
- collector
- aggregator

~~~
{: title="Source: https://www.synonym.com/synonyms/philatelist"}

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terminology defined in
{{RFC7950}}.

In addition, this document defines the following terms:

TSDB
: Time Series Database.

Sensor
: An entity in a system that delivers a snapshot value of some quantity pertaining to the system.  Sensors are identified by their Sensor Path.

Sensor Path
: A textual representation of the sensor's address within the system.

# Architecture Overview

## Functional Role Diagram

The following role diagram explains the basic concepts of the architecture. Many of the functional units would exist in many instances in a real deployment. For example, in a real deployment there would be lots of Network Elements with the Provider role.

On top we have a Network Orchestrator that ensures the Network Controller functions get a suitable configuration. The Collector, Index and Aggregator functions run as part of one or more Network Controllers. Collectors and Aggregators are responsible for ensuring there is a TSDB Partition (also known as bucket, interval, segment, etc. in various TSDB implementations) to receive the potentially large volumes of telemetry data that they produce themselves, or in the case of the Collector, may configure a Provider Network Element to send the collected data to the TSDB, or to the Collector itself, which then passes it on to the TSDB.

~~~
                    +--------------+
                    |   Network    |
                    | Orchestrator |
                    +------+-------+
                           |
       +-------------------+-------------------+
       v                   v                   v
+--------------+    +--------------+    +--------------+
|  *Collector* |    |   *Index*    |    | *Aggregator* |
|   Network    |--->|   Network    |<---|   Network    |
|  Controller  |    |  Controller  |    |  Controller  |
+--------------+    +--------------+    +--------------+
       |   /\  \\                :                  /\
       |   ||    \\              :                  ||
       v   ||      \\          ____________         ||
+--------------+     \\       /            \       //
|  *Provider*  |       ====> (     TSDB     ) <====
|   Network    | ==========> |\____________/|
|   Element    | ==========> |              |
+--------------+             | *Partition*  |
                             |              |
                              \____________/

~~~
{: title="Philatelist Functional Role Diagram."}

In the figure, single line arrows indicate control/configuration flow.
Double line arrows indicate telemetry data flow. The dotted line
between the Index and the TSDB indicates that the index reflects the TSDB
partition contents.

## Dashboards

In addition to the functional roles, there is a concept of Dashboards, which is used both within Providers and Collectors. A Dashboard is a particular collection of sensors and controls that have been predefined for particular use cases.  Network Elements may implement and publish one or more of these predefined Dashboards, and Controllers may know how to interpret one or more of them. Dashboards contain one or more dashboard items, each one a sensor or control.

For example, a particular Network Element might publish two dashboards. One Dashboard might be called "Current Power Draw" and contain only a single dashboard item which allows the Controller to read out the Network Element's total power draw at this instance. The second Dashboard might be called "Subsystem Power" and contain a tree of dashboard items, which allows the Controller to read out the current power draw of the Network Element's various subsystems.  The contents of each Dashboard is defined as a YANG structure in some standards document, or might be a vendor specific YANG definition.

A key point of this architecture is that Dashboard descriptions (in YANG) can be provided also for Network Elements that offer no YANG-based management interfaces at all, or for Network Elements hosting a YANG-based interface, but that were released prior to this document being written.

## Collection Data Flow Tree Diagram

The deployment of a Philatelist framework consists of a collection of Controller plug-in components with well defined interfaces.  Here is an example of a deployment.  Each box is numbered in the lower right for easy reference.

~~~
                      +-----------------+
                      | USER INTERFACE  |
                      |   Graph View    |
                      +--------------11-+
                               |
                      +-----------------+
                      |    PROCESSOR    |
                      | Recommendation  |
                      |     Engine      |
                      +--------------21-+
                               |
                      +-----------------+
                      |   AGGREGATOR    |
                      |   Data Center   |
                      +--------------31-+
                               |
       +---------------+-------+-------+--------------+
       |               |               |              |
+------------+  +------------+  +------------+  +------------+
| PROCESSOR  |  | AGGREGATOR |  | AGGREGATOR |  | AGGREGATOR |
| Normalizer |  |  Network   |  |  Storage   |  |  Compute   |
+---------41-+  +---------42-+  +---------43-+  +---------44-+
       |           |                   |\             |\
+------------+     |     +------+------------+  +------------+------+
| COLLECTOR  |     |     | YANG | COLLECTOR  |  | COLLECTOR  | YANG |
|  Cooling   |     |     +---52-+ Storage 1  |  | Compute 1  +---55-+
+---------51-+     |            +---------53-+  +---------54-+
       |           |             \ Storage 2  \  \ Compute 2  \
+------------+     |              +------------+  +------------+
|  PROVIDER  |     |               \ Storage N  \  \ Compute N  \
|Utility Bill|     |                +------------+  +------------+
+---------61-+     |
                   +--------------+
                   |              |
            +------------+  +------------+
            | PROCESSOR  |  | COLLECTOR  |
            | Normalizer |  |  Routers   |
            +---------71-+  +---------72-+
                   |              |\
            +------------+  +------------+
            | COLLECTOR  |  |  PROVIDER  |
            |  Firewall  |  |  Router 1  |
            +---------81-+  +---------82-+
                   |         \  Router 2  \
            +------------+    +------------+
            |  PROVIDER  |     \  Router N  \
            |  Firewall  |      +------------+
            +---------91-+

~~~
{: title="Example Philatelist component deployment."}

Each component in the above diagram, represents a logical function.  Many of the functions represented by these boxes could be running within a single server, or they could be fully distributed, or, perhaps more likely, something in between.

Provider components (61, 82, 91) are running on a Network Element system that supports a YANG-based telemetry data server.  The telemetry data flows from the telemetry source system to a Time Series Database (TSDB).

Collector components (51, 72, 81) ensure the Providers are programmed properly to deliver the telemetry data to the TSDB Partition designated by the Collector.  In some cases this flow may be direct (e.g. via a message bus) from the Provider to the TSDB, in other cases, it may be going through the Collector.  In some cases the collector may be polling the Provider, in others it may have set up an automatic, periodic subscription.

Many telemetry Provider systems will not have any on-board YANG-based telemetry server.  Such servers will instead be managed by a Collector capable of handling a particular kind of Provider (53, 54).  Such a Collector is still responsible to set up a telemetry data stream to the Collector's TSDB.  In this case, the Collector will also supply a YANG description (52, 55) of the incoming data stream.

Processor components (21, 41, 71) are transforming the data stream in some way, e.g. converting from one unit of measurement to another, or adjusting the data values recorded to also include some aspect that this particular sensor is not taking into account.

Aggregator components (31, 42, 43, 44) combine the time series telemetry data flows using some operation, e.g. summing, averaging or computing the max or min over them.  In this example there are aggregators for Network, Storage, Compute and the entire Data Center

On top of the stack, we may often find a (graphical) user interface (11), for human consumption of the intelligence acquired by the system.  Equally relevant is of course an (AI) application making decisions based on findings in the aggregated telemetry flow.

## The Provider Component

A Provider is a Network Element, or any other kind of relevant sensor, that is the source of telemetry data that may or may not offer a YANG-based management interface.  It may, for example, provide an SNMP, Redfish or proprietary REST API.

Each Provider typically has a large number of "sensors" that can be polled or in some cases subscribed to. It may also offer some controls (configurables or actionables).

One problem with the sensors is that the sensors relevant for a given use case are often spread around inside the Provider system, and many may not know about all of them.  Also, the metadata associated with each sensor is often only missing or only available in human readable form (free form strings), rather than in a strict machine parsable format.

~~~
    /hardware/component[name="psu3"]/.../sensor-data/value
    ...
    /interfaces/interface[name="eth0/0"]/.../out-broadcast-packets
    ...
    /routing/mpls/mpls-label-blocks/.../inuse-labels-count
    ...
~~~
{: title="Example of scattered potential sensors in a device."}

To solve these problems, the Provider YANG module contains a list of Dashboards.  Each dashboard contains the sensors and controls useful for some particular use case. The contents of the Dashboards is often defined by a standard, but could also be defined in proprietary YANG modules. Each dashboard item is listed with their sensor paths and machine parsable units, definition and any other metadata.

An admin user or application can then copy the sensor definition from the Dashboard and insert into the Collector's configuration with items to collect and send to the TSDB.

~~~ yang-tree
module: ietf-tlm-philatelist-provider
  +--rw tlm-provider
     +--rw dashboards
     |  +--rw dashboard* [id]
     |     +--rw id                       identityref
     |     +--rw items* [tsdb-path]
     |        +--rw tsdb-path             -> ../../../../dash-items/
     |                                       dash-item/tsdb-path
     +--rw dash-items
     |  +--rw dash-item* [tsdb-path]
     |     +--rw tsdb-path                string
     |     +--rw item-type                identityref
     |     +--rw accuracy
     |     |  +--rw max-error-relative?   something
     |     |  +--rw max-error-offset?     something
     |     +--rw label* [name]
     |     |  +--rw name                  string
     |     |  +--rw (value-source)?
     |     |     +--:(static-values)
     |     |     |  +--rw static-values*  string
     |     |     +--:(runtime-values)
     |     |        +--rw runtime-values* -> ../../../dash-item/
     |     |                                 tsdb-path
     |     +--rw access-path?             string
     |     +--rw access-params?           -> ../../../accesses/
                                             access/id
~~~
{: title="YANG tree diagram of the Provider Dashboard list."}

Note: The "something" YANG-type is used in many places in this document.  That is just a temporary placeholder we use until we have figured out what the appropriate type should be.

Each Dashboard in the dashboard list has a name that is an identityref. That makes it possible to define particular dashboards with well known names and contents in YANG, so that Providers and Collectors know what to expect. Each dashboard refers to a set of dashboard items (some of which may be the same in multiple Dashboards). Each dashboard item has a type that is defined as a YANG identity, making them maximally extensible.  Examples of sensor types might be a sensor for energy measured in kWh, or energy measured in J, or temperature measured in F, or in C, or in K.

Each dashboard item has an access path and access parameters. These are a mapping into the access mechanism the Collector must use to poll or subscribe to the sensor value.

~~~ yang-tree
module: ietf-tlm-philatelist-provider
  +--rw tlm-provider
     +--rw accesses
     |  +--rw access* [id]
     |     +--rw id                                 string
     |     +--rw method?                            identityref
     |     +--rw get-local-file-once
     |     |  +--rw filename?                       string
     |     +--rw get-static-url-once
     |     |  +--rw url?                            something
     |     +--rw gnmi-polling
     |     |  +--rw encoding?                       something
     |     |  +--rw protocol?                       something
     |     +--rw restconf-get-polling
     |     |  +--rw xxx?                            string
     |     +--rw netconf-get-polling
     |     |  +--rw xxx?                            string
     |     +--rw restconf-yang-push-subscription
     |     |  +--rw xxx?                            string
     |     +--rw netconf-yang-push-subscription
     |     |  +--rw xxx?                            string
     |     +--rw redfish-polling
     |     |  +--rw xxx?                            string
     |     +--rw frequency?                         sample-frequency
~~~
{: title="YANG tree diagram of the Provider Accesses list."}

The list of access methods contains a number of YANG-based and non-YANG based access methods, but this set of access methods can also be extended by YANG-augmentation. The get-local-file-once access method allows reading fixed values from a data sheet encoded in YANG-instance data file format {{RFC9195}}, and  the get-static-url-once access method does the same from a given URL. That URL may be served from the Network Element, or from the Collector itself or anywhere else on the network or even internet.

The access-path leaf discussed above (/tlm-provider/dash-items/dash-item/access-path) contains the path to the item that should be read or subscribed to. If the dash-item in question is for a YANG-based interface, then that path would be an XPath expression, with prefixes. Those prefixes need to be mapped to XML namespaces (NETCONF) or YANG module names (RESTCONF). That mapping is provided by the prefix-mappings list.

~~~ yang-tree
module: ietf-tlm-philatelist-provider
  +--rw tlm-provider
     +--rw prefix-mappings
        +--rw prefix-mapping* [prefix]
           +--rw prefix         string
           +--rw namespace?     string
           +--rw module-name?   string
~~~
{: title="YANG tree diagram of the Provider Prefix Mapping list."}

## The Collector Component

The Collector component is part of a Network Controller that collects telemetry data from Providers, typically by periodic polling or subscriptions, and ensures the collected data is stored in a Time Series Database (TSDB).  The actual data stream may or may not be passing through the collector component; the collector is responsible for ensuring data flows from the Provider to the destination TSDB, and that the data has a YANG description and is tagged with necessary metadata.  How the Collector agrees with a Provider to deliver data in a timely manner is beyond the scope of this document.

~~~
         +-------------+
         |  COLLECTOR  |
         +-------------+                     ___________
                |                           /           \
      +------------------+                 (    TSDB     )
      v                  v                 |\___________/|
+------------+    +------------+  STREAM 1 |             |
|  PROVIDER  |    |  PROVIDER  |  =======> |             |
| - sensor 1 |    | - sensor 1 |           |  Partition  |
| - sensor 2 |    | - sensor 4 |  STREAM 2 |             |
| - sensor 3 |    | - sensor 7 |  =======> |             |
+------------+    +------------+           |             |
          \\                      STREAM 3 |             |
            =============================>  \___________/
~~~
{: title="Example of Collector setting up three streams of telemetry data from two Providers to one TSDB Partition."}

The top of the Collector model contains a list of organizations, as a single Collector component might be doing collection work for different organizations (customers, departments, scopes) that must not be intermixed. Each organization has a list of device-groups pointing out specific Network Elements. Each group has a list of dashboards that will be queried. Since each Provider has a list of supported dashboards, the Collector simply copies the dashboards it is interested in to its own collection list.

~~~ yang-tree
module: ietf-tlm-philatelist-collector
  +--rw tlm-collector
     +--rw organizations
        +--rw organization* [name]
           +--rw name                     string
           +--rw device-groups
           |  +--rw device-group* [name]
           |     +--rw name               string
           |     +--rw devices*           string
           |     +--rw dashboards
           |     |  +--rw dashboard* [id]
           |     |     +--rw id           identityref
           |     |     +--rw items* [tsdb-path]
           |     |        +--rw tsdb-path  -> ../../../../
                                              dash-items/
                                              dash-item/
                                              tsdb-path
~~~
{: title="YANG tree diagram of the top part of the Collector model."}

The Collector model also contains the same dash-item list as shown above in the Provider model. This allows the Collector configuration to hold a local copy of the dashboards and dash-items it finds relevant, to guide its own collection work.

Finally, the Collector keeps a list of streams to work with, pointing out the sources (device-group and dash-name) and destination (a TSDB Partition).

~~~ yang-tree
module: ietf-tlm-philatelist-collector
  +--rw tlm-collector
     +--rw organizations
        +--rw organization* [name]
           +--rw tlm-streams
              +--rw tlm-stream* [id]
                 +--rw id                 something
                 +--rw sources* [device-group dash-name]
                 |  +--rw device-group    -> ../../../../
                 |  |                        device-groups/
                 |  |                        device-group/name
                 |  +--rw dash-name       -> ../../../../
                 |                           device-groups/
                 |                           device-group/
                 |                           dashboards/dashboard/id
                 +--rw destination?       partition-ref-t
~~~
{: title="YANG tree diagram of the Collector tlm-streams."}

The sensor groups are then arranged into streams from a collection of sources (that support the same set of sensor groups) to a destination.  This structure has been chosen with the assumption that there will be many source devices with the same set of sensor groups, and we want to minimize repetition.

## The Index Component

When Collectors collect, and when Aggregators process and aggregate telemetry data, they need to send this data to a TSDB Partition as destination. To keep track of which data is sent where, and what the connection details are for that partition, the Network Controller implements the Index YANG module. Both the Collector and Aggregator modules reference this module.

~~~ yang-tree
module: ietf-tlm-philatelist-index
  +--rw tlm-index
     +--rw partitions
        +--rw partition* [id]
           +--rw id                     something
           +--rw url?                   something
           +--rw organization?          something
           +--rw partition?             something
           +--rw impl-specific
              +--rw binding* [key]
                 +--rw key              string
                 +--rw (value-type)?
                    +--:(value)
                    |  +--rw value?     string
                    +--:(values)
                    |  +--rw values*    string
                    +--:(env-var)
                       +--rw env-var?   string
~~~
{: title="YANG tree diagram of the Index TSDB Partitions."}

The implementation specific part of the model is for integration with specific TSDB implementations. Each such integration may need a specific sef of key-value bindings, that can be provided in this list.

## The Processor and Aggregator Components

Processor components are parts of a Network Controller that take an incoming data flow and transforms it somehow, and possibly augments it with a flow of derived information.  The purpose of the transformation could be to convert between different units of measurement, correct for known errors in the input data, or fill in approximate values where there are holes in the input data.

Aggregator components take multiple incoming data flows and combine them, typically by adding them together, taking possible differences in cadence in the input data flows into account.

Processor and Aggregator components provide a YANG model of their output data, just like the Collector components, so that all data flowing in the system has a YANG description and is associated with metadata.

Note: In the current version of the YANG modules, a Processor is simply an Aggregator with a single input and output.  Unless we see a need to keep these two component types separate, we might remove the Processor component and keep it baked in with the Aggregator.

~~~
                  +-------------+
                  | AGGREGATOR  |
                  +-------------+
                         |
            +------------+------------+
            v                         v
        ___________               ___________
       /  SOURCE   \             /DESTINATION\
      ( PARTITION 1 )           (  PARTITION  )
      |\___________/| STREAM 1  |\___________/|
      |             | ========> |             |
      |             |           |             |
      |             | STREAM 2  |             |
       \___________/  ===##===>  \___________/
                         ||
        ___________      ||
       /  SOURCE   \     ||
      ( PARTITION 2 )   //
      |\___________/| ==
      |             |
      |             |
      |             |
       \___________/
~~~
{: title="Example of an Aggregator setting up two streams of telemetry data from two sources to one destination."}

In this diagram, the sources and destination look like separate TSDBs, which they might be.  They may also be different partitions within the same TSDB.

Each stream is associated with one or more inputs, one output and a processing operation.  All the input streams are combined using one or more aggregation operations.  Some basic operations have been defined in the Aggregator YANG module, but the set of operations has been designed to be maximally extensible.

~~~ yang-tree
module: ietf-tlm-philatelist-aggregator
  +--rw tlm-aggregator
     +--rw aggregations
     |  +--rw aggregation* [id]
     |     +--rw id           string
     |     +--rw input* [source]
     |     |  +--rw source    partition-ref-t
     |     +--rw operation?   -> ../../../operations/operation/id
     |     +--rw output
     |        +--rw destination?   partition-ref-t
~~~
{: title="YANG tree diagram of the top Aggregator model."}

The operations listed below are basic aggregation operations.  Linear-sum is just adding all the input sources together, with linear interpolation when their data points don't align perfectly in time.  Rolling average is averaging the input flows over a given length of time.  The filter-age drops all data points that are outside the min to max age.  The function allows plugging in any other function the Aggregator may have defined, but more importantly, the operations choice is easily extended using YANG augment to include any other IETF or vendor specified augmentations.

~~~ yang-tree
module: ietf-tlm-philatelist-aggregator
  +--rw tlm-aggregator
     +--rw operations
        +--rw operation* [id]
           +--rw id                       something
           +--rw (op-type)?
              +--:(linear-sum)
              |  +--rw linear-sum
              +--:(linear-average)
              |  +--rw linear-average
              +--:(linear-max)
              |  +--rw linear-max
              +--:(linear-min)
              |  +--rw linear-min
              +--:(rolling-average)
              |  +--rw rolling-average
              |     +--rw timespan?       something
              +--:(filter-age)
              |  +--rw filter-age
              |     +--rw min-age?        something
              |     +--rw max-age?        something
              +--:(function)
                 +--rw function
                    +--rw name?           something
~~~
{: title="YANG tree diagram of the Aggregator operations list."}

## The Link to Assets

In {{I-D.draft-palmero-ivy-ps-almo-01}}, the DMLMO team has built an inventory structure that describes systems, subsystems and their soft- and hardware components.  They are called assets in the DMLMO YANG models.  Some of the collected telemetry data streams may pertain to quite precisely to these assets, and it may be interesting to see the linkage.  For this reason, there is an optional module, ietf-tlm-philatelist-assets, that augments the Philatelist Index structure and adds the possibility to point to a DMLMO asset that the TSDB Partition pertains to.

# YANG Modules

## Base types module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-types.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-types@2024-04-15.yang”}

## Dashboard abstract interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-dashboard.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-dashboard@2024-04-15.yang”}

## Provider interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-provider.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-provider@2024-04-15.yang”}

## Index interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-index.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-index@2024-04-15.yang”}

## Collector interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-collector.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-collector@2024-04-15.yang”}

## Aggregator interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-aggregator.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-aggregator@2024-04-15.yang”}

## Assets interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-assets.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-assets@2024-04-15.yang”}

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


# Changes (to be deleted by RFC Editor)

## From version -02 to -03
- Updated author affiliation and document working group
- Updated references to other documents

## From version -01 to -02
- Adopted {{RFC6570}} style URI Templates in ietf-tlm-philatelist-dashboard
- Moved up the outlook section, and clarified the relation to existing device side telemetry solutions.
- Added several informative references to prior work
- Reformatted YANG modules for shorter lines to fit IETF layout

## From version -00 to -01
- Introduced Dashboard and Index concepts
- Restructured YANG into three controller modules: collector, index, aggregator
- Restructured YANG into one device module: provider
- Restructured YANG common parts into one abstract module: dashboard
- Split YANG modules, some contents going into poweff-specific modules
- Renamed remaining YANG modules from -poweff- to -tlm-philatelist-
- Updated text to reflect new module organization
- Added optional linkage to DMLMO assets

## Version -00
- Initial version.

--- back

# Acknowledgments
{:numbered="false"}

Kristian Larsson has provided invaluable insights, experience and validation of the design.  Many thanks to the entire POWEFF team for their committment, flexibility and hard work behind this.  Thanks to James Henderson for the review, a number of small fixes and several good susggestions.  Hat off to Benoît Claise, who inspires by the extensive work produced in IETF over the years, and in this area in particular.
