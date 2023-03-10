# ECN and WRED Statistics


## Table of Contents 

### Revision  

| Rev | Date     | Author          | Change Description |
|:---:|:--------:|:---------------:|--------------------|
| 0.1 |17/Feb/23 | **Marvell Team**| Initial Version   |

### Scope  


This document provides the high level design for ECN Queue Statistics, WRED Port Statistics in SONiC

### Definitions/Abbreviations



| SONiC ACL Table Type     | Description |
|:---------------:|--------------------|
| __ECN__ | Explicit Congestion Notification|
| __WRED__  | Weighted Random Early Detection|
| __CLI__  | Command Line interface |


### Overview 

The main goal of this feature is to provide better WRED impact visibility in SONiC by providing a mechanism to count the packets that are discarded or ECN-marked due to WRED.

The other goal of this feature is to display these statistics only if the underlying platform supports it (Capability based statistics). Every platform may have unique statistics capabilities, and they change over time, and so it is important for this feature to be capability based.

We will accomplish both the goals by adding support for per-queue ECN marked packets/bytes counters and per-port WRED dropped packets. Existing “queue counters” and “show interface counters detailed” CLI  will be enhanced for displaying these statistics.


### Requirements

1. Support per-queue total ECN marked packets counters
2. Support per-queue total ECN marked byte counters
3. Support per-port WRED dropped packets counters (per-color and total count)
4. STATE_DB can be queried for ECN-WRED stats capability



### Architecture Design 
<p align=center>
<img src="ArchWredEcnStatistics.png" alt="Architectural diagram">
</p>
The architectural interactions happens as described below,

1. Syncd fetches the platform statistics capabilty for ECN-WRED counters from SAI
2. The stats capability will be updated to STATE_DB by Syncd
3. Orchagent checks the ECN-WRED stats capability in STATE_DB
4. Based on the stats capability, Orchagent sets stat-ids to FLEX_COUNTER_DB
    * In case, the platform is capable of ECN-WRED stats, it will be added to the respective QUEUE or PORT groups of the FLEX_COUNTERS_DB
5. Syncd has subscribed to Flex Counter DB and it will set up flex counters.
6. Flex counters periodically query ASIC counters and publishes data to COUNTERS_DB
7. CLI will lookup the Capability at STATE_DB
8. Only the platform supported statistics will be fetched and displayed on the CLI output
 


### High-Level Design 

This section covers the high level design of the ECN-WRED statistics feature. 
This feature extends the existing queue stats and interface stats.

#### Changes in COUNTER_DB

The following new port counters will be added along with existing counters

* COUNTERS_PORT_NAME_MAP:port
    * SAI_PORT_STAT_GREEN_WRED_DROPPED_PACKETS
    * SAI_PORT_STAT_YELLOW_WRED_DROPPED_PACKETS
    * SAI_PORT_STAT_RED_WRED_DROPPED_PACKETS
    * SAI_PORT_STAT_WRED_DROPPED_PACKETS

For every egress queue, the following statistics will be added along with existing queue conters

* COUNTERS_QUEUE_PORT_MAP:queue
    * SAI_QUEUE_STAT_WRED_ECN_MARKED_PACKETS
    * SAI_QUEUE_STAT_WRED_ECN_MARKED_BYTES

#### Changes in FLEX_COUNTER_DB

There are no new flex counter groups introduced for this feature. ECN-WRED stats will use existing flexcounter groups  QUEUE_STAT_COUNTER_FLEX_COUNTER_GROUP and PORT_STAT_COUNTER_FLEX_COUNTER_GROUP. Also the Flexcounter interval will continue to be the same(1s for port counters and 10s for queue counters). 

#### Changes in STATE_DB
State DB will store information about WRED/ECN counter support as per the platform capability. This information will be populated during Syncd startup by checking the platform capability.

```


"WRED_COUNTER_CAPABILITIES": {
    "QUEUE_ECN_MARKED_PKT_COUNTER": {
        "isSupported": "true",
    },
    "QUEUE_ECN_MARKED_BYTE_COUNTER": {
       "isSupported": "true",
    },
    "PORT_WRED_GREEN_DROP_COUNTER": {
        "isSupported": "true",
    },
    "PORT_WRED_YELLOW_DROP_COUNTER": {
        "isSupported": "true",
    },
    "PORT_WRED_RED_DROP_COUNTER": {
        "isSupported": "true",
    },
    "PORT_WRED_TOTAL_DROP_COUNTER": {
        "isSupported": "true",
    },
}

```

