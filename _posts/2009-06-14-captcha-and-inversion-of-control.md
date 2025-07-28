---
title: "Captcha and Inversion of Control"
date: 2009-06-14 20:44:00 -0700
layout: post
categories:
- inversion of control
- single responsibility principle
- tdd
- unity
---

<img class="plain" src="/images/posts/lorenzoflip.jpg">

I made a few tweaks to a captcha library I found here and basically wrapped their CaptchaImage object in a service interface to use in my application. Pretty easy stuff, but I didn’t get it right the first time.

What’s wrong with this class?

```c#
public class HttpContextCacheCaptchaProvider : ICaptchaProvider
{
    private const string httpContextCacheKey = "4D795A45-8015-475C-A6C4-765B09EB9955";
    private ICaptchaImageFactory factory;
    private HttpContextBase context;

    public HttpContextCacheCaptchaProvider(ICaptchaImageFactory factory, HttpContextBase context)
    {
        this.factory = factory;
        this.context = context;
    }

    public void Render(Stream stream)
    {
        CaptchaImage captchaImage = factory.Create();

        context.Cache[httpContextCacheKey] = captchaImage.Text; // hint

        using (Bitmap bitmap = captchaImage.RenderImage())
        {
            bitmap.Save(stream, ImageFormat.Jpeg);
        }
    }

    public bool Verify(string text)
    {
        string captchaText = (string)context.Cache[httpContextCacheKey]; // hint

        if (text == null || captchaText == null)
            return false;
        else
            return text.ToLower().Equals(captchaText.ToLower());
    }
}
```

Perhaps your nose can lead you in the right direction. The smell is hard to test, and it’s hard to test because it requires mocking the `HttpContextBase`. You might say “no problem, I can blast out a stub with Moq in no time” but you’re missing the real problem. That would be like taking aspirin for a brain tumor.

<img class="plain" src="/images/posts/arnold.jpg">

> It’s not a tumor

The real problem is this class is violating the Single Responsibility Principle (SRP) by doing more than one thing. I don’t mean the two methods `Render()` and `Verify()`, those are a cohesive unit operating on the same data. The other thing is lifetime management. Look how simple the class gets when you invert control and take out the notion of lifetime management:

```c#
[Serializable]
public class LifetimeManagedCaptchaProvider : ICaptchaProvider
{
    private ICaptchaImageFactory factory;
    private string captchaText;

    public LifetimeManagedCaptchaProvider(ICaptchaImageFactory factory)
    {
        this.factory = factory;
    }

    public void Render(Stream stream)
    {
        CaptchaImage captchaImage = factory.Create();

        captchaText = captchaImage.Text;

        using (Bitmap bitmap = captchaImage.RenderImage())
        {
            bitmap.Save(stream, ImageFormat.Jpeg);
        }
    }

    public bool Verify(string text)
    {
        if (text == null || captchaText == null)
            return false;
        else
            return text.ToLower().Equals(captchaText.ToLower());
    }
}
```

Now this class is a breeze to test and all you have to do is register it like this:

```c#
Container.RegisterType<ICaptchaProvider, LifetimeManagedCaptchaProvider>(
    new HttpSessionStateLifetimeManager());
```

The moral of the story is let your IoC container do things it’s good at.
