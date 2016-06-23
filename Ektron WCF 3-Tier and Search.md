#Ektron 3-Tier WCF and Search

I generally love when a [question or request in the community](http://developer.ektron.com/Forums/?v=t&t=3447) is the inspiration for a blog post, and this is one of those cases. An Ektron developer, Uttam (not sure if that's an actual or screen name), came looking for the *right way* to go about making a search query in Ektron, but from a remote app (3-Tier).

For those who don't know, Ektron actually provides all the tools necessary to do this, including a remote-app .NET SDK. So you don't need to integrate the services directly, just pull over some DLLs and config files (with some adjustments).

##Prepping your Ektron Site