### CLI Changes

The CLI displays the ECN-WRED statistics only if the capability is supported by the platform. It gets the capability from STATE_DB and queries only the supported statistics from COUNTER_DB.
Following are the CLI usages,

* ECN Marked Statistics
    * Cleared on user request (clear queue) :  clear queuecounters
    * Used by CLI (show queue) : show queue counters [interface-name]
* WRED drop statistics
    * Cleared on user request (clear counters) :  clear counters
    * Used by CLI (show queue) : show interfaces counters detailed \<interface-name\>

#### show queue counters on a ECN stats supported platform

```
sonic-dut:~# show queue counters Ethernet16
      Port    TxQ    Counter/pkts    Counter/bytes    Drop/pkts    Drop/bytes   EcnMarked/pkts EcnMarked/bytes
----------  -----  --------------  ---------------  -----------  ------------   -------------- ---------------
Ethernet16    UC0               0                0            0             0               0               0
Ethernet16    UC1               1              120            0             0               1             120
Ethernet16    UC2               0                0            0             0               0               0
Ethernet16    UC3               0                0            0             0               0               0
Ethernet16    UC4               0                0            0             0               0               0
Ethernet16    UC5               0                0            0             0               0               0
Ethernet16    UC6               0                0            0             0               0               0
Ethernet16    UC7               0                0            0             0               0               0
```
#### show queue counters on a platform which does not support ECN statistics
```
sonic-dut:~# show queue counters Ethernet16
      Port    TxQ    Counter/pkts    Counter/bytes    Drop/pkts    Drop/bytes
----------  -----  --------------  ---------------  -----------  ------------
Ethernet16    UC0               0                0            0             0
Ethernet16    UC1               1              120            0             0
Ethernet16    UC2               0                0            0             0
Ethernet16    UC3               0                0            0             0
Ethernet16    UC4               0                0            0             0
Ethernet16    UC5               0                0            0             0
Ethernet16    UC6               0                0            0             0
Ethernet16    UC7               0                0            0             0

```
#### show interface counters on a WRED discard counters supported platform
```
root@sonic-dut:~# show interfaces counters detailed Ethernet8
Packets Received 64 Octets..................... 0
Packets Received 65-127 Octets................. 2
Packets Received 128-255 Octets................ 0
Packets Received 256-511 Octets................ 0
Packets Received 512-1023 Octets............... 0
Packets Received 1024-1518 Octets.............. 0
Packets Received 1519-2047 Octets.............. 0
Packets Received 2048-4095 Octets.............. 0
Packets Received 4096-9216 Octets.............. 0
Packets Received 9217-16383 Octets............. 0

Total Packets Received Without Errors.......... 2
Unicast Packets Received....................... 0
Multicast Packets Received..................... 2
Broadcast Packets Received..................... 0

Jabbers Received............................... N/A
Fragments Received............................. N/A
Undersize Received............................. 0
Overruns Received.............................. 0

Packets Transmitted 64 Octets.................. 32,893
Packets Transmitted 65-127 Octets.............. 16,449
Packets Transmitted 128-255 Octets............. 3
Packets Transmitted 256-511 Octets............. 2,387
Packets Transmitted 512-1023 Octets............ 0
Packets Transmitted 1024-1518 Octets........... 0
Packets Transmitted 1519-2047 Octets........... 0
Packets Transmitted 2048-4095 Octets........... 0
Packets Transmitted 4096-9216 Octets........... 0
Packets Transmitted 9217-16383 Octets.......... 0

Total Packets Transmitted Successfully......... 51,732
Unicast Packets Transmitted.................... 0
Multicast Packets Transmitted.................. 18,840
Broadcast Packets Transmitted.................. 32,892
Time Since Counters Last Cleared............... None

WRED Green Discarded Packets................... 1
WRED Yellow Discarded Packets.................. 3
WRED RED Discarded Packets..................... 10
WRED Total Discarded Packets................... 14

```

