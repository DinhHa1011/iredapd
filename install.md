# Iredapd
## Install
- Create user on Red Hat, CentOS, Debian, Ubuntu, OpenBSD:
```
useradd -m -s /sbin/nologin -d /home/iredapd iredapd
```
- Install required packages
    ```
    sudo apt-get install python-ldap python-mysqldb python-sqlalchemy python-webpy python3-pip -y
    ```
    ```
    pip install git+https://github.com/rthalley/dnspython
    ```
    - If **error** not found package: 
    ```
    sudo apt-get install python3-ldap python3-mysqldb python3-sqlalchemy python3-webpy
    ```
- Download project file
```
cd /opt
git clone https://github.com/iredmail/iRedAPD.git
ln -s /opt/iRedAPD /opt/iredapd
chown -R root:root /opt/iRedAPD/
chmod -R 0500 /opt/iRedAPD/
```
- Copy RC script to /etc/init.d/ (Linux), or /usr/local/etc/rc.d/ (FreeBSD), /etc/rc.d/ (OpenBSD), and set correct permission.
```
cp /opt/iredapd/rc_scripts/iredapd.rhel /etc/init.d/iredapd
chmod +x /etc/init.d/iredapd
```
- Create a new config file by copying sample config.
    ```
    cp /opt/iredapd/settings.py.sample /opt/iredapd/settings.py
    chown root:root /opt/iredapd/settings.py
    chmod 0400 /opt/iredapd/settings.py
    ```
    - Config hiện tại đang như sau:
    ```
    ############################################################
    # DO NOT MODIFY THIS LINE, IT'S USED TO IMPORT DEFAULT SETTINGS.
    from libs.default_settings import *
    ############################################################

    # Listen address and port.
    listen_address = '127.0.0.1'
    # Port for normal Postfix policy requests.
    listen_port = 7777
    # Ports for SRS (Sender Rewriting Scheme).
    # - `srs_forward_port` is used in Postfix parameter `sender_canonical_maps`.
    # - `recipient_canonical_maps` is used in Postfix parameter `recipient_canonical_maps`.
    srs_forward_port = 7778
    srs_reverse_port = 7779

    # Run as a low privileged user.
    run_as_user = 'iredapd'

    # Path to pid file.
    pid_file = '/var/run/iredapd.pid'

    # Log level: info, debug.
    log_level = 'info'

    # Backend: ldap, mysql, pgsql.
    backend = 'mysql'

    # Enabled plugins.
    plugins = ['reject_null_sender', 'wblist_rdns', 'reject_sender_login_mismatch', 'greylisting', 'throttle', 'amavisd_wblist', 'sql_alias_access_policy']

    # SRS (Sender Rewriting Scheme)
    #
    # The secret key(s) used to generate cryptographic hash.
    # The first secret key is used for generating AND verifying hash in SRS
    # address. If you have old keys, you can append them also for verification only.
    srs_secrets = ['7d86deed2cdee17baa8cf216348efe05']

    # Rewrite address will be 'xxx@<srs_domain>', so please make sure `srs_domain`
    # is a resolvable mail domain name and pointed to your server.
    srs_domain = 'anthanh264.com'

    # For LDAP backend.
    #
    # LDAP server setting.
    # URI must starts with ldap:// (plain/TLS) or ldaps:// (SSL).
    #
    # Tip: You can get binddn, bindpw from /etc/postfix/ldap/*.cf.
    #

    #ldap_uri = 'ldap://127.0.0.1:389'
    #ldap_basedn = 'o=domains,dc=iredmail,dc=org'
    #ldap_binddn = 'cn=vmail,dc=iredmail,dc=org'
    #ldap_bindpw = 'password'
    # Enable TLS for secure connection.
    # `ldap_uri` should start with `ldap://` and use port 389.
    #ldap_enable_tls = False

    # For SQL (MySQL/MariaDB/PostgreSQL) backends, used to query mail accounts.
    vmail_db_server = '127.0.0.1'
    vmail_db_port = '3306'
    vmail_db_name = 'maildb'
    vmail_db_user = 'mailuser'
    vmail_db_password = 'mailPWD'

    # For Amavisd policy lookup and white/blacklists.
    amavisd_db_server = '127.0.0.1'
    amavisd_db_port = '3306'
    amavisd_db_name = 'amavisd'
    amavisd_db_user = 'amavisd'
    amavisd_db_password = 'password'

    # iRedAPD database, used for greylisting, throttle.
    iredapd_db_server = '127.0.0.1'
    iredapd_db_port = '3306'
    iredapd_db_name = 'iredapd'
    iredapd_db_user = 'iredapd'
    iredapd_db_password = 'password'

    # mlmmjadmin integration, used to query whether sender is a subscriber of
    # mailing list when `ALLOWED_LOGIN_MISMATCH_LIST_MEMBER = True`.
    mlmmjadmin_api_endpoint = 'http://127.0.0.1:7790/api'
    mlmmjadmin_api_auth_token = ''

    ```

- Create log file: /var/log/iredapd/iredapd.log.
```
mkdir -p /var/log/iredapd
touch /var/log/iredapd/iredapd.log
chown -R iredapd:iredapd /var/log/iredapd
```
- Copy /opt/iredapd/samples/rsyslog.d/iredapd.conf to /etc/rsyslog.d/ and restart rsyslog service
```
cp /opt/iredapd/samples/rsyslog.d/iredapd.conf /etc/rsyslog.d/
chown root:root /etc/rsyslog.d/iredapd.conf
service rsyslog restart
```
- Run command crontab -e -u root to add cron jobs for root user:
```
# iRedAPD: Clean up database hourly.
1   *   *   *   *   python /opt/iredapd/tools/cleanup_db.py >/dev/null

