---
title: "Philatelist, YANG-based collection and aggregation framework integrating Telemetry data and Time Series Databases"
abbrev: "Philatelist"
category: std

docname: draft-lindblad-tlm-philatelist-latest
submissiontype: IETF
number:
date: 2024-03-14
consensus: true
v: 3
area: OPS
workgroup: NETMOD
keyword:
 - YANG
 - telemetry
 - collection
 - aggregation
 - time series database
 - TSDB
venue:
  group: NETMOD
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "janlindblad/netmod-tlm-philatelist"
  latest: "https://janlindblad.github.io/netmod-tlm-philatelist/draft-lindblad-tlm-philatelist.html"

author:
 -
    fullname: "Jan Lindblad"
    organization: Cisco
    email: "jlindbla@cisco.com"

normative:
  RFC7950:
  I-D.draft-kll-yang-label-tsdb-00:
  I-D.draft-palmero-opsawg-ps-almo-00:

informative:
  I-D.draft-ietf-opsawg-collected-data-manifest-01:
  I-D.draft-claise-netconf-metadata-for-collection-03:

--- abstract

Timestamped telemetry data is collected en masse today.  Mature tools are typically used, but the data is often collected in an ad hoc manner.  While the dashboard graphs look great, the resulting data is often of questionable quality, not well defined, and hard to compare with seemingly similar data from other organizations.

This document proposes a standard, extensible, cross domain framework for collecting and aggregating timestamped telemetry data in a way that combines YANG, metadata and Time Series Databases to produce more dependable and comparable results.

--- middle

# Introduction

## The Problem

Many organizations today are collecting large amounts of telemetry data from their networks and data centers for a variety of purposes.  Much (most?) of this data is funneled into a Time Series Database (TSDB) for display in a dashboard or further (AI-backed) processing and decision making.

While this data collection is often handled using standard tools, there generally seems to be little commonality when it comes to what is meaured, how the data is aggregated, or definitions of the measured quantities (if any).

Data science issues like adding overlapping quantities, adding quantities of different units of measurement, or quantities with different scopes, are likely common.  Such errors are hard to detect given the ad hoc nature of the collection.  This often leads to uncertainty regarding the quality of the conclusions drawn from the collected data.

## The Solution

The Philatelist framework proposes to standardize the collection, definitions of the quantities measured and meta data handling to provide a robust ground layer for telemetry collection.  The architecture defines a few interfaces, but allows great freedom in the implementations with its plug-in architecture.  This allows flexibility enough that any kind of quantitiy can be measured, any kind of collection protocol and mechanism employed, and the telemetry data flows aggregated using any kind of operation.

To do this, YANG is used both to describe the quantities being measured, as well as act as the framework for the metadata management.  Note that the use of YANG here does not limit the architecture to traditional YANG-based transport protocols.  YANG is used to describe the data, regardless of which format it arrives in.

Initially developed in context of the Power and Energy Efficiency work (POWEFF), we realized both the potential and the need for this collection and aggregation architecture to become a general framework for collection of a variety of metrics.

There is not much point in knowing the "cost side" of a running system (as in energy consumption or CO2-emissions) if that cannot be weighed against the "value side" delivered by the system (as in transported bytes, VPN connections, music streaming hours, or number of cat videos, etc.), which means traditional performance metrics will play an equally important role in the collection.

## The Philatelist Name

This specification is about a framework for collection, aggregation and interpretation of timestamped telemetry data.  The definition of "philatelist" seems close enough.

~~~

1. philatelist

noun. ['fɪˈlætəlɪst'] a collector and student of postage stamps.

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

# Architecure Overview

The deployment of a Philatelist framework consists of a collection of plug-in compomnents with well defined interfaces.  Here is an example of a deployment.  Each box is numbered in the lower right for easy reference.

~~~

                      +-----------------+
                      | USER INTERFACE  |
                      |    Dashboard    |
                      |                 |
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

Each component in the above diagram, represents a logical function.  Many boxes could be running within a single server, or they could be fully distrubuted, or anything in between.

Provider components (61, 82, 91) are running on a telemetry source system that supports a YANG-based telemetry data server.  The telemetry data flows from the telemetry source system to a Time Series Database (TSDB).

