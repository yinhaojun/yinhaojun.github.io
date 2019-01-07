# Ocean

Ocean is a mobile-enabled Hexo theme based on the features in Hexo's default theme landscape. Since I am a Designer and not a Coder, so please advise! I am very grateful to [youchen1992](https://github.com/youchen1992) for providing technical support during the Ocean production process.


[Preview](https://zhwangart.github.io)

[中文说明](https://zhwangart.github.io/2018/11/30/Ocean/)

![Screenshot](screenshots/hexo-theme-ocean.jpg)

### Install

``` bash
$ git clone https://github.com/zhwangart/hexo-theme-ocean.git themes/ocean
```

### Enable

Modify `theme` setting in `_config.yml` to `ocean`

``` yml
theme: ocean
```

### Update

``` bash
cd themes/ocean
git pull
```

### Configuration

let me know if you can’t find something.

``` yml
# Menu
menu:
  Home: /
  Archives: /archives
  Gallery: /gallery
  About: /about
rss: /atom.xml

# Miscellaneous
favicon: /favicon.ico
brand: /images/hexo.svg

# Ocean Video
# Because I put videos in multiple formats on the same path, I just labeled the path here.
ocean:
  overlay: true      # Overlay that can be closed
  path: images/ocean/      # Video storage path, formats: mp4/ogg/webm
  brand: /images/hexo-inverted.svg      # Optional, a small logo

# Content
excerpt_link: Read More...
nav_prev_text: Newer posts
nav_next_text: Olde posts
share_text: Share

# fancybox
fancybox: true

# Local search
search_text: Search

# Gitalk
gitalk:
  enable: false # true
  clientID:      # GitHub Application Client ID
  clientSecret:      # Client Secret
  repo:      # Repository name
  owner:      # GitHub ID
  admin:      # GitHub ID
```

The [feathericon](https://feathericon.com) in the menu is programmed ordely in "CSS `source/css/_partial/navbar.styl` " and can be changed or added if needed.

``` css
.nav-item
  &:nth-child(1)         // home
    .nav-item-link
      &::before
        content '\f12f'
  &:nth-child(2)         // archives
    .nav-item-link
      &::before
        content '\f12a'
  //&:nth-child(3)         // gallery
  //  .nav-item-link
  //    &::before
  //      content '\f1a9'
  //&:nth-child(4)         // about
  //  .nav-item-link
  //    &::before
  //      content '\f174'
```

### Post poster

``` md
title: Post name

photos: [
        ["img_url"],
        ["img_url"]
        ] 
```

### Gallery
Need to write in the head of the markdown, this is not a good way to write, I hope to get a better way to write on github.

``` md
title: Gallery

albums: [
        ["img_url","img_caption"],
        ["img_url","img_caption"]
        ]
---
```

