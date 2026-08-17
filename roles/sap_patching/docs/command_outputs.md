# Examples of command outputs used for parsing results

This file shows examples of some commands that are executed during patching and need to be parsed to detect patching results.
Outputs were captured on two lab systems: `S02` (ABAP/NetWeaver) and `H02` (HANA database).
> NOTE: Command outputs were captured 08/2026.


## Command: `hdbnsutil -sr_state -sapcontrol=1`

### HSR Disabled
```bash
h02adm@h02hana0:/usr/sap/H02/HDB00> hdbnsutil -sr_state -sapcontrol=1
SAPCONTROL-OK: <begin>
online=true
mode=none
SAPCONTROL-OK: <end>
done.
```

### HSR Enabled and Active on 3 tier configuration
```bash
h02adm@h02hana0:/usr/sap/H02/HDB00> hdbnsutil -sr_state -sapcontrol=1
SAPCONTROL-OK: <begin>
online=true
mode=sync
operation mode=logreplay
site id=1
site name=DC01
isSource=false
isConsumer=true
hasConsumers=false
isTakeoverActive=false
isPrimarySuspended=false
isTimetravelEnabled=false
replayMode=auto
active primary site=4
primary masters=h02hana1
mapping/h02hana0=DC04/h02hana1
mapping/h02hana0=DC03/h02hana3
mapping/h02hana0=DC02/h02hana2
mapping/h02hana0=DC01/h02hana0
siteTier/DC04=1
siteTier/DC02=2
siteTier/DC03=3
siteTier/DC01=2
siteReplicationMode/DC04=primary
siteReplicationMode/DC02=async
siteReplicationMode/DC03=sync
siteReplicationMode/DC01=sync
siteOperationMode/DC04=primary
siteOperationMode/DC02=logreplay
siteOperationMode/DC03=logreplay
siteOperationMode/DC01=logreplay
siteMapping/DC04=DC02
siteMapping/DC02=DC03
siteMapping/DC04=DC01
hintBasedRoutingSite=
SAPCONTROL-OK: <end>
done.
Performing Final Memory Release with 12 threads.
Finished Final Memory Release successfuly.
```

### HSR Enabled with system down on 2 tier configuration
```bash
h02adm@h02hana1:/usr/sap/H02/HDB00> hdbnsutil -sr_state -sapcontrol=1
SAPCONTROL-OK: <begin>
online=false
mode=sync
operation mode=unknown
site id=2
site name=DC02
isSource=unknown
isConsumer=true
hasConsumers=unknown
isTakeoverActive=false
isPrimarySuspended=false
isTimetravelEnabled=false
replayMode=auto
active primary site=1
primary masters=h02hana0
SAPCONTROL-OK: <end>
done.
```

### HSR state of primary after relocation with parameter `AUTOMATED_REGISTER=False`
#### Before registering with `-sr_register`
```bash
h02adm@h02hana1:/usr/sap/H02/HDB90> hdbnsutil -sr_state -sapcontrol=1
SAPCONTROL-OK: <begin>
online=false
mode=primary
operation mode=unknown
site id=2
site name=DC02
isSource=unknown
isConsumer=false
hasConsumers=unknown
isTakeoverActive=false
isPrimarySuspended=unknown
SAPCONTROL-OK: <end>
done.
```

#### After registering with `-sr_register`, but still offline
```bash
h02adm@h02hana1:/usr/sap/H02/HDB90> hdbnsutil -sr_state -sapcontrol=1
SAPCONTROL-OK: <begin>
online=false
mode=sync
operation mode=unknown
site id=2
site name=DC02
isSource=unknown
isConsumer=true
hasConsumers=unknown
isTakeoverActive=false
isPrimarySuspended=unknown
isTimetravelEnabled=false
replayMode=auto
active primary site=1
primary masters=h02hana0
SAPCONTROL-OK: <end>
done.
```


## Command: `HDBSettings.sh systemReplicationStatus.py`

This script does not produce expected return codes, so we must be careful with handling.<br>
**NOTE: Current return codes can be extracted from the `systemReplicationStatus.py` script itself!**

