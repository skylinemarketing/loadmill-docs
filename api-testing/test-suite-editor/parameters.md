# Parameters

Working with [Loadmill](https://www.loadmill.com) makes it very easy to convert recorded browser/network sessions \(via HAR files\) into test scenarios. But once a scenario has been created, it often needs to be **parametrized** in order to denote the dynamic parts of your API.

This is best explained by example, so let's have a look at the following test scenario:

1. User adds a blog post: `curl https://www.myblog.com/posts --data "Hello World!"`
2. User likes his own post: `curl -X PUT https://www.myblog.com/posts/123/like`

The `123` part is the identifier of the blog post created by the user - it cannot be known in advance because it is generated by the server at runtime. This is a very common use case for **parameters** and is handeled by two simple steps:

1. [Extract](parameters.md#parameter-extraction) the ID from the first response into a parameter, e.g. `postId`
2. Embed the parameter in the second request URL: `https://www.myblog.com/posts/${postId}/like`

You may embed parameters with the `${}` syntax in the request URL as well as the request body, request headers, [extractions](parameters.md#parameter-extraction), [assertions](https://github.com/loadmill/loadmill-docs/tree/75b2138469fd07320dae2a78a4f6a2518591d128/assertions.html) and more.

Note that any parametrized expression such as `posts/${postId}` will remain as-is if no such parameter is defined or extracted before the expression is evaluated.

## Default Parameters

Another common use case for parametrization is when you want to **reuse test scenarios** on different environments or with small adjustments.

Let's extend our first example by requiring that the user provide credentials via **basic authentication**. This means our new URLs could look like this:

1. `https://testUser:testPassword@www.myblog.com/posts`
2. `https://testUser:testPassword@www.myblog.com/posts/${postId}/like`

If you would like to use different credentials for every test run, you may replace the username and password with parameters and set their values in the **parameter defaults** section in your test scenario whenever you reuse the test. So now the URLs will look like:

1. `https://${user}:${pass}@www.myblog.com/posts`
2. `https://${user}:${pass}@www.myblog.com/posts/${postId}/like`

Using parameter defaults is especially useful for automated testing and CI where you may be testing a different server every time you run a test. Please refer to the [Loadmill CLI and npm module](https://www.npmjs.com/package/loadmill#parameters) for more information about how to inject parameters dynamically in such scenarios.

## Parameter Extraction

Parameters can be defined and populated with values dynamically after each request in your test scenario. There are several **extraction query types** that may be used:

1. **JSONPath** - used for extracting values from a JSON response. For example, the query `post.id` will extract the value `123` from this JSON response:

   ```javascript
    {
        "post": {
            "id": 123,
            "text": "Hello World!"
        }
    }
   ```

2. **JQuery \(Cheerio\)** - used for extracting values from XML/HTML responses. We use a subset of the JQuery selector syntax called [Cheerio](https://cheerio.js.org). You may add an optional \(but very useful\) **attribute** input to your query that selects an attribute value from the first element found by the jQuery. If you do not provide an attribute to select, the query will simply output the inner content of said element.
3. **JS RegExp** - used for extracting arbitrary values from any kind of textual response via regular expressions with capture groups. For example, we can extract the `id` field from the same JSON response we've seen above using a regular expression: `.*"id":\s*([0-9]*)`.
4. **Header** - used for extracting response header values via header names.
5. **Assignment** - used for assigning an explicit value to a parameter. Previously defined or built-in parameters may be embedded within the string, e.g. `https://${host}/path/to/glory` or `The time is ${__now}`.
6. **Clojure** - used for extracting values from Clojure \(EDN content type\) responses. Querying the data is done using JSONPath. For example, the query `$[":user"][":id"]` will extract the value `56` from this EDN response: 

   ```
    {:user {:role :viewer, :name "Rivi", :teams nil, :id "56"}}
 
   ```

Previously defined or built-in parameters may be embedded within **any kind of extraction query**. These parameters will be evaluated right before the query itself is evaluated.

## Built-in Parameters

There are several **built-in** parameters that you can use in your test scenario. They are:

* `__status` The status code of the last HTTP response.
* `__statusText` The status text of the last HTTP response.
* `__responseTime` The total response time \(in milliseconds\) of the last HTTP response.
* `__testRunId` - The test run id: \(Suite / Flow / Load\)
* `__testStartTime` - The test run start time \(UTC in milliseconds\)
* `__launchedBy` - The name of the user running the test.

**Note:** some previous built-in parameters are now defined as no-argument [parameter functions](parameters.md#functions) and can still be used in the same way.

## Advanced Usage

So far, we've only seen how to inject a simple parameter value into an arbitrary expression, e.g. `Hello ${name}`. However, it is also possible to inject a _**computed value**_ using [Parameter Operators](parameters.md#operators) or [Parameter Functions](parameters.md#functions), e.g. `The total price is ${price + __mult(price,tax)}`. You may use operators or functions anywhere parameters may be used.

Computed values can be extreamly useful when you need to introduce conditional behavior to your test. Say you want to skip **Purchase Request** if the preceding **Get Price Request** response returns a price above the current budget. This could be acomplished by extracting the price to a parameter and setting the **Purchase Request** skip condition to `${price <= budget}`.

You may also use _**literal values**_ within expressions, e.g. `${__if_then_else(is_good,'Success!',':_(')}` - but be aware there are some [syntactic limitations](parameters.md#notes-and-limitations).

Function calls without arguments can be used with or without parentheses, e.g. `__random_uuid` can be used instead of `__random_uuid()`.

See below the full list of supported [operators](parameters.md#operators) and [functions](parameters.md#functions).

### Notes And Limitations

Current syntax has some limitations. Syntax errors are easy to spot in the GUI - a malformed expression will simply not be highlighted.

If an expression is invalid due to its syntax it will simply remain as-is and will not be computed nor replaced at run-time. If, however, an operator or a function receives invalid input \(e.g. division by zero or any of the parameter arguments not having a value\) the test will fail with an error.

Note that predefined parameter values \(AKA [Parameter Defaults](parameters.md#default-parameters)\) are computed whenever a test configuration is validated - thus computation related errors will render the test configuration invalid.

Current syntax limitations are:

* Operators _**must**_ be separated from their arguments by spaces, e.g. `${x + y}` is fine but `${x+y}` will not be computed.
* Spaces are _**not allowed**_ anywhere else within an expression, e.g. both `${__add(x, y)}` and `${fullName == 'John Doe'}` will not be computed.
* **Literal values** may not contain whitespace characters, commas \(`,`\) or single quotes \(`'`\). These cannot be escaped - simply define a previous parameter with the desired value when the need arises. Note that numeric literal values need to be quoted the same as any other value, e.g. `${x > '0'}` is OK but `${x > 0}` will not be computed.
* You may chain multiple operations together, e.g. `${x * y + z}` but you may _**not**_ use parentheses to set precedence, e.g. `${(x * y) + z}` will not be computed. This can usually be worked around using functions though, e.g. `${__mult(x,y) + z}`
* All operators have the _**same precedence**_ - computations always conform to right-associativity, i.e. `${x * y + z - j + k}` will be computed as `x * (y + (z - (j + k)))`.
* Computations may _**not be nested**_, i.e. you may not pass a computed value as an argument to function, e.g. `${__mult(x,y) + z}` is OK but neither `${__mult(x + y,z)}` nor `${__mult(__add(x,y),z)}` will be computed.
* Unary operators, e.g. `${-x}` are _**not**_ supported. This can be overcome using functions such as [\_\_neg](parameters.md#__neg-p-1) or [\_\_not](parameters.md#__neg).

### Operators

The currently supported operators are:

#### Textual Operators

* `=` Strict equals operator. Aliases: `==` and `===`.
* `!=` Strict not-equals operator. Alias: `!==`.

May be applied to any two parameters which have values.

#### Boolean Operators

* `|` Logical OR operator. Alias: `||`.
* `&` Logical AND operator. Alias: `&&`.

May be applied to any two parameters which have values.

#### **True Semantics**

A parameter translates to boolean `true` if and only if

* It has a value _**and**_
* The value is not an empty string _**and**_
* The value is not equal to `false`, `FALSE`, `FaLsE` or any other combination of upper-case and lower-case letters that forms the word `false`.

The computed value of a valid boolean operation is either exactly `true` or exactly `false`.

#### Numeric Operators

* `+` Addition operator.
* `-` Subtraction operator.
* `*` Multiplication operator.
* `/` Division operator.
* `<` Less-then operator.
* `<=` Less-then-or-equals operator.
* `>` Greater-then operator.
* `>=` Greater-then-or-equals operator.

May be applied to any two parameters which have values that translate to _**finite numbers**_. Computed values are _**not**_ rounded to integers.

### Functions

The supported built-in functions are:

#### Numeric Functions

#### `__abs(p1)`

Returns the absolute value of `p1`.

#### `__add(p1,[p2,[...]])`

Same as the `+` operator, applied to any number of arguments.

#### `__sub(p1,p2)`

Same as the `-` operator.

#### `__neg(p1)`

Unary minus, equivalent to `__sub('0',p1)`.

#### `__mult(p1,[p2,[...]])`

Same as the `*` operator, applied to any number of arguments.

#### `__div(p1,p2)`

Same as the `/` operator.

#### Conditional Functions

#### `__true()`

Always returns `true`.

#### `__false()`

Always returns `false`.

#### `__and(p1,[p2,[...]])`

Logical AND \(same as the `&` operator\), applied to any number of arguments.

#### `__or(p1,[p2,[...]])`

Logical OR \(same as the `|` operator\), applied to any number of arguments.

#### `__not(p1)` <a id="__neg"></a>

Logical NOT. See also [True Semantics](parameters.md#true-semantics).

#### `__eq(p1,p2)`

Same as the `=` operator.

#### `__neq(p1,p2)`

Same as the `!=` operator.

#### `__eqi(p1,p2)`

Same as `__eq` but case-insensitive.

#### `__neqi(p1,p2)`

Same as `__neq` but case-insensitive.

#### `__lt(p1,p2)`

Same as the `<` operator.

#### `__lte(p1,p2)`

Same as the `<=` operator.

#### `__gt(p1,p2)`

Same as the `>` operator.

#### `__gte(p1,p2)`

Same as the `>=` operator.

#### `__matches(target,regex)`

Returns `true` if and only if the `target` matches the `regex`.

#### `__contains(target,search)`

Returns `true` if and only if the `target` contains the string `search`.

#### `__containsi(target,search)`

Same as `__contains` but case-insensitive.

#### `__if_then_else(condition,then,else)`

Returns `then` if `condition` is [semantically true](parameters.md#true-semantics), otherwise returns `else`.

#### `__switch(target,case1,value1,[case2,value2,[...],[default]])`

Returns `value1` if `target` equals `case1`, otherwise returns `value2` if `target` equals `case2` and so on.

If no match is made, the returned value will be an empty string - a default value may be given as the last argument.

#### `__switchi(target,case1,value1,[case2,value2,[...],[default]])`

Same as `__switch` but case-insensitive.

#### `__pick(selection,p1,[p2,[...]])`

Returns one of `p1` or `p2` or `p3`, etc. according to the `selection` - either a zero-based index \(e.g. `2` picks `p3`\) or the word `random` in order to pick a random value.

#### `__pick_random(p1,[p2,[...]])`

Same as `__pick('random',p1,[p2,[...]])`.

#### `__split_pick(target,delim,[selection=0])`

Splits the value of `target` into multiple strings separated by `delim` and returns one of them according to the `selection` as defined by the `__pick` function. If a `selection` argument is not provided, the first value will be returned.

#### Textual Functions

#### `__usd()`

Returns the `$` character.

#### `__length(target)`

Counts the number of characters in `target`.

#### `__array_length(target)`

Counts the number of elements in `target` array.

#### `__escape_regexp(target)`

Returns the value of `target` after escaping special RegExp characters.

#### `__encode_url(target)`

Returns the value of `target` after [URL encoding](https://en.wikipedia.org/wiki/Percent-encoding) special characters.

#### `__decode_url(target)`

Returns the value of `target` after [URL decoding](https://en.wikipedia.org/wiki/Percent-encoding).

#### `__escape_quotes(target)`

Returns the value of `target` after escaping special characters. This functions like escape quotes in JavaScript and will escape characters like \n \r \t and \".

#### `__lower(target)`

Returns the value of `target` after converting all characters to lower case.

#### `__upper(target)`

Returns the value of `target` after converting all characters to upper case.

#### `__slice(target,[begin=0,[end]])`

Returns a sub-string of `target` which starts at `begin` index \(inclusive\) and ends at `end` index \(exclusive\). Both indexes are zero-based.

#### Extraction Functions

See also [Parameter Extractions](parameters.md#parameter-extraction).

#### `__regexp(target,regexp,[default])`

Extracts a value from `target` using `regexp` as a JS RegExp. If there is no match, an empty string will be returned or, if present, the given `default` value.

#### `__jsonpath(target,jsonpath,[default])`

Extracts a value from `target` using `jsonpath` as a JSONPath query. If there is no match, an empty string will be returned or, if present, the given `default` value.

#### `__jquery(target,jquery,[selection=0,[attribute,[default]]])`

Extracts a value from `target` using `jquery` as a jQuery selector.

If multiple elements are matched, `selection` is applied as in `__pick` with the first element being selected by default.

If a non-empty `attribute` is given, the returned value will be the attribute value of the selected element, otherwise, the inner content of the element will be returned.

If there is no match, an empty string will be returned or, if present, the given `default` value.

#### Randomization Functions

#### `__random_uuid()`

Returns a random v4 UUID string.

#### `__random_boolean([probability=50])`

Returns a random boolean value. You may pass an integer between 0 and 100 as the `probability` to get `true` - defaults to 50%.

#### `__random_number([max],[min=0,max=2^32])`

Returns a random integer between 0 and 232. By passing a positive integer you can set a lower maximum, e.g. `__random_number('30')` will resolve to a number between 0 and 30, inclusive. You can also set the minimum, e.g. `__random_number('10','30')` will be between 10 and 30, inclusive.

#### `__random_chars([length=10])`

Returns a random string of `length` alpha-numeric characters.

#### `__random_digits([length=10])`

Returns a random string of `length` decimal digits.

#### `__random_letters([length=10])`

Returns a random string of `length` english letters.

#### `__random_uppers([length=10])`

Returns a random string of `length` upper-case english letters.

#### `__random_lowers([length=10])`

Returns a random string of `length` lower-case english letters.

#### `__random_hex([length=10])`

Returns a random string of `length` hexadecimal digits.

#### `__random_from(chars,[length=10])`

Returns a random string of `length` characters present in `chars`. For example, you may generate a random **uppercase** hexadecimal string using `__random_from('0123456789ABCDEF')`.

#### Time Functions

#### `__now()`

Returns the current time \(of evaluation\) given as UTC milliseconds. Alias: `_now_ms`.

#### `__now_iso([addedMinutes=0])`

The same as `__now` but given in ISO-8601 format while adding `addedMinutes` . For example,  you may generate the current time + 15 minutes using `__now_iso('15')`.

#### **`__date_iso()`**

The same as `__now_iso` but given in a "date only" format - i.e. `2020-03-03` 

