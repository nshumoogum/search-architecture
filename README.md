# search-architecture

Source of truth for backend processing of search data for ONS sitewide search.

## Sequence Diagrams

The source script for the each diagram is maintained in sequence folder.

To update a diagram:
- Copy the source script into [sequencediagram.org](http://sequencediagram.org)
- Update a diagram as required
- Export as an image on [sequencediagram.org](http://sequencediagram.org)
- Copy the updated source script back into `sequence/<name>.txt` and commit.

### Add/update search documents when a collection is published

![Publish Search Data](sequence/PublishSearchData.png)

#### Steps

Pre-requisite steps to publishing data not in this workflow as we will not be changing that part of the process. The following steps follow
on from a florence (DP internal user) publishes a collection that can contain 1 to many ons webpages or datasets by making a request to zebedee
which does he publishing of new webpages or updates to existing pages and lastly triggering updates to sitewide search; which is where the flow begins.

**1: Kafka messages**

On publishing a collection, write kafka message

```
Datastore: kafka

Topic: search-data-extraction
Record: {
    "job_id": string, // Empty
    "collection_id": string,
    "search_index": string // Use alias, `ONS`
}
```

**2: Retrieve docs for collection**

Consume kafka message in 1 and request Collection docs from Zebedee API endpoint.

**3: Read JSON file from disc**

Retrieve a list of documents for a specific collection from zebedee content (files on disc)

**4: Get dataset docs for collection**

This could be in parallel with 2 to increase performance.

```
Method: GET
Path: /datasets?collection_id=<collection id>
```

*Note: Dataset API needs to be extended to handle query parameter*

**5: Find all datasets for collection**

Call mongoDB with Find query include filter on collection id

**6: Kafka messages**

For each data type send a message to kafka topic 

```
Datastore: kafka

Topic: search-data-import
Record: {
    "data_type": string,
    "job_id": string, // empty
    "search_index": string, // Should use search alias ONS
    "cdid": string,
    "dataset_id": string,
    "description": string,
    "edition:: string,
    "keywords": string,
    "meta_description": string,
    "release_date": string // date format: ISO8601 or strict_date_optional_time||epoch_millis to match existing docs in search?
    "summary": string,
    "title": string,
    ... other fields we decide need to be in search
}
```

**7: Get Multi-operational request to update/add docs**

Documents received via consumption of kafka topic `search-data-import`, store documents in memory until 500 messages consumed or time from first 
message exceeded 5 seconds before making bulk request. 5 second limit will allow for the last set of messages to still be reindexed.

```
Datastore: elasticsearch
Index: ONS

Method: POST
Header: 'Content-Type: application/json'
Path: /_bulk
Body: { "update": { <search doc - Depends on data type>, "_index": "ONS" } }
```

The search document should set an `_id` field that matches the unique identifier for that document, either uuid or compound identifier (multiple fields to represent a documents uniqueness). Either way this should be determined by the data stored in Zebedee content and the dataset API.


See [Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html)

### Reindex

![Reindex Search](sequence/ReindexSearch.png)

#### Steps

**1: Http POST Reindex job**

```
Datastore: elasticsearch

Path: /search-reindex/jobs
Path parameters: none
Header: Authorization: Bearer <token>
Body: none
```

**2: Retrieve list of in-progress jobs**

```
Datastore: mongoDB
Database: search
Collection: jobs
Filter job docs by state and time

BSON Document: {
    "state": string, // enumurated list, in-progress, failed, completed
    "reindex_started": ISODate, // ISO8601 timestamp,
    "id": string,
    ...
}
```

**Validation** - Check there are no jobs that are in-progress and created within the last hour (depends on how long a reindex takes).
If validation fails return status code 409 (conflict), message 'existing reindex job in progress'.

**3: Create new job doc in mongoDB**

```
Datastore: mongoDB
Database: search
Collection: jobs

BSON Document: {
    "id": string, // Generated unique id
    "last_updated": ISODate, // ISO8601 timestamp
    "links" : {
        "self" : string // format: http://localhost:<PORT>/search-reindex/jobs/<id>
    },
    "reindex_started": ISODate, // ISO8601 timestamp
    "reindex_completed": ISODate, // Empty
    "search_index_name": string, // format: {ONS-<ISO8601 timestamp>}
    "state": string // set to in-progress
}
```

**4: Create new sitewide search index (new-ONS)**

Call elasticsearch cluster to create new index.

```
Index: ONS-<ISO8601 timestamp>
Alias: new-ONS
```

**5: Go routine to create event for all collections**

Start a go routine to handle further steps 7, 8 and 9. This will free up the handler to respond to reindex job request in 6.

**6: Successful response**

```
Status Code: 201
Body: {
    "id": string, // unique id for job
    "last_updated": ISODate, // ISO8601 timestamp
    "links" : {
        "self" : string // format: http://localhost:<PORT>/search-reindex/jobs/<id>
    },
    "reindex_started": ISODate, // ISO8601 timestamp
    "search_index_name": string, // format: {ONS-<ISO8601 timestamp>}
    "state": string // set to in-progress
}
```

**7: Retrieve list of collections**

Call Zebedee to retrieve list of collections via API, should contain a list of collections ids and the number of documents for each one. We are only interested in published collections.

**8: Update reindex job ...**

... with a list of collection ids and the total document count

```
Datastore: mongoDB
Database: search
Collection: jobs
Document Identifier (id): job.ID

BSON Document update:
{
    "collections": [string],
    "total_search_documents": integer
}
```


**9: Send kafka messages**

For each collection (id) send a message to kafka topic 

```
Datastore: kafka

Topic: search-data-extraction
Record: {
    "job_id": string,
    "collection_id": string,
    "search_index": string
}
```

**10: Get Docs from collection**

Consume kafka message in 9 and request Collection docs from Zebedee API endpoint.

**11: READ JSON file**

Retrieve a list of documents for a specific collection from zebedee content (files on disc)

**12: Get CMD dataset docs for collection**

This could be in parallel with 10 to increase reindex performance.

```
Method: GET
Path: /datasets?collection_id=<collection id>
```

*Note: Dataset API needs to be extended to handle query parameter*

**13: Find all datasets for collection**

Call mongoDB with Find query include filter on collection id

**14: Send Kafka messages**

For each data type send a message to kafka topic 

```
Datastore: kafka

Topic: search-data-import
Record: {
    "data_type": string,
    "job_id": string,
    "search_index": string,
    "cdid": string,
    "dataset_id": string,
    "description": string,
    "edition:: string,
    "keywords": string,
    "meta_description": string,
    "release_date": string // date format: ISO8601 or strict_date_optional_time||epoch_millis to match existing docs in search?
    "summary": string,
    "title": string,
    ... other fields we decide need to be in search
}
```

**15 Bulk query to add docs to search index**

Documents received via consumption of kafka topic `search-data-import`, store documents in memory until 500 messages consumed or time from first 
message exceeded 5 seconds before making bulk request. 5 second limit will allow for the last set of messages to still be reindexed.

```
Datastore: elasticsearch
Index: search_index

Method: POST
Path: /_bulk
Body: Depends on data type
```

See [Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html)

**16: Update uploaded document count for reindex job**

```
Method: PUT 
Path: /search-reindex/jobs/{id}/inserted-documents/{count}
```

**17 Update job doc**

```
Datastore: mongoDB
Database: search
Collection: jobs
Document Identifier (id): job.id

BSON Document update: {
    "total_inserted_search_documents": integer
}
```

*Note: When updating the count, this should be adding to the current value*

Check counts match: `"total_inserted_search_documents" is equal to "total_search_documents"`

**18 Validate search index document count**

Call elasticsearch to check the index count matches the `total_search_documents` value.

```
Datastore: elasticsearch

Method: GET
Path: /{index}/_count
```

Wait before making a new request (attempt) to a maximum of 3?

**19 Multi-operational request to update aliases**

See [index alias API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html) to see how this is done

```
Datastore: elasticsearch

Method: POST
Path: /_aliases
Body: {
    "actions" : [
    {"remove": {"index" : "<new-index>", "alias" : "new-ONS"}},
    {"remove": {"index" : "<old-index>", "alias" : "ONS"}},
    {"add" : {"index" : "<new-index>", "alias" : "ONS" }},
    {"remove-index" : {"index" : "<old-index>"}},
  ]
}
```

Note: alias `new-ONS` may not be needed in performing a reindex, it may just be less cumbersome and more obvious to a developer to use
new-ONS and ONS to refer to the new index and an existing index respectively instead of the index name (e.g. comparing 2 ONS-<datetime> values).