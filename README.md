## Solar Logger

This is a datalogger for a solar inverter. Tried to make it so it can be used with any other inverter, if you create a file with all the register and status constants


![Dashboard](dashboard.png)

I'm certain that by not having access to other inverters and only knowing mine, I did throw in some bias into the code. However, hope it isn't bad

So for now it is configured for a Huawei's Sun2000 usl0 version, but probably should run with other residential models. For commercial models, you would need to update the register map (in Huawei.py).



This code was inspired on a series of other repositories to as a guideline to create the current datalogger. Including portions from https://github.com/meltaxa/solariot/

### Basics

You will need these python packages pymodbus, influxdb, pytz, and influxdb (the database itself) and grafana

Also included the solcast folder from basking-in-the-sun2000/Radiation, hopefully FidFenix will incorporate the changes and keep it current

For grafana you will need these plugins:
- agenty-flowcharting-panel
- blackmirror1-singlestat-math-panel
- graph
- yesoreyeram-boomtable-panel


### Description of files

- write_solar populates the db with expected production values. These are daily average for the month in kWh. The values should account for your system, layout, shadowing, depreciation, etc, throughout the life of the system. This was used before solcast, but gives you an idea of how your production is going

- scanner should allow to poll the inverterer for valid registers

- scanner2 will display the values of the registers listed in regs

- fill_solcast_measurements sends the recorded data from the inverter to solcast

- modbustcp utility for modbus

