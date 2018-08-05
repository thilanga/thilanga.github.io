---
layout: post
title: "Dependency Injection with PHP"
date: 2018-08-02
comments: true
tags: dependency-injection clean-code php
description: Clean and scalable Dependency Injections with PHP 
---
```
Dependency Injection. Everyone has heard about it, but do they really know it?
```

This is a common theme I get from most candidates I have interviewed in the past couple of years when looking for Software Engineers to work on our SaaS based BI platform [EngineRoom](https://digital360.com.au/technology){:target="_blank"} at [Digital360](https://digital360.com.au/){:target="_blank"}.
But when I dig a little deeper into solutions they have built in the past, it has shown that most haven’t followed proper implementation techniques.

In this blog post, I'll try to explain my take on dependency injection, and in particular how to implement a scalable solution.

## What is a dependency?
According to [Wikipedia](https://en.wikipedia.org/wiki/Dependency_injection){:target="_blank"}.
In software engineering, dependency injection is a technique whereby one object (or static method) supplies the dependencies of another object.
A dependency is an object that can be used (a service). An injection is the passing of a dependency to a dependent object (a client) that would use it.

In this context an object (A) that expects another object's (B) help to complete a job that object (A) is responsible for.
To explain the theory, I'll build a simple music player that can be used against multiple music services like Google Music, Spotify, etc.

## Objective
Music player should be able to play music from user's preferred music service.

> ProTip: How to find which dependencies to inject?

>> If we look at the above requirement, the music player is more on the parental side, it requires any number of music services to function. Therefore it makes sense to inject the music service into the music player.

### Approaches
#### Common Approach

> This is a common approach that doesn't use Dependency Injection

```php
class MusicPlayer
{
    private $musicService;

    public function __construct(GoogleMusic $googleMusic)
    {
        $this->musicService = $googleMusic;
    }
}
```

If you look at the code above, someone might say:
> "yeah that's dependency injection."

But, due to that method injecting the object into the `MusicPlayer` class using the constructor, this is considered Constructor Injection.

This is considered bad practice as it permanently binds the `GoogleMusic` class to the `MusicPlayer` class.

Another example is creating the `GoogleMusic` object inside the `MusicPlayer` class.

```php
class MusicPlayer
{
    private $musicService;

    public function __construct()
    {
        $this->musicService = new GoogleMusic();
    }
}
```
Both code snippets above are doing the same mistake. They are permanently binding the `GoogleMusic` service to the `MusicPlayer` class.
Now the `MusicPlayer` class has become unusable for anything other than the `GoogleMusic` service, so other services like the `SpotifyMusic` service cannot be used.

Let's add some support for the `SpotifyMusic` service. 

```php
class MusicPlayer
{
    private $musicService;

    public function __construct(GoogleMusic $googleMusic, SpotifyMusic $spotifyMusic)
    {
    
        $this->musicService = null;
        
        if($user->getMusicService()->getName() == 'GOOGLE_MUSIC') {
            $this->musicService = $googleMusic;
        }
        
        if($user->getMusicService()->getName() == 'SPOTIFY_MUSIC') {
            $this->musicService = $spotifyMusic;
        }
    }
}
```

In this scenario, the music player can work with Google Music and Spotify. But the music services are still coupled tightly with the player.

##### What issues are we are going to face if we couple our code tightly to the Music player?
1. It's not scalable, we can't easily add another music service such as `AppleMusic`.
With the current implementation, there are two ways to support this by editing both the `MusicPlayer` class and the `AppleMusic` class: 

    - Inject AppleMusic as an argument
    - Initiate AppleMusic object within the constructor

#### A better approach

> Using Dependency Injection.

First. Let's look at the problem we are trying to solve once more. 

> We are building a system that can play music based on a user's prefered service.

If you read the problem again you can see that we are going to have multiple music services.
To use multiple services with dependency injections we need to build services against contracts. A contract in OOP is an `Interface`.

> ProTip: When to create an Interface?

>> I have seen some developers go overboard with writing code against interfaces for everything and make the code so dynamic without having the real need. If you are going to have more than one similar type of class (Google Music, Spotify, Apple Music, etc.), then create one interface.

As we know that we are going to support more than one music service, we will be creating an `IMusicService` interface and an `ITrack` interface.

We don't need an interface for the music player because there is only going to be one player.

> But why an Interface for the Track?

>> We don't know how other services will return track listings or other information, therefore to be on the safe side, we can have a separate ITrack implementation to handle separate responses.

```php
// MusicService Interface
namespace Sample\Contracts;

interface IMusicService
{
    public function getName(): string;

    public function fetchPlayList();

    public function fetchTrack();
}
```

```php
// Music Track Interface
namespace Sample\Contracts;

interface ITrack
{
    public function getName(): string;

    public function getArtist(): string;

    public function getTrackPath(): string;

    public function play();
}
```

Now let's write the music player class.

```php
namespace Sample;

use Exception;
use Sample\Contracts\IMusicService;
use Sample\Contracts\ITrack;
use Sample\Services\NoMusicService;

class MusicPlayer
{
    private $musicService;

    /**
     * MusicPlayer constructor.
     *
     * @param \Sample\Contracts\IMusicService $musicService
     *
     * @throws \Exception
     */
    public function __construct(IMusicService $musicService)
    {
        if ($musicService instanceof NoMusicService) {
            throw new Exception('No Music service been configured for this user');
        }

        $this->musicService = $musicService;
    }

    public function play()
    {
        /** @var ITrack $track */
        $track = $this->musicService->fetchTrack();
        $track->play();
    }
}
```

If you look at the MusicPlayer constructor, it takes one argument. It's the Music service Interface.

It's time to let the music player play some tracks from the injected services. 

```php
require __DIR__ . '/vendor/autoload.php';

use Sample\Contracts\IMusicService;
use Sample\MusicPlayer;
use Sample\Services\Google\GoogleMusic;
use Sample\Services\NoMusicService;
use Sample\Services\Spotify\SpotifyMusic;

switch (strtolower($_GET['user'])) {
    case 'alice':
        $musicService = new GoogleMusic();
    case 'bob':
        $musicService = new SpotifyMusic();
    default:
        $musicService = new NoMusicService();
}

try {
    $musicPlayer = new MusicPlayer($container->make($musicService));
    $musicPlayer->play();
} catch (Exception $e) {
    die('Can\t find a music service.');
}
```

#### The ideal approach.
We can use an IoC container (Inversion of Control) to handle the creation of these objects and not worry about it again until there is a new service that needs to be supported.

As IoC is its own topic I won't get too deep on that subject here. For the moment think about it as a dynamic object injection based on required dependencies by the `MusicPlayer` class (although it's much more than that).

> In the example that follows, I'll be using Laravel's IoC container.

Once the IoC installation is done using composer, we can create an IoC instance and let it handle injection to the `MusicPlayer` instance based on the logic we want. 

In our case loading different music service objects based on different user preferences.

```php
require __DIR__ . '/vendor/autoload.php';

use Illuminate\Container\Container;
use Sample\Contracts\IMusicService;
use Sample\MusicPlayer;
use Sample\Services\Google\GoogleMusic;
use Sample\Services\NoMusicService;
use Sample\Services\Spotify\SpotifyMusic;

// Create new IoC Container instance
$container = Container::getInstance();

$container->singleton(IMusicService::class, function ($app) {
    switch (strtolower($_GET['user'])) {
        case 'alice':
            return new GoogleMusic();
        case 'bob':
            return new SpotifyMusic();
        default:
            return new NoMusicService();
    }
});

try {
    $musicPlayer = new MusicPlayer($container->make(IMusicService::class));
    $musicPlayer->play();
} catch (Exception $e) {
    die('Can\t find a music service.');
}
```

## Conclusion

> Test the code!

You can find the completed code in the [Github](https://github.com/thilanga/di_sample){:target="_blank"}. It comes with a Dockerfile for you to try it out.

If you have any questions/suggestions around the implementation feel free to contact me. 