```python
class ServiceStatus:
    NoHSR        = 10
    Error        = 11
    Unknown      = 12
    Initializing = 13
    Syncing      = 14
    Active       = 15
    __strmap={NoHSR:'System Replication not active', Error:'ERROR', Unknown:'UNKNOWN', Initializing:'INITIALIZING', Syncing:'SYNCING', Active:'ACTIVE'}
```


### HSR Enabled and Active (RC 15)
```bash
h02adm@h02hana0:/usr/sap/H02/HDB90> set -o pipefail && source ~/.profile && HDBSettings.sh systemReplicationStatus.py
|Database |Host     |Port  |Service Name |Volume ID |Site ID |Site Name |Secondary |Secondary |Secondary |Secondary |Secondary     |Replication |Replication |Replication    |Secondary    |
|         |         |      |             |          |        |          |Host      |Port      |Site ID   |Site Name |Active Status |Mode        |Status      |Status Details |Fully Synced |
|-------- |-------- |----- |------------ |--------- |------- |--------- |--------- |--------- |--------- |--------- |------------- |----------- |----------- |-------------- |------------ |
|SYSTEMDB |h02hana0 |39001 |nameserver   |        1 |      1 |DC01      |h02hana1  |    39001 |        2 |DC02      |YES           |SYNC        |ACTIVE      |               |        True |
|H02      |h02hana0 |39007 |xsengine     |        2 |      1 |DC01      |h02hana1  |    39007 |        2 |DC02      |YES           |SYNC        |ACTIVE      |               |        True |
|H02      |h02hana0 |39003 |indexserver  |        3 |      1 |DC01      |h02hana1  |    39003 |        2 |DC02      |YES           |SYNC        |ACTIVE      |               |        True |

status system replication site "2": ACTIVE
overall system replication status: ACTIVE

Local System Replication State
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

mode: PRIMARY
site id: 1
site name: DC01
```

### HSR Disabled (RC 10)
```bash
h02adm@h02hana1:/usr/sap/H02/HDB90> set -o pipefail && source ~/.profile && HDBSettings.sh systemReplicationStatus.py
this system is either not running or not primary system replication site

Local System Replication State
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

mode: SYNC
site id: 2
site name: DC02
active primary site: 1
primary masters: h02hana0
```


## Command: `HDBSettings.sh systemReplicationStatus.py --sapcontrol=1`

