!SLIDE bullets

# Extending Puppet



!SLIDE bullets

# Hi, I&#8217;m Richard Crowley

* Equal opportunity technology hater.
* DevStructure&#8217;s operator and UNIX hacker.



!SLIDE bullets

## Part 1

# Custom types

## (Warmup.)



!SLIDE bullets

# TODO



!SLIDE bullets

# TODO Providers for existing types



!SLIDE bullets

# TODO puppet-pip



!SLIDE bullets

# Custom functions

* TODO



!SLIDE bullets

## Part 2

# WTF is an indirector?

## (It&#8217;s not even in the dictionary.)



!SLIDE bullets

# How I&#8217;m running Puppet<br />in this demo

	@@@ sh
	# Master
	sudo env RUBYLIB=$HOME/work/puppet/lib \
		puppet master --no-daemonize --verbose

	# Agent
	sudo env RUBYLIB=$HOME/work/puppet/lib \
		puppet agent --no-daemonize --onetime \
		--verbose



!SLIDE bullets

# Node terminus

	$ puppet --genconfig | grep node_terminus
	    # node_terminus = plain
	$



!SLIDE bullets small

# `lib/puppet/indirector/ node/plain.rb`

	@@@ Ruby
	require 'puppet/node'
	require 'puppet/indirector/plain'

	class Puppet::Node::Plain < Puppet::Indirector::Plain
	  # ...

	  # Just return an empty node.
	  def find(request)
	    node = super
	    node.fact_merge
	    node
	  end
	end



!SLIDE bullets smaller

# `lib/puppet/indirector/plain.rb`

	@@@ Ruby
	require 'puppet/indirector/terminus'

	# An empty terminus type, meant to just return empty objects.
	class Puppet::Indirector::Plain < Puppet::Indirector::Terminus
	  # Just return nothing.
	  def find(request)
	    indirection.model.new(request.key)
	  end
	end



!SLIDE bullets

# How did we get here?

	@@@ Ruby
	  def find(request)
	    puts caller
	    indirection.model.new(request.key)
	  end



!SLIDE bullets smaller

# How did we get here?

	lib/puppet/indirector/node/plain.rb:15:in `find'
	lib/puppet/indirector/indirection.rb:193:in `find'
	lib/puppet/indirector/catalog/compiler.rb:91:in `find_node'
	lib/puppet/indirector/catalog/compiler.rb:119:in `node_from_request'
	lib/puppet/indirector/catalog/compiler.rb:33:in `find'
	lib/puppet/indirector/indirection.rb:193:in `find'
	lib/puppet/network/http/handler.rb:106:in `do_find'
	lib/puppet/network/http/handler.rb:68:in `send'
	lib/puppet/network/http/handler.rb:68:in `process'
	lib/puppet/network/http/webrick/rest.rb:24:in `service'
	# ...



!SLIDE bullets

# `Puppet::Indirector:: Indirection#find`

* Picks and caches the terminus class.
* Dispatches calls to it.



!SLIDE bullets smaller

# What&#8217;s the node object for?

	@@@ Ruby
	  def find(request)
	    extract_facts_from_request(request)

	    node = node_from_request(request)

	    if catalog = compile(node)
	      return catalog
	    else
	      # This shouldn't actually happen; we should either return
	      # a config or raise an exception.
	      return nil
	    end
	  end



!SLIDE bullets

# A node to compile

	@@@ Ruby
	  def find(request)
	    indirection.model.new(request.key)
	  end

* `indirection.model` returns `Puppet::Node`.
* A `Puppet::Node` has `@parameters` and `@classes`.
* The node terminus should populate `@parameters` and `@classes`.



!SLIDE bullets

# Catalog compilation



!SLIDE bullets smaller

# From `lib/puppet/indirector/ catalog/compiler.rb`

	@@@ Ruby
	  def compile
	    # Set the client's parameters into the top scope.
	    set_node_parameters
	    create_settings_scope

	    evaluate_main

	    evaluate_ast_node

	    evaluate_node_classes

	    evaluate_generators

	    finish

	    fail_on_unevaluated

	    @catalog
	  end



!SLIDE bullets

# Catalog compilation

* `set_node_parameters` and `evaluate_node_classes` handle `@parameters` and `@classes` set by the node terminus.
* `evaluate_ast_node` handles `node` definitions from Puppet code.



!SLIDE bullets

# `@parameters` and `@classes`

* Sound suspiciously like the makings of the external node classifier.



