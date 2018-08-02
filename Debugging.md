Debugging
=========

Cacti users sometimes complain about NaN's in their graphs. Unfortunately, there are several reasons for this result. The following is a step-by-step procedure recommended for debugging.

Check Cacti Log File
--------------------

Your cacti log file should be located at `<path_cacti>/log/cacti.log`. If it is not, see `Settings`, `Paths`. Check for this kind of error:

    CACTID: Host[...] DS[....] WARNING: SNMP timeout detected [500 ms], ignoring host '........'

For "reasonable" timeouts, this may be related to a snmpbulkwalk issue. To change this, see `Settings`, `Poller` and lower the value for `The Maximum SNMP OID's Per SNMP Get Request`. Start at a value of 2 and increase it again, if the poller starts working. (1 or less disables snmpbulkwalk) Some agent's don't have the horsepower to deliver that many OID's at a time. Therefore, we can reduce the number for those older/underpowered devices.

Check Basic Data Gathering
--------------------------

For scripts, run them as cactiuser from cli to check basic functionality. E.g. for a perl script named `your-perl-script.pl` with parameters "p1 p2" under *nix this would look like:

    su - cactiuser
    /full/path/to/perl your-perl-script.pl p1 p2
    ... (check output)

For snmp, snmpget the _exact_ OID you're asking for, using same community string and snmp version as defined within cacti. For an OID of `.1.3.6.1.4.something`, community string of `very-secret` and version 2 for target host `target-host` this would look like

    snmpget -c very-secret -v 2c target-host .1.3.6.1.4.something
    .... (check output)

Check cacti's poller
--------------------

First make sure that crontab always shows poller.php. This program will either call cmd.php, the PHP based poller _or_ cactid, the fast alternative, written in C. Define the poller you're using at `Settings`, `Poller`. Cactid has to be implemented seperately, it does not come with cacti by default.

Now, clear `./log/cacti.log` (or rename it to get a fresh start)

Then, change `Settings`, `Poller Logging Level` to DEBUG for _one_ polling cycle. You may rename this log as well to avoid more stuff added to it with subsequent polling cycles.

Now, find the host/data source in question. The `Host[<id>]` is given numerically, the <id> being a specific number for that host. Find this <id> from the `Devices` menue when editing the host: The url contains a string like

    id=<id>

Check, whether the output is as expected. If not, check your script (e.g. `/full/path/to/perl`). If ok, proceed to next step

This procedure may be replaced by running the poller manually for the failing host only. To do so, you need the <id>, again. If you're using cmd.php, set the DEBUG logging level as defined above and run

    php -q cmd.php <id> <id>

If you're using cactid, you may override logging level when calling the poller:

    ./cactid --verbosity=5 <id> <id>

All output is printed to STDOUT in both cases. This procdure allows for repeated tests without waiting for the next polling interval. And there's no need to manually search for the failing host between hundreds of lines of output.

Check MySQL updating
--------------------

In most cases, this step can be skipped. You may want to return to this step if the next one fails (e.g. no rrdtool update to be found)

From debug log, find the MySQL update statement for that host concerning table `poller_output`. On very rare occasions, this will fail. Copy that sql statement and paste it to a mysql session started from cli. This may as well be done from some tool like phpMyAdmin. Check the sql return code.

Check rrd file updating
-----------------------

Down in the same log, you should find some

    rrdtool update <filename> --template ...

You should find exactly one update statement for each file.

RRD files should be created by the poller. If it does not create them, it will not fill them either. If it does check your `Poller Cache` from Utilities and search for your target. Does the query show up here?

Check rrd file ownership
------------------------

If rrd files were created e.g. with root ownership, a poller running as cactiuser will not be able to update those files

    cd /var/www/html/cacti/rra
    ls -l localhost*
    -rw-r--r--  1 root      root      463824 May 31 12:40 localhost_load_1min_5.rrd
    -rw-r--r--  1 cactiuser cactiuser 155584 Jun  1 17:10 localhost_mem_buffers_3.rrd
    -rw-r--r--  1 cactiuser cactiuser 155584 Jun  1 17:10 localhost_mem_swap_4.rrd
    -rw-r--r--  1 cactiuser cactiuser 155584 Jun  1 17:10 localhost_proc_7.rrd
    -rw-r--r--  1 cactiuser cactiuser 155584 Jun  1 17:10 localhost_users_6.rrd

