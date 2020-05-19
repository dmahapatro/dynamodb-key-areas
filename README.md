# dynamodb-key-areas
Key areas & best practices in AWS DynamoDB

> Assumption: 
> - We have already level set on how NoSQL works.
> - Aware DynamoDB Basics (Table, Items, Attributes, etc)

## Change in Mindset
 - Do not see NoSQL (in this case DynamoDB) from a relational database lens
 - Understand that normalization is not a criteria for DynamoDB
 - Relations will exist in DynamoDB but not in the form of Foreign Key
 - CAP Theorem:
    - Relational databases stresses on Strong consistency model (ACID)
    - DynamoDB stresses on performance and eventual consistency model (BASE)
 - Normalization in RDBMS helps in building more # of Tables.
 - DynamoDB works best on the notion of partitions instead of # of tables
    - Take a good amount of time to understand the rationale of data, how will it be retreived, how will it be written to and from DynamoDB Tables
    - Above analysis will drive the performance and cost of DynamoDB

## How to model Data in DynamoDB?
 - For new applications, review user stories about activities and objectives. Document the various use cases you identify, and analyze the access patterns that they require.
 - For existing applications, analyze query logs to find out how people are currently using the system and what the key access patterns are.
 - [Reference](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-modeling-nosql.html)
 - [Example](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-modeling-nosql-B.html)

## What is RCU & WCU?
 - **RCU (Read Capacity Unit):**
   - One read capacity unit represents one strongly consistent read per second, or two eventually consistent reads per second, for an item up to 4 KB in size
 - **WCU (Write Capacity Unit)**
   - One write capacity unit represents one write per second for an item up to 1 KB in size
 - [Reference Document](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html#HowItWorks.ProvisionedThroughput.Manual)

## Do I need to care about RCU and WCU?
 - It depends on which mode you want to setup
 - **On-Demand Mode**:
   - When you choose on-demand mode, DynamoDB instantly accommodates your workloads as they ramp up or down to any previously reached traffic level
   - On-demand mode is a good option if any of the following are true:
     - You create new tables with unknown workloads.
     - You have unpredictable application traffic.
     - You prefer the ease of paying for only what you use.
   - [Reference Document](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html#HowItWorks.OnDemand)

 - **Provisioned Mode**:
   - If you choose provisioned mode, you specify the number of reads and writes per second that you require for your application
   - Provisioned mode is a good option if any of the following are true:
     - You have predictable application traffic.
     - You run applications whose traffic is consistent or ramps gradually.
     - You can forecast capacity requirements to control costs.
   - [Reference Document](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html#HowItWorks.ProvisionedThroughput.Manual)

## Key Concepts to understand
 - [Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithTables.html)
 - [Items](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithItems.html)
   - Understand when to use `UpdateItem` vs `PutItem`
   - Understand when to use `BatchWriteItem`
   - Understand the benefits of [Conditional Writes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithItems.html#WorkingWithItems.ConditionalUpdate)
 - [Queries](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html)
 - [Scans](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Scan.html)
 - [Indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html)
   - [Global Secondary Indexes (GSI)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html)
     - An index with a partition key and a sort key that can be different from those on the base table
     - Consumes additional RCU and WCU than the base table. Hence, additional cost
     - GSI can be added to base table after table creation
   - [Local Secondary Indexes (LSI)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/LSI.html)
     - An index that has the same partition key as the base table, but a different sort key
     - Uses the RCU and WCU allocated to the base table.
     - LSI can only be added to table during table creation
   - See `Working with Indexes` for additional comparison characteristics
   - Indexes are the key features in DynamoDB and should be used efficiently to get the maximum benefit out of the table
   - Data Modeling decisions also depend on how Indexes will be built in future
 - [Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
   - A DynamoDB stream is an ordered flow of information about changes to items in a DynamoDB table. When you enable a stream on a table, DynamoDB captures information about every modification to data items in the table
   - Similar to CDC (Change Data Capture) and gets integrated with services like AWS Lambda to do additional operations on change record.
   - Streams are really helpful to build aggregations where item changes are processed via Lambda and the Lambda writes back to DynamoDB with the aggregated value

## Security
 - Encryption at Rest
   - Using AWS KMS (Use AWS Owned CMK or AWS managed CMK or Customer Managed CMK)
 - **[VPC Gateway Endpoint for DynamoDB](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints-ddb.html)**
   - A VPC endpoint enables you to privately connect your VPC to supported AWS services without requiring an internet gateway, NAT device, VPN connection, or AWS Direct Connect connection. 
   - Instances in your VPC do not require public IP addresses to communicate with resources in the service. 
   - Traffic between your VPC and the other service does not leave the Amazon network.
   - Use `aws:sourceVpce` in IAM Policy document `Condition` to specify that connection is only allowed to DynamoDB via a specific VPC endpoint
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AccessFromSpecificEndpoint",
         "Action": "dynamodb:*",
         "Effect": "Deny",
         "Resource": "arn:aws:dynamodb:region:account-id:table/*",
         "Condition": { 
           "StringNotEquals" : { 
             "aws:sourceVpce": "vpce-11aa22bb" 
            } 
          }
        }  
      ]
   }
   ```
 - **Principal Specific Access**
   - Provide access to the principal provided as a condition
   ```json
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Sid": "DenyAccessToDynamoDBOutsidePrincipal",
        "Effect": "Deny",
        "Action": [
            "dynamodb:*"
        ],
        "Resource": "arn:aws:dynamodb:region:account-id:table/*",
        "Condition": {
          "ArnNotLike": {
            "aws:PrincipalARN":"arn:aws:iam::123456789:role/DynamoDBUserRole"
          }
        }
      }]
    }
   ```
 - **Account Specific Access**
   - Provide access to the Pricipal account specified in the policy
   ```json
   {    
     "Version": "2012-10-17",
     "Statement": [{
       "Sid": "DenyAccessToDynamoDBOutsidePrincipalAccount",
       "Effect": "Deny",
       "Action": [
         "dynamodb:*"
       ],
       "Resource": "arn:aws:dynamodb:region:account-id:table/*",
       "Condition": {
         "StringNotEquals": {
           "aws:PrincipalAccount": "123456789"
         }
       }
     }]
   }  
   ```

 - **[Fine-Grained Access Control using IAM Policy Conditions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/specifying-conditions.html)**
   - In DynamoDB, you have the option to specify conditions when granting permissions using an IAM policy. For example, you can:
     - Grant permissions to allow users read-only access to certain items and attributes in a table or a secondary index.
     - Grant permissions to allow users write-only access to certain attributes in a table, based upon the identity of that user.
    - Sample Policy Document:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowAccessToOnlyItemsMatchingUserID",
        "Effect": "Allow",
        "Action": [
          "dynamodb:GetItem",
          "dynamodb:BatchGetItem",
          "dynamodb:Query",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:BatchWriteItem"
        ],
        "Resource": [
          "arn:aws:dynamodb:us-east-2:123456789012:table/InsurancePolicy"
        ],
        "Condition": {
          "ForAllValues:StringEquals": {
            "dynamodb:LeadingKeys": [
              "${federated_user_id_which_is_the_partition_key_in_table}"
            ],
            "dynamodb:Attributes": [
              "UserId",
              "PolicyType",
              "Claims",
              "Collision",
              "Comprehensive"
            ]
          },
          "StringEqualsIfExists": {
            "dynamodb:Select": "SPECIFIC_ATTRIBUTES"
          }
        }
      }
    ]
  }  
  ```
  - `dynamodb:LeadingKeys` – This condition key allows users to access only the items where the partition key value matches their user ID.

  - `dynamodb:Attributes` – This condition key limits access to the specified attributes so that only the actions listed in the permissions policy can return values for these attributes. In addition, the `StringEqualsIfExists` clause ensures that the app must always provide a list of specific attributes to act upon and that the app can't request all attributes.

