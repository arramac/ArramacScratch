DocumentDB server side JavaScript SDK
====
DocumentDB’s language integrated, transactional execution of JavaScript supports stored procedures, triggers and user defined functions (UDFs) written natively in JavaScript. This allows developers to write application logic which can be shipped and executed directly on the database storage partitions. JavaScript support at the server side has a number of intrinsic advantages that can be utilized to build rich applications:

Here we look at Javascript SDK, and the key classes that can be used to interact with DocumentDB JSON documents.

Context Object
---
The **Context** object provides access to all operations that can be performed on DocumentDB data, as well as access to the request and response objects. 

```js
function () {
        /* the context method can be accessed inside stored procedures and triggers*/
        var context = getContext();
        
        /* access all database operations - CRUD, query against documents in the current collection */
        var collection = context.getCollection();

        /* access HTTP request body and headers for the procedure */
        var request = context.getRequest();
        
        /* access HTTP response body and headers from the procedure */
        var response = context.getResponse();

    }

```

Request Object
---
Inside the stored procedure or trigger, the **Request** object can be used to manipulate the request message associated with the operation. For example, here we modify the document content inside a pre-trigger used on a create.


```js

    /* read the request from the context*/
    var request = context.getRequest();

    var documentToCreate = request.getBody();
    documentToCreate["createdTime"] = new Date().getTime();
    
    request.setBody(documentToCreate);
    
```

Response Object
---
The **Response** object can be similarly used to modify the response from a stored procedure or trigger.

```js
    /* read the response from the context*/
    var response = context.getResponse();

    /* modify the raw HTTP response from the procedure */
    response.setBody("Hello, World");

```

Collection Object
---
The **Collection** object supports create, read, update and delete (CRUD) against documents in the current collection. Here's how you create a document.

```JavaScript
    var docToCreate = { 
        name: "artist_profile_1023",
        artist: "The Band",
        albums: ["Hellujah", "Rotators", "Spinning Top"]
    };

    /* crate document transactionally */
     collection.createDocument(collection.getSelfLink(), docToCreate, 
         function(err, doc, options) {
                  if(err) throw "Unable to create doc, aborting transaction";
           });


```
Replacing or deleting documents is similar, except that you have to pass in the document's self link. 

```JavaScript
    /* replace document transactionally */
     collection.replaceDocument(doc._self, doc, 
         function(err, docReplaced) {
                  if(err) throw "Unable to update metadata, aborting transaction";
           });


```
Documents can be read using the readDocument and readDocuments (scan) methods. You can query documents within a collection using the DocumentDB SQL syntax. These are processed against efficiently against the index.

```JavaScript
    /* query for a JSON path in the documents */
    collection.queryDocuments(
        collection.getSelfLink(), 
        'SELECT * FROM books b WHERE b.author.name = "Leo Tolstoy"',
        queryCallback);

```

All DocumentDB operations (stored procedures, triggers and UDFs) must complete within the server specified request timeout duration of 5 seconds. If an operation does not complete with that time limit, the transaction is rolled back. JavaScript functions must finish within the time limit or implement a continuation based model to batch/resume execution.

In order to simplify development of stored procedures and triggers to handle time limits, all functions under the collection object (for create, read, replace and delete of documents and attachments) return a boolean return value which represents whether that operation will complete. If this value is false, it is an indication that the time limit is about to expire and that the procedure must wrap up execution.  Operations queued prior to the first unaccepted store operation are guaranteed to complete if the stored procedure completes in time and does not queue any more requests.

Request and Feed Options
------
Additional options can be passed into requests using the FeedOptions, ReadOptions, ReplaceOptions and DeleteOptions classes. These include properties accessible through the REST API like etag, pageSize and continuation.

```js
    // Use a continuation token to paginate over results
    var options = { continuation: nextContinuationToken };
    collection.queryDocuments(collection.getSelfLink(),  "SELECT * FROM Books", options, 
        queryCallback) :

```

Example: Bulk Import
----
Below is an example of a stored procedure that is written to bulk-import documents into a collection. Note how the stored procedure handles bounded execution by checking the Boolean return value from createDocument, and then uses the count of documents inserted in each invocation of the stored procedure to track and resume progress across batches.

```js
function bulkImport(docs) {
    var collection = getContext().getCollection();
    var collectionLink = collection.getSelfLink();

    // The count of imported docs, also used as current doc index.
    var count = 0;

    // Validate input.
    if (!docs) throw new Error("The array is undefined or null.");

    var docsLength = docs.length;
    if (docsLength == 0) {
        getContext().getResponse().setBody(0);
    }

    // Call the create API to create a document.
    tryCreate(docs[count], callback);

    // Note that there are 2 exit conditions:
    // 1) The createDocument request was not accepted. 
    //    In this case the callback will not be called, we just call setBody and we are done.
    // 2) The callback was called docs.length times.
    //    In this case all documents were created and we don’t need to call tryCreate anymore. Just call setBody and we are done.
    function tryCreate(doc, callback) {
        var isAccepted = collection.createDocument(collectionLink, doc, callback);

        // If the request was accepted, callback will be called.
        // Otherwise report current count back to the client, 
        // which will call the script again with remaining set of docs.
        if (!isAccepted) getContext().getResponse().setBody(count);
    }

    // This is called when collection.createDocument is done in order to process the result.
    function callback(err, doc, options) {
        if (err) throw err;

        // One more document has been inserted, increment the count.
        count++;

        if (count >= docsLength) {
            // If we created all documents, we are done. Just set the response.
            getContext().getResponse().setBody(count);
        } else {
            // Create next document.
            tryCreate(docs[count], callback);
        }
    }
}

```

References
-----
* DocumentDB server side JavaScript Tutorial
* DocumentDB Node.js SDK reference
* DocumentDB .NET SDK reference




