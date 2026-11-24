Create a folder, create a user for homepage to use

chown -R user:user homepage

add the dockercompose file

change the variables for your reverse proxy domain name and server IP address.

Find the config files for homepage in the `config.md` file:
```
vi homepage/config/settings.yaml
```