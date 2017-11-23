# node-sse-specification

## Goal

This document specifies an API for managing Server-Sent Events (SSE) in Node.js. Discussion and suggestions for improvements are welcome.

A  [standard](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events)-compliant implementation shall be written once this draft reaches sufficient stability.

**Why another NPM package for Server-Sent Events ?**

Many NPM modules already deal with Server-Sent Events. Most of them (if not all) are unmaintained, poorly documented, and do not give enough guarantees regarding the EventSource specification. 
In general, they do not give enough confidence to use them in a production environment. This document aims at providing a better alternative. 

## General principles

The API SHOULD be intuitive and be expressive enough to handle most relevant use cases with ease

The implementation SHOULD focus on performance and resource efficiency. It should also handle most of the technical details related to SSE, and hide them from the user.
  
The implementation MUST be fully compliant with the EventSource specification. It MUST also have **complete control** over the server response to guarantee this compliance. 

## API proposal

To better make sense of the suggested API, a classical workflow shall first be presented with common usages. The full API documentation comes next. 

### Classical workflow

#### Setup

Server-sent events are managed through a *service* whose class is exported directly from the (currently hypothetical)  `sse-service` module.

    const SSEService = require('sse-service');
    const sseService = new SSEService();
    
The SSE route should then register the connection to the service with `SSEService.register`

**Example in Node.js**

    const http = require('http');
    const SSEService = require('sse-service');
    
    const sseService = new SSEService()
    http.createServer((request, response) => {
      if(request.method === 'GET' && request.url === '/sse')
        sseService.register(request, response);
    }).listen(8080);

**Example in Express.js**

    const http = require('http');
    const express = require('express');
    const SSEService = require('sse-service');
    
    const app = express();
    const sseService = new SSEService();

    // 'next' is ignored by SSEService.register 
    // to prevent subsequent middlewares to mess with the response
    app.get('/sse', sseService.register);
 
    http.createServer(app).listen(8080);

To guarantee soundness with respect to the EventSource specification, it is **imperative** that only the `sseService` handles the sending of headers/data to the response. Any other operation, such as storing metadata in the response object, is allowed outside of the `SSEService` implementation.

#### Reacting to a connection

When a connection is established, the sseService emits the `connection` event, which gives the opportunity to send data to the client right after it has connected.

    sseService.on('connection', sseId => {
      // react to the connection establishment here
    });

#### Sending data

The `sseId` passed from the `connection` event handler is an ID that has been assigned to the response object.

To send data to a *single* connection (identified by `sseId`), use the general-purpose `SSEService.send` :

    sseService.on('connection', sseId => {
      // writes down 'data:greetings\n\n' to the response
      sseService.send('greetings', sseId);
    });
    
This method can be called from anywhere, provided that the `sseService` reference is available.

> In the example above, only a `data` payload is sent to the client. As described in the documentation (next section), 
the `SSEService.send` method allows for sending the event name and the last event ID along with the data. 
Other convenience methods are exposed to manage less common use cases (such as `retry` or comment messages). 
See the doc for more information

#### Broadcasting events    

Sending data to all open connections can be done by omitting the `sseId` :

    sseService.on('connection', sseId => {      
      // writes down 'event:userConnected\ndata:\n\n' to all open connections
      sseService.send('', 'userConnected');
    });
    
The above example broadcasts the 'userConnected' event with no data, indicating that a user just connected to the server. 
It does not give any information regarding that user, because right now only the `sseId` is provided to the event handler. 

In many cases, it would be convenient that some connection metadata is passed along with the `sseId` to the handler, such as authentication information. 

In order to achieve this, we shall mirror the `Express.js` convention that stores metadata associated with the response into `res.locals`. 

For instance, in an Express.js codebase :

    // The 'authenticate' middleware decodes the authentication token from the request, and stores the result in res.locals
    app.get('/sse', authenticate, sseService.register);
 
The 'connection' event emitted from the sseService can now provide the `locals` object along with the `sseId`.
This leads to the following usage, considering an example where a user 'john' connects to the server 

    sseService.on('connection', (sseId, locals) => {
      const {userName} = locals;
      
      // broadcasts 'event:userConnected\ndata:{"userName":"john"}\n\n' to all open connections
      sseService.send({userName}, 'userConnected');
    }); 
    
