---
layout: post
title:  "CSV and SQL list of geocoded Belgian ZIP/Cities"
date:   2012-07-12 20:49:25 +0200
categories: dev web php
---
For a project I needed a list of cities with their latitude/longitude location.

I used the technic mentionned by Patrick on his blog :

http://www.jedi.be/blog/2009/04/29/geocoding-belgium-postal-codes/

and I grabbed the postal code/city CSV file from Vincent’s blog :

http://www.vinch.be/blog/2010/03/16/villes-de-belgique-aux-formats-csv-xml-json-et-sql/

Aaaand some scripting later, I had a script ready to fill those lat/lng fields I prepared in my SQL table :

{% highlight php %}
$cities = database::getAll('SELECT * FROM cities 
                     WHERE lat IS NULL and lng IS NULL');
foreach($cities as $city) { 
  list($id,$zip,$name) = $city;
  $data = shell_exec('curl -s "http://maps.google.com/maps/geo?q='.
    urlencode(utf8_encode($name)).
    '+'.$zip.'+Belgie&amp;output=csv&amp;key="')."\n";
  list($junk,$junk,$lat,$lng) = explode(',',$data);
  echo $id." ".$zip." ".utf8_encode($name).' '.$lat.' '.$lng.", saving...\n";
  if((int) $lat != 0) 
    database::query('UPDATE cities SET lat = ? , lng =? WHERE id = ?',
      array($lat,$lng,$id));
}

{% endhighlight %}
