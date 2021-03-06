# GuzzleHttpMock

A mock library for verifying requests made with the [Guzzle Http Client](http://guzzle.readthedocs.org/), and mocking responses.

This is a fork of the [original version](https://github.com/aerisweather/GuzzleHttpMock). This version was changed to support Guzzle 6 and did receive some other minor changes.

- - -


<!-- generated by `marked-toc`: https://www.npmjs.com/package/marked-toc -->
<!-- toc -->

* [Installation](#installation)
  * [Composer](#composer)
* [Overview](#overview)
  * [How does it work?](#how-does-it-work)
* [Usage](#usage)
  * [Attaching to a Guzzle Client](#attaching-to-a-guzzle-client)
  * [Creating Request Expectations](#creating-request-expectations)
		* [Available Expectations](#available-expectations)
		* [Default Expectations](#default-expectations)
		* [Directly Setting an Expected Request](#directly-setting-an-expected-request)
		* [Custom Expectations](#custom-expectations)
  * [Mocking Responses](#mocking-responses)
  * [Verifying Expectations](#verifying-expectations)
  * [Gotchyas](#gotchyas)
    * [Unspecified expectations](#unspecified-expectations)
    * [Where's my UnexpectedRequestException?](#wheres-my-unexpectedrequestexception)
    * [Why's it doing that thing I don't think it should do?](#whys-it-doing-that-thing-i-dont-think-it-should-do)
  * [Contributing](#contributing)
    * [Wish List](#wish-list)

<!-- toc stop -->




## Installation

### Composer

You can install GuzzleHttpMock using composer:

```sh
php composer.phar require --dev systemhaus/guzzle-http-mock
```


## Overview

GuzzleHttpMock allows you to setup Http request expectations, and mock responses.

```php
// Create a mock object
$httpMock = new \Aeris\GuzzleHttp\Mock();

// Create a guzzle http client and attach the mock handler
$guzzleClient = new \GuzzleHttp\Client([
	'base_url' => 'http://www.example.com',
	'handler' => $httpMock->getHandlerStackWithMiddleware();
]);

// Setup a request expectation
$httpMock
    ->shouldReceiveRequest()
    ->withUrl('http://www.example.com/foo')
    ->withMethod('GET')
    ->withBodyParams([ 'foo' => 'bar' ])
    ->andRespondWithJson([ 'faz', 'baz' ], $statusCode = 200);

// Make a matching request
$response = $guzzleClient->get('/foo', ['json' => ['foo' => 'bar'] ]);
json_decode((string) $response->getBody(), true) == ['faz' => 'baz'];  // true
$response->getStatusCode() == 200;      // true
$httpMock->verify();                    // all good.

// Make an unexpected request
$guzzleClient->post('/bar', ['json' => ['faz' => 'baz'] ]);;
$httpMock->verify();
// UnexpectedHttpRequestException: Request does not match any expectation:
// 	Request url does not match expected value. Actual: '/bar', Expected: '/foo'
//	Request body params does not match expected value. Actual: [ 'faz' => 'baz'], Expected: ['foo' => 'bar' ]
```

### How does it work?

When a GuzzleHttpMock Handler is attached to the Guzzle Http client, it will intercept all requests made by the client. Whenever a request is made, the mock checks the request against set expectations, and sends a response to matching requests.
If your client uses custom middleware you can attach it to the handler stack of the mock handler.

Calling `$httpMock->verify()` checks that all expected requests have been made, and complains about any unexpected requests.


## Usage

### Attaching to a Guzzle Client

To start intercepting Http requests, the GuzzleHttpMock must be attached to a GuzzleClient:

```php
// Create a mock object
$httpMock = new \Aeris\GuzzleHttp\Mock();

// Create a guzzle http client and attach mock handler
$guzzleClient = new \GuzzleHttp\Client([
	'base_url' => 'http://www.example.com',
	'handler' => $httpMock->getHandlerStackWithMiddleware()
]);
```

### Creating Request Expectations

The `shouldReceiveRequest` method returns a `\Aeris\GuzzleHttpMock\Expectation\RequestExpectation` object.

```php
$requestExpectation = $httpMock->shouldReceiveRequest();
```

The `RequestExpectation` object uses `withXyz` methods to set expectations:

```php
$requestExpectation->withUrl('http://www.example.com/foo');
```

The expectation setters are chainable, allowing for a fluid interface:

```php
$httpMock
    ->shouldReceiveRequest()
    ->withUrl('http://www.example.com/foo')
    ->withMethod('POST');
```

#### Available Expectations

The following expectations are available on a `\Aeris\GuzzleHttpMock\Expectation\RequestExpectation` object.


Method | Notes
------ | ------
`withUrl($url:string)` | URL (full absolute path)
`withMethod($url:string)` | Http method.
`withQuery($query:\GuzzleHttp\Query)` | Query with a Guzzle query object
`withQueryParams($params:array)` | Query string with array 
`withQueryString($queryString:string)` | Literal query string comparison
`withContentType($contentType:string)` | Content Type
`withJsonContentType()` | JSON Content Type
`withBody($stream:StreamInterface)` | Compare request body
`withBodyParams($params:array)` | Compare request body params
`withJsonBodyParams($params:array)` | JSON body params
`once()` | The request should be made a single time
`times($callCount:number)` | The request should be made `$callCount` times.

#### Default Expectations

By default, a request is expected to be made one time, with an Http method of 'GET'.

```php
// So this:
$httpMock
    ->shouldReceiveRequest()
    ->withUrl('http://www.example.com/foo');

// is the same as this:
$httpMock
    ->shouldReceiveRequest()
    ->withUrl('http://www.example.com/foo')
    ->once()
    ->withMethod('GET');
```


#### Directly Setting an Expected Request

In addition to specifying request expectations individually, you can also directly set a `Psr\Http\Message\RequestInterface` object as an expectation.

```php
$expectedRequest = new Request(
    'PUT',
    'http://www.example.com/foo?faz=baz',
    [
		'body'    => json_encode(['shazaam' => 'kablooey']),
		'headers' => [
			'Content-Type' => 'application/json'
		],
	]
);

$httpClient->shouldReceiveRequest($expectedRequest);
```

#### Custom Expectations

All expectation methods accept either a value or a `callable` as a parameter. By passing a callable, you can create custom expectations. For example:

```php
$httpMock
    ->shouldReceiveRequest()
    ->withBodyParams(function($actualParams) {
        return $actualParams['foo'] === 'bar';
    });
```

In this case, the expectation will fail if the actual request body has a `foo` params which does not equal `bar`.

GuzzleHttpMock provides some built-in custom expectations, as well. For example:

```php
use Aeris\GuzzleHttpMock\Expect;

$httpMock
	->shouldReceiveRequest()
	// Check URL against a regex
	->withUrl(new Expect\Matches('/^https:/'))
	// Check query params against an array
	->withQueryParams(new Expect\ArrayEquals(['foo' => 'bar']))
	// Allow any body params
	->withBodyParams(new Expect\Any());
```

### Mocking Responses

When a request is made which matches an expectation, the GuzzleHttpMock will intercept the request, and respond with a mock response.

```php
$httpMock
  ->shouldReceiveRequest()
  ->withMethod('GET')
  ->withUrl('http://www.example.com/foo')
  ->andRespondWithJson(['foo' => 'bar']);

$response = $guzzleClient->get('/foo');
$response->json() == ['foo' => 'bar'];  // true
```

#### Available Responses

The following methods are available for mocking responses:

Method | Notes
------ | -----
`andRespondWith($response:\Psr\Http\Message\ResponseInterface)` | See [Directly Setting a Mock Response](#directly-setting-a-mock-response)
`andRespondWithContent($data:array, $statusCode:string)` | Sets the response body
`andRespondWithJson($data:array, $statCode:String)` | Sets a JSON response body


#### Directly Setting a Mock Response

You may mock a response directly using a response object:

```php
$response = new \GuzzleHttp\Psr7\Response(
    200,
    ['Content-Type' = 'application/json'],
    json_encode(['foo' => 'bar' ]
);

$httpMock
    ->shouldReceiveRequest()
    ->withMethod('GET')
    ->withUrl('http://www.example.com/foo')
    ->andResponseWith($response);
```

### Verifying Expectations

Expectations may be verified using the `\Aeris\GuzzleHttpMock::verify()` method.

```php
$httpMock
  ->shouldReceiveRequest()
  ->withUrl('http://www.example.com/foo');

$guzzleClient->get('/bar');

$httpMock->verify();
// UnexpectedRequestException: Request does not match any expectation.
//	Request url does not match expected value. Actual: '/bar', Expected: '/foo'.
```

#### With PHPUnit

When using GuzzleHttpMock with PHPUnit, make sure to add `Mock::verify()` to your teardown:

```php
class MyUnitTest extends \PHPUnit_Framework_TestCase {
    private $guzzleClient;
    private $httpMock;

    public function setUp() {
    	// Setup your guzzle client and mock
    	$this->httpMock = new \Aeris\GuzzleHttpMock();
                
    	$this->guzzleClient = new \GuzzleHttp\Client([
			'base_url' => 'http://www.example.com',
			'handler' => $this->httpMock->getHandlerStackWithMiddleware();
		]);
   	}

    public function tearDown() {
    	// Make sure all request expectations are met.
    	$this->httpMock->verify();
        // Failed expectations will throw an \Aeris\GuzzleHttpMock\Exception\UnexpectedHttpRequestException
    }
}
```

### Gotchyas

We have used GuzzleHttpMock enough internally to feel comfortable using it on production projects, but also enough to know that there are a few "gotchyas". Hopefully, knowing these issues up-front will prevent much conflict between your forehead and your desk.

If you'd like to take a shot at resolving any of these issues, take a look at our [contribution guidelines](#contributing).

#### Unspecified expectations

In the current version of GuzzleHttpMock, any expectations which are not specified will result in a failed request.

```php
$httpMock
    ->shouldReceiveRequest()
    ->withUrl('http://www.example.com/foo');

$guzzleClient->get('/foo', [
	'query' => ['foo' => 'bar']
]);

$httpMock->verify();
// UnexpectedHttpRequestException: Request does not match any expectation:
// 	Request query params does not match any expectation: Actual: [ 'foo' => 'bar' ], Expected: []
```

You might argue that it would make more sense for the RequestExpectation to accept any value for unspecified expectations by default. And you might be right. Future versions of GuzzleHttpMock may do just that.





#### Where's my UnexpectedRequestException?

There are a couple of possible culprits here:

1. Make sure you're calling `Mock::verify()`. If you're using a testing framework (eg PHPUnit), you can put `verify()` in the `tearDown` method.

2. Another exception may be thrown before you had a chance to verify your request expectations.

Solving #2 can be a little tricky. If a RequestExpectation cannot be matched, GuzzleHttpClient will not respond with your mock response, which may cause other code to break before you have a chance to call `verify()`.

If you're calling `verify()` in your test `tearDown`, you may want to try adding another `verify()` call immediately after the http request is made. 

You can also try wrapping the offending code in a `try...catch` block, to give the `UnexpectedRequestException` priority.

```php
$this->httpMock
    ->shouldReceiveRequest()
    ->withXYZ()
    ->andRespondWith($aValidResponse);

try {
    $subjectUnderTest->doSomethingWhichExpectsAValidHttpResponse();
}
catch (\Exception $ex) {
    // uh oh, $subjectUnderTest made an unexpected request,
    // and now if does not have a valid response to work with!
    
    // Let's check our http mock, and see what happened
    $httpMock->verify();
    
    // If it's not a request expectation problem, throw the original error
    throw $ex;
}
```

That's more verbosity than you may want in all of your tests, but it can be helpful if you're debugging.


#### Why's it doing that thing I don't think it should do?

I don't know. That's really weird. Bummer...

Hey, why don't you open a new issue and tell us about it? Maybe we can help.


### Contributing

For that warm fuzzy open-sourcey feeling, contribute to GuzzleHttpMock today!

We only ask that you include PHPUnit tests, and update documentation as needed. Also, if it's not an open issue or on our wish list, you might want to open an issue first, to make sure you're headed in the right direction.

#### Wish List

Take a look at the ["Gotchyas"](#gotchyas) section for some things that could be fixed. Have another idea? Open an issue, and we'll talk.
