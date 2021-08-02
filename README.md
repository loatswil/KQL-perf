# KQL-perf
KUSTO Query Language - performance queries

Here you will find basic examples of using the KUSTO Query language for use in Azure Log Analytics that I have collected and used over the years.

Most original sources are no longer cited as much of the code has changed or been updated to suit my own needs.

Top 5 running processes in the last 3 days

let RunProcesses = SecurityEvent
| where TimeGenerated > ago(3d)
| where EventID == '4688';
let Top5Processes =
RunProcesses
| summarize count() by Process
| top 5 by count_;
RunProcesses
| where Process in (Top5Processes)
| summarize count() by bin (TimeGenerated, 1h), Process
| render timechart
 
Perf

let StartTime = datetime(2017-11-12 00:30);
let EndTime = datetime(2017-11-22 00:30);

Perf
| where CounterName == “% Processor Time”
| where TimeGenerated > StartTime and TimeGenerated < EndTime
| project TimeGenerated, Computer, cpu=CounterValue //project | Select the columns to include, rename or drop, and insert new computed columns
| join kind= inner ( //join | Merge the rows of two tables to form a new table by matching values of the specified column(s) from each table

Perf
| where CounterName == “% Committed Bytes In Use”
| where TimeGenerated > StartTime and TimeGenerated < EndTime
| project TimeGenerated , Computer, mem=CounterValue
) on TimeGenerated, Computer
| summarize avgCpu=avg(cpu), maxMem=max(mem) by Computer, TimeGenerated bin=1hr //summarize | Produces a table that aggregates the content of the input table
| render timechart
 
let StartTime = now(-1h);
let EndTime = now();
Perf
| where CounterName == “% Processor Time”
| where TimeGenerated > StartTime and TimeGenerated < EndTime
| project TimeGenerated, Computer, cpu=CounterValue //project | Select the columns to include, rename or drop, and insert new computed columns
| join kind= inner ( //join | Merge the rows of two tables to form a new table by matching values of the specified column(s) from each table
Perf
| where CounterName == “% Committed Bytes In Use”
| where TimeGenerated > StartTime and TimeGenerated < EndTime
| project TimeGenerated , Computer, mem=CounterValue
) on TimeGenerated, Computer
| summarize avgCpu=avg(cpu), maxMem=max(mem) by Computer, TimeGenerated bin=1hr //summarize | Produces a table that aggregates the content of the input table
| render timechart
 
let group1 = Heartbeat | where Computer contains “1598” | summarize by Computer, group=“group1”;
let group2 = Heartbeat | where Computer contains “1599”| summarize by Computer, group=“group2”;
let combinedGroup= union group1, group2;
let projectedComputers = combinedGroup | summarize makeset(Computer);
Perf
| where ObjectName == “Processor”
| where CounterName == “% Processor Time”
| where Computer in (projectedComputers)
| summarize avg(CounterValue) by bin(TimeGenerated, 15m), Computer
| render timechart
 
Perf
| where CounterName == “% Processor Time” and ObjectName == “Processor” and InstanceName == “_Total”
| summarize AVGCPU = avg(CounterValue) by bin(TimeGenerated, 1h), Computer
| sort by TimeGenerated desc
| render timechart
// Oql: Type:Perf CounterName=”% Processor Time” ObjectName=Processor InstanceName=_Total | measure avg(CounterValue) as AVGCPU by Computer | display linechart // WorkspaceId: {b438b4f6-912a-46d5-9cb1-b44069212abc} // Version: 0.1.115
 
let endTime=now();
let timerange =1d;
let startTime=now() – timerange;
let mInterval=4;
let mAvgParm = repeat(1, mInterval);
Perf
| where ObjectName == “Processor”
| where CounterName == “% Processor Time”
| make-series avgCpu=avg(CounterValue) default=0 on TimeGenerated in range(startTime, endTime, 15m) by Computer
| extend moving_avgCpu = series_fir(avgCpu, mAvgParm)
| render timechart
 
let PercentSpace = 50; //enter the threshold for the disk space percentage
Perf
| where ObjectName == “LogicalDisk” and CounterName == “% Free Space”
| summarize FreeSpace = min(CounterValue) by Computer, InstanceName
| where InstanceName contains “:”
| where FreeSpace < PercentSpace
| sort by FreeSpace asc
| render barchart kind = unstacked
 