### HSR Enabled and Active (RC 15)
```bash
h02adm@h02hana0:/usr/sap/H02/HDB90> HDBSettings.sh systemReplicationStatus.py --sapcontrol=1
SAPCONTROL-OK: <begin>
service/h02hana0/39001/DATABASE=SYSTEMDB
service/h02hana0/39001/HOST=h02hana0
service/h02hana0/39001/PORT=39001
service/h02hana0/39001/SERVICE_NAME=nameserver
service/h02hana0/39001/VOLUME_ID=1
service/h02hana0/39001/SITE_ID=1
service/h02hana0/39001/SITE_NAME=DC01
service/h02hana0/39001/SECONDARY_HOST=h02hana1
service/h02hana0/39001/SECONDARY_PORT=39001
service/h02hana0/39001/SECONDARY_SITE_ID=2
service/h02hana0/39001/SECONDARY_SITE_NAME=DC02
service/h02hana0/39001/SECONDARY_ACTIVE_STATUS=YES
service/h02hana0/39001/SECONDARY_CONNECT_TIME=2026-08-11 07:49:33.683598
service/h02hana0/39001/SECONDARY_RECONNECT_COUNT=0
service/h02hana0/39001/SECONDARY_FAILOVER_COUNT=0
service/h02hana0/39001/REPLICATION_MODE=SYNC
service/h02hana0/39001/OPERATION_MODE=logreplay
service/h02hana0/39001/REPLICATION_STATUS=ACTIVE
service/h02hana0/39001/REPLICATION_STATUS_DETAILS=
service/h02hana0/39001/FULL_SYNC=DISABLED
service/h02hana0/39001/LAST_LOG_POSITION=59600896
service/h02hana0/39001/LAST_LOG_POSITION_TIME=2026-08-11 08:01:29.411210
service/h02hana0/39001/LAST_SAVEPOINT_VERSION=13
service/h02hana0/39001/LAST_SAVEPOINT_LOG_POSITION=59600066
service/h02hana0/39001/LAST_SAVEPOINT_START_TIME=2026-08-11 08:00:39.540581
service/h02hana0/39001/SHIPPED_LOG_POSITION=59600896
service/h02hana0/39001/SHIPPED_LOG_POSITION_TIME=2026-08-11 08:01:29.411210
service/h02hana0/39001/SHIPPED_LOG_BUFFERS_COUNT=1360
service/h02hana0/39001/SHIPPED_LOG_BUFFERS_SIZE=60694528
service/h02hana0/39001/REPLAYED_LOG_POSITION=59600896
service/h02hana0/39001/REPLAYED_LOG_POSITION_TIME=2026-08-11 08:01:29.411210
service/h02hana0/39001/SHIPPED_LOG_BUFFERS_DURATION=2503562
service/h02hana0/39001/SHIPPED_SAVEPOINT_VERSION=7
service/h02hana0/39001/SHIPPED_SAVEPOINT_LOG_POSITION=58652675
service/h02hana0/39001/SHIPPED_SAVEPOINT_START_TIME=2026-08-11 07:49:34.457339
service/h02hana0/39001/SHIPPED_FULL_REPLICA_COUNT=1
service/h02hana0/39001/SHIPPED_FULL_REPLICA_SIZE=3742232576
service/h02hana0/39001/SHIPPED_FULL_REPLICA_DURATION=53201179
service/h02hana0/39001/SHIPPED_LAST_FULL_REPLICA_SIZE=3742232576
service/h02hana0/39001/SHIPPED_LAST_FULL_REPLICA_START_TIME=2026-08-11 07:49:34.457339
service/h02hana0/39001/SHIPPED_LAST_FULL_REPLICA_END_TIME=2026-08-11 07:50:27.658518
service/h02hana0/39001/SHIPPED_DELTA_REPLICA_COUNT=0
service/h02hana0/39001/SHIPPED_DELTA_REPLICA_SIZE=0
service/h02hana0/39001/SHIPPED_DELTA_REPLICA_DURATION=0
service/h02hana0/39001/SHIPPED_LAST_DELTA_REPLICA_SIZE=0
service/h02hana0/39001/SHIPPED_LAST_DELTA_REPLICA_START_TIME=-
service/h02hana0/39001/SHIPPED_LAST_DELTA_REPLICA_END_TIME=-
service/h02hana0/39001/RESET_COUNT=0
service/h02hana0/39001/LAST_RESET_TIME=2026-08-11 07:48:54.651241
service/h02hana0/39001/CREATION_TIME=2026-08-11 07:48:54.651241
service/h02hana0/39001/SECONDARY_FULLY_SYNCED=True
service/h02hana0/39007/DATABASE=H02
service/h02hana0/39007/HOST=h02hana0
service/h02hana0/39007/PORT=39007
service/h02hana0/39007/SERVICE_NAME=xsengine
service/h02hana0/39007/VOLUME_ID=2
service/h02hana0/39007/SITE_ID=1
service/h02hana0/39007/SITE_NAME=DC01
service/h02hana0/39007/SECONDARY_HOST=h02hana1
service/h02hana0/39007/SECONDARY_PORT=39007
service/h02hana0/39007/SECONDARY_SITE_ID=2
service/h02hana0/39007/SECONDARY_SITE_NAME=DC02
service/h02hana0/39007/SECONDARY_ACTIVE_STATUS=YES
service/h02hana0/39007/SECONDARY_CONNECT_TIME=2026-08-11 07:49:59.018176
service/h02hana0/39007/SECONDARY_RECONNECT_COUNT=0
service/h02hana0/39007/SECONDARY_FAILOVER_COUNT=0
service/h02hana0/39007/REPLICATION_MODE=SYNC
service/h02hana0/39007/OPERATION_MODE=logreplay
service/h02hana0/39007/REPLICATION_STATUS=ACTIVE
service/h02hana0/39007/REPLICATION_STATUS_DETAILS=
service/h02hana0/39007/FULL_SYNC=DISABLED
service/h02hana0/39007/LAST_LOG_POSITION=25600
service/h02hana0/39007/LAST_LOG_POSITION_TIME=2026-08-11 08:01:35.325112
service/h02hana0/39007/LAST_SAVEPOINT_VERSION=11
service/h02hana0/39007/LAST_SAVEPOINT_LOG_POSITION=24579
service/h02hana0/39007/LAST_SAVEPOINT_START_TIME=2026-08-11 08:00:39.607139
service/h02hana0/39007/SHIPPED_LOG_POSITION=25600
service/h02hana0/39007/SHIPPED_LOG_POSITION_TIME=2026-08-11 08:01:35.325112
service/h02hana0/39007/SHIPPED_LOG_BUFFERS_COUNT=216
service/h02hana0/39007/SHIPPED_LOG_BUFFERS_SIZE=884736
service/h02hana0/39007/REPLAYED_LOG_POSITION=25540
service/h02hana0/39007/REPLAYED_LOG_POSITION_TIME=2026-08-11 08:01:35.325112
service/h02hana0/39007/SHIPPED_LOG_BUFFERS_DURATION=388840
service/h02hana0/39007/SHIPPED_SAVEPOINT_VERSION=5
service/h02hana0/39007/SHIPPED_SAVEPOINT_LOG_POSITION=11906
service/h02hana0/39007/SHIPPED_SAVEPOINT_START_TIME=2026-08-11 07:50:02.284410
service/h02hana0/39007/SHIPPED_FULL_REPLICA_COUNT=1
service/h02hana0/39007/SHIPPED_FULL_REPLICA_SIZE=83906560
service/h02hana0/39007/SHIPPED_FULL_REPLICA_DURATION=1937655
service/h02hana0/39007/SHIPPED_LAST_FULL_REPLICA_SIZE=83906560
service/h02hana0/39007/SHIPPED_LAST_FULL_REPLICA_START_TIME=2026-08-11 07:50:02.284410
service/h02hana0/39007/SHIPPED_LAST_FULL_REPLICA_END_TIME=2026-08-11 07:50:04.222065
service/h02hana0/39007/SHIPPED_DELTA_REPLICA_COUNT=0
service/h02hana0/39007/SHIPPED_DELTA_REPLICA_SIZE=0
service/h02hana0/39007/SHIPPED_DELTA_REPLICA_DURATION=0
service/h02hana0/39007/SHIPPED_LAST_DELTA_REPLICA_SIZE=0
service/h02hana0/39007/SHIPPED_LAST_DELTA_REPLICA_START_TIME=-
service/h02hana0/39007/SHIPPED_LAST_DELTA_REPLICA_END_TIME=-
service/h02hana0/39007/RESET_COUNT=0
service/h02hana0/39007/LAST_RESET_TIME=2026-08-11 07:48:54.646604
service/h02hana0/39007/CREATION_TIME=2026-08-11 07:48:54.646604
service/h02hana0/39007/SECONDARY_FULLY_SYNCED=True
service/h02hana0/39003/DATABASE=H02
service/h02hana0/39003/HOST=h02hana0
service/h02hana0/39003/PORT=39003
service/h02hana0/39003/SERVICE_NAME=indexserver
service/h02hana0/39003/VOLUME_ID=3
service/h02hana0/39003/SITE_ID=1
service/h02hana0/39003/SITE_NAME=DC01
service/h02hana0/39003/SECONDARY_HOST=h02hana1
service/h02hana0/39003/SECONDARY_PORT=39003
service/h02hana0/39003/SECONDARY_SITE_ID=2
service/h02hana0/39003/SECONDARY_SITE_NAME=DC02
service/h02hana0/39003/SECONDARY_ACTIVE_STATUS=YES
service/h02hana0/39003/SECONDARY_CONNECT_TIME=2026-08-11 07:49:57.783179
service/h02hana0/39003/SECONDARY_RECONNECT_COUNT=0
service/h02hana0/39003/SECONDARY_FAILOVER_COUNT=0
service/h02hana0/39003/REPLICATION_MODE=SYNC
service/h02hana0/39003/OPERATION_MODE=logreplay
service/h02hana0/39003/REPLICATION_STATUS=ACTIVE
service/h02hana0/39003/REPLICATION_STATUS_DETAILS=
service/h02hana0/39003/FULL_SYNC=DISABLED
service/h02hana0/39003/LAST_LOG_POSITION=58459072
service/h02hana0/39003/LAST_LOG_POSITION_TIME=2026-08-11 08:01:35.326648
service/h02hana0/39003/LAST_SAVEPOINT_VERSION=12
service/h02hana0/39003/LAST_SAVEPOINT_LOG_POSITION=58454850
service/h02hana0/39003/LAST_SAVEPOINT_START_TIME=2026-08-11 08:00:50.397561
service/h02hana0/39003/SHIPPED_LOG_POSITION=58459072
service/h02hana0/39003/SHIPPED_LOG_POSITION_TIME=2026-08-11 08:01:35.326648
service/h02hana0/39003/SHIPPED_LOG_BUFFERS_COUNT=6270
service/h02hana0/39003/SHIPPED_LOG_BUFFERS_SIZE=911372288
service/h02hana0/39003/REPLAYED_LOG_POSITION=58459072
service/h02hana0/39003/REPLAYED_LOG_POSITION_TIME=2026-08-11 08:01:35.326648
service/h02hana0/39003/SHIPPED_LOG_BUFFERS_DURATION=16669385
service/h02hana0/39003/SHIPPED_SAVEPOINT_VERSION=6
service/h02hana0/39003/SHIPPED_SAVEPOINT_LOG_POSITION=44224066
service/h02hana0/39003/SHIPPED_SAVEPOINT_START_TIME=2026-08-11 07:50:01.011097
service/h02hana0/39003/SHIPPED_FULL_REPLICA_COUNT=1
service/h02hana0/39003/SHIPPED_FULL_REPLICA_SIZE=2936729600
service/h02hana0/39003/SHIPPED_FULL_REPLICA_DURATION=34643583
service/h02hana0/39003/SHIPPED_LAST_FULL_REPLICA_SIZE=2936729600
service/h02hana0/39003/SHIPPED_LAST_FULL_REPLICA_START_TIME=2026-08-11 07:50:01.011097
service/h02hana0/39003/SHIPPED_LAST_FULL_REPLICA_END_TIME=2026-08-11 07:50:35.654680
service/h02hana0/39003/SHIPPED_DELTA_REPLICA_COUNT=0
service/h02hana0/39003/SHIPPED_DELTA_REPLICA_SIZE=0
service/h02hana0/39003/SHIPPED_DELTA_REPLICA_DURATION=0
service/h02hana0/39003/SHIPPED_LAST_DELTA_REPLICA_SIZE=0
service/h02hana0/39003/SHIPPED_LAST_DELTA_REPLICA_START_TIME=-
service/h02hana0/39003/SHIPPED_LAST_DELTA_REPLICA_END_TIME=-
service/h02hana0/39003/RESET_COUNT=0
service/h02hana0/39003/LAST_RESET_TIME=2026-08-11 07:48:54.647505
service/h02hana0/39003/CREATION_TIME=2026-08-11 07:48:54.647505
service/h02hana0/39003/SECONDARY_FULLY_SYNCED=True
site/2/SITE_NAME=DC02
site/2/SOURCE_SITE_ID=1
site/2/REPLICATION_MODE=SYNC
site/2/REPLICATION_STATUS=ACTIVE
overall_replication_status=ACTIVE
site/1/REPLICATION_MODE=PRIMARY
site/1/SITE_NAME=DC01
local_site_id=1
site/2/SECONDARY_FULLY_SYNCED=True
overall_in_sync_status=SYSTEM_IN_SYNC
SAPCONTROL-OK: <end>
```


