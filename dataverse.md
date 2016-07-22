# DataVerse

- Researchers create a project.
- All project data goes into the project.
- Researchers have to provide credentials.

## Publish Service

We need to support dynamic loading of interface implementations. Mike was able to get a prototype of this working
using Java annotations and reflection. This can be relatively easily incorporated into the service.

Launching apps from within the service is going to require information that normally wouldn't be accessible to the
plug-ins. I believe that we can manage this by sending an implementation of an interface that can be used to submit
jobs to each of the plug-ins.
