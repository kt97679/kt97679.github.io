---
layout: post
title:  "Hiding ssh behind web server"
date:   2019-03-29 22:00:00
---

Sometime you need to access your ssh server from a very restricted environment. For example you may have only outgoing ports 80 and 443 allowed on the firewall. 
The simplest way to access your ssh server in this situation will be to run it on port 443. To hide it even better you may want to use [sslh](https://github.com/yrutschle/sslh).
On gentoo linux you need to put following line in the /etc/conf.d/sslh:
{% highlight bash %}
DAEMON_OPTS="-p 0.0.0.0:443 --ssl 127.0.0.1:8443 --ssh 127.0.0.1:22 --user nobody"
{% endhighlight %}
With those settings sslh will run as user nobody on port 443 and will forward all https connections to port 8443 (where you need to run web server) and ssh connections will go to port 22.


Unfortunately sometime environment can be even more restrictive and will allow only http and https protocols. In this case you can encapsulate ssh connection in the websocket stream.
To do this you need to use [websocat](https://github.com/vi/websocat). I'm using nginx as a frontend to websocat. Here is http section of the configuration:
{% highlight bash %}
http {
    server {
        listen 0.0.0.0:8443 ssl;
        server_name your.host.com;
        ssl_certificate /etc/letsencrypt/live/your.host.com/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/your.host.com/privkey.pem; # managed by Certbot
        location /wstunnel/ {
            proxy_pass http://127.0.0.1:8022;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
        }
    }
}
{% endhighlight %}
Websocat backend can be run as follows:
{% highlight bash %}
websocat -E --binary ws-l:127.0.0.1:8022 tcp:127.0.0.1:22
{% endhighlight %}
All connections to https://your.host.com/wstunnel/ will be forwarded to the websocat which will pipe data to the sshd. On the client you can use following command to connect:
{% highlight bash %}
ssh -o ProxyCommand='websocat --binary wss://your.host.com/wstunnel/' your.host.com
{% endhighlight %}
Now you are connected to your ssh server and your connection is a legitimate https stream :).

I measured network throughput using following command:
{% highlight bash %}
ssh -o ProxyCommand='websocat --binary wss://your.host.com/wstunnel/' your.host.com 'dd if=/dev/zero count=32768 bs=8192' >/dev/null
{% endhighlight %}
When using ws:// throughput was identical to the direct ssh connection without ProxyCommand. When using wss:// I've got 50% throughput of the direct ssh connection.

If you want to run [sshuttle](https://github.com/sshuttle/sshuttle) over websockified ssh you can do it like this:
{% highlight bash %}
sshuttle -e 'ssh -o ProxyCommand="websocat --binary wss://your.host.com/wstunnel/"' -r your.host.com 0/0 -x $(dig +short your.host.com)/32
{% endhighlight %}
