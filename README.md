## Introduction

[Nessus](http://www.tenable.com/products/nessus-vulnerability-scanner) is a high-quality vulnerability scanner.  One of its main advantages is its extensive and continually evolving plugin database of vulnerability checks.  Additionally, one can configure Nessus to constantly scan a network with a continuously updated collection of plugins with minimal user interaction -- an extremely appealing notion!

One drawback, though, is that plugins not explicitly listed in the client configuration file -- eg, new plugins -- are enabled by default (unless you've enabled safe_checks, in which case dangerous plugins are disabled).  Further, the only way to update the configuration file out-of-the-box is via the GUI manually.

To get around this shortcoming, I've written *update-nessusrc*, which queries a Nessus server for its list of available plugins and updates a Nessus client configuration file named on the commandline.  Specifically, it completely updates the sections `SCANNER_SET` and `PLUGIN_SET` whenever it is run.  When used periodically along with `nessus-update-plugins`, it ensures your client configuration files are as current as possible.

*update-nessusrc* is written in Perl and calls the Nessus client to obtain a list of current plugins (using the option `-qp`).  It should work on any unix-like system with Perl 5.003 or better and Nessus 1.1.13 or better.  It also requires the following Perl modules:

* `Carp`
* `Getopt::Long`
* `IPC::Open2`
* `LWP::Debug`
* `LWP::UserAgent`
* `Safe`

If your system does not have these modules installed already, visit [CPAN](http://search.cpan.org/).  Note that `Safe` must be at least version 2.0, which does not work with versions of Perl older than 5.003.  Note also that `LWP::Debug` and `LWP::UserAgent` are not included with the default Perl distribution so you may need to install them yourself; they're included as part of the [LWP library](http://search.cpan.org/dist/libwww-perl/).

Note: Jay Jacobson of [Edgeos](http://www.edgeos.com/) wrote a Python script, also named update-nessusrc, that offers functionality similar to this script. I'm not sure it's still available, though.


## Installation

* Retrieve [the script](update-nessusrc) and save it locally.
* Verify ownership and permissions on the script - it can (and probably should) be invoked as an ordinary user rather than root and will hold a userid and password used to connect with a Nessus server.
* Edit the script and set `$nessusd_host`, `$nessusd_port`, `$nessusd_user`, `$nessusd_user_pass`, and `$proxy` according to your environment. Also, you may wish to adjust the location of the perl interpreter in the first line, `$ENV{PATH}`, `@plugin_cats`, `@plugin_fams`, `@plugin_excludes`, `@plugin_includes`, `@plugin_risks`, and/or `$script_config` to suit your tastes.
* Have each user create a script configuration file with personalized settings, if desired.


## Use

*update-nessusrc* offers considerable control in the selection of plugins - you can include entire categories and families of plugins, include specific plugins, exclude specific plugins, or even include based on the risk factor of the vulnerabilities scanned for by plugins.

Much of the script's behaviour is controlled by variables set either in the script itself or in a separate script configuration file, specified by `$script_config`.  If it exists, the script configuration file is treated as Perl code and evaluated in a _sandbox_, which supports only variable definitions.  Use of a separate script configuration file makes it possible for several people to share *update-nessusrc* on the same system and promises to make upgrading the script much easier.

There are several commandline arguments you can use to override variables defined in the script or the script configuration file:

| Option | Meaning |
| ------ | ------- |
| -c, --categories <plugin categories> | Enable plugins in the specified categories, overriding `@plugin_cats`. `_all_` can be used to represent all plugin categories and the prefix `!` to skip specific ones. |
| -d, --debug | Display debugging messages while running.  Creates a scratch configuration file but doesn't actually replace the original one. |
| -f, --families <plugin families> | Enable plugins in the specified families, overriding `@plugin_fams`. `_all_` can be used to represent all plugin families and `!` to skip specific ones. |
| -i, --includes <plugin ids> | Include the specified plugin ids, overriding `@plugin_includes`. `_all_` can be used to represent all plugin ids, `!` to skip specific ones, and `x-y` to cover the range of plugins between ids _x_ and _y_ inclusive. |
| -r, --risks <risk factors> | Enable plugins that scan for vulnerabilities with the specified risk factors, overriding `@plugin_risks`. *Note:* unlike other options, risk factors specified are regarded as regular expressions and matched case-insensitively against the risk factors appearing in plugin descriptions.
| -s, --summary | Display a summary of the changes, detailing plugins added, removed, enabled, or disabled. |
| -t, --top20 | Include plugins to scan for the [SANS Top 20 Vulnerabilities](http://www.sans.org/top20/). |
| -x, --excludes <plugin ids> | Exclude the specified plugin ids, overriding `@plugin_excludes`.  `_all_` can be used to represent all plugin ids, `!` to skip specific ones, and `x-y` to cover the range of plugins between ids _x_ and _y_ inclusive. |

Notes:

* For a list of plugin families, see [http://www.tenable.com/plugins/index.php?view=all](http://www.tenable.com/plugins/index.php?view=all); and for plugin categories, see the file `doc/WARNING.En` in the nessus-core source for Nessus 2.x. Unfortunately, risk factors are not standardized; while `Low`, `Medium`, and `High` are common ones, they are by far from the only ones used. Further, some risk factors depend on the outcome of the plugin. Possible category and family names will both change over time, especially the latter: category names may vary with the version of Nessus you use while family names are specified by plugin writers as an arbitrary string using the `script_family` function. Thus, you may wish to periodically review the specific categories and families you use with this script.
* Commandline arguments take precedence over variables defined in a script configuration file, which in turn take precedence over variables defined in the script itself. For example, you can disable all plugin categories by using the commandline argument `-c ""`.
* *Plugin categories, families, and risk factors are considered together when deciding whether to include plugins.*  For example, choosing `-c denial -f "SMTP problems" -r "High"` will run only denial of service attacks for high-risk vulnerabilities specifically associated with SMTP servers.
* To negate a range of plugin ids, prefix the first id only with '!'; eg, `!10000-100010`.
* Nessus will run plugins in the category `settings` as needed regardless of what's in the configuration file.  These plugins only update the knowledge bases; they do not actually send any packets.
* The `'--top20'` option on one hand and the '--categories'`, `'--families'`, and `'--risks'` options on the other are mutually exclusive.
* Plugins explicitly excluded will never be used regardless of the other variables or commandline options.
* Multiple categories / families / risks / plugin ids can be specified  either by a comma-delimited string or by multiple argument pairs. For example, `-c "settings,infos"` is equivalent to `-c settings -c infos`.

Examples:

| Invocation | Meaning |
| ---------- | ------- |
| `update-nessusrc ~/.nessusrc` | updates nessusrc using default settings (eg, non-dangerous plugins and ping / tcp_connect scanners in the script as distributed). |
| `update-nessusrc -s ~/.nessusrc` | same as above but also print a summary of the changes. |
| `update-nessusrc -d ~/.nessusrc` | produces an alternate nessusrc without replacing original and also prints lots of debugging info. |
| `update-nessusrc -c "" -f "" -r "" -i "10335,11835" .nessusrc-ms03-039` | updates a special nessusrc to use tcp_connect scanner (plugin #10335) and test just for MS RPC interface buffer overruns (plugin #11835). |
| `update-nessusrc -c "denial,destructive_attack,flood,kill_host" -i 10335 ~/.nessusrc-destructive` | updates a special nessusrc w/ destructive plugins and tcp connect scanner. |
| `update-nessusrc -c "_all_" -f "SMTP problems" ~/.nessusrc-smtp` | updates a special nessusrc w/ plugins associated with SMTP servers in all categories. |
| `update-nessusrc -c "_all_" -c !destructive_attack -f "SMTP problems" ~/.nessusrc-smtp` | updates a special nessusrc w/ plugins associated with SMTP servers in all categories except destructive_attack. |
| `update-nessusrc -r "(Critical|High)" ~/.nessusrc-risky` | updates a special nessusrc w/ plugins that scan only for critical- and high-risk vulnerabilities. |
| `update-nessusrc -t ~/.nessusrc-top20` | Update a special nessusrc to scan for the SANS Top 20 Vulnerabilities. |


## Known Bugs and Caveats

This script may hang indefinitely if `paranoia_level` is not set in your config file (for example, if you've exported a policy from NessusClient 3.2+) or it is set but the SSL certificate presented by the server has changed for some reason.  If this happens, either update the script so it calls the client with `-x`, which disables certificate verification, or run `nessus` manually and resolve SSL_paranoia / certificate issue outside of this script, at least until I can figure out a satisfactory way to handle such cases.

As of August 2005, SANS appears to be adjusting their content based on the _browser's_ user agent and, in the case of my script, it redirects to a non-existent page.  A work-around is to change the user-agent string, `$useragent`, to something more common; eg,

>  Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.7.10) Gecko/20050716 Firefox/1.0.6

You can make the change either in the script itself or in the file `~/.update-nessusrc`.

This script is not a substitute for the Nessus client in terms of managing a configuration file.  On one hand, it requires that a configuration file already exists.  On the other, several plugins require additional configuration - simply adding them to the list of plugins used may not be optimal.

There is a limit to the size of the arguments passed to `script_cve_id()`, which sets the CVE IDs of the flaws tested by each plugin.  Additional CVE IDs, which by convention are listed in comments, are not handled by this script since they can not be reliably identified.  Thus, you would do well to review the report of Top 20 Vulnerabilities for which no plugins were found and update the configuration file manually after examining plugins available on your server.  Otherwise, you risk generating a configuration file that's not as comprehensive as it could be.

To ensure an accurate scan for the SANS Top 20 Vulnerabilities, you must make sure `auto_enable_dependencies` is set to `yes` in the configuration file; *update-nessusrc* will *_not_* do this for you.

Finally, realize that this script along with its script configuration files may hold userids and passwords used to connect to a Nessus server; protect them accordingly!


## Copyright and License

Copyright (c) 2003-2016, George A. Theall.
All rights reserved.

This script is free software; you can redistribute it and/or modify it under the same terms as Perl itself.

