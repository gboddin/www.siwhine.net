---
layout: post
title:  "vSphere RBVMomi REST interface, POC"
date:   2012-04-29 20:49:25 +0200
categories: dev ruby sysadmin vmware
---

Ok, I hear it from many webdev now, Ruby is easy, Ruby is great, Ruby is … ect :)

I found myself bound to Ruby, because the only descent library (not over-complicated) I could find to connect to a VMWare vSphere server was RBVMomi.

This is just nice, now I want to learn Ruby but… It’s just too bad if you want a simple interface for another language/platform.

Since HTTP is a widely used protocol, I decided to implement a REST bridge between rbvmomi and any other applications I could use.

RBVMomi being in ruby, I decided to use the Sinatra framework.

So far, I just started learning it, and I’m really happy with ruby. That piece of code says it all. I got a POC done quickly and it’s working just awesome for my use :)

## The code

### rbvmomi-server.rb 

{% highlight ruby %}
# File rbvmomi-server.rb 
require 'sinatra'
require 'rbvmomi'
require 'json'
require 'iniparse'
require 'singleton'
require 'redis'
#
# Persistent API server. Lifetime is process lifetime
# Implementing redis as queue storage makes it multithread and restart safe
#
class APIServer
  include Singleton
   
  #internal : singletone init:
  def initialize 
    @config = IniParse.parse( File.read('config.ini') )
    @redis = Redis.new(@config['redis'])
    @vim = RbVmomi::VIM.connect(:host => @config['vsphere']['host'], :user => @config['vsphere']['user'],
      :password=> @config['vsphere']['password'], :insecure => @config['vsphere']['insecure'])
    @dc = @vim.serviceInstance.find_datacenter(@config['vsphere']['datacenter'])
    
  end
  #internal : jobqueue
  def getjob(jobid)
    return JSON.parse(@redis.get(jobid))
    
  end
  def setjob(jobid,data)
    return @redis.set(jobid, data.to_json)
  end
  
  #API functions :
  def _api_getdatastoreinfo(params)
    output = Array.new
    paths = %w(name info.url summary.accessible summary.capacity summary.freeSpace)
    propSet = [{ :type => 'Datastore', :pathSet => paths }]
    filterSpec = { :objectSet => @dc.datastore.map { |ds| { :obj => ds } }, :propSet => propSet }
    data = @vim.propertyCollector.RetrieveProperties(:specSet => [filterSpec])
    data.select { |d| d['summary.accessible'] }.sort_by { |d| d['name'] }.each do |d|
        size = d['summary.capacity']
        free = d['summary.freeSpace']
        used = size - free
        pct_used = used*100.0/size
        if d['name'].match('pcc-')
          output << {'url' => d['info.url'], 'size' => size,  'used' =>  used, 'free' => free, 'percent' => pct_used, 'name' => d['name']}
        end
    end
    return output
  end
  
  def _api_getjob(params)
    return getjob(params['jobid'])
  end
  
  def _api_setjob(params)
    return setjob(params['jobid'],params['data'])
  end
  
end

get '/api/*' do |method|
  APIServer.instance.method(['_api_' , method].join()).call(@params).to_json
end
{% endhighlight %}

### config.ini

{% highlight ini %}
[vsphere]
hostname = xxxx
user = xxxx
password = xxxx
insecure = xxxx
datacenter = xxxx
[redis]
path = /var/run/redis.sock
---or----
host = xxxxx
port = xxxxx
{% endhighlight %}

## Testing

We start the server :

{% highlight bash %}
$ ruby -rubygems rbvmomi-server.rb 
[2012-04-29 14:41:23] INFO  WEBrick 1.3.1
[2012-04-29 14:41:23] INFO  ruby 1.8.7 (2011-06-30) [x86_64-linux]
== Sinatra/1.3.2 has taken the stage on 4567 for development with backup from WEBrick
[2012-04-29 14:41:23] INFO  WEBrick::HTTPServer#start: pid=18829 port=4567
{% endhighlight %}

And we test ! :

{% highlight bash %}
$ curl "http://localhost:1234/api/getdatastoreinfo"
[{"percent":64.4684278916377,"used":188676997120,"free":103988736000,"size":292665733120,"name":"xxx","url":"netfs://3xxxxx/main/"},{"percent":60.7995903847724,"used":178150768640,"free":114862338048,"size":293013106688,"name":"xxx","url":"netfs://2xxx/main/"},{"percent":17.0715763111187,"used":49474035712,"free":240329523200,"size":289803558912,"name":"xxx","url":"netfs://1xxx/main/"}]
$
{% endhighlight %}

Yeaah

Running getjob/setjob

Sorry for my poor Ruby skills, I’m new at it.