!SLIDE bullets

# Setup external node classifier in `puppet.conf`

	[master]
		external_nodes=/usr/local/bin/classifier
		node_terminus=exec



!SLIDE bullets

# `/usr/local/bin/classifier`

	@@@ sh
	#!/bin/sh
	cat <<EOF
	---
	classes: {}
	parameters:
	  foo: bar
	EOF



!SLIDE bullets

# From `lib/puppet/ indirector/node/exec.rb`

	@@@ Ruby
	  def find(request)
	    output = super or return nil

	    # Translate the output to ruby.
	    result = translate(request.key, output)

	    create_node(request.key, result)
	  end



!SLIDE bullets

# From `lib/puppet/ indirector/exec.rb`

	@@@ Ruby
	  def find(request)
	    # Run the command.
	    unless output = query(request.key)
	      return nil
	    end

	    # Translate the output to ruby.
	    output
	  end



!SLIDE bullets

# How did we get here?

	@@@ Ruby
	  def find(request)
	    puts caller
	    # Run the command.
	    unless output = query(request.key)
	      return nil
	    end

	    # Translate the output to ruby.
	    output
	  end



!SLIDE bullets smaller

# How did we get here?

	lib/puppet/indirector/node/exec.rb:17:in `find'
	lib/puppet/indirector/indirection.rb:193:in `find'
	lib/puppet/indirector/catalog/compiler.rb:91:in `find_node'
	lib/puppet/indirector/catalog/compiler.rb:119:in `node_from_request'
	lib/puppet/indirector/catalog/compiler.rb:33:in `find'
	lib/puppet/indirector/indirection.rb:193:in `find'
	lib/puppet/network/http/handler.rb:106:in `do_find'
	lib/puppet/network/http/handler.rb:68:in `send'
	lib/puppet/network/http/handler.rb:68:in `process'
	lib/puppet/network/http/webrick/rest.rb:24:in `service'
	# ...



!SLIDE bullets

# Node terminus indirected

* As expected, everything&#8217;s the same except where the indirector looked for the node.

* Where&#8217;s that `foo` parameter injected by the external node classifier?



!SLIDE bullets smaller

# From `lib/puppet/ indirector/catalog/compiler.rb`

	@@@ Ruby
	  def find_node(name)
	    begin
	      return nil unless node =
	        Puppet::Node.indirection.find(name)
	    rescue => detail
	      puts detail.backtrace if Puppet[:trace]
	      raise Puppet::Error,
	        "Failed when searching for node #{name}: #{detail}"
	    end


	    # Add any external data to the node.
	    add_node_data(node)

	    node
	  end



!SLIDE bullets smaller

# What&#8217;s in a `node`?

	@@@ Ruby
	  def find_node(name)
	    begin
	      return nil unless node =
	        Puppet::Node.indirection.find(name)
	    rescue => detail
	      puts detail.backtrace if Puppet[:trace]
	      raise Puppet::Error,
	        "Failed when searching for node #{name}: #{detail}"
	    end


	    # Add any external data to the node.
	    add_node_data(node)

	    puts node.inspect
	    node
	  end



!SLIDE bullets smaller

