---
title: Lambda Functions
type: docs
---

The Caddy `awslambda` plugin uses JSON request and reply envelopes to
encode HTTP specific request and reply information. This allows AWS Lambda
functions to access HTTP request headers, specify HTTP response status codes and headers,
and return reply bodies in formats other than JSON.

All examples in this document use the `node-4.3` AWS Lambda runtime.

### Examples

Consider this `Caddyfile`:

```
awslambda /caddy/ {
   aws_access  redacted
   aws_secret  redacted
   aws_region  us-west-2
   include     caddy-*
}
```

And this Lambda function, named `caddy-echo`:

```javascript
'use strict';
exports.handler = (event, context, callback) => {
    callback(null, event);
};
```

When we request it via `curl` we receive the following response, which reflects the
request envelope Caddy sent to the lambda function:


```
$ curl -s -X POST -d 'hello' http://localhost:2015/caddy/caddy-echo | jq .
{
  "type": "HTTPJSON-REQ",
  "meta": {
    "method": "POST",
    "path": "/caddy/caddy-echo",
    "query": "",
    "host": "localhost:2020",
    "proto": "HTTP/1.1",
    "headers": {
      "accept": [
        "*/*"
      ],
      "content-length": [
        "5"
      ],
      "content-type": [
        "application/x-www-form-urlencoded"
      ],
      "user-agent": [
        "curl/7.43.0"
      ]
    }
  },
  "body": "hello"
}
```

The request envelope format is described in detail below, but there are three top level fields:

* `type` - always set to `HTTPJSON-REQ`
* `meta` - JSON object containing HTTP request metadata such as the request method and headers
* `body` - HTTP request body (if provided)

Since our Lambda function didn't respond using the reply envelope, the raw reply was sent
to the HTTP client and the `Content-Type` header was set to `application/json` automatically.

Let's write a 2nd Lambda function that uses the request metadata and sends a reply using the
envelope format.

Lambda function name: `caddy-echo-html`

```javascript
'use strict';
exports.handler = (event, context, callback) => {
    var html, reply;
    html = '<html><head><title>Caddy Echo</title></head>' +
           '<body><h1>Request:</h1>' +
           '<pre>' + JSON.stringify(event, null, 2) +
           '</pre></body></html>';
    reply = {
        'type': 'HTTPJSON-REP',
        'meta': {
            'status': 200,
            'headers': {
                'Content-Type': [ 'text/html' ]
            }
        },
        body: html
    };
    callback(null, reply);
};
```

If we request `http://localhost:2015/caddy/caddy-echo-html` in a desktop web browser, the HTML
formatted reply is displayed with a pretty-printed version of the request inside `<pre>` tags.

In a final example we'll send a redirect using a 302 HTTP response status.

Lambda function name: `caddy-redirect`

```javascript
'use strict';
exports.handler = (event, context, callback) => {
    var redirectUrl, reply;
    redirectUrl = 'https://caddyserver.com/'
    reply = {
        'type': 'HTTPJSON-REP',
        'meta': {
            'status': 302,
            'headers': {
                'Location': [ redirectUrl ]
            }
        },
        body: 'Page has moved to: ' + redirectUrl
    };
    callback(null, reply);
};
```

If we request `http://localhost:2015/caddy/caddy-redirect` we are redirected to the Caddy home page.


### Request envelope

The request payload sent from Caddy to the AWS Lambda function is a JSON object with the following fields:

* `type` - always the string literal `HTTPJSON-REQ`
* `body` - the request body, or an empty string if no body is provided.
* `meta` - a JSON object with the following fields:
  * `method` - HTTP request method (e.g. `GET` or `POST`)
  * `path` - URI path without query string
  * `query` - Raw query string (without '?')
  * `host` - Host client request was made to. May be of the form host:port
  * `proto` - Protocol used by the client
  * `headers` - a JSON object of HTTP headers sent by the client. Keys will be lower case. Values will be string arrays.

### Reply envelope

AWS Lambda functions should return a JSON object with the following fields:

* `type` - always the string literal `HTTPJSON-REP`
* `body` - response body
* `meta` - optional response metadata. If provided, must be a JSON object with these fields:
  * `status` - HTTP status code (e.g. 200)
  * `headers` - a JSON object of HTTP headers. Values **must** be string arrays.

If `meta` is not provided, a 200 status will be returned along with a `Content-Type: application/json` header.

### Gotchas

* Request and reply header values must be **string arrays**. For example:

```javascript
// Valid
var reply = {
    'type': 'HTTPJSON-REP',
    'meta': {
        'headers': {
            'content-type': [ 'text/html' ]
        }
    }
};

// Invalid
var reply = {
    'type': 'HTTPJSON-REP',
    'meta': {
        'headers': {
            'content-type': 'text/html'
        }
    }
};
```

* Reply must have a top level `'type': 'HTTPJSON-REP'` field. The rationale is that since
all Lambda responses must be JSON we need a way to detect the presence of the envelope. Without
this field, the raw reply JSON will be sent back to the client unmodified.
