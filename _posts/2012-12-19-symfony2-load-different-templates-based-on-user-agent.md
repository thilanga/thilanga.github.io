---
layout: post
title: Symfony2 - Load different templates based on user agent
date: 2012-12-19
comments: true
tags: Howtos Mobile Request Listener Symfony2 Templates Tutorial
---

If you want to load different templates for different user agents while keeping the same backend code base,
you can accomplish this simply in 2 ways.

    1. Responsive Template
    2. Load different templates against user agents.

In this blog post I'm trying to demonstrate the option 2 (load different templates against user agents)

>The trick is to use `RequestFormat` in Symfony

1. Create a Request Listener

    ```bash
        php app/console generate:bundle --namespace=Acme/RequestListenerBundle --format=yml
    ```

    >Note: You don't need Controller or Tests directories

    Past the below code to `AcmeRequestListnerBundle.php`

    ```php
        <?php
            namespace Acme\RequestListenerBundle;
            use Symfony\Component\HttpKernel\HttpKernelInterface;
            use Symfony\Component\HttpKernel\Event\GetResponseEvent;

            class AcmeRequestListenerBundle
            {
                public function onKernelRequest(GetResponseEvent $event)
                {
                    $request = $event->getRequest();

                    if(preg_match('/(alcatel|amoi|android|avantgo|blackberry|benq|cell|cricket|docomo|elaine|htc|
                        iemobile|iphone|ipad|ipaq|ipod|j2me|java|midp|mini|mmp|mobi|motorola|nec-|nokia|palm|panasonic|
                        philips|phone|playbook|sagem|sharp|sie-|silk|smartphone|sony|symbian|t-mobile|telus|up\.browser|
                        up\.link|vodafone|wap|webos|wireless|xda|xoom|zte)/i', $request->headers->get('user-agent')
                            )
                        )
                    {
                        $request->setRequestFormat('mobile', 'text/html');
                    }else {
                        $request->setRequestFormat('html', 'text/html');
                    }
                }

                private function setFormat (\Symfony\Component\HttpFoundation\Request $request, $format='html')
                {
                    $request->setRequestFormat($format, 'text/html');
                }
            }
        ?>
    ```

2. Create a test action in your controller
    - Action Method

        ```php
            /**
            * @Route("/test", name="_demo_test")
            * @Template()
            */

            public function testAction()
            {
                return array(
                    'name' => 'Thilanga Pitigala',
                );
            }
        ```
    - Run and Verify the route

        ```bash
            php app/console router:debug | grep _demo_test
        ```
    - Template

        Create the template for newly created action `Acme/DemoBundle/Resources/views/Demo/test.html.twig`

        ```{% raw %}
                {% extends "AcmeDemoBundle::layout.html.twig" %}
                    {% block title "Hello " %}
                    {% block content %}
                    Hello Desktop !
                {% endblock %}
            {% endraw %}
        ```

3. Create a Service using RequestListener
    Add below code to `config.yml`

    ```
        services: acme.listener.request:
        class: Acme\RequestListenerBundle\AcmeRequestListenerBundle
        tags: - {
                name: kernel.event_listener,
                event: kernel.request,
                method: onKernelRequest
            }
    ```
    - Run

        ```bash
            php app/console container:debug | grep acme.listener.request
        ```
        and That will output the newly created service


3. Test RequestListener service
    Try visiting the web applications from mobile device. It will throw an error complaining about missing `test.mobile.twig` file.

    Create the `test.mobile.twig` file in `Acme/DemoBundle/Resources/views/Demo/test.html.twig` as we created the `test.html.twig` template.
    Now you will be able to see the mobile template from mobile devices while desktop template load for desktop agents.

    By that way we can load different templates for different agents without doing any responsive templates.

Happy Coding..