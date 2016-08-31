#Syslog

##Syslog installation
First, let's install some dependecies
```
sudo apt-get install syslog-ng-core

```
Then,install it syslog-ng:
```python 
sudo apt-get install syslog-ng
```
For the server machine add this following in other to support Mongo DB driver
```
sudo apt-get install syslog-ng-mod-mongodb
```
##Recommended architecture
We have two types of architectures:

1. Client syslog ------> Server syslog -----> Database MongoDb (Recommended for centralized Log management)

2. Client Syslog ------> Database MongoDB (For Unique machine monitoring)

The first is the one that is recommended because Mongo DB may fail from time to time resulting in data loss. 
As we rely on MongoDB, only for easier search and the actual logs are still stored on hard disk, 
`this is not an issue at all`.

##Syslog client configuration
```
@version: 3.5
@include "scl.conf"
@include "`scl-root`/system/tty10.conf"

# Syslog-ng configuration file, compatible with default Debian syslogd
# installation.

# First, set some global options.
options { chain_hostnames(off); flush_lines(0); use_dns(no); use_fqdn(no);
	  owner(root); group(root); perm(0640); stats_freq(0);
	  bad_hostname("^gconfd$");
	  dir_perm(0700); dir_group(root);
	  flush_lines(0); log_fifo_size(1000); 
	  create_dirs (yes); time_reopen (10);
};
########################
## SOURCES
########################

source s_all {
        # message generated by Syslog-NG
        internal();
        
        # standard Linux log source (this is the default place for the syslog()
        # function to send logs to)
        unix-stream("/dev/log");
        # messages from the kernel
        file("/proc/kmsg" program_override("kernel"));
};


########################
#  Destinations  
########################

########################
# Local Destinations  
########################

# some standard log files
destination df_auth { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/auth.log"); };
destination df_syslog { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/syslog"); };
destination df_cron { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/cron.log"); };
destination df_daemon { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/daemon.log"); };
destination df_kern { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/kern.log"); };
destination df_lpr { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/lpr.log"); };
destination df_mail { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/mail.log"); };
destination df_user { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/user.log"); };
destination df_uucp { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/uucp.log"); };

# these files are meant for the mail system log files
# and provide re-usable destinations for {mail,cron,...}.info,
# {mail,cron,...}.notice, etc.
destination df_facility_dot_info { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.info"); };
destination df_facility_dot_notice { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.notice"); };
destination df_facility_dot_warn { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.warn"); };
destination df_facility_dot_err { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.err"); };
destination df_facility_dot_crit { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.crit"); };

# these files are meant for the news system, and are kept separated
# because they should be owned by "news" instead of "root"
destination df_news_dot_notice { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/news/news.notice" owner("news")); };
destination df_news_dot_err { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/news/news.err" owner("news")); };
destination df_news_dot_crit { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/news/news.crit" owner("news")); };

# some more classical and useful files found in standard syslog configurations
destination df_debug { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/debug"); };
destination df_messages { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/messages"); };

# pipes
# a console to view log messages under X
destination dp_xconsole { pipe("/dev/xconsole"); };

# consoles
# this will send messages to everyone logged in
destination du_all { usertty("*"); };

########################
# Network Destinations  
########################
destination d_net { tcp("127.0.0.1" port(27017)  ); };



########################
# Filters
########################
# Here's come the filter options. With this rules, we can set which 
# message go where.

######
# filters

# all messages from the auth and authpriv facilities
filter f_auth { facility(auth, authpriv); };

# all messages except from the auth and authpriv facilities
filter f_syslog { not facility(auth, authpriv); };

# respectively: messages from the cron, daemon, kern, lpr, mail, news, user,
# and uucp facilities
filter f_cron { facility(cron); };
filter f_daemon { facility(daemon); };
filter f_kern { facility(kern); };
filter f_lpr { facility(lpr); };
filter f_mail { facility(mail); };
filter f_news { facility(news); };
filter f_user { facility(user); };
filter f_uucp { facility(uucp); };

# some filters to select messages of priority greater or equal to info, warn,
# and err
# (equivalents of syslogd's *.info, *.warn, and *.err)
filter f_at_least_info { level(info..emerg); };
filter f_at_least_notice { level(notice..emerg); };
filter f_at_least_warn { level(warn..emerg); };
filter f_at_least_err { level(err..emerg); };
filter f_at_least_crit { level(crit..emerg); };

# all messages of priority debug not coming from the auth, authpriv, news, and
# mail facilities
filter f_debug { level(debug) and not facility(auth, authpriv, news, mail); };

# all messages of info, notice, or warn priority not coming form the auth,
# authpriv, cron, daemon, mail, and news facilities
filter f_messages {
        level(info,notice,warn)
            and not facility(auth,authpriv,cron,daemon,mail,news);
};

# messages with priority emerg
filter f_emerg { level(emerg); };

# complex filter for messages usually sent to the xconsole
filter f_xconsole {
    facility(daemon,mail)
        or level(debug,info,notice,warn)
        or (facility(news)
                and level(crit,err,notice));
};

######################################################
# Save all logs locally and send them in the network
#######################################################

######
# logs
# order matters if you use "flags(final);" to mark the end of processing in a
# "log" statement

# these rules provide the same behavior as the commented original syslogd rules

# auth,authpriv.*                 /var/log/auth.log
log {
        source(s_all);
        filter(f_auth);
        destination(df_auth); destination(d_net);
};

# *.*;auth,authpriv.none          -/var/log/syslog
log {
        source(s_all);
        filter(f_syslog);
        destination(df_syslog); destination(d_net);
};

# this is commented out in the default syslog.conf
# cron.*                         /var/log/cron.log
#log {
#        source(s_all);
#        filter(f_cron);
#        destination(df_cron); destination(d_net);
#};

# daemon.*                        -/var/log/daemon.log
log {
        source(s_all);
        filter(f_daemon);
        destination(df_daemon); destination(d_net);
};

# kern.*                          -/var/log/kern.log
log {
        source(s_all);
        filter(f_kern);
        destination(df_kern); destination(d_net);
};

# lpr.*                           -/var/log/lpr.log
log {
        source(s_all);
        filter(f_lpr);
        destination(df_lpr); destination(d_net);
};

# mail.*                          -/var/log/mail.log
log {
        source(s_all);
        filter(f_mail);
        destination(df_mail); destination(d_net);
};

# user.*                          -/var/log/user.log
log {
        source(s_all);
        filter(f_user);
        destination(df_user); destination(d_net);
};

# uucp.*                          /var/log/uucp.log
log {
        source(s_all);
        filter(f_uucp);
        destination(df_uucp); destination(d_net);
};

# mail.info                       -/var/log/mail.info
log {
        source(s_all);
        filter(f_mail);
        filter(f_at_least_info);
        destination(df_facility_dot_info); destination(d_net);
};

# mail.warn                       -/var/log/mail.warn
log {
        source(s_all);
        filter(f_mail);
        filter(f_at_least_warn);
        destination(df_facility_dot_warn); destination(d_net);
};

# mail.err                        /var/log/mail.err
log {
        source(s_all);
        filter(f_mail);
        filter(f_at_least_err);
        destination(df_facility_dot_err); destination(d_net);
};

# news.crit                       /var/log/news/news.crit
log {
        source(s_all);
        filter(f_news);
        filter(f_at_least_crit);
        destination(df_news_dot_crit); destination(d_net);
};

# news.err                        /var/log/news/news.err
log {
        source(s_all);
        filter(f_news);
        filter(f_at_least_err);
        destination(df_news_dot_err); destination(d_net);
};

# news.notice                     /var/log/news/news.notice
log {
        source(s_all);
        filter(f_news);
        filter(f_at_least_notice);
        destination(df_news_dot_notice); destination(d_net);
};


# *.=debug;\
#         auth,authpriv.none;\
#         news.none;mail.none     -/var/log/debug
log {
        source(s_all);
        filter(f_debug);
        destination(df_debug); destination(d_net);
};


# *.=info;*.=notice;*.=warn;\
#         auth,authpriv.none;\
#         cron,daemon.none;\
#         mail,news.none          -/var/log/messages
log {
        source(s_all);
        filter(f_messages);
        destination(df_messages); destination(d_net);
};

# *.emerg                         *
log {
        source(s_all);
        filter(f_emerg);
        destination(du_all); destination(d_net);
};


# daemon.*;mail.*;\
#         news.crit;news.err;news.notice;\
#         *.=debug;*.=info;\
#         *.=notice;*.=warn       |/dev/xconsole
log {
        source(s_all);
        filter(f_xconsole);
        destination(dp_xconsole); destination(d_net);
};

```
##Syslog server configuration (Central loghost)
```
@version: 3.5
@include "scl.conf"
@include "`scl-root`/system/tty10.conf"

# Syslog-ng configuration file, compatible with default Debian syslogd
# installation.

# First, set some global options.
options { chain_hostnames(off); flush_lines(0); use_dns(no); use_fqdn(no);
	  owner(root); group(root); perm(0640); stats_freq(0);
	  bad_hostname("^gconfd$");
	  dir_perm(0700); dir_group(root);
	  flush_lines(0); log_fifo_size(1000);
	  create_dirs (yes); time_reopen (10);
};
#####################
#	SOURCE
######################

source s_network { tcp(ip(127.0.0.1) port(27017)); };

#don t forget yourself 
source s_local {
        # message generated by Syslog-NG
        internal();
        
        # standard Linux log source (this is the default place for the syslog()
        # function to send logs to)
        unix-stream("/dev/log");
        # messages from the kernel
        file("/proc/kmsg" program_override("kernel"));
};

###################################
# Destinations ( Local + MongoDB)
#####################################


########################
# Local Destinations  
########################

# some standard log files
destination df_auth { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/auth.log"); };
destination df_syslog { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/syslog"); };
destination df_cron { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/cron.log"); };
destination df_daemon { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/daemon.log"); };
destination df_kern { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/kern.log"); };
destination df_lpr { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/lpr.log"); };
destination df_mail { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/mail.log"); };
destination df_user { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/user.log"); };
destination df_uucp { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/uucp.log"); };

# these files are meant for the mail system log files
# and provide re-usable destinations for {mail,cron,...}.info,
# {mail,cron,...}.notice, etc.
destination df_facility_dot_info { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.info"); };
destination df_facility_dot_notice { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.notice"); };
destination df_facility_dot_warn { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.warn"); };
destination df_facility_dot_err { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.err"); };
destination df_facility_dot_crit { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/$FACILITY.crit"); };

# these files are meant for the news system, and are kept separated
# because they should be owned by "news" instead of "root"
destination df_news_dot_notice { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/news/news.notice" owner("news")); };
destination df_news_dot_err { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/news/news.err" owner("news")); };
destination df_news_dot_crit { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/news/news.crit" owner("news")); };

# some more classical and useful files found in standard syslog configurations
destination df_debug { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/debug"); };
destination df_messages { file("/var/log/net-daily/$YEAR/$MONTH/$DAY/messages"); };

# pipes
# a console to view log messages under X
destination dp_xconsole { pipe("/dev/xconsole"); };

# consoles
# this will send messages to everyone logged in
destination du_all  { usertty("*"); };

########################
 #MONGO DB  Destinations
########################

#${R_YEAR}-${R_MONTH}-${R_DAY} ${R_HOUR}:${R_MIN}:${R_SEC} #message when receiveds
#${S_YEAR}-${S_MONTH}-${S_DAY} ${S_HOUR}:${S_MIN}:${S_SEC}  message when Sent
#${S_UNIXTIME} Pour le Temps Timestamps
# Voir bien https://www.balabit.com/sites/default/files/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/date-macros.html#idp15122112
destination d_mongodb {
    mongodb(
        servers("localhost:27017")
        database("syslog")
        collection("messages")

    #   this does not support TLS  see https://goo.gl/t5BMGh
    #   password("aStrongAndLongPassword")
    #   username() 
     
	value-pairs(
		pair("date", "${UNIXTIME}")
		pair("ip","$SOURCEIP")
		pair("facility", "$FACILITY")
		pair("level", "$LEVEL")
		pair("host", "$HOST")
		pair("program", "$PROGRAM")
		pair("pid", int64("$PID"))
		pair("message", "$MSGONLY")
             )

    );
};

########################
# Filters
########################
# Here's come the filter options. With this rules, we can set which 
# message go where.

######
# filters

# all messages from the auth and authpriv facilities
filter f_auth { facility(auth, authpriv); };

# all messages except from the auth and authpriv facilities
filter f_syslog { not facility(auth, authpriv); };

# respectively: messages from the cron, daemon, kern, lpr, mail, news, user,
# and uucp facilities
filter f_cron { facility(cron); };
filter f_daemon { facility(daemon); };
filter f_kern { facility(kern); };
filter f_lpr { facility(lpr); };
filter f_mail { facility(mail); };
filter f_news { facility(news); };
filter f_user { facility(user); };
filter f_uucp { facility(uucp); };

# some filters to select messages of priority greater or equal to info, warn,
# and err
# (equivalents of syslogd's *.info, *.warn, and *.err)
filter f_at_least_info { level(info..emerg); };
filter f_at_least_notice { level(notice..emerg); };
filter f_at_least_warn { level(warn..emerg); };
filter f_at_least_err { level(err..emerg); };
filter f_at_least_crit { level(crit..emerg); };

# all messages of priority debug not coming from the auth, authpriv, news, and
# mail facilities
filter f_debug { level(debug) and not facility(auth, authpriv, news, mail); };

# all messages of info, notice, or warn priority not coming form the auth,
# authpriv, cron, daemon, mail, and news facilities
filter f_messages {
        level(info,notice,warn)
            and not facility(auth,authpriv,cron,daemon,mail,news);
};

# messages with priority emerg
filter f_emerg { level(emerg); };

# complex filter for messages usually sent to the xconsole
filter f_xconsole {
    facility(daemon,mail)
        or level(debug,info,notice,warn)
        or (facility(news)
                and level(crit,err,notice));
};
######################################################
# Save all logs locally and send them in the network
#######################################################

######
# logs
# order matters if you use "flags(final);" to mark the end of processing in a
# "log" statement

# these rules provide the same behavior as the commented original syslogd rules

# auth,authpriv.*                 /var/log/auth.log
log {
        source(s_network); source(s_local);
        filter(f_auth);
        destination(df_auth); destination(d_mongodb);
};

# *.*;auth,authpriv.none          -/var/log/syslog
log {
        source(s_network); source(s_local);
        filter(f_syslog);
        destination(df_syslog); destination(d_mongodb);
};

# this is commented out in the default syslog.conf
# cron.*                         /var/log/cron.log
#log {
#        source(s_network); source(s_local);
#        filter(f_cron);
#        destination(df_cron); destination(d_mongodb);
#};

# daemon.*                        -/var/log/daemon.log
log {
        source(s_network); source(s_local);
        filter(f_daemon);
        destination(df_daemon); destination(d_mongodb);
};

# kern.*                          -/var/log/kern.log
log {
        source(s_network); source(s_local);
        filter(f_kern);
        destination(df_kern); destination(d_mongodb);
};

# lpr.*                           -/var/log/lpr.log
log {
        source(s_network); source(s_local);
        filter(f_lpr);
        destination(df_lpr); destination(d_mongodb);
};

# mail.*                          -/var/log/mail.log
log {
        source(s_network); source(s_local);
        filter(f_mail);
        destination(df_mail); destination(d_mongodb);
};

# user.*                          -/var/log/user.log
log {
        source(s_network); source(s_local);
        filter(f_user);
        destination(df_user); destination(d_mongodb);
};

# uucp.*                          /var/log/uucp.log
log {
        source(s_network); source(s_local);
        filter(f_uucp);
        destination(df_uucp); destination(d_mongodb);
};

# mail.info                       -/var/log/mail.info
log {
        source(s_network); source(s_local);
        filter(f_mail);
        filter(f_at_least_info);
        destination(df_facility_dot_info); destination(d_mongodb);
};

# mail.warn                       -/var/log/mail.warn
log {
        source(s_network); source(s_local);
        filter(f_mail);
        filter(f_at_least_warn);
        destination(df_facility_dot_warn); destination(d_mongodb);
};

# mail.err                        /var/log/mail.err
log {
        source(s_network); source(s_local);
        filter(f_mail);
        filter(f_at_least_err);
        destination(df_facility_dot_err); destination(d_mongodb);
};

# news.crit                       /var/log/news/news.crit
log {
        source(s_network); source(s_local);
        filter(f_news);
        filter(f_at_least_crit);
        destination(df_news_dot_crit); destination(d_mongodb);
};

# news.err                        /var/log/news/news.err
log {
        source(s_network); source(s_local);
        filter(f_news);
        filter(f_at_least_err);
        destination(df_news_dot_err); destination(d_mongodb);
};

# news.notice                     /var/log/news/news.notice
log {
        source(s_network); source(s_local);
        filter(f_news);
        filter(f_at_least_notice);
        destination(df_news_dot_notice); destination(d_mongodb);
};


# *.=debug;\
#         auth,authpriv.none;\
#         news.none;mail.none     -/var/log/debug
log {
        source(s_network); source(s_local);
        filter(f_debug);
        destination(df_debug); destination(d_mongodb);
};


# *.=info;*.=notice;*.=warn;\
#         auth,authpriv.none;\
#         cron,daemon.none;\
#         mail,news.none          -/var/log/messages
log {
        source(s_network); source(s_local);
        filter(f_messages);
        destination(df_messages); destination(d_mongodb);
};

# *.emerg                         *
log {
        source(s_network); source(s_local);
        filter(f_emerg);
        destination(du_all); destination(d_mongodb);
};


# daemon.*;mail.*;\
#         news.crit;news.err;news.notice;\
#         *.=debug;*.=info;\
#         *.=notice;*.=warn       |/dev/xconsole
log {
        source(s_network); source(s_local);
        filter(f_xconsole);
        destination(dp_xconsole); destination(d_mongodb);
};


```

#Starting server 
```
sudo service syslog-ng start 
```

#Reload Server
```
sudo service syslog-ng reload 
```
#