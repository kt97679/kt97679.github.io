---
layout: post
title:  "Locking in bash"
date:   2017-03-05 16:17:00
---

To ensure that there is not more than one running instance of your bash script you need to implement locking.
If your platform has flock utility this is pretty straightforward:

{% highlight bash %}
#!/bin/bash

LOCK_FILE=/tmp/my-script.lock
LOCK_FD=9

get_lock() {
    # need to use eval here for proper expansion
    eval "exec $LOCK_FD>$LOCK_FILE"
    flock -n $LOCK_FD
}

get_lock || exit

# ...

{% endhighlight %}

Using this approach you need to remember, that any forked processes will inherit file handles from this script. I have a script, that is run by cron. This script starts ssh-agent if it is not running and runs commands via ssh on several hosts.
The problem is that with approach above ssh-agent will hold the lock, so this script will run only once until ssh-agen will exit. To avoid this you need to close explicitly file handles while starting processes that will fork.
For ssh-agent example this will look like:

{% highlight bash %}
#!/bin/bash

LOCK_FILE=/tmp/my-script.lock
LOCK_FD=9
SSH_KEY=/root/.ssh/id_rsa.for.ssh-agent

get_lock() {
    # need to use eval here for proper expansion
    eval "exec $LOCK_FD>$LOCK_FILE"
    flock -n $LOCK_FD
}

get_lock || exit

socket=$(find /tmp/ssh-*/agent.* -user root 2>/dev/null || true)
if [ -z "$socket" ]; then
    # need to use eval here for proper expansion
    # we need to close explicitly fd of the lock file
    # otherwise open fd is kept by ssh-agent and lock can't be aquired until ssh-agent exits
    eval ". <(ssh-agent $LOCK_FD>&-)"
    ssh-add $SSH_KEY
    return
else
# ...
fi
#...
{% endhighlight %}

If for some reason you can't use flock locking is still possible. Here is how you can implement it:

{% highlight bash %}
#!/bin/bash

set -u

PID_LIST=/tmp/test-get-lock.pid

get_lock() {
    local pid
    while true; do
        while read pid; do
            kill -0 $pid || continue
            [ "$pid" != "$BASHPID" ] && return 1
            echo $BASHPID >$PID_LIST.new && mv $PID_LIST.new $PID_LIST && return 0
        done < $PID_LIST
        echo $BASHPID >>$PID_LIST
    done
}

if get_lock 2>/dev/null; then
    sleep 1
    pids="$(cat $PID_LIST)"
    pid=$(echo "$pids"|head -n1)
    [ "$BASHPID" != "$pid" ] && echo "pid: $BASHPID unexpected pid: $pid $pids"
    echo "pid: $BASHPID get_lock success"
else
    echo "pid: $BASHPID get_lock failed"
fi
{% endhighlight %}

How this works:


- Pids are tracked via pid list. We go through all pids and check if any of them are alive.
- Dead pids are ignored.
- If we encounter alive pid, that is not current pid - some other process was started earlier, we report failure.
- For alive pid that is this process we truncate pid list to just current pid (mv is atomic) and report success.
- If we exited pid check loop we append current pid to the pid list and try again. Append is atomic operation (more details [here](http://stackoverflow.com/questions/9926616/is-echo-atomic-when-writing-single-lines))


How reliable is this?

For testing I used following command:

{% highlight bash %}
rm -f /tmp/*.log; for x in {0000..9999}; do ./lock-test.sh >/tmp/$x.log 2>&1 & done; wait; echo "success: $(grep success /tmp/*.log|wc -l), failure: $(grep failed /tmp/*.log|wc -l), unexpected pid: $(grep unexpected /tmp/*.log|wc -l)"
{% endhighlight %}

Success criteria was no unexpected pids.

For final testing I used following command:

{% highlight bash %}
for y in {000..999}; do echo -n " $y"; bash -c 'rm -f /tmp/*.log; for x in {0000..9999}; do ./lock-test.sh >/tmp/$x.log 2>&1 & done; wait' 2>/dev/null; grep unexpected /tmp/*.log && break; done
{% endhighlight %}

I tried to run this test on my laptop with 4 cores i7, 2 core vm and 24 core server. There were no failures. However I'm not sure I can forsee all possible scenarios and suspect this code may fail.
You shouldn't use this code to control nuclear reactor, but it is good enough to ensure that your cron job will run as a singleton.
