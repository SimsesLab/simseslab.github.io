+++
title = 'Setting up a Homepage for my HomeLab on my personal NixOS machine'
date = 2024-03-22T15:49:47+01:00
draft = false
+++
![Picture of my dashboard](/Dashboard-front.png)
## *What is this Homepage you speak of?*
When [homelab'ing](https://linuxhandbook.com/homelab/), you often end up with a bunch of different selfhosted(info) services and URLs you need to keep track of and visit repeatedly. Using bookmarks or shortcuts to access all of your stuff is fine, but can end up becoming a bit inconvenient as your homelab grows. Using a nice front for accessing your services, like [Heimdall](https://heimdall.site/) or [Homepage](https://github.com/gethomepage/homepage), can be really nice to quickly navigate to where you want to go, with big visible icons instead of plain text labels, or small unrecognizable icons. What's cool, is that these dashboards often have extra stuff included, that can incorporate with your services, and give quick information directly in the dashboard, before even clicking on them. Really nice, especially if the reason to visit the site was to lookup said information.

### Heimdall or Homepage?
I originally used Heimdall, but was never really to pleased with it. Heimdall is configure via its nice UI, and you set up each app/service with a nice visual interface. However, I really like having everything configured through configuration files, because it gives me both a history over what I have configured, by simply reading the file, but also a natural way to backup my personal dashboard layout. If the container hosting Heimdall goes down and needs to be reconfigured, then I would have to setup each and every service once again, with potential Icons, colours and API keys. Not a fan.
After a lot of coverage from different YouTubers like [Techno Tim](https://www.youtube.com/@technotim), [Christian Lempa](https://www.youtube.com/@christianlempa) and [DB Tech](https://www.youtube.com/@DBTechYT) I jumped to Homepage, because it solved my issue with configuration, and - in my opinion - has a more pleasing look. Simple as that, because they both do their job really well, and the main difference is mostly the userbase. If you don't like to use configuration files, Homepage would be a chore and Heimdall would likely be a better match.

## *If you are hosting other stuff, then why not put your dashboard on a your server?*
I have hosted these dashboards on my server before, and tried to use *easy to remember hostnames* for my services. But, when something breaks in the network chain, I often ended up not having access to my dashboard, and then not having my quick route to fix the issue. And because of my easy to remember hostnames, I ended up forgetting the IP addresses and ports for the services, further slowing down my troubleshooting. It has only happened a couple of times, but was still really annoying each time. Having my dashboard on my local machine, served via ``localhost:<port>``, means if the homepage is down, I know it's an local issue with my specific machine. And if a greater network issue occurred, I would still have the route to the different servers, my router and switch, and can read the IP and port of the specific service by hovering over it in the dashboard.

## Some issues I encountered

After my initial setup, I had some trouble with my system hanging on shutdown with the message:

```A stop job is running for docker-Hompage.service ( 1s / 2min) ```
[Jump to my fix](#hanging-fix)

I googled around, and got the idea that it might be Docker not shutting down the container properly, and then tried switching to Podman, which was as simple as replacing ```backend = "docker";``` with ```backend = "podman";```.
This did solve the issue, although presented a new issue with the container not being able to ping, and every widget with a status check showing as down:
![Services showing down](/services_down.png)
This is because of Podman's more restricted handling of containers, meaning you have to be more deliberate about what the container should be allowed to have access to. Its more secure and something I like, although something I need to get used to. I fixed the issue with pinging by making the container use the host's network with ```--net=host```.
#### My fix for docker hanging on shutdown {#hanging-fix}
I later found out through some more testing, that the original error actually came from:
```nix
environment = {
  PUID = "1000";
  PGID = "1000";
};
```
which I had added to the container because I have experienced some file permission issues on other containers, and expected Homepage to need it, based on its ability to read image files. A fine lesson on not adding stuff you aren't sure that you'll need...
Removing it made the docker backend work without halting the shutdown process, but I have since decided to stick with Podman.

## Final notes and files
To keep my *configuration.nix* file clean, I separate the container configuration into its own nix file called *containers.nix*. This will also be a nice place for adding more local containers in the future.

My containers.nix file:
```nix
{ config, pkgs, ... }:
{
  virtualisation.oci-containers = {
    backend = "podman";
    containers = {
      Homepage = {
        autoStart = true;
        image = "ghcr.io/gethomepage/homepage:latest";
        extraOptions = [ "--net=host" ];
        ports = [
          "3000:3000"
        ];
        volumes = [
          "/home/simon/.dotfiles/Homepage-startpage/config:/app/config"
          "/home/simon/.dotfiles/Homepage-startpage/Graphics/icons:/app/public/icons"
        ];
      };
    };
  };
}
```
I save icons and Homepage config files in my *.dotfiles* folder, on my home directory called *simon*.


I then import the module into my configuration.nix file:
```nix
imports =
[ # Include the results of the hardware scan.
  ./hardware-configuration.nix
  ./containers.nix
];
```

This may not be the most exiting post, but I wanted to make it on the off chance that someone else occurred a similar issue with OCI-containers on NixOS, and that my fixes could solve their issues as well.

Anyways - Thanks for the read, and a good day to you!
