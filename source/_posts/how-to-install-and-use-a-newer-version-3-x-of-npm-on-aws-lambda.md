---
title:  How to install and use a newer version (3.x) of NPM on AWS Lambda.
date: 2016-07-19
---
My current experiment is to build a serverless deploy pipeline (With AWS CodePipeline) which uses AWS Lambda for the build steps. One step includes to invoke NPM to build a static website out of JavaScript components (which would be deployed to an S3 bucket in a later step).

Ok, so let's go ahead and look what is actually installed in the preconfigured Node 4.3 env on AWS Lambda. First we want to find out if NPM is actually already installed. So we just create a new Lambda function which invokes a \`find' command, here is the used source code: 

    
    exports.handler = (event, context, callback) => {  
      var child_process = require('child_process'); 
      console.log(child_process.execSync('find /usr -name npm -type f', {encoding: 'utf-8'}));   
    }; 
    

And, voila, we found something, here is the output: 

    
    /usr/local/lib64/node-v4.3.x/lib/node_modules/npm/bin/npm
    

So let's try to execute it! 

    
    console.log(child_process.execSync('/usr/local/lib64/node-v4.3.x/lib/node_modules/npm/bin/npm version', {encoding: 'utf-8'}));
    

And here is the output:

    
    module.js:327
        throw err;
        ^
    
    Error: Cannot find module '/usr/local/lib64/node-v4.3.x/lib/node_modules/npm/bin/node_modules/npm/bin/npm-cli.js'
        at Function.Module._resolveFilename (module.js:325:15)
        at Function.Module._load (module.js:276:25)
        at Function.Module.runMain (module.js:441:10)
        at startup (node.js:134:18)
        at node.js:962:3
    
        at checkExecSyncError (child_process.js:464:13)
        at Object.execSync (child_process.js:504:13)
        at exports.handler (/var/task/index.js:4:29)
    

Ok, that doesn't look good, does it? Actually the 'node\_modules/npm/bin/node\_modules/npm/bin/npm-cli.js' part looks broken.

Ok, so my next step was to find the correct path to npm-cli.js, so I have a chance to call it without the apparently broken executable wrapper: 

    
    console.log(child_process.execSync('find /usr -type f -name npm-cli.js', {encoding: 'utf-8'}));
    
    
    /usr/local/lib64/node-v4.3.x/lib/node_modules/npm/bin/npm-cli.js
    

So let's try to call it directly:

    
    console.log(child_process.execSync('node /usr/local/lib64/node-v4.3.x/lib/node_modules/npm/bin/npm-cli.js version', {encoding: 'utf-8'}));
    

gives us:

    
    { npm: '2.14.12',  ... }
    

**Yay! We got NPM working!**

**But NAY, it's an old version! **

So let's go ahead and try to install a newer version! Lambda gives us a writable `/tmp`, so we could use that as a target dir. NPM actually wants to do much stuff in the `$HOME` directory (e.g. trying to create cache dirs), but it is not writable within a Lambda env.

So my "hack" was to set the `$HOME` to `/tmp`, and then install a newer version of NPM into it (by using the `--prefix option`):

    
    process.env.HOME = '/tmp';
    console.log(child_process.execSync('node /usr/local/lib64/node-v4.3.x/lib/node_modules/npm/bin/npm-cli.js install npm --prefix=/tmp --progress=false', {encoding: 'utf-8'}));
    console.log(child_process.execSync('node /tmp/node_modules/npm/bin/npm-cli.js version', {encoding: 'utf-8'}));
    

Ok, NPM got installed and is ready to use!

    
    npm@3.10.5 ../../tmp/node_modules/npm
    

The last step is to symlink the npm wrapper so it can be used without hassle. And actually many build systems seem to expect a working npm executable:

    
    fs.mkdirSync('/tmp/bin');
    fs.symlinkSync('/tmp/node_modules/npm/bin/npm-cli.js', '/tmp/bin/npm');
    process.env.PATH = '/tmp/bin:' + process.env.PATH;
    console.log(child_process.execSync('npm version', {encoding: 'utf-8'}));
    

And here we go!  Now it's possible to use a up-to-date version of NPM within a Lambda function. 

Some additional learnings:

* NPM needs a lot of memory, so I configured the Lambda function with max memory of 1500MB RAM. Otherwise it seems to misbehave or garbage collect a lot.
* You should start with a clean tmp before installing NPM in order to avoid side effects, as containers might get reused by Lambda. That step did it for me:

    
    child_process.execSync('rm -fr /tmp/.npm');  
    // ... npm install steps ...
    

* Downloading and installing NPM every time the build step is executed makes it more flaky (remember the fallacies of networking!). It also reduces the available execution time by 10 seconds (the time it takes to download and install NPM). One could pack the installed npm version as an own Lambda function in order to decouple it. But that's a topic for another blog post. 