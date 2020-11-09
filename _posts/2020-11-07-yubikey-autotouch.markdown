---
layout: post
title:  "Yubikey autotouch"
date:   2020-11-07 22:00:00
---

[![DSCF0153.JPG]({{ "/assets/DSCF0153.JPG" | absolute_url }}){:height="300px"}]({{ "/assets/DSCF0153.JPG" | absolute_url }})

Yubikey is a popular solution for the 2 Factor Authentication. Typical usage scenario is: you enter credentials and then touch yubikey sensor to complete authentication.
I was curious if touch can be automated. 1st I looked for purely programmatic solution, but [this forum thread](https://forum.yubico.com/viewtopic5ac4.html?p=2409)
showed that this is impossible. My next experiment was to simulate touch using capacitor. I took 22uF 50V capacitor, connected 1 wire to the yubikey sensor and touched
connector casing (which is grounded) with another wire. This worked! So next thought was: what device I can use to commutate capacitor? After some search I decided to 
try [Digispark USB Development Board](http://digistump.com/products/1). I found a very helpful [Little-Wire project](https://github.com/kt97679/Little-Wire) that can
be used to control pins on the Digispark board. I've created fork of the repository with [yubikey branch](https://github.com/kt97679/Little-Wire/tree/yubikey) and
[added](https://github.com/kt97679/Little-Wire/commit/6d7d494d06f46f5cc8cc2a08f3e713bf4cf6150a) file [yubikey-touch.c](https://github.com/kt97679/Little-Wire/blob/yubikey/software/examples/yubikey-touch.c)
based on the examples.


Here is sample a session:
{% highlight bash %}
$ sudo ./yubikey-touch 2500
> 1 Little Wire device is found with serialNumber: 512
> Little Wire firmware version: 1.3
eifjcchchfidlvjnvithvhvbkivnbtgrhjfdukvt@joy:~/git/Little-Wire/software$ eifjcchchfidlvjnvithvhvbkivnbtgrhjfduckibnih
eifjcchchfidlvjnvithvhvbkivnbtgrhjfduckibnih: command not found
$ 
{% endhighlight %}

Now you can programatically touch yor yubikey :-)!

You may find useful another [project](https://github.com/maximbaz/yubikey-touch-detector) that can help you detect situations when yubikey requires touch.
