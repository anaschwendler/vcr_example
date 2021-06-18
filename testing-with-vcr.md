# Using VCR to Mock Your Requests

In the past months I've been working with a lot of external API requests. Since I'm part of a logistics team we have the challenge to make sure our requests are done in time and everything is delivered in good shape.

Part of this task, as software developer, is making sure the code I write is well tested, and does what is supposed to do. But, when integrating our code connection, we don't want to hit APIs with real requests every time we run our tests. This not only generates unnecessary requests, but as well, make them run slower. In order pretend that we are making the real request, there is a tool that helps "faking" this call, while using accurate data from a true request, and recording it as it was a "cassete tape". The name of the tool I refer to is [VCR](https://github.com/vcr/vcr).

And in this post I'll explain how I've been learning how to use it as well as why I think it is useful.

![](https://media.giphy.com/media/xT1R9RfuBqWvfo8oDe/giphy.gif)

## What is a mock test?

Since this is not about the mocking technique, I'll just briefly introduce it and link a useful article on the topic. In [this post](https://circleci.com/blog/how-to-test-software-part-i-mocking-stubbing-and-contract-testing/) we can see mock testing defined as:

> Mocking means creating a fake version of an external or internal service that can stand in for the real one, helping your tests run more quickly and more reliably. When your implementation interacts with an object’s properties, rather than its function or behavior, a mock can be used.

And that is exactly the reason why we use VCR.

## What is VCR?

According to [their documentation](https://github.com/vcr/vcr):

> Record your test suite's HTTP interactions and replay them during future test runs for fast, deterministic, accurate tests.

Which means that when you first run your test with the VCR syntax, it will record what happened and next time you run the test again, you will have this recorded version ready to stand in for the real request.

## How to use VCR?

So basically, whenever you want to test a part of code that requires a request, you use the `.use_cassette` method to say that you want VCR to deal with that with a cassete file. It means that if there is a file already recorded, VCR will use it, otherwise, VCR will **automatically create** one (this time making a real request) based on the request made in the test.

Example:

```ruby
require 'rubygems'
require 'test/unit'
require 'vcr'

VCR.configure do |config|
  config.cassette_library_dir = "fixtures/vcr_cassettes"
  config.hook_into :webmock
end

class VCRTest < Test::Unit::TestCase
  def test_example_dot_com
    VCR.use_cassette("synopsis") do
      response = Net::HTTP.get_response(URI('http://www.iana.org/domains/reserved'))
      assert_match /Example domains/, response.body
    end
  end
end
```

Here you are requesting to use a cassete file called `synopsis`, to mock a request to iana.org.

If this is running the first time, VCR will generate a "cassete" file, which will be stored at `fixtures/vcr_cassettes`, which for the example will be called `synopsis.yml`.

So if you take a look at `fixtures/vcr_cassettes/synopsis.yml` it looks like (I've removed some parts as it was too big):

```
---
http_interactions:
- request:
    method: get
    uri: http://www.iana.org/domains/reserved
    body:
      encoding: US-ASCII
      string: ''
    headers:
      Accept-Encoding:
      - gzip;q=1.0,deflate;q=0.6,identity;q=0.3
      Accept:
      - "*/*"
      User-Agent:
      - Ruby
  response:
    status:
      code: 200
      message: OK
    headers:
      Date:
      - Fri, 18 Jun 2021 08:06:40 GMT
      Server:
      - Apache
      Vary:
      - Accept-Encoding
      Last-Modified:
      - Thu, 21 May 2020 22:41:39 GMT
      X-Frame-Options:
      - SAMEORIGIN
      Expires:
      - Fri, 18 Jun 2021 09:56:45 GMT
      Referrer-Policy:
      - origin-when-cross-origin
      X-Content-Type-Options:
      - nosniff
      Age:
      - '595'
      Content-Type:
      - text/html; charset=UTF-8
      Cache-Control:
      - public, max-age=21603
      Content-Security-Policy: /*cropped security policy*/
      Transfer-Encoding:
      - chunked
    body:
      encoding: ASCII-8BIT
      string: !binary |-
        /*cropped binary string*/
    http_version:
  recorded_at: Fri, 18 Jun 2021 08:06:40 GMT
recorded_with: VCR 5.0.0
```

So for now on you, whenever you run your tests, you suite will use this cassete file. It will also run faster, as it has the file in place and don't need to make the request which can take more time.

## Examples in real life

One example of project that uses VCR is this `site-search-ruby` client from elastic: https://github.com/elastic/site-search-ruby

In their context of `Search`, when searching all DocumentTypes in the engine, they use a cassete called `engine_search` to mock a request:

https://github.com/elastic/site-search-ruby/blob/master/spec/client_spec.rb

And this is the cassete file that is used in this call: https://github.com/elastic/site-search-ruby/blob/master/spec/fixtures/vcr/engine_search.yml

## Conclusion

VCR seems to be a very reliable tool to fake request to APIs. But we still need to be aware that sometimes APIs can change and with that we need to record our tests again. This process should be simple and easy to reproduce, so for more tips regarding VCR use, I recommend [this](https://fabioperrella.github.io/10_tips_to_help_using_the_VCR_gem_in_your_ruby_test_suite.html) blog post

## References
https://github.com/vcr/vcr

https://github.com/elastic/site-search-ruby

https://circleci.com/blog/how-to-test-software-part-i-mocking-stubbing-and-contract-testing/

https://fabioperrella.github.io/10_tips_to_help_using_the_VCR_gem_in_your_ruby_test_suite.html
