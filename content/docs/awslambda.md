---
title: awslambda
type: docs
directive: true
plugin: true
link: https://github.com/coopernurse/caddy-awslambda
---

awslambda proxies requests to AWS Lambda functions using the
[AWS Lambda Invoke](http://docs.aws.amazon.com/lambda/latest/dg/API_Invoke.html) operation.
It provides an alternative to AWS API Gateway and provides a simple way to declaratively proxy
requests to a set of Lambda functions without per-function configuration.

Given that AWS Lambda has no notion of request and response headers, this plugin defines a standard
JSON envelope format that encodes HTTP requests in a standard way, and expects the JSON returned from
the Lambda functions to conform to the response JSON envelope format.

### Syntax

<code class="block"><span class="hl-directive">awslambda</span> <span class="hl-arg"><i>path-prefix</i></span> {
    <span class="hl-subdirective">aws_access</span>         <i>aws access key value</i>
    <span class="hl-subdirective">aws_secret</span>         <i>aws secret key value</i>
    <span class="hl-subdirective">aws_region</span>         <i>aws region name</i>
    <span class="hl-subdirective">qualifier</span>          <i>qualifier value</i>
    <span class="hl-subdirective">include</span>            <i>included function names...</i>
    <span class="hl-subdirective">exclude</span>            <i>excluded function names...</i>
    <span class="hl-subdirective">name_prepend</span>       <i>string to prepend to function name</i>
    <span class="hl-subdirective">name_append</span>        <i>string to append to function name</i>
    <span class="hl-subdirective">single</span>             <i>name of a single lambda function to invoke</i>
    <span class="hl-subdirective">strip_path_prefix</span>  <i>If true, path and function name are stripped from the path</i>
}</code>

*   **aws_access** is the AWS Access Key to use when invoking Lambda functions. If omitted, the AWS_ACCESS_KEY_ID env var is used.
*   **aws_secret** is the AWS Secret Key to use when invoking Lambda functions. If omitted, the AWS_SECRET_ACCESS_KEY env var is used.
*   **aws_region** is the AWS Region name to use (e.g. 'us-west-1'). If omitted, the AWS_REGION env var is used.
*   **qualifier** is the qualifier value to use when invoking Lambda functions. Typically this is set to a function version or alias name. If omitted, no qualifier will be passed on the AWS Invoke invocation.
*   **include** is an optional space separated list of function names to include. Prefix and suffix globs ('*') are supported. If omitted, any function name not excluded may be invoked.
*   **exclude** is an optional space separated list of function names to exclude. Prefix and suffix globs are supported.
*   **name_prepend** is an optional string to prepend to the function name parsed from the URL before invoking the Lambda.
*   **name_append** is an optional string to append to the function name parsed from the URL before invoking the Lambda.
*   **single** is an optional function name. If set, function name is not parsed from the URI path.
*   **strip_path_prefix** If 'true', path and function name is stripped from the path sent as request metadata to the Lambda function. (default=false)

Function names are parsed from the portion of request path following the path-prefix in the
directive based on this convention: `[path-prefix]/[function-name]/[extra-path-info]` unless `single` attribute is set.

For example, given a directive `awslambda /lambda/`, requests to `/lambda/hello-world` and `/lambda/hello-world/abc`
would each invoke the AWS Lambda function named `hello-world`.

The `include` and `exclude` globs are simple wildcards, not regular expressions.
For example, `include foo*` would match `food` and `footer` but not `buffoon`, while
`include *foo*` would match all three.

`include` and `exclude` rules are run before `name_prepend` and `name_append` are applied and
are run against the parsed function name, not the entire URL path.

If you adopt a simple naming convention for your Lambda functions, these rules can be used to
group access to a set of Lambdas under a single URL path prefix.

`name_prepend` and `name_append` allow for shorter names in URLs and works well with tools such
as Apex, which prepend the project name to all Lambda functions. For example, given an URL path
of `/api/foo` with a `name_prepend` of `acme-api-`, the plugin will try to invoke the function
named `acme-api-foo`.

### Writing Lambdas

See [Lambda Functions](/docs/awslambda-functions) for details on the JSON request and reply
envelope formats. Lambda functions that comply with this format may set arbitrary HTTP response
status codes and headers.

### Examples

Proxy all requests starting with /lambda/ to AWS Lambda, using env vars for AWS access keys and region:

<code class="block"><span class="hl-directive">awslambda</span> <span class="hl-arg">/lambda/</span></code>

Proxy requests starting with `/api/` to AWS Lambda using the `us-west-2` region, for functions staring with `api-` but not ending with `-internal`. A qualifier is used to target the `prod` aliases for each function.

<code class="block"><span class="hl-directive">awslambda</span> <span class="hl-arg">/api/</span> {
    <span class="hl-subdirective">aws_region</span>  us-west-2
    <span class="hl-subdirective">qualifier</span>   prod
    <span class="hl-subdirective">include</span>     api-*
    <span class="hl-subdirective">exclude</span>     *-internal
}</code>
    
