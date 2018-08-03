---
layout: post
title: Magento - How to switch Base URLs without modifying the Database
date: 2013-01-19
comments: true
tags: eCommerce magento php
---

As Magento keeps its base URL in database, there is no easy way of accessing the same database from two different URLs
(dev/ staging) or moving the same database to staging environment from development without updating  the `core_config_data` table.


After fiddling around with Magento core, I found that `getBaseUrl()` is responsible of retrieving base URL.
So If we could override `getBaseUrl()` method to get the base URL from dev or staging environments without depending on the database,
then we could keep database records with live settings.

> Below code will explain how I achieved this task and the code itself is explainable.

- app/etc/modules/My_Configurator.xml

```xml
    <?xml version="1.0"?>
    <config>
        <modules>
          <my_configurator>
            <active>true</active>
            <codePool>local</codePool>
          </my_configurator>
        </modules>
    </config>
```
- app/code/local/My/Configurator/etc/config.xml

```xml
  <?xml version="1.0"?>
  <config>
    <global>
      <models>
        <core>
          <rewrite>
            <store>My_Configurator_Model_Store</store>
            </rewrite>
          </core>
        </models>
      </global>
    </config>
```
- app/code/local/My/Configurator/Model/Store.php

```php
    <?php
        class My_Configurator_Model_Store extends Mage_Core_Model_Store
        {
            public function getBaseUrl($type=self::URL_TYPE_LINK, $secure=null)
            {
                $url = parent::getBaseUrl($type, $secure);
                //domain from the DB
                $host = parse_url($url, PHP_URL_HOST);
                return str_replace('://'.$host.'/', '://'.$_SERVER['HTTP_HOST'].'/', $url);
            }
        }
```
>Note: Make sure you disable this extension from production server as we want to run `getBaseUrl()` method from the Core.

Happy Coding..

###Comments

>jennychalek
>>This is making me pull my hair out. I used this exact code and even though my IDE recognizes that the class
      has been successfully overriden, magento itself does not seem to know or care - it never calls the override
      class, just the original. Is there some sort of config setting I&#39;m missing? Any other ideas what could
      cause this? Grasping at straws at this point.

>Thilanga Pitigala
>>@jennychalek Did you enable the extension? Can you see it under Admin-&gt;System-&gt;Configuration-&gt;Advanced?