Collector components (51, 72, 81) ensure the Providers are programmed properly to deliver the telemetry data to the TSDB designated by the collector.  In some cases this flow may be direct from the source to the TSDB, in other cases, it may be going through the collector.  In some cases the collector may be polling the source, in others it may have set up an automatic, periodic subscription.

Many telemetry source systems will not have any on-board YANG-based telemetry server.  Such servers will instead be managed by a collector specialized to handle a particular kind of source server (53, 54).  These specialized collectors are still responsible to set up a telemetry data stream from them to the collector's TSDB.  In this case, the collector will also supply a YANG description (52, 55) of the incoming data stream.

Processor components (21, 41, 71) are transforming the data stream in some way, e.g. converting from one unit of measurement to another, or adjusting the data values recorded to also include some aspect that this particular sensor is not taking into account.

Aggregator components (31, 42, 43, 44) combine the time series telemetry data flows using some operation, e.g. summing, averaging or computing the max or min over them.  In this example there are aggregators for Network, Storage, Compute and the entire Data Center

On top of the stack, we may often find a (graphical) user interface (11), for human consumption of the intelligence acquired by the system.  Equally relevant is of course an (AI) application making decisions based on findings in the aggregated telemetry flow.

## The Provider Component

A Provider is a source of telemetry data that also offers a YANG-based management interface.  Each provider typically has a large number of "sensors" that can be polled or in some cases subscribed to.

One problem with the sensors is that they are spread around inside the source system, and may not be trivial to locate.  Also, the metadata assciated with the sensor is often only missing or only available in human readable form (free form strings), rather than in a strict machine parsable format.

~~~
    /hardware/component[name="psu3"]/.../sensor-data/value
    ...
    /interfaces/interface[name="eth0/0"]/.../out-broadcast-packets
    ...
    /routing/mpls/mpls-label-blocks/.../inuse-labels-count
    ...
~~~
{: title="Example of scattered potential sensors in a device."}

To solve these problems, the Provider YANG module contains a sensor-catalog list.  Essentially a list of all interesting sensors available on the system, with their sensor paths and machine parsable units, definition and any other metadata.

An admin user or application can then copy the sensor definition from the sensor catalog and insert into the configuration in the colletor.

~~~ yang-tree
  +--ro sensor-catalog
      +--ro sensors
        +--ro sensor* [path]
            +--ro path?                     string
            +--ro sensor-type?              identityref
            +--ro sensor-location?          something
            +--ro sensor-state?             something
            +--ro sensor-current-reading?   something
            +--ro sensor-precision?         something
~~~
{: title="YANG tree diagram of the Provider sensor-catalog."}

Note: The "something" YANG-type is used in many places in this document.  That is just a temporarty placeholder we use until we have figured out what the appropriate type should be.

The sensor types are defined as YANG identities, making them maximally extensible.  Examples of sensor types might be energy measured in kWh, or energy measured in J, or temperature measured in F, or in C, or in K.

## The Collector Component

Collector components collect data points from sources, typically by periodic polling or subscriptions, and ensure the collected data is stored in a Time Series Database (TSDB).  The actual data stream may or may not be passing through the collector component; the collector is responsible for ensuring data flows from the source to the destination TSDB and that the data has a YANG description and is tagged with necessary metadata.  How the collector agrees with a source to deliver data in a timely manner is beyond the scope of this document.

~~~

         +-------------+
         |  COLLECTOR  |
         +-------------+                     ___________
                |                           /           \
      +------------------+                 ( DESTINATION )
      v                  v                 |\___________/|
+------------+    +------------+  STREAM 1 |             |
|   SOURCE   |    |   SOURCE   |  =======> |             |
| - sensor 1 |    | - sensor 1 |           |             |
| - sensor 2 |    | - sensor 4 |  STREAM 2 |             |
| - sensor 3 |    | - sensor 7 |  =======> |             |
+------------+    +------------+           |             |
          \\                      STREAM 3 |             |
            =============================>  \___________/

~~~
{: title="Example of Collector setting up three streams of telemetry data from two sources to one desitination."}

