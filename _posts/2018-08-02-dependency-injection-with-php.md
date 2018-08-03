---
layout: post
title: "Dependency Injection with PHP"
date: 2018-08-02
comments: true
tags: dependency-injection clean-code php
description: How to do Dependency Injections properly and keep code clean and scalable 
---
```
Dependency Injection. Everyone knows about it, has heard about it.
```

This is the common answer I get from most of candidates I have interviewed in the past couple of years when looking for Software Engineers to work on the BI software [EngineRoom](https://digital360.com.au/technology){:target="_blank"} that we are building at [Digital360](https://digital360.com.au/){:target="_blank"}.
But when digging bit deeper to solutions they have built in the past, it has shown that most of solutions haven’t followed proper implementation techniques.

In this blog post, I'll try to explain my take on the dependency injection.

### What is a dependency?
According to the [Wikipedia](https://en.wikipedia.org/wiki/Dependency_injection){:target="_blank"}.
In software engineering, dependency injection is a technique whereby one object (or static method) supplies the dependencies of another object.
A dependency is an object that can be used (a service). An injection is the passing of a dependency to a dependent object (a client) that would use it.

In this context an object (A) that expects another object's (B) help to complete a job that object (A) is responsible for.
To explain the theory, I'll be building a simple music player that can be used against multiple music services like Google Music, Spotify, etc.

### Objective
Music player should be able to play music from user's preferred music service.

#### ProTip: How to find which dependencies to inject?
`If we look at the above requirement, the music services are on the strong side and the music player has less authority in making decisions.
In this case we inject the music service to the music player.`

#### First let's start with a bad code to get it working.

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

If you look at the code above, someone might say `yeah that's dependency injection`. Not just injection.
It's constructor injection. Because it injects the object in to the Music player class using the constructor.

This is not a good dependency injection. Because GoogleMusic class is hard wired to the MusicPlayer class.

We also can create the GoogleMusic object inside the MusicPlayer class.

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
Sad truth is, both code snippets above doing the same mistake. They are hard wired to the MusicPlayer class.
Now Music class has become unusable for Spotify music service and only can be used by GoogleMusic.

Let's add support to Spotify music service. 

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

In this scenario, the music player can work with Google Music and Spotify. But the Music services are still coupled tightly with the player.

#### What are the issue we are going to face if we couple our code tightly to the Music player?
1. At the moment we can't support another Music Service. For an example, if we need to support Apple Music.
Only there are two ways to support this with the current implementation. 

    - Inject AppleMusic as an argument
    - Initiate AppleMusic object within the constructor

2. Another issue is having to edit two files minimum (MusicPlayer class and AppleMusic class) 

#### Ok. Enough of bad approaches. What's a correct approach?
`Dependency Injection is the correct approach here.`

First. Let's look at the problem we are trying to solve once more. 

`We are building a system that can play music based on a user's prefered service.`

If you read the problem again you can see that we are going to have multiple music services.
To use multiple services with dependency injections we need to build services against contracts. When I say a contract I meant an `Interface class`.

#### ProTip: When to create an Interface?
`I have seen some developers go overboard with writing codes against interfaces for everything and make the code so dynamic without having the real need.
If you are going to have more than one similar type of class (Google Music, Spotify, Apple Music, etc.), then only create an interface.`

As we know that we are going to support more than one music service, we will be creating `IMusicService interface` and `ITrack` interface.

We don't need an interface for the music player because there is going to be only one player.

#### But why an Interface for the Track?

We don't know how other services will return track listings and other information.
In this case to be safe we can have separate ITrack implementations to handle separate responses.

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

It's time to let the music player to play some tracks from services. 

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

#### We can do better.
We can use an IoC container (Inversion of Control) to handle these objects creations and not to worry about it again until there is a new service that we need to support.

As IoC is its own topic to talk about and I won't get deeper in to the subject here. For the moment think about it as a dynamic object injection based on required dependencies by the MusicPlayer class (Just remember IoC is more than that).

`In this article/tutorial I'll be using Laravel's IoC container.`

Once the IoC installation is done using the composer, we can create an IoC instance and let it to handle injection to the MusicPlayer instance based on the logic we want. 

In our case loading different music service objects bases on different user preferences.

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

#### Now it's time to test the code.
You can find the completed code in the [Github](https://github.com/thilanga/di_sample){:target="_blank"}. It comes with a Dockerfile for you to try it out.

If you have any questions/suggestions around the implementation feel free to contact me. 