# What&#8217;s in a `node`?

	#<Puppet::Node:0xb70fa674 @name="devstructure.hsd1.ca.comcast.net.",
	@classes={}, @expiration=Sat Jan 08 22:47:33 +0000 2011,
	@parameters={"network_eth1"=>"33.33.33.0", "netmask"=>"255.255.255.0",
	"kernel"=>"Linux", "processorcount"=>"1", "swapfree"=>"998.67 MB",
	"physicalprocessorcount"=>"0", "uniqueid"=>"007f0101",
	"lsbmajdistrelease"=>"10", "fqdn"=>"devstructure.hsd1.ca.comcast.net.",
	"operatingsystemrelease"=>"10.10", "virtual"=>"physical",
	"ipaddress"=>"10.0.2.15", "memorysize"=>"346.15 MB",
	"is_virtual"=>"false", :_timestamp=>Sat Jan 08 22:17:32 +0000 2011,
	"clientversion"=>"2.6.4", "hardwaremodel"=>"i686",
	"kernelrelease"=>"2.6.35-22-generic-pae",
	"rubysitedir"=>"/usr/local/lib/site_ruby/1.8", "ps"=>"ps -ef",
	"macaddress_eth0"=>"08:00:27:8e:9e:65", "domain"=>"hsd1.ca.comcast.net.",
	"netmask_eth0"=>"255.255.255.0", "serverip"=>"10.0.2.15",
	"servername"=>"devstructure.hsd1.ca.comcast.net.", "timezone"=>"UTC",
	"uptime_days"=>"9", "macaddress_eth1"=>"08:00:27:c3:78:5a", "id"=>"root",
	"netmask_eth1"=>"255.255.255.0", "processor0"=>"Intel(R) Core(TM)2 Duo CPU
	T8300  @ 2.40GHz", "hardwareisa"=>"unknown", "lsbdistrelease"=>"10.10",
	"selinux"=>"false", "manufacturer"=>"innotek GmbH", "interfaces"=>"eth0,eth1",
	"memoryfree"=>"225.10 MB", "uptime_hours"=>"216", "foo"=>"bar",
	"lsbdistdescription"=>"Ubuntu 10.10", "lsbdistcodename"=>"maverick", "kernelversion"=>"2.6.35", "path"=>"/home/vagrant/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/var/lib/gems/1.8/bin", "hostname"=>"devstructure", "uptime"=>"9 days", "puppetversion"=>"2.6.4", "environment"=>"production", "serialnumber"=>"0", "macaddress"=>"08:00:27:8e:9e:65", "facterversion"=>"1.5.8", "ipaddress_eth0"=>"10.0.2.15", "kernelmajversion"=>"2.6", "swapsize"=>"1011.00 MB", "sshrsakey"=>"AAAAB3NzaC1yc2EAAAADAQABAAABAQDrJ0U+iSED6LNmnC0bHfpLKaDvWh0UXgQzdElhtY1xOS065TRMJIrb6LPaAxJH3iWXLP57rM8Vldurx+pOJWVz6ZmsGSz/EOugg6rC6K2cPAS8V5nRq6SUKksb/eBR0LbM0ygMoVEKi0ogBIkQuZNeVqdUtIcT5P7DBrasPkxzkBqxqAgYxzMeR82vYSLBEwhpeyrVUzpOHLh9mVxfCQoZRdEpnckGmMxYSOW7OBzf46CBhPrXAB5ZX3aZB+iDxNhMVvMYZFircDb/ZKZ4ph6qHFVWvBtd54N9yXf+XSLZPfox4zIRR8BRoUz8rx7ccZDZ6SWVdMCZ6jO3MuxEIeFJ", "operatingsystem"=>"Ubuntu", "serverversion"=>"2.6.4", "network_eth0"=>"10.0.2.0", "lsbdistid"=>"Ubuntu", "architecture"=>"i386", "uptime_seconds"=>"779012", "sshdsakey"=>"AAAAB3NzaC1kc3MAAACBAJNDwltq7JCbq7ql1a3g2IxLWwhJNLxk0jYj/oLLc7LvpiN1kcfuNoK2bZooUCnGkcyZs+6U3jbz2idISOncq+hfFjrMv0qIGKHXWANh1qLZuhRjBHwRlkD6ll5gw4ioTohmt3VfUbE4hJfK0z3wtQn/SLLoz4MSNnR09OIZOCgzAAAAFQDLW+bfrDl+WdsE4ixDCRr4vEW1lwAAAIBt+Bz62PhQLZSWpLaHCOOaFeoHwv9IPvQ/ors0zDOxDEoK5GHOY3BcCO8s4rHr17aQKmMP5ztNwzUBl+OlkSlQ7kAEMBRvehFbhK+SOcjvqIt6i2i9/3nn9ba0AZ8bQ+1T8Z+A/6lRmWtaeUqsZNRJitfzez7eWRfvd+X/GK1eaAAAAIAqyACyAElrdGd4YfwV/YPGjiUpIjvzqiBCogO2qvMj6/ohzdN7wBmIdIUmYQ1uYsQ6avd3GL5S2I/xJ1uGosIh6xlKa0Dk4SqOq1LGebdCpv1aUKIwecQlkxrQE6Da9Q4zVgHrtlglOW7A8uq8gfl/VQdwU1sow36scLSE8PCn/g==", "rubyversion"=>"1.8.7", "ipaddress_eth1"=>"33.33.33.33", "productname"=>"VirtualBox", "clientcert"=>"devstructure.hsd1.ca.comcast.net."}, @time=Sat Jan 08 22:17:33 +0000 2011, @environment="production">

* `foo` is indeed `bar` but where&#8217;d the rest of the facts come from?



!SLIDE bullets small