## Command: `sapcontrol -function UpdateSystem 900 3600` (RC 1)

> NOTE: This warning is applicable for both RHEL and SLES, regardless if High Availability cluster is correctly configured according to SAP Note.
> This command output was captured on fully working cluster on SLES 16.0, with `sap-suse-cluster-connector` package installed and configured.

```bash
s02pas:s02adm 1002> sapcontrol -nr 01 -function UpdateSystem 900 3600

10.08.2026 14:02:31
UpdateSystem
FAIL: RKS Warning(s):

HA Product detected: SUSE Linux Enterprise Server for SAP Applications 16.0. Please refer to SAP note 2077934 and HA product specific notes to ensure the system fulfills the requirements for this kernel update.
```


## Command: `sapcontrol -function UpdateSystem 900 3600 1` (RC 0)

```bash
s02pas:s02adm 1006> sapcontrol -nr 01 -function UpdateSystem 900 3600 1

10.08.2026 13:54:08
UpdateSystem
OK
```


## Command: `sapcontrol -function GetSystemUpdateList` (RC 0)

> **NOTE:** This command is currently not used, because Rolling Kernel Switch is generally very long running procedure.
> This role will not wait for its completion as it can take a long time depending on the system load and timeout settings.
> The command also returns RC 0 regardless of the actual state, so the progress can only be determined by parsing
> the `dispstatus` column (GREEN = finished, YELLOW = pending or in progress, RED = failed).

