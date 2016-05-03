#The Search for More Cache

In Spaceballs, just before as Lone Starr departs, the all-seeing, all-knowing Yogurt says to him “God willing, we'll all meet again in Spaceballs 2: The Search for More Money.”

Just like Yogurt’s never-ending search for more cash (remember “Merchandising!”), most web developers are on the constant lookout for more performance. (It’s a loose tie-in, okay? Besides, I think I’m the first from Ektron to mention Spaceballs in a blog post, so cut me some slack.) If you have a slow-performing server, if pages aren’t loading as quickly as you like, caching may be an integral part of the solution. But before you dive in, you should observe your server and figure out where the problem lies.

For folks who know why and when to implement caching and just want to get to the good stuff, here are a couple of shortcuts:
* Config files
* Code

For example, if your server is pegging over 80% CPU but RAM and network resources are fine, then Caching may be right for you. However, if you’ve got low CPU usage and are maxing out your RAM, Caching isn’t going to help.

Caching, put simply, is taking data, either rendered output or data retrieved from the DB, and storing it in memory. If you do this, the server does use a little more RAM (RAM is a relatively cheap upgrade, all things considered), but doesn’t need to make so many calls to the database  and, depending on what you cache, can free up valuable processor resources.

There are many ways to implement Caching and I’m not going to say one is wrong or one is right. This is just how I’ve been doing it lately and I think it works out pretty well. Many parts of this method I learned from Brad McDavid, a fellow Ektron CMS Architect, who is brilliant with this stuff.

We use .NET object-level caching rather than page-level. Object-level caching allows you to set different cache times (the time before the object refreshes from the DB) for various items on the page. For example, you may not want to cache an alert on your home page at all, but it would help to cache the navigation for a while. Page-level caching is more effective at freeing up resources, but gives you no granular control over how often inner-page objects refresh. You also don’t want to cache most community controls where your users are contributing content (they want to see their contributions immediately). After describing the difference to a client, he summed it up well in saying “[page caching] is like using a hammer, [object caching] is more like using a scalpel.”

That being said, object-level caching is more time consuming to implement and, if implemented traditionally, can be a pain to adjust later on. What I’m going to show you is how to store your cache timeout values in one place so resetting one will immediately adjust the timeouts for all items at that level.

We will start in the web.config. There are two items to consider here. First, is the setting of the Application.config file in the AppSettings section. Depending on which version you have, this value may already be set. All this really does is allow you to store these settings in a separate file instead of putting everything in the web.config. It makes them easier to find and edit later.

```xml
<appSettings file="Application.config">
```

The second value is just one I want to bring your attention to, and that is one of Ektron’s own keys. Using this toggle, you can enable or disable all caching for Ektron controls and, with the code below, your own logic. Set this value to 0, caching is disabled.

```xml
<add key="ek_CacheControls" value="1" />
```

Obviously, you should almost never flip this switch on your production servers as it will immediately disable all performance improvement you gain from caching your site. This is typically only recommended in development environments and even then I only toggle it off if absolutely necessary. Mostly, I use Brad’s Server Cache widget, available at http://developer.ektron.com/Templates/CodeLibraryDetail.aspx?id=2024&blogid=116, to clear my cached items on-demand. It’s pretty much awesome.

Now to define your cache values. I recommend setting five levels, as shown here in this complete copy of an Application.config file.

```xml
<?xml version="1.0" encoding="utf-8"?>
<appSettings>
  <add key="ekCacheLevel1" value="300" />
  <add key="ekCacheLevel2" value="900" />
  <add key="ekCacheLevel3" value="1800" />
  <add key="ekCacheLevel4" value="3600" />
  <add key="ekCacheLevel5" value="7200" />
</appSettings>
```

I generally never cache anything for less than five minutes (300 seconds). For any given object, you need to ask yourself “How long am I willing to wait for this to update?” And then set the cache value accordingly. Having standard set levels like this is a good thing so you can predict your content refresh cycle a little easier.

Best practice would dictate that the more complex the object, the longer you should cache it (if you can). ContentBlocks are relatively simple in the Ektron world, and so might be set at ekCacheLevel1 or ekCacheLevel2 if you would like them to update more frequently. More complex objects, such as Menus or Taxonomy (especially multi-level taxonomy) are vastly more complex and should be set to higher intervals if your business rules allow for it.

These are simply general guidelines to an approach since business requirements change with every web site. Find the intervals that work for you and make this part of your editor education. Inform your content contributors that it may take up to 15 minutes for their work to go live on the site. Most are willing to accept this if you explain it in the right way.

Now for the fun stuff. Implementing the code.

To make it easier to access those Application.config settings, add the following to your includes:

```c#
using System.Configuration;
```

These two lines retrieve the information from AppSettings.

