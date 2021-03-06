# Setting up Local Auth

A `local` auth strategy authenticates users against users in the local database. The [feathers-authentication](https://github.com/feathersjs/feathers-authentication) plugin makes it easy to add local auth to your app.

If you already have an app started, you can simply add the following lines to your server setup.  Just make sure you have used the `bodyParser` and, if applicable, `cors` plugins before you use the `feathers-authentication` plugin.
```js
app.configure(feathersAuth({
  secret: 'your-feathers-auth-secret'
}));
```

See the [feathers-authentication api options](https://github.com/feathersjs/feathers-authentication#options) for all available configuration options.

## Getting Started Tutorial

Here's an example server that uses the [feathers-mongoose](https://github.com/feathersjs/feathers-mongoose) database adapter.  You can use the following examples to get an idea of how it works.

```js
/* * * Import Feathers and Plugins * * */
var feathers = require('feathers');
var hooks = require('feathers-hooks');
var bodyParser = require('body-parser');
var feathersAuth = require('feathers-authentication').default;
var authHooks = require('feathers-authentication').hooks;

/* * * Prepare the Mongoose service * * */
var mongooseService = require('feathers-mongoose');
var mongoose = require('mongoose');
var Schema = mongoose.Schema;
var UserSchema = new Schema({
  username: {type: String, required: true, unique: true},
  password: {type: String, required: true },
  createdAt: {type: Date, 'default': Date.now},
  updatedAt: {type: Date, 'default': Date.now}
});
var UserModel = mongoose.model('User', UserSchema);

/* * * Connect the MongoDB Server * * */
mongoose.Promise = global.Promise;
mongoose.connect('mongodb://localhost:27017/feathers');

/* * * Initialize the App and Plugins * * */
var app = feathers()
  .configure(feathers.rest())
  .configure(feathers.socketio())
  .configure(hooks())
  .use(bodyParser.json())
  .use(bodyParser.urlencoded({ extended: true }))

  // Configure feathers-authentication
  .configure(feathersAuth({
    secret: 'feathers-rocks'
  }));

/* * * Setup the User Service and hashPassword Hook * * */
app.use('/api/users', new mongooseService('user', UserModel))
var service = app.service('/api/users');
service.before({
  create: [authHooks.hashPassword('password')]
});

/* * * Start the Server * * */
var port = 3030;
var server = app.listen(port);
server.on('listening', function() {
  console.log(`Feathers application started on localhost:3030`);
});
```

Please note that the above User service does not include any kind of authorization or access control.  That will require setting up additional hooks, later.  For now, leaving out the access control will allow us to easily create a user.  Here's an example request to create a user (make sure your server is running):

```js
// Create User (POST http://localhost:3030/api/users)
jQuery.ajax({
  url: 'http://localhost:3030/api/users',
  type: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  contentType: 'application/json',
  data: JSON.stringify({
    'username': 'feathersuser',
    'password': 'fowlplay'
  })
})
.done(function(data, textStatus, jqXHR) {
    console.log('HTTP Request Succeeded: ' + jqXHR.status);
    console.log(data);
})
.fail(function(jqXHR, textStatus, errorThrown) {
    console.log('HTTP Request Failed', arguments);
});
```

Once you've created a user, logging in is as simple as `POST`ing a request to the `loginEndpoint`, which is `/api/login` by default.  Here's an example request for logging in:

```js
// Login by email (POST http://localhost:3030/api/login)
jQuery.ajax({
  url: 'http://localhost:3030/api/login',
  type: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  contentType: 'application/json',
  data: JSON.stringify({
    'username': 'feathersuser',
    'password': 'fowlplay'
  })
})
.done(function(data, textStatus, jqXHR) {
  console.log('HTTP Request Succeeded: ' + jqXHR.status);
  console.log(data);
})
.fail(function(jqXHR, textStatus, errorThrown) {
  console.log('HTTP Request Failed', arguments);
});
```
The server will respond with an object that contains two properties, `user` and `token`.  The `user` property contains an object with whatever data was returned from your user service.  You'll notice that it currently includes the password.  You really don't want it exposed, so when you're ready to secure the service, you'll need an additional feathers-hook to remove the password property from the response. 

The `token` property contains a JWT token that you can use to authenticate REST requests or for socket connections.  You can learn more about how JWT tokens work on [jwt.io](http://jwt.io/).


### Authenticating REST Requests

Authenticated REST requests must have an `Authorization` header in the format `'Bearer <token>'`, where the <token> is the JWT token. The header should have the following format:
```
Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IklseWEgRmFkZWV2IiwiYWRtaW4iOnRydWV9.YiG9JdVVm6Pvpqj8jDT5bMxsm0gwoQTOaZOLI-QfSNc
```

Assuming you've set up a todos service, here is full request example you can try out. Be sure to replace `<token>` with a token you've retrieved from your `loginEndpoint`:

```js
// List Todos (GET http://localhost:3030/api/todos)
jQuery.ajax({
  url: 'http://localhost:3030/api/todos',
  type: 'GET',
  headers: {
    "Authorization": "Bearer <token>",
    "Accept": "application/json",
  },
})
.done(function(data, textStatus, jqXHR) {
    console.log("HTTP Request Succeeded: " + jqXHR.status);
    console.log(data);
})
.fail(function(jqXHR, textStatus, errorThrown) {
    console.log("HTTP Request Failed", arguments);
});
```

### Authenticating Socket.io Connections

In order to authenticate a Websocket connection, you must first obtain a token using an Ajax request to your `loginEndpoint`.  You then include that token in the request.  The example below is for Socket.io, but the same `query` key can be passed to Primus.

```js
socket = io('', {
  // Assuming you've already saved a token to localStorage.
  query: 'token=' + localStorage.getItem('featherstoken'),
  transports: ['websocket'], // optional, see below
  forceNew:true,             // optional, see below
});
```

In the above example, the `transports` key is only needed if you, for some reason, need to force the browser to only use websockets.  The `forceNew` key is only needed if you have previously connected an *unauthenticated* Websocket connection and you now want to start an *authenticated* request.

### What's next?
Adding authentication allows you to know **who** users are. Now you'll want to specify **what** they can do. This is done using authorization hooks. Learn more about it in the [Authorization section](authorization.md).