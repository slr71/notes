# NCBI SRA Submission Utility

### Usage

```
usage: ncbi_sra_submit.py [options]

Prepares files and metadata downloaded from the Discovery Environment Data
Store for submission to the NCBI Sequence Read Archive (SRA).

optional arguments:
  -s SUBMIT_MODE, --submit-mode SUBMIT_MODE
                        specify if the SRA submission is a BioProject "create"
                        (default) or an "update" request.
  -i PRIVATE_KEY_PATH, --private-key PRIVATE_KEY_PATH
                        (optional) specify an alternative path to the id_rsa
                        private-key file.
  -f INPUT_DIR, --input-dir INPUT_DIR
                        specify the path to the input BioProject folder to
                        submit to SRA
  -m METADATA_PATH, --input-metadata METADATA_PATH
                        specify the path to the BioProject folder metadata
                        file.
  -v, --validate-metadata-only
                        when included, no data will be submitted and only the
                        BioProject folder metadata file will be validated.
  -d SUBMIT_DIR, --submit-dir SUBMIT_DIR
                        specify the path to the destination BioProject SRA
                        submission folder
  -?, --help

```

### Code Trace

* Parse the command-line arguments.
* Instantiate a metadata client.
* Instantiate a Jinja2 Environment object.
* Call `metadata_client.get_metadata()` to extract the metadata from the exported metadata file.
  * Open the file and parse the JSON.
  * Verify that both `metadata` and `folders` keys exist.
  * Build a metadata object containing a dictionary of attribute names to attribute values.
  * Call `self._parse_folder_metadata()` to parse the metadata for each bio sample.
    * Verify that the required fields are present in the bio sample metadata.
    * Build a metadata object containing a dictionary of attribute names to attribute values, including only
      attributes whose names appear in the list of reserved bio sample attributes.
    * Attributes whose names do not appear in this list are added to a list of attribute objects with each
      object containing a `name` key and a `value` key.
    * Call `self._parse_library_metadata()` to parse the metadata for each library in the bio sample.
      * Verify that the library has the required metadata attributes.
      * The library metadata object is built in roughly the same way as the bio sample metadata object.
      * Call `self._parase_file_metadata()` to parse the metadata for each library in the file.
        * Verify that the file has all of the required attributes.
        * Return an object containing the name and MD5 hash of the file.
  * Extract the libraries from the parsed folder metadata and add it as a top-level key in the bio project.
* Create the destination directory for the submission.
* Determine the submission template based on the value of the `--submit-mode` option.
* Generate the submission XML using the submission template.
* Validate the generated XML against the XML schema.
* If doing a dry run, just print a message. Otherwise, send the submission to NCBI.
