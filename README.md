# Let mongoose support forceServerObjectId 
###### (and the little bug in the driver)
**Author:** *Guérin Olivier*

Important: node-mongodb-native < 2.0.43 doesn't support forceObjectId

##Step 1: Mongoose
### Step 1.1: changing the mon’
Logically, you already have installed mongoose with *npm* or any other procedure. 

First, you need to set the **forceServerObjectId** property to true. 

To do so, you need to edit in *node_module* the file **/mongoose/lib/drivers/node-module-native/connection.js** 

and set **o.db.forceServerObjectId** to **true**

> - "Can I use {_id:false} in my Schema now ?" 
> - "Not, yet… you need to remove some stuff"

### step 1.2:changing the  ‘goose

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

now, mongoose will never return an error if you don’t insert an _id before sending the doc to the database. (Usually by setting {_id:false} )

And, you can FINALLY store documents without _id with mongoose by [using the parameter **{_id:false}** when you design your schema!](http://mongoosejs.com/docs/guide.html#_id) But, you shouldn't do it if you're network is not reliable. 

In conclusion, {_id:false} should be use only when you deal with embedded documents/model in mongoose.

## what if I want to insert an ObjectId when {_id:false} is set

Well, mongoose expose the ObjectId type. Therefore, you just need to retrieve it, and create an new object.

```js
var mongoose = require('mongoose');
var ObjectId =  mongoose.Types.ObjectId;
var x = new ObjectId(); //return a new ObjectId()
```

Enjoy! :wink:

