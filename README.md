# GSoC: Holmes Automated Malware Relationships (WIP)

## Introduction

![GitHub Logo](/images/architecture.png)
*Figure 1: System Architecture*

###### Overview

The purpose of this project is to develop a system capable of automatically
identifying and managing the relationships between malware objects (IP addresses,
Domains, Executables, Files etc). This system will use the analytic results as
generated and stored by Holmes-Totem and Holmes-Totem-Dynamic. The goals are:


1. Define the malware attributes necessary for relationship detection through querying.
2. Implement Machine Learning algorithms for relationship detection.
3. Implement an algorithm for aggregating and scoring the final relationships.
4. Visualize relationships for the client.

This system will perform malware relationship detection and scoring by using a range of queries and ML algorithms. We will implement and optimize some existing and new ML algorithms in order to ensure accuracy and efficiency. The whole relationship detection and rating process will go through two stages and at the end the user will receive a visual representation of the generated final relationships.

###### Technology

We will use **Apache Spark** and **Tensorflow** for writing and running the
necessary Queries and Machine Learning algorithms. The system will use a mix of
batch and stream processing so **Spark Streaming** and/or **Apache
Beam** are the framework of choice. **RabbitMQ** is the AMQP library of choice to
support the streaming functionality. The data is stored in **Apache Cassandra**.
Since this is a work in progress, there is a good chance that some new technologies and frameworks may be added along the way. This section will be updated accordingly.

## Defining Relationships

The relationship detection process goes through two stages. The first stage
(Offline Training) is
going to generate the first level of relationships, while the second stage (Final Relationships Generate) will
define the final relationships and their score by using the data created from
the first stage as seed.

From a technical standpoint, the analytics of the first stage happen independently of any user requests. The Query and ML components automatically perform all queries and ML algorithms for new malware analytic results based on specific events and triggers. All of the data generated at this stage is permanently stored on Cassandra using an appropriate schema. (During the rest of the section, I may refer to the primary relationships simply as relationships for simplicity’s sake.)

###### Primary Relationships

There are 4 types of artefacts in our database: IP addresses, Domains, Files, and Binary Executables. All of these types can potentially have a relationship with each other.

*For example:* An executable may issue a call to a specific domain who is associated with one or more IPs, which might be in turn related to other artefacts. In this scenario we already have identified several relationships:
1. Executable <-> Domain
2. Domain <-> IP
3. (and by the transitive property of a bidirectional connection): Executable <-> IP

The whole purpose of this stage of the process is to look for meaningful primary relationships between the different artefacts based on the available analytic results, and store these relationships permanently. The following graph is a high level view of the potential relationship types between artefacts.

![GitHub Logo](/images/Relationship_Types.png)
*Figure 2: High-level view of artefact relationships*

The relationships between artefacts will be defined in detail by the indicators that the Querying and ML components will detect/calculate. The first step of relationships discovery is finding good indicators of relationships between the different artefacts. These indicators are extracted by processing the analytic results from Holmes Totem (Dynamic). The components responsible for performing this analytic processes are the Query and ML components.

###### Final Relationships


 Final relationship (Direct relationship) | Indirect relationships 
  ----------------------------------------- | -------------
  Malware -> Malware | \
  Malware -> Domain  | 1. Malware -> Malware -> IP </br>2. Malware -> IP -> Domain 
  Malware -> IP | 1. Malware -> Malware -> IP </br> 2. Malware -> Domain -> IP  
  Domain -> Malware | 1. Domain -> Malware -> Malware </br> 2. Domain -> Domain -> Malware </br> 3. Domain -> IP -> Malware
  Domain -> Domain | 1. Domain -> Malware -> Domain </br> 2. Domain -> IP -> Domain </br> 3. Domain -> Malware -> Malware -> Domain (optional)
  Domain -> IP | 1. Domain -> Malware -> IP </br> 2. Domain -> Domain -> IP </br> 3. Domain -> IP -> IP
  IP | All IP final relationships are similar to Domain Final relationships