Perf
| where ObjectName == “Processor”
| where CounterName == “% Processor Time”
| extend group = case(Computer contains “1598”, “admgrup”, Computer contains “1599”, “bsgroup”, “other”)
| summarize avg(CounterValue) by bin(TimeGenerated, 1h) , group
| render timechart
 
Perf
| where ObjectName==“Processor” and CounterName==“% Processor Time”
| summarize avg(CounterValue) by Computer | where avg_CounterValue > 10
 
Perf
| where ObjectName==“Memory” and CounterName==“% Committed Bytes In Use”
| summarize avg(CounterValue) by Computer | where avg_CounterValue > 70
 
Perf
| where ObjectName == “Processor” and CounterName == “% Processor Time” and InstanceName == “_Total” and Computer in ((Heartbeat
| where OSType == “Windows”
| distinct Computer))
| summarize AggregatedValue = avg(CounterValue) by bin(TimeGenerated, 1h), Computer
| render timechart
 
Perf
| where TimeGenerated > ago(1d)
| where CounterName == “% Processor Time”
| summarize avg(CounterValue), percentiles(CounterValue, 50, 95) by bin(TimeGenerated, 1h)
| extend Threshold = 10 // set a refernce line
| render timechart
 
let StartTime = now()-5d; let EndTime = now()-4d; Perf | where CounterName == “% Processor Time” | where TimeGenerated > StartTime and TimeGenerated < EndTime and TimeGenerated < EndTime | project TimeGenerated, Computer, cpu=CounterValue | join kind= inner ( Perf | where CounterName == “% Used Memory” | where TimeGenerated > StartTime and TimeGenerated < EndTime | project TimeGenerated , Computer, mem=CounterValue ) on TimeGenerated, Computer | summarize avgCpu=avg(cpu), maxMem=max(mem) by TimeGenerated bin=30m | render timechart
 

Perf | where TimeGenerated > ago(4h) | where Computer startswith “Contoso” | where CounterName == @“% Processor Time” | summarize avg(CounterValue) by Computer, bin(TimeGenerated, 15m) | render timechart
 
let computerContains = “ejukebox”; // add name to define the computer you want to look at
let perfCounterName = “% Processor Time”; // change to be the perf counter you want to compare
let binSize = 15m; // using the bin function with 15 minute bins to aggregate average perf counter values
let refDelay = 3d; // reference data will be 3 days before now
let EndTime = now();
let StartTime = EndTime – 1d;
let ReferenceEndTime = EndTime – refDelay;
let ReferenceStartTime = StartTime – refDelay;
Perf
| where Computer contains computerContains and CounterName == perfCounterName
| where TimeGenerated between(StartTime .. EndTime) or TimeGenerated between(ReferenceStartTime .. ReferenceEndTime)
| extend RefTimeGenerated = iif( TimeGenerated between( StartTime .. EndTime ),TimeGenerated, TimeGenerated + refDelay) //extend | Create calculated columns and append them to the result set
| extend ID = iif( TimeGenerated between( StartTime .. EndTime ), “Current”, “Reference”)
| summarize avg(CounterValue) by ID, bin(RefTimeGenerated, binSize)
| render timechart
 
//Average CPU Utilization across all computers
Perf 
| where ObjectName == “Processor” and CounterName == “% Processor Time” and InstanceName == “_Total” 
| summarize AVGCPU = avg(Average) by Computer
 
//Maximum CPU Utilization across all computers
Perf | where CounterName == “% Processor Time” | summarize AggregatedValue = max(Max) by Computer
 
//Average Current Disk Queue length across all the instances of a given computer
Perf 
| where ObjectName == “LogicalDisk” and CounterName == “Current Disk Queue Length” and Computer == “MyComputerName” 
| summarize AggregatedValue = avg(Average) by InstanceName
 
Perf 
| where CounterName == “% Processor Time” and InstanceName == “_Total” 
| summarize AggregatedValue = avg(CounterValue) by bin(TimeGenerated, 1h), Computer
  
