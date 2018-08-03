---
layout: post
title: How to override a Controller
description: When you can't find an event or a module to customize Magento your only savior is controller. Let's have a look at how we can bend Magento by overriding controllers.
date: 2014-09-08
comments: true
tags: magento
---

Overriding controllers is bit different to Model/Blocks/Helpers as Controllers do not get loaded though the autoloader.

##Scenario
Send a copy of the mail when a user/visitor submits the contact form.
***

In Magento contact form is handled by `Mage_Contacts`. If you take a look at the `Mage_Contacts_IndexController`
you can find the `postAction` method which is responsible for the contact form submission.

In this example we will be overriding the `postAction` in `Mage_Contacts_IndexController` with `Acme_ContactExtended_IndexController`

##### Acme/ContactExtended/etc/config.xml
```
    <frontend> <!-- Magento Area -->
        <routers>
            <contacts> <!-- Core module we want to override -->
                <args>
                    <modules>
                        <Acme_ContactExtended before="Mage_Contacts">Acme_ContactExtended</Acme_ContactExtended> <!-- Our custom Controller -->
                    </modules>
                </args>
            </contacts>
        </routers>
    </frontend>
```
##### Acme/ContactExtended/controllers/IndexController.php

> I haven't added system.xml in this tutorial and this code might not work if you copy-paste in to you file :). I will explain `system.xml` in a different article in the future.

```
<?php

require_once Mage::getModuleDir('controllers', 'Mage_Contacts') . DS . 'IndexController.php';

class Acme_ContactExtended_IndexController extends Mage_Contacts_IndexController
{
    const XML_PATH_SENDER_NOTIFICATION_ENABLED = 'contacts/sender_notification/enabled';
    const XML_PATH_SENDER_NOTIFICATION_EMAIL_TEMPLATE = 'contacts/sender_notification/email_template';

    public function postAction()
    {

        $post = $this->getRequest()->getPost();
        if ($post) {
            $translate = Mage::getSingleton('core/translate');
            /* @var $translate Mage_Core_Model_Translate */
            $translate->setTranslateInline(false);
            try {
                $postObject = new Varien_Object();
                $postObject->setData($post);

                $error = false;

                if (!Zend_Validate::is(trim($post['first-name']), 'NotEmpty')) {
                    $error = true;
                }

                if (!Zend_Validate::is(trim($post['last-name']), 'NotEmpty')) {
                    $error = true;
                }


                if (!Zend_Validate::is(trim($post['email']), 'EmailAddress')) {
                    $error = true;
                }

                if (!Zend_Validate::is(trim($post['comment']), 'NotEmpty')) {
                    $error = true;
                }


                if (Zend_Validate::is(trim($post['hideit']), 'NotEmpty')) {
                    $error = true;
                }

                if ($error) {
                    throw new Exception();
                }

                $mailTemplate = Mage::getModel('core/email_template');
                /* @var $mailTemplate Mage_Core_Model_Email_Template */
                $mailTemplate->setDesignConfig(array('area' => 'frontend'))
                    ->setReplyTo($post['email'])
                    ->sendTransactional(
                        Mage::getStoreConfig(self::XML_PATH_EMAIL_TEMPLATE),
                        Mage::getStoreConfig(self::XML_PATH_EMAIL_SENDER),
                        Mage::getStoreConfig(self::XML_PATH_EMAIL_RECIPIENT),
                        null,
                        array('data' => $postObject)
                    );

                if (!$mailTemplate->getSentSuccess()) {
                    throw new Exception();
                }


                /* send sender notification */
                if (Mage::getStoreConfigFlag(self::XML_PATH_SENDER_NOTIFICATION_ENABLED)) {
                    $customerMailTemplate = Mage::getModel('core/email_template');
                    /* @var $mailTemplate Mage_Core_Model_Email_Template */
                    $customerMailTemplate->setDesignConfig(array('area' => 'frontend'))
                        ->setReplyTo(self::XML_PATH_EMAIL_RECIPIENT)
                        ->sendTransactional(
                            Mage::getStoreConfig(self::XML_PATH_SENDER_NOTIFICATION_EMAIL_TEMPLATE),
                            Mage::getStoreConfig(self::XML_PATH_EMAIL_SENDER),
                            $post['email'],
                            null,
                            array('data' => $postObject)
                        );

                    if (!$customerMailTemplate->getSentSuccess()) {
                        throw new Exception();
                    }
                }
                /*  */

                $translate->setTranslateInline(true);

                Mage::getSingleton('customer/session')->addSuccess(Mage::helper('contacts')->__('Your inquiry was submitted and will be responded to as soon as possible. Thank you for contacting us.'));
                $this->_redirect('*/*/');

                return;
            } catch (Exception $e) {
                $translate->setTranslateInline(true);

                Mage::getSingleton('customer/session')->addError(Mage::helper('contacts')->__('Unable to submit your request. Please, try again later'));
                $this->_redirect('*/*/');
                return;
            }

        } else {
            $this->_redirect('*/*/');
        }
    }

}

```

When overriding core files always the best practice is to minimize the foot print of custom codes.
We shouldn't be copying the whole function from the core file. Minimum custom foot prints will help us to upgrade Magento easily.

In `Acme/ContactExtended/controllers/IndexController.php` you will see that I haven't followed the best practice I mentioned above.
This is because:

1. We have custom fields in the contact form (`ex: First Name, Last Name etc..`)
2. We need custom validations
3. `postAction` use $_POST to get posted values in the core

### Summary

By following the above method, Magento will process our Controller before Mage_Contact. So you are the master of Contact form now.
Happy coding!