Each source holds a number of sensors that may be queried or subscribed to.  The collector arranges the sensors into sensour groups that presumably are logically related, and that are collected using the same method.  A number of collection methods (some YANG-based, some not) are modeled directly in the ietf-poweff-collector.yang module, but the set is designed to be easily extensible.

~~~ yang-tree
  +-- sensor-groups
  |  +-- sensor-group* [id]
  |     +-- id?                                something
  |     +-- method?                            identityref
  |     +-- get-static-url-once
  |     |  +-- url?                            something
  |     |  +-- format?                         something
  |     +-- gnmi-polling
  |     |  +-- encoding?                       something
  |     |  +-- protocol?                       something
  |     +-- restconf-get-polling
  |     |  +-- xxx?                            something
  |     +-- netconf-get-polling
  |     |  +-- xxx?                            something
  |     +-- restconf-yang-push-subscription
  |     |  +-- xxx?                            something
  |     +-- netconf-yang-push-subscription
  |     |  +-- xxx?                            something
  |     +-- redfish-polling
  |     |  +-- xxx?                            something
  |     +-- frequency?                         sample-frequency
  |     +-- path* [path]
  |        +-- path?                           string
  |        +-- sensor-type?                    identityref
  +-- tlm-streams
    +-- tlm-stream* [id]
        +-- id?                                something
        +-- source*                            string
        +-- sensor-group* [name]
        |  +-- name?   -> ../../../sensor-groups/sensor-group/id
        +-- destination?    -> ../../../destinations/destination/id
~~~
{: title="YANG tree diagram of the Collector sensor-groups and streams."}

The sensor groups are then arranged into streams from a collection of sources (that support the same set of sensor groups) to a destination.  This structure has been chosen with the assumption that there will be many source devices with the same set of sensor groups, and we want to minimize repetition.

## The Processor and Aggregator Components

Processor components take an incoming data flow and transforms it somehow, and possibly augments it with a flow of derived information.  The purpose of the transformation could be to convert between different units of measurement, correct for known errors in in the input data, or fill in approximate values where there are holes in the input data.

Aggregator components take multiple incoming data flows and combine them, typically by adding them together, taking possible differences in cadence in the input data flows into account.

Processor and Aggregator components provide a YANG model of the output data, just like the Collector components, so that all data flowing in the system has a YANG description and is associated with metadata.

Note: In the current version of the YANG modules, a Processor is simply an Aggregator with a single input and output.  Unless we see a need to keep these two component types separate, we might remove the Processor component and keep it baked in with the Aggregator.

~~~

                +-------------+
                | AGGREGATOR  |
                +-------------+
                       |
           +-----------+-----------+
           v                       v
      ___________             ___________
     /           \           /           \
    (  SOURCE 1   )         ( DESTINATION )
    |\___________/| FLOW 1  |\___________/|
    |             | ======> |             |
    |             |         |             |
    |             | FLOW 2  |             |
     \___________/  ===##=>  \___________/
                       ||
      ___________      ||
     /           \     ||
    (  SOURCE 2   )   //
    |\___________/| ==
    |             |
    |             |
    |             |
     \___________/

~~~
{: title="Example of an Aggregator setting up two flows of telemetry data from two sources to one desitination."}

In this diagram, the sources and destination look like separate TSDBs, which they might be.  They may also be different buckets within the same TSDB.

Each flow is associated with one or more inputs, one output and a series of processing operations.  Each input flow and output flow may have an pre-processing or post-processing operation applied to it separately.  Then all the input flows are combined using one or more aggregation operations.  Some basic operations have been defined in the Aggregator YANG module, but the set of operations has been designed to be maximally extensible.

