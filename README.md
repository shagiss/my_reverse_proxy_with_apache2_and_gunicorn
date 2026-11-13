# Make a Reverse Proxy to Gunicorn with Apache2
**IMPORTANT:** Use at your own risk! No warranty, You're solely responsible for anything that might happen. If you broke something - It's your fault!
This guide is not comprehensive, nor completed. There's more to come. so again - **If you choose to use it, It's on you - You are responsible for whatever happens, good or bad**. Don't blame me for mistakes. but do let me know so I can fix it ASAP.

**No AI was used during the composition of this tutorial. (I hate AI)**

This tutorial is based on the following sources:

1. [Gunicorn Documentation](https://gunicorn.org/uwsgi)
2. [Apache 2.4 Documentation](https://httpd.apache.org/docs/2.4)
3. And a bit of help from [the uWSGI Documentation](https://uwsgi-docs.readthedocs.io/en/latest/Apache.html)

## Requirements
1. **Apache2**, Including the following modules: `mod_proxy` AND `mod_proxy_uwsgi`/`mod_proxy_http`
2. Of course, **You need Gunicorn**.

If you're interested to know, I tested this guide inside an Alpine Linux container using Incus.
You may test it however you like.

### Tips
[You might want to enable the apache2's `event` MPM to mimic Nginx.](https://httpd.apache.org/docs/2.4/en/mpm.html) it's up to you... some say it's recommended. I've done it.
[You might also want to enable http2. it is worth to mention that http2 won't work without TLS.](https://httpd.apache.org/docs/2.4/en/howto/http2.html)

## Let's begin, shall we?
### Step 1: Enable uWSGI Support on Gunicorn
[According to the Gunicorn documentation,](https://gunicorn.org/uwsgi/) you can enable uWSGI support as simple as:
```
cd /path/to/where_your_app_exists
gunicorn myapp:app --protocol uwsgi --bind unix:/tmp/gunicorn.ctl
```

You could also use HTTP if you desire. it's up to you. In this example, my apache2 configuration lets you choose whether HTTP or uWSGI (it checks with `<IfModule>`).

### Step 2: Make some configurations in your apache2 server
First of all you must enable the `mod_proxy`.
Then you must choose which way you prefer more:
the `mod_proxy_http`, or the `mod_proxy_uwsgi` module.
Once you choose your desired module, use `a2enmod proxy proxy_[uwsgi|http]` to enable it,
and apply the following to your apache2 configuration wherever is appropriate (vhosts is the best place):
```
# Here are some cute variables that I will be use:
Define v_mod_uwsgi "proxy_uwsgi_module"
Define v_mod_http "proxy_http_module"
# As you can guess, these define the modules that
# should be used (preferably one).
# This variable:
Define v_gunicorn_location "/"
# will tell apache on which paths the ProxyPass applies.
# For example, you could set it to: /api or /backend or /whatever.
# then when requesting /api or /backend or /whatever,
# Gunicorn handles the rest.

# For this one you may want to use a load balancer.
<Proxy "balancer://gunicorn_set">

	# You can proxy to a unix socket using HTTP
	<IfModule "${v_mod_http}">
		BalancerMember "unix:/tmp/gunicorn.ctl|http://localhost/"
	</IfModule>
	
	# Or, using mod_uwsgi. For help with mod_uwsgi, See:
	# <https://uwsgi-docs.readthedocs.io/en/latest/Apache.html>
	# and also: <https://gunicorn.org/uwsgi/>
	<IfModule "${v_mod_uwsgi}">
		BalancerMember "unix:/tmp/gunicorn.ctl|uwsgi://localhost/"
	</IfModule>
	
	# The choice is up to you!
	# (and your gunicorn configurations, of course.)
	
</Proxy>

<VirtualHost *:80>

	# Reverse Proxy to Gunicorn
	# -------------------------
	# For security reasons, You must set "ProxyRequests" to "off".
	# See: <https://httpd.apache.org/docs/current/mod/mod_proxy.html#proxyrequests>
	ProxyRequests    off
	ProxyPass        "${v_gunicorn_location}" "balancer://gunicorn_set"
	ProxyPassReverse "${v_gunicorn_location}" "balancer://gunicorn_set"
	
</VirtualHost> 
```

### Step 3: Reload
* If you're using SysV: `service apache2 reload`
* If you're using OpenRC: `rc-service apache2 reload`
* If you're using Systemd: `systemctl reload apache2`

### Step 4: Check it
Use a browser and see if it works. Also, check for errors within the logs.
Congrats! You made it!
