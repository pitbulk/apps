INTRODUCTION
============

This App provide SAML authentication support based on the simpleSAMLphp SP software.


How install and configure simpleSAMLphp as SP
=============================================

Install simpleSAMLphp
---------------------

First of all install the [simpleSAMLphp library dependences](http://simplesamlphp.org/docs/stable/simplesamlphp-install#section_3):

    CentOS --> yum install php5 php-ldap php-mbstring php-xml mod_ssl openssl
    Debian --> apt-get install php5 php5-mcrypt php5-mhash php5-mysql openssl


Now we can download the [latest version of simpleSAMLphp](http://code.google.com/p/simplesamlphp/downloads/list>), now is the 1.11: ::

 Directly --> http://simplesamlphp.googlecode.com/files/simplesamlphp-1.11.0.tar.gz
 From subversion repository --> svn co http://simplesamlphp.googlecode.com/svn/tags/simplesamlphp-1.11.0

Put the simplesamlphp directory at ``/var/www/simplesamlphp``


The SSL cert
------------

simpleSAMLphp requires a SSL cert. We can buy one or create it.

Create a self-signed cert
.........................

In order to generate a self-signed cert you need openssl:

    Centos --> yum install openssl
    Debian --> apt-get install openssl

Using OpenSSL we will generate a self-signed certificate in 3 steps.

* Generate private key:

    openssl genrsa -out server.pem 1024

* Generate CSR: (In the "Common Name" set the domain of your instance)

    openssl req -new -key server.pem -out server.csr

* Generate Self Signed Key:

    openssl x509 -req -days 365 -in server.csr -signkey server.pem -out server.crt

Override the certs of the ``/var/www/simplesamlphp/cert`` folder with the  generated certs.


Configure simpleSAMLphp
-----------------------

Copy the default config file from the template directory:

    cp /var/www/simplesamlphp/config-templates/config.php /var/www/simplesamlphp/config/config.php


And configure some values:

    'auth.adminpassword' => 'secret'      # Set a new password for admin web interface

    'enable.saml20-idp' => true,          # Enable ssp as IdP

    'secretsalt' => 'secret',             # Set a Salt, in the config file there is documentation to generate it

    'technicalcontact_name' => 'Admin name',          # Set admin data
    'technicalcontact_email' => 'xxxx@example.com',

    'session.cookie.domain' => '.example.com',        # Set the global domain, to share cookie with the rest of componnets

In production environment set also those values:

    'admin.protectindexpage'        => true,    # To protect the index page of simpleSAMLphp
    'debug'                 =>      FALSE,
    'showerrors'            =>      FALSE,      # To hide error-trace


Change the permission for some directories, execute the following command at the simpleSAMLphp folder:

    chown -R apache:apache cert log data metadata



Copy the default authsource file from the template directory:

    cp /var/www/simplesamlphp/config-templates/authsources.php /var/www/simplesamlphp/config/authsources.php

And set admin and an sp source. Something like:

    <?php

        $config = array(

                'admin' => array(
                        'core:AdminPassword',
                ),

                'owncloud' => array(
                    'saml:SP',

                    'entityID' => 'https://sp.example.com/simplesaml/module.php/saml/sp/metadata.php/owncloud',
                    
                    'idp' => 'https://idp.example.com/simplesaml/saml2/idp/metadata.php', # Set the entityID of the IdP you gonna use
		
                    'privatekey' => 'server.pem',
                    'certificate' => 'server.crt',
                ),
        );
    ?>


And now paste the metadata of your IdP in simpleSAMLphp format at ``/var/www/simplesamlphp/metadata/saml20-idp-remote.php``


Apache configuration
--------------------

The apache configuration may look like this:

 <VirtualHost *:80>
        ServerName sp.example.com
        DocumentRoot /var/www/simplesamlphp/www

        Alias /simplesaml /var/simplesamlphp/www
 </VirtualHost>

 <VirtualHost *:443>
        ServerName sp.example.com
        DocumentRoot /var/www/simplesamlphp/www

        Alias /simplesaml /var/simplesamlphp/www

        SSLEngine on
        SSLCertificateFile /var/www/simplesamlphp/cert/server.crt
        SSLCertificateKeyFile /var/www/simplesamlphp/cert/server.pem
 </VirtualHost>



Make sure that your IdP server runs HTTPS (SSL)



NTP server
----------

To get Saml2 run correctly we need have sure that all machine's clock are synced.

Install ntp: 

    Centos --> yum install ntp
    Debian --> apt-get install ntp

Configure the ntp service `/etc/ntp.conf`:

    server 0.uk.pool.ntp.org
    server 1.uk.pool.ntp.org
    server 2.uk.pool.ntp.org
    server 3.uk.pool.ntp.org

`Check the` [ntp server list](http://www.pool.ntp.org/use.html) `and use the server that is near from your server.`

Enable the server and put it on the system boot

    Centos --> service ntpd start
               chkconfig ntpd on

    Debian -> /etc/init.d/ntp start
              update-rc.d ntp defaults



More info at [http://simplesamlphp.org/docs/stable/simplesamlphp-sp](http://simplesamlphp.org/docs/stable/simplesamlphp-sp)


HOW INSTALL AND CONFIGURE THE SAML PLUGIN
=========================================

1. Copy the `user_saml` folder inside the ownCloud's apps folder and give to apache server privileges on whole the folder.
2. Access to ownCloud web with an user with admin privileges.
3. Access to the Appications pannel and enable the SAML app.
4. Access to the Administration pannel and configure the SAML app.
5. Take care of session issue. ownCloud 4.5.5 and after version set for ownCloud its own session cookiename and that makes conflicts with simpleSAMLphp. There are 2 solutions for this problem:
 
* Set the same cookiename to simpleSAMLphp and ownCloud. Check the value of the 'instanceid' at config/config.php in ownCloud, and set the same value to the 'session.phpsession.cookiename' var of the config/config.php of simpleSAMLphp

* Use different session handler for ownCloud and simpleSAMLphp, Use memcache or SQL backend in simpleSAMLphp (http://simplesamlphp.org/docs/stable/simplesamlphp-maintenance#section_2)


EXTRA INFO
==========

* If you enable the "Autocreate user after saml login" option, then if an user does not exist, will be created. If this option is disabled and the user does not existed then the user will be not allowed to log in ownCloud.

* If you enable the "Update user data" option, when an existed user enter, then his email and groups will be updated.

  By default the SAML App will unlink all the groups from a user and will provide the group defined at the groupMapping attribute. If the groupMapping is not defined
  the value of the defaultGroup field will be used instead. If both are undefined, then the user will be set with no groups.
  But if you configure the "protected groups" field, those groups will not be unlinked from the user.

* If you want to redirect to any specific app after force the login you can set the url param linktoapp. Also you can pass extra args to build the target url using the param linktoargs (the value must be urlencoded).
  Ex. ?app=user_saml&linktoapp=files&linktoargs=file%3d%2ftest%2ftest_file.txt%26getfile%3ddownload.php
      ?app=user_saml&linktoapp=files&linktoargs=dir%3d%2ftest

* There is a parameter in the settings named `force_saml_login` to avoid the login form, redirecting directly to the IdP when accesing owncloud.
  If you are an admin and you want to log in using the login form, then use the GET param `admin_login` to deactivate the forced redirection.

NOTES
=====

If you had an older version of this plugin installed and the SAML link no appears at the main view, edit the index.php and set the $RUNTIME_NOAPPS to FALSE;


 
