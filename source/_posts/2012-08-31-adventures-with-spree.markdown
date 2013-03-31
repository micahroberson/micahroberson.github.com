---
layout: post
title: "Adventures with Spree"
date: 2012-08-31 13:37
comments: true
categories: 
---

Yesterday I began researching and really getting into building a new website for a client which should have a look and feel very similar to that of a classic ecommerce site. Although they don't intend to offer purchases online at this time, they would like a feature similar to a shopping cart where users can assembly custom products from the many components offered on the site. For this reason, I opted to go with an ecommerce/cms platform. At this point, I think Spree has everything I need, which includes the ability to host on Heroku, and upload bulk images/products to S3 via the Spree extension spree_import_products. I've documented the steps I've taken to get everything playing nicely together and how I'm theming the site as well.

First prereq was bulk image/product upload. spree_import_products accepts a url for each product image, so my plan to populate a production database is to upload all of the original images to an S3 bucket and populate an Excel doc with those urls. Then I can upload the csv via the Spree extension and the images will be pulled from the staging S3 bucket, get processed and stored in the production S3 bucket. 

To get spree_imports_products working properly with the latest version of rails and spree, I forked the repo and modified a few dependencies in the GemSpec and also the method with which Spree::Image attributes were being set for non-whitelisted attrs.

Working branch can be found here: 
https://github.com/micahroberson/spree-import-products/tree/1_2_X

In order to get multi domain/multi store functionality, I forked the spree_multi_domain extension and modified the GemSpec for Spree 1.2.0.
https://github.com/micahroberson/spree-multi-domain

The multi domain extension adds a new table, Stores, which taxonomies and products are both associated to.This means I need to make sure all taxonomy queries are referencing the current store. (The extension provides a helper method, current_store for the store_id) To do this, I first had to modify the spree_core gem to add 'store_id' to attr_accessible in the taxonomy model. And subsequently, get_taxonomies in products_helper.rb to:

{% codeblock %}
def get_taxonomies
  @taxonomies ||= Spree::Taxonomy.includes(:root => :children).find_all_by_store_id(current_store)
end
{% endcodeblock %}

More to come this weekend!