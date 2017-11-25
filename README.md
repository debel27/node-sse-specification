# node-sse-specification

An unopinionated, minimalist, [standard](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events)-focused API for managing Server-Sent Events (SSE) in Node.js.

Discussion and suggestions for improvements are welcome.

**Node.js**

    const http = require('http');
    const SSEService = require('sse-service');
    
    const sseService = new SSEService()
    http.createServer((request, response) => {
      if(request.method === 'GET' && request.url === '/sse')
        sseService.register(request, response);
    }).listen(8080);
    
    sseService.on('connection', ({sseId}) => {
      // sends data to one response
      sseService.send('hello', sseId);
       
       // broadcasts data to all responses
      sseService.send({event:'user-connected'});
    });

**Express.js**

    const http = require('http');
    const express = require('express');
    const SSEService = require('sse-service');
    
    const app = express();
    const sseService = new SSEService();

    app.get('/sse', sseService.register);
 
    http.createServer(app).listen(8080);

    sseService.on('connection', ({sseId}) => {
      // sends data to one response
      sseService.send('hello', sseId);
       
       // broadcasts data to all responses
      sseService.send({event:'user-connected'});
    })

## API

### Concepts

Server-Sent Events are entirely managed through an `sseService`, that provides convenience methods to send data to one, several or all open connections.

When matching the route for Server-Sent Events, the server **must** delegate the request to the `sseService`. Only the `sseService` is allowed to write headers/data to the response. 

### Core

#### `new SSEService([opts])`

  - `opts {Object}` (optional)
  - `opts.heartbeatInterval {number}` (optional) - Number of seconds between heartbeats. 
         Heartbeats are `':heartbeat\n\n'` comments sent periodically to all clients in order to keep their connection alive. 
         If this parameter is set to a negative number, heartbeat is disabled.
         The setInterval that deals with heartbeats is [unReffed](https://nodejs.org/api/timers.html#timers_timeout_unref).
         Defaults to `15` 
  - `opts.maxNbConnections {integer}` (optional) - Allowed maximum number of simultaneously open SSE connections. 
         If set to a negative number, there is no limit. 
         If the number of open SSE connections reaches the limit, incoming connections will be ended immediately, following the congestion avoidance strategy.
         Defaults to `-1`
  - `opts.congestionAvoidanceStrategy {SSEService.CongestionAvoidanceStrategy}` (optional)

> While it is allowed to have multiple instances of an `SSEService` on a same server, it is *not* recommended as doing so would multiply the number of open connections to that server, consuming resources unnecessarily.
>
> Instead, it is preferable to have a single route for Server-Sent Events per server, and implement event management at the application level. 

#### `SSEService.close([cb])`

  - `cb {function}` (optional) - Callback function

Closes the service by terminating all open connections, and frees up resources. The service won't accept any more connection. 
Further incoming connections will be terminated immediately with a `204` HTTP status code, preventing clients from attempting to reconnect.

### Connection management

> `http.IncomingMessage` and `http.ServerResponse` respectively correspond to request and response objects from the Node `http` API.

#### Class: `SSEService.SSEID`

Object representing an open SSE connection on the server

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
 
This function accepts no callback, to avoid subsequent code to possibly sending data to the `res` object. 
Instead, the `SSEService.connection` event is emitted if connection was successful. The `SSEService.error` event is emitted if there was an error during registration.
    
#### `SSEService.unRegister([target[, cb]])`

  - `target {SSEService.SSEID | function}` - The target connection(s). Defaults to `null` (targets all connections)
  - `cb {function}` (optional) - Callback function 

This operation closes the response(s) object(s) matching the `target` argument and frees up resources. If no `target` argument is provided, all connections will be closed.

Once a connection has been closed, it can't be sent down any more data. 

Clients that close the connection on their own will be automatically unregistered from the service.

> **Note** Due to the optional nature of both the `target` and `cb` arguments, if `SSEService.unRegister` is called
> with only one function as its argument, this function will be considered as the callback. This behaviour will be applied to all methods having a `target` argument.

### Sending data

#### `SSEService.send(opts[, target[, cb]])`

  - `opts {Object}`
  - `opts.data {*}` (optional) - Defaults to the empty string
  - `opts.event {string}` (optional) - If falsy, no `event` field will be sent
  - `opts.id {string}` (optional) - If falsy, no `id` field will be sent
  - `opts.retry {number}` (optional) - If falsy, no `retry` field will be sent
  - `opts.comment {string}` (optional) - If falsy, no comment will be sent
  - `target {SSEService.SSEID | function}` (optional) - The target connection(s). Defaults to `null` (targets all connections)
  - `cb {function}` (optional) - Callback function
  
General-purpose method for sending information to the client. Convenience methods are also available :
 
 - `sendData(data[, target[, cb]])`
 - `sendEvent(event, data[, id[, target[, cb]]])` 
 - `sendComment(comment[, target[, cb]])`
 - `sendRetry(retry[, target[, cb]])`
  
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

### Events

#### Event: `SSEService.connection`

  - `sseId {SSEService.SSEID}` - Connection's SSE identifier
  - `locals {Object}` - The `res.locals` object of the connection
  
Event emitted when an SSE connection has been successfully established
  
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

Supports Node.js 6.x and above.

Implementation of this specification is expected to use Node.js core methods to respond to the client, in particular `ServerResponse.writeHead()`, `ServerResponse.write()` and `ServerResponse.end()`.
For this reason, it is **not expected to support Koa.js**, that has its own way of handling responses (see http://koajs.com/#context)