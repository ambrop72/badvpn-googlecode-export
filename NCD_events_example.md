# NCD #

NCD, the Network Configuration Daemon, is a daemon and programming language for configuration of network interfaces and other aspects of the operating system.

This page describes the event handling features of NCD. For a guide to NCD and its basic feaures, see [NCD](NCD.md).

# Introduction #

NCD has some support for imperative-style event handling, still using its simple execution mode. See [this introduction](http://code.google.com/p/badvpn/wiki/NCD#Event_handling_and_imperative_style) first. This page provides a similar example to the DHCP example there, but somehow more complex.

# Goal #

This program responds to volume key presses by synchronously calling an external script for muting and adjusting volume, and responds to power button presses by suspending using pm-suspend.

It uses process\_manager() and sys.watch\_input() to dynamically create and remove processes that deal with specific input devices. The individual input device processes then use sys.evdev() to handle input events from their input device.

# Code #

```
#
# NCD input event handling example program.
#
# This program responds to volume key presses by synchronously calling an external
# script for muting and adjusting volume, and responds to power button presses by
# suspending using pm-suspend.
#
# It uses process_manager() and sys.watch_input() to dynamically create and remove
# processes that deal with specific input devices. The individual input device processes
# then use sys.evdev() to handle input events from their input device.
#

process events_main {
    # Volume control script, called with argument "up", "down" or "mute".
    var("/usr/local/bin/volumekey") volume_script;

    # Suspend command.
    list("/usr/sbin/pm-suspend") suspend_cmd;

    provide("GLOBAL");
}

process events_watcher {
    depend("GLOBAL");
  
    # Create process manager.
    process_manager() manager;

    # Wait for input device events.
    sys.watch_input("event") watcher;

    # Determine event.
    strcmp(watcher.event_type, "added") is_added;
    strcmp(watcher.event_type, "removed") is_removed;

    # Dispatch.
    If (is_added) {
        manager->start(watcher.devname, "events_input_device", {watcher.devname});
    } elif (is_removed) {
        manager->stop(watcher.devname);
    };

    # Next event.
    watcher->nextevent();
}

template events_input_device {
    # Alias arguments.
    var(_arg0) dev;

    # Get global.
    depend("GLOBAL") gdep;

    # Dispatch map.
    value([
        {"1", "KEY_MUTE"}:"events_input_event_mute",
        {"1", "KEY_VOLUMEUP"}:"events_input_event_vup",
        {"1", "KEY_VOLUMEDOWN"}:"events_input_event_vdown",
        {"1", "KEY_POWER"}:"events_input_event_power"
    ]) dispatch_map;

    # Wait for input events.
    sys.evdev(dev) evdev;

    # Query event details.
    dispatch_map->try_get({evdev.value, evdev.code}) dispatch_func;

    # Dispatch or ignore.
    If (dispatch_func.exists) {
        call(dispatch_func, {});
    };

    # Next event.
    evdev->nextevent();
}

template events_input_event_mute {
    runonce({_caller.gdep.volume_script, "mute"});
}

template events_input_event_vup {
    runonce({_caller.gdep.volume_script, "up"});
}

template events_input_event_vdown {
    runonce({_caller.gdep.volume_script, "down"});
}

template events_input_event_power {
    runonce(_caller.gdep.suspend_cmd);
}
```

**WARNING**: you should pass `--loglevel warning` to NCD to avoid spamming the terminal or system log with tens of lines every time the mouse is moved or key pressed, and to avoid wasting CPU building the log messages.

For completeness, here's an example volume script, which you should place at `/usr/local/bin/volumekey`. It should be made executable, and the card name should be adjusted (look at the symlinks in `/proc/asound`).

```
#!/bin/bash

CARD=Intel
CHANNEL=Master
INCREMENT=4%

KEY=$1

MUTED=no
amixer -c "${CARD}" get "${CHANNEL}" | grep '\[off\]' >/dev/null
[[ $? = 0 ]] && MUTED=yes

if [[ $KEY = up ]] || [[ $KEY = down ]]; then
        if [[ $MUTED = yes ]]; then
                amixer -c "${CARD}" set "${CHANNEL}" unmute
        else
                if [[ $KEY = up ]]; then
                        amixer -c "${CARD}" set "${CHANNEL}" "${INCREMENT}"+
                else
                        amixer -c "${CARD}" set "${CHANNEL}" "${INCREMENT}"-
                fi
        fi
elif [[ $KEY = mute ]]; then
        amixer -c "${CARD}" set "${CHANNEL}" mute
fi
```