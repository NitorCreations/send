# send

[![NPM Version][npm-image]][npm-url]
[![NPM Downloads][downloads-image]][downloads-url]
[![Linux Build][travis-image]][travis-url]
[![Windows Build][appveyor-image]][appveyor-url]
[![Test Coverage][coveralls-image]][coveralls-url]
[![Gratipay][gratipay-image]][gratipay-url]

Send is a library for streaming files from the file system as a http response
supporting partial responses (Ranges), conditional-GET negotiation, high test
coverage, and granular events which may be leveraged to take appropriate actions
in your application or framework.

Looking to serve up entire folders mapped to URLs? Try [serve-static](https://www.npmjs.org/package/serve-static).

## Installation

```bash
$ npm install send
```

## API

```js
var send = require('send')
```

### send(req, path, [options])

Create a new `SendStream` for the given path to send to a `res`. The `req` is
the Node.js HTTP request and the `path` is a urlencoded path to send (urlencoded,
not the actual file-system path).

#### Options

##### dotfiles

Set how "dotfiles" are treated when encountered. A dotfile is a file
or directory that begins with a dot ("."). Note this check is done on
the path itself without checking if the path actually exists on the
disk. If `root` is specified, only the dotfiles above the root are
checked (i.e. the root itself can be within a dotfile when when set
to "deny").

  - `'allow'` No special treatment for dotfiles.
  - `'deny'` Send a 403 for any request for a dotfile.
  - `'ignore'` Pretend like the dotfile does not exist and 404.

The default value is _similar_ to `'ignore'`, with the exception that
this default will not ignore the files within a directory that begins
with a dot, for backward-compatibility.

##### end

Byte offset at which the stream ends, defaults to the length of the file
minus 1. The end is inclusive in the stream, meaning `end: 3` will include
the 4th byte in the stream.

##### etag

Enable or disable etag generation, defaults to true.

##### extensions

If a given file doesn't exist, try appending one of the given extensions,
in the given order. By default, this is disabled (set to `false`). An
example value that will serve extension-less HTML files: `['html', 'htm']`.
This is skipped if the requested file already has an extension.

##### index

By default send supports "index.html" files, to disable this
set `false` or to supply a new index pass a string or an array
in preferred order.

##### lastModified

Enable or disable `Last-Modified` header, defaults to true. Uses the file
system's last modified value.

##### maxAge

Provide a max-age in milliseconds for http caching, defaults to 0.
This can also be a string accepted by the
[ms](https://www.npmjs.org/package/ms#readme) module.

##### precompressed

Precompressed files are extra static files that are compressed before
they are requested, as opposed to compressing on the fly. Compressing
files once offline (for example during site build) allows using
stronger compression methods and both reduces latency and lowers cpu
usage when serving files.

The `precompressed` option enables or disables serving of precompressed
content variants. The option defaults to `false`, if set to `true` checks
for existence of brotli and gzip compressed files with `.br` and `.gz`
extensions, preferring brotli if the file exists and supported by the
browser.

Example scenario:

The file `site.css` has both `site.css.gz` and `site.css.br`
precompressed versions available in the same directory.
When a request comes with an `Accept-Encoding` header with value `gzip, br`
requesting `site.css` the contents of `site.css.br` is sent instead and
a header `Content-Encoding` with value `br` is added to the response.
In addition a `Vary: Accept-Encoding` header is added to response allowing
proxies to work correctly.

Custom configuration:

It is also possible to customize the searched file extensions and header
values (used with Accept-Encoding and Content-Encoding headers) by specifying
them explicitly in an array in the preferred priority order. For example:
`[{encoding: 'bzip2', extension: '.bz2'}, {encoding: 'gzip', extension: '.gz'}]`.

Compression tips:
   * Precompress at least all static `js`, `css`  and `svg` files.
   * Precompress using both brotli (supported by Firefox and Chrome) and
     gzip encoders.
   * Use zopfli for gzip compression for and extra 5% benefit for all browsers.

##### root

Serve files relative to `path`.

##### start

Byte offset at which the stream starts, defaults to 0. The start is inclusive,
meaning `start: 2` will include the 3rd byte in the stream.

#### Events

The `SendStream` is an event emitter and will emit the following events:

  - `error` an error occurred `(err)`
  - `directory` a directory was requested
  - `file` a file was requested `(path, stat)`
  - `headers` the headers are about to be set on a file `(res, path, stat)`
  - `stream` file streaming has started `(stream)`
  - `end` streaming has completed

#### .pipe

The `pipe` method is used to pipe the response into the Node.js HTTP response
object, typically `send(req, path, options).pipe(res)`.

### .mime

The `mime` export is the global instance of of the
[`mime` npm module](https://www.npmjs.com/package/mime).

This is used to configure the MIME types that are associated with file extensions
as well as other options for how to resolve the MIME type of a file (like the
default type to use for an unknown file extension).

## Error-handling

By default when no `error` listeners are present an automatic response will be
made, otherwise you have full control over the response, aka you may show a 5xx
page etc.

## Caching

It does _not_ perform internal caching, you should use a reverse proxy cache
such as Varnish for this, or those fancy things called CDNs. If your
application is small enough that it would benefit from single-node memory
caching, it's small enough that it does not need caching at all ;).

## Debugging

To enable `debug()` instrumentation output export __DEBUG__:

```
$ DEBUG=send node app
```

## Running tests

```
$ npm install
$ npm test
```

## Examples

### Small example

```js
var http = require('http');
var send = require('send');

var app = http.createServer(function(req, res){
  send(req, req.url).pipe(res);
}).listen(3000);
```

### Custom file types

```js
var http = require('http');
var send = require('send');

// Default unknown types to text/plain
send.mime.default_type = 'text/plain';

// Add a custom type
send.mime.define({
  'application/x-my-type': ['x-mt', 'x-mtt']
});

var app = http.createServer(function(req, res){
  send(req, req.url).pipe(res);
}).listen(3000);
```

### Serving from a root directory with custom error-handling

```js
var http = require('http');
var send = require('send');
var url = require('url');

var app = http.createServer(function(req, res){
  // your custom error-handling logic:
  function error(err) {
    res.statusCode = err.status || 500;
    res.end(err.message);
  }

  // your custom headers
  function headers(res, path, stat) {
    // serve all files for download
    res.setHeader('Content-Disposition', 'attachment');
  }

  // your custom directory handling logic:
  function redirect() {
    res.statusCode = 301;
    res.setHeader('Location', req.url + '/');
    res.end('Redirecting to ' + req.url + '/');
  }

  // transfer arbitrary files from within
  // /www/example.com/public/*
  send(req, url.parse(req.url).pathname, {root: '/www/example.com/public'})
  .on('error', error)
  .on('directory', redirect)
  .on('headers', headers)
  .pipe(res);
}).listen(3000);
```

## License 

[MIT](LICENSE)

[npm-image]: https://img.shields.io/npm/v/send.svg
[npm-url]: https://npmjs.org/package/send
[travis-image]: https://img.shields.io/travis/pillarjs/send/master.svg?label=linux
[travis-url]: https://travis-ci.org/pillarjs/send
[appveyor-image]: https://img.shields.io/appveyor/ci/dougwilson/send/master.svg?label=windows
[appveyor-url]: https://ci.appveyor.com/project/dougwilson/send
[coveralls-image]: https://img.shields.io/coveralls/pillarjs/send/master.svg
[coveralls-url]: https://coveralls.io/r/pillarjs/send?branch=master
[downloads-image]: https://img.shields.io/npm/dm/send.svg
[downloads-url]: https://npmjs.org/package/send
[gratipay-image]: https://img.shields.io/gratipay/dougwilson.svg
[gratipay-url]: https://www.gratipay.com/dougwilson/
