# API data flow example

Here is a full flow of how a request is being processed at the moment

My target is to give separate dataBase to `tenant A` and `tenant B`

`tenant A` might have 10 users and they will stay in seprate dataBase.
`tenant B` might have 10 users which will stay in another database

## Database connection

### Connection manager

`.env` file for help

```javascript
MONGO_URI = mongodb+srv://<my_datbase_user>:<password>@imssystems.bg1d4.mongodb.net
ADMIN_DB = tenantsDb

```

`dbConnectionManager.js` file :

```javascript
// mongodb+srv://<my_datbase_user>:<password>@imssystems.bg1d4.mongodb.net is the clusters connections string
// where 100 databases are allowed at most for an M2 cpu according to the atlas documentation

let connectionMap = []

const connect = db => mongoose.createConnection(process.env.MONGO_URI+`/${db}?retryWrites=true&w=majority`, {
	useNewUrlParser: true,
	useCreateIndex: true,
	useFindAndModify: false,
	useUnifiedTopology: true
});

exports.connectAllDb = async () => {
	try {
		let adminConnection = await connect(process.env.ADMIN_DB)
		let tenants = await Tetants(adminConnection).find({})
		databaseCluster = adminConnection
		console.log('MongoDB admin Connected...\ntenants:', tenants)
		tenantsMap = tenants
		console.log("Tenants cached successfully.")
	} catch (err) {
		console.error(err);
		// Exit process with failure
		process.exit(1);
	}
};

// this function actually maps a client request to it's relevant database
exports.getConnectionByTenant = (tenantName) => {
	if (/localhost/.test(tenantName)) {
		return databaseCluster.useDb('localhost',{useCache:true})
	}
	return databaseCluster.useDb(tenantName,{useCache:true})
}

```

### Connection resolver

`resolveConnection.js` is a middleware i created that is executed before every request can be processed.
Here this function may help to understand the mapping process of tenants

```javascript
exports.resolveConnection = (req,res,next) => {
    let tenant = req.query.tenant

    if(!tenant) return res.status(400).json({msg:"No tenant specified"})
    let connection = getConnectionByTenant(tenant) // this is comming from the dbConnectionManager.js
  
    if(!connection) return res.status(400).json({msg:"Unauthorized tenant"})
    req.tenantConnection = connection
    next()
}

```

## Data modeling

### Example data model

`userModel.js` is an example model. Here i used mongoose to build the system.

```javascript
const mongoose = require('mongoose');
const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  // more user realted fields
},{timestamps:true});

module.exports = (connection)=>{
    UserSchema.statics = {
        populateUser: function(connection){
            return function(user){}
        },
    }
    return connection.model('user', UserSchema)
};

```

I only included the relavant informations from the model just not to complicate things. Sorry for my brevity.

## An API Endpoint looks like this

`controllers/users.js` is handling the requests. Again i'll make it short not to make things complicated.

```javascript
let UsersModel = require('models/users')

app.get('/api/users',resolveConnection,(req,res)=>{
    try{
        let users = await UserModel(req.tenantConnection).find({})
        res.status(200).json({message:'Users found successfully',users})
    }catch(err){
        logError(err)
        res.status(400).json({messaage:err.message})
    }
})
```
