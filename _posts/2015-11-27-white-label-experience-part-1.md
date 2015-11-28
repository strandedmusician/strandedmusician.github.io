---
title: The White-Label Experience (Part 1)
---

One of the projects I have been working on has been a consumer-facing white-label site for utility companies to process payments. One of the pre-requisites for this was the ability to allow them to customize their consumers' experience with their own branding, color, etc.

Thus far nobody has shown interest in the customization features, and this has provided us the opportunity to address some of the problems we have found along our journey.

## Some background

Our current stack for the application consists of an Angular front-end sitting in a Java WAR along with its REST service, running on a Wildfly instance. The webservice makes remote service calls internally across the network to our Load-balanced CORE wildfly instances for the actual heavy-lifting of internal business logic.

### UI Customization
The customizations to the UI (field relabeling, etc) were done manually, with each field doing something similar to:

{% highlight html %}
<label for="password">{{biller.gui.login.passwordFieldLabel || 'Account Password'}}
	<input type="password" id="password" placeholder="{{biller.gui.login.passwordFieldLabel || 'Account Password'}}" /></label>
{% endhighlight %}

This was cumbersome, took a lot of time to target each customizable piece we wanted, and ultimately was not going to provide the best results. It was a result of an immediate need on a very short timeline, but would not scale. More on that later. But, we had a bigger problem:

### Determining the context of the rendered site

To determine the context of the site (what biller was this consumer attempting to sign in to? What white-label customizations needed to be applied?), we initially attempted to pass in the biller's UUID as a part of the initial page render, such as:

{% highlight linenos %}
https://www.p2d.io/#eb4501d7-44dd-4fe6-99e6-3d9b64e898a5
{% endhighlight %}

When the SPA site loaded up, in our Angular Config phase, we would grab the UUID out of the URL, pull the pertinent details relating to it from the database, and set the "context" of the site to be based on this white-label experience. Due to the fact that the URL would change and we would lose that context in the URL as Angular navigates around routes, we would store it in a cookie. A user could navigate around the site, choose to refresh the page, and we would be able to pull the biller config again with the information in the cookie.

This lead to many problems, which have haunted us many times (mostly to those NOT using the site in a normal flow), such as:
- **Typos** by business users (for demos, primarily)
- Getting **locked in** on one biller (issues with the cookie), and having to refresh the page a couple times to get the new context (never battled that bug into oblivion)
- Could not log in to two billers using our system at the same time (session collision across the cookie for our www.p2d.io domain)
- Simply put, awkward URLs
- Feeling that this solution was brittle
- **Crippling depression**

The use of the cookie was the largest piece of concern for me. There seemed to always be some weird conflict we'd get with how we were handling the information, how the page was loaded up, bookmarked URLs, etc, etc. I wanted it gone, and I wanted it gone NOW!

## What's a better, more elegant solution?
We needed a more elegant solution, both for the code-side of things, and something that would be more elegant for the business, hence we could sell the business on spending the time to move to a new solution.

What if we could have something prettier like this?

{% highlight %}
https://ellis-water.p2d.io/
{% endhighlight %}

Would this solve a code problem? Is there additional maintenance? Is there a logical way to make this happen?

Enter: Wildcard DNS Routing

Wildcard domain routing allows us to route anything without a specific entry, to a common entry-point. For example, we could have our normal "www.p2d.io" now go to our sales page, "help.p2d.io" continue to route to our FAQ/Help S3 Bucket, but everything else (ex: https://mymadeupbiller.p2d.io) would go to the server hosting our Angular Consumer GUI.

The initial perceived benefits of this are:

- URLs are shorter, more **user friendly**
- Brand identity remains in the URL for **customer confidence**
- We **alleviate confusion** about what biller we are interacting with (biller name remains in URL throughout all navigation) in Testing phases
- The context (biller) will always be available on the URL, even as the consumer moves around the site.

On the code-side, the biggest thing is:
We can always get the biller context and no longer need to save it in a cookie, etc. We can cache the request, as well, and it will not collide with requests when logging in to other billers. The cached request is off of a unique subdomain for this biller.

Cons:
- We have to tell billers to update the link on their websites to point to a better-looking link that their customers will have more faith in.
- Consumers who have bookmarked the site will have to update their bookmarks. We will have to know to react to the link and provide the consumer pertinent details to update their links and go to the correct URLs

Not too shabby of a list, and with the ability to notify and forward the consumers using the old link, I think we might have sold the idea!

## Implementation

So, what is required to make this dream a reality? What is required to create a working demo of this? Even with the added tasks of a demo (creating a mock static site for R&D), we have just a few simple steps:
- Create a few fixed DNS entries, to show that the wildcard doesn't trample over them
- Create a wildcard DNS entry to go to our new static R&D site
- Display that we are capable of reading the URL to receive the required information via Javascript (as we will with the full Angular setup)
- Create mock redirects that would occur from the old URLs to the new ones

For this demonstration, a domain like mydnsplayground.net will work nicely!

We leverage Amazon AWS where we can, thus we will be using Route 53 to set up our DNS entries, as well as S3 to serve up the demonstration HTML.

1.  Create an S3 bucket to store the static HTML page that will read the domain
2.  Create an S3 bucket to store content from a "fixed" entry. Putting some identifier in there so that we can identify that it is not the wildcard page
3. Create an entry in Route 53
	- Name: *
	- Type: A Record
	- Alias: YES
	- Alias Target: {Our S3 Bucket from Step #1}
	- Save 
4. Create an entry in Route 53
	- Name: fixed
	- Alias: YES
	- Alias Target: {Our S3 Bucket from Step #2}
	- Save

test
Fixed Domain HTML page:
{% highlight html linenos %}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Wildcard Page!</title>
  </head>
  <body>
  <h1>Wildcard Page!</h1>
    <!-- page content -->
	<p>Biller: <span id="billerId"></span></p>
	
	<script type="text/javascript">
		var domain = window.location.host.split('.');
		var biller = domain.shift();
		
		document.getElementById('billerId').innerText = biller;
	</script>
  </body>
</html>
{% endhighlight %}

Wildcard Domain HTML page:
{% highlight html linenos %}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Fixed DNS Record Page!</title>
  </head>
  <body>
  <h1>Fixed DNS Record Page!</h1>
    <!-- page content -->
	<p>Fixed DNS RecordB</p>
  </body>
</html>
{% endhighlight %}

## Testing
Wait a few moments for the DNS Records to apply, then attempt hitting some URLs

{% highlight %}
# Goes to static/fixed dns page
https://fixed.mydnsplayground.net/ 

# Goes to wildcard page
https://fake.mydnsplayground.net/
https://biller2.mydnsplayground.net/ 
https://biller1.mydnsplayground.net/
{% endhighlight %}

Success!

## What's next?

Next, we'll look into how we handle custom styling (biller customizing colors, etc or providing CSS overrides). More to come...