> Please, note that broadcasting an event implies browsing all open connections to send data. For performance reasons, 
when the number of connections is large, connections may be browsed in a series of asynchronous batches 
to avoid blocking the event loop. This may result in some delays between client responses, depending on the charge on the server. 
    
#### Sending data to a specific subset of open connections

Until now, the `SSEService.send` method either sends data down a specific response, or broadcasts data to all open connections. 
There may however be situations where 
  
  - we don't have the `sseId` of interest when we desire to send an event
  - we want to to target a specific subset of connections
  
For those use cases, instead of an `sseId`, `SSEService.send` can accept a filter function to determine which connections to write data to.

The example below illustrates the idea with `SSEService.unregister` instead of `SSEService.send`, but the principles are exactly the same. In this example, we handle the case where a user has an open SSE connection on one tab in his browser, but also have another tab open on the same website. He logs out from this other tab. We desire to close all SSE connections related to that user.

    app.post('/logout', 
    
      // Authenticates the user and stores the userName in res.locals
      authenticate, 
    
      (req, res, next) => {
        // SSEService.unregister closes all connections for which the filter function passed in argument returns true
        sseService.unregister((sseId, locals) => {
          return locals.userName === res.locals.userName;
        }, next);
      }, 
      
      // Actual log out middleware
      logout
    );
    
Note that the `next` function is passed as a callback to `SSEService.unregister` so the `logout` middleware can be reached if no error has occured.
    
> Note that passing a filter function implies finding connection subsets in `Omega(n)` complexity. As for the broadcasting use case, connections are browsed in asynchronous batches if the number of registered connections is large.  

That is pretty much about it. The full API specification can be found below.

### API documentation

> `http.IncomingMessage` and `http.ServerResponse` respectively correspond to request and response objects from the Node `http` API.

