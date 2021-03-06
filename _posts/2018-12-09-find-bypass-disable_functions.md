---
layout: post
title: Searching systematically for PHP disable_functions bypasses
date: 2018-12-09 13:00:00
categories: posts
en: true
description: Some ideas about how to extract hidden parameters in PHP functions and how to find potential bypasses
keywords: "disable_function, chankro, bypass, RedTeam, Red Team, zend_parse_parameters"
authors:
    - X-C3LL
---

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
In the past days a vulnerability was discovered in __imap_open__ function ([CVE-2018-19518](https://www.cvedetails.com/cve/cve-2018-19518)). The main problem with this issue is that a local user can abuse this vulnerability to bypass some restrictions in hardened servers, and execute OS commands, because this function usually is allowed. This kind of bypass is similar to the one based in ShellShock: someone can inject OS commands inside functions not banned by disable_functions. I wanted to find a way to discover similar bypasses in an automated way. 


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
The main idea is to split the problem in few tasks:

1. Extract the parameters expected by every PHP function
2. Execute and trace calls to every function with the right parameters
3. Search in the trace for potential dangerous calls (for example, an execve)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Of course this is a naive approach, but this same work can be reused to help me while fuzzing... so is not an useless effort I guess __:)__.

## 0x01 Extracting the parameters 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
The first thing we have to deal is a realiable way to guess or identify rightly the parameters used for every function. Of course we can parse the information from public documentation (like PHP.net for example), but some functions can have hidden parameters not documented or the documentation just name them as "mixed". This point is important because if the function is not called correctly then our trace will miss potential calls to dangerous functions/syscalls. There are different ways to accomplish this identification, each one with its advantage and downsides, so we need to combine all of them to discover the maximum number of parameters (and __what type__ are them).


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
One extremely lazy approach can be to use the class [ReflectionFunction](http://php.net/manual/es/class.reflectionfunction.php). Through this simple class we can obtain (from the own PHP) the name and parameters used by every function available, but it has the disadvantage of not know what type is really. We can discrimiante between "strings" and "arrays" only. An example:

```php
<?php
//Get all defined functions in a lazy way
$all = get_defined_functions();
//Discard annoying functions
//From https://github.com/nikic/Phuzzy
$bad = array('sleep', 'usleep', 'time_nanosleep', 'time_sleep_until','pcntl_sigwaitinfo', 'pcntl_sigtimedwait',
                        'readline', 'readline_read_history','dns_get_record','posix_kill', 'pcntl_alarm','set_magic_quotes_runtime','readline_callback_handler_install',);
$all = array_diff($all["internal"], $bad);

foreach ($all as $function) {
    $parameters = "$function ";
    $f = new ReflectionFunction($function);
    foreach ($f->getParameters() as $param) {
        if ($param->isArray()) {
            $parameters .=  "ARRAY ";
        } else {
            $parameters .= "STRING ";
        }
    }
    echo substr($parameters, 0, -1);
    echo "\n";
}
?>
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
This code generates a list of functions that we can parse later to generate the tests where we are going to trace calls:
```
json_last_error_msg
spl_classes
spl_autoload STRING STRING
spl_autoload_extensions STRING
spl_autoload_register STRING STRING STRING
spl_autoload_unregister STRING
spl_autoload_functions
spl_autoload_call STRING
class_parents STRING STRING
class_implements STRING STRING
class_uses STRING STRING
spl_object_hash STRING
spl_object_id STRING
iterator_to_array STRING STRING
iterator_count STRING
iterator_apply STRING STRING ARRAY
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
A far better approach can be to hook the function used internally by PHP to parse the parameters, as it is used in the article "[Hunting for hidden parameters within PHP built-in functions (using frida)](http://www.libnex.org/blog/huntingforhiddenparameterswithinphpbuilt-infunctionsusingfrida)". The authors use FRIDA to hook the function __zend_parse_parameters__ and parse the pattern used to validate the parameters passed (if you want to know a bit more about FRIDA feel free to check my article [Hacking a game to learn FRIDA basics (Pwn Adventure 3)](https://x-c3ll.github.io/posts/Frida-Pwn-Adventure-3/)). This approach is one of the best because with the patterns we can know what type is expecting, but has a small downside: this function is deprecated, so in the future will be not used.


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
The internals of PHP 7 are a bit different from PHP 5 and some APIs, like the one used for parameter parsing, are affected by those changes. The old API is string-based meanwhile the new API (used by PHP 7) is based in macros. Where we had the zend_parse_parameters function now we have the macro __ZEND_PARSE_PARAMETERS_START__ and its family. To know more about how PHP parses the arguments feel free to check this amazing article from phpinternals.net: [Zend Parameter Parsing (ZPP API)](https://phpinternals.net/categories/zend_parameter_parsing). Essentially now we can not just say "Hey FRIDA! do your magic!" hooking a master key function.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
If we remember, in our article [Improving PHP extensions as a persistence method](https://x-c3ll.github.io/posts/PHP-extension-backdoor/) we saw that the md5 function was using the new ZPP API to parse the parameters:

```c
ZEND_PARSE_PARAMETERS_START(1, 2)
		Z_PARAM_STR(arg)
		Z_PARAM_OPTIONAL
		Z_PARAM_BOOL(raw_output)
ZEND_PARSE_PARAMETERS_END();
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
In order to extract the parameters, a shabby (and effective) approach is to compile PHP with symbols and use a script in GDB to parse the information. Of course there are better ways than using GDB, but recently I had to script some helpers for debugging PHP in GDB that fit really well in this other task, so I am not ashamed for reusing them. Let's compile the last PHP version:

```bash
cd /tmp
wget http://am1.php.net/distributions/php-$(wget -qO- http://php.net/downloads.php | grep -m 1 h3 | cut -d '"' -f 2 | cut -d "v" -f 2).tar.gz
tar xvf php*.tar.gz
rm php*.tar.gz
cd php*
./configure CFLAGS="-g -O0"
make -j10
sudo make install
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Our script for GDB will work as follows:
1. Execute  `list functionName`
2. If __ZEND_PARSE_PARAMETERS_END__ is not present, increment the number of lines to show in list and try again
3. If it is present, then break the loop and extract the lines between the macros __..._START__ and __..._END__
4. Parse the parameters between them

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
This flow of actions can be summarized in this simple snippet:
```python
# When I do things like this I feel really bad
# Satanism courtesy of @TheXC3LL

class zifArgs(gdb.Command):
    "Show PHP parameters used by a function when it uses PHP 7 ZPP API. Symbols needed."

    def __init__(self):
        super (zifArgs, self).__init__("zifargs", gdb.COMMAND_SUPPORT, gdb.COMPLETE_NONE, True)

    def invoke (self, arg, from_tty):
        size = 10
        while True:
            try:
                sourceLines = gdb.execute("list zif_" + arg, to_string=True)
            except:
                try:
                    sourceLines = gdb.execute("list php_" + arg, to_string=True)
                except:
                    try:
                        sourceLines = gdb.execute("list php_if_" + arg, to_string=True)
                    except:
                        print("\033[31m\033[1mFunction " + arg + " not defined!\033[0m")
                        return
            if "ZEND_PARSE_PARAMETERS_END" not in sourceLines:
                size += 10
                gdb.execute("set listsize " + str(size))
            else:
                gdb.execute("set listsize 10")
                break
        try:
            chunk = sourceLines[sourceLines.index("_START"):sourceLines.rindex("_END")].split("\n")
        except:
            print("\033[31m\033[1mParameters not found. Try zifargs_old <function>\033[0m")
            return
        params = []
        for x in chunk:
            if "Z_PARAM_ARRAY" in x:
                params.append("\033[31mARRAY")
            if "Z_PARAM_BOOL" in x:
                params.append("\033[32mBOOL")
            if "Z_PARAM_FUNC" in x:
                params.append("\033[33mCALLABLE")
            if "Z_PARAM_DOUBLE" in x:
                params.append("\033[34mDOUBLE")
            if "Z_PARAM_LONG" in x or "Z_PARAM_STRICT_LONG" in x:
                params.append("\033[36mLONG")
            if "Z_PARAM_ZVAL" in x:
                params.append("\033[37mMIXED")
            if "Z_PARAM_OBJECT" in x:
                params.append("\033[38mOBJECT")
            if "Z_PARAM_RESOURCE" in x:
                params.append("\033[39mRESOURCE")
            if "Z_PARAM_STR" in x:
                params.append("\033[35mSTRING")
            if "Z_PARAM_CLASS" in x:
                params.append("\033[37mCLASS")
            if "Z_PARAM_PATH" in x:
                params.append("\033[31mPATH")
            if "Z_PARAM_OPTIONAL" in x:
                params.append("\033[37mOPTIONAL")
        if len(params) == 0:
            print("\033[31m\033[1mParameters not found. Try zifargs_old <function> or zifargs_error <function>\033[0m")
            return
        print("\033[1m"+' '.join(params) + "\033[0m")

zifArgs()
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
The results are pretty good (everything after "OPTIONAL" is optional):
```
pwndbg: loaded 171 commands. Type pwndbg [filter] for a list.
pwndbg: created $rebase, $ida gdb functions (can be used with print/break)
[+] Stupid GDB Helper for PHP loaded! (by @TheXC3LL)
Reading symbols from php...done.
pwndbg> zifargs md5
STRING OPTIONAL BOOL
pwndbg> zifargs time
OPTIONAL LONG BOOL
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
As we said before every technique has its own downside, and approach such naive can fail:
```
pwndbg> zifargs array_map
CALLABLE
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
The function [array_map](http://php.net/manual/es/function.array-map.php) has an array as second parameter and our snippet could not detect it. Another technique to extract the parameters is to parse the descriptive errors generated by some functions in PHP. For example, array_map, will tell you how many parameters need:

```
psyconauta@insulatergum:~/research/php/|
⇒  php -r 'array_map();'

Warning: array_map() expects at least 2 parameters, 0 given in Command line code on line 1
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
And if we set the two parameters as strings he will warning about what kind of parameter is expecting:
```
psyconauta@insulatergum:~/research/php/
⇒  php -r 'array_map("aaa","bbb");'

Warning: array_map() expects parameter 1 to be a valid callback, function 'aaa' not found or invalid function name in Command line code on line 1
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
So we can use this errors to infer the parameters:
1. Call the function empty
2. Check errors to see how many parameteres need
3. Fill them with strings
4. Parse the expected type from warning
5. Change the parameter for one of the right type
6. Repeat from 4

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
I implemented another extremely shabby command for GDB to take this task:
```python
# Don't let me use gdb when I am drunk
# Sorry for this piece of code :(

class zifArgsError(gdb.Command):
    "Tries to infer parameters from PHP errors"

    def __init__(self):
        super(zifArgsError, self).__init__("zifargs_error", gdb.COMMAND_SUPPORT, gdb.COMPLETE_NONE,True)

    def invoke(self, arg, from_tty):
        payload = "<?php " + arg + "();?>"
        file = open("/tmp/.zifargs", "w")
        file.write(payload)
        file.close()
        try:
            output = str(subprocess.check_output("php /tmp/.zifargs 2>&1", shell=True))
        except:
            print("\033[31m\033[1mFunction " + arg + " not defined!\033[0m")
            return
        try:
            number = output[output.index("at least ")+9:output.index("at least ")+10]
        except:
            number = output[output.index("exactly ")+8:output.index("exactly")+9]
        print("\033[33m\033[1m" + arg+ "(\033[31m" + number + "\033[33m): \033[0m")
        params = []
        infered = []
        i = 0
        while True:
            payload = "<?php " + arg + "("
            for x in range(0,int(number)-len(params)):
                params.append("'aaa'")
            payload += ','.join(params) + "); ?>"
            file = open("/tmp/.zifargs", "w")
            file.write(payload)
            file.close()
            output = str(subprocess.check_output("php /tmp/.zifargs 2>&1", shell=True))
            #print(output)
            if "," in output:
                separator = ","
            elif " file " in output:
                params[i] = "/etc/passwd" # Don't run this as root, for the god sake.
                infered.append("\033[31mPATH")
                i +=1
            elif " in " in output:
                separator = " in "

            try:
                dataType = output[:output.rindex(separator)]
                dataType = dataType[dataType.rindex(" ")+1:].lower()
                if dataType == "array":
                    params[i] = "array('a')"
                    infered.append("\033[31mARRAY")
                if dataType == "callback":
                    params[i] = "'var_dump'"
                    infered.append("\033[33mCALLABLE")
                if dataType == "int":
                    params[i] = "1337"
                    infered.append("\033[36mINTEGER")
                i += 1
                #print(params)
            except:
                if len(infered) > 0:
                    print("\033[1m" + ' '.join(infered) + "\033[0m")
                    return
                else:
                    print("\033[31m\033[1mCould not retrieve parameters from " + arg + "\033[0m")
                    return
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Try it with array_map:
```
pwndbg> zifargs_error array_map
array_map(2):
CALLABLE ARRAY
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Until now we explained different techniques that can be combined to obtain automatically the parameters needed to run correctly (or almost) every PHP function. As I said before, this techniques can be used too for fuzzing in order to reach additional codepaths, or to run disregarded fuzzing instances. Let's see now how we can use the information gathered.

## 0x02 Getting and analyzing traces
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
The simplest way to obtain a trace is using well-known tools like strace and ltrace. With few lines of bash we can parse the logs generated in the previous step with the function name and parameters, run the tracer and save the logs to a file. Let's analyze the log generated by the mail() function for example:

```
⇒  strace -f /usr/bin/php -r 'mail("aaa","aaa","aaa","aaa");' 2>&1 | grep exe
execve("/usr/bin/php", ["/usr/bin/php", "-r", "mail(\"aaa\",\"aaa\",\"aaa\",\"aaa\");"], [/* 28 vars */]) = 0
[pid   471] execve("/bin/sh", ["sh", "-c", "/usr/sbin/sendmail -t -i "], [/* 28 vars */] <unfinished ...>
[pid   471] <... execve resumed> )      = 0
[pid   472] execve("/usr/sbin/sendmail", ["/usr/sbin/sendmail", "-t", "-i"], [/* 28 vars */]) = -1 ENOENT (No such file or directory)
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Did you see that __execve()__ with the sendmail? That means that this function can be abused in order to bypass disable_functions (as far as we have allowed putenv to manipulate the LD_PRELOAD ). Indeed this is how [CHANKRO](https://github.com/TarlogicSecurity/Chankro) works: if we can set environment variables, we can set LD_PRELOAD to load a malicious file when an external binary (as we see in the log trace) is called. Just run the script, wait, and execute few greps to check for juicy calls. __Easy peasy :)__

## 0x03 Final words
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Automate the parameter extraction can be a bit tricky so I decide to write this article to contribute my grain of sand. Months ago I read [this article](http://www.libnex.org/blog/huntingforhiddenparameterswithinphpbuilt-infunctionsusingfrida) where FRIDA is used to hook zend_parse_parameters and I wanted to complete a bit more this information for new-comers on PHP internals. The imap_open() vulnerability was a perfect excuse to write about the topic __:)__.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
If you find useful this article, or wanna point me to an error or a typo, feel free to contact me at twitter [@TheXC3LL](https://twitter.com/TheXC3LL).
