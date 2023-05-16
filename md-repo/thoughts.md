# Random Thoughts

- If we're using tickets then using a staging area outside of the data store won't work.
- Can tickets be write-only?

## New Workflow

- User logs into the system and indicates that they want to upload some data sets.
- The web UI generates a ticket (write-only)?
- The web UI displays a command that the user can run to upload the data.
- The user runs the command to upload the data.
- DataWatch detects that the file is present and calls a webhook.
- The web hook kicks off the post-processing tasks.

## Shim

- Client will have checkpointing and parallel transfers.
