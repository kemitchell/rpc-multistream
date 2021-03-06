
rpc-multistream is similar to [rpc-stream](https://github.com/dominictarr/rpc-stream) but you can:

* Return streams from both sync and async remote functions
* Return multiple streams per call
* Have multiple callbacks per call
* Have infinite remote callback chains (like [dnode](https://github.com/substack/dnode))
* Mix and match efficient binary streams with text and objectMode streams.

rpc-multistream uses streams2 and [multiplex](https://github.com/maxogden/multiplex) under the hood, so is efficient for binary streams.

If you need authentication then check out [rpc-multiauth](https://github.com/biobricks/rpc-multiauth).

# Usage 

```javascript
var fs = require('fs');
var rpc = require('rpc-multistream');

var server = rpc({
  foo: rpc.syncStream(function() {
    return fs.createReadStream('foo.txt');
  }),
  bar: function(cb) {
    console.log("bar called");
    cb(null, "bar says hi");
  }
});

var client = rpc();

client.pipe(server).pipe(client)

client.on('methods', function(remote) {

  var stream = remote.foo();
  stream.on('data', function(data) {
    console.log(data);
  });

  remote.bar(function(err, msg) {
    console.log(msg);
  });
});
```

# Options

These are are the defaults:

```javascript
rpc({ ...methods... }, { 

  init: true, // automatically send rpc methods manifest on instantiation
  encoding: 'utf8', // default encoding for streams
  objectMode: false, // default objectMode for streams
  explicit: false, // include encoding/objectMode even if they match defaults
  debug: false, // Enable debug output
  flattenError: <function_or_false>, // function for serializing errors, or false
  expandError: <function>, // function for deserializing errors, or false
  onError: <function> // function to run on orphaned errors, or false
}
```

If you set init to false then the list of functions will not be sent and the remote end will not emit a 'methods' event until you call rpc.init().

The encoding and objectMode options will be used as for streams returned by synchronous-style calls unless you explicity define these per-stream. If you set the encoding and objectMode options to the same on both ends then all streams that have these options will forego sending the options across the stream, saving you bandwidth. If you don't set encoding and objectMode to the same on both ends then you _must_ set explicit to true, which turns off this bandwidth-saving feature.

For the error-related options see the next section.

# Error handling

If an error occurs internally in rpc-multistream while calling a remote function, and the last argument to the remote function is a function, then that last-argument-function will be treated as the primary callback and will called with an error as first argument. Likewise if an uncaught exception occurs while calling a function with a callback then the exception will be converted to an error and passed as the first argument to the assumed callback. See examples/asyncException.js for a demonstation.

For synchronous calls that return one or more streams, an 'error' event is emitted on all streams.

By setting flattenError and expandError you can change how rpc-multistream serializes and deserializes error objects. The default results in an Error object with the original .message intact to be re-created on the remote end. Disable by setting both to false or overwrite with your own like so:

```javascript
var rpc = require('rpc-multistream');

var endpoint = rpc({ ... some methods ... }, {
  flattenError: function(err) {
    // prepare err before serialization here
    return err; 
  },
  expandError: function(err) {
    // process err after serialization here
    return err;
  }
});
```

Orphaned errors are errors where neither a callback nor a stream exists that can be used to report the error back to the caller. The default action is to emit an 'error' event on both ends of the parent rpc-multistream stream. Disable this by setting onError to false or overwrite with your own function like so:

```javascript
var rpc = require('rpc-multistream');

var endpoint = rpc({ ... some methods ... }, {
  onError: function(err) {

  }
});
```

# Bi-directional RPC

There is no difference between server and client: Both can call remote functions if remote end has specified any functions.

# Synchronous calls

If you declare a function with no wrapper then rpc-multistream assumes that is is an asynchronous function.

It is also possible to define functions that directly return only a stream by wrapping your functions using e.g:

```javascript
var server = rpc({
  foo: rpc.syncReadStream(function() {
    return fs.createReadStream('foo.txt');
  })
};
```

See examples/sync.js for a demo.

The following wrappers exist:

* rpc.syncStream: For functions returning a duplex stream
* rpc.syncReadStream: For functions returning a readable stream
* rpc.syncWriteStream: For functions returning a writable stream

It is _not_ possible to define synchronous functions that return something other than a stream. Why not? Because the function call would have to block until the server responded. For synchronous functions returning streams the streams are instantly created on the client and when the server creates the other endpoint of the stream at some later point in time the two streams are piped together.

For synchronous functions remote errors are reported via the returned stream emitting an error. This is true even if an exception occurs before the remote stream has been created. Here's how it works:

```javascript
var server = rpc({
  myFunc: rpc.syncReadStream(function() {
    throw new Error("I am an error!");
  })
});

// ... more code here ...

var stream = remote.myFunc()
stream.on('error', function(err) {
  console.error("Remote error:", err);
});
```

# Per stream options

For streams returned via callbacks it will be auto-detected whether the stream is a readable, writable or duplex stream and both encoding and objectMode will be auto-detected and set correctly on both ends.

For streams returned via synchronous-style calling there is no way to know in advance what the remote stream options are going to be. If you do not specify any options then the encoding and objectMode from the parent rpc-multistream options will be used. To explicitly specify:

```javascript
var server = rpc({
  foo: rpc.syncReadStream(function() {
    return fs.createReadStream('foo.txt');
  })
};
```

# Gotchas 

Either both ends must agree on the following opts (as passed to rpc-multistream):

* encoding
* objectMode

or you must set explicit to true. Setting explicit to true will cost more bandwidth since each call will include stream options even if they match your defaults.

The streams returns by async callbacks are currently all duplex streams, no matter if the original stream on the remote side was only a read or write stream.

If using synchronous calls then both RPC server and client cannot be in the same process or error reporting won't work. But why would you even use an RPC system in that situation in the first place?

# ToDo

* Make sure the 'explicit' option works.
* Automatically close irrelevant ends of read/write streams

* More examples:
* Multiple-callbacks per call
* Binary stream
* Async call with multiple streams + client sending stream to server
* Calling remote callbacks from remote callbacks (turtles)

# Copyright and license

Copyright (c) 2014, 2015 Marc Juul <juul@sudomesh.org>

License: AGPLv3 (will probably change to MIT soon)

