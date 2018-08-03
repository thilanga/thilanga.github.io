---
layout: post
title: Adding Observers to Magento Events
description: How to customize Magento's behaviour and flow using events and observers.
date: 2014-08-13
comments: true
tags: magento
---
1. To write an Observer you need minimum of 2 files
  1. etc/config.xml (Details about the EVENT_TO_HOOK and the area [frontend|adminhtml])
  2. Model/Observer.php (Your php code goes here)

2. Find the event you want
  1. http://www.nicksays.co.uk/magento-events-cheat-sheet-1-8/

###Sample
#### Remove an action from Mass Action list in admin (Order list page)
##### config.xml
```
    <adminhtml> <!-- Area -->
        <events>
            <adminhtml_block_html_before> <!-- EVENT_TO_HOOK you want to bind with -->
                <observers>
                    <adminhtml_block_html_before> <!-- Your unique tag name -->
                        <class>Acme_Adminhtml_Model_observer</class>  <!-- Your custom Observer -->
                        <method>removeMassAction</method> <!--Method you want to trigger -->
                    </adminhtml_block_html_before>
                </observers>
            </adminhtml_block_html_before>
        </events>
    </adminhtml>
```
##### Observer.php
```
<?php

class Acme_Adminhtml_Model_Observer extends Varien_Object
{
    public function removeMassAction($event) {

        $block = $event->getBlock();
        if ($block instanceof Mage_Adminhtml_Block_Widget_Grid_Massaction) {
            $block->removeItem('hold_order');
            $block->removeItem('unhold_order');

            return $this;
        }
    }
}
```