#### show interface counters on a platform which does not support WRED discard counters
```
root@sonic-dut:~# show interfaces counters detailed Ethernet8
Packets Received 64 Octets..................... 0
Packets Received 65-127 Octets................. 2
Packets Received 128-255 Octets................ 0
Packets Received 256-511 Octets................ 0
Packets Received 512-1023 Octets............... 0
Packets Received 1024-1518 Octets.............. 0
Packets Received 1519-2047 Octets.............. 0
Packets Received 2048-4095 Octets.............. 0
Packets Received 4096-9216 Octets.............. 0
Packets Received 9217-16383 Octets............. 0

Total Packets Received Without Errors.......... 2
Unicast Packets Received....................... 0
Multicast Packets Received..................... 2
Broadcast Packets Received..................... 0

Jabbers Received............................... N/A
Fragments Received............................. N/A
Undersize Received............................. 0
Overruns Received.............................. 0

Packets Transmitted 64 Octets.................. 32,893
Packets Transmitted 65-127 Octets.............. 16,449
Packets Transmitted 128-255 Octets............. 3
Packets Transmitted 256-511 Octets............. 2,387
Packets Transmitted 512-1023 Octets............ 0
Packets Transmitted 1024-1518 Octets........... 0
Packets Transmitted 1519-2047 Octets........... 0
Packets Transmitted 2048-4095 Octets........... 0
Packets Transmitted 4096-9216 Octets........... 0
Packets Transmitted 9217-16383 Octets.......... 0

Total Packets Transmitted Successfully......... 51,732
Unicast Packets Transmitted.................... 0
Multicast Packets Transmitted.................. 18,840
Broadcast Packets Transmitted.................. 32,892
Time Since Counters Last Cleared............... None
```

The below sequence diagram explains the "show queue counters" interaction among CLI, STATE_DB and COUNTERS_DB,
<p align=center>
<img src="queue_counter_cli_flow.png" alt="queue CLI interactions">
</p>

The below sequence diagram explains the "show interface counters detailed" interaction among CLI, STATE_DB and COUNTERS_DB,
<p align=center>
<img src="port_counter_cli_flow.png" alt="port CLI interactions">
</p>

### Changes in Orchagent
Orchagent checks the ECN-WRED stats capability in STATE_DB during ports initialization, then adds the ECN-WRED statistics for polling to Flexcounter DB.

### Changes in Syncd
Syncd gets the counter capability during the startup and updates the STATE_DB with supported ECN-WRED statistics.
The interactions are explained below,

#### STATE_DB interactions with Syncd
<p align=center>
<img src="stateDB_syncd_interactions.png" alt="StateDB syncd interactions">
</p>

### SAI API 
Following SAI APIs and stats are used for this feature,
* sai_query_stats_capability() is used to query the stats capability
    * For queue statistics, the object type must be SAI_OBJECT_TYPE_QUEUE
    * For port statistics,  the object type must be SAI_OBJECT_TYPE_PORT
* sai_queue_api_t APIs  are used for counter get and clear
    * get_queue_stats()
    * clear_queue_stats()
    * get_port_stats()
    * clear_port_stats()
    * clear_port_all_stats()
* SAI counters,
    * SAI_QUEUE_STAT_WRED_ECN_MARKED_PACKETS
    * SAI_QUEUE_STAT_WRED_ECN_MARKED_BYTES
    * SAI_PORT_STAT_GREEN_WRED_DROPPED_PACKETS
    * SAI_PORT_STAT_YELLOW_WRED_DROPPED_PACKETS
    * SAI_PORT_STAT_RED_WRED_DROPPED_PACKETS
    * SAI_PORT_STAT_WRED_DROPPED_PACKETS
It uses the sai_queue_api_t APIs get_queue_stats() and clear_queue_stats(). 


There are no new SAI APIs required for this feature.

### Configuration and management 
No new CLI or datamodel changes are introduced.  
No new fields are added to the Config DB.

### Manifest (if the feature is an Application Extension)
Not applicable?

### CLI/YANG model Enhancements
Not applicable

	
### Warmboot and Fastboot Design Impact  
There is no impact to existing counters on warmboot or fastboot.
New statistics will get introduced only in Coldboot.


### Restrictions/Limitations  

* Existing FlexCounters limitations if any

* This fetaure is limited only to the platforms that support these capabilities

### Testing Requirements/Design  

#### Unit Test cases  
- Enable ECN-marking, Send respective traffic and verify the ECN-marked counters in queue stats
- Enable WRED discard, Send respective traffic and verify the WRED drop counters in port stats
- Disable ECN-marking and WRED discard, Send traffic and verify these counters are not incrementing
- On ECN-WRED stats non-supported platforms,
    - Verify that CLI does not show the respective Headers in queue stats
    - Verify that CLI does not show the respective rows in port stats

#### System Test cases
* Enhance existing sonic-mgmt(PTF) ecn_wred testcase to verify the statistics on supported platforms