# shopify_theme_tweaks
A collection of theme tweaks for Shopify themes

**NOTE:** The content and examples within this repository come with no support or warranty. Use at your own risk.

## Table of Contents
- [Redirect sold out products](#redirect-sold-out-products)
- [Cart Donation Option](#cart-donation-option)
- [Collection Landing Page](#collection-landing-page)
- [PO Box Checkout Shipping Restriction](#po-box-checkout-shipping-restriction)
- [Shipping Address Field Character Limit](#shipping-address-field-character-limit)
- [Basic Free Shipping Upsell Banner](#basic-free-shipping-upsell-banner)
- [Discount cheapest product in the checkout with a discount code](#discount-cheapest-product-in-the-checkout-with-a-discount-code)
- [Add Cart Attribute from URL UTM Param](#add-cart-attribute-from-url-utm-param)

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

## Cart Donation Option
Based on the code from [this guide](https://www.tetchi.ca/shopify-tutorial-adding-a-tip-or-donation-to-the-cart-page) the following is a slightly modified/updated method for adding a donation field to a Shopify store's cart page:

1. Add the following snippet to the `cart-template.liquid`. The location will likely differ slightly from theme to theme.

For the [Debut](https://help.shopify.com/en/manual/using-themes/themes-by-shopify/debut) theme:
```
<p class="cart-donation">
  <label for="tip-amount">Tip amount (optional)</label>
  <input id="tip-amount" type="text" name="Tip amount" placeholder="Optional driver tip">
  <input type="button" name="addtip" class="btn btn--secondary small--hide" value="Add Tip">
</p>
```
For the [Venture](https://help.shopify.com/en/manual/using-themes/themes-by-shopify/venture) theme:
```
<p class="cart-donation">
	<label for="tip-amount">Tip amount (optional)</label>
	<input id="tip-amount" type="text" name="Tip amount" placeholder="Optional tip" style="background-color: #dcdcdc;">
	<input type="button" name="addtip" class="btn btn--secondary" value="Add Tip" style="margin-top: 10px;">
</p>
```

2. Add the following code to a new snippet called `donation-snippet.liquid` and replace `TIP_PRODUCT_VARIANT_ID` with the variant ID of your donation product (configured with a value of $0.01). 
```
<script>
$(document).ready(function () {
	$('input[name="addtip"]').click(function(e) {
		e.preventDefault();
		var amount = $('#tip-amount').val();

		amount = amount * 100;
		amount = parseInt(amount);

		function addDonation(){
			var params = {
				type: 'POST',
				url: '/cart/add.js',
				data: 'quantity='+amount+'&id=TIP_PRODUCT_VARIANT_ID',
				dataType: 'json',
				success: function(line_item) { 
					console.log("success!");
					if (document.location.pathname == "/cart"){
						document.location.href = '/cart';
					}
				},
				error: function() {
					console.log("fail");
				}
			};
			$.ajax(params);
		}
		addDonation();
	});
});
</script>
```

3. Include the donation snippet in the `cart-template.liquid` file by adding the following line just before the end div tag:
```
{% include 'donation-snippet' %}
```

4. If not already included in `theme.liquid`, include jQuery just before the end of the head tag:
```
{{ '//ajax.googleapis.com/ajax/libs/jquery/1.9.0/jquery.js' | script_tag }}
```

## Collection Landing Page

Based on [this guide](https://community.shopify.com/c/Shopify-Design/Collection-Feature-a-subset-of-collections-on-a-page/m-p/614952) from the Shopify forums, the following example is a simplified and modified version that was written to work with the allure version of the [Prestige](https://themes.shopify.com/themes/prestige/styles/allure) theme.

For installation; follow the linked guide and simply use the following code as the contents for the cloned `page.list-collections.liquid` file rather than the code in that guide: [page.list-collections.liquid](https://github.com/HaywardPeirce/shopify_theme_tweaks/blob/master/files/collection_landing_page/page.list-collections.liquid)

## PO Box Checkout Shipping Restriction

This script blocks a customer from continuing past the first page of the Shopify checkout if they are attempting to ship their order to a PO box.

**Please note:** this solution only works for merchant on Shopify Plus as it requires access to `checkout.liquid`

With the underlying code from wherever [this guy](https://stackoverflow.com/q/60370581) also got his code, I updated the regex to support the following variations of addresses that include PO box wording:

* `Post Office` Box
* `post office` box
* `Post office` box
* `Post Box`
* `Post box`
* `post box`
* `post Box`
* `P.O.` Box
* `P.O.` box
* `PO Box`
* `PO box`
* `PO.` Box
* `PO.` box
* `P.O.`
* `PO.`

1. Create a new snippet called `po-box-no-shipping.liquid` with the following contents:
```
  <script>
    (function($) {
      $(document).on('ready page:load page:change', function() {
        var regex = /^.*p(\.O\.|o box|ost office|ost box)/i;
        var fieldErrorClass = 'field--error';
        var fieldErrorMessageSelector = '.field__message--error';
        var errorText = '{{ 'plus.PO_error_message' | t }}';
        var $inputs = $("[data-step] [name='checkout[shipping_address][address1]'], [data-step] [name='checkout[shipping_address][address2]']");
        
        var regexCheckFn = function(elem) {
          var $current = $(elem);
          var $parent = $current.closest('.field__input-wrapper');
          var $field = $current.closest('.field');
          if (regex.test($current.val())) {
            if (!$field.hasClass(fieldErrorClass)) {
              $field.addClass(fieldErrorClass);
            }
            if ($field.find(fieldErrorMessageSelector).length < 1) {
              $parent.after("<p class='field__message field__message--error'>"+ errorText +"</p>");
            }
            return false;
           } else {
            if ($field.hasClass(fieldErrorClass)) {
              $field.removeClass(fieldErrorClass);
            }
            if ($field.find(fieldErrorMessageSelector).length > 0) {
              $field.find(fieldErrorMessageSelector).remove();
            }
            return true;
          }
        };
        
        // Call regex check on form submit
        $(document).on('submit', '[data-step] form', function() {
          // default to true and will be set to false if there is an error to prevent form submission
          var isValid = true;
          $inputs.each(function() {
            isValid = isValid && regexCheckFn($(this));
          });
          return isValid;
        });
        
        // Call regex check on blur
        $inputs.blur(function() {
          regexCheckFn($(this));
        });
        
      });
    })(Checkout.$);
  </script> 
```
2. Inlcude the snippet in `checkout.liquid`, just after the opening `<body>` tag using:

```
{% include 'po-box-no-shipping' %}
```

3. [Add in the wording](https://help.shopify.com/en/manual/using-themes/change-the-layout/change-wording) for the error message displayed to the customer when they try and ship to a PO box.

The field will be under the "Plus" tab, and is called "Po error message". E.g. `This store is not able to ship orders to a PO BOX`

**Please note:** If the "Po error message" language field doesn't show up in your theme's language settings, then you need to go back into the theme code and add the variable (or the Plus translations section as a whole) to each of the applicable langauges your theme uses (e.g. `en.default.json`). 

```
"plus": {
    "PO_error_message": "This store is not able to ship orders to a PO BOX"
  },
```

## Shipping Address Field Character Limit

This script blocks a customer from continuing past the first page of the Shopify checkout if either of the two main address field are longer than 35 characters.

**Please note:** this solution only works for merchant on Shopify Plus as it requires access to `checkout.liquid`

This code was built with the underlying code from wherever [this guy](https://stackoverflow.com/q/60370581) also got his code, and with further customizations for my above PO box restriction example:

1. Create a new snippet called `ship-field-max-length.liquid` with the following contents:
```
  <script>
    (function($) {
		$(document).on('ready page:load page:change', function() {
			var fieldErrorClass = 'field--error';
			var fieldErrorMessageSelector = '.field__message--error';
			var errorText = '{{ 'plus.Shipping_field_max_char_length_error_message' | t }}';
			var $inputs = $("[data-step] [name='checkout[shipping_address][address1]'], [data-step] [name='checkout[shipping_address][address2]']");

			var regexCheckFn = function(elem) {
				var $current = $(elem);
				var $parent = $current.closest('.field__input-wrapper');
				var $field = $current.closest('.field');
				
				//   test the regex string against the value of the current element
				// If the regex string matches
				if ($current.val().length >= 36) {
					if (!$field.hasClass(fieldErrorClass)) {
						$field.addClass(fieldErrorClass);
					}
					if ($field.find(fieldErrorMessageSelector).length < 1) {
						$parent.after("<p class='field__message field__message--error'>"+ errorText +"</p>");
					}
					return false;
				} else {
					if ($field.hasClass(fieldErrorClass)) {
						$field.removeClass(fieldErrorClass);
					}
					if ($field.find(fieldErrorMessageSelector).length > 0) {
						$field.find(fieldErrorMessageSelector).remove();
					}
					return true;
				}
			};

			// Call regex check on form submit
			$(document).on('submit', '[data-step] form', function() {
				// default to true and will be set to false if there is an error to prevent form submission
				var isValid = true;
				$inputs.each(function() {
					isValid = isValid && regexCheckFn($(this));
				});
				return isValid;
			});

			// Call regex check on blur
			$inputs.blur(function() {
				regexCheckFn($(this));
			});

		});
    })(Checkout.$);
</script> 
```
2. Inlcude the snippet in `checkout.liquid`, just after the opening `<body>` tag using:

```
{% include 'ship-field-max-length' %}
```

3. [Add in the wording](https://help.shopify.com/en/manual/using-themes/change-the-layout/change-wording) for the error message displayed to the customer when they try and ship to a PO box.

The field will be under the "Plus" tab, and is called "Shipping field max char length error message". E.g. `Address field cannot be more than 35 characters`

**Please note:** If the "Shipping field max char length error message" language field doesn't show up in your theme's language settings, then you need to go back into the theme code and add the variable (or the Plus translations section as a whole) to each of the applicable langauges your theme uses (e.g. `en.default.json`). 

```
"plus": {
    "Shipping_field_max_char_length_error_message": "Address field cannot be more than 35 characters"
  },
```


## Basic Free Shipping Upsell Banner

The following theme tweak adds a basic "You are $X away from free shipping" upsell banner to a Shopify theme. 

**Please note:** This example doesn't currenlty support automatically updating the banner messaging when the cart is updated (without a page refresh) and the theme of the banner has been built for the [Warehouse](https://themes.shopify.com/themes/warehouse/styles/metal) theme.

This example also adds in banner configuration options to the theme sections settings which let merchants control the visibility of the banner, whether it just shows up on the cart page or not, free shipping threshold, and the colour of the banner and banner text.

1. Create a new section called `upsell-announcement-bar.liquid` with the following contents:

```
{%- if section.settings.show_announcement -%}
    <section data-section-id="{{ section.id }}">
    
    {% assign cartCheck = false %}

    {%- if section.settings.cart_page_only == true and template == 'cart' -%}
        {% assign cartCheck = true %}
    {%- endif -%}

    {%- if cartCheck == true or section.settings.cart_page_only == false -%}

        {%- if section.settings.free_shipping_amount != blank -%}

            {% assign free_shipping_amount = section.settings.free_shipping_amount | plus: 0 %}

            {% assign cart_total = cart.total_price | divided_by: 100 %}
            <div class="announcement-bar free-shipping-announcement-bar">
                <div class="container">
                    <div class="announcement-bar__inner">
                        <p class="announcement-bar__content announcement-bar__content--{{section.settings.text_location}}" >
                            {% if cart_total < free_shipping_amount %}
                                {% assign shipping_value_left = free_shipping_amount | minus: cart_total %}
                                <span id="free_shipping_announcement_remaining">You are ${{shipping_value_left}} away from free shipping</span>
                            {% else %}
                                <span id="free_shipping_announcement_remaining">Your order qualifies for free shipping!</span>
                            {%- endif -%}
                        </p>
                    </div>
                </div>
            </div>
        {%- endif -%}
    {%- endif -%}
    </section>
    <style>
        .free-shipping-announcement-bar {
            background: {{ section.settings.background }};
            color: {{ section.settings.text_color }};
        }
    </style>
{%- endif -%}

<script>
</script>

{% schema %}
{
  "name": "Free Shipping Banner",
  "settings": [
    {
      "type": "checkbox",
      "id": "show_announcement",
      "label": "Show cart upsell announcement",
      "default": false
    },
    {
      "type": "checkbox",
      "id": "cart_page_only",
      "label": "Cart page only",
      "default": true
    },
    {
      "type": "text",
      "id": "free_shipping_amount",
      "label": "Free Shipping Amount",
      "default": "50"
    },
    {
      "type": "color",
      "id": "background",
      "label": "Background",
      "default": "#1e2d7d"
    },
    {
      "type": "color",
      "id": "text_color",
      "label": "Text",
      "default": "#ffffff"
    },
    {
        "type": "select",
        "id": "text_location",
        "options": [
            { "value": "left", "label": "Left"},
            { "value": "center", "label": "Center"}
        ],
        "label": "Text alignment",
        "default": "center"
    }
  ]
}
{% endschema %}
```

2. Include the snippet in `theme.liquid` by adding the following line just above the `{{ content_for_layout }}` line in `<main>:

```
{% section 'upsell-announcement-bar' %}
```

## Discount cheapest product in the checkout with a discount code

Shopify discount codes an sometimes be overly limiting in terms of the functionality they are able to provide. Thankfully, merchants can setup 0% discount codes, and then use Shopify Scripts to conduct additional logic on the line items in an checkout in order to provide a discount.

The following script is a modified version of the [existing example script](https://help.shopify.com/en/manual/apps/apps-by-shopify/script-editor/examples#percentage-off-least-expensive-items) showing providing a discount on the cheapest item in an order. In this example the 15% discount is only applied when the customer uses the 0% discount code `CHEAPPRODUCT`: 

```
DISCOUNT_AMOUNT = 15

if Input.cart.discount_code
    if (Input.cart.discount_code.code == "CHEAPPRODUCT")
        if (Input.cart.line_items.size > 1)
            line_item = Input.cart.line_items.sort_by { |line_item| line_item.variant.price }.first
            if line_item.quantity > 1
            line_item = line_item.split(take: 1)
            Input.cart.line_items << line_item
            end
            line_item.change_line_price(line_item.line_price * (1.0 - (DISCOUNT_AMOUNT / 100.0)), message: "#{DISCOUNT_AMOUNT}% off!")
        end
    end
end

Output.cart = Input.cart
```

## Add Cart Attribute from URL UTM Param

The following example shows up to setup a simple script to collect a UTM parameter (often from an external link to shopify) and apply it as a [Shopify cart attribute](https://shopify.dev/docs/themes/liquid/reference/objects/cart#cart-attributes).

1. If not already included in `theme.liquid`, include jQuery within the head:
```
{{ '//ajax.googleapis.com/ajax/libs/jquery/1.9.0/jquery.js' | script_tag }}
```

2. Add the following script to the end of the head tag. Be sure to include the value of `paramKey` with the UTM parameter key you are looking to have the script pick up on (e.g. from the URL `https://example.com/?utm_campaign=email_promo` you would set the value of `parmaKey` to `utm_campaign`):
```
<script>
    const queryString = window.location.search;
    const urlParams = new URLSearchParams(queryString);
    const paramKey = "REPLACE_ME";
    const paramValue = urlParams.get(paramKey);
    
    if (paramValue){
      jQuery.post('/cart/update.js', "attributes[" + paramKey + "]="+ paramValue + "");
    }
</script>
  ```
  
This will add a new cart attibute based on the key/value of the UTM param pulled from the URL.
