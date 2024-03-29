---
title: "Github Actions кеширование зависимостей"
author: ""
type: ""
date: 2021-09-09T19:43:15+03:00
subtitle: ""
image: ""
tags: []
url: /russian/2021/09/09/github_actions_cache
private: false
---
В заметке я подготовил несколько сниппетов кода для быстрого включения кеширования в Github Actions.

<!--more-->
Github Actions умеет кешировать файлы. 
Например, вот так можно кешировать `Gradle Wrapper` и зависимости приложения, в результате они не будут скачиваться при каждой сборке:
```
# Cache Gradle dependencies
- name: Setup Gradle Dependencies Cache
  uses: actions/cache@v2
  with:
    path: ~/.gradle/caches
    key: ${{ runner.os }}-gradle-caches-${{ hashFiles('**/*.gradle', '**/*.gradle.kts') }}

# Cache Gradle Wrapper
- name: Setup Gradle Wrapper Cache
  uses: actions/cache@v2
  with:
    path: ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-wrapper-${{ hashFiles('**/gradle/wrapper/gradle-wrapper.properties') }}
```

## JVM
В случае если вы используете [`actions/setup-java`](https://github.com/actions/setup-java),
то его вторая версия уже поддерживает кеширование зависимостей и враперов, достаточно ее включить:
```
  - name: Set up JDK
    uses: actions/setup-java@v2
    with:
      distribution: zulu
      java-version: 11
      cache: gradle
```
Поддерживаются gradle и maven.

[Подробнее в офф документации](https://docs.github.com/en/actions/guides/caching-dependencies-to-speed-up-workflows)
