====== Lock your script (against parallel execution) ======

{{keywords>bash shell scripting mutex locking run-control}}

===== Why lock? =====

Sometimes there's a need to ensure only one copy of a script runs, i.e prevent two or more copies running simultaneously. Imagine an important cronjob doing something very important, which will fail or corrupt data if two copies of the called program were to run at the same time. To prevent this, a form of ''MUTEX'' (**mutual exclusion**) lock is needed.

The basic procedure is simple: The script checks if a specific condition (locking) is present at startup, if yes, it's locked - the scipt doesn't start.

This article describes locking with common UNIX(r) tools. There are other special locking tools available, But they're not standardized, or worse yet, you can't be sure they're present when you want to run your scripts. **A tool designed for specifically for this purpose does the job much better than general purpose code.**

==== Other, special locking tools ====

As told above, a special tool for locking is the preferred solution. Race conditions are avoided, as is the need to work around specific limits.

  * ''flock'': http://www.kernel.org/pub/software/utils/script/flock/
  * ''solo'': http://timkay.com/solo/

===== Choose the locking method =====

The best way to set a global lock condition is the UNIX(r) filesystem. Variables aren't enough, as each process has its own private variable space, but the filesystem is global to all processes (yes, I know about chroots, namespaces, ... special case).
You can &quot;set&quot; several things in the filesystem that can be used as locking indicator:

  * create files
  * update file timestamps
  * create directories


