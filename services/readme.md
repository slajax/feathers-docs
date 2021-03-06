# Services

Services are the heart of every Feathers application. A service can be any JavaScript object that offers one or more of the `find`, `get`, `create`, `update`, `remove` and `setup` service methods. It can be used just like an [Express middleware](http://expressjs.com/en/guide/using-middleware.html) with `app.use('/path', serviceObject)`. A very simple service can looks like this:

```js
import feathers from 'feathers';

const app = feathers();

app.use('/todos', {
  get(id, params) {
    return Promise.resolve({
      id,
      description: `You have to do ${id}!`
    });
  }
});
```

With the [REST provider](rest.html) set up, going to `/todos/dishes` in the browser will return a JSON object like this:

```json
{
  "id": "dishes",
  "description": "You have to do dishes!"
}
```

The complete list of service method signatures is as follows:

```js
const myService = {
  find: function(params [, callback]) {},
  get: function(id, params [, callback]) {},
  create: function(data, params [, callback]) {},
  update: function(id, data, params [, callback]) {},
  patch: function(id, data, params [, callback]) {},
  remove: function(id, params [, callback]) {},
  setup: function(app, path) {}
}

app.use('/my-service', myService);
```

Or as an instance of an ES6 class:

```js
class MyService {
  find(params [, callback]) {}
  get(id, params [, callback]) {}
  create(data, params [, callback]) {}
  update(id, data, params [, callback]) {}
  patch(id, data, params [, callback]) {}
  remove(id, params [, callback]) {}
  setup(app, path) {}
}

app.use('/my-service', new MyService());
```

## Service methods

Service methods should return a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) and use the following parameters:

- `id` is the unique identifier for the resource. A resource is the data identified by a unique id.
- `data` is the resource data
- `params` can contain any extra parameters, for example the authenticated user. `params.query` contains the query parameters from the client (see the [REST](rest.html) and [real-time](real-time.html) providers).
- `callback` can be called instead of returning a Promise. It is a Node-style callback function following the `function(error, data)` convention.

Below is a more detailed description of the purpose of each service method. Consider the `send` method as a placeholder for the [real-time API socket](real-time.html) (e.g. `socket.emit` for Socket.io).

### find

`find(params [, callback])` retrieves a list of all resources from the service. Provider parameters will be passed as `params.query` to the service.

__REST__

    GET todo?status=completed&user=10

__Websockets__

```js
send('todo::find', {
  status: 'completed'
  user: 10
}, function(error, data) {
});
```

Will call .create with `params` as `{ query: { status: 'completed', user: 10 } }`

### get

`get(id, params [, callback])` retrieves a single resource with the given `id` from the service.

__REST__

    GET todo/1

__Websockets__

```js
send('todo::get', 1, {}, function(error, data) {

});
```

### create

`create(data, params [, callback])` creates a new resource with `data`. The callback should be called with the newly created resource data.

__REST__

    POST todo
    { "description": "I really have to iron" }

The body type is determined by the Express [body-parser](https://github.com/expressjs/body-parser) middleware which has to be registered before every service.

__Websockets__

```js
send('todo::create', {
  description: 'I really have to iron'
}, {}, function(error, data) {
});
```

### update

`update(id, data, params [, callback])` replaces the resource identified by `id` with `data`. The callback should be called with the updated resource data.

__REST__

    PUT todo/2
    { "description": "I really have to do laundry" }

__Websockets__

```js
send('todo::update', 2, {
  description: 'I really have to do laundry'
}, {}, function(error, data) {
  // data -> { id: 2, description: "I really have to do laundry" }
});
```

### patch

`patch(id, data, params [, callback])` merges the existing data of the resource identified by `id` with the new `data`. The callback should be called with the updated resource data. Implement `patch` additionally to `update` if you want to separate between partial and full updates and support the `PATCH` HTTP method.

__REST__

    PATCH todo/2
    { "description": "I really have to do laundry" }

__Websockets__

```js
send('todo::patch', 2, {
  description: 'I really have to do laundry'
}, {}, function(error, data) {
  // data -> { id: 2, description: "I really have to do laundry" }
});
```

### remove

`remove(id, params [, callback])` removes the resource with `id`. The callback should be called with the removed resource.

__REST__

    DELETE todo/2

__Websockets__

```js
send('todo::remove', 2, {}, function(error, data) {
});
```

### setup

`setup(app, path)` initializes the service, passing an instance of the Feathers application and the path it has been registered on. If enabled, the SocketIO server is available via `app.io`. `setup` is a great way to connect services:

```js
class TodoService {
  get(id, params) {
    return Promise.resolve({
      id,
      description: `You have to ${id}!`
    });
  }
}

class MyService {
  setup: function(app) {
    this.todo = app.service('todo');
  }

  get(name, params) {
    return this.todo.get('take out trash')
      .then(todo => ({ name, todo }));
  }
}

feathers().use('/todo', new TodoService())
    .use('/my-service', new MyService())
    .listen(8000);
```

You can see the combination when going to `http://localhost:8000/my/test`.

> __Pro tip:__ Bind the apps `service` method to your service to always look services up dynamically:

```js
class MyService {
  setup(app) {
    this.service = app.service.bind(app);
  }

  get(name, params) {
    return this.service('todos')
      .get('take out trash')
      .then(todo => ({ name, todo }));
  }
}
```

## Events

Any registered service will automatically turn into an event emitter that emits events when a resource has changed, that is a `create`, `update`, `patch` or `remove` service call returned successfully. For more information about events, please follow up in the [real-time events chapter](events.html).