#### `new SSEService([opts])`

  - `opts {Object}` (optional)
  - `opts.heartbeatInterval {number}` (optional) - Number of seconds between heartbeats. 
         Heartbeats are `':heartbeat\n\n'` comments sent periodically to all clients in order to keep their connection alive. 
         If this parameter is set to a negative number, heartbeat is disabled.
         The setInterval that deals with heartbeats is [unReffed](https://nodejs.org/api/timers.html#timers_timeout_unref).
         Defaults to `15` 
  - `opts.maxNbConnections {integer}` (optional) - Maximum number of connections allowed. 
         If set to a negative number, there is no limit. 
         If the number of connections reaches the limit, incoming connections will be ended immediately with a `204` status code. 
         Defaults to `-1`    

While it is allowed to have multiple instances of an `SSEService` on a same server, it is not recommended as it would  multiply the number of open connections to that server, consuming resources unnecessarily.   

#### `SSEService.register(req, res)`

  - `req {http.IncomingMessage}` - The incoming HTTP request
  - `res {http.ServerResponse}` - The server response

Sets up an SSE connection by doing the following :

  - sends appropriate HTTP headers to the response 
  - maintains connection alive with the regular sending of heartbeats, unless disabled in the constructor options
  - assigns an `sseId` to the response and extends the `res.locals` object with an `sse` property :
    - `res.locals.sse.id {object}` : the `sseId` assigned to the response
    - `res.locals.sse.lastEventId {string}` (optional) : the `Last-Event-ID` HTTP header specified in `req`, if any
       
The connection will be rejected if the `Accept` header in the `req` object is not set to 'text/event-stream'.
 
The `SSEService.connection` event is emitted if connection is successful. If the SSE connection fails, an `SSEService.error` event is emitted.    This function accepts no callback, to avoid possible subsequent code to touch the `res` object. Instead, an `error` or `connection` event shall be emitted at the end of the registration.
    
#### `SSEService.unRegister([target[, cb]])`

  - `target {SSEService.SSEID | function}` - The target connection(s). Defaults to `null` (targets all connections)
  - `cb {function}` (optional) - Callback function 

This operation closes the response(s) object(s) matching the `target` argument and frees up resources. If no `target` argument is provided, all connections will be closed.

Once a connection has been closed, it can't be sent down any more data. 

Clients that close the connection on their own will be automatically unregistered from the service.

> **Note** Due to the optional nature of both the `target` and `cb` arguments, if `SSEService.unRegister` is called
> with only one function as its argument, this function will be considered as the callback. This behaviour will be applied to all methods having a `target` argument.

#### `SSEService.send(data[, event[, id[, target[, cb]]]])`

  - `data {*}` - The payload of data to send
  - `event {string}` (optional) - The `event` field's value. Defaults to `null` (no `event` field will be sent)
  - `id {string}` (optional) - The `id` field's value. Defaults to `null` (no `id` field will be sent)
  - `target {SSEService.SSEID | function}` (optional) - The target connection(s). Defaults to `null` (targets all connections)
  - `cb {function}` (optional) - Callback function
  
Sends data to one or multiple clients. The targeted clients are specified by the `target` argument. If the `target` argument is missing, data is sent to all clients.

The `data` argument is passed through `JSON.stringify()` before being sent.

> This method has favored convenience of over covering the wide range of possibilities of the EventSource specification. 
> Less common use cases have their own utility methods (such as `SSEService.sendComment`, etc). 
>
> Maybe should we add an all-purpose `SSEService._send(fields, target, cb)`, where `fields` is a key-value map for all the possible supported fields (e.g. `{event: 'myEvent', data:'some data'}`)  

**Examples**

 - `send({hello: 'world'})` : sends `'data:{"hello":"world"}\n\n'` to all open connections
 - `send({hello: 'world'}, 'greetings', 'e-000')` sends `'id:e-000\nevent:greetings\ndata:{"hello":"world"}\n\n'` to all open connections
 
#### `SSEService.sendComment(comment[, target[, cb]])`

  - `comment {*}` - The comment to send
  - `target {SSEService.SSEID | function}` (optional) - The target connection(s). Defaults to `null` (targets all connections)
  - `cb {function}` (optional) - Callback function
  
**Example** : `sendComment('heart-beat')` sends `:heart-beat\n\n` to all connections

#### `SSEService.sendRetry(retry[, cb])`
  
  - `retry {number}` - Time between connection re-attempt (in seconds)
  - `cb {function}` (optional) - Callback function
  
#### `SSEService.resetLastEventId([cb])`

  - `cb {function}` (optional) - Callback function
  
Resets the Last-Event-ID to the client

#### `SSEService.pipeEvents(eventEmitter, sourceEvent[, opts])`

  - `eventEmitter {EventEmitter}` - The source of the events
  - `sourceEvent {string}` - The name of the source event to pipe
  - `opts {Object}` (optional)
  - `opts.targetEvent {string}` (optional) - The name of event to send to the connection. Defaults to the `sourceEvent` argument
  - `opts.dataTransformer {function}` (optional) - The transformation function to apply on the object passed to the event handler.
         Defaults to the identity function 
  - `opts.target {SSEService.SSEID | function}` (optional) - The target connection(s). Defaults to `null` (targets all connections)

Any `sourceEvent` event emitted from the `eventEmitter` argument shall be piped to `SSEService.send`.
  
#### `SSEService.close([cb])`

  - `cb {function}` (optional) - Callback function

Closes the service by terminating all open connections, and frees up resources. The service won't accept any more connection. 
Further incoming connections will be terminated immediately with a `204` HTTP status code, preventing clients from attempting to reconnect.

#### Class: `SSEService.SSEID`

Object representing an SSE connection.

#### Event: `SSEService.connection`

  - `sseId {SSEService.SSEID}` - Connection's SSE identifier
  - `locals {Object}` - The `res.locals` object of the connection
  
Event emitted when an SSE connection has been established
  
**Example**

    sseService.on('connection', (sseId, locals) => {
      sseService.send('greetings', sseId);   
      sseService.send(locals.userName, 'userConnected');
    });
    
#### Event: `SSEService.error`

  - `err {Error}` - The error

Event emitted when an error occurred during SSE connection's establishment.

## TODO

  - how to handle congestion ? Congestion may occur when a large connection pool needs to be browsed several times in quick succession

## Support

Implementation of this specification is expected to use Node.js core methods to respond to the client, in particular `ServerResponse.writeHead()`, `ServerResponse.write()` and `ServerResponse.end()`.
For this reason, it is **not expected to support Koa.js**, that has its own way of handling responses (see http://koajs.com/#context)