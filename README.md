# SvelteKit Node Adapter Returns an "Invalid request body" HTTP Error that Can Be Misleading and Hard to Debug

A minimal reproducible example.

It simply constists in the skeleton app one gets with `npm create svelte@latest` (with the "latest" at the time of writing, see `package-lock.json`), using [the Node adapter](https://kit.svelte.dev/docs/adapter-node). To reproduce, simply run the following in your shell:

```
npm install
ORIGIN=localhost:3000 node build # note the invalid value for ORIGIN
```

Then go to http://localhost:3000/: you should get an "Invalid request body" response with a `HTTP 400 Bad Request` status.

The error message is misleading because your request obviously does not have an invalid body (it does not even have a body since it is a `GET` request). Furthermore it is quite hard to debug because the error is not caught by the `handleError` hook in `src/hooks.server.ts`.

The cause of the error is that the `ORIGIN` environment variable is invalid, it should be `http://localhost:3000` (or just `localhost:3000`; [see also the MDN page about "origin"](https://developer.mozilla.org/en-US/docs/Glossary/Origin)). This causes an error in `build/handler.js` that happens too early to be caught by `handleError`. The code that handles this error (which does seem like it could be instrumented at the moment) mistakenly assumes that an error here can only be caused by an invalid body, hence the error message: https://github.com/sveltejs/kit/blob/d4043863e5aa6b5fb003ebafef2a6e9098255b45/packages/adapter-node/src/handler.js#L90

```
	} catch (err) {
		res.statusCode = err.status || 400;
		res.end('Invalid request body');
		return;
	}
```

If we add a `console.error(err);` in this `catch` block we get the following error log:

```
TypeError: Failed to parse URL from localhost/
    at new Request (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/shims.js:12760:16)
    at getRequest (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1031:9)
    ... 4 lines matching cause stack trace ...
    at handle (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1257:23)
    at file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1257:40
    at Array.<anonymous> (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:653:28)
    at handle (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1257:23) {
  [cause]: TypeError [ERR_INVALID_URL]: Invalid URL
      at new NodeError (node:internal/errors:387:5)
      at URL.onParseError (node:internal/url:565:9)
      at new URL (node:internal/url:641:5)
      at new Request (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/shims.js:12758:22)
      at getRequest (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1031:9)
      at Array.ssr (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1186:19)
      at handle (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1257:23)
      at file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1257:40
      at Array.<anonymous> (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1175:4)
      at handle (file:///home/cedricvr/tinkering/sveltekit-invalid-body/build/handler.js:1257:23) {
    input: 'localhost/',
    code: 'ERR_INVALID_URL'
  }
}
```

We then see that the error comes from NodeJS builting `new URL(...)` constructor, due to the invalid origin. We can easily reproduce it on NodeJS itself:

```
$ docker run -it node:18 node
Welcome to Node.js v18.16.0.
Type ".help" for more information.
> new URL('localhost/');
Uncaught TypeError [ERR_INVALID_URL]: Invalid URL
    at __node_internal_captureLargerStackTrace (node:internal/errors:490:5)
    at new NodeError (node:internal/errors:399:5)
    at new URL (node:internal/url:560:13) {
  input: 'localhost/',
  code: 'ERR_INVALID_URL'
}
```
