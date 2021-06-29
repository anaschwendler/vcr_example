# Using VCR to Mock Your Requests

In the past months, I've been working with a lot of external API requests. Since I'm part of the Marley Spoon’s logistics team, our challenge is to make sure our requests are done in time and everything is delivered in good shape.

Part of this task, as a software developer, is making sure the code I write is well tested, and does what it is supposed to do. Our software has many integrations with our logistics partners. When integrating with their systems, we don't want to hit external APIs with real requests every time we run our tests. This not only generates unnecessary requests that can cause problems with API rate limits and other unwanted side-effects. It also makes them run slower. To avoid that, we pretend to make real requests and there is a tool that helps "faking" these calls: by recording real data from a real network request, we are able to stub a third party’s web service response. The saved data is called a "cassette tape" and the name of the tool I refer to is [VCR](https://github.com/vcr/vcr).

In this post I'll explain how I've been learning to use it and why I think it is useful.

![](https://media.giphy.com/media/xT1R9RfuBqWvfo8oDe/giphy.gif)

## What is a mock test?

Since this is not about the mocking technique, I'll just briefly introduce it and link to a useful article on the topic. In [this post](https://circleci.com/blog/how-to-test-software-part-i-mocking-stubbing-and-contract-testing/) we can see mock testing defined as:

> Mocking means creating a fake version of an external or internal service that can stand in for the real one, helping your tests run more quickly and more reliably. When your implementation interacts with an object’s properties, rather than its function or behavior, a mock can be used.

… and that is exactly the reason why we use VCR.

## What is VCR?

According to [their documentation](https://github.com/vcr/vcr):

> Record your test suite's HTTP interactions and replay them during future test runs for fast, deterministic, accurate tests.

Which means that when you first run your test with the VCR syntax, it will record what happened and next time you run the test again, you will have the recorded version ready to stand in for the real request.

## How to use VCR?

I have a basic example [here](https://github.com/anaschwendler/vcr_example).

So basically, whenever you want to test a part of code that requires an external web request, you use the  `.use_cassette` method to state that you want VCR to deal with that with a cassette file. If there already is data with a pre-recorded HTTP interaction, VCR will use it. If not, VCR will **automatically create** a cassette file (this time making a real request) based on the request made in the test.

But still, if you are looking forward to making real requests, VCR also offers different record modes, which can be checked [here](https://relishapp.com/vcr/vcr/v/6-0-0/docs/record-modes)

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

Here you are requesting to use a cassette called `synopsis`, which mocks a request to [iana.org](http://www.iana.org/domains/reserved).

If this is running the first time, VCR will generate a "cassette" file, which will be stored at `fixtures/vcr_cassettes`, which for the example will be called `synopsis.yml`.

This is what `fixtures/vcr_cassettes/synopsis.yml` looks like (I've removed some parts as it was too big):

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

From now on your test suite will use this cassette file. It will also run faster, as it has the file in place and doesn't need to make the request which can take more time.

## Examples in real life

One example of project that uses VCR is this `site-search-ruby` client from elastic: https://github.com/elastic/site-search-ruby

In their context of `Search`, when searching all DocumentTypes in the engine, they use a cassette called `engine_search` to mock a request:

https://github.com/elastic/site-search-ruby/blob/master/spec/client_spec.rb

This is the cassette file that is used in this call: https://github.com/elastic/site-search-ruby/blob/master/spec/fixtures/vcr/engine_search.yml

## Conclusion

VCR seems to be a very reliable tool to fake requests to APIs. But we still need to be aware that sometimes APIs can change and with that we need to record our tests again. This process should be simple and easy to reproduce, so for more tips regarding VCR use, I recommend [this](https://fabioperrella.github.io/10_tips_to_help_using_the_VCR_gem_in_your_ruby_test_suite.html) blog post.

### Hiding confidential credentials in vcr_cassettes

There is a configuration option available to filter sensitive data that can be used to prevent it from being written to the cassette files, the documentation is [here](https://relishapp.com/vcr/vcr/v/5-0-0/docs/configuration/filter-sensitive-data). This is important, as you want to track your cassette YML files in your source control management tool. And secrets don’t belong there.

## References
https://github.com/vcr/vcr

https://github.com/elastic/site-search-ruby

https://circleci.com/blog/how-to-test-software-part-i-mocking-stubbing-and-contract-testing/

https://fabioperrella.github.io/10_tips_to_help_using_the_VCR_gem_in_your_ruby_test_suite.html

Thanks [Memuna](https://github.com/memunaharuna) for reviewing it! :tada:
