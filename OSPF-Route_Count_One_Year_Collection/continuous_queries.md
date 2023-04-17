### Downsampling Data using Continuous Queries in InfluxQL

- #### What are continuous queries?

	- Continuous Queries are InfluxQL queries that Continuous queries (CQ) are InfluxQL queries that run automatically and periodically on realtime data and store query results in a specified measurement

	[Official Documention for InfluxQL Continuous Queries](https://docs.influxdata.com/influxdb/v1.8/query_language/continuous_queries/)

- #### Use Case
	- An **AT&T** official keeps track of routing parameters like:
		- LSA (Link State Advertisement) Count
		- Routes Count
		- Packet Drop Count
	- This data is recorded for a very long time (a year) in spreadsheets by the official. Since, the job is tedious, it should be automated.
	- **Model Driven Telemetry** in XR Routers already helps in recording and storing router data into InfluxDB. However, this data is captured every 5 minutes and storing this data for a year is very costly in terms of memory consumption.
	- So, we can downsample this data using continuous queries and retain it for a year using a new retention policy which retains data for 52 weeks.

- #### Steps to access Health Insights InfluxDB
	- SSH to the Data Gateway
	- `sudo su`
	- Next, use the command `docker exec -it robot-astack-influxdb bash` to get a bash shell in the container.
	- Initialize the Influx CLI using the `influx` command. The influx command line interface (CLI) includes commands to manage many aspects of InfluxDB, including buckets, organizations, users, tasks, etc.
	- Get an authorization token. `auth <username> <password>`
	- Execute the `use robot_alertdb` command to switch to the robot_alertdb database.
	- We can now access the Health Insights Influx Database.

- #### Executing Continuous Queries
	- First of all, we must create a new retention-policy for retaining data for a year. The default retention-policy implemented in robot_alertdb is rp3d\_1 which stores data collected for 3 days only, which is of no use for this case.
    ```sql
    CREATE RETENTION POLICY a_year ON "robot_alertdb" DURATION 56w REPLICATION 1
    ```
    - Next we create a continuous query on the measurement of our choice.
    ```sql
    CREATE CONTINUOUS QUERY "cq_15m" ON "robot_alertdb" 
    -- specifying the query-name and database-name
    BEGIN 
    SELECT mean("active-routes-count")
    -- selecting the column and the operation to be performed for downsampling
    AS "active-routes-count"
    INTO "a_year"."downsampled_routes" 
    -- this would be a new measurement in which yearly data would be stored
    FROM rp3d_1."Cisco-IOS-XR-ip-rib-ipv4-oper:rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/ospf/as/information"
    GROUP BY time(15m),"af-name","as", "vrf-name", "saf-name" 
    -- group by specifies the interval at which mean would be taken
    END
    ```
	
#### Displaying output for year-long data 
 - By default, Health Insights Grafana dashboard will show only past 3 days data - because retention policy is for 3 days.
 - To display year-long data, the dashboard file for the corresponding Key Performance Indicator must undergo the following changes:
```javascript
/*Change the lower limit of time-axis so accordingly past 365 days data can be shown*/
// x.time.from = "now-365d"
"time" : {
	"from": "now-365d",
	"to": "now"
}
/* Replace retention policy "rp3d_1" by "a_year" everywhere */
x.templating.list[6].current.text = "a_year"
x.templating.list[6].current.value = "a_year"
x.templating.list[6].regex = "a_year"

/* Change the measurement name everywhere, from 'sensor-path measurement' to the corresponding measurement where the downsampled data is stored */
/* In active routes case, the measurement is "downsampled_routes" */


/* Change the InfluxQL query which was earlier meant for 3-day data to one which is for meant for 365-day data */
x.rows[0].panels[0].targets[0].query = "SELECT /^$KpiFields$/ FROM $RP.\"downsampled_routes\" GROUP BY *"
```
