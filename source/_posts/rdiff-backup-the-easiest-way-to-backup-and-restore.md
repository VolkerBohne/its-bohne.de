---
title: "rdiff-backup: the easiest way to backup (and restore)"
date: 2010-07-11
---
Today I finally managed to set up an incremental backup for my workstation. What do we need? Well, nothing more than **[rdiff-backup](http://rdiff-backup.nongnu.org/)**, an opensource command line tool with all the powers you need.

it just backups. and your last backup is always available 1:1 at the destination (no strange storage formats etc., just dirs and files). Diffs and metadata are stored separately. So if you want a
backup that does its job, is plain, is easy to restore, has no unneccessary features and "just works" than rdiff-backup is the right tool for you.

Here a small bash script I write to accomplish the mission:

    
    #!/bin/bash  
    P=/home/soenke  
    DESTINATION=mydestionationserver.local  
    INCLUDE="  
    $P/Documents  
    $P/.gnupg  
    "  
      
    echo "$INCLUDE" | rdiff-backup --include-filelist-stdin --exclude $P/ $P/ $DESTINATION::/home/soenke/backup  
    
    

In my case "mydestionationserver.local" is a local mediacenter server running Ubuntu and a SSH server. INCLUDE has one backup src per line. P is the prefix dir, in my case my homedir. As you can
see, I'm using a whitelist of dirs/files to be backupped. If you want your full homedir get saved, just use:

    
    rdiff-backup /path/to/src <server>::<destination-dir>
    

Just check the [examples](http://www.gnu.org/savannah-checkouts/non-gnu/rdiff-backup/examples.html) to learn more.
