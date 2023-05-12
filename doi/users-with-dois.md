# Finding Users with DOIs

The first step was to get a list of paths to data sets with DOIs. That one is pretty easy in our case because all of the
data sets that have DOIs are in a single directory in our system.

```
$ ils /iplant/home/shared/commons_repo/curated | awk '/^ +C-/ {print $2}' > paths.txt
```

The next step is to get the corresponding UUIDs:

```
$ for dataset_path in $(< paths.txt); do imeta ls -C "$dataset_path" ipc_UUID; done | awk '/^value:/ {print $2}' > uuids.txt
```
