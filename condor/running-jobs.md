# Useful Commands for Findin Running Jobs

```
condor_q -allusers -nobatch -af:ht Iwd RemoteHost IpcUsername IpcUuid
```

Filtering rows can be done with a constraint:

```
condor_q -allusers -nobatch -af:ht Iwd RemoteHost IpcUsername IpcUuid -constraint 'IpcUsername =?= "upendra"'
```

Or it's possible to do it with a grep command as well.
