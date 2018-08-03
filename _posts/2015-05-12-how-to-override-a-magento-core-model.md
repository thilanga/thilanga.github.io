---
layout: post
title: "How to override a Magento core model"
date: 2015-05-12
comments: true
tags: magento
description: Modify/ alter data in magento within the default workflow by override a core model
---

If you want to modify/ alter some of data in magento with in the default workflow you can achieve this in 2 ways.

1. Use an Event
2. Override the core model.

Then the big question. What is the best way from above options. Without an argument Event based override is the best option. Because this method is decoupled from the core and will give us a flexibility around upgrading.

We can't use events all the time. Because Magento got limited events. So in that case   we will have to override the the core method to do our job.

##Scenario
Stop custom temporary options get saved against products when adding to cart
***

`app/code/core/Mage/Checkout/Model/Cart.php` has `addProduct` which is responsible for saving products to the cart. 

If you observer the method you can see that there is only one event we can use `checkout_cart_product_add_after`, Which is not gong to help in our case as we need to modify `$requestInfo` before it get pass in to the `$this->_getProductRequest($requestInfo)`. 

So as I explained above let's override the `addProduct`. By doing this we can have more flexibility around the code.

##### Acme/Test/etc/config.xml
```
    <global> <!-- Magento Scope -->
        <models> 
            <checkout> <!-- Core module we want to override -->
                <rewrite>
                    <cart>Acme_Test_Model_Cart</cart> <!-- Module file we want to override -->
                </rewrite>
            </checkout>

        </models>
    </global>
```

##### Acme/Test/Model/Cart.php
```
<?php

class Acme_Test_Model_Cart extends Mage_Checkout_Model_Cart
{
    /**
     * Add product to shopping cart (quote)
     *
     * @param   int|Mage_Catalog_Model_Product $productInfo
     * @param   mixed $requestInfo
     * @return  Mage_Checkout_Model_Cart
     */
    public function addProduct($productInfo, $requestInfo=null)
    {
        if (isset($requestInfo['custom_label'])) {
            unset($requestInfo['custom_label']);
        }
 
        //Keeping the golden rule of Magento in the head
        //we passed our modified $requestInfo to the parent method.
        //Keep the custom method foot print as small as you can

        parent::addProduct($productInfo, $requestInfo);

    }
}
```

