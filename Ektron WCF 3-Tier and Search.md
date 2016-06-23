#Ektron 3-Tier WCF and Search

I generally love when a [question or request in the community](http://developer.ektron.com/Forums/?v=t&t=3447) is the inspiration for a blog post, and this is one of those cases. An Ektron developer, Uttam (not sure if that's an actual or screen name), came looking for the *right way* to go about making a search query in Ektron, but from a remote app (3-Tier).

For those who don't know, Ektron actually provides all the tools necessary to do this, including a remote-app .NET SDK. So you don't need to integrate the services directly, just pull over some DLLs and config files (with some adjustments).

##Prepping your Ektron Site

First, you need to make sure that your app is able to access the Ektron WCF endpoints. When you install Ektron, the default behavior is to add IP restrictions in IIS so that only localhost (127.0.0.1) can access those services. If your app is on the same machine, no worries. Otherwise, you'll have to add your app host IP to the whitelist in IIS.

If you're going to be doing any *writing* or accessing any private content between the app and Ektron, it has to be done with an authenticated user. You can either build authentication into your app or use a dedicated account with the proper authorization and restrictions in place; in other words, only allow the necessary permissions and nothing more - *no admins!*

This article will not cover authenticated WCF requests because they're not necessary to simply perform a search.

###Prerequisites

* Ektron CMS 8.5 or later
* Microsoft Search Server, Microsoft FAST, or SOLR set up, configured, and indexing and returning data successfully
* An IDE, such as Visual Studio

For my test, I'm running Ektron 9.1SP3 (latest at time of writing) and the matching SOLR installation. I'm also running Visual Studio 2015.

##The Approach

You really can use any .NET app to do what I'm about to do. Though the OP is building something in MVC, I chose to spin up a simple Console App in Visual Studio because it requires less effort to get a result and less overhead on my virtual machine to run. The process is the same for either - this is just .NET libraries from Ektron and Unity config.

The real advantage to this approach (Unity) is that it gives you access to (more or less) the exact same Framework APIs that you use in the Ektron application itself. To the point that, for many operations, you can copy the code from one and put it into the other and expect the same outcome.

You can, as the OP was trying to do, integrate directly with the WCF endpoint, but that eliminates all of the constructors and helper methods Ektron provides as part of its API. My way preserves those.

##Adding and Configuring References

Once you've created your app, you'll need to add Ektron's 3-Tier references to your app. Open Solution Explorer and right-click on References and choose Add Reference. You'll want to Browse for the DLLs in your Ektron installation - usually `C:\Program Files (x86)\Ektron\(version)\startersites\3TierMin\Content\bin`. Select all and add them.

![Select all references](images/3-tier-search/references.png)

Next, you'll need Ektron's Unity configurations and some changes to your web/app.config to be able to use the APIs.

