# Docker Poltergeist

This image is created for testing websites with Capybara, Minitest and
Poltergeist. This docker image is using the PhantomJS docker image as
base https://hub.docker.com/r/wernight/phantomjs/

## Software

- phantomjs 2.1
- ruby 2.3
- gems: bundler, capybara, minitest, poltergeist

## Usage

Below an example `.gitlab-ci.yml` file in which this image is used

### .gitlab-ci.yml
```yaml
image: mipmip/poltergeist:latest

stages:
  - review
  - test
  - deploy_and_test
  - deploy

test:
  stage: test_review
  script:
    - bundle install
    - BASEURL=http://$CI_BUILD_REF_SLUG-$CI_PROJECT_ID.review.ci-domain.net bundle exec rake
  artifacts:
    paths:
    - test/tmp/
    when: on_failure
  only:
    - branches
  except:
    - master
```


Below a test helper and a test file which are used to test TYPO3
websites.

### helper.rb
```ruby
require 'rubygems'
require 'minitest'
require 'minitest/unit'
require 'minitest/autorun'
require 'minitest/pride'
require 'fileutils'
require 'capybara/dsl'
require "capybara/poltergeist"
require "phantomjs"

Capybara.configure do |config|
  config.javascript_driver = :poltergeist
  config.default_driver = :poltergeist
  config.ignore_hidden_elements = false
  config.app_host   = ENV['BASEURL']
end

PHANTOMJS_OPTIONS = [
  '--webdriver-logfile=/dev/null',
  '--load-images=yes',
  '--debug=no',
  '--ignore-ssl-errors=yes',
  '--ssl-protocol=TLSv1'
]
PHANTOMJS_URL_BLACKLIST = [
  'youtube.com', # the youtube embed player doesn't support phantomjs
  'ytimg.com'
]

Capybara.register_driver :poltergeist do |app|
  Capybara::Poltergeist::Driver.new(app,
                                    debug: false,
                                    js_errors: true, # Use true if you are really careful about your site
                                    phantomjs_logger: '/dev/null',
                                    timeout: 60,
                                    :phantomjs_options => PHANTOMJS_OPTIONS,
                                    url_blacklist: PHANTOMJS_URL_BLACKLIST,
                                    window_size: [1920,1080]
                                   )
end

module Minitest
  module TYPO3
    class Test < Minitest::Test
      include Capybara::DSL

      def initialize(name = nil)
        print "\nRunning test case: #{name} "
        @test_name = name
        super(name) unless name.nil?
      end

      def setup
      end

      def teardown
        unless passed?
          page.save_screenshot "test/tmp/testname-#{@test_name}-#{Time.now.strftime('%Y%m%d-%H%M%S')}.png", full: true
        end
      end
    end
  end
end
```

### test_home.rb
```
require 'helper'

class TestHome < Minitest::TYPO3::Test

  def test_home_page_ua_code
    visit '/'
    assert page.has_selector?('script', visible: false, text: 'UA-6530520')
  end

  def test_home_page_base_url
    visit '/'
    assert page.has_xpath?("//base[@href='#{ENV['BASEURL']}/']")
  end

  def test_home_page_visible_content
    visit '/'

    assert page.has_content?('Wat kan het CCV voor u betekenen?')
    assert page.has_content?('Algemene voorwaarden')

    # Nieuw en Agenda Items zijn totaal 8
    assert page.has_selector?('h1.heading--4', :text => "Agenda")
    assert page.has_selector?('h1.heading--4', :text => "Nieuws")
    assert page.has_css?('ul li.news__item', count: 8)
  end

end

```

