<VirtualHost *:80>
    ServerName http://115.159.64.159
    DocumentRoot /var/www/laravel/public
    <Directory /var/www/laravel/public>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog /var/www/laravel/storage/logs/error.log
    CustomLog /var/www/laravel/storage/logs/access.log combined
</VirtualHost>


<VirtualHost *:80>
      ServerName http://115.159.64.159
      DocumentRoot /var/www/recycling

      <Directory /var/www/recycling>
          AllowOverride All
          Require all granted
          Options -Indexes +FollowSymLinks
      </Directory>

      ErrorLog /var/log/httpd/recycling-error.log
      CustomLog /var/log/httpd/recycling-access.log combined
  </VirtualHost>