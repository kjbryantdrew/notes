```bash
GGSCI (node43) 1> info all

Program     Status      Group       Lag at Chkpt  Time Since Chkpt

MANAGER     RUNNING
REPLICAT    RUNNING     REENS       00:00:00      00:00:04


GGSCI (node43) 2> info reens

REPLICAT   REENS     Last Started 2021-07-28 15:43   Status RUNNING
Checkpoint Lag       00:00:00 (updated 00:00:05 ago)
Process ID           2784
Log Read Checkpoint  File /home/ogg12/dirdat/de000000020
                     2021-07-28 16:32:27.913562  RBA 4229203


GGSCI (node43) 3> stop reens

Sending STOP request to REPLICAT REENS ...
Request processed.


GGSCI (node43) 4> alter reens extrba 0

2021-07-28 16:33:01  INFO    OGG-06594  Replicat REENS has been altered. Even the start up position might be updated, duplicate suppression remains active in next startup. T
o override duplicate suppression, start REENS with NOFILTERDUPTRANSACTIONS option.

REPLICAT altered.


GGSCI (node43) 5> start reens NOFILTERDUPTRANSACTIONS

Sending START request to MANAGER ...
REPLICAT REENS starting
```

