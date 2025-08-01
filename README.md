if you have "csrf check failed" error you should : 

Open the Apache configuration file:

```bash
sudo vim /etc/apache2/apache2.conf
```
then ;

```
<Directory /var/www/>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

That ```AllowOverride None``` line will prevent ```Apache``` from reading the ```.htaccess``` file in ownCloud at all ❗
And since ownCloud controls security settings like CSRF through ```.htaccess```, that's exactly why you get a ```CSRF check failed```!

You need to enable ```AllowOverride All``` for ownCloud. If the ownCloud installation path is exactly ```/var/www/owncloud```, add or replace this setting:

```
<Directory /var/www/owncloud>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

Restart Apache:

```bash
sudo systemctl restart apache2
```
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

In some cases, if the above solution does not work, you can get the answer by running the following command and then restarting Apache:
```bash
a2enmod rewrite
```
and then
```bash
service apache2 restart
```