Run the following command to cure this problem

    chown cactiuser:cactiuser *.rrd

Check rrd file numbers
----------------------

You're perhaps wondering about this step, if the former was ok. But due to data sources MINIMUM and MAXIMUM definitions, it is possible, that valid updates for rrd files are suppressed, because MINIMUM was not reached or MAXIMUM was exceeded.

Assuming, you've got some valid `rrdtool update` in step 3, perform a

    rrdtool fetch <rrd file> AVERAGE

and look at the last 10-20 lines. If you find NaN's there, perform

    rrdtool info <rrd file>

and check the `ds[...].min` and `ds[...].max` entries, e.g.

    ds[loss].min = 0.0000000000e+00
    ds[loss].max = 1.0000000000e+02

In this example, MINIMUM = 0 and MAXIMUM = 100. For a `ds.[...].type=GAUGE` verify, that e.g. the number returned by the script does not exceed `ds[...].MAX` (same holds for MINIMUM, respectively).

If you run into this, not only should you update the data source definition within the Data Template, but also perform a:

    rrdtool tune <rrd file> --maximum <ds-name>:<new ds maximum>

for all existing rrd files belonging to that Data Template.

At this step, it is wise to check `step` and `heartbeat` of the rrd file as well. For standard 300 seconds polling intervals (step=300), it is wise to set `minimal_heartbeat` to 600 seconds. If a single update is missing and the next one occurs in less than 600 seconds from the last one, rrdtool will interpolate the missing update. Thus, gaps are "filled" automatically by interpolation. Be aware of the fact, that this is no "real" data! Again, this must be done in the Data Template itself and by using rrdtool tune for all existing rrd files of this type.

Check rrdtool graph statement
-----------------------------

Last resort would be to check, that the correct data sources are used. Goto `Graph Management` and select your Graph. Enable DEBUG Mode to find the whole `rrdtool graph` statement. You should notice the `DEF` statements. They specify the rrd file and data source to be used. You may check, that all of them are as wanted.

Miscellaneous
-------------

Up to current cacti 0.8.6h, table `poller_output` may increase beyond reasonable size.

This is commonly due to php.ini's memory settings of 8MB default. Change this to at least 64 MB.

To check this, run the following sql from mysql cli (or phpmyadmin or the like)

    select count(*) from poller_output;

If the result is huge, you may get rid of those stuff by

    truncate table poller_output;

As of current SVN code for upcoming cacti 0.9, I saw measures were taken on both issues (memory size, truncating poller_output).

RPM Installation?
-----------------

Most rpm installations will setup the crontab entry now. If you've followed the installation instructions to the letter (which you should always do ;-) ), you may now have two poller running. That's not a good thing, though. Most rpm installations will setup cron in `/etc/cron.d/cacti`

Now check all your crontabs, especially `/etc/crontab` and crontabs of users root and cactiuser. Leave only one poller entry for all of them. Personally, I've chosen `/etc/cron.d/cacti` to avoid problems when updating rpm's. Mosten often, you won't remember this item when updating lots of rpm's, so I felt more secure to put it here. And I've made some slight modifications, see

    shell> vi /etc/cron.d/cacti
    */5 * * * *     cactiuser       /usr/bin/php -q /var/www/html/cacti/poller.php > /var/local/log/poller.log 2>&1

This will produce a file `/var/local/log/poller.log`, which includes some additional informations from each poller's run, such as rrdtool errors. It occupies only some few bytes and will be overwritten each time.

If you're using the crontab of user "cactiuser" instead, this will look like

    shell> crontab -e -u cactiuser
    */5 * * * *     /usr/bin/php -q /var/www/html/cacti/poller.php > /var/local/log/poller.log 2>&1

Not NaN, but 0 (zero) values?
-----------------------------

Pay attention to custom scripts. It is required, that external commands called from there are in the `$PATH` of the cactiuser running the poller. It is therefor recommended to provide `/full/path/to/external/command`

User "criggie" reported an issue with running smartctl. It was complaining "you are not root" so a quick `chmod +s` on the script fixed that problem.
