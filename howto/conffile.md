====== Config files for your script ======

{{keywords>bash shell scripting config files include configuration}}

===== General =====

For this task, you don't have to write large parser routines (unless you want it 100% secure or you want a special file syntax) - you can use the Bash source command. The file to be sourced should be formated in key=&quot;value&quot; format, otherwise bash will try to interpret commands:

<code>
#!/bin/bash
echo &quot;Reading config....&quot; >&2
source /etc/cool.cfg
echo &quot;Config for the username: $cool_username&quot; >&2
echo &quot;Config for the target host: $cool_host&quot; >&2
</code>

So, where do these variables come from? If everything works fine, they are defined in /etc/cool.cfg which is a file that's sourced into the current script or shell. Note: this is **not** the same as executing this file as a script!
The sourced file most likely contains something like:

<code>
cool_username=&quot;guest&quot;
cool_host=&quot;foo.example.com&quot;
</code>

These are normal statements understood by Bash, nothing special. Of course (and, a big disadvantage under normal circumstances) the sourced file can contain **everything** that Bash understands, including malicious code!

The ''source'' command also is available under the name ''.'' (dot). The usage of the dot is identical:

<code>
#!/bin/bash
echo &quot;Reading config....&quot; >&2
. /etc/cool.cfg #note the space between the dot and the leading slash of /etc.cfg
echo &quot;Config for the username: $cool_username&quot; >&2
echo &quot;Config for the target host: $cool_host&quot; >&2
</code>

===== Per-user configs =====

There's also a way to provide a system-wide config file in /etc and a custom config in ~/(user's home) to override system-wide defaults. In the following example, the if/then construct is used to check for the existance of a user-specific config:

<code>
#!/bin/bash
echo &quot;Reading system-wide config....&quot; >&2
. /etc/cool.cfg
if [ -r ~/.coolrc ]; then
  echo &quot;Reading user config....&quot; >&2
  . ~/.coolrc
fi
</code>



===== Secure it =====

As mentioned earlier, the sourced file can contain anything a Bash script can. Essentially, it **is** an included Bash script. That creates security issues. A malicicios person can &quot;execute&quot; arbitrary code when your script is sourcing its config file.
You might want to allow only constructs in the form ''NAME=VALUE'' in that file (variable assignment syntax) and maybe comments (though technically, comments are unimportant).
Imagine the following &quot;config file&quot;, containing some malicious code:

<code>
# cool config file for my even cooler script
username=god_only_knows
hostname=www.example.com
password=secret ; echo rm -rf ~/*
parameter=foobar && echo &quot;You've bene pwned!&quot;;
# hey look, weird code follows...
echo &quot;I am the skull virus...&quot;
echo rm -fr ~/*
mailto=netadmin@example.com
</code>

You don't want these ''echo''-commands (which could be any other commands!) to be executed. One way to be a bit safer is to filter only the constructs you want, write the filtered results to a new file and source the new file. We also need to be sure something nefarious hasn't been added to the end of one of our name=value parameters, perhaps using ; or && command separators. In those cases, perhaps it is simplest to just ignore the line entirely. Egrep (''grep -E'') will help us here, it filters by description:

<code>
#!/bin/bash
configfile='/etc/cool.cfg'
configfile_secured='/tmp/cool.cfg'

# check if the file contains something we don't want
if egrep -q -v '^#|^[^ ]*=[^;]*' &quot;$configfile&quot;; then
  echo &quot;Config file is unclean, cleaning it...&quot; >&2
  # filter the original to a new file
  egrep '^#|^[^ ]*=[^;&]*'  &quot;$configfile&quot; > &quot;$configfile_secured&quot;
  configfile=&quot;$configfile_secured&quot;
fi

# now source it, either the original or the filtered variant
source &quot;$configfile&quot;
</code>
**__To make clear what it does:__** egrep checks if the file contains something we don't want, if yes, egrep filters the file and writes the filtered contents to a new file. If done, the original file name is changed to the name stored in the variable ''configfile''. The file named by that variable is sourced, as if it were the original file.

