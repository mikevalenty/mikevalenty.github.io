---
layout: post
title: "Where Does it Hurt?"
date: 2009-12-26 20:30
comments: true
categories: [AOP, Chain of Responsibility, Decorator Pattern, Inversion of Control, Open Closed Principle]
---

Notes from my Brown Bag Learning Forum Presentation. [Download the source code](http://www.secure-session.com/files/20/205/1664955444/E20E1B7ED7/i/brownbag-20110205b.zip), or just sit back and relax.

### GoF Patterns:

* Chain of Responsibility
* Decorator
* Adapter

### Buzz Phrases:

* Single Responsibility Principle
* Open-Closed Principle
* Inversion of Control
* Aspect Oriented Programming

Embracing the Single Responsibility Principle, Open-Closed Principle and Inversion of Control results in trading fragile application logic for fragile configuration logic. That’s a pretty good trade.

Fragile application logic is costly and it will come back to hurt you repeatedly. It goes without saying that fragile application logic is not testable, otherwise it wouldn’t be fragile. No tests mean changes are scary, so you have to compensate by regaining an intimate understanding of all the twists and turns in order to have enough confidence to make the change. The time and mental energy it takes to work through delicate and subtle conditional logic is enormous. My mediocre brain can only manage a short call stack and juggle a handful of variables at once.

Let’s say you’re sold on the promise of pretentious acronyms like SRP, OCP, IoC and the like. So now you end up with a billion small classes and gobs of code dedicated to wiring-up your favorite inversion of control container (mine is Unity). Are we better for it? Let’s examine this trade-off by implementing the same functionality conventionally and using fancy patterns and principles.

The Scenario: Implement captcha as the first step in some business process, like a registration form.

{% img plain /images/posts/captcha2.png %}

That’s pretty easy, I don’t need anything fancy to do that.

``` c#
public class SimpleController : Controller {
    private const string CaptchaTextKey = "captcha_text";

    public ActionResult Index() {
        return View();
    }

    public ActionResult Render() {

        var captchaImage = new CaptchaImage();

        HttpContext.Session[CaptchaTextKey] = captchaImage.Text;

        using (Bitmap bitmap = captchaImage.RenderImage()) {
            using (var stream = new MemoryStream()) {
                bitmap.Save(stream, ImageFormat.Jpeg);
                return new FileContentResult(stream.ToArray(), "image/jpeg");
            }
        }
    }

    [AcceptVerbs(HttpVerbs.Post)]
    public ActionResult Verify(string captchaText) {

        var actualText = (string)HttpContext.Session[CaptchaTextKey];

        if (captchaText.Equals(actualText)) {
            ViewData["message"] = "You are a human";
        } else {
            ViewData["message"] = "fail";
        }

        return View("Index");
    }
}
```

Fast forward 3 months and amazingly new requirements have crept into the simple captcha controller. Consider these contrived yet poignant scenarios:

* After the project goes to QA, you realize there needs to be a way to bypass the captcha check so automated Selenium tests can get to the rest of the application.
* After the project goes live, the business decides it wants to know how many users are hitting the form. So an audit log is added.
* After reviewing the audit log, it is discovered that some IPs are attacking the form, so we decide to implement a black list.
* After the black list is live, the servers begin to perform slowly, so detailed logging is added.
* The logs show the black list lookup is slow during attacks, so it is determined that caching should be implemented.
* The business is doing a television promotion and expects traffic spikes. IT wants real-time visibility to monitor the application during heavy load.

Now our `SimpleController` has morphed into a `SmellyController`:

``` c#
public class SmellyController : Controller {

    // ...

    [AcceptVerbs(HttpVerbs.Post)]
    public ActionResult Verify(string captchaText) {

        var ip = HttpContext.Request.ServerVariables["REMOTE_ADDR"];

        var isBlackListed = IsBlackListed(ip);

        if (IsMatch(captchaText) && !isBlackListed || captchaText == "selenium" {
            ViewData["message"] = "You are a human";

            // reset consecutive error count
            ConsecutiveErrorCount = 0;
        } else {
            ViewData["message"] = "fail";

            // add to black list
            if (ConsecutiveErrorCount++ >= 3 && !isBlackListed) {
                AddToBlackList(ip);
            }
        }

        WriteToAuditLog(ip);

        return View("Index");
    }

    private bool IsBlackListed(string ip) {
        var sessionKey = BlackListKey + ip;
        object result = HttpContext.Cache[sessionKey];

        if (result == null) {
            using (var connection = new SqlConnection(connectionString)) {
                connection.Open();
                var sql = "select count(*) from blacklist where ip = @ip";
                using (var command = new SqlCommand(sql)) {
                    command.Connection = connection;
                    command.Parameters.AddWithValue("@ip", ip);
                    result = Convert.ToBoolean(command.ExecuteScalar());
                }
            }
            HttpContext.Cache[sessionKey] = result;
        }

        return (bool)result;
    }

    private int ConsecutiveErrorCount {
        get { return Convert.ToInt32(HttpContext.Session[ErrorCountKey] ?? "0"); }
        set { HttpContext.Session[ErrorCountKey] = value; }
    }

    // ...

    private bool IsMatch(IEquatable<string> captchaText) {
        var actualText = (string)HttpContext.Session[CaptchaTextKey];
        return captchaText.Equals(actualText);
    }
```

How does this smell? Let me count the ways:

1. [Hard to test](http://xunitpatterns.com/Hard%20to%20Test%20Code.html)
2. [Mixed levels of abstraction](http://c2.com/cgi/wiki?MixingLevels)
3. No separation of concerns

What if we had followed [Uncle Bob’s SOLID principles](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)? Our controller might look like this:

``` c#
public class SolidController : Controller {
    private readonly ICaptchaProvider captchaProvider;

    public SolidController(ICaptchaProvider captchaProvider) {
        this.captchaProvider = captchaProvider;
    }

    public ActionResult Index() {
        return View();
    }

    public ActionResult Render() {
        using (var stream = new MemoryStream()) {
            captchaProvider.Render(stream, ImageFormat.Jpeg);
            return new FileContentResult(stream.ToArray(), "image/jpeg");
        }
    }

    [AcceptVerbs(HttpVerbs.Post)]
    public ActionResult Verify(string captchaText) {

        if (captchaProvider.Verify(captchaText)) {
            ViewData["message"] = "You are a human";
        } else {
            ViewData["message"] = "fail";
        }

        return View("Index");
    }
}
```

And our original simple captcha provider could look like this:

``` c#
[Serializable]
public class SimpleCaptchaProvider : ICaptchaProvider {
    private string captchaText;

    public void Render(Stream stream, ImageFormat format) {
        var captchaImage = new CaptchaImage();

        captchaText = captchaImage.Text;

        using (Bitmap bitmap = captchaImage.RenderImage()) {
            bitmap.Save(stream, format);
        }
    }

    public bool Verify(string text) {
        return text.Equals(captchaText);
    }
}
```

If you’re wondering why we don’t have to store the captcha text in the session, it’s because we’re putting the onus on the container to give us the same instance of `SimpleCaptchaProvider` each time it’s requested in the same session.

Let’s revisit the list of features that made our controller smelly and see how we could have done it open-closed style (by writing new code instead modifying old code). The go-to technique for this is the decorator pattern. So let’s make a decorator to look for a secret captcha password that our selenium test knows and let it through.

``` c#
public class SeleniumBypassCaptchaProvider : ICaptchaProvider {
    private readonly ICaptchaProvider captchaProvider;
    private readonly string password;

    public SeleniumBypassCaptchaProvider(
        ICaptchaProvider captchaProvider,
        string password) {

        this.captchaProvider = captchaProvider;
        this.password = password;
    }

    public void Render(Stream stream, ImageFormat format) {
        captchaProvider.Render(stream, format);
    }

    public bool Verify(string captchaText) {
        if (captchaText == password) {
            return true;
        }
        return captchaProvider.Verify(captchaText);
    }
}
```

Next is the audit log, then the black list. These could be implemented as two more decorators, however we’re outgrowing this solution which means it’s time to refactor. Let’s switch hats for a few minutes and promote our decorator chain into an explicit chain of responsibility. This is like adding another lane to the freeway, it really opens things up. We’re modifying existing code so maybe you’re wondering what happened to our open-closed principle? It’s still there, I promise. The first point I’ll make is that refactoring is a special activity. It does not change the observable behavior of the application. When working with single responsibility classes, all we end up doing is adapting the logic to a different interface. In our case, we’re moving logic from `BlackListCaptchaProvider` to `BlackListVerifyFilter`. The logic stays intact and the unit tests are minimally impacted.

The end result of this refactor might look like this:

``` c#
public class VerifyChainCaptchaProvider : ICaptchaProvider {
    private readonly ICaptchaProvider captchaProvider;
    private readonly IServiceProvider serviceProvider;

    public VerifyChainCaptchaProvider(
        ICaptchaProvider captchaProvider,
        IServiceProvider serviceProvider) {

        this.captchaProvider = captchaProvider;
        this.serviceProvider = serviceProvider;
    }

    public void Render(Stream stream, ImageFormat format) {
        captchaProvider.Render(stream, format);
    }

    public bool Verify(string captchaText) {
        return new Pipeline<string , bool>(serviceProvider)
            .Add<AuditLoggingFilter>()
            .Add<BlackListingFilter>()
            .Add<SeleniumBypassFilter>()
            .Add(new CaptchaProviderAdapter(captchaProvider))
            .Process(captchaText);
    }
}
```

Was it worth it? Well, we’re left with all these little classes each with their single responsibility, however we still have to wire it up. I’m not going to lie, it’s ugly and it’s fragile. So why is this design any better? It’s better because troubleshooting bad configuration is better than troubleshooting bad application logic. Bad application logic can do really bad things. In a 24/7 business-critical application, this usually happens around 3 AM and involves you waking up and trying adjust your eyes to a harsh laptop screen. With bad configuration on the other hand, whole chunks of functionality are just missing. Chances are your app won’t even start, or maybe it starts but the black list or the audit logging isn’t wired in. These issues are easy to test and when you fix them, you have enormous confidence that the functionality you just wired-in will work and continue to work in the wee hours of the morning.

The second point I’ll make is the code more closely follows the way we think about our application. This makes it easier to respond to change because new requirements are extensions of the way we already think about our application. Consider the scenario that our selenium secret passphrase is not secure enough for production and we want to add an IP restriction or signature to make sure it’s really our selenium test that is getting through. In our smelly controller, a selenium test bypass is not an explicit concept, it’s just an or-clause tacked on to the end of an already abused if statement. We’ll have to go into our smelly controller and do some thrashing around in the most important and delicate block of code. In our solid controller however, we have a nicely abstracted testable single responsibility class we can isolate our change to.

As another example, consider the scenario that our black list caching is consuming too much memory on the web server. With our SOLID design we can surgically replace our `ICacheProvider` with an implementation backed by Memcached. Bits of functionality are free to evolve at their own pace. Some areas of your application will need a beefy solution and some will be just fine with a simple one. The important thing is that concerns are isolated from each other and allowed to fulfill their own destiny.

## Aspect-Oriented Programming

I mentioned aspect oriented programming at the beginning of the article in a shallow attempt to pique your interest. So before I wrap things up I’ll show how it fits in. Since we’re already using an IoC container and faithfully employing our SOLID design principles, we pretty much get AOP for free. This is a big deal. Software running under service level agreements and government regulations demands visibility and having aspects in your toolbox is a must. Because aspects are reusable, they are typically higher quality and more mature than something hand-rolled for a one-off scenario. And because they are bolt-on, our core business logic stays focused on our business domain.

Consider the cliché logging example. It’s overused, but works well not unlike the calculator example for unit testing or the singleton job interview question. The idea is that we tell our IoC container to apply a logging aspect to all objects it has registered. Here’s what my logging aspect produces:

<pre style="background-color: black; color: #ddd;">
DEBUG VerifyChainCaptchaProvider.Render() stream = MemoryStream, format = Jpeg
DEBUG SimpleCaptchaProvider.Render() stream = MemoryStream, format = Jpeg
DEBUG SimpleCaptchaProvider.Render() [72 ms]
DEBUG VerifyChainCaptchaProvider.Render() [74 ms]
DEBUG VerifyChainCaptchaProvider.Verify() text = afds
DEBUG AuditLoggingFilter.Process() input = afds
DEBUG BlackListingFilter.Process() input = afds
DEBUG CachingBlackListService.IsBlocked() ip = 127.0.0.1
DEBUG CachingBlackListService.IsBlocked() > False [0 ms]
DEBUG SeleniumBypassFilter.Process() input = afds
DEBUG SimpleCaptchaProvider.Verify() text = afds
DEBUG SimpleCaptchaProvider.Verify() > False [0 ms]
DEBUG SeleniumBypassFilter.Process() > False [1 ms]
DEBUG BlackListingFilter.Process() > False [4 ms]
DEBUG AuditLoggingFilter.Process() > False [8 ms]
DEBUG VerifyChainCaptchaProvider.Verify() > False [14 ms]
</pre>

Again, this is powerful because we didn’t have to pollute our code with logging statements, yet we see quality log entries with input, output and execution time. In addition to logging, we can attach performance counters to meaningful events, add exception policies to notify us when things go wrong, selectively add caching at run-time and lots more. To see it all in action, you can [download the source code](http://www.secure-session.com/files/20/205/1664955444/E20E1B7ED7/i/brownbag-20110205b.zip) for the examples used in this article.