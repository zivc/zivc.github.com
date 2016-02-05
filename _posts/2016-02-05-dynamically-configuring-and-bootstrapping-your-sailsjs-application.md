---
layout:     post
title:      Dynamically configuring and bootstrapping your SailsJS application
date:       2016-02-02 13:30:30
summary:    I'll show you how to rewrite your `app.js` file and dynamically launch your SailsJS application
categories: sails
---

Where I work we had a business requirement that meant our application configuration was goverened by an external source, and I needed to introduce that to Sails. This went as far as dynamically generating models.

Not to give too much away but due to contractual regulations we had to silo off our clients data, meaning that we have different database set ups and policies (mostly for performance) for different clients. When starting Sails usually, you'd have all of these hand-rolled and pre-built, but we needed to do something that did it automatically, that created 5 Models to be used in Waterline, per client, with N number of clients.

Originally, it sounds like you need to do this in the `config/bootstrap.js` file, but unfortunately establishing database connections to be handled/pooled by Sails, you need to do it earlier than here, which for the majority of us would be the `app.js` file.

So for the first step we tore out the entire `app.js` file provided with sails and replaced it with one we rolled ourselves.
<sup>(Remember, this is what works for us and may not necessarily work for you)</sup>

## The app.js file

<pre>
	<code class="javascript">
var program = require('commander'),
    path = require('path'),
    pkg = require(path.resolve(__dirname, './package.json')),
    NOOP = function() { };

process.chdir(__dirname);
process.on('uncaughtException', function (error) { console.log(error.stack); });

program.unknownOption = NOOP;

program
    .version(pkg.version)
    .option('-c, --config [config]', 'Specify config file if remoteConfig.js does not exist')
    .option('--server', 'Run this in a non-dev environment')
    .option('--dev', 'Run this in a dev environment')
    .option('--build', 'Build artifact for production and run tests');

var commands = [
    'build',
    'dev',
    'server',
    'release'
];

commands.forEach(function(command) {
    program
        .command(command)
        .action(require('./tasks/startup/'+command));
});

if (process.argv.filter(function(argument) { return commands.indexOf(argument) !== -1; }).length === 0) process.argv.push('server') ;

program.parse(process.argv);
	</code>
</pre>

This let us use the same command with a different argument to control how we wanted to run the application. Note, these commands are to be ran by people who have no idea what Sails is, or nodejs for that matter - traditional Java developers.

So in the exmaple above we have these steps;
* _build_ - Bundle the *required* dependencies, compile all the JS and CSS and then throw it all into a file and zip it up
* _dev_ - Essentially run `sails console`
* _server_ - Same as `sails lift` but with `--prod` appended to the arguments
* _release_ - Does the build task but also creates a Docker image and sends to the Docker registry

The main crux here though is that all of these arguments correspond to a file that lives inside `tasks/startup/`.

## Our glorified 'sails console' command

Here is what our `tasks/startup/dev.js` looks like:

<pre>
	<code class="javascript">
var path = require('path');
module.exports = function(command) {

    console.log('Launching dev environment');

    require('./generateConfig')(command).then(function() {

        var sails = path.resolve(__dirname, '../../node_modules/sails/bin/sails-console');
        process.chdir(path.resolve(__dirname, '../../'));
        require(sails)();

    }, function(err){
        console.log(err);
    });

};
	</code>
</pre>

We've got an additional bit of functionality here, this is our `generateConfig` method which does all the leg work before we lift our sails application with `require(sails)();`

`require('./generateConfig')(command)` returns a Promise when we've built our `config/local.js` config file and created our dynamic `api/models/` - meaning we can then lift our application.

## Our --prod command

And the contents of our `tasks/startup/server.js` isn't too dissimilar, just a few extra flags thrown in for good measure.

<pre>
	<code class="javascript">
var path = require('path'),
    chalk = require('chalk'),
    pkg = require('../../package.json');

module.exports = function(command) {

    console.log(chalk.yellow('Launching'), pkg.name, chalk.yellow('Version'), pkg.version);
    process.argv.push('--prod');
    process.env.NODE_ENV = 'production';

    require('./generateConfig')(command).then(function() {

        console.log(chalk.green('prod'), 'Lifting environment');

        var sails = require('sails');
        sails.lift({
            hooks:{
                grunt:false
            }
        });

        console.log(chalk.green('prod'), 'Lifted environment');

    }, function(err){
        console.log(err);
    });

};
	</code>
</pre>

You can see here the only big real differences are that we make sure that we're running in a production environment, and also disable the grunt hook to decrease the time it takes for the application to spin up.

## generateConfig

I can't show you this method because of IP reasons my employer may have issues with, however I can certainly describe it to you.

We basically connect to a MySQL server to build a dynamic config object in our `config/local.js`. We mostly use `Model.query()` in our Controllers (`sails-mysql`).

So in our `config/local.js` we might have a database connection like so:
<pre>
	<code class="javascript">
"client_Resource": {
    "host": "127.0.0.1",
    "user": "user",
    "port": "3306",
    "password": "password",
    "multipleStatements": true,
    "adapter": "sails-mysql",
    "database": "reporting"
}
	</code>
</pre>

And we could create a file in our `api/models/` folder named `__client_Resource.js` and the contents would be:
<pre>
	<code class="javascript">
module.exports = {
    "connection": "client_Resource",
    "autoCreatedAt": false,
    "autoUpdatedAt": false,
    "autoPK": false,
    "migrate": "safe"
};
	</code>
</pre>

It doesn't matter if you have a million models with the same connection details, the `mysql` package and the way its Pool logic works will make sure that it won't make duplicate connections unless necessary.

NOTE: Don't forget to purge all models that start with `__` before you run `generateConfig` - because you're gonna rebuild them again anyway!

To choose the right Sails model and connection we used a service which lives in `api/services/Model.js`. Its not very complex and just abuses the fact Sails uses globals for the Models.
<pre>
	<code class="javascript">
exports.get = function(tenant, type) {
    return global['__'+tenant+'_'+type];
};
	</code>
</pre>

And then in our controllers we would `Model.get('client','Resource')` which would return the `mysql` object, so for example:
<pre>
	<code class="javascript">
Model.get('client','Resource').query('SELECT 1;', function(err,results) {
	if (err) return res.status(500).send(err);
	res.ok(results);
});
	</code>
</pre>


Hope that helps you dynamically build models and database connections for your sails application before it gets lifted!

Any questions drop us a line :)