~~~ yang-tree
  +-- tlm-flows
  |  +-- tlm-flow* [id]
  |     +-- id?                                string
  |     +-- (chain-position)?
  |        +--:(input)
  |        |  +-- input
  |        |     +-- source?
  |        |           -> ../../../../../sources/source/id
  |        +--:(output)
  |        |  +-- output
  |        |     +-- destination?
  |        |           -> ../../../../../destinations/destination/id
  |        +--:(middle)
  |           +-- middle
  |              +-- inputs*
  |              |     -> ../../../../flows/flow/id
  |              +-- pre-process-inputs?
  |              |     -> ../../../../operations/operation/id
  |              +-- aggregate?
  |              |     -> ../../../../operations/operation/id
  |              +-- post-process-output?
  |                    -> ../../../../operations/operation/id
  +-- operations
    +-- operation* [id]
        +-- id?                                something
        +-- (op-type)?
          +--:(linear-sum)
          |  +-- linear-sum
          +--:(linear-average)
          |  +-- linear-average
          +--:(linear-max)
          |  +-- linear-max
          +--:(linear-min)
          |  +-- linear-min
          +--:(rolling-average)
          |  +-- rolling-average
          |     +-- timespan?                  something
          +--:(filter-age)
          |  +-- filter-age
          |     +-- min-age?                   something
          |     +-- max-age?                   something
          +--:(function)
              +-- function
                +-- name?                      something
~~~
{: title="YANG tree diagram of the Aggregator flows and operations."}

The operations listed above are basic aggregation operations.  Linear-sum is just adding all the input sources together, with linear interpolation when their data points don't align perfectly in time.  Rolling average is averaging the input flows over a given length of time.  The filter-age drops all data points that are outside the min to max age.  The function allows plugging in any other function the Aggregator may have defined, but more importantly, the operations choice is easily extended using YANG augment to include any other IETF or vendor specified extensions.

## The Link to Assets

In {{I-D.draft-palmero-opsawg-ps-almo-00}}, the DMLMO team has built an inventory strucure that describes systems, subsystems and their soft- and hardware components.  They are called assets in the DMLMO YANG models.  Some of the collected telemetry data streams may pertain to quite precisely to these assets, and it may be interesting to see the linkage.  For this reason, there is an optional module, ietf-tlm-philatelist-assets, that augments the DMLMO structure and adds the possibility for an asset to point to a provider or aggregated data stream. FIXME

# YANG-based Telemetry Outlook

Much work has already gone into the area of telemetry, YANG, and even their intersection.  E.g. {{I-D.draft-ietf-opsawg-collected-data-manifest-01}} and {{
I-D.draft-claise-netconf-metadata-for-collection-03}} come to mind.

Even though that work has a solid foundation and shares many or most of the goals with this work, we (the POWEFF team) have not found it easy to apply the above work directly in the practical work we do.  So what we have tried to do is a very pragmatic approach to telemetry data collection the way we see it happening on the ground combined with the benefits of Model Driven Telemetry (MDT), in practice meaning YANG-based with additional YANG-modeled metadata.

Many essential data sources in real world deployments do not support any YANG-based interfaces, and that situation is expected to remain for the forseable future, which is why we find it important to be able to ingest data from free form (often REST-based) interfaces, and then add the necessary rigor on the Collector level.  Then output the datastreams in formats that existing, mature tools can consume directly.

In particular, this draft depends on the mapping of YANG-based structures to the typical TSDB tag-based formats described in {{I-D.draft-kll-yang-label-tsdb-00}}.

For the evolution of the YANG-based telemetry area, we believe this approach, combining pragmatism in the data flow interfaces with rigor regarding the data content, is key.  We would like to make this work fit in with the works of others in the field.

# YANG Modules

## Base types module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-types.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-types@2024-02-14.yang”}

## Provider interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-provider.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-provider@2024-02-14.yang”}

## Collector interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-collector.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-collector@2024-02-14.yang”}

## Aggregator interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-aggregator.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-aggregator@2024-02-14.yang”}

## Assets interface module for Philatelist

~~~~ yang
{::include yang/ietf-tlm-philatelist-assets.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-tlm-philatelist-assets@2024-02-14.yang”}

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


# Changes (to be deleted by RFC Editor)

## From version -00 to -01
- Split YANG modules, some contents going into poweff-specific modules
- Renamed remaining YANG modules from -poweff- to -tlm-philatelist-
- Updated text to reflect new module organization
- Added optional linkage to DMLMO assets

## Version -00
- Initial version.

--- back

# Acknowledgments
{:numbered="false"}

Kristian Larsson has provided invaluable insights, experience and validation of the design.  Many thanks to the entire POWEFF team for their committment, flexibility and hard work behind this.  Hat off to Benoît Claise, who inspires by the extensive work produced in IETF over the years, and in this area in particular.
