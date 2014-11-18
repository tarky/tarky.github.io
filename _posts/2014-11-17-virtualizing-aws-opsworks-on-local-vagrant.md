---
layout: post
title:  "Virtualizing AWS OpsWorks on Local Vagrant"
date:   2014-11-17 17:37:35
categories: AWS OpsWorks Vagrant 
---
##Why to Do It
---

AWS OpsWorks is a powerful cloud application management tool.

But the only weak point is that OpsWorks enable us to build environments only on AWS. So we can't test chef recipes on local environment. Also one of the chef's strong point that it can enable us to build environment everywhere dissapears. The reason why we can build environment only on AWS is that OpsWokrs agent is preinstalled to each ec2 instance and it cooperates with OpsWorks builtin recipes to configure various settings when setup.

So we have to launch instances every time we test recipes and then we are charged. I hate being charged with such a subtle operation and it is ideal to develop in the same environment as the production. So I managed to virtualize OpsWorks on Vagrant. I did it with Rails project, but it can be applied to other kind of project.


##Procedure
---

###Step 1. Making Vagrant box file with OpsWorks Agent installed

[https://github.com/wwestenbrink/vagrant-opsworks](https://github.com/wwestenbrink/vagrant-opsworks)

Using this tool, make a box file. Make sure things in dependencies list are installed in your machine and proceed according to usage.

###Step 2. Make an environment which you want virtulize on  OpsWorks as usual. Get node information when setup

Step 1 will take several tens of minutes. During it, make an environment on OpsWorks as usual. And ssh login with terminal, and get json of node information when setup with following command.

```
sudo opsworks-agent-cli get_json setup
```

OpsWorks saves a json file of node information used by chef every time OpsWorks manipulate ec2 instances. There are several kinds of OpsWorks manipulation such as setup, deploy, configure and so on. The command above is display node information of the most recent setup manipulation. With the json and opsworks-agent-cli, we can virtualize the process run by OpsWorks on instances.

###Step 3. Make a Vagrantfile and cookbooks for local setup

Those are put in rails project. Note that the cookbooks is different from custom cookbooks used by OpsWorks as usual. The custom cookbooks is managed in the other repo, as the official document say.

~~~
├── Gemfile
├── Gemfile.lock
├── README.rdoc
├── Rakefile
├── Vagrantfile
├── app
├── bin
├── config
├── config.ru
├── cookbooks
 |        └── mimic_opsworks
 |                  ├── attributes
 |                  │        └── default.rb
 |                  ├── recipes
 |                  │        ├── default.rb
 |                  │        └── link_local.rb
 |                  └── templates
 |                             └── default
 |                                      └── json.erb
├── db
├── lib
├── log
├── public
├── test
├── tmp
└── vendor
~~~

Vagrantfile

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  app_name = "yourapp"
  config.vm.box = "ubuntu1404-opsworks" #specify the name of the box file created at step1
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.synced_folder ".", "/vagrant", type: 'nfs'
  config.vm.provision "shell", inline: "apt-get update > /dev/null"
  config.vm.provision "chef_solo" do |chef|  
    chef.json = { app_name: app_name }
    chef.run_list = ["mimic_opsworks::default"]
  end
  config.vm.provision "shell", inline: "opsworks-agent-cli run_command"
  config.vm.provision "chef_solo" do |chef|  
    chef.run_list = ["mimic_opsworks::link_local"]
  end
  config.vm.provision "shell", inline: "cd /srv/www/#{app_name}/current; bundle install"
end
{% endhighlight %}

Mounting synced folder via nfs in order that the synced folder can be written by Rails app.

    config.vm.synced_folder ".", "/vagrant", type: 'nfs'


I dont know why errors happen without following.

    config.vm.provision "shell", inline: "apt-get update > /dev/null"

Note that during the process of the line above, some red color warnings might be displayed. But it's negligible.

The following is to put modified json which is originally from step2 into a certain directory.
{% highlight ruby %}
config.vm.provision "chef_solo" do |chef|  
  chef.json = { app_name: app_name }
  chef.run_list = ["mimic_opsworks::default"]
end
{% endhighlight %}

The folloning is `mimic_opsworks::default` recipe.

default.rb
{% highlight ruby %}
directory "/var/lib/aws/opsworks/chef" do
  mode '0755'
  action :create
end
template "/var/lib/aws/opsworks/chef/2014-11-11-03-03-49-02.json" do
  source "json.erb"
end
{% endhighlight %}

`json.erb` which I made is below. But it might differ depending on the structure of your system. So it's highly recommended that you create it from step2 json by yourself. I will show you the points of modification after the following code.

json.erb

~~~
{
  "ssh_users": {
  },
  "opsworks": {
    "agent_version": "328",
    "layers": {
      "db-master": {
        "instances": {
        }
      },
      "rails-app": {
        "name": "Rails App Server",
        "id": "99999999-9999-9999-9999-999999999999",
        "elb-load-balancers": [

        ],
        "instances": {
          "rails-app1": {
            "public_dns_name": "ec2-99-99-99-99.ap-northeast-1.compute.amazonaws.com",
            "private_dns_name": "ip-99-99-99-99.ap-northeast-1.compute.internal",
            "backends": 8,
            "ip": "99.99.99.99",
            "private_ip": "99.99.99.99",
            "instance_type": "c3.large",
            "status": "online",
            "id": "99999999-9999-9999-9999-999999999999",
            "aws_instance_id": "i-99999999",
            "elastic_ip": null,
            "created_at": "2014-11-08T10:16:10+00:00",
            "booted_at": "2014-11-10T00:56:37+00:00",
            "region": "ap-northeast-1",
            "availability_zone": "ap-northeast-1a",
            "infrastructure_class": "ec2"
          }
        }
      },
      "postgres": {
        "name": "postgres",
        "id": "99999999-9999-9999-9999-999999999999",
        "elb-load-balancers": [

        ],
        "instances": {
          "rails-app1": {
            "public_dns_name": "ec2-99-99-99-99.ap-northeast-1.compute.amazonaws.com",
            "private_dns_name": "ip-99-99-99-99.ap-northeast-1.compute.internal",
            "backends": 8,
            "ip": "99.99.99.99",
            "private_ip": "99.99.99.99",
            "instance_type": "c3.large",
            "status": "online",
            "id": "99999999-9999-9999-9999-999999999999",
            "aws_instance_id": "i-99999999",
            "elastic_ip": null,
            "created_at": "2014-11-08T10:16:10+00:00",
            "booted_at": "2014-11-10T00:56:37+00:00",
            "region": "ap-northeast-1",
            "availability_zone": "ap-northeast-1a",
            "infrastructure_class": "ec2"
          }
        }
      }
    },
    "activity": "setup",
    "valid_client_activities": [
      "reboot",
      "stop",
      "deploy",
      "setup",
      "configure",
      "update_dependencies",
      "install_dependencies",
      "update_custom_cookbooks",
      "execute_recipes"
    ],
    "sent_at": 9999999999,
    "deployment": null,
    "applications": [
      {
      "name": "<%= node[:app_name] %>",
      "slug_name": "<%= node[:app_name] %>",
        "application_type": "rails"
      }
    ],
    "stack": {
      "name": "Railspostgres",
      "id": "99999999-9999-9999-9999-999999999999",
      "vpc_id": "vpc-99999999",
      "elb-load-balancers": [

      ],
      "rds_instances": [

      ]
    },
    "instance": {
      "id": "99999999-9999-9999-9999-999999999999",
      "hostname": "rails-app1",
      "instance_type": "c3.large",
      "public_dns_name": "ec2-99-99-99-99.ap-northeast-1.compute.amazonaws.com",
      "private_dns_name": "ip-99-99-99-99.ap-northeast-1.compute.internal",
      "ip": "99.99.99.99",
      "private_ip": "99.99.99.99",
      "architecture": "x86_64",
      "layers": [
        "postgres",
        "rails-app"
      ],
      "backends": 8,
      "aws_instance_id": "i-99999999",
      "region": "ap-northeast-1",
      "availability_zone": "ap-northeast-1a",
      "subnet_id": "subnet-99999999",
      "infrastructure_class": "ec2"
    },
    "ruby_version": "2.1",
    "ruby_stack": "ruby",
    "rails_stack": {
      "name": "nginx_unicorn"
    }
  },
  "deploy": {
  "<%= node[:app_name] %>": {
  "application": "<%= node[:app_name] %>",
      "application_type": "rails",
      "environment": {
      "RAILS_SECRET_KEY": "<%= node["RAILS_SECRET_KEY"] %>"
      },
      "environment_variables": {
      "RAILS_SECRET_KEY": "<%= node["RAILS_SECRET_KEY"] %>"
      },
      "auto_bundle_on_deploy": true,
      "deploy_to": "/srv/www/<%= node[:app_name] %>",
      "deploying_user": null,
      "document_root": "public",
      "domains": [
      "<%= node[:app_name] %>"
      ],
      "migrate": true,
      "mounted_at": null,
      "rails_env": "development",
      "restart_command": null,
      "sleep_before_restart": 0,
      "ssl_support": false,
      "ssl_certificate": null,
      "ssl_certificate_key": null,
      "ssl_certificate_ca": null,
      "scm": {
        "scm_type": "git",
        "repository": "/vagrant",
        "revision": null,
        "ssh_key": null,
        "user": null,
        "password": null
      },
      "symlink_before_migrate": {
        "config/database.yml": "config/database.yml",
        "config/memcached.yml": "config/memcached.yml"
      },
      "symlinks": {
        "system": "public/system",
        "pids": "tmp/pids",
        "log": "log"
      },
      "database": {
        "adapter": "postgresql",
        "username": "<%= node[:postgresql][:username] %>",
        "password": "<%= node[:postgresql][:password] %>",
        "host": "localhost"
      },
      "memcached": {
        "host": null,
        "port": 11211
      }
    }
  },
  "languages": {
    "ruby": {
      "ruby_bin": "/usr/bin/ruby"
    }
  },
  "rails": {
    "max_pool_size": 8
  },
  "unicorn": {
  },
  "opsworks_custom_cookbooks": {
    "enabled": true,
    "scm": {
      "type": "git",
      "repository": "https://github.com/tarky/ops_berks.git",
      "user": null,
      "password": null,
      "revision": null,
      "ssh_key": null
    },
    "manage_berkshelf": true,
    "berkshelf_version": "3.1.3",
    "recipes": [
      "opsworks_initial_setup",
      "dependencies",
      "opsworks_ganglia::client",
      "unicorn::rails",
      "postgresql::client",
      "postgresql::server",
      "deploy::default",
      "deploy::rails"
    ]
  },
  "chef_environment": "_default",
  "recipes": [
    "opsworks_custom_cookbooks::load",
    "opsworks_custom_cookbooks::execute"
  ],
  "opsworks_commons": {
    "assets_url": "https://opsworks-instance-assets-us-east-1.s3.amazonaws.com"
  },
  "opsworks_berkshelf": {
    "version": "3.1.3",
    "prebuilt_versions": [ "3.1.3"]
  },
  "opsworks_bundler": {
    "version": "1.5.3",
    "manage_package": true
  },
  "opsworks_rubygems": {
    "version": "2.2.2"
  },
  "postgresql": {
    "password": {
      "<%= node[:postgresql][:username] %>": "<%= node[:postgresql][:md5_password] %>"
    }
  }
}
~~~

#### The points of modification of json

* Change the url of app's the source repository to the local repository

In the json which is just gotten at step2, "repository" is set to your app's repo url onthe service like github. But in the local environment, `/vagrant` is also the same repository. So it's better to get source from `/vagrant`. 

Also by this change, you don't have to manage a ssh key to private git repo which on the real OpsWorks you need.


~~~
      "scm": {
        "scm_type": "git",
        "repository": "/vagrant",
~~~ 

* Change rails_env to "development"

~~~
"rails_env": "development",
~~~

This let us do without `asset:precompile`.

* List recipes you want to run

~~~
"berkshelf_version": "3.1.3",
"recipes": [
  "opsworks_initial_setup",
  "dependencies",
  "opsworks_ganglia::client",
  "unicorn::rails",
  "postgresql::client",
  "postgresql::server",
  "deploy::default",
  "deploy::rails"
]
~~~

Here specify the setup recipes which you see on Layer screen of OpsWorks console. Of course include custome cookbooks.

![OpsWorks console screenshot]({{ site.url }}/images/2014-11-16-21-54-09.png)

In my case, I include the setup recipes which belongs to  Rails layer and PostgreSQL Layer. And you might as well remove the recipes you don't need in the local environment. For example, I remove MySQL client, ebs and so on.

* Add the following lines

~~~
  "opsworks_berkshelf": {
    "version": "3.1.3",
    "prebuilt_versions": [ "3.1.3"]
  },
~~~

This is to install the AWS prebuilt version of berkshelf. Without this, the process try to download berkshelf via gem and result in an error.

* Move the sensitive information to attribute

~~~
"RAILS_SECRET_KEY": "<%= node["RAILS_SECRET_KEY"] %>"
~~~

* replace the info like ip address with dummy number

~~~
"private_dns_name": "ip-99-99-99-99.ap-northeast-1.compute.internal",
~~~

So far json.erb is done. I don't remove any item from the json. But you might as well remove the items you don't need to make it smaller. And make sure you make the attribute file. And put it in `.gitignore` because it includes sensitive informations.

Let's go back to Vagrantfile.

The following runs the setup process using node information set by the above `json.erb`.

{% highlight ruby %}
config.vm.provision "shell", inline: "opsworks-agent-cli run_command"
{% endhighlight %}

Actuall so far the virtualization of OpsWorks setup process is complete. If you run the provision process so far and access by a browser, you should see your system working.

The following provisions is to link the synced directory `/vagrant` to the directory where rails app should be to make the development easier. Otherwise you can't see actual changes, immediately after you modify the app's souces because OpsWorks gets apps sources from git service like github.

{% highlight ruby %}
  config.vm.provision "chef_solo" do |chef|  
    chef.run_list = ["mimic_opsworks::link_local"]
  end
{% endhighlight %}

link_local.rb

{% highlight ruby %}
link "/srv/www/#{node['app_name']}/current" do
  action :delete
end
link "/srv/www/#{node['app_name']}/current" do
  to "/vagrant"
end
{% endhighlight %}

After linking, run `bundle install`

{% highlight ruby %}
config.vm.provision "shell", inline: "cd /srv/www/#{app_name}/current; bundle install"
{% endhighlight %}


### Step 4. Make database.yml

Make the file effective on development environment.

database.yml

~~~
development:
  adapter: "postgresql"
  database: ""
  encoding: "utf8"
  host: "localhost"
  username: "xxxxx"
  password: "xxxxx"
  reconnect: false
~~~

### Step 5. Vgrant up

The last to do to make the same environment as AWS OpsWorks on local Vagrant is only `vagrant up`. 

~~~
date ; vagrant up ; date
~~~

It took me 20 minutes to complete.
And Note that you had better run this command at stable network situation. The proccess downloads various things. If the network is broken during the process, errors might happen.

## Conclusion
---
You got it! You got the same environment as OpsWorks on Vagrant! You can test recipes easily. Above all, it decreases the situations such as that the code works in development doesn't work in procuction.

If possible, depending on your system's structure, the process can't go througn. But refferring to my procedure's essences, try it in the various way. 

And if you know smarter way, please tell me.

### Reference
* [wwestenbrink/vagrant-opsworks](https://github.com/wwestenbrink/vagrant-opsworks)
* [Virtualizing AWS OpsWorks with Vagrant](http://pixelcog.com/blog/2014/virtualizing-aws-opsworks-with-vagrant/)
