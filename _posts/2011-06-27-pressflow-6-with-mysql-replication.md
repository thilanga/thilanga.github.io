---
layout: post
title: Pressflow 6 with mysql Replication
date: 2011-06-27
comments: true
tags: drupal php pressflow
---

Pressflow6 should support database replication according to the
[ pressflow wiki](https://wiki.fourkitchens.com/display/PF/Using+database+replication+with+Pressflow+5+and+6).
but when I configured master and slave, pressflow instance always pick master. If master was down,
site couldn't load and site redirected to the site-offline error page.

By doing some modifications to `database.inc`, `database.mysqli.inc` and `settings.php` I was able to route DB calls to slave when master is down.

##Scenario

 * If mysql master is inactive, connect to Slave
 * If slave is inactive but Master is active, then connect to Master
 * if Master and Slave both are inactive, Redirect user to Offline page


####database.inc
 * Below change will modify the db_connect call to Master instance by passing 2nd parameter ("master") and by assigning
 `$db_slave_conns[$name]` values to `$active_db`.
 * Validate before slave connection create. If master is active we are not calling Slave by adding `$db_conns[$name]->connect_errno > 0`

```php
function db_set_active($name = 'default') {
  global $db_url, $db_slave_url, $db_type, $active_db, $active_slave_db;
  static $db_conns, $db_slave_conns, $active_name = FALSE;


  if (empty($db_url)) {
    include_once 'includes/install.inc';
    install_goto('install.php');
  }


  if (!isset($db_conns[$name])) {
    // Initiate a new connection, using the named DB URL specified.
    if (is_array($db_url)) {
      $connect_url = array_key_exists($name, $db_url) ? $db_url[$name] : $db_url['default'];
      if (is_array($db_slave_url[$name])) {
        $slave_index = mt_rand(0, count($db_slave_url[$name]) - 1);
        $slave_connect_url = $db_slave_url[$name][$slave_index];
      }
      else {
        $slave_connect_url = $db_slave_url[$name];
      }
    }
    else {
      $connect_url = $db_url;
      if (is_array($db_slave_url)) {
        $slave_index = mt_rand(0, count($db_slave_url) - 1);
        $slave_connect_url = $db_slave_url[$slave_index];
      }
      else {
        $slave_connect_url = $db_slave_url;
      }
    }


    $db_type = substr($connect_url, 0, strpos($connect_url, '://'));
    $handler = "./includes/database.$db_type.inc";


    if (is_file($handler)) {
      include_once $handler;
    }
    else {
      _db_error_page("The database type '". $db_type ."' is unsupported. Please use either 'mysql' or 'mysqli' for MySQL, or 'pgsql' for PostgreSQL databases.");
    }


    $db_conns[$name] = db_connect($connect_url,'master');
    if ($db_conns[$name]->connect_errno > 0 && !empty($slave_connect_url)) {
      $db_slave_conns[$name] = db_connect($slave_connect_url);
    }
  }

  $previous_name = $active_name;
  // Set the active connection.
  $active_name = $name;
  $active_db = $db_conns[$name];
  if (isset($db_slave_conns[$name])) {
    $active_slave_db = $db_slave_conns[$name];
    $active_db = $db_slave_conns[$name];
  }
  else {
    unset($active_slave_db);
  }


  return $previous_name;
}
```

####database.mysqli.inc
 * db_connect function accept a 2nd parameter. default value will be slave
 * wrapped `_db_error_page(mysqli_connect_error())` with slave server validation. this will stop error page redirection if master server is down.
 * wrapped with else block to void calling mysqli_query with uninitiated mysql resource.

```php
function db_connect($url,$server='slave') {
  // Check if MySQLi support is present in PHP
  if (!function_exists('mysqli_init') && !extension_loaded('mysqli')) {
    _db_error_page('Unable to use the MySQLi database because the MySQLi extension for PHP is not installed. Check your php.ini to see how you can enable it.');
  }


  $url = parse_url($url);


  // Decode url-encoded information in the db connection string
  $url['user'] = urldecode($url['user']);
  // Test if database url has a password.
  $url['pass'] = isset($url['pass']) ? urldecode($url['pass']) : '';
  $url['host'] = urldecode($url['host']);
  $url['path'] = urldecode($url['path']);
  if (!isset($url['port'])) {
    $url['port'] = NULL;
  }


  $connection = mysqli_init();
  @mysqli_real_connect($connection, $url['host'], $url['user'], $url['pass'], substr($url['path'], 1), $url['port'], NULL, MYSQLI_CLIENT_FOUND_ROWS);


  if (mysqli_connect_errno() > 0) {
    if($server == 'slave'){
  _db_error_page(mysqli_connect_error());
    }
  }else {


   // Force UTF-8.
   mysqli_query($connection, 'SET NAMES "utf8"');
  }
  return $connection;
}

```

####settings.php

```php
$db_url = array();
$db_url['default'] = 'mysqli://user:password@master-host/database';


$db_slave_url = array();
$db_slave_url['default'] = 'mysqli://user:password@slave-host/database';
```
