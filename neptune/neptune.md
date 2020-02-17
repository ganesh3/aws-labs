## Creating a simple recommendation engine with Amazon Neptune (Level 200)
In this lab you will learn the basics of how to use Amazon Neptune in order to create a recommendation system using collaborative filtering.

## Contents
1. Language & Framework Definitions
2. Creating your very first Amazon Neptune Cluster
3. Setting up a Neptune Notebook
4. Interacting with your Neptune Database
5. Graph Definitions
6. Graph Basics
7. Create a simple recommendation engine
8. Where to from here?
9. Extras
    9.1 Bulk Load Data
    9.2 Backups
    9.3 Failover
10. Legal

## 1. Definitions

### Gremlin ([Apache TinkerPop™](http://tinkerpop.apache.org)) / [SparQL](https://www.w3.org/TR/sparql11-overview/) (*Pronounced Sparkle*)
Gremlin and SparQL are the two main frameworks commonly used with modelling and interacting graph networks. Gremlin includes multiple language drivers for most programming languages.

SparQL is an RDF query language - semantic query language for databases.

----

## 2. Creating your very first Amazon Neptune Cluster

An Neptune Cluster consists of one or more DB instances. A Neptune DB cluster instance can span multiple Availability Zones, which each AZ having a copy of the DB cluster data. Two types of DB instances make up a Neptune DB Cluster:

- Primary DB Cluster
    - A primary DB Cluster instance supports both write and read requests, and performs all the data modifications to the cluster volume. Each Neptune DB cluster has one primary DB instance
- Neptune replica
    - A replica instance connects to the same storage volume as a primary DB instance, but only supports read operations. Each Neptune DB cluster a number of replicas, in addition to the primary DB Instance. [Please refer to the official documentation for up-to-date supported replica numbers](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-db-clusters.html).
    - It is recommended to at least have one replica in a different Availability Zone to support automatic fail-over in the event of an outage.
    - Fail-over priority can also be specified.

