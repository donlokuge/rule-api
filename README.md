# Rule Architecture

Rule processing system can be sepetated into few parts.

1. Rule JSON
2. SDK to validate and execute rules
3. API to create/update/delete rules
4. Process rules on forecating event

## Format of a Rule

Rule can be in the JSON format. It will be stored in aws dynamodb through an api.
Rule sdk can be used to validate and execute the rule.

The following is an example of the format. In this example, the JSON is formatted for easy reading

```javascript
{
    "id": "rule#id#uId", // id of rule
    "name": "RuleName", // name of rule
    "description": "", // description of rule
    "enabled": true, // rule is enabled or disabled
    "notificationGroup": "SalesAdmins", // name of the notification group assuming there will be notifications for group of users
    "comparisonOperator": "GreaterThanThreshold", // comparison operator value for comparison [>, <, >=, <=]
    "threshold": "20", // value of threshold
    "points": [ // evaluate rule by following points
        {// fist point
            "type": "Table", // type of point values [table, s3 etc]
            "table": "Sales",  // name of the table
            "function": {// function to execute to get the point
                "name": "lastPointSaleByDepartment", // function name lastPointSaleBy[Hierarchy] assuming there will be different functions.
                "parameters": ["departmentId"] // any relevant parameter. could be empty, or id
            }
        },
        {// second point
            "type": "Table",
            "table": "Forecast",
            "function" :{
                "name": "firstForecastPointSalesByDepartment",
                "parameters":["departmentId"]
            }
        }
    ]
}

```

## SDK

There will be a sdk to validate and execute rule

Example

```javascript

const ruleEngine  = require('@company/rule-engine')

const rule = {...}

// Validation
const valid = ruleEngine.isValid(rule);


// Execution
const options = {...}
const result = ruleEngine.execute(rule, options);

// {
// success:true,
// type:"notification",
//  .....
// }

// publish notification
ruleEngine.notify(result);



```

## AWS Architecture Diagrams

### 1. Rule api (Create, Update, Delete Rules)

Following image shows how rule api is implemented. Using api gateway and set of lambda functions rules are saved into dynamo db.

Using dynamodb streams we can push updated rules to a sqs queue. Queue is use to execute updated rules. This flow is explained here.

Alternate idea would be to send updated rules to a SNS Topic.

![Api](/rule_api_flow.png)

### 2. Forecasting Flow

Following image assumes how forcating is processed.
The transformation lambda function will publish event to the Forecast ready SNS topic.

![Forecast](/forecast_flow.png)

### 3. Rule Processing Flow

Following image shows how rule processing flow is handled. We use pub/sub pattern  for rule processing.

Rules processing event can be triggered by following ways.

1. When forecasting is finished and SNS notification is published
2. When CRUD event happens on api gateway and SNS notification is published
3. When CRUD event happens on api gateway and message to sqs is published

Rules are processed in `Process Rule lambda function` using sdk and then call to verify notification lambda function which which validate notification and send event to SES.

Process rule lambda should be idempotence function.

AWS Cloud watch to process lambda logs and metrics

AWS Secrects manager to manage secrets for lambda and dynamodb

![Forecast](/rule_processing.png)
