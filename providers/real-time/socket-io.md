# The Socket.io Provider

To expose services via [SocketIO](http://socket.io/) call `app.configure(feathers.socketio())`. It is also possible pass a `function(io) {}` when initializing the provider where `io` is the main SocketIO object. Since Feathers is only using the SocketIO default configuration, this is a good spot to initialize the [recommended production settings](https://github.com/LearnBoost/Socket.IO/wiki/Configuring-Socket.IO#recommended-production-settings):

```js
app.configure(feathers.socketio(function(io) {
  io.enable('browser client minification');  // send minified client
  io.enable('browser client etag');          // apply etag caching logic based on version number
  io.enable('browser client gzip');          // gzip the file

  // enable all transports (optional if you want flashsocket support, please note that some hosting
  // providers do not allow you to create servers that listen on a port different than 80 or their
  // default port)
  io.set('transports', [
      'websocket'
    , 'flashsocket'
    , 'htmlfile'
    , 'xhr-polling'
    , 'jsonp-polling'
  ]);
}));
```

> Note: io.set is deprecated in Socket.IO 1.0. The above configuration will still work but will be replaced with the recommended production configuration for version 1.0 (which isn't available at the moment).

This is also the place to listen to custom events or add [authorization](https://github.com/LearnBoost/socket.io/wiki/Authorizing):

```js
app.configure(feathers.socketio(function(io) {
  io.on('connection', function(socket) {
    socket.emit('news', { hello: 'world' });
    socket.on('my other event', function (data) {
      console.log(data);
    });
  });

  io.use(function (socket, next) {
    // Authorize using the /users service
    app.service('users').find({
      username: socket.request.username,
      password: socket.request.password
    }, next);
  });
}));
```

Similar than the REST middleware, the SocketIO socket `feathers` property will be extended
for service parameters:

```js
app.configure(feathers.socketio(function(io) {
  io.use(function (socket, next) {
    socket.feathers.user = { name: 'David' };
    next();
  });
}));

app.use('todos', {
  create: function(data, params, callback) {
    // When called via SocketIO:
    params.user // -> { name: 'David' }
  }
});
```

Once the server has been started with `app.listen()` the SocketIO object is available as `app.io`.