# iRedAPD: Convert SPF DNS record of specified domain names to IP
#          addresses/networks hourly.
2   *   *   *   *   python /opt/iredapd/tools/spf_to_greylist_whitelists.py >/dev/null
```
- SQL Iredapd
    + Create database and user;
    ```
    mysql -u root
    CREATE DATABASE iredapd;
    CREATE USER 'iredapd'@'%' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON iredapd.* TO 'iredapd'@'%' WITH GRANT OPTION;
    FLUSH PRIVILEGES;
    ```
    + Create tables:
    ```
    sed  -i '1i use iredapd;' /opt/iredapd/SQL/iredapd.mysql
    mysql </opt/iredapd/SQL/iredapd.mysql
    ```

- Enable iRedAPD service:
```
sudo update-rc.d iredapd defaults
```
- Start iRedAPD service:
``
/etc/init.d/iredapd restart
``
## Configure Postfix to use iRedAPD as SMTP policy server
Note: Restarting Postfix service is required after you modified its config files (/etc/postfix/main.cf and /etc/postfix/master.cf).

- Use iRedAPD as SMTP policy server: In Postfix config file /etc/postfix/main.cf (it’s /usr/local/etc/postfix/main.cf on FreeBSD), update parameter smtpd_recipient_restrictions and smtpd_end_of_data_restrictions like below to enable iRedAPD:
    ```
    smtpd_recipient_restrictions =
        ...
        check_policy_service inet:127.0.0.1:7777,
        permit_mynetworks,
        ...

    smtpd_end_of_data_restrictions =
        check_policy_service inet:127.0.0.1:7777
    ```
    - WARNING:
        - Order of restriction rules is very important, please make sure you have check_policy_service inet:127.0.0.1:7777 **before** permit_mynetworks.
        - If you update Postfix mynetworks= with some IP addresses/networks, please also list them in iRedAPD config file /opt/iredapd/settings.py, parameter MYNETWORKS =.

- [**OPTIONAL**] Use iRedAPD as SRS (Sender Rewriting Scheme) policy server: In Postfix config file `/etc/postfix/main.cf` (it’s /usr/local/etc/postfix/main.cf on FreeBSD), add 4 new parameters:
```
sender_canonical_maps = tcp:127.0.0.1:7778
sender_canonical_classes = envelope_sender
recipient_canonical_maps = tcp:127.0.0.1:7779
recipient_canonical_classes= envelope_recipient,header_recipient
```

- Restart Postfix service to enable iRedAPD
```
service postfix restart
```

## Log check 
```
tail -f /var/log/iredapd/iredapd.log
```

 
