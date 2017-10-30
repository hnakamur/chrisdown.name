---
layout: post
title: "Adding power related targets to systemd"
---

systemd has a bunch of nice "power" related targets that come shipped by
default. For example, `sleep.target` is extremely useful if you want to lock
your screen on sleep.

One thing which systemd doesn't have, at least yet, is targets which relate to
going on to and coming back off of AC and battery power. This is pretty easy to
do, though, and provides the convenience of sleep.target for events like this.
I want to use these to do some power tweaks when coming on to and off of
battery.

You need [acpid](https://sourceforge.net/projects/acpid2/), since systemd
currently has only very limited support for ACPI events.

First off, create the two targets that you want to use for AC/battery:

    cat > /etc/systemd/system/ac.target << 'EOF'
    [Unit]
    Description=On AC power
    DefaultDependencies=no
    StopWhenUnneeded=yes
    EOF

<!-- -->

    cat > /etc/systemd/system/battery.target << 'EOF'
    [Unit]
    Description=On battery power
    DefaultDependencies=no
    StopWhenUnneeded=yes
    EOF

A systemd "target" is essentially a systemd unit that is used to perform
actions when a certain state is reached in the system. We'll be using these as
the binding points for services that we want to run when we switch to and from
AC/battery.

Now, we need to tell acpid to start `ac.target` when it sees us coming on to
AC, and start `battery.target` when it sees us coming off of AC. This should be
simple, but unfortunately on my T470s, coming on and off of battery looks like
this:

    % acpi_listen | grep battery
    battery PNP0C0A:00 00000080 00000001 <-- unplug
    battery PNP0C0A:00 00000080 00000001 <-- plug

Well, that's not very helpful. We get exactly the same battery event for unplug
and plug in acpid. This means that we need to look elsewhere to get information
about the AC state, but we can still use acpid to assist us in knowing when to
run. A useful file here is `/sys/class/power_supply/AC/online`, which tells
whether we are on power (value "1"), or off power (value "0"). You may have
more than one `AC*` directory if you have multiple power supplies.

This means that we can then do our activation in a script,
`/usr/local/bin/powertargets`, like so:

{% highlight bash %}
#!/bin/bash -e

on_ac=$(cat /sys/class/power_supply/AC/online)

if (( on_ac )); then
    exec systemctl start ac.target
else
    exec systemctl start battery.target
fi
{% endhighlight %}

This script will activate the correct target when run. All we have to do now is
to run it, which can be done by placing a file in acpid's `events` directory
(the path is likely `/etc/acpid`, not `/etc/acpi` except on Arch Linux):

    cat > /etc/acpi/events/powertargets << 'EOF'
    event=battery.*
    action=/usr/local/bin/powertargets
    EOF

After reloading acpid's config, we can now test it out by unplugging the power
and setting what happens:

    % sudo systemctl status battery.target
    ● battery.target - On battery power
       Loaded: loaded (/etc/systemd/system/battery.target; static; vendor preset: disabled)
       Active: inactive (dead)

    Oct 29 12:24:33 roujiamo systemd[1]: Reached target On battery power.
    Oct 29 12:24:33 roujiamo systemd[1]: battery.target: Unit not needed anymore. Stopping.
    Oct 29 12:24:33 roujiamo systemd[1]: Stopped target On battery power.

Sweet, it works! Now that everything is setup, we can add the services that we
want to start based on these targets. Take this as an example, and note the
`[Install]` section. You need to enable the service, as well, or the symlinks
to `battery.target.wants` won't be created.

    % systemctl cat powerdown.service
    # /etc/systemd/system/powerdown.service
    [Unit]
    Description=Laptop battery savings

    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/powerdown

    [Install]
    WantedBy=battery.target

Try unplugging again after enabling the service, and you should see it working,
too:

    % sudo systemctl status powerdown
    ● powerdown.service - Laptop battery savings
       Loaded: loaded (/etc/systemd/system/powerdown.service; enabled; vendor preset: disabled)
       Active: inactive (dead) since Sun 2017-10-29 02:24:33 GMT; 22min ago
      Process: 15977 ExecStart=/usr/local/bin/powerdown (code=exited, status=0/SUCCESS)
     Main PID: 15977 (code=exited, status=0/SUCCESS)

    Oct 29 12:24:33 roujiamo systemd[1]: Starting Laptop battery savings...
    Oct 29 12:24:33 roujiamo systemd[1]: Started Laptop battery savings.

Plug it back in, and you should also see the same for whatever you have
attached to `ac.target`.

The commit in which I add all of this to my config management is
[433013fa](https://github.com/cdown/ansible-desktop/commit/433013fafb2c03cc7b94ae80299e1c001b06263b).