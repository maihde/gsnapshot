Glacier Snapshot (gsnapshot)
============================

This script is designed to combine rsnapshot with glacier-cli so that
you can periodically upload files to glacier.

You must run gsnapshot once with the "--sync" argument to make the
first upload.

    /path/to/gsnapshot --sync weekly

This will upload the entire contents of `weekly.0` to glacier.  After
that you can run without the `--sync` argument and only the differences
will be uploaded.  This can also be useful if you want to periodically
upload a full snapshot to glacier.

**IMPORTANT** When you run with `--sync` you need to ensure sufficent
disk space if available to create a full set of archives.  By default
the archives are created in the `rsnapshot_root`, but you can provide
the `--archive-root` to place them in an alternate location.

Obviously, you must run gsnapshot at least as often as you run the
equivalent rsnapshot.  It is safe to run gsnapshot more often and 
it will not upload duplicates to glacier (assuming that your glacier-cli
database is kept up to date) 
 
To automate `gsnapshot` alter your rsnapshot cron, for example:

    30 3  	* * *		root	/usr/bin/rsnapshot daily
    0  3  	* * 1		root	/usr/bin/rsnapshot weekly && /path/to/gsnapshot weekly
    30 2  	1 * *		root	/usr/bin/rsnapshot monthly
    0  2  	1 1 *		root	/usr/bin/rsnapshot yearly && /path/to/gsnapshot --sync weekly 

It's highly recommended that you have cron correctly setup to email you if
a program exits with an error.  This will allow you to detect that gsnapshot
has failed for some reason and that you need to manually upload the archives.

Approach
--------
Here is a breif synopsis of what gsnapshot does when you run `gsnapshot weekly`

1. Use the modification time stamp for `weekly.0` to create an archive filename.
1. If a manifest file is found with the correct name, assume that the archives
have already been uploaded.
1. Compares the `weekly.0` and `weekly.1`.
1. Tars and compress all of the inodes found in `weekly.0` that are not in
`daily.1`.  This information is added to a manifest file. 
1. Checks to see if the produced files (including the manifest) have already
been uploaded to glacier.  If the archive is not in the glacier-cli database
*OR* the contents of the file differ from the hash stored in the glacier-cli
database the file is uploaded to glacier.
1. The archive files are erased; the manifest file is kept on disk.

Details
-------
Because glacier download costs can be prohibitive, archives are split into
`--max-size` (default 1GB) files.  This allows you to easily spread downloads
across many months to take advantage of the free download quota you receive.
If you set `--max-size=0` then the archives will not be split, regardless of
size.

By default archives are deleted after they have been uploaded to glacier.  You
can use the `--keep-archives` to skip this.

If a manifest file is deleted, the archives will be recreated but will only
be uploaded to glacier if they are not found in the glacier-cli database.