Perf 
| where Computer == “MyComputer” and CounterName startswith_cs “%” and InstanceName == “_Total” 
| summarize AggregatedValue = percentile(CounterValue, 70) by bin(TimeGenerated, 1h), CounterName
 

Perf 
| where CounterName == “% Processor Time” and InstanceName == “_Total” and Computer == “MyComputer” 
| summarize [“min(CounterValue)”] = min(CounterValue), [“avg(CounterValue)”] = avg(CounterValue), [“percentile75(CounterValue)”] = percentile(CounterValue, 75), [“max(CounterValue)”] = max(CounterValue) by bin(TimeGenerated, 1h), Computer
 

Perf 
| where ObjectName == “MSSQL$INST2:Databases” and InstanceName == “master”

AzureDiagnostics
| where ResourceProvider == “MICROSOFT.NETWORK” and Category == “ApplicationGatewayFirewallLog”
| summarize Count=count() by details_file_s, action_s
| top 5 by Count desc
| render piechart
 
AzureDiagnostics
| where ResourceProvider == “MICROSOFT.NETWORK” and Category == “ApplicationGatewayFirewallLog”
| summarize Count=count() by clientIp_s, action_s
| top 5 by Count desc
| render piechart
 
AzureDiagnostics
| where ResourceProvider == “MICROSOFT.NETWORK” and Category == “ApplicationGatewayFirewallLog”
| summarize Count=count() by Message, action_s
| top 5 by Count desc
| render piechart
 
AzureDiagnostics
| where ResourceProvider == “MICROSOFT.NETWORK” and Category == “ApplicationGatewayPerformanceLog”
| project Time=TimeGenerated, Latency_ms=latency_d
| render timechart
 
AzureDiagnostics
| where ResourceProvider == “MICROSOFT.NETWORK” and Category == “ApplicationGatewayPerformanceLog”
| extend Throughput_Mb = (throughput_d/1000)/1000
| project Time=TimeGenerated, Throughput_Mb 
| render timechart
 
AzureDiagnostics
| where ResourceProvider == “MICROSOFT.NETWORK” and Category == “ApplicationGatewayPerformanceLog”
| project Requests=requestCount_d, FailedRequests=failedRequestCount_d, TimeGenerated 
| render timechart
 
WireData
| where Direction == “Outbound”
| sort by SessionStartTime desc
| summarize count(), makeset(SessionStartTime), makeset(SessionEndTime), makeset(LocalIP), makeset(RemoteIP), makeset(RemotePortNumber), makeset(SessionState), makeset(ProtocolName), makeset(IPVersion) by Computer
 

WireData
| where Direction == “Inbound”
| sort by SessionStartTime desc
| summarize count(), makeset(SessionStartTime), makeset(SessionEndTime), makeset(LocalIP), makeset(RemoteIP), makeset(RemotePortNumber), makeset(SessionState), makeset(ProtocolName), makeset(IPVersion) by Computer
 

WireData
| sort by SessionStartTime desc
| summarize count(), makeset(SessionStartTime), makeset(SessionEndTime), makeset(LocalIP), makeset(RemoteIP), makeset(RemotePortNumber), makeset(SessionState), makeset(ProtocolName), makeset(IPVersion), makeset(Direction) by Computer
 
search in (WireData) * | summarize AggregatedValue = sum(TotalBytes) by Computer | limit
search in (WireData) * | summarize AggregatedValue = count() by LocalIP
search in (WireData) * | summarize AggregatedValue = sum(TotalBytes) by ProcessName
search in (WireData) * | summarize AggregatedValue = sum(TotalBytes) by IPVersion
search in (WireData) Direction == “Outbound” | summarize AggregatedValue = count() by RemoteIP

Syslog | sort by TimeGenerated desc
| where SeverityLevel == “err”
| summarize count(), makeset(Facility), makeset(SyslogMessage), makeset(HostIP), makeset(ProcessName), makeset(Type) by Computer

Syslog | sort by TimeGenerated desc
| where SeverityLevel == “warn”
| summarize count(), makeset(Facility), makeset(SyslogMessage), makeset(HostIP), makeset(ProcessName), makeset(Type) by Computer