*Table 1: Definitions for Final relationships*

The direct relationships can be retrieved directly from primary relationships. 
It is obviously that column:Final relationship is same to column:Direct relationship, so they are merged into one column.

The indirect relationships need other artefact as intermediary to transfer relationship.


 

## Storage and Schema

The storage schema will store the primary relationships for both the Query and the ML components. The schema should be easily and efficiently queried in order to provide stage 2 with all the necessary data with as little lag and computational effort as possible. The schema table will contain all the relationship_types and relationship_values generated by the Query and ML components. The relationship_types correspond to the ones that can be seen in Figure 1. The possible relationship_values can be seen in Table 1. Currently, Table 1 only includes potential relationship data based on the information provided by the Query Component. The table will be updated as the components are developed.

The table schema developed in this stage should satisfy 2 main queries:

1. **Q1:** Give me all relationships for obj_id
2. **Q2:** Give me all objects that subscribe to relationship_value
3. (**Q3:** Give me all objects that subscribe to relationship_type)

![GitHub Logo](/images/schema.png)

*Figure 3: Table View*

The picture above represents the two main tables that should satisfy the queries from Stage 2.
The original base table easily satisfies Q1. The table created through the MV can satisfy Q2.
Even Q3 can be easily addressed with a slightly different MV.

The relationship values for each primary relationship can be either direct values or references unique identifiers that can be used to query lookup tables for additional details on the relationship value. Lookup tables are generated for a specific subset of relationship values.

###### Final relationship storage
The final relationship storage is used to extract the query result when an artefact is queried second time or more. In storage, we do not distinguish between direct and indirect relationships and we use one table to save all final relationship.

	final_relationship_table(
		query_object_id text,
		relationship_object_id text,
		relationship_type text,              
		confidence_score int,
	same_indicators_between_object  list,     
		TTL int,
		PRIMARY KEY ( query_object_id, relationship_type )
		) 

1). query\_object\_id and relationship\_object\_id consist of the relationship pair.

2). relationship\_type represents the relationship type between query\_object and relationship\_object and is granularity. It can be malware\_direct\_domains,  malware\_malware\_indirect\_domains, malware\_IP\_indirect\_domains and so on.
We take malware\_IP\_indirect\_domain as an example. This relationship is indirect relationship and relationship are malware <-> IP <-> domain.

3). same\_indicators\_between\_object is a list and used to display the relationships in the website.


## Implementation

#### Offline Search and Training

###### Query Component


This component will look for atomic indicators of relationships. Atomic indicators are either identical values that can be shared by artefacts’ analytic results or calculated values that are used to provide some measure of similarity between artefacts. The Query component will generate the following relationship_types and values. These primary relationships will also have an assigned weight that can be used by the second stage of the process to calculate the final relationships.

 (relationship_type, service, relationship_value)  | weight_definition
 ------------------------------------------------- | -----------------
 (imphash_similar_to, PEInfo, imphash)  |   
 (pehash_similar_to, PEInfo, pehash) |  
 (signed_by, PEInfo, signature) |  
 (communicated_with_ip, CUCKOO, ip) |  
 (communicated_with_dom, CUCKOO, domain) |  
 (related_to_ip, DNSMeta, ip) |  
 (resolves_to_ip, DNSMeta, A Record )  |  
 (resolves_to_ip, DNSMeta, AAAA Record)  |  
 (related_to, DNSMeta, metadata)  |  
 (av_similar_to, VirusTotal, signature_similarity) |  
 (yara_similar_to, YARA, complex_AV_match) |  

*Table 2: Definitions for primary relationships*



###### ML Component

This component will utilize ML algorithms to train models based on a labeled dataset and then assign every new unknown incoming artefact (depending on the type of artefact) to one of the trained malicious clusters/classes.


