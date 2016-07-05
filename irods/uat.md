# iRODS Setup for UAT

* Production resource servers mapped to directories on the local file system.
  * Use `isysmeta -l ls` to determine which resources to check.
  * Use `ilsresc -l` to determine where the resource is located.
* Files are copied from the production resource servers by referencing a list of paths to be copied.
  * Transfer script: `$IRODS_HOME/transfer.sh`
  * Files to copy: `qa_data_sets`
  * Command to transfer the files: `./transfer.sh < qa_data_sets`
