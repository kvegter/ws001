# ws001

Short temporary workshop project to load some sample data into neo4j 
Data is placed in the data directory

This is only an initial step and it is not finished, just to show the approach.

The data loaded will be using this model:
<img src="wsmodel.png">

## Loading data 

Make sure you have an empty database and have the apoc library installed!

You can copy and paste the 'code' sections in the neo4j browser, make sure "Enable multi statement query editor" in the settings is checked.

### Step 1: creating constraints

Before you load data into neo4j you need to think about the constraints. Copy and paste the following into the Neo4j browser
```
CREATE CONSTRAINT ukBuilding IF NOT EXISTS FOR (n:Building) REQUIRE n.id IS UNIQUE;
CREATE CONSTRAINT ukSensor IF NOT EXISTS FOR (n:Sensor) REQUIRE n.eui IS UNIQUE;
```

### Step 2: Loading the building room and sensor structure

```
:auto LOAD CSV WITH HEADERS FROM "https://github.com/kvegter/ws001/raw/main/data/pap_villa_data.csv" as row
fieldterminator ";"
WITH row
CALL {
  with row
	MERGE (bld:Building { id : toInteger(row.villa_id) })
	MERGE (bld)-[:HAS_ROOM]->(room:Room { name : trim(row.room) })
	MERGE (kwh:Sensor:KWH { eui: row.device_eui_kwh } )
	MERGE (mvm:Sensor:Movement { eui : row.device_eui_sensor })
	MERGE (room)-[:HAS_MOVEMENT_SENSOR]->(mvm)
	MERGE (room)-[:HAS_KWH_SENSOR]->(kwh)
}
return count(row);
```

### Step 3: Loading the movement data

This can take a while!
```
:auto LOAD CSV WITH HEADERS FROM "https://github.com/kvegter/ws001/blob/main/data/pap_beweging_data.csv?raw=true" as row
fieldterminator ";"
WITH row
CALL {
   with row
MATCH (room)-[:HAS_MOVEMENT_SENSOR]->(mvm:Sensor { eui: row.device_eui } )
MERGE (room)-[r:HAS_EVENT]-(hd:Event { time: toInteger(row.tijdstip) })
ON CREATE SET hd.movement = [toInteger(row.beweging)]
, hd.date = datetime({year: toInteger(substring(row.tijdstip, 0, 4))
                   , month: toInteger(substring(row.tijdstip, 4, 2))
                   ,   day: toInteger(substring(row.tijdstip, 6, 2)) 
                   ,  hour: toInteger(substring(row.tijdstip, 8, 2)) })
ON MATCH SET hd.movement = hd.movement + [toInteger(row.beweging)] 
MERGE (mvm)-[:REPORTS_MOVEMENT]->(hd)
   return mvm, r, hd                
} IN transactions of 100 rows
return count(*) ;
```

### Step 4: loading the usage data

```
:auto LOAD CSV WITH HEADERS FROM "https://github.com/kvegter/ws001/raw/main/data/pap_verbruik_data.csv" as row
fieldterminator ";"
WITH row
CALL {
   with row
MATCH (room)-[:HAS_KWH_SENSOR]->(mvm:Sensor { eui: row.device_eui } )
MERGE (room)-[r:HAS_EVENT]->(hd:Event { time: toInteger(row.tijdstip) })
ON CREATE SET hd.usage = [toInteger(row.verbruik)]
, hd.date = datetime({year: toInteger(substring(row.tijdstip, 0, 4))
                   , month: toInteger(substring(row.tijdstip, 4, 2))
                   ,   day: toInteger(substring(row.tijdstip, 6, 2)) 
                   ,  hour: toInteger(substring(row.tijdstip, 8, 2)) })
ON MATCH SET hd.usage = CASE WHEN hd.usage is null or hd.usage = [] THEN [toInteger(row.verbruik)] ELSE  hd.usage + [toInteger(row.verbruik)]  END                 
MERGE (mvm)-[:REPORTS_KWH]->(hd)
   return mvm, r, hd                
} IN transactions of 500 rows
return count(*) ;

```

### Step 5: Loading more sensor data 

In this file also temperature, humidity, pressure and battery information is present

```:auto LOAD CSV WITH HEADERS FROM "https://github.com/kvegter/ws001/blob/main/data/condensed_sensor_data.csv?raw=true" as row
WITH row
CALL {
   with row
MATCH (room)-[:HAS_MOVEMENT_SENSOR]->(mvm:Sensor { eui: row.eui } )
MERGE (room)-[r:HAS_EVENT]-(hd:Event { time: toInteger(row.sdate) })
ON CREATE SET hd.movement = [toInteger(row.motion)]
, hd.date = datetime({year: toInteger(substring(row.sdate, 0, 4))
                   , month: toInteger(substring(row.sdate, 4, 2))
                   ,   day: toInteger(substring(row.sdate, 6, 2)) 
                   ,  hour: toInteger(substring(row.sdate, 8, 2)) })
ON MATCH SET hd.movement = hd.movement + [toInteger(row.motion)] 
SET hd.co2 = toFloat(row.co2)
,   hd.temperature = toFloat(row.temperature)
,   hd.humidity = toFloat(row.humidity)
,   hd.pressure = toFloat(row.pressure)
,   hd.battery = toFLoat(row.battery)
MERGE (mvm)-[:REPORTS_MOVEMENT]->(hd)
   return mvm, r, hd                
} IN transactions of 500 rows
return count(*) ;
```

### Step 6: Creating NEXT relationships between room Events


```call apoc.periodic.iterate("
MATCH (v:Building )-[:HAS_ROOM]->(room:Room )
RETURN id(room) as nid",
"  MATCH (room:Room )-->(evt:Event)  
  WHERE id(room) = nid
  WITH room, apoc.coll.sortNodes(collect(evt) ,'^time') as events where size(events) > 1
	WITH room, events, range(1,size(events) -1) as indexList 
	UNWIND indexList as index 
	WITH room, events[index -1 ] as first, events[index] as next , index
	MERGE (first)-[:NEXT]->(next)
	FOREACH (y in CASE WHEN index = 1 THEN [1] ELSE [] END |
		MERGE (room)-[:FIRST_SENSOR_DATA]->(first)
	)	
	return count(*) as nextCount
"
, { batchSize : 1 })
;
```