To create a file or set a file timestamp, usually the command touch is used. The following problem is implied:
A locking mechanism checks for the existance of the lockfile, if no lockfile exists, it creates one and continues. Those are **two separate steps**! That means it's **not an atomic operation**. There's a small amount of time between checking and creating, where another instance of the same script could perform locking (because when it checked, the lockfile wasn't there)! In that case you would have 2 instances of the script running, both thinking they are succesfully locked, and can operate without colliding.
Setting the timestamp is similar: One step to check the timespamp, a second step to set the timestamp.

<WRAP center round tip 60%>
__**Conclusion:**__ We need an operation that does the check and the locking in one step.
</WRAP>

A simple way to get that is to create a **lock directory** - with the mkdir command. It will:

    * create a given directory only if it does not exist, and set a successful exit code
    * it will set an unsuccesful exit code if an error occours - for example, if the directory specified already exists


With mkdir it seems, we have our two steps in one simple operation. A (very!) simple locking code might look like this:
<code bash>
if mkdir /var/lock/mylock; then
  echo &quot;Locking succeeded&quot; >&2
else
  echo &quot;Lock failed - exit&quot; >&2
  exit 1
fi
</code>
In case ''mkdir'' reports an error, the script will exit at this point - **the MUTEX did its job!**

//If the directory is removed after setting a successful lock, while the script is still running, the lock is lost. Doing chmod -w for the parent directory containing the lock directory can be done, but it is not atomic. Maybe a while loop checking continously for the existence of the lock in the background and sending a signal such as USR1, if the directory is not found, can be done. The signal would need to be trapped. I am sure there there is a better solution than this suggestion// --- //[[sunny_delhi18@yahoo.com|sn18]] 2009/12/19 08:24//

**Note:** While perusing the Internet, I found some people asking if the ''mkdir'' method works &quot;on all filesystems&quot;. Well, let's say it should. The syscall under ''mkdir'' is guarenteed to work atomicly in all cases, at least on Unices. Two examples of problems are NFS filesystems and filesystems on cluster servers. With those two scenarios, dependencies exist related to the mount options and implementation. However, I successfully use this simple method on an Oracle OCFS2 filesystem in a 4-node cluster environment. So let's just say &quot;it should work under normal conditions&quot;.

Another atomic method is setting the ''noclobber'' shell option (''set -C''). That will cause redirection to fail, if the file the redirection points to already exists (using diverse ''open()'' methods). Need to write a code example here.

<code bash>

if ( set -o noclobber; echo &quot;locked&quot; > &quot;$lockfile&quot;) 2> /dev/null; then
  trap 'rm -f &quot;$lockfile&quot;; exit $?' INT TERM EXIT
  echo &quot;Locking succeeded&quot; >&2
  rm -f &quot;$lockfile&quot;
else
  echo &quot;Lock failed - exit&quot; >&2
  exit 1
fi

</code>

Another explanation of this basic pattern using ''set -C'' can be found [[http://pubs.opengroup.org/onlinepubs/9699919799/xrat/V4_xcu_chap02.html#tag_23_02_07 | here]].
===== An example =====

This code was taken from a production grade script that controls PISG to create statistical pages from my IRC logfiles.
There are some differences compared to the very simple example above:

  * the locking stores the process ID of the locked instance
  * if a lock fails, the script tries to find out if the locked instance still is active (unreliable!)
  * traps are created to automatically remove the lock when the script terminates, or is killed


Details on how the script is killed aren't given, only code relevant to the locking process is shown:
<code bash>
#!/bin/bash

# lock dirs/files
LOCKDIR=&quot;/tmp/statsgen-lock&quot;
PIDFILE=&quot;${LOCKDIR}/PID&quot;

# exit codes and text
ENO_SUCCESS=0; ETXT[0]=&quot;ENO_SUCCESS&quot;
ENO_GENERAL=1; ETXT[1]=&quot;ENO_GENERAL&quot;
ENO_LOCKFAIL=2; ETXT[2]=&quot;ENO_LOCKFAIL&quot;
ENO_RECVSIG=3; ETXT[3]=&quot;ENO_RECVSIG&quot;

###
### start locking attempt
###

trap 'ECODE=$?; echo &quot;[statsgen] Exit: ${ETXT[ECODE]}($ECODE)&quot; >&2' 0
echo -n &quot;[statsgen] Locking: &quot; >&2

if mkdir &quot;${LOCKDIR}&quot; &>/dev/null; then

    # lock succeeded, install signal handlers before storing the PID just in case 
    # storing the PID fails
    trap 'ECODE=$?;
          echo &quot;[statsgen] Removing lock. Exit: ${ETXT[ECODE]}($ECODE)&quot; >&2
          rm -rf &quot;${LOCKDIR}&quot;' 0
    echo &quot;$$&quot; >&quot;${PIDFILE}&quot; 
    # the following handler will exit the script upon receiving these signals
    # the trap on &quot;0&quot; (EXIT) from above will be triggered by this trap's &quot;exit&quot; command!
    trap 'echo &quot;[statsgen] Killed by a signal.&quot; >&2
          exit ${ENO_RECVSIG}' 1 2 3 15
    echo &quot;success, installed signal handlers&quot;

else

    # lock failed, check if the other PID is alive
    OTHERPID=&quot;$(cat &quot;${PIDFILE}&quot;)&quot;

    # if cat isn't able to read the file, another instance is probably
    # about to remove the lock -- exit, we're *still* locked
    #  Thanks to Grzegorz Wierzowiecki for pointing out this race condition on
    #  http://wiki.grzegorz.wierzowiecki.pl/code:mutex-in-bash
    if [ $? != 0 ]; then
      echo &quot;lock failed, PID ${OTHERPID} is active&quot; >&2
      exit ${ENO_LOCKFAIL}
    fi

    if ! kill -0 $OTHERPID &>/dev/null; then
        # lock is stale, remove it and restart
        echo &quot;removing stale lock of nonexistant PID ${OTHERPID}&quot; >&2
        rm -rf &quot;${LOCKDIR}&quot;
        echo &quot;[statsgen] restarting myself&quot; >&2
        exec &quot;$0&quot; &quot;$@&quot;
    else
        # lock is valid and OTHERPID is active - exit, we're locked!
        echo &quot;lock failed, PID ${OTHERPID} is active&quot; >&2
        exit ${ENO_LOCKFAIL}
    fi

fi
</code>

===== Related links =====

  * [[http://mywiki.wooledge.org/BashFAQ/045 | BashFAQ/045]]
  * [[http://wiki.grzegorz.wierzowiecki.pl/code:mutex-in-bash | Implementation of a shell locking utility]]
