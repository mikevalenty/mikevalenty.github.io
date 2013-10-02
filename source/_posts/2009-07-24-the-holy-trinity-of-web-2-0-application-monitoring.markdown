---
layout: post
title: "The Holy Trinity of Web 2.0 Application Monitoring"
date: 2009-07-24 06:37
comments: true
categories: [Decorator Pattern, Inversion of Control, Open Closed Principle]
---

We had just rolled out a new system for a client and they were doing a high profile launch of their product. We had all our normal monitoring in place like CPU, memory, connections and page load time. Everything was swell…

On their signup form, we did an ajax call to check if their desired username was available. If it wasn’t, we displayed a validation error an prevented the user from submitting the form. Turns out our little jquery script was silently bombing out and always returning ‘false’ meaning nobody could sign up!

{% img plain /images/posts/scaredmonkey_thumb.png %}

This little issue slipped through the cracks and it hurt pretty bad. I couldn’t just tell the stake holders “sorry”, I needed something a little better so I spent some time with [Matt, our super do-everything networking guy](http://blog.mattbeckman.com/), and we put together the holy trinity of web 2.0 application monitoring (insert enlightenment music here).

{% img /images/posts/trinity.png %}

We were already using [Nagios](http://www.nagios.org/) for monitoring and alerting, [CruiseControl](http://cruisecontrol.sourceforge.net/) for running unit tests, and [Selenium](http://seleniumhq.org/) for automated web application testing. We just needed to glue it all together!

1. The first step was to write a selenium test to go through the online order sequence. We then exported it as a phpUnit test and dropped it in a our svn repository in a folder named “monitoring.”
2. Next, we configured a CruiseControl project named “selenium-bot” to pull down all the phpUnit tests from the “monitoring” folder in svn and run the whole test suite every 10 minutes.
3. The last step was to use Nagios to monitor the CruiseControl log file to make sure it was actually running every 10 minutes and returning all green. If anything stops working, Nagios takes care of the alerting.

I should also mention that since this hits the live online order form every 10 minutes, we needed a way to a way to short circuit the test orders. Fortunately, we already had a convenient OrderGateway interface, so we were able accomplish this in a very [open-closed](http://en.wikipedia.org/wiki/Open/closed_principle) manner using the decorator pattern:

``` php
class TestOrderInterceptor implements OrderGateway
{
  const TEST_CODE = '843feaa7-bf13-4aff-91f6-a074434f9c14';
  const SUCCESS_RESULT = 1;

  private $orderGateway;
  private $logger;

  public function __construct(OrderGateway $orderGateway, Logger $logger)
  {
    $this->orderGateway = $orderGateway;
    $this->logger = $logger;
  }

  public function createOrder(CreateOrderRequest $order)
  {
    if ($this->isTest($order))
    {
      $this->logger->debug('test order intercepted');
      return self::SUCCESS_RESULT;
    }

    return $this->orderGateway->createOrder($order);
  }

  private function isTest($request)
  {
    return strpos($request->name1, self::TEST_CODE) !== false;
  }
}
```

The decorator chain is wired up using [PicoContainer](http://www.picocontainer.org/), and looks like this:

``` php
$pico->regComponentImpl('SoapOrderGateway', 'SoapOrderGateway');
        
$pico->regComponentImpl('OrderGateway', 'TestOrderInterceptor', array(
    new BasicComponentParameter('SoapOrderGateway'),
    new BasicComponentParameter('Logger')));
```

This infrastructure has paid dividends more than a few times and now I can’t imagine rolling out a site without it.