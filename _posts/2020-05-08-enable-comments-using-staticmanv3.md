---
title:  "Enable comments using staticman v3"
date:   2020-05-08
categories:
  - blog
tags:
  - staticman
  - minimal mistakes
---
This post is about how I enabled [staticman](https://staticman.net/) on this github pages site powered by [Minimal Mistaks](https://github.com/mmistakes/minimal-mistakes) as remote theme.

I tried follwoing the [Getting Started](https://staticman.net/docs/) guide but unfortunately it is a bit out of date, so my initial try at setting up comments did not work. Here is what finally worked for me follwing the same structure as [Getting Started](https://staticman.net/docs), so I am writting incase someone else may find it useful.

## Step 1. Add Staticman to your repository
Staticman need push access to the repository of the site we want to enable comments for. For GitHub, [Eduardo Bouças](https://github.com/eduardoboucas) has developed a GitHub App named [Staticman.net](https://github.com/apps/staticman-net). For some reason this app is not searchable through GitHub Marketplace, so you have to visit the app page https://github.com/apps/staticman-net and install it, you can configure it to enable only access to site repository that you want to enable comments for. In my case it was my personal GitHub pages repository.  
<figure>
  <a href="/assets/images/2020-05-08/001.png"><img src="/assets/images/2020-05-08/001.png"></a>
  <figcaption>Add staticman to repository.</figcaption>
</figure>

## Step 2. Create a configuration file
There is nothing to add from me. I will just quote the original author to save redirection.
> I will look for a staticman.yml file in the root of the repository, where various configuration parameters will be defined.
>
> You can use the [sample config file](https://github.com/eduardoboucas/staticman/blob/master/staticman.sample.yml) as a starting point and check the [available configuration parameters](https://staticman.net/docs/configuration) for more information.
>
<cite><a href="https://staticman.net/docs/">GETTING STARTED</a></cite>

## Step 2.1. Enable staticman comments in Minimal Mistakes
Update `_config.yml` to set comment provider to `staticman_v2` and update `branch` and `endpoint` under `staticman` node.
<figure>
  <a href="/assets/images/2020-05-08/002.png"><img src="/assets/images/2020-05-08/002.png"></a>
  <figcaption>Enable staticman in Minimal Mistakes.</figcaption>
</figure>
Please also make sure your `repository` value is also set correctly in `_config.yml` file.  
```
repository: # GitHub username/repo-name e.g. "mmistakes/minimal-mistakes"
```

## Step 3. Hook up your forms
No change needed as I am using [Minimal Mistaks](https://github.com/mmistakes/minimal-mistakes) as theme. 

## Step 4. Approve entries (optional)
Again no change from [Getting Started](https://staticman.net/docs) Step 4.  

That's it, happy commenting.

## References
In no particular order  
[staticman](https://staticman.net/)  
[Minimal Mistaks](https://github.com/mmistakes/minimal-mistakes)  
[spinningnumbers.org](https://github.com/willymcallister/willymcallister.github.io)
[How to Enable Staticman v3 on Minimal Mistakes](https://ovvo.cc/minimal-mistakes-staticman-v3/)  
And many more
