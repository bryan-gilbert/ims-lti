# ims-lti

[![Build Status](https://travis-ci.org/omsmith/ims-lti.svg?branch=master)](https://travis-ci.org/omsmith/ims-lti)
[![Coverage Status](https://coveralls.io/repos/omsmith/ims-lti/badge.png)](https://coveralls.io/r/omsmith/ims-lti)
[![dependencies Status](https://david-dm.org/omsmith/ims-lti/status.svg)](https://david-dm.org/omsmith/ims-lti)
[![devDependencies Status](https://david-dm.org/omsmith/ims-lti/dev-status.svg)](https://david-dm.org/omsmith/ims-lti?type=dev)

This is a nodejs library used to help create Tool Providers for the
[IMS LTI standard](http://www.imsglobal.org/lti/index.html). Tool Consumer implmentation is left as an excersise to the reader :P

## Install

```shell
npm install ims-lti --save
```

To require the library into your project

```javascript
require('ims-lti')
```

## Supported LTI Versions

* 1.0 - [Implementation Guide](http://www.imsglobal.org/lti/blti/bltiv1p0/ltiBLTIimgv1p0.html)
* 1.1 - [Implementation Guide](http://www.imsglobal.org/LTI/v1p1/ltiIMGv1p1.html)
* 1.1.1 - [Implementation Guide](http://www.imsglobal.org/LTI/v1p1p1/ltiIMGv1p1p1.html)

## Usage

The LTI standard won't be covered here, but it would be good to familiarize yourself with the specs. [LTI documentation](http://www.imsglobal.org/lti/index.html)

This library doesn't help you manage or distribute the consumer keys and secrets. The POST
parameters will contain the `oauth_consumer_key` and your application should use that to look up the consumer secret from your own datastore.

This library offers a few interfaces to use for managing the OAuth nonces to make sure the same nonce
isn't used twice with the same timestamp. [Read the LTI documentation on OAuth](http://www.imsglobal.org/LTI/v1p1pd/ltiIMGv1p1pd.html#_Toc309649687). They will be covered below.

### Setting up a Tool Provider (TP)
As a TP your app will receive a POST request with [LTI launch data](http://www.imsglobal.org/lti/v1p1pd/ltiIMGv1p1pd.html#_Toc309649684) that will be signed with OAuth using a key/secret that both the TP and Tool Consumer (TC) share. This is all covered in the [LTI security model](http://www.imsglobal.org/lti/v1p1pd/ltiIMGv1p1pd.html#_Toc309649685)

Once you find the `oauth_consumer_secret` based on the `oauth_consumer_key` in the POST request, you can initialize a `Provider` object with them and a few other optional parameters:

```javascript
const { Provider } = require('ims-lti');

provider = new Provider(consumer_key, consumer_secret, [nonce_store=MemoryStore], [signature_method=HMAC_SHA1])
```

Once the provider has been initialized, a reqest object can be validated against it. During validation, OAuth signatures are checked against the passed consumer_secret and signautre_method ( HMAC_SHA1 assumed ). isValid returns true if the request is an lti request and is properly signed.

```javascript
provider.valid_request(req, (err, isValid) => {
  // isValid = Boolean | always false if err
  // err = Error object with method descibing error if err, null if no error
})
```

After validating the reqest, the provider object both stores the requests parameters (excluding oauth parameters) and provides convinience accessors to common onces including `provider.student`, `provider.ta`, `provider.username`, and more. All request data can be accessed through `provider.body` in an effort to namespace the values.

Currently there is not an emplementation for posting back to the Tool Consumer, although there is a boolean accessor `provider.outcome_service` that will return true if the TC will accept a POSTback.

### Nonce Stores

`ims-lti` does not standardize the way in which the OAuth nonce/timestamp is to be stored. Since it is a crutial part of OAuth security, this library implements an Interface to allow the user to implement their own nonce-stores.

#### Nonce Interface
All custom Nonce stores should extend the NonceStore class and implment `isNew` and `setUsed`

```javascript
class NonceStore
  isNew(nonce, timestamp, callback) {}
  setUsed(nonce, timestamp, callback) {
    // Sets any new nonce to used
  }
```

Two nonce stores have been implemented for convinience.

#### MemoryNonceStore
The default nonce store (if none is specified) is the Memory Nonce Store. This store simply keeps an array of nocne/timestamp keys. Timestamps must be valid within a 5 minute grace period.

#### RedisNonceStore
A superior nonce store is the RedisNonceStore. This store requires a secondary input into the constructor, a redis-client. The redis client is used to store the nonce keys and set them to expire within a set amount of time (default 5 minutes). A RedisNonceStore is initialized like:

```javascript
const RedisNonceStore = require('../src/redis-nonce-store')
const client          = require('redis').createClient()
const store           = new RedisNonceStore('consumer_key', client)

const provider = new Provider(consumer_key, consumer_secret, store)
```

### Outcomes Extension

The outcomes feature is part of the LTI 1.1 specification and is new to ims-lti 1.0. All of the behind-the-scenes work necessary to get the ball rolling with it is already implemented for you, all you need to do is submit grades.

```javascript
const provider = new Provider(consumer_key, consumer_secret)

provider.valid_request(req, (err, is_valid) => {
  // Check if the request is valid and if the outcomes service exists.
  if (!is_valid || !provider.outcome_service) return false

  // Check if the outcome service supports the result data extension using the
  // text format. Available formats include text and url.
  console.log provider.outcome_service.supports_result_data('text')

  // Replace accepts a value between 0 and 1.
  // Log value will be True or false
  provider.outcome_service.send_replace_result(0.5, (err, result) => console.log(result))

  // Value of the result already submitted from this embed
  provider.outcome_service.send_read_result((err, result) => console.log(result))

  // Log value will be True or false
  provider.outcome_service.send_delete_result((err, result) => console.log(result))

  // Log value will be True or false
  provider.outcome_service.send_replace_result_with_text(0.5, 'Hello, world!', (err, result) => console.log(result))

  // Log value will be True or false
  provider.outcome_service.send_replace_result_with_url(0.5, 'https://google.com', (err, result) => console.log(result))
})
```

### Content Extension

The content extension is an extension supported by most LMS platforms. It provides LTI providers a way to send content back to the LMS in the form of urls, images, files, oembeds, iframes, and even lti launch urls.

```javascript
provider = new Provider consumer_key, consumer_secret

provider.valid_request (req, (err, is_valid) => {
  // check if the request is valid and if the content extension is loaded.
  if (!is_valid || !provider.ext_content) return false

  provider.ext_content.has_return_type('file') // Does the consumer support files
  provider.ext_content.has_file_extension('jpg') // Does the consumer support jpg

  // All send requests take a response object as the first parameter. How the
  // response object is manipulated can be overrided by replacing
  // lti.Extensions.Content.redirector with your own function that accepts two
  // parameters, the response object and the url to redirect to.
  provider.ext_content.send_file(res, file_url, text, content_mime_type)

  provider.ext_content.send_iframe(res, iframe_url, title_attribute, width, height)

  provider.ext_content.send_image_url(res, image_url, text, width, height)

  provider.ext_content.send_lti_launch_url(res, launch_url, title_attribute, text)

  provider.ext_content.send_oembed(res, oembed_url, endpoint)

  provider.ext_content.send_url(res, hyperlink_url, text, title_attribute, target_attribute)
})
```

## Running Tests
To run the test suite first installing the dependencies:

```shell
npm install
npm test
```


## Additions from original

### Problem solving authentication issues

During development it can help to see inside this plugin and look at the parameters being used
to create the signature.  The provider class now has an optional ```withDetailsCallback``` argument.
If this is a function then it will be used at key places. The function is to take one argument ```details``` 
which is an object containing information. It will tell you which class and method is invoking the call.

This can be used in the section of your code that handles a failed validation.

It was very useful to solve two problems.  1. HTTPS proxy support and 2 HTTP to HTTPS

### HTTPS proxy support
When the LTI plugin is used by an API that is proxied behind a HTTPs server the request protocol is converted
from the caller's HTTPS to HTTP. The solution is to set the request x-forwarded-proto header in the proxy setup.
This plugin now looks for ```req.headers['x-forwarded-proto'] ==='https'``` and corrects the protocol for signing.

### HTTP to HTTPS
If the LMS is running on an HTTP server (e.g. your development laptop) and it is calling into a HTTPS API server the
validation may fail.  
This issue is unresolved.