# From `lib/puppet/ indirector/node/exec.rb`

	@@@ Ruby
	  def create_node(name, result)
	    node = Puppet::Node.new(name)
	    set = false
	    [:parameters, :classes, :environment].each do |param|
	      if value = result[param]
	        node.send(param.to_s + "=", value)
	        set = true
	      end
	    end

	    node.fact_merge
	    node
	  end



!SLIDE bullets small

# What&#8217;s in a `node`?

	@@@ Ruby
	  def create_node(name, result)
	    node = Puppet::Node.new(name)
	    set = false
	    [:parameters, :classes, :environment].each do |param|
	      if value = result[param]
	        node.send(param.to_s + "=", value)
	        set = true
	      end
	    end

	    puts node.inspect
	    node.fact_merge
	    node
	  end



!SLIDE bullets

# What&#8217;s in a `node`?

	#<Puppet::Node:0xb71eb704
	@name="devstructure.hsd1.ca.comcast.net.",
	@classes={}, @parameters={"foo"=>"bar"},
	@time=Sat Jan 08 22:23:12 +0000 2011>



!SLIDE bullets smaller

# From `lib/puppet/node.pp`

	@@@ Ruby
	  def fact_merge
	      if facts = Puppet::Node::Facts.indirection.find(name)
	        merge(facts.values)
	      end
	  rescue => detail
	      error = Puppet::Error.new(
	        "Could not retrieve facts for #{name}: #{detail}")
	      error.set_backtrace(detail.backtrace)
	      raise error
	  end



!SLIDE bullets

# Facts terminus

	$ puppet --genconfig | grep facts_terminus
	    # facts_terminus = facter
	$



!SLIDE bullets smaller

# From `lib/puppet/ indirector/facts/facter.rb`

	@@@ Ruby
	  def find(request)
	    result = Puppet::Node::Facts.new(request.key, Facter.to_hash)

	    result.add_local_facts
	    result.stringify
	    result.downcase_if_necessary

	    result
	  end



!SLIDE bullets smaller

# What are our options?

	$ ls -l lib/puppet/indirector/facts/
	total 24
	-rw-r--r-- 1 rcrowley rcrowley 1067 2010-12-22 01:10 active_record.rb
	-rw-r--r-- 1 rcrowley rcrowley  674 2010-12-22 01:10 couch.rb
	-rw-r--r-- 1 rcrowley rcrowley 2296 2010-12-22 01:10 facter.rb
	-rw-r--r-- 1 rcrowley rcrowley  358 2010-12-22 01:10 memory.rb
	-rw-r--r-- 1 rcrowley rcrowley  262 2010-12-22 01:10 rest.rb
	-rw-r--r-- 1 rcrowley rcrowley  235 2010-12-22 01:10 yaml.rb
	$



!SLIDE bullets smaller

# See the pattern?

	lib/puppet/indirector/node/plain.rb:15:in `find'
	lib/puppet/indirector/indirection.rb:193:in `find'
	lib/puppet/indirector/catalog/compiler.rb:91:in `find_node'
	lib/puppet/indirector/catalog/compiler.rb:119:in `node_from_request'
	lib/puppet/indirector/catalog/compiler.rb:33:in `find'
	lib/puppet/indirector/indirection.rb:193:in `find'
	# ...

	lib/puppet/indirector/node/exec.rb:17:in `find'
	lib/puppet/indirector/indirection.rb:193:in `find'
	lib/puppet/indirector/catalog/compiler.rb:91:in `find_node'
	lib/puppet/indirector/catalog/compiler.rb:119:in `node_from_request'
	lib/puppet/indirector/catalog/compiler.rb:33:in `find'
	lib/puppet/indirector/indirection.rb:193:in `find'
	# ...

* The catalog compiler is itself an indirection we can change!



!SLIDE bullets

# Catalog terminus

	$ puppet --genconfig | grep catalog_terminus
	    # catalog_terminus = compiler
	$



!SLIDE bullets smaller

# What are our options?

	$ ls -l lib/puppet/indictor/catalog/
	total 24
	-rw-r--r-- 1 rcrowley rcrowley 1198 2010-12-22 01:11 active_record.rb
	-rw-r--r-- 1 rcrowley rcrowley 4933 2011-01-08 22:18 compiler.rb
	-rw-r--r-- 1 rcrowley rcrowley  140 2010-12-22 01:10 queue.rb
	-rw-r--r-- 1 rcrowley rcrowley  189 2010-12-22 01:10 rest.rb
	-rw-r--r-- 1 rcrowley rcrowley  534 2010-12-22 01:10 yaml.rb
	$