```bash
s02pas:s02adm 1001> sapcontrol -nr 01 -function GetSystemUpdateList

10.08.2026 13:56:26
GetSystemUpdateList
OK
hostname, instanceNr, status, starttime, endtime, dispstatus
s02ers-ha, 10, Started, 2026 08 10 13:54:10, 2026 08 10 13:54:48, GREEN
s02ascs-ha, 0, Restarting, 2026 08 10 13:54:48, , YELLOW
s02aas, 11, Scheduled, , , YELLOW
s02pas, 1, Scheduled, , , YELLOW
```


## Command: `hdblcm --version` (RC 0)

```bash
h02adm@h02hana1:/hana/shared/H02/hdblcm> ./hdblcm --version
SAP HANA Lifecycle Management (hdblcm)
Copyright © 2000-2026 by SAP SE
Version: 2.8.102 (git hash: 41c53164a806)
GitHash: 41c53164a806
Perl: v5.40.4
```

## Command: `ps -U <sidadm> -o cmd --no-header`

### User does not exist (RC 1)
```bash
s02ascs:~ ps -U h02adm -o cmd --no-header
error: user name does not exist

Usage:
 ps [options]

 Try 'ps --help <simple|list|output|threads|misc|all>'
  or 'ps --help <s|l|o|t|m|a>'
 for additional help text.

For more details see ps(1).
```

### User exists but processes are not running (RC 1)
```bash
s02hana0:~ ps -U s02adm -o cmd --no-header
s02hana0:~ 
```

### User exists and processes are running (RC 0)

```bash
s02pas:~ ps -U s02adm -o cmd --no-header
/usr/sap/S02/D01/exe/sapstartsrv pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas -cleanipc 01
sapstart pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
dw.sapS02_D01 pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
ig.sapS02_D01 -mode=profile pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
/usr/sap/S02/D01/exe/igsmux_mt -mode=profile -restartcount=0 -wdpid=849645 pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
/usr/sap/S02/D01/exe/igspw_mt -mode=profile -no=0 -restartcount=0 -wdpid=849645 pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
/usr/sap/S02/D01/exe/igspw_mt -mode=profile -no=1 -restartcount=0 -wdpid=849645 pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
gwrd -dp pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
icman -attach pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
dw.sapS02_D01 pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
dw.sapS02_D01 pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
... one 'dw.sapS02_D01' line per configured work process ...
dw.sapS02_D01 pf=/usr/sap/S02/SYS/profile/S02_D01_s02pas
```
