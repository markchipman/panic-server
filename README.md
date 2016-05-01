# Panic-server
*Distributed Javascript runner*

[![Travis branch](https://img.shields.io/travis/PsychoLlama/panic-server/master.svg?style=flat-square)](https://travis-ci.org/PsychoLlama/panic-server)
[![npm](https://img.shields.io/npm/dt/panic-server.svg?style=flat-square)](https://www.npmjs.com/package/panic-server)
[![npm](https://img.shields.io/npm/v/panic-server.svg?style=flat-square)](https://www.npmjs.com/package/panic-server)
[![Gitter](https://img.shields.io/gitter/room/nwjs/nw.js.svg?style=flat-square)](https://gitter.im/amark/gun)

> **TL;DR:**<br />
It's a glorified `eval()` with platform queries.

Panic-server is designed as the underlying layer for panic-room, the distributed testing framework. It allows you to dynamically and reactively group connected clients and evaluate code on platform subsets.

For example:
```javascript
// import the runner
var panic = require('panic-server')

// Start the server.
// This is what you'll connect
// to from a client.
panic.server().listen(3000)

// The list of all connected clients,
// updating in real-time.
panic.clients

panic.clients.run(function () {
	console.log('This code runs on every client');
}).then(function () {
	// this runs when all clients have finished.
}).catch(function () {
	// this runs if a client throws an error.
})
```

The `.run` command sends a function to be evaluated on every connected client at that point in time. To add a client, you'll need to import the code and point it to the server.

**Browser**
```html
<!--
The port number and hostname are
configured using `panic.server()`
-->
<script src="http://localhost:3000"></script>
<!--
As soon as it finishes loading,
`panic` will be a global variable.
-->
<script>
	// this attempts to connect to your panic server
	panic.server('http://localhost:3000')
</script>
```

**Server**
```javascript
// imports the client code
var panic = require('panic-client')

// connects to your panic server
panic.server('http://localhost:3000')
```

Now that the clients are connected, you can run any code you want on them from the panic-server. Obviously you won't want to do this when, no, hang on, lemme make this bigger...

> **WARNING:** `eval()` is used. Including this library in user-facing code may open serious [XSS vulnerabilities](https://en.wikipedia.org/wiki/Cross-site_scripting). This library is targeted towards testing frameworks, and should not be used in production code unless you really really know what you're doing.

Think of it as a control center for your code. You can pick out a group of clients, and run arbitrary code on them. Now that all that is out of the way, let's get to the API...

## API
Panic-server consists of two parts:
 - the server
 - the clients

### `panic.server([http.Server])`
If an [`http.Server`](https://nodejs.org/api/http.html#http_class_http_server) is passed, panic will use it to configure [socket.io](http://socket.io/) and the `/panic.js` route will be added that servers up the [`panic-client`](https://github.com/gundb/panic-client) browser code.

If no server is passed, a new one will be created.

If you're not familiar with Node.js' http module, that's okay. The quickest way to get up and running is to call `.listen(8080)` which listens for requests on port 8080. In a browser, the url will look something like this: `http://localhost:8080/panic.js`.

**Create a new server**
```javascript
var panic = require('panic-server')

// create a new http server instance
var server = panic.server()

// listen for requests on port 8080
server.listen(8080)
```

**Reuse an existing one**
```javascript
var panic = require('panic-server')

// create a new http server
var server = require('http').createServer()

server.on('request', doThings)
server.on('request', doOtherThings)

// pass it to panic
panic.server(server)

// start listening on a port
server.listen(8080)
```

> If you want to listen on port 80 (the default for browsers), you may need to run node as `sudo`.

Once you have a server listening, point browsers/servers to your address. If you're on a browser, you can import the code by pointing it to a script tag:

```html
<!-- Imports the "panic-client" browser code

	The url depends on what
	port/hostname you're listening on.
-->
<script src="http://localhost:8080/panic.js"></script>
<script>
	// once imported, panic exposes the "panic" global.
	// `.server()` in this context will connect to the server.
	panic.server('http://localhost:8080')
</script>
```

> **Note:** if you're using [PhantomJS](https://github.com/ariya/phantomjs), you'll need to serve the html page over http/s for socket.io to work.

If you want to add a Node.js server, the only difference is how the code is imported. You'll need to install [`panic-client`](https://github.com/gundb/panic-client) for this.

To install, run this:
```bash
npm install panic-client
```

**Node.js**
```javascript
// imports the node code for panic client
var panic = require('panic-client')

// connects to the server
panic.server('http://localhost:8080')
```

### `panic.clients`
Every group is a ClientList instance, and inherits from EventEmitter. They update in real-time as clients are added and disconnected, and have array-like methods for manipulating and subgrouping. `panic.clients` is the root level list, and contains every client currently connected.

#### Events
As the list changes, it will emit one of two mutation events:
 - `add`: a new client is added to the list
 - `remove`: a client is removed from the list

Both events pass the client and it's id.

**Examples**
```javascript
// listen for new clients
panic.clients.on('add', function (client, id) {
	// a new client is added
})

// listen for removed clients
panic.clients.on('remove', function (client, id) {
	// a client has been removed
})
```

#### Methods

<a name='clients'></a>
> Every client inside a list is an object with two properties, `platform` and `socket`. The platform is sent as part of the client handshake, while the socket is a websocket interface provided by [`socket.io`](http://socket.io/).
```javascript
// each client has this structure
var client = {
	// the websocket is a socket.io interface
	socket: WebSocket,
	platform: { /* platformjs */ }
}
```

**Table of Contents**
 - [`.filter()`](#filter)
 - [`.excluding()`](#excluding)
 - [`.run()`](#run)
 - [`.len()`](#len)
 - [`.get()`](#get)
 - [`.add()`](#add)
 - [`.remove()`](#remove)
 - [`.each()`](#each)

##### <a name='filter'></a> `.filter(query)`
Returns a filtered list containing everything that matches a platform query.

> Platform data is generated by [platform.js](https://github.com/bestiejs/platform.js/).

When passed a `String` or `RegExp`, it'll be used to match against the `platform.name`. For example, `clients.filter('Firefox')` will return a dynamic list of all firefox clients, as will `clients.filter(/Firefox/)`. A more complex query can be formed by passing an object containing more platform descriptors.

```javascript
var list = clients.filter({
	layout: /(Gecko|Blink)/,
	os: {
		architecture: 64,
		family: 'OS X'
	}
})
```

Every setting above is optional, and you can create as loose or specific a query as you need. If you need a more complex query than that, you can also pass a filtering callback, which functions much like [`Array.prototype.filter`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter).

> [client definition](#clients)

```javascript
var firefox = clients.filter(function (client, id, list) {
	// `id`: The unique client id
	// `list`: The parent list object, in this case `clients`

	var platform = client.platform;

	/*
		This query only adds versions of
		Firefox later than version 36.
	*/
	if (platform.name === 'Firefox' && platform.version > 36) {
		// add this client to the new list
		return true;
	} else {
		// leave the client out of the new list
		return false;
	}
});
```

To make things cooler, you can chain filters off one another. For example, the above query only allows versions of firefox after 36. You could write that as two separate queries...

```javascript
// the list of all firefox clients
var firefox = clients.filter('Firefox')

// the list of firefox newer after version 36
var firefoxAfter36 = firefox.filter(function (client) {
	return client.platform.version > 36
});
```

As new clients are added, they'll be run through the firefox filters, and if added, will be run through the version filter. The dynamic filtering process allows for some cool RxJS style code.

##### <a name='excluding'></a> `.excluding(ClientList)`
You can also create lists that exclude other lists, like a list of browsers might be anything that isn't a server, or perhaps you want to exclude all Chrome browsers from a list. You can do that with `.excluding`.

```javascript
// create a dynamic list of all node.js clients
var servers = clients.filter('Node.js')

// the list of all clients,
// except anything that belongs to `servers`.
var browsers = clients.excluding(servers)
```

Like filter, you can chain queries off each other to create really powerful queries.

```javascript
// using `browsers` from above
var chrome = browsers.filter('Chrome')
var notChrome = browsers.excluding(chrome)
```

##### <a name='run'></a> `.run(Function[, Object])`
`.run` is where the magic happens. This method allows you to evaluate a function on all platforms belonging to this list, and reject or resolve a promise when either everyone finishes or one fails. Asynchronous code is supported.

`.run` takes one argument: the function to evaluate. It can be weird to think about, and may trip you up a couple times. **The function will not be run on panic-server. It is run on the client**, it does not have your local scope, and may not have your platform tools (like CommonJS, window variables, npm modules, ES2015 compatibility, etc). You are potentially evaluating on an entirely different machine. Code responsibly :wink:

That said, here's an example:

```javascript
clients.run(function () {
	// this code is evaluated on every platform
})

clients.filter('Node.js').run(function () {
	var http = require('http')
	// evaluating live on every server
});
```

The function passed is first stringified, then sent to the clients for evaluation. When it's invoked on the client, it's given a special `this` context and some control parameters to work with async data.

The function is passed two parameters: the `this` context, and the `done` callback. If the callback is set as a parameter, it's assumed that the code is asynchronous and won't report a `done` event until the callback is invoked or an error is thrown.

After sending off your function, `.run` returns a promise. When every client has finished running the code, the promise fulfills. If you're not familiar with promises, I recommend reading the [MDN page](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise). They're an invaluable tool and native support is well on it's way. They're also incredibly useful when paired with `.run`.

For example, you could have a list of 100 browsers, and perhaps you want each of them to load a script before running any other code. First, you'd send the code to load the script, call `done` when you're finished. Once all the browsers report `done`, the promise resolves and you can run more code. Here's what that looks like:

```javascript
var servers = clients.filter('Node.js')
var browsers = clients.excluding(servers)

function loadExpectJS(browser, done) {
	// create a script element
	var script = document.createElement('script')

	// set the source to expect.js
	script.src = 'https://cdn.rawgit.com/Automattic/expect.js/master/index.js'

	script.onload = done;
	script.onerror = this.fail;
}

browsers.run(loadExpectJS)
.then(function () {
	// all browsers successfully loaded expect.js
})
.catch(function (error) {
	// one or more browsers failed to load expect.js
})
```

> **Fun fact:** by returning a new promise in a `.then` callback, everything chained after refers to that new promise. This means you can create really cool chains, like "load expect.js, then run a function, then refresh all browsers", all executed synchronously. Here's an example:

```javascript
// using the function defined above,
// load expect.js
browsers.run(loadExpectJS)
.then(function () {
	// once they've all loaded the file...
	return browsers.run(function () {
		// run this assertion code
		expect(true).to.eq(true)
	})
})
.then(function () {
	// once everyone's ran the assertion code,
	return browsers.run(function () {
		// refresh every browser.
		location.reload()
	})
})
```

###### async/await
If you're using Babel.js, promises become much more succinct using the ES7/2016 async/await controls, or if you're using the [`asyncawait`](https://www.npmjs.com/package/asyncawait) module on npm.

```javascript
// define an async function
async function runCodeStuff () {
	// pause for all browsers to finish 1st chunk
	await browsers.run(function () {
		// 1st code chunk
	})

	// pause for 2nd chunk
	await browsers.run(function () {
		// 1st chunk finished,
		// 2nd code chunk running
	})

	console.log('Both chunks finished!')
}
```

However, that syntax is completely optional.

##### `.run` scope controls
This section is gonna be a little tricky, hold on...

`.run` allows you to send local variables to the clients and continue using them as locals. If you're familiar with Javascript's [`with`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/with) statement, it's basically a fancy `with`.

By passing an object as the second parameter, you can export variables into the callback as local scope variables.

```javascript
clients.run(function () {
	console.log(localVariable) // 'visible'
	console.log(numbers) // array
}, {
	localVariable: 'visible',
	numbers: [1, 2, 3, 5, 8]
})
```

Alternatively, all variables are accessable in the `this.data` object:

```javascript
clients.run(function (client) {
	// using `this.data`
	console.log(this.data.localVariable) // 'visible'

	// the `client` param is the same as `this`
	console.log(client.data.numbers) // array
}, {
	localVariable: 'visible',
	numbers: [1, 2, 3, 5, 8]
})
```

In this way, you can share variables on the server with the clients without duplicating code. This is also useful for injecting variables into a mixin, like `loadScript`:

```javascript
function loadScript(client, done) {
	var script = document.createElement('script');

	// use the exported `src` variable
	script.src = src;

	script.onload = done;
	document.body.appendChild(script);
}

// expose the `src` variable
clients.run(loadScript, {
	src: 'http://...'
})
```

Some Javascript linters may complain about using undefined variables, in which case you can either turn off the linter rule, or use `this.data` and disable the local variable injection by setting a flag in the scope object, called 'export vars'.

```javascript
clients.run(function () {
	typeof src; // 'undefined'
	this.data.src; // 'http://...'
}, {
	'export vars': false,
	src: 'http://...'
})
```

##### <a name='len'></a> `.len()`
Returns the number of clients in a list.


**Low-level API**
-----------------

##### <a name='get'></a> `.get(id)`
Returns the client corresponding to the id. Presently, socket.io's `socket.id` is used to uniquely key clients.

##### <a name='add'></a> `.add(client)`
Manually adds a client to the list. This is low-level enough that you should never need it, though soon it may accept ClientList instances.

##### <a name='remove'></a> `.remove(client)`
Removes a client from the list, emitting a `remove` event with the client object. This API is low-level enough that you shouldn't need to use it.

##### <a name='each'></a> `.each(Function)`
Iterate over a collection of clients. This method accepts a callback to be invoked for each item in the collection, and is passed three arguments:

 - client
 - id
 - list

The client is defined [here](#clients), the id is the unique name that identifies the client, and the list is the ClientList that `.each` was called on.

**Example**
```javascript
clients.each(function (client, id, list) {
	client; // { platform: Object, socket: Object }
	typeof id; // 'string'
	list === clients; // true
})
```


## Support
If you have questions or ideas, we'd love to hear them! Just swing by our [gitter channel](https://gitter.im/amark/gun) and ask for @PsychoLlama or @amark. We're usually around :wink:

Built with :heart: by the team at [gunDB](https://github.com/amark/gun).
