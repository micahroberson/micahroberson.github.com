---
layout: post
title: "Dynamic Subdomains and Capistrano"
date: 2012-11-25 10:38
comments: true
categories: [PHP, Capistrano, Deployment]
---

While the majority of my time is spent on RoR projects, we recently took on a project which we opted to develop in PHP due to its relatively light feature-set, for which we determined Rails would be a little overkill. James spearheaded this project and chose CakePHP as the framework for GreekSquare (soon to be www.greeksquare.com). Although I have little knowledge of CakePHP, I have worked on a few PHP projects over the years, so the language isn't entirely foreign, just a little rusty. I stepped in the add dynamic subdomains (think Basecamp style) and to revamp the existing login page (mainly all css changes). 

To start, I added a method to the AppController:

{% codeblock lang:php %}
function getSubdomain() {
	$domain = parse_url($_SERVER['HTTP_HOST']);
	$domain = explode('.',$domain['path']);
	if(count($domain)==3 and !empty($domain[0])) {
		return $domain[0];
	}
	return '';
}
{% endcodeblock %}

I then added a couple of lines to call this method from the UsersController in the login method, assuming the request is a GET. Once the subdomain is determined, the correct image can be looked up and set as the background image for the view. To take this further, we could only allow users to login from their specific subdomain and deny access if they attempt to login from another organization's subdomain. However, as long as we check the subdomain and, if necessary, redirect to the correct subdomain upon successful login, I don't think this is necessary. 

While I was finishing up the changes, I decided we ought to automate the deployment process, since we're getting close to the initial pilot test. Capistrano is the obvious choice for a Rails app, but I wasn't too familiar with PHP deploymment strategies other than Phing. A quick google for Capistrano and CakePHP turned up an article from <a href="http://bakery.cakephp.org/articles/j15e/2010/04/22/deploying-cakephp-with-capistrano">CakePHP's The Bakery</a>. I followed this article pretty closely, only changing a few config options and adding a chmod command to the :finalize_update task. Setting up the deployment with Capistrano was quick and painless and the only real dependencies are Ruby and the Capistrano gem. The current setup doesn't allow for database migrations, so for now we'll still have to update the db manually. 

