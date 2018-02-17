---
title: "Sensible nginx config management"
date: 2018-02-16T12:04:52+01:00
keywords: []
draft: false
---

At my last work we had a basic nginx reverse proxy that load balanced all requests between two backends, each in a separate private cloud data center.
The basic requirement was to provide resiliency in case of a data center outage.

Adding a load balancer to your stack (like any new moving part) requires some careful consideration.
In this post I will highlight the setup we decided upon to make operations as simple and straightforward as possible.

## Managing configuration changes

Use version control, duh!
What we found to be most reliable is checking in the *entire* nginx config directory into git, cloning this repo on the server and then pointing the nginx config directory to this repo.

If you run `nginx -V` you can see under `--conf-path=/usr/local/etc/nginx/nginx.conf` where nginx is getting its config.
Say you have your repository checked out at `~/nginx-config`, then you could symlink these folders by running: `ln -sf /usr/local/etc/nginx/ ~/nginx-config`.

Now, if you have changes you want to bring live simply go to your server pull the latest changes and reload nginx: `cd ~/nginx-config && git pull && nginx -s reload`

If the new config is invalid nginx will continue using its previous config. In any case you will probably (i hope) test your changes locally or on a staging server first.

## Environment specific configuration

If you have more than one environment (test, accept, production) you're more than likely going to want to reuse the config.
But how do you deal with environment specific values?

Well you could:

- maintain a config version per environment
- set environment variables on your host and reference them from your config with the nginx [env directive](http://nginx.org/en/docs/ngx_core_module.html#env)
- use nginx scripting abilities to define at runtime which values should be chosen (according to request headers maybe)

The above seemed too finicky so we chose a cleaner way of doing things: making use of a templating language.
You can store your environment specific values in a json file, and reference them in your nginx template files.
Using handlebars and a node.js build script we empowered a simple and straightforward reuse of config.

Say you have the following config that loadbalances requests across two tomcat application servers:

```
upstream backend {
    ip_hash;

    # List of Tomcat application servers
    server 10.100.100.11:8080;
    server 10.100.100.12:8080;
}

server {
	listen 443 ssl;
	server_name example.com;

	# SSL stuff...

	location / {
		proxy_pass http://backend;
		#  other stuff...
	}
}
```

The backend ip addresses will probably not be the same across environments. You can extract these values to a json file `config/staging.json`:

```
{
	"application_servers": {
		"left": "10.100.100.11",
		"right": "10.100.100.12"
	}
}
```

And let's make another file for production `config/production.json`:

```
{
	"application_servers": {
		"left": "10.100.100.1",
		"right": "10.100.100.2"
	}
}
```

Reference the values from your nginx config template:

```
upstream backend {
    ip_hash;
    
    server {{application_servers.left}}:8080;
    server {{application_servers.right}}:8080;
}
```

So you end up with something like the following:

```
│── config
│   ├── production.json
│   └── staging.json
└── src
    ├── loadbalanced-servers.tpl.conf
    └── nginx.tpl.conf
```

We want every source file to be compiled for each environment defined in the `config` directory.
The result should look something like this:

```
target
├── production
│   ├── loadbalanced-servers.conf
│   └── nginx.conf
└── staging
    ├── loadbalanced-servers.conf
    └── nginx.conf
```

This is where some basic scripting comes in use.
We defined a `build.js` file that compiles each template file with as input each json file, leaving you with the disered output shown above.

```
#!/usr/bin/env node

var shell = require("shelljs");
var _ = require("lodash");
var fs = require('fs');
var path = require('path');
var handlebars = require('handlebars');
var recursive = require('recursive-readdir');

var paths = {
    src: 'src',
    target: 'target'
};

// clean up target
shell.rm('-rf', paths.target + '/');

var configs = [
    _.assign({env: 'staging'}, require('./config/staging.json')),
    _.assign({env: 'production'}, require('./config/production.json'))
];


// for each config file
_.forEach(configs, function (config) {

    // read each template file
    recursive(paths.src, function (error, templates) {
        _.forEach(templates, function (templateFile) {

            // compile template file to target/env folder
            var outputFolder = path.join(paths.target, config.env);
            var outputPath = templateFile.replace(paths.src, outputFolder);
            outputPath = outputPath.replace('.tpl.conf', '.conf');

            var templateFileContent = fs.readFileSync(templateFile, 'utf8');
            var template = handlebars.compile(templateFileContent);
            var output = template(config);
            shell.mkdir('-p', path.dirname(outputPath));
            fs.writeFileSync(outputPath, output);

        })
    });
});
```

On your server you would now symlink to the correct target folder.
Before committing into git you run build.js first.
To update the server config all you have to do is pull the latest config and reload nginx.
Nice and simple.

Checkout [nginx-template-based-config](https://github.com/Arnovw/nginx-template-based-config) for all the code.