!SLIDE bullets

# Blazing our own trail



!SLIDE bullets

# DNS TXT record<br />node terminus



!SLIDE bullets smaller

# `lib/puppet/indirector/node/dns.rb`

	@@@ Ruby
	require 'puppet/node'
	require 'puppet/indirector/plain'
	require 'resolv'

	class Puppet::Node::Dns < Puppet::Indirector::Plain

	  def find(request)
	    node = super
	    begin
	      resolver = Resolv::DNS.new
	      resource = resolver.getresource(
	        request.key, Resolv::DNS::Resource::IN::TXT)
	      node.classes += resource.data.split
	    rescue Resolv::ResolvError
	    end
	    node.fact_merge
	    node
	  end

	end



!SLIDE bullets

# Example record

	foo.example.com. IN TXT foo bar baz quux
	#   certname   #        #   @classes   #



!SLIDE bullets

# Hierarchical<br />facts terminus



!SLIDE bullets smaller

# `lib/puppet/indirector/facts/hier.rb`

	@@@ Ruby
	require 'puppet/indirector/facts/facter'

	class HierValue < Hash
	  attr_accessor :top
	  def initialize(top=nil)
	    @top = top
	  end
	  def to_s
	    @top
	  end
	end

	class Puppet::Node::Facts::Hier < Puppet::Node::Facts::Facter
	  def destroy(facts)
	  end
	  def save(facts)
	  end
	end



!SLIDE bullets smaller

# `lib/puppet/indirector/facts/hier.rb`

	@@@ Ruby
	class Puppet::Node::Facts::Hier < Puppet::Node::Facts::Facter
	  def find(request)
	    hier = {}
	    super.values.reject do |key, value|
	      Symbol === key
	    end.each do |key, value|
	      value = value.split(",") if value.index(",")
	      h = hier
	      if key.index("_") and keys = key.split("_")
	        while 1 < keys.length and key = keys.shift
	          h = HierValue === h[key] ?
	            h[key] : h[key] = HierValue.new(h[key])
	        end
	        key = keys.shift
	      end
	      if HierValue === h[key]
	        h[key].top = value
	      else
	        h[key] = value
	      end
	    end
	    Puppet::Node::Facts.new(request.key, hier)
	  end
	end



!SLIDE bullets smaller

# Example

	{"kernel"=>"Linux",
	 "netmask"=>{"eth0"=>"255.255.255.0", "eth1"=>"255.255.255.0"},
	 "ipaddress"=>{"eth0"=>"10.0.2.15", "eth1"=>"33.33.33.33"},
	 "kernelrelease"=>"2.6.35-22-generic-pae",
	 "ps"=>"ps -ef",
	 "network"=>{"eth0"=>"10.0.2.0", "eth1"=>"33.33.33.0"},
	 "interfaces"=>["eth0", "eth1"],
	 "kernelversion"=>"2.6.35",
	 "puppetversion"=>"2.6.4",
	 "hostname"=>"devstructure",
	 "uptime"=>{"seconds"=>"858930", "days"=>"9", "hours"=>"238"}}

	facts["ipaddress"] # => "10.0.2.15"
	facts["ipaddress"]["eth1"] # => "33.33.33.33"



!SLIDE bullets

# Caching catalog terminus



!SLIDE bullets smaller

# `lib/puppet/indirector/ catalog/caching_compiler.rb`

	@@@ Ruby
	require 'puppet/indirector/catalog/compiler'

	class Puppet::Resource::Catalog::CachingCompiler <
	  Puppet::Resource::Catalog::Compiler

	  @commit, @cache = `git rev-parse HEAD`.chomp, {}

	  def self.cache
	    commit = `git rev-parse HEAD`.chomp
	    @commit, @cache = commit, {} if @commit != commit
	    @cache
	  end

	  def find(request)
	    self.class.cache[request.key] ||= super
	  end

	end



!SLIDE bullets

# Limitations

* In-process cache is not<br />shared between masters.
* Invalidation brings a<br />thundering herd when `HEAD` moves.



!SLIDE bullets

# Thank you

* [github.com/rcrowley/puppet/tree/demo](https://github.com/rcrowley/puppet/tree/demo)

* <richard@devstructure.com> or [@rcrowley](http://twitter.com/rcrowley)
* P.S. use DevStructure.
