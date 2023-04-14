# mirror
Mirror synchronises contents of sourcedir into  destdir which can be on a remote machine, a docker container, or on a docker image. Mirror wraps the `rsync` command to give it a simpler interface to ease mirror and diffdir operations. 

## Description

The `mirror` command synchronises contents of sourcedir into destdir
where either sourcedir or destdir may be a directory on a remote 
machine, a docker container, a docker image or on a docker volume.
Destination is updated to match source, including deleting 
files if necessary.

Because mirror does delete files if necessary it by default 
gives a warning, and asks you to continue or not:
	
>We STRONGLY advice to do a 'dry-run' first (option -n), because 
>mirroring can destroy the destination if you are not careful!<br>
>Are you sure you want to continue? <y/N> y

Mirror wraps the `rsync` command to give it a simpler interface
to ease mirroring operations. The name mirror describes
better the functionality because rsync does mirroring,
and cannot do bi-directional sync like many cloud storage
solutions nowadays do.

The `diffdir` command is just `mirror --dry-run` to quickly find
the difference between sourcedir into  destdir. 

Both `mirror` and `diffdir` list the changes applied to the destdir. 
Using the `-i` option more details of the changes are listed such as 
attribute changes. With the `-q` option on `mirror` the operation 
is done quietly without anything reported. 
  
## Installation 


The `mirror` and `diffdir` commands are simple scripts in `bash`, so you can easily fetch it for a specific version from github:


    VERSION="v1.1.0" 
    INSTALL_DIR=/usr/local/bin # make sure INSTALL_DIR is in your PATH environment variable
    DOWNLOAD_URL="https://raw.githubusercontent.com/harcokuppens/mirror/${VERSION}/bin/"
    curl -Lo ${INSTALL_DIR}/mirror  "$DOWNLOAD_URL/mirror"
    chmod a+x ${INSTALL_DIR}/mirror
    curl -Lo ${INSTALL_DIR}/diffdir  "$DOWNLOAD_URL/diffdir"
    chmod a+x ${INSTALL_DIR}/diffdir
    
      
Requirements:  

* `bash` shell
* `rsync` tool
* `ssh` tool, only needed for mirroring to/from remote machine
* `docker` tool, only needed for mirroring to/from docker image/container/volume

