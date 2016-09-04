---
title: awslambda
type: docs
directive: true
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
    <span class="hl-subdirective">aws_access</span>    <i>aws access key value</i>
    <span class="hl-subdirective">aws_secret</span>    <i>aws secret key value</i>
    <span class="hl-subdirective">aws_region</span>    <i>aws region name</i>
    <span class="hl-subdirective">qualifier</span>     <i>qualifier value</i>
    <span class="hl-subdirective">include</span>       <i>included function names...</i>
    <span class="hl-subdirective">exclude</span>       <i>excluded function names...</i>
}</code>

*   **aws_access** is the AWS Access Key to use when invoking Lambda functions. If omitted, the AWS_ACCESS_KEY_ID env var is used.
*   **aws_secret** is the AWS Secret Key to use when invoking Lambda functions. If omitted, the AWS_SECRET_ACCESS_KEY env var is used.
*   **aws_region** is the AWS Region name to use (e.g. 'us-west-1'). If omitted, the AWS_REGION env var is used.
*   **qualifier** is the qualifier value to use when invoking Lambda functions. Typically this is set to a function version or alias name. If omitted, no qualifier will be passed on the AWS Invoke invocation.
*   **include** is an optional space separated list of function names to include. Prefix and suffix globs ('*') are supported. If omitted, any function name not excluded may be invoked.
*   **exclude** is an optional space separated list of function names to exclude. Prefix and suffix globs are supported.

Function names are parsed from the portion of request path following the path-prefix in the
directive. For example, given a directive `awslambda /lambda/`, a request to `/lambda/hello-world`
would invoke the AWS Lambda function named `hello-world`.

The `include` and `exclude` globs are simple wildcards, not regular expressions.
For example, `include foo*` would match `food` and `footer` but not `bafoon`, while
`include *foo*` would match all three.

If you adopt a simple naming convention for your Lambda functions, these rules can be used to
group access to a set of Lambdas under a single URL path prefix.

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
    
