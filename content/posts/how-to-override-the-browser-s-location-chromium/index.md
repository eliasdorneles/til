---
title: "How to override the browser's location"
date: "2023-12-09T21:27:19+01:00"
tags:
- web development
- frontend
- timezone
- browser
- chromium
---

In a web application, it's a common pattern to have the backend to send
a UTC timestamp, which the frontend will later display in the user's timezone.
In these situations, you may want to spoof your browser's location in order to
verify that things are working as you'd expect.

This is possible in Chromium and in Google Chrome, by opening the developer
tools (<kbd>F12</kbd>), and pull up the menu with three dots `⋮` (next to the
developer tools close button), then choose **More Tools** -> **Sensors**, and
you'll see options to override geolocation sensors.

An image is worth a thousand words:

{{< figure src="./screenshot.jpg" alt="Chromium Location Override Example" >}}

The console displays the result of running `new Date()`, before and after
overriding the location.

### In Firefox

Unfortunately, there is no graphical user interface to do the same in Firefox. 😢

You're forced to fiddle with the `about:config` settings, [the method is explained here](https://security.stackexchange.com/questions/147166/how-can-you-fake-geolocation-in-firefox)

DISCLAIMER: I have not tried it!