Note when mirroring into a docker container/image, then the container/image must have `rsync`
installed. When mirroring into a docker image then the image also must have `sh`, `exec`,
and `sleep` installed.

      
For Windows you could use WSL to run the `mirror` utility. You can also install the `mirror` utility with [cygwin](https://cygwin.org) to get a bash shell to run the script.     
	    
	    
## Usage 
 

    mirror
      local mode
         mirror [OPTIONS] SOURCEDIR                               DESTDIR
      ssh mode    
         mirror [OPTIONS] [ssh://]HOSTNAME:SOURCEDIR              DESTDIR
         mirror [OPTIONS] SOURCEDIR                               [ssh://]HOSTNAME:DESTDIR
      docker mode    
         mirror [OPTIONS] docker://CONTAINERNAME:SOURCEDIR        DESTDIR
         mirror [OPTIONS] SOURCEDIR                               docker://CONTAINERNAME:DESTDIR
         mirror [OPTIONS] docker-img://IMAGENAME[:TAG]:SOURCEDIR  DESTDIR
         mirror [OPTIONS] SOURCEDIR                               docker-img://IMAGENAME[:TAG]:DESTDIR
         mirror [OPTIONS] docker-vol://VOLUMENAME[:SOURCEDIR]     DESTDIR
         mirror [OPTIONS] SOURCEDIR                               docker-vol://VOLUMENAME[:DESTDIR]       
      
    diffdir 
         diffdir == mirror --dry-run
      
    More documentation
      * mirror --help
      * https://github.com/harcokuppens/mirror

## Options
     
    -a            to preserve all; meaning preserving also user,group,other and their permissions
                  Without this option the only attributes preserved are the modification times,
                  which often suffices for a single user computer
    --debug       Print mirror's debug messages.                 
    --dry-run     dry runs the mirror. Which means it doesn't do any file transfers, 
                  instead it will just report the actions it would have taken.
    --file        mirror a file instead of a directory              
    -f PATTERN    
    --filter PATTERN
                  applies a filter on what to mirror; with '- path' you can exclude a path, 
                  and with '+ subpath' you can include a subpath from the excluded path again. 
                  Paths starting with '/' are seen as absolute root-based paths, and
                  without are seen as relative paths which are matched in each subdir. 
                  For more details see the man page of rsync. 
    -i            Itemize changes: output a change-summary for all updates on destination to let it
                  mirror the source. Each item's change-summary is represented with the format:

                      YXcstpog
                      
                  where
                      Y is updatetype letter:  
                          .(same) c(change/creation) <(transfer to sourcedir) > (transfer to destdir)
                      X is filetype letter:  
                          f(ile), d(irectory), an L for a symlink, a D for a device, and a S for a 
                          special file (e.g. named sockets and fifos).
                      cstpog are possible letters which specify what is different:
                          c(hecksum) s(size) t(ime modification) p(ermissions) o(wner) g(roup)
                          per letter L we have 3 cases: 
                            a) on change the letter 'L' is printed
                            b) no change then '.' is printed
                            c) on a new file or directory '+' is printed
                  for more info: see man page of rsync      
                  The '-i' option disables the implicit '-v' option. To enable both
                  then give both options explicit.
    --keep-dest-only      
                  do not delete files on destination which are not in source   
    --no-git      excludes .git folder from mirroring; same as:  -f '- .git
    --no-warn     do not warn about the risks, but directly do the mirror
    -t TRIGGER   
    --trigger TRIGGER   
                  The trigger option determines how it is determined whether a file must be
                  synced or not. We have the following triggers: 
                    - modsize: either modification or size change triggers sync (default)
                    - sizeonly: only size change triggers sync 
                    - checksum: difference in local and remote checksum triggers sync
                    - always: always triggers sync
                  Note: even when doing always a sync the rsync algorithm does a smart
                        delta sync, meaning it only needs to transfer the difference 
                        between a local and remote file to sync them.
    -u, --update  skip files that are newer on the destination         
    -v or -vv     This  option increases the amount of information you are given during the transfer.  
                  By default, mirror take the -v option implicitly enabled. 
                  A single -v will give you information about what files are being transferred and a 
                  brief summary at the end.  Two -v options will give you information on what files are 
                  being skipped and slightly more information at the end.              
    -q            Quiet. Mirror without reporting anything.  
    --rsync-options RSYNC_OPTIONS  
                  add extra rsync options; see man page of rsync

## Notes
    
DESTDIR and SRCDIR are taken relative from the default remote directory,
unless they start with a '/' then they are taken as absolute paths from the
root directory on the remote.

Mirror is wrapper around `rsync`. In rsync you can specify a remote shell for
either source or destination.  The remote shell is used to start rsync on the
remote side with which the local rsync communicates to via `stdin`/`stdout` to do the sync. 

So it is required that `rsync` is installed on the remote.
For a remote host using ssh the remove shell is `ssh`. For a remote docker 
container the the remote shell is `docker exec -i`.

For a remote using `docker://` we use a docker container as remote,
so we use the containername instead of hostname to specify the remote location.

For a remote using `docker-img://` we use a docker image as remote, 
but because we can only rsync into a running container a dummy container
is created for the image. After mirroring on the container is done a 
modified image is extracted from the container. (using `docker commit`)

For a remote using `docker-vol://` we use a docker volume as remote, 
but because we can only rsync into a running container a dummy container
is created which mounts the volume. After mirroring on the volume mount 
location in the dummy container is done, we can stop and delete the 
dummy container. The volume, which is independent of the dummy container,
is then changed.
