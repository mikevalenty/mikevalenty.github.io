---
layout: post
title: "Domain Driven Design in the Small"
date: 2009-08-08 06:24
comments: true
categories: [Decorator Pattern, Domain Driven Design, Refactoring]
---

A few months ago we built a [Magento](http://www.magentocommerce.com/) extension to send orders to a product supplier via soap for payment and fulfillment along with affiliate tracking. As part of the process, a contact record was created in the affiliate’s CRM account.

 {% img /images/posts/SubmitYourIdea1.jpg %}

Recently, the stake holders came up with a twist that went something like this: If the order contains a gift card, add the contact to a specific folder in the CRM application. No big deal, we had a nicely abstracted `OrderGateway` interface and I was already envisioning a quick addition to the existing decorator chain.

``` php
class OrderWithGiftCardGateway extends OrderGatewayDecorator
{
    ...
    
    public function createOrder(CreateOrderRequest $order)
    {
        if ($this->containsGiftCard($order))
        {
            $this->addContactToFolder($order);
        }

        return parent::createOrder($order);
    }
}
```

I had a few minutiae questions like what happens with duplicates, etc. It took me nearly an hour to track down the right person and get real answers. During the conversation, a subtle comment was made that I almost missed.

Stake holder: We should check with the product supplier to make sure the gift card sku I made up isn’t for a real product.

**Me:** Say what?

**Stake holder:** The gift card is not fulfilled by the product supplier, it’s fulfilled by the affiliate.

**Me:** %$@!&, I’m glad we had this conversation.

We talked about what it meant for the the affiliate to fulfill the product and basically the folder stuff was okay, but I recommended we remove the fake sku from the order before sending it through.

``` php
public function createOrder(CreateOrderRequest $order)
{
    if ($this->containsGiftCard($order))
    {
        $this->addContactToFolder($order);
        $this->removeGiftCardFromOrder($order);
    }

    return parent::createOrder($order);
}
```

I didn’t have a buddy to pair with so I just grabbed Keith for a minute at the end of the day to walk through things. I recapped the stake holder discussion and we looked through the code. He pointed out I was missing the concept of fulfillment and that was hard earned knowledge that would be lost!

``` php
public function createOrder(CreateOrderRequest $order)
{
    if ($this->containsGiftCard($order))
    {
        $this->sendToAffiliateForFulfillment($order);
        $this->removeGiftCardFromOrder($order);
    }

    return parent::createOrder($order);
}
```

> That was huge. – Paris Hilton

<iframe class="plain" width="640" height="360" src="//www.youtube.com/embed/3nGAk_mo6Rw?feature=player_embedded" frameborder="0" allowfullscreen></iframe>

Why was that huge? Because it changed the conversation. Right away we thought of important requirements like this should be in a transaction with the order and the template email the affiliate gets should include the customer’s address, etc.

It tells an important story for the next guy looking at the code and it changes the role of the programmer from code monkey to business partner. Maybe you think I’m crazy, but this stuff matters to me.