```c#
bool enableCache = (ConfigurationManager.AppSettings["ek_CacheControls"] == "1");
double defaultCache = double.Parse(ConfigurationManager.AppSettings["ekCacheLevel4"]);
```

And I like to define my CacheKey with a string object to reduce the possibility that you’ll type it differently later (plus it’s less typing overall). Note that here I am including the reason I’m using Taxonomy (LeftMenu) as well as the data type (Taxonomy), the ID, the language and the cache duration. This keeps the string unique to this implementation. Taxonomy has many, many options and in another template I may want to use the same taxonomy in a different way. This string keeps those two functions from colliding.

```c#
string cacheString = "Ektron:Cache:LeftMenu:Taxonomy:ID" + taxonomyId.ToString() + 
":LangType" + taxAPI.ContentLanguage.ToString() + ":T" + defaultCache.ToString();
```

Next, and this is important, grab the item from Cache and store it in a generic object (we’ll take care of that later). Notice in the next line that I am testing this object to see whether it’s null and also testing whether Caching is enabled (based on the setting in web.config; see how it ties together?).

Note that one reason for grabbing the Cached value instead of checking the Cache for null directly and retrieving the value later is that there is a very remote chance that the cache could expire in the cycles in between those two actions. This way we avoid that possibility altogether, remote though it may be.

I also need to create the object I’m going to populate with the cached value.

```c#
TaxonomyData data;
object cachedObject = Cache[cacheString];
if (cachedObject == null || !enableCache)
{
```

Next comes the API code, which I will not describe in detail. Suffice it to say that you can put just about anything here and cache the result. You can even build the output and cache that, which saves even more CPU cycles. Here I’m just caching the data from the DB because of the specific needs for this site.

```c#
	tREQ = new TaxonomyRequest();
	tREQ.Depth = 0;
	tREQ.GetAnalyticsData = false;
	tREQ.IncludeItems = true;
	tREQ.PageSize = 999;
	tREQ.Preview = (Request["cmsmode"] != null && Request["cmsmode"].ToLower() == "preview");
	tREQ.SortOrder = Ektron.Cms.Controls.EkWebControl.TaxonomyItemSortOrder.taxonomy_item_display_order.ToString();
	tREQ.TaxonomyId = taxonomyId;
	tREQ.TaxonomyItemType = EkEnumeration.TaxonomyItemType.Content;
	tREQ.TaxonomyLanguage = taxAPI.ContentLanguage;
	data = taxAPI.LoadTaxonomy(ref tREQ);
```

Now we actually insert the data into cache. Note that I am double-checking that caching is actually enabled and that the data I’ve retrieved is not null. Trying to cache a null value will result in an error.

The insert function is as simple as they come. First is the CacheKey (the identifying string we created earlier), followed by the data, dependencies (of which there currently are none since I built them all into the cache key), then the expiration. You’ll see two options for the expiration and you can use one or the other. The absolute expiration says that this cache will expire in X seconds, no questions asked. The sliding expiration option operates differently in that it will expire the cache only if it’s been unused in the duration specified. So using a sliding expiration of 10 minutes on your home page might mean the cache doesn’t expire until the app pool recycles (every 29 hours, by default) provided your site is accessed more frequently than that.

```c#
	if (enableCache && data != null)
	{
		Cache.Insert(cacheString, data, null, DateTime.Now.AddSeconds(defaultCache), System.Web.Caching.Cache.NoSlidingExpiration);
	}
}
```

Now that’s all well and good if the Cache is null, but what about when it’s not? Well, that’s just a very simple matter of casting the generic object into whatever I need it to be. In this case, TaxonomyData.

```c#
else
{
	data = (TaxonomyData)cachedObject;
}
```

Now I’m free to use the TaxonomyData object just as I would without caching.

You may be saying “That’s great for custom code, but what about Ektron’s out-of-the-box controls?” Not to worry. There’s a one-line-per-control solution there as well. And it looks like this:

```c#
ContentBlock1.CacheInterval = double.Parse(ConfigurationManager.AppSettings[“ekCacheLevel2”]);
```

Since Ektron controls have all the code in place for caching already, as well as adherence to the ek_CacheControls setting in web.config, this is all you have to do to make sure those controls also fall in line with your global settings.

With this in place throughout your site, you can adjust the cache time up or down from the config file. One change affects all objects which you set to use that level. Plus, since you’ve made the cache duration part of the cache key, a change in the config file has an instant affect and you don’t have to wait for the existing cache to expire. Since this has such an effect, of course, it’s best to consider the ramifications of changing one of these values and to do it during a low-traffic time of day whenever possible.

I hope this helps you take more control over the performance of your site. Remember that not all items deserve to be cached and that some items can and should be cached longer than others. Set up as many levels as you think you’ll need.

If you like this approach, see room for improvement or have other recommendations, I’d love to hear them in your comments below.

Happy coding,
eGandalf