You may use either the provided CloudFormation scripts [here](https://docs.aws.amazon.com/neptune/latest/userguide/get-started-create-cluster.html) or the simplified version provided below to create a Neptune Cluster in your AWS account.

The following script automatically generates a Neptune Cluster, with 1 Instance. It also generates all of the networking, IAM policy stacks and route tables to create Neptune in a different VPC. However, as per recommendations please test on a different account with appropriate backups and also review in detail before using in any production environment.

```
aws cloudformation create-stack --stack-name NeptuneQuickStart \
--template-url "https://steven-neptune-cloudformation-scripts.s3-ap-southeast-2.amazonaws.com/neptune-full-stack-nested-template.json" \
--capabilities "CAPABILITY_NAMED_IAM" "CAPABILITY_AUTO_EXPAND"
```

*The reference to the SSH PEM key has been removed in the above Cloudformation script, so you will not be able to SSH into the Neptune Notebook instance.*

## 3. Setting up a Neptune Notebook

Interacting with the Neptune cluster can be performed through the Neptune workbench. The workbench uses Jupyter notebooks hosted by Amazon SageMaker.

[For more information, view the official documentation here](https://docs.aws.amazon.com/neptune/latest/userguide/notebooks.html)

Workbench resources are billed under Amazon SageMaker, separately from Neptune.

** Important **
In order to use the workbench, the security group that you attach in the VPC where Neptune is running must have an additional rule that allows inbound connections from itself - otherwise you will not be able to connect to the Neptune cluster.

[Please refer to the documentation for IAM Roles and IAM Policies required for the Notebooks](https://docs.aws.amazon.com/neptune/latest/userguide/notebooks.html)

## 4. Interacting with your Neptune Database

Interactions with the Neptune Database will primarily be managed using Neptune Notebooks and Gremlin Syntax. An alternative language is SparQL, however Gremlin syntax will be shown in this document.

### 4.1. Navigating to the Neptune Notebook

4.1.1 Start by navigating to the Amazon Neptune service by using the search field, or the product list on your AWS Web Console.

4.1.2. Within the Neptune service, click on the Sidebar to expand the menu.

![NeptuneSidebar](https://st-summit-2020-resources.s3-ap-southeast-2.amazonaws.com/public/images/neptune-lab/neptune_sidebar.png)

4.1.3. Click on the **Notebooks** option, and click on the **Neptune Notebook** that should have been created for you.

4.1.4. Click on **Open notebook** on the Jupyter notebook summary page. A new tab should appear.

### 4.2 Creating a new IPython Notebook (.ipynb) file

We have successfully navigated to the Jupyter notebook, from the Amazon Neptune web console page. We will now create a new Jupyter Notebook, and run some queries against the Neptune cluster.

4.2.1. Clicking on the New dropdown, and select the **Python 3** option

![image](https://st-summit-2020-resources.s3-ap-southeast-2.amazonaws.com/public/images/neptune-lab/neptune-juypter-setup-01.png)

4.2.2. A new tab should appear with the document ready for input.

### 4.3 Confirming that you can connect to the Neptune Instance

In the first cell, type in `%status` and hit **run**.

![image](https://st-summit-2020-resources.s3-ap-southeast-2.amazonaws.com/public/images/neptune-lab/neptune_status.png)

If everything goes according to plan, you should see an output similar to:

```
{
    "status": "healthy",
    "startTime": "Wed Feb 12 07:21:17 UTC 2020",
    "dbEngineVersion": "1.0.2.1.R4",
    "role": "writer",
    "gremlin": {
        "version": "tinkerpop-3.4.1"
    },
    "sparql": {
        "version": "sparql-1.1"
    },
    "labMode": {
        "ObjectIndex": "disabled",
        "Streams": "disabled",
        "ReadWriteConflictDetection": "enabled"
    }
}
```

----

## 5. Graph Definitions

Before we really get stuck in, we need to go through some definitions which we will refer to during the rest of the lab.

### Vertex / Verticies

A vertex, or vertice, is a denotation representing a discrete object, such as a person, place or event.

### Edge

An edge denotes a relationship between verticies, such as relationships, likes, or having been to a place, etc.

### Properties
A vertex or edge may include properties, which express non-relational information. Examples include but are not limited to:

- A person vertex can include user and age properties
- A Edge can include timestamp and weight

## 6. Graph Basics

Now that we have connected to our Neptune Cluster, and learnt the basic terminologies on verticies and edges, we can start pushing data into the Graph Database.

We need to prefix each *cell* with the `%%gremlin` syntax so the processor understands that the next syntax is a Gremlin query.

Start by clearing all verticies
```
%%gremlin

g.V.drop()
```

and check that we don't have any verticies

```
%%gremlin

g.V()
```

and write a comment

```
%status # this is a comment

```

### 6.1 Create a Vertex

```
%%gremlin

g.addV('person')
```

When adding a Vertex of type 'person', the graph database will automatically generate a unique identifier for that given vertex.

You can specify the exact ID that you wish, by defining the property on creation.

```
%%gremlin

g.addV('person').property(id, "bob")
```

The properties `id` and `label` are reserved attributes for verticies and edges, and therefore are not specified with string denotation.

### 6.2 Create a Vertex with a property
```
%%gremlin

g.addV('person').property('name', 'bob')
```

This syntax allows you to create a vertex, and give it a custom property key and value. 

You can also add multiple properties using the following syntax.

```
%gremlin

g.addV('person')
    .property('age', 17)
    .property('lname', 'jones')
```

### 6.3 Show all Verticies - the V is case sensitive!
```
%%gremlin

g.V() // V is upper case!
```

You can also show all Verticies and their values, by using the `valueMap` syntax.
```
%%gremlin

g.V().valueMap()
```

### 6.4 Drop a specific Vertex

You may need to drop a specific vertex, and can utilise query commands such as `has` to drop. 

```
%%gremlin

g.V('bob').drop()
```

```
%%gremlin

g.V().has('name', 'dan').drop()
```

```
%%gremlin

g.V().hasLabel('person')
```

### 6.5 Edges

This example creates an edge between two verticies
```
%%gremlin

g.V('1').addE('knows').to(g.V('2')).property('likes', 0.6)
```
Breakdown explanation:

1. We select a vertex using `g.V('1')`
2. We add an Edge using `.addE('knows')`
3. We define which node it is, using a selector `.to(g.V('2'))`
4. We define a property of `'likes'` with a value

## 7. Create a simple social recommendation engine

We are now going to attempt to create a simple social recommendation engine, which follows a couple of rules.

1. We have a number of users who are friends with each other
2. Friends of friends, may be recommended
3. For simplicity, we will start with a simple rule that *if number of mutual friends is > 2, recommend the friend*

### 7.1 Creation of the Verticies

The following code will generate a Graph Database with the following verticies, and edges.

![image](https://st-summit-2020-resources.s3-ap-southeast-2.amazonaws.com/public/images/neptune-lab/6.1_simple_users.png)

```
%%gremlin

g.addV('person').property(id, "bob")
 .addV('person').property(id, "jess")
 .V("bob").addE('FRIEND').to(g.V("jess"))
```

We can now check the edges by using "Bob" as a reference vertex, and viewing all `outbound edges`

```
%%gremlin

g.V("bob").outE()
```

```
Total Results: 1
1	e[b8b82641-08fd-dea8-5d35-0cbf5c2393a7][bob-FRIEND->jess]
```

### 7.2 Adding more users

We will expand upon our graph network by adding another user, and making them Jess's friend.

![image](https://st-summit-2020-resources.s3-ap-southeast-2.amazonaws.com/public/images/neptune-lab/62_friend.png)

```
%%gremlin

g.addV("person").property(id, "sarah")
 .addV("person").property(id, "charlotte")
 .V("jess").addE("FRIEND").to(g.V("sarah"))
 .V("jess").addE("FRIEND").to(g.V("charlotte"))
```

```
Total Results: 1
1	e[ceb82646-6ed7-1553-22e2-812b2172753d][jess-FRIEND->charlotte]
```

### 7.3 Friend Recommendation

We can now create a really simple friend recommendation.

```
%%gremlin

g.V("bob").out("FRIEND").out("FRIEND")
```

```
Total Results: 2
1	v[sarah]
2	v[charlotte]
```

What is happening here is that we are `traversing` from `Bob`, and navigating outwards to other Verticies that are `FRIENDS`. We are then traversing from those `Verticies`, out to other `Friends`.

![image](https://st-summit-2020-resources.s3-ap-southeast-2.amazonaws.com/public/images/neptune-lab/63_traversal.png)

### 7.4 Friend Strengths to Improve Recommendations

What if we wanted to improve our friend strength recommendations based on strengths - instead of just a simple friend value?

Well we can utilise a `strength property` to denote whether the traversal applies, and only traverse if the strength is greater than 1.


### 7.4.1 Start by clearing the graph database
```
%% gremlin

g.V().drop()
```

### 7.4.2 Create the following Vertex network

```
%%gremlin

g.addV('person').property(id, "bob")
 .addV('person').property(id, "jess")
 .addV("person").property(id, "sarah")
 .addV("person").property(id, "charlotte")
 .V("jess").addE("FRIEND").to(g.V("sarah")).property("strength", 0.5)
 .V("jess").addE("FRIEND").to(g.V("charlotte")).property("strength", 2)
 .V("bob").addE('FRIEND').to(g.V("jess")).property("strength", 1)
 ```

```
%%gremlin

g.V("bob").outE("FRIEND").has("strength", P.gte(1))
```

The syntax `P` denotes that given an object, evaluate whether the result is `true` or `false`. In this example its "greater-than-or-equal to 1.

```
%%gremlin

g.V("bob").outE("FRIEND").has("strength", P.gte(1)).otherV()
```

otherV = move to another vertex that is not the vertex it came from.

```
%%gremlin

g.V("bob").outE("FRIEND").has("strength", P.gte(1)).otherV().outE("FRIEND").has("strength", P.gte(1))
```

We can now build the traversal to do the following:

1. Given the start from "Bob", traverse outwards on Edges of type "FRIEND" and has "strength" of greater than 1. Move to another vertex that is not the vertex it came from, and then navigate outwards to edges that are also `FRIEND` and has a `strength of greater than 1`. 

This means that Sarah will not appear as the strength is only `0.5`, while Charlotte is `2`.

```
e[4ab8264f-b637-ab4e-5001-596ea652439e][jess-FRIEND->charlotte]
```

Try changing the `gte` value of the queries to see the traversal results change. 

First strength value is > 1, should return 0 results
```
%%gremlin

g.V("bob").outE("FRIEND").has("strength", P.gte(2)).otherV().outE("FRIEND").has("strength", P.gte(0.4))

Total Results: 0
```

Second strength value is > 0.4, should show two results
```
%%gremlin

g.V("bob").outE("FRIEND").has("strength", P.gte(1)).otherV().outE("FRIEND").has("strength", P.gte(0.4))

Total Results: 2
1	e[08b8264f-b637-6bf2-04b4-de13b7682328][jess-FRIEND->sarah]
2	e[4ab8264f-b637-ab4e-5001-596ea652439e][jess-FRIEND->charlotte]
```

## Where to from here?

We have only scratched the surface of what we can accomplish with a Graph Database. We can build even more complex relationships - such as item and music recommendations, while including properties and other rules that change what Verticies are able to be traversed.

Please refer to the Gremlin recipes link included at the bottom of the lab for more inspiration.

----

## Extras

----

## Importing of Data

Amazon Neptune supports bulk loading of external files directly into the Neptune DB instance. This process supports executing a large number of `INSERT`, `addVertex`, `addEdge`, or other API calls.

The Neptune **Loader** supports both RDF (Resource Description Framework) and Gremlin data, and is designed for large datasets.

[Please refer to the official documentation to use the Neptune Loader](https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load.html)

![Image](https://docs.aws.amazon.com/neptune/latest/userguide/images/load-diagram.png)

## Backups
Amazon Neptune supports cluster snapshots and restoration.

[Please refer to the official documentation to learn more](https://docs.aws.amazon.com/neptune/latest/userguide/backup-restore.html)

## High Availability & Failover

Amazon Neptune stores copies of the data in a DB cluster across multiple Availablity Zones in a single AWS Region. You must specify to create Neptune replicas across AZs, and if configured will automatically provision and maintain them synchronously.

If a DB cluster is in a single Availability Zone, you can make it Multi-AZ by adding a Neptune replica to a different Availability Zone.

As of 18-FEB-2020, Neptune supports up to 15 Neptune replicas to process read-only queries.

[Please refer to the official documentation to learn more](https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-availability.html)

## Dates

Neptune does not support Java Date. Use the datetime() function instead. datetime() accepts an ISO8061-compliant datetime string.

It supports the following formats:

- YYYY-MM-DD
- YYYY-MM-DDTHH:mm
- YYYY-MM-DDTHH:mm:SS
- YYYY-MM-DDTHH:mm:SSZ.

```
%%gremlin

g.V().property(single, 'lastUpdate', datetime('2018-01-01T00:00:00'))
```

----

## Iterators and next()

```
%%gremlin
​
g.V().hasLabel('person').properties('age').drop().iterate()
g.V('1').drop().iterate()
g.V().outE().hasLabel('created').drop()
```
Note

The `.next()` step does not work with `.drop()`. Use `.iterate()` instead.

----

## Copy of Neptune Prebuilt (.ipynb) Resources

- [01 - About the Neptune Notebook](https://steven-neptune-notebooks.s3-ap-southeast-2.amazonaws.com/Notebooks/01-Getting-Started/01-About-the-Neptune-Notebook.ipynb)
- [02 - Access Graph with Gremlin](https://steven-neptune-notebooks.s3-ap-southeast-2.amazonaws.com/Notebooks/01-Getting-Started/02-Using-Gremlin-to-Access-the-Graph.ipynb)
- [03 - RDF and SparQL to access the graph](https://steven-neptune-notebooks.s3-ap-southeast-2.amazonaws.com/Notebooks/01-Getting-Started/03-Using-RDF-and-SPARQL-to-Access-the-Graph.ipynb)
- [04 - Social Network Recommendations](https://steven-neptune-notebooks.s3-ap-southeast-2.amazonaws.com/Notebooks/01-Getting-Started/04-Social-Network-Recommendations-with-Gremlin.ipynb)

----

## References

- Official Documentation
    - https://docs.aws.amazon.com/neptune/latest/userguide/get-started-create-cluster.html
    - CF NOTES
        - A key pair is required to access the EC2 instance, and run the CF script. Ensure that you have the PEM file for the key pair which you will select for the CF script.

- Blog Post
    - https://aws.amazon.com/blogs/database/analyze-amazon-neptune-graphs-using-amazon-sagemaker-jupyter-notebooks/

- Best Practices
    - https://docs.aws.amazon.com/neptune/latest/userguide/best-practices.html

- Neptune Developer Resources
    - https://aws.amazon.com/neptune/developer-resources/

- Glue-Neptune
    - https://github.com/awslabs/amazon-neptune-tools/tree/master/glue-neptune

- Kinesis to Neptune
    - https://github.com/aws-samples/amazon-neptune-samples/tree/master/gremlin/stream-2-neptune

- .NET connector
    - https://docs.aws.amazon.com/neptune/latest/userguide/access-graph-gremlin-dotnet.html

- Gremlin Recipes
    - http://tinkerpop.apache.org/docs/current/recipes/#recommendation

## Feedback

If you have any feedback, concerns or would like to have a chat, just send me an email.

## Author 

Steven Tseng (stetseng@amazon.com)

Solutions Architect - Digital Natives MEL/SYD