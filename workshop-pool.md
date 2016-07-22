# Workshop Pool

The purpose of this change is to create a pool of job execution nodes that is dedicated to CyVerse
workshops so that participants don't have to wait for resources in the general pool to become
available before their jobs run.

Desired features:

- Group for running jobs on a dedicated pool of execute nodes.
- Belphegor UI for updating the group.
- Time limits for tools integrated at a workshop.
- Notification of impending time limit expiration.
- Time limit extension. (This feature is optional.)

## Required Changes

### Dedicated Pool for Workshops

The first change required to support this feature will be a way to allow administrators to add users to
and remove users from the group that has access to the dedicated pool of execute nodes. The easiest and
safest way to support this feature at the moment is to provide a simple UI in belphegor that can be used
to manage a group in Grouper. Some convenience endpoints can be created so that Belphegor doesn't have to
worry about keeping environments segregated.

The second change required is to get the group information down to the job launcher. This doesn't seem as
though it should be too difficult. We simply need to get the group membership information from Grouper and
pass it down. The apps service can do this easily enough. The one thing that I'm worried about is the time
that it takes to look up group membership information using the Grouper web services. I may cheat and add
an endpoint to the permissions service for this task. The permissions service already looks up group
membership information directly in the database, so adding an endpoint to expose this capability to other
services wouldn't be too much of a stretch.

I know less about what is required to send this information down to Condor, but it doesn't seem like that
change would be extremely difficult. I'm sure that John would be able to point either Ian or I to the code
that needs to be changed.

The last change is to update the Condor configuration so that this pool may only be used by members of the
workshop group. Chances are that this change would fall on Core Services rather than us. Trying to create
an automated way to do this is too much to accomplish given the limited amount of time we have.

### Belphegor UI for Updating the Workshop Group

This was mentioned briefly in the last section. Since Ian mentioned that he'd like to work on the job
services and Sriram will be out of town at the time, this might be something good for me to work on. As
long as I'm not _only_ working on the apps and terrain services, this is fine with me. I'm sure I can bug
Paul, Ashley or Stroot if I have any questions about the UI.

### Time Limit Notifications

The hard part about this change is getting the notifications sent from the job services themselves. At a
high level, `road runner` would have to be modified to set two timers when the job is launched: one for the
time limit expiration notification and one for the actual time limit expiration. This is conceptually simple,
but the implementation details could be hairy. John knows the most about the implementation details, and I
imagine that Ian will end up working on this portion of the project.

The modifications to the apps service should be fairly straightforward. When the apps service receives a
status update indicating that a job's time limit is about to expire, it'll trigger a notification. I don't
anticipate needing to make any changes to the notification agent itself; the generic notification endpoint
should provide all of the features that we need to support this. If any notification agent changes are
required, they'll be minimal.

The update to the UI will require support for a new type of notification. I know very little about what is
required to support this, but it seems as though it should be easy enough.

### Time Limit Extension

There are two aspects of this change that are fairly difficult. The first is getting the message indicating that
the time limit is going to be expected down to `road runner`. I believe that this task is mostly handled at the
moment, however. `road runner` already listens to an AMQP queue for job cancellations if I remember correctly. If
this is the case then we should be able to extend this to support time limit extensions as well. The second is
actually extending the timer, which would involve somehow determining exactly how much time is left and extending
this time period for some pre-determined time interval. This seems doable, but I'm not entirely sure how at this
time. We'd probably also have to worry about an overall time limit in order to prevent users from extending their
job time limits indefinitely.

Once again, the apps changes for this are relatively simple. Extra endpoints in apps, terrain, and the JEX adapter
should be sufficient to support this.

The UI changes will require us to add support for actionable notifications. That is, when the user receives one
of these new notifications, he or she would be able to request a time limit extension. There are some nasty corner
cases to avoid here. For example, what happens if a user doesn't log into the DE again until after the job has
been canceled. In this case, displaying the option to extend the time limit would be moot because the job isn't
running any longer.
