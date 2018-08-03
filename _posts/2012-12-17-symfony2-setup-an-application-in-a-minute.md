---
layout: post
title: Symfony2 - Setup an Application in a minute
date: 2012-12-17
comments: true
tags: Framework Howtos Symfony2 Tutorial
---

1. Download

    Download the Symfony Standard Edition from [symfony.com](http://symfony.com/download "symfony.com")
    >At the moment of writing this post latest stable version is 2.1

    ```bash
        cd /home/user/Downloads/
        wget --content-disposition 'http://symfony.com/download?v=Symfony_Standard_Vendors_2.1.4.tgz'
        tar -zxvf Symfony_Standard_Vendors_2.1.4.tgz
        mv Symfony /var/www/symfony
    ```
    Then Point `/var/www/symfony/web` directory to the your web server

2. Configuration

    Now We are going to use `app.php` as the main index file instead of playing with `app_dev.php` within the development environment.

    ```bash
        vim web/app.php
    ```
    Replace your `app.php` with the below code
        
    ```php
        // Make group permissions stick in `cache/log` dirs
        umask(0002);

        use Symfony\Component\ClassLoader\ApcClassLoader;
        use Symfony\Component\HttpFoundation\Request;

        $loader = require_once __DIR__.'/../app/bootstrap.php.cache';

        require_once __DIR__.'/../app/AppKernel.php';
        $env = 'prod';
        if (isset($_SERVER['HTTP_HOST']))
        {
            if (preg_match('/(localhost|local\.)/', $_SERVER['HTTP_HOST']))
            {
                $env = 'dev';
            }
        }

        $kernel = new AppKernel($env, $env == 'dev');
        $kernel->loadClassCache();

        $request = Request::createFromGlobals();

        $response = $kernel->handle($request);
        $response->send();

        $kernel->terminate($request, $response);
    ```

    Load your Symfony site by typing `http://localhost/symfony` in your web browser.
    Now you can access Symfony Development environment without visiting to app_dev.php.