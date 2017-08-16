---
title: "Showing UTF8 Characters in TeamCity 10 Build Logs"
date: 2017-08-15T21:43:19-04:00
draft: false
tags: [
    "teamcity"
]
categories: [
    "TeamCity",
    "Continuous Integration",
]
---

As part of a continuous integration pipeline I'm running my Postman api tests after every deployment to our development environment via TeamCity and Newman. One issue that I encountered was that the TeamCity agent build log wasn't showing formatted newman output correctly because by default it doesn't show UTF8. In order to fix this you just have to update your `C:\TeamCity\buildAgent\conf\buildAgent.properties` file to include `teamcity.runner.commandline.stdstreams.encoding=UTF-8` and then restart the TeamCity Agent service. The next time newman runs, you'll be able to decipher your build log.