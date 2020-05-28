---
layout: post
excerpt_separator: <!--more-->
title: Getting Started With Jekyll Without Installing Ruby
categories: jekyll docker
---

# Ruby 설치없이 Jekyll 사용하기
## 개요

이 문서는 Ruby 설치없이 Jekyll 도커 컨테이너를 사용하여 jekyll 블로그를 만드는 방법을 설명한다.
<!--more-->

## 도커 설치

먼저, 도커를 설치해야 한다. 개발자라면 도커가 굉장히 유용하기 때문에 설치를 적극 권장한다.

도커가 설치되어 있지 않다면 [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)를 참고하여 도커를 설치한다.

## 컨테이너에 진입

원하는 곳에 디렉터리를 만들어, 디렉터리 밑으로 이동한다.

```shell
mkdir ~/my-jekyll-blog
cd ~/my-jekyll-blog
```

그 다음 다음과 같이 실행하여 jekyll 컨테이너로 진입한다.

```shell
docker run -it --rm --volume="$PWD:/srv/jekyll" -p 4000:4000 jekyll/jekyll:4 bash
```

## 블로그 디렉터리 생성

`/srv/jekyll` 로 이동하여 다음 명령을 실행하면 blog 디렉터리가 생긴다(디렉터리 이름은 원하는 것을 사용하기 바란다). 

```shell
/usr/loca/bundle/bin/jekyll new blog
```

## Gemfile 편집

`blog` 디렉터리 안으로 들어가서 `Gemfile`을 편집한다. vim 사용법은 따로 설명하지 않는다.

```shell
vi Gemfile
```

```ruby
# Hello! This is where you manage which Jekyll version is used to run.
# When you want to use a different version, change it below, save the
# file and run `bundle install`. Run Jekyll with `bundle exec`, like so:
#
#     bundle exec jekyll serve
#
# This will help ensure the proper Jekyll version is running.
# Happy Jekylling!
#gem "jekyll", "~> 4.0.1"
# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem "minima", "~> 2.5"
# If you want to use GitHub Pages, remove the "gem "jekyll"" above and
# uncomment the line below. To upgrade, run `bundle update github-pages`.
gem "github-pages", group: :jekyll_plugins
gem "jekyll-manager", group: :jekyll_plugins

# If you have any plugins, put them here!
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
install_if -> { RUBY_PLATFORM =~ %r!mingw|mswin|java! } do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :install_if => Gem.win_platform?
```

위의 내용에서 `jekyll`을 주석처리하고, `github-pages`와 `jekyll-manager`를 활성화하여 저장한다.

## _config.yml 편집

그리고 `_config.yml`을 수정한다.

```shell
vi _config.yml
```

```yaml
# Welcome to Jekyll!
#
# This config file is meant for settings that affect your whole blog, values
# which you are expected to set up once and rarely edit after that. If you find
# yourself editing this file very often, consider using Jekyll's data files
# feature for the data you need to update frequently.
#
# For technical reasons, this file is *NOT* reloaded automatically when you use
# 'bundle exec jekyll serve'. If you change this file, please restart the server process.
#
# If you need help with YAML syntax, here are some quick references for you:
# https://learn-the-web.algonquindesign.ca/topics/markdown-yaml-cheat-sheet/#yaml
# https://learnxinyminutes.com/docs/yaml/
#
# Site settings
# These are used to personalize your new site. If you look in the HTML files,
# you will see them accessed via {{ site.title }}, {{ site.email }}, and so on.
# You can create any custom variable you would like, and they will be accessible
# in the templates via {{ site.myvariable }}.

title: Trust Me
author: Ilho Yu
email: acid.acidrain@gmail.com
description: This is my blog about programming.
baseurl: "" # the subpath of your site, e.g. /blog
url: "" # the base hostname & protocol for your site, e.g. http://example.com
github_username: m0rph2us

# Build settings
theme: minima
plugins:
  - jekyll-feed

# Exclude from processing.
# The following items will not be processed, by default.
# Any item listed under the `exclude:` key here will be automatically added to
# the internal "default list".
#
# Excluded items can be processed by explicitly listing the directories or
# their entries' file path in the `include:` list.
#
exclude:
  - .sass-cache/
  - .jekyll-cache/
  - gemfiles/
  - Gemfile
  - Gemfile.lock
  - node_modules/
  - vendor/bundle/
  - vendor/cache/
  - vendor/gems/
  - vendor/ruby/
```

위에서처럼 블로그 정보 설정을 하고, exclude를 활성화 한다음 저장한다.

## 번들

다음 명령으로 번들링한다.

```shell
bundle install
```

## 로컬 서버 실행

다음으로 로컬에서 실행해 볼 수 있다.

```shell
bundle exec jekyll serve --host 0.0.0.0
```

`http://localhost:4000`으로 접근하면 블로그 화면으로, `http://localhost:4000/admin`으로 접근하면 블로깅 툴을 사용할 수 있다.

## github.io로 올리기

github page를 만드는 방법은 [https://pages.github.com/](https://pages.github.com/)의 지침을 따르기 바란다. 생성한 리포지토리로 생성한 파일들을 푸시하기만 하면 된다. 띄울때 문제가 생기면 github 계정에 연결된 이메일로 알림이 온다.

## 참고

1. [구글 검색엔진 등록하기](https://gmlwjd9405.github.io/2017/10/20/include-blog-in-a-GoogleSearchEngine.html)

