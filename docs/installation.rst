
Prerequirements
------------------------------------

A recent enough version of the Apache HTTP server is required. 2.2.6 or later
should be used. In addition, the apr-util library needs to be 1.3.0 or newer.
This is because the DBD database pool functionality was developed mainly
between 2006 and 2007, and reached production quality at the time.



Install the PostgreSQL ip4r datatype
------------------------------------

* install the postgresql-ip4r package

  * project page: http://ip4r.projects.postgresql.org/
  * openSUSE/SLE rpm package: 
    http://download.opensuse.org/repositories/server:/database:/postgresql/
  * The Debian package is called postgresql-8.3-ip4r.
  * Gentoo portage overlay:
    http://github.com/ramereth/ramereth-overlay/tree

* install the datatype, done by executing sql statements from the shipped file::

    su - postgres
    psql -f /usr/share/postgresql-ip4r/ip4r.sql template1

  ("template1" means that all databases that are created later will have the datatype.
  To install it onto an existing database, use your database name instead.)

  (It is normal to see a a good screenful of out printed out by psql.)


Create the database table
------------------------------------

* it is assumed that a database exist already.

* execute the sql statements from asn.sql (shipping with mod_asn):
    psql -U <dbuser> -f asn.sql <dbname>

  In this example, a table named pfx2asn would be created in the 
  <dbname>. database.

  (It is normal to see a "NOTICE" printed out by psql.)


Config file for the import script
------------------------------------

If you happen to have a MirrorBrain setup, you'll have a configuration file
named /etc/mirrorbrain.conf, which is found and used by the asn_import script.
No further configuration is needed. The MirrorBrain instance can be selected
with the -b commandline option, if required.

Alternatively, you need to create config file with the database connection
info, named /etc/asn_import.conf, looking like this:

    [general]
    user = database_user
    password = database_password
    host = database_server
    dbname = name_of_database


Load the database with routing data
------------------------------------

* download the data and import it into the database:

    asn_get_routeviews.py | asn_import.py

* this will take a few minutes. The routing data is 900 MB uncompressed
  (beginning of 2009). 

* the same command can also be used to update the database later, with fresh
  routeviews data. Just run it again. It can be done in production while the
  database is in active use.

* you should set up this script to run once per week by cron, so the database
  keeps updated regularly.
  Here's an example for a setup in conjunction with MirrorBrain::

    # update ASN data three times a week
    35 2 * * mon,wed,fri   mirrorbrain  for i in $(mb instances); do \
                                asn_get_routeviews | asn_import -b $i; done

  The data is downloaded to the user's home directory in this case. Make sure the
  script runs in a directory where other users don't have write permissions.



Build the Apache module
------------------------------------

* compile, install, and enable it:
    apxs2 -ci mod_asn.c

* or install a binary package from here:

  * openSUSE/SLE:
    http://download.opensuse.org/repositories/Apache:/MirrorBrain/ 
  * Debian/Ubuntu:
    http://download.opensuse.org/repositories/Apache:/MirrorBrain/
  * Gentoo portage overlay:
    http://github.com/ramereth/ramereth-overlay/tree

* and enable it::
    a2enmod asn

Configure Apache / mod_dbd
------------------------------------

* mod_dbd is the database adapter that provides a connection pool.
  Enable it, e.g.::

    a2enmod dbd

* Put the following configuration into server-wide context::

    # whis configures the connection pool.
    # for prefork, this configuration is inactive. prefork simply uses 1
    # connection per child.
    <IfModule !prefork.c>
            DBDMin  0
            DBDMax  32
            DBDKeep 4
            DBDExptime 10
    </IfModule>

* configure the database driver.

  Put this configuration into server-wide OR vhost context. Make the file
  chmod 0640, owned root:root because it will contain the database password::

    DBDriver pgsql
    # note that the connection string (which is passed straight through to
    # PGconnectdb in this case) looks slightly different - pass vs. password
    DBDParams "host=localhost user=mb password=12345 dbname=mb_samba connect_timeout=15"


Troubleshooting
------------------------------------

If Apache doesn't start, or anything else seems wrong, make sure to check
Apache's error_log. It usually points into the right direction.

A general note about Apache configuration which might be in order. With most
config directives, it is important to pay attention where to put them - the
order does not matter, but the context does. There is the concept of directory
contexts and vhost contexts, which must not be overlooked.
Things can be "global", or inside a <VirtualHost> container, or within a
<Directory> container.

This matters because Apache applies the config recursively onto subdirectories,
and for each request it does a "merge" of possibly overlapping directives.
Settings in vhost context are merged only when the server forks, while settings
in directory context are merged for each request. This is also the reason why
some of mod_asn's config directives are programmed to be used in one or the
other context, for performance reasons.

The install docs you are reading attempt to always point out in which context
the directives belong.



Configure mod_asn
------------------------------------

* simply set "ASLookup On" in the directory context where you want it.
* the shipped config (mod_asn.conf) shows an example.

* set "ASSetHeaders Off" if you don't want the data to be added to the HTTP
  response headers.

* you may use the ASLookupQuery directive (server-wide context) to define a
  custom SQL query. The compiled in default is:
  SELECT pfx, asn FROM pfx2asn WHERE pfx >>= ip4r(%s) ORDER BY ip4r_size(pfx) LIMIT 1

* the client IP address is the one that the requests originates from. But if
  mod_asn is running behind a frontend server, the frontend can pass the IP via
  a header and mod_asn can look at the header instead, and you can configure it
  to look at that header like this::

    ASIPHeader X-Forwarded-For

* alternatively, if you want to use mod_rewrite you can also make mod_asn look
  at a variable in Apache's subprocess environment::

    ASIPEnvvar CLIENT_IP

* "ASLookupDebug On" can be set to switch on debug logging. It can be set per
  directory.



Testing
------------------------------------


Once mod_asn is configured, you should be able to verify that it works by doing
some arbitrary request and looking at the response::

     % curl -sI 'http://download.opensuse.org/distribution/11.1/iso/openSUSE-11.1-Addon-Lang-i586.iso' 
    HTTP/1.1 302 Found
    Date: Fri, 26 Jun 2009 22:35:50 GMT
    Server: Apache/2.2.11 (Linux/SUSE)
    X-Prefix: 87.78.0.0/15
    X-AS: 8422
    X-MirrorBrain-Mirror: ftp.uni-kl.de
    X-MirrorBrain-Realm: country
    Location: http://ftp.uni-kl.de/pub/linux/opensuse/distribution/11.1/iso/openSUSE-11.1-Addon-Lang-i586.iso
    Content-Type: text/html; charset=iso-8859-1

The X-Prefix and X-AS headers are obviously only added in the response if mod_asn is configured to do so.

When testing with local IP addresses (like 192.168.x.x), there's not much to
look up, though. You could however play with sending X-Forwarded-For headers,
provided that you configured "ASIPHeader X-Forwarded-For", and can lookup
arbitrary IPs thereby. You can use curl with the following option, causing it
to add an X-Forwarded-For header with arbitrary value to the request headers:

   % curl -sv -H "X-Forwarded-For: 128.176.216.184" <url>

It can be helpful to set "ASLookupDebug On" for some directory - you'll see
every step which the module does being logged to the error_log.



Logging
------------------------------------

* since the data being looked up is stored in the subprocess environment, it is
  trivial to log it, by adding the following placeholder to the LogFormat::

    ASN:%{ASN}e P:%{PFX}e


That's it!

Questions, bug reports, patches are welcome at mirrorbrain@mirrorbrain.org.