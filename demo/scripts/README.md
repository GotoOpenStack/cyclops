# Scripts usage
This section will show how to run script to generate custom usage data and produce example inctance of bill.

## Demo Scenario Description

The scripts included generates synthetic data stream for a fixed user account in a hypothetical Kubernetes cluster. The script generates configured number of usage data points tracking pod memory consumption over the specified time period. The usage value is randomized number between 0 and 2048 MB.

Each synthetic usage data value that the generator script creates has the following example JSON outline:

```json
 {
 	"metric": "memory",
 	"account": "dord",
 	"time": 1493450936000,
 	"usage": 22560,
 	"unit": "KB",
 	"data":{
 		"pod_id":"string id"
 	}   
 }
```

Specifically, the released code send values as MB instead of KB. 

### Rules
As cyclops framework is rules driven, pricing and billing models have to be injected before meaningful processing can be done by the framework. `rulesApply.sh` script registers all necessary rules in Cyclops, including pricing rules, which defines how the consumed resources metrics are charged.
```
import ch.icclab.cyclops.facts.Usage;
import ch.icclab.cyclops.facts.Charge;

rule "Static rule for ram"
salience 50
when
  $usage: Usage(metric == "memory") 
then
  Charge charge = new Charge($usage);
  charge.setCharge(0.01 * $usage.getUsage());
  charge.setCurrency("CHF");

  retract($usage);
  insert(charge);
```
As can be seen by the rule definition above, memory parameter is charged as a rate of 0.01 CHF per MB. This is a purely for demonostration purposes. In realistic scenarios, first you will have to transform memory levels into MB-hr format or MB-month value. Based on how the inter-sample arrival window has been configured, the computation will happen accordingly (BUT NOT FOR THIS DEMO THOUGH). 

To execute pricing and billing rule registration with Cyclops, simply execute the following command.

```sh
$ bash rulesApply.sh
```
### Simulating a service usage data stream
To generate usage data there is `dataGeneration.sh` script which allows you to specify some window (in minutes) for which usage data points should be generated as part of simulation. You will also need to specify the inter-arrival duration between samples. This script can be executed as shown below. During execution, it will generate and send simulated usage data stream into Cyclop framework. It simulates data, that would be collected from real clusters periodicaly. The command is provided next:   

```sh
$ bash dataGeneration.sh -t0=22-Sep-2017 -t1=23-Sep-2017 -i=360 \\in linux
$ bash dataGeneration.sh -t0=22-09-2017 -t1=23-09-2017 -i=360 \\in MacOS
```
- t0 = start of the data simulation window (a calendar date)
- t1 = end of the data simulation window (a calendar date)
- i = inter sample arrival interval (in minutes)

Upon termination, necessary simulation data has been fed into Cyclops and we are ready to start the next phases: generation of usage reports, charge data records and finally a bill object.

### Generation of invoice 

Borrowing ideas from the world of telecommunication, Cyclops is able to consolidate large number of usage data points for different services used by the customer in a given time window. Such a consolidation results in Usage Data Record (UDR). The pricing rules act of UDR objects to transform aggregated usage amounts to a cost value. This is referred to as Charge Data Record (CDR) in Cyclops. Ultimately, a bill is consolidation of all CDRs in an invoicing period + some additional transformations (if necessary). To produce a demo bill object, included script ```getInvoice.sh``` runs these commands in a row:
 - GenerateUDRs
 - FlushUDRs
 - GenerateBill

User has to specify time window (invoice period) as well. Cyclops will generate invoice for this particular time. ```getInvoice.sh``` script accepts invoice period. As the framework may take non-zero time in seconds to complete number crunching and aggregation activities for UDR and CDR record generation, the script also takes a delay parameter to allow the previous stage to finish before the subsequent stage can begin.

```sh
$ bash getInvoice.sh -t0=22-Sep-2017 -t1=23-Sep-2017 -d=5 //in linux
$ bash getInvoice.sh -t0=22-09-2017 -t1=23-09-2017 -d=5 //in MacOS
```
* t0 = start date of the invoice period (inclusive)
* t1 = end date of the invoice period (inclusive)
* d = delay parameter in seconds, you must make a reasonable judgement call regarding this value, it should be sufficient to allow each of the stages to finish data processing before the next stage can start.

If the bill is properly generated, you can view it in the browser by navigating to ```http(s)://billing-service-ip-address:port/bills```. In this demo, you can access it using ```localhost``` instead over port 4569.

Sample output may look similar to -

```json
{
  "data": [
    {
      "id": 2,
      "account": "dord",
      "charge": 3346.64,
      "time_from": 1530454049000,
      "time_to": 1531663649000,
      "data": [
        {
          "data": {
            "unit": "MB",
            "usage": 334664.0,
            "pod_id": "pod1"
          },
          "charge": 3346.64,
          "metric": "memory",
          "time_to": 1531663649000,
          "time_from": 1530454049000
        }
      ],
      "currency": "CHF"
    }
  ],
  "pageLimit": 500,
  "selectedPage": 0,
  "recordsShown": 1
}
```

### Database cleaning

We have included a script to clean up your database by dumping any stale data points so that this demo is easier to play with. `cleanDB.sh` can dump all usage points, udr entries, cdr entries, or bills (`-d` | `--data`) or any existing rules (`-r` | `--rules`) or both (`-r -d`).  Below are a few possibilities -
```sh
bash cleanDB.sh --rules
bash cleanDB.sh --data
bash cleanDB.sh -d -r
```

You will need psql utility available for the above cleaning commands to work. 

 ### License

```reStructuredText
      Licensed under the Apache License, Version 2.0 (the "License"); you may
      not use this file except in compliance with the License. You may obtain
      a copy of the License at
 
           http://www.apache.org/licenses/LICENSE-2.0
 
      Unless required by applicable law or agreed to in writing, software
      distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
      WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
      License for the specific language governing permissions and limitations
      under the License.
```