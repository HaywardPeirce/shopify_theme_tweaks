# shopify_theme_tweaks
A collection of theme tweaks for Shopify themes

## Redirect sold out products

There are cases where merchants have a large catalogue of products and stock many that are one-off items that once sold out won't be available again. Rather than having to manaully hide these products they would like them to automatically redirects customers that might still have a link in the wild to a collection of similar products. 

Native redirects aren't an option as they aren't able to redirect from pages in a store that exist (which this product will still).

One option would be to setup an app like [EasyRedirects](https://apps.shopify.com/easylockdown) (which is able to redirect from existing pages) to a desired collection, but this would be just as manual as simply hidding the product and setting up a native redirect.

A more extenisble solution would involved tweaks to the theme to handle this redirect automatically:

1. Edit the `product-template.liquid` file, or make changes to a cloned version of the original, to include the following:  

```
<script>
  var redirectTagString = null;
</script>
{% assign productTags = product.tags | join:',' %}
{% if product.available == false and productTags contains "NoStockRedirect"%}
	{% for tag in product.tags %}
		{% if tag contains "NoStockRedirect" %}
			 <script>
               redirectTagString = "{{ tag }}";
               console.log("redirectTagString");
			</script>
		{% endif %}
	{% endfor %}
	<script>
      if (redirectTagString){
        var redirectType = redirectTagString.split(":")[1];
        var redirectHandle = redirectTagString.split(":")[2];
      	location.replace("https://" + window.location.hostname + "/" + redirectType + "/" + redirectHandle);
      }
	</script>
{% endif %}
```

2. Configure the desired products to include a tag in the following format:
To redirect to different product: `NoStockRedirect:products:product-handle`
To redirect to a collection: `NoStockRedirect:collections:collection-handle`

3. Configure the desired products to use the cloned template (if not making the changes from step 1 to the original)

