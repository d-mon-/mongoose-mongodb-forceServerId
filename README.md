# Let mongoose support forceServerObjectId 
###### (there is a little bug with mongodb)
## Introduction:
Hello, 
I’m *Olivier* and today I’ll present you a short step by step tutorial to let mongoose insert document in a mongo database without any _id.

First of all, you should never do something like this if you’ve [a lot] less mongod (\<PRIMARY>) processes than node.js processes. 
If you’re asking why, then I should introduce you the default structure of an ObjectId according to the [mongoDB documentation](http://docs.mongodb.org/manual/reference/object-id/):



>* a 4-byte value representing the seconds since the Unix epoch,

>* a 3-byte machine identifier,

>* a 2-byte process id, and

>* a 3-byte counter, starting with a random value.



So, for the same process and host, you’ll be able to create 16 777 216 (counter) unique objectId each seconds (timesteamp). Which is enough for small and medium projects. 

Furthermore, the algorithm use two more fields: the *machine id* and the current *process id* that compute our primary key, those fields will lower even more the risk of collisions between _ids. 

Therefore, like *uuid*, your application is unlikely to generate two similar ObjectIds at the same time. The only condition where a collision might occur is when two different machines with the same “machine identifier” are creating an ObjectId at the same time, with the same process id and counter value. 

In conclusion, ObjectId are quite safe on very [very] large application.

So, because your application are more prone to have more node.js processes than DB processes, we can understand why mongoose disable the **forceServerObjectId** parameter by default.

However, if you’re still reading this tutorial, maybe you really want to force your database to create _ids. If, it’s the case, let me introduce you to the joy of changing the behavior of libraries.

## Step 1.1: changing the moon’
Logically, you already have installed mongoose with *npm* or any other procedure. 

First, you need to set the **forceServerObjectId** property to true. 

To do so, you need to edit in *node_module* the file **/mongoose/lib/drivers/node-module-native/coonection.js** 

and set **o.db.forceServerObjectId** to **true**

> - "Can I use {_id:false} in my Schema now ?" 
> - "Not, yet…"

## step 1.2:changing the  ‘goose

Now, we need to change all save/create queries to avoid the default creation of _ids.
Edit the **lib/model.js** file to remove this portion of code in the **$__handleSave function**:

```js
if (!utils.object.hasOwnProperty(obj || {}, '_id')) {
  setTimeout(function() {
    reject(new Error('document must have an _id before saving'));
  }, 0);
  return;
}
```

now, mongoose will never return an error if you don’t insert an _id before sending the doc to the database.

> - "YES! Now I can send _id less objects!!!"
> - "Sadly, not yet."
> - "HUH?"

## Step 2. Debug mongoDB

Sadly, the **node-mongodb-native** driver doesn’t really support *forceServeObjectId*… (Even in its latest version 2.0.42) I published an [issue ticket about this little problem](https://jira.mongodb.org/browse/NODE-543)

So you'll need to **deal with it** manually!


So if you look at my issue ticket, you’ll notice that 3 documents need to be updated.

###Step 2.1 collection.js
You’ll need to edit the following files in **mongoose/node_module/mongodb/lib/**:
>* **collection.js**
>* **bulk/ordered.js**
>* **bulk/unordered.js**


you can help yourself with the [issue ticket](https://jira.mongodb.org/browse/NODE-543) to find the correct line number.

####In *collection.js*:
#####In **insertMany** function:
modify the condition as follows
```js
// Do we want to force the server to assign the _id key
if(self.s.db.options.forceServerObjectId!==true) {
  // Add _id if not specified
  for(var i = 0; i < docs.length; i++) {
    if(docs[i]._id == null) docs[i]._id = self.s.pkFactory.createPk();
  }
}
```
####In **insertDocuments** function:
add the condition before the for loop.
```js
// Add _id if not specified
if( self.s.db.options.forceServerObjectId!==true) {
  for (var i = 0; i < docs.length; i++) {
    if (docs[i]._id == null) docs[i]._id = self.s.pkFactory.createPk();
  }
}
```

### Step 2.2 BULK! SMACH!

Every time you see this line in **bulk/ordered.js** and **bulk/unordered.js** (or find any new ObjectId() call): 
```js
if(document._id == null) document._id = new ObjectID();
```
replace it with:
```js
if( this.s.collection.s.db.options.forceServerObjectId!==true &&  document._id == null) document._id = new ObjectID();
```


And, you can FINALLY store documents without _id with mongoose by using the parameter **{_id:false}** when you design your schema !




**Author:** *Guérin Olivier*
