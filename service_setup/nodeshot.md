# phillymesh.net Nodeshot

*This is a collection of notes for installing [NodeShot](https://github.com/ninuxorg/nodeshot), for mapping mesh nodes geographically. NodeShot is currently only in development use for Philly Mesh and is not ready for production.*

## Set Up NodeShot

1. The best way to install NodeShot is to follow the official documentation to the letter. The documentation is available here, http://nodeshot.readthedocs.io/en/latest/topics/install_production_manual.html

## Update 2017-03-28 - Map Tiles

At the time of this update, the map tiles server provided by OpenStreetMap is down for some reason. We can get around this by using one of two alternate map tile services.

This generated a ticket, https://github.com/ninuxorg/nodeshot/issues/272

1. MapBox has a lot of cool different maps, get an API Key at https://www.mapbox.com/studio/ and modify `/var/www/nodeshot/phillymesh/settings.py` replacing the LEAFLETS_CONFIG.update variable with:

```
LEAFLET_CONFIG.update({
    'DEFAULT_CENTER': (49.06775, 30.62011),
    'DEFAULT_ZOOM': 4,
    'MIN_ZOOM': 1,
    'MAX_ZOOM': 18,
    # Uncomment to customize map tiles
    # More information here: https://github.com/makinacorpus/django-leaflet#default-tiles-layer
    'TILES': [
        ('Map', 'https://api.tiles.mapbox.com/v4/mapbox.streets/{z}/{x}/{y}.png?access_token=YOURAPIKEY', '&copy; <a href="ht
tp://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a> contributors | Tiles Courtesy of <a href="http://www.mapquest.com/" target="_blank">MapQuest</a> &nbsp;<img src="https://developer.mapqu
est.com/content/osm/mq_logo.png">'),
        ('Satellite', 'https://api.tiles.mapbox.com/v4/mapbox.satellite/{z}/{y}/{x}.png?access_token=YOURAPIKEY', 'Source: <a
 href="http://www.esri.com/">Esri</a> &copy; and the GIS User Community ')
    ],
})
```

1. Alternatively, there is a public tiles API with no key needed. Modify `/var/www/nodeshot/phillymesh/settings.py` replacing the LEAFLETS_CONFIG.update variable with:

```
LEAFLET_CONFIG.update({
    'DEFAULT_CENTER': (49.06775, 30.62011),
    'DEFAULT_ZOOM': 4,
    'MIN_ZOOM': 1,
    'MAX_ZOOM': 18,
    # Uncomment to customize map tiles
    # More information here: https://github.com/makinacorpus/django-leaflet#default-tiles-layer
    'TILES': [
        ('Map', 'https://a.tile.openstreetmap.org/{z}/{x}/{y}.png', '&copy; <a href="ht
tp://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a> contributors | Tiles Courtesy of <a href="http://www.mapquest.com/" target="_blank">MapQuest</a> &nbsp;<img src="https://developer.map$
est.com/content/osm/mq_logo.png">'),
        ('Satellite', ''http://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}.png', 'Source: <a
 href="http://www.esri.com/">Esri</a> &copy; and the GIS User Community ')
    ],
})
```

1. When done, restart with `suervisor restart all`

## Update 2017-03-28 - No Admin Account

When doing a manual production installation, there is no admin account configured. Because of this, you can't configure many settings.

This generated a ticket, https://github.com/ninuxorg/nodeshot/issues/275

To get around this, I needed to register an account normally and grant it proper permissions via the postgres console.

1. Go to your postgres user, `su postgres`.

1. Enter the console for the `nodeshot` database, `psql nodeshot`.

1. Run the following commands to grant staff and superuser privs to our first account:

```
postgres=# update profiles_profile set is_staff = true where id = 1;
postgres=# update profiles_profile set is_superuser = true where id = 1
postgres=# \q
```

1. Now access the admin console at https://yournodeshotsite.com/admin to get the admin console and login with your nodeshot site account.

## Update 2017-03-28 - Can't Create Layer

When trying to add a node, I realized this wasn't possible until a layer is set up. It looks like this can only be done via the admin console. So, I logged in via /admin/ and attempted to create a layer but would trigger a 500 Internal Server Error whenever I saved.

This generated a ticket, https://github.com/ninuxorg/nodeshot/issues/274

The ticket contains the error message from ` /var/www/nodeshot/log/nodeshot.error.log`.

## Update 2017-03-28 Outgoing Email Timing Out

I noticed that the confirmation emails coming out of my postfix server were timing out. This was fixed by messaging Vultr support. They were able to remove the outgoing email block on my machine in under a minute, afterwards the machine must be restarted from the Vultr console, NOT from within the VPS.

If you want to suck the email out of the mail queue, check the mail.log via `tail /var/log/mail.log`.

You will see some failed email messages in the log, prefixed with a 10-character hex code. Note this code and dump it to a file via `postcat -qbh YOURCODE > tempfile.eml`.

Just cat the file to see the message, which will contain the activiation link for your account, `cat tempfile.eml`.

Unfortunately, NodeShot only allows the use of a local outgoing mailserver with no TLS.