- last_adjustment allows to adjust the total energy balance (exported - from_grid + adj). Just fill the values (date and deviation in kWh)
	- There is a adjustment in the config file (extra_load) to allow you to have a daily adjustment (this isn't used for hourly, but only for the daily values)

- Huawei contains the inverter's registers, status constants

- fill_daily_blanks takes the data from the inverter (influxdb) and writes daily summaries. It now uses the 5m data (logger_ds), unlike the equivalent in the main file use 30s data from that day

- config.default has the constant values for your site. It copies this onto config.py if you don't have one

- main is the code that runs all the time to gather and store data onto the database (influxdb)

- supla_api is the api to the supla monitoring service (https://www.supla.org/en/)

- utils is a library with support utilities

- solar.service.app has the values to set the logger as a service

- setting_up.md gives a detailed install to get up and running

#### Grafana files:
- dashboard.json is the main dashboard for the logger
- solar.json is the detail view dashboard for the logger
- status.json shows the historical values of the status registers
- data_sources.json has information to setup the data sources for grafana


### Setup

After installing influxdb, grafana, the required python packages and grafana plugins, you might want to change the `config.py` values to suit your installation.
You need to connect to the data sources in grafana. Influxdb creates three dbs: logger, logger_ds and logger_lt (or the names you used if you changed those).

### Operation

Running shouldn't need much if all the requirements are satisfied
- cd to the directory with the code
- `python3 main.py`

If everything is working, you can get it running as a service and will autostart after booting. See [setting_up.md](setting up.md)

### Notes about raspberry pi 4

Been running it off a raspberry pi 4 and has behaved well. Influx is a bit demanding, up to 40% of the cpu. Did eventually ran into trouble, since wal files had grown, and influxdb couldn't run (see below). You might want to turn off the vnc and desktop  to conserve resources. 

Also, my sd card started acting up (changes weren't saved to the drive). Ended up using an ssd, which hopefully will be enough for the next 25 years. On the positive side, it allowed me to do a clean install and test everything from scatch. Had to install it on the usb2. Wouldn't boot if it was on the usb3, maybe the case I'm using.

Did add this to the `/boot/cmdline.txt`, just to make sure it checked the drives at bootup
	fsck.mode=auto fsck.repair=yes

You should have the log file at `~/var/log/solar/solar.log` (unless you changed the location). but you need to create the folde first (mkdir)

You could also try something that moves a high frequency write directory (`/var/log`) to ram 
https://github.com/azlux/log2ram



### Other notes

It uses solcast.com, so you need to create an account (free) and a rooftop site. It pulls the forecast several times during the day, but only sends the measurements at midnight. 

Also updates the daily summary data at midnight. In case you missed that time, there are tools that allow you to send data to solcast or update the daily db

After tuning (sending your data), they claim I'm getting 0.97 correlation of the data. There is a adjustment factor in the Energy pane of grafana, for 0.98. With use it will improve (solcast tuning has improved the forecasts a lot!). The time period is hardwired to 5 minutes, not sure it makes much difference if you use 10m or even 30m. However, since we are only sending data for the active production once a day, it sounded as a reasonable value.

In grafana, there are several limits that show values in different colors. You can adjust these to suit your site. The solcast forecasts include all 3 sets. These are the regular, the 10th percentile (low scenario), and 90th percentile (high scenario) estimates. If you rather, you can delete the ones that don't suit you.

My system is new, so I have panes that show the behaviour throughout the day. If yours is older or don't care, you can swap the panes for daily behaviour instead. Just scroll down and move the panes up, or make your own.


### About Influx

Just had an issue with influxdb, by which it wouldn't start. Somehow it didn't worked as expected, and started leaving around a huge amount of files. Had he logger offline a couple of days, trying to figure a way to fix it. 

Got the 4gb raspberry pi, thinking i would need it. However, always thought that would suffice for this use. Eventually moved it to my computer, and ran influxd from there. Kept getting a out of memory error and later a too many files open error.

Not sure if this will sufice, but maybe these changes might help

Add to `/etc/security/limits.d/influxd.conf` and `/etc/security/limits.conf`:

    *                hard    nofile          40962
    *                soft    nofile          40962 

Set these values to the influxdb config (`/etc/influxdb/influxdb.conf`), not certain they will solve the problem
```
[meta]
  dir = "/var/lib/influxdb/meta"
[data]
  dir = "/var/lib/influxdb/data"
  wal-dir = "/var/lib/influxdb/wal"
  index-version = "tsi1"
  query-log-enabled = false
  cache-snapshot-memory-size = "64m"
  cache-snapshot-write-cold-duration = "15m"
  compact-full-write-cold-duration = "4h"
  max-concurrent-compactions = 1  
  compact-throughput = "24m"
  compact-throughput-burst = "48m"
  max-index-log-file-size = "1m"
```
create a continuous query 
---
`CREATE DATABASE logger_ds`

    CREATE CONTINUOUS QUERY downsample_solar ON logger_ds BEGIN SELECT first(M_PExp) AS M_PExp, first(M_PTot) AS M_PTot, first(P_accum) AS P_accum, first(P_daily) AS P_daily, first(P_peak) AS P_peak, MEAN("M_A-I") AS "M_A-I", MEAN("M_A-U") AS "M_A-U", MEAN("M_B-I") AS "M_B-I", MEAN("M_B-U") AS "M_B-U", MEAN("M_C-I") AS "M_C-I", MEAN("M_C-U") AS "M_C-U", MEAN("U_A-B") AS "U_A-B", MEAN("η") AS "η", MEAN(Frequency) AS Frequency, MEAN(I_A) AS I_A, MEAN(M_Freq) AS M_Freq, MEAN(M_PF) AS M_PF, MEAN(M_U_AB) AS M_U_AB, MEAN(M_U_BC) AS M_U_BC, MEAN(M_U_CA) AS M_U_CA, MEAN(P_active) AS P_active, MEAN(P_reactive) AS P_reactive, MEAN(PF) AS PF, MEAN(PV_In) AS PV_In, MEAN(PV_P) AS PV_P, MEAN(PV_Un) AS PV_Un, MEAN(Temp) AS Temp, MEAN(U_A) AS U_A, MEAN(U_B) AS U_B, PERCENTILE("M_A-I", 20) AS "M_A-I_p20", PERCENTILE("M_A-I", 95) AS "M_A-I_p95", PERCENTILE("M_A-U", 20) AS "M_A-U_p20", PERCENTILE("M_A-U", 95) AS "M_A-U_p95", PERCENTILE("M_B-I", 20) AS "M_B-I_p20", PERCENTILE("M_B-I", 95) AS "M_B-I_p95", PERCENTILE("M_B-U", 20) AS "M_B-U_p20", PERCENTILE("M_B-U", 95) AS "M_B-U_p95", PERCENTILE("M_C-I", 20) AS "M_C-I_p20", PERCENTILE("M_C-I", 95) AS "M_C-I_p95", PERCENTILE("M_C-U", 20) AS "M_C-U_p20", PERCENTILE("M_C-U", 95) AS "M_C-U_p95", PERCENTILE("U_A-B", 20) AS "U_A-B_p20", PERCENTILE("U_A-B", 95) AS "U_A-B_p95", PERCENTILE(I_A, 20) AS I_A_p20, PERCENTILE(I_A, 95) AS I_A_p95, PERCENTILE(M_PF, 20) AS M_PF_p20, PERCENTILE(M_PF, 95) AS M_PF_p95, PERCENTILE(M_U_AB, 20) AS M_U_AB_p20, PERCENTILE(M_U_AB, 95) AS M_U_AB_p95, PERCENTILE(M_U_BC, 20) AS M_U_BC_p20, PERCENTILE(M_U_BC, 95) AS M_U_BC_p95, PERCENTILE(M_U_CA, 20) AS M_U_CA_p20, PERCENTILE(M_U_CA, 95) AS M_U_CA_p95, PERCENTILE(P_active, 20) AS P_active_p20, PERCENTILE(P_active, 95) AS P_active_p95, PERCENTILE(P_reactive, 20) AS P_reactive_p20, PERCENTILE(P_reactive, 95) AS P_reactive_p95, PERCENTILE(PV_In, 20) AS PV_In_p20, PERCENTILE(PV_In, 95) AS PV_In_p95, PERCENTILE(PV_P, 20) AS PV_P_p20, PERCENTILE(PV_P, 95) AS PV_P_p95, PERCENTILE(PV_Un, 20) AS PV_Un_p20, PERCENTILE(PV_Un, 95) AS PV_Un_p95, PERCENTILE(U_A, 20) AS U_A_p20, PERCENTILE(U_A, 95) AS U_A_p95, PERCENTILE(U_B, 20) AS U_B_p20  , PERCENTILE(U_B, 95) AS U_B_p95, MEAN("M_A-P") + 0.00001 AS "M_A-P", MEAN("M_B-P") + 0.00001 AS "M_B-P", MEAN("M_C-P") + 0.00001 AS "M_C-P", MEAN(M_P) + 0.00001 AS M_P, MEAN(M_Pr) + 0.00001 AS M_Pr, PERCENTILE("M_A-P", 20) + 0.00001 AS "M_A-P_p20", PERCENTILE("M_A-P", 95) + 0.00001 AS "M_A-P_p95", PERCENTILE("M_B-P", 20) + 0.00001 AS "M_B-P_p20", PERCENTILE("M_B-P", 95) + 0.00001 AS "M_B-P_p95", PERCENTILE("M_C-P", 20) + 0.00001 AS "M_C-P_p20", PERCENTILE("M_C-P", 95) + 0.00001 AS "M_C-P_p95", PERCENTILE(M_P, 20) + 0.00001 AS M_P_p20, PERCENTILE(M_P, 95) + 0.00001 AS M_P_p95, PERCENTILE(M_Pr, 20) + 0.00001 AS M_Pr_p20, PERCENTILE(M_Pr, 95) + 0.00001 AS M_Pr_p95 INTO logger_ds.autogen.Huawei FROM logger.autogen.Huawei GROUP BY time(5m) END
  
If you already have your logger running, before doing the next step you need to populate the logger_ds with the older data (CQ only does current data). Just ran from the query (within the begin and end limiters of the cq)
  
add a retention policy (this will delete anything older than 70 days from the 30s data. You should get 5m data from the cq)

    ALTER RETENTION POLICY autogen on logger DURATION 70d REPLICATION 1 SHARD DURATION 15d DEFAULT