## Best Practices
 - [Best Practices for Designing and Architecting with DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
 - [Security Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices-security.html)

## TODOs
 - Add sample code/template
 - Add edge cases scenario

## General Resources for Reference
 - Developer Guide
   - [AWS DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
 - Books and Articles
   - [The DynamoDB Book](https://www.dynamodbbook.com/) - By Alex DeBrie
   - [12 Important Lessons from The DynamoDB Book](https://www.jeremydaly.com/important-lessons-from-the-dynamodb-book/) - By Jeremy Daly
 - Videos
   - [Advanced Design Patterns for DynamoDB](https://www.youtube.com/watch?v=HaEPXoXVf2k) - re:Invent 2018
   - [Data modeling with Amazon DynamoDB](https://www.youtube.com/watch?v=DIQVJqiSUkE) - re:Invent 2019
   - [Intro to Amazon DynamoDB](https://www.youtube.com/watch?v=W3S1OnDqWl4&list=PLg90sQCQnXJS9MhcgY8lJGP96kIuvKgum&index=25&t=32s)
   - [Data Modeling in Amazon DynamoDB: Part 1](https://www.youtube.com/watch?v=Rmf8mrJ3X2s&list=PLg90sQCQnXJS9MhcgY8lJGP96kIuvKgum&index=26&t=37s)
   - [Data Modeling in Amazon DynamoDB: Part 2](https://www.youtube.com/watch?v=KlhS7hSnFYs&list=PLg90sQCQnXJS9MhcgY8lJGP96kIuvKgum&index=27&t=0s)
   - [Advanced NoSQL Data Modeling in Amazon DynamoDB](https://www.youtube.com/watch?v=nhUtZ7suZWI&list=PLg90sQCQnXJS9MhcgY8lJGP96kIuvKgum&index=28&t=0s)

## FAQ
**How to name tables in DynamoDB per environment if the same AWS Account is used for multiple environments like (dev/test/stage)?**
 
Ideally DynamoDB table should be created per account per region. The table name can be re-used for the same account in a different region. Example:

A table named `InsurancePolicy` can be present in an AWS account in `us-east-1` (N. Virginia) and also be present in the same account in region `us-east-2` (Ohio)

If allowed `dev` and `test` can use a different region than `stage` but keep the same table name.

If multi-region access is not possible then another option will be to use a convention so that the environment name is prefixed/suffixed to the table name for better visibility like `env.domain.table`:

*Example:*  
`dev.nationwide.InsurancePolicy`  => for `dev` environment
`test.nationwide.InsurancePolicy` => for `test` environment

The pipeline taking care of development and test respectively have to prefix the environment while working on their respective table