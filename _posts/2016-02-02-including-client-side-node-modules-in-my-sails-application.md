---
layout:     post
title:      Including npm client side modules in your sails application
date:       2016-02-02 13:30:30
summary:    I for one don't like using bower, so I shall show you how I feel is the best way to including packages from npm into your assets directory in sailsjs
categories: random
---

I for one don't like using Bower because 99.9% of the packages I need are available through npm - or you can just npm install straight from git anyway.

But because Sails file structure is the way it is, the `node_modules/` directory sits outside of the publically accessible `assets/` and `.tmp/public` directories. This could cause a headache if you're new to Sails and the easy way out is to use something like Bower. Don't do that. Don't copy them to `assets/` either if that directory is under version control in git/svn etc.

Instead, install your packages like normal to and then inside `tasks/config/copy.js` you can just add a new object to the array under the `dev` task, like so:

<pre>
	<code class="javascript">
module.exports = function(grunt) {

	grunt.config.set('copy', {
		dev: {
			files: [
            {
                expand:true,
                cwd: './node_modules/font-awesome/fonts',
                src: ['**/*'],
                dest: '.tmp/public/fonts'
            },
            {
                expand:true,
                cwd: './node_modules/bootstrap/fonts',
                src: ['**/*'],
                dest: '.tmp/public/fonts'
            },
			{
				expand:true,
				cwd:'./node_modules/angular/',
				src:['angular.js'],
				dest: '.tmp/public/js/libs'
			},
            {
                expand:true,
                cwd:'./node_modules/jquery/dist',
                src:['jquery.js'],
                dest:'.tmp/public/js/libs'
            },
            {
                expand:true,
                cwd:'./assets/js',
                src:['app.js'],
                dest:'.tmp/public/js'
            },
			{
				expand: true,
				cwd: './assets',
				src: ['**/*.!(coffee|less)'],
				dest: '.tmp/public'
			}]
		},
		build: {
			files: [{
				expand: true,
				cwd: '.tmp/public',
				src: ['**/*'],
				dest: 'www'
			}]
		}
	});

	grunt.loadNpmTasks('grunt-contrib-copy');
};
	</code>
</pre>


Restart your sails application and the `compileAssets` task will trigger the `dev:copy` task.

Don't forget to modify `tasks/pipeline.js` if you need your new packages linked into your templates in a specific order, here is mine as an exmaple:

<pre>
	<code class="javascript">
var jsFilesToInject = [

    // Load sails.io before everything else
    'js/libs/sails.io.js',

    'js/libs/jquery.js',
    'js/libs/qs.js',

    // angular is pretty important
    'js/libs/angular.js',
    'js/libs/ng-sails.js',
    'js/libs/angular-animate.js',

    // kendo ui includes, some are required before others
    'js/libs/kendo/kendo.core.js',
    'js/libs/kendo/cultures/kendo.culture.en-GB.js',
    'js/libs/kendo/kendo.angular.js',
    'js/libs/kendo/kendo.data.js',
    'js/libs/kendo/kendo.list.js',
    'js/libs/kendo/kendo.userevents.js',
    'js/libs/kendo/kendo.groupable.js',
    'js/libs/kendo/kendo.tabstrip.js',
    'js/libs/kendo/kendo.color.js',
    'js/libs/kendo/kendo.drawing.js',
    'js/libs/kendo/kendo.dataviz.core.js',
    'js/libs/kendo/kendo.dataviz.themes.js',
    'js/libs/kendo/kendo.dataviz.chart.js',
    'js/libs/kendo/kendo.dataviz.chart.polar.js',
    'js/libs/kendo/kendo.dataviz.chart.funnel.js',

    'js/libs/**/*.js',

    'js/app/app.js',
    'js/app/templates.js',
    'js/app/controller/**/*.js',
    'js/app/factory/**/*.js',
    'js/app/service/**/*.js',
    'js/app/directive/**/*.js',

    // All of the rest of your client-side js files
    // will be injected here in no particular order.
    'js/**/*.js'

];
	</code>
</pre>


Hope this helps :)

Ash
