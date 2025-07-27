---
title: "Automatic Deployment From TeamCity Using WebDeploy"
date: 2010-10-27 11:07:00 -0700
layout: post
categories:
---

A solid continuous integration strategy for an enterprise web application is more than just an automated build. You really need a layered approach to maintain a high level of confidence in your code base.

<img class="plain" src="/images/posts/tcp1.jpg">

1.  **Run unit tests** – these are fast running unit tests with no external dependencies. We use NUnit.
2.  **Run integration tests**  – these are tests that have a dependency on a database. The primarily purpose is to test NHibernate mappings.
3.  **Run acceptance tests** – these tests are written in the Given, When, Then style. We write the tests in BehaveN, but we expect that a stakeholder could read them and verify the business rules are correct.
4.  **Deploy to CI and run UI tests** – these are qunit and Selenium tests. They require the code to be deployed to a real web server before the tests run. That’s what this article is about.
5.  **Deploy to QA** – once the automated UI tests have passed, we deploy to our QA server for manual testing.

**Step 1:** [Install WebDeploy on the web server you want to deploy to.](http://weblogs.asp.net/scottgu/archive/2010/09/13/automating-deployment-with-microsoft-web-deploy.aspx)

**Step 2:** [Configure Web.config transforms.](http://blogs.msdn.com/b/webdevtools/archive/2009/05/04/web-deployment-web-config-transformation.aspx) This will enable you to change connection strings and whatnot based on your build configuration.

Currently this is only supported for web applications, but since it’s built on top of MSBuild tasks, you can do the same thing to an `App.config` with a little extra work. Take a peak at `Microsoft.Web.Publishing.targets` (C:\Program Files\MSBuild\Microsoft\VisualStudio\v10.0\Web) to see how to use the build tasks.

```xml
<UsingTask TaskName="TransformXml" AssemblyFile="Microsoft.Web.Publishing.Tasks.dll"/>
<UsingTask TaskName="MSDeploy" AssemblyFile="Microsoft.Web.Publishing.Tasks.dll"/>
```

**Step 3:** Figure out the MSBuild command line arguments that work for your application. This took a bit of trial and error before landing on this:

    C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe MyProject.sln 
      /p:Configuration=QA
      /p:OutputPath=bin
      /p:DeployOnBuild=True 
      /p:DeployTarget=MSDeployPublish 
      /p:MsDeployServiceUrl=https://myserver:8172/msdeploy.axd 
      /p:username=***** 
      /p:password=***** 
      /p:AllowUntrustedCertificate=True 
      /p:DeployIisAppPath=ci
      /p:MSDeployPublishMethod=WMSVC

**Step 4:** Configure the Build Runner in TeamCity

Paste the command line parameters you figured out in Step 3 into the Build Runner configuration in TeamCity:

<img class="plain" src="/images/posts/teamcity2.jpg">

**Step 5:** Configure build dependencies in TeamCity. This means the integration tests will only run if the unit tests have passed and so on.

<img class="plain" src="/images/posts/dep1.jpg">

<img class="plain" src="/images/posts/snap1.jpg">

**Step 6:** Write some code.
