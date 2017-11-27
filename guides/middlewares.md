# SSE - middlewares

> **[Stability](https://nodejs.org/dist/latest-v8.x/docs/api/documentation.html#documentation_stability_index) :** 1 - Experimental

The API proposed in this project is minimalist. It is therefore essential for users to extend it with plug-ins (or 'middlewares')

**This section is at the very early stage of brainstorming. Inaccuracies may be present. No formal API is drafted yet.**

## Open discussion

Middlewares are functions that can be plugged-in to the `sseService`, like so :

    sseService.use(myMiddleware);
    
Each `use` method adds a middleware to the stack. As `use` returns the sseService, calls to this method can be chained together.

Once we have our middleware stack, when do we call it ? One natural approach would be to call the stack around `sseService.register(req, res)`. We say 'around' because we need yet to decide if we call the middleware stack before or after this method.  

It is probably better to support both cases. Consider the [Server-Side Requirements](https://github.com/Yaffle/EventSource#server-side-requirements) for the EventSource polyfill. 
We want to implement a middleware that supports those requirements.

Depending on the condition, one time you need to call a middleware before the `sseService.register` method, the other time you need to call it after the method :

**before** 

> *"Last-Event-ID" is sent in a query string (CORS + "Last-Event-ID" header is not supported by all browsers)*

This condition can be implemented as a *before* middleware, by normalizing the request to ensure `req.headers['Last-Event-ID']` is set from the possible query parameter.

Other idea as a *before* middleware : anything that need to be extracted from the request and put in res.locals. 
Even though this can be implemented in a middleware for express.js, this would split the SSE logic into separate technologies. 
So it is preferable to have a *before* middleware in any case

**after**

> *It is required to send 2 KB padding for IE < 10 and Chrome < 13 at the top of the response stream*

This condition can be implemented in an *after* middleware, by sending the required padding right after the registration (before the 'connection' event is fired)

**When neither before or after work**

> *You need to send "comment" messages each 15-30 seconds, these messages will be used as heartbeat to detect disconnects*

The regular sending of comments is already handled natively by the SSEService. But imagine that it weren't, needing us to implement a middleware to enforce it. 

This feature clearly can't be handled *before* `sseService.register` because in this land we can't send either headers or data.
And in fact, even though possible, it wouldn't be desirable to implement an *after* middleware neither. 
Since we only have the sseId at our disposal, this would imply setting a timeout for every incoming connection. This would be a disaster that would flood the event loop. 

So, how do we handle the use case ? Currently, the only smart solution we can think of is to allow the use of middlewares at another level of the lifecycle (in addition to 'before' and 'after' `sseService.register` options). 
For instance, at the core. Core middlewares would have access to the `sseService` instance. It is unsure that we want them to access the internals, such as dataStructures containing references to res and all sseIds

In the case we're dealing with right now, the reference to the `sseService` suffices, as just need to send data to all open connections

Assuming all this, we can in fact converge towards the following pattern : middlewares are functions that return an object providing lifecycle hooks for the `sseService`.

    function eventSourcePolyfillMiddleware(sseService) {
      return {
        core() {
          // e.g. setInterval(() => sseService.send(':keep-alive'), 15000) 
        }
  
        beforeRegister(req, res, next) {
          req.headers['Last-Event-ID'] = extractLastEventIdFromQuery(req.query) || req.headers['Last-Event-ID'];
          next();
        }

        afterRegister(sseId, locals, next) {
          sseService.send({comment:`${new Array(2049).join(' ')}\n`}, sseId);
          next();
        }
        
        beforeSend(opts, target, next){
          // nothing to do
        }
        
        afterSend(err, next){
          // nothing to do
        }
      };
    }
    
