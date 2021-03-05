---
title: "Gradle in Tight Security Environments"
date: 2021-03-05T13:30:43+08:00
allowComments: true
tags: [gradle]
---

In tightly controlled, secured environments _\*cough, corporate, cough\*_,
using Gradle requires a little bit more setup. The following works if the
environment has an artifact repository such as Artifactory in place.

In `settings.gradle.kts`, configure `pluginManagement` as follows:

```kotlin
pluginManagement {
  repositories {
    maven {
      url = uri("https://path.to/corporate/artifact/repository")
      credentials {
        username = "approved-corporate-userid-to-access-repository"
        password = "password-of-the-approved-corporate-userid"
      }
  }
}
```

The snippet above assumes the corporate artifact repository uses Maven layout
in organizing the artifacts. Also, in such tightened environments, having
to specify the credentials is also a must. Next, specify the non-plugin
artifact repositories settings in `build.gradle.kts`:

```kotlin
repositories {
  maven {
    url = uri("https://path.to/corporate/artifact/repository")
    credentials {
      username = "approved-corporate-userid-to-access-repository"
      password = "password-of-the-approved-corporate-userid"
    }
}
```

It's the same as the `pluginManagement` settings, except that there isn't
the need to wrap `repositories` configuration in anything.

Do note that the above will only work if the corporate artifact repository
already have artifacts you want to use in it. Otherwise, the above
setups won't work, no matter how hard you cry.
