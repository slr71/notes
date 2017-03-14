# Grouper Database Upgrade

- Shut down old version of Grouper.
- Start the old version of `gsh`.
- Run `loaderRunOneJob("CHANGE_LOG_changeLogTempToChangeLog")`

Heh, I haven't been able to get that last command to finish. It might be because I'm using a new Grouper container,
which means that the log files are unavailable. At any rate, I can figure out how to upgrade Grouper later. In the
meantime, I'll add some cache settings to the version we're currently using.
