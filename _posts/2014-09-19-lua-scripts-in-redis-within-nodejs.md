---
layout: post
author: <a href="http://ilyapimenov.com/">Ilya Pimenov</a>
title: Lua scripts in Redis within Node.js
comments: true
categories:
- blog
---
We don't need to tell you what Redis is. Most engineers built a strong love to it over the years for the simplicity and power it gives you. 

Although Lua support was introduced quite some time ago, the average internet tells us that it didn't really kick off. 

Here we share our experience with replacing part of the server side written in Node.JS with the Lua being executed in Redis,

## Why

So, we have this awesome product Bidder, that plays on the add market exchange. And in order to have all nodes notified of current situation, we use Redis database to share the current state, amount of money available to a particular operation among all the bidders running in the cloud.

This data structure is rather complex and dependent on each other. For example, consider it to be a tree of nodes, where each branch eats cookies 

Like so:

{% highlight bash %}
a [ ate: 0 ]
|--- ab [ ate: 0 ]
      |--- abe [ ate: 0 ]
      |--- abi [ ate: 0 ]
|--- ac [ ate: 0 ]
{% endhighlight %}

So, if the branch `abe` ate 2 cookies, `a` should have a balance of two eaten cookies — `a [ ate: 2 ]`.
And then if `abi` eats `3` and `ac` eats `1`, then it should go like:

{% highlight bash %}
a [ ate: 6 = 0 + 2 + 3 + 1 ]
|--- ab [ ate: 5 = 0 + 2 + 3 ]
      |--- abe [ ate: 2 = 0 + 2 ]
      |--- abi [ ate: 3 = 0 + 3 ]
|--- ac [ ate: 1 = 0 + 1 ]
{% endhighlight %}

So, previously we would have this big script in NodeJS that will batch all outgoing operations, and respectively update all nodes of the bank, this would be done from a single machine, called `Bank`. All other machines would connect to it through a `BankClient` — basically EventEmitter that talks to a `Bank` service. In this architecture you cannot have two Bank machines, as you would've run into concurrency issues that way. There are transactions in Redis, but when you have to combine them together with business logic, it gets rather messy as you cannot simply rollback.

But most importantly of all, this is ultimately not scalable and you have to maintain a lot of synchronization code in NodeJS to interact with your bank's persistence.

## Solution

An absolute win in this case is to incorporate all your logic into Redis. Think about it — you have one single threaded database that performs your business logic requests in an atomic way. And later on you can shard it, by the main key of each data structure being stored. It doesn't really get any easier than this.

Basically, it's not a database any longer, it is a persistence container, and you deploy your accounting business logic into it.

## How

Now, how do you do this with Redis? By utilizing the `EVAL` command — http://redis.io/commands/EVAL. It enables you to run Lua scripts inside your Redis instance.

Now, a lot of people might get sceptical once they hear Lua. Another language, another platform, all the burden of maintenance and testing issues.

What we aim to show here — is that all those are addressed in rather classical fashion, by unit-tests and integration tests, and you will find the approach of running Lua on top of Redis surprisingly pleasing.

## Getting all together

Let's put it all together. We will try to address task stated in Why section of this article: a tree-like structure that propagates it changes to all the parent nodes. 

First things first — you need to get Lua running locally on your machine in order to run Lua scripts as you are developing them.

I recommend you to install `luarocks`, a Lua-packet manager (what `npm` is to NodeJS, `cocoapods` are to Objective C,  `mvn` to Java, `leiningen` to Clojure). With `brew` on Mac: 

{% highlight bash %}
$ brew install luarocks
{% endhighlight %}

One thing to keep in mind, is that together with Lua runtime you will require a bunch of auxiliary libraries that are installed in Redis, in order to be able to write good, full blown scripts. 

A quick note on the data format we use in Redis. Since most of our data structures are rather complex and require a set parameters on different levels, we utilize JSON in our Redis values.

The only third party libraries, not native to Lua are `struct`, `CJSON` and `cmsgpack`, as can be seen in [EVAL spec](http://redis.io/commands/EVAL). With `luarocks` it is a no-brainer:

{% highlight bash %}
$ luarocks install lua-cjson
{% endhighlight %}

Now we are good to write our first Lua script!

### Update the node and all parent nodes

In our sample task we will update the node and all it parent nodes. Nodes are keys in Redis that are composed of its parent names and its own name all dot separated.

So, if you have a node `a.b.c` and you want to persist `{spent: 5}` with the key `a.b.c`,  the `spent` amount in both keys `a` and `a.b` should also go up.

The basic approach to it will be — 

{% highlight lua %}

if #ARGV ~= 1 then
    error('your script: there\'s something fishy here, me not like this')
end

if #KEYS ~= 1 then
    error('your script: invalid inputs, one key at a time')
end

local spent = tonumber(ARGV[1])

-- This node will give you parent node. Supposedly if there is a key 
-- nodeOne.nodeTwo.nodeTree, then nodeTwo is a direct parent of
--  nodeThree; and nodeOne is an indirect parent to nodeThree
local parent = function(account)
    -- searches from the end of the string instead
    -- of the beginning, as string.find does
    local i = account:match('^.*()%.')
    if i then
        return string.sub(account, 0, i - 1)
    end
    return nil
end

local spend = nil -- in order to reference to itself in a recursive way
spend = function(key, amount)
    local node = redis.call('GET', key)
    if not node then
        error('your script: key ' .. key .. ' doesnt exist')
    else
        -- read it from Redis and decode
        node = cjson.decode(node)
    end

    -- Nil check
    if not node.spent then
        node.spent = 0
    end

    -- recurse, first spend this amount on the parent nodes
    local parent = parent(key)
    if parent then
        local spentByParent = spend(parent, amount)
    end

    -- spend this amount on the child node
    node.spent = node.spent - amount

    local nodeString = cjson.encode(node)
    local embeddedNodeString = cjson.encode(nodeString) -- escape inner quotes, adds surrounding quotes
    -- parser friendly log
    redis.log(redis.LOG_NOTICE, 'spend.update: { "key":\"' .. key .. '\", "spent":' .. amount .. ', "value":' .. embeddedNodeString ..'}')

    redis.call('SET', key, nodeString)

    return spend
end

-- Since there are no arrays in Lua, only table, keep in mind that they are not 0-based, but 1
spend(KEYS[1], tonumber(ARGV[1]))
{% endhighlight %}

Save this file as `spend.lua` file and let's run this sweetness.

### Running the sweetness

We need to put in some initial values. Normally you would have a separate script to initialize them. For the purpose of this article this will do:

{% highlight bash %}
$ redis-cli set a "{\"spent\": 0}"
$ redis-cli set a.b "{\"spent\": 0}"
$ redis-cli set a.b.c "{\"spent\": 0}"
{% endhighlight %}

Sweet, now to run the script up-below on node `a.b.c` with a value of `7`:

{% highlight bash %}
$ redis-cli evalsha $(redis-cli script load "$(cat ./spend.lua)") 1 "a.b.c" 7
{% endhighlight %}

[Can you feel it](http://www.youtube.com/watch?v=bYPqZlhpbQo)? It just ran your Lua function in Redis. Note how we pass `1` to our script before `"a.b.c"`, it tells Redis how many arguments should be treated as keys and how many as a script arguments, those variables `KEYS` and `ARGV` respectively. 

### Logs and the situation

All logging from your scripts goes to the Redis log, therefore to see what is happening under the covers:

{% highlight bash %}
$ tail -F /usr/local/var/log/redis.log
{% endhighlight %}

Check your Redis config if not found — `redis-cli CONFIG GET logfile`.

### Transactions and exception handling

Now, for the main part our script is good. Yet, there are some issues to it.

Say, what if `a.b` is not initialized? Then, according to our implementation we will first increase `spent` amount on key `a`, but then we will fail with `your script: key a.b doesn't exist`. 

That will bring our system in an inconsistent state — not cool. Overall we want either all in or all out.

Now there is a fairly simple thing you can do in order introduce transaction like behaviour:

1. Utilize `MSET` as a single write operation. Since Redis is single threaded — you don't have to worry about all those `GET`s you have
1. Wrap the whole script with Lua's `pcall` — basically a `try ... catch` block

#### `MSET` for a single all in or all out

You'll need an utility function at the top of your script:

{% highlight lua %}
local _cache = {}
local redisMSET = function(_, key, value)
    table.insert(_cache, key)
    table.insert(_cache, value)
end
{% endhighlight %}

In code you would use these statements instead of all `SET`s:
{% highlight lua %}
redisMSET('SET', 'a.b', '{"spent": 12}')
{% endhighlight %}

And by the end of you script you would _commit_ it all at once with `MSET`:

{% highlight lua %}
redis.call('MSET', unpack(_cache))
{% endhighlight %}

You can also go smart about it and do a wrapper around `GET` that will first check your `_cache` if it holds the desired key and if not — perform a call with `redis.call('GET'...`.

#### Wrapping into `try` ... `catch` block

There is no magic with this one — you wrap the whole thing in `pcall` and you are done — it gives you `status` variable, that tells you if it went fine or not and the result of executing that block of code.

The code will go like this —

{% highlight lua %}
local main = function()
    
    -- Your code goes here and constructs some `result` object

    return result
end

local status, res = pcall(main)

if status then
    -- `main' went fine: return `result`
    return res
else
    -- `main' raised an error: take appropriate actions
    -- there is no redis.LOG_ERROR level
    redis.log(redis.LOG_WARNING, 'error: ' .. res)
    return redis.error_reply('error: ' .. res)
end
{% endhighlight %}

Combining it together with transactions:

{% highlight lua %}
local main = function()
    local _cache = {}
    local redisMSET = function(_, key, value)
        table.insert(_cache, key)
        table.insert(_cache, value)
    end

    -- your code goes here and constructs some `result` object

    redis.call('MSET', unpack(_cache))

    return result
end

local status, res = pcall(main)

if status then
    -- `main' went fine: return `result`
    return res
else
    -- `main' raised an error: take appropriate actions
    -- there is no redis.LOG_ERROR level
    redis.log(redis.LOG_WARNING, 'error: ' .. res)
    return redis.error_reply('error: ' .. res)
end
{% endhighlight %}

Voilà!

### Unit-Testing our sweet Lua awesomeness

Now all this is good, but here at Screen6 we do unit-testing, and just to be sure and because it is a good way to go about things.

There are two ways you can go about tests:

1. Test your Lua code with Lua unit tests
1. Test your Lua code on the application level. In our case — calling Lua script within Redis from CoffeeScript

### Unit Testing with Busted

We use [Busted](http://olivinelabs.com/busted/) framework to run unit tests on our Lua scripts. If you are familiar with [Mocha](http://visionmedia.github.io/mocha/), you will feel at home right away.

Testing is rather straightforward, there is only one thing to note, that you would prefer to run tests  "as if you are in Redis", and therefore you would require some wrapper code to mock the API Redis provides you within Lua.

A very basic implementation that will enable you to use these operations:

1. redis.call( 'SET', ... )
1. redis.call( 'MSET', ... )
1. redis.call( 'GET', ... )
1. redis.log( ... )

Will look like this:

{% highlight lua %}
cjson = require "cjson"

local DATABASE = {}

function switch(t)
    t.case = function(self, x)
        local f = self[x] or self.default
        if f then
            if type(f) == "function" then
                return f(x, self)
            else
                error("case "..tostring(x).." not a function")
            end
        end
    end
      return t
end

redis = {
    call = function(operation, ...)
        mockRedis = switch {
            ['SET'] = function (x)
                    if #arg ~= 2 then
                        error('invalid amount of argumetns passed on to mockRedis.set')
                    else
                        DATABASE[arg[1]] = cjson.decode(arg[2])
                    end
                end,
            ['MSET'] = function (x)
                    if #arg % 2 ~= 0 then
                        error('invalid amount or arguments (not even) passed on to the mockRedis.mset')
                    else
                        for i = 1, #arg, 2 do
                            DATABASE[arg[i]] = cjson.decode(arg[i + 1])
                        end
                    end
                end,
            ['GET'] = function (x)
                    if #arg ~= 1 then
                        error('invalid amount of arguments passed on to mockRedis.get')
                    else
                        if DATABASE[arg[1]] then
                            return cjson.encode(DATABASE[arg[1]])
                        else
                            return nil
                        end
                    end
                end,
            default = function (x)
                    error('unsupported method ' .. operation .. ' called on mockRedis')
                end,
        }
        return mockRedis:case(operation)
    end,

    LOG_DEBUG = "DEBUG",
    LOG_VERBOSE = "VERBOSE",
    LOG_NOTICE = "NOTICE",
    LOG_WARNING = "LOG_WARNING",

    log = function(level, message)
        --print(level .. "# " .. message)
    end
}
{% endhighlight %}

It might be just slightly overwhelming if you are not that much into Lua, although — it's pretty basic, once you get your head around it, it won't be a biggy to extend it further, to suit your needs.

### Integration test: Spawning Redis process from Node.JS to execute Lua in it

Now, for the simplicity sake of this article I will drop the part about doing integration test through a real instance of Redis.

In short we wrote a wrapper that launches Redis from Node.JS while you are running your Mocha tests in `before` and `after` blocks, something like this —

{% highlight coffeescript %}

RedisServer = (require './redis_spawn').RedisServer
redisServerOpts =
    workingDirectory: RedisServer::optionsDefault.workingDirectory
    save: true
    timeout: 2000
redisServer = new redisSpawn.RedisServer redisServerOpts

...

    before (done) ->
        redisServer.start () ->
            redisClient = redis.createClient redisServerOpts.config.port
            redisClient.once 'connect', () ->
                redisClient.flushall done

{% endhighlight %}

And then interacts with the Lua script that is running on the recently spawned Redis instance. Might there be any interest in it — we'll put up a separate article on the topic. Don't hesitate to reach out to us!

## In Production

### Installing Lua script upon application start

Now that you have your Lua scripts thoroughly tested it is time to run them in production. The basic idea is to load them once on application startup and keep the resulting `SHA`, so that you won't have to send the script there and back each and every time upon request.

Instead of thousand of words, the CoffeeScript snippet that will do just that — 

{% highlight coffeescript %}
redis = require 'redis'

module.exports.YourDbClient =
class RdbClient extends EventEmitter
    constructor: () ->
        @scripts = {
            someScript1:
                text: (fs.readFileSync (path.join __dirname, 'someScript1.lua')).toString()
            someScript2:
                text: (fs.readFileSync (path.join __dirname, 'someScript2.lua')).toString()
                sha1: null
        }
        script.name = name for name, script of @scripts      # copy name -> .name
        @connect()

    connect: (cb) ->
        if @rdb then return cb?()

        @rdb = redis.createClient <redisPort>, <redisHost>, <redisOpts>

        loadScripts = (cb) =>
            loader = @rdb.multi()
            loader.script 'load', @scripts.someScript1.text
            loader.script 'load', @scripts.someScript2.text
            loader.eval @scripts.configure.text, 0, (JSON.stringify { clients: @config.bidder.clients }), Date.now()
            loader.script 'load', @scripts.spend.text
            loader.exec (err, results) =>
                if err
                    cb? err
                    throw err
                @scripts.someScript1.sha1 = results[0]
                @scripts.someScript2.sha1 = results[1]
                cb?()

        @rdb.on 'connect', () =>
            loadScripts () => @emit 'connected'
        @rdb.on 'reconnect', () =>
            loadScripts () => @emit 'reconnected'
        @rdb.once 'end', () =>
            @rdb = null

    actionWithRedis: (args...) ->
        @txn ?= @rdb.multi()
        @operation = [@client.scripts.spend.sha1]
        @txn.evalsha.apply @txn, (@operation.concat args)
        @txn.exec (err, results) =>
            # further sweetness
            ...
{% endhighlight %}

### Puppet configuration for Jenkins

We utilize Jenkins with JUnit test result reports, Busted works with it just fine with XUnit reporting engine — 

{% highlight bash %}
$ busted -o junit ./test
{% endhighlight %}

And here are the small bits of puppet configuration to enable your Jenkins with Lua, Busted and the necessary libs:

{% highlight ruby %}
class genericpackages::lua {
    package { "luarocks":
        ensure => installed,
    }
    exec{ "install busted":
        command => "luarocks install busted",
        require => Package['luarocks'],
    }
    exec{ "install cjson redis library":
        command => "luarocks install lua-cjson",
        require => Package['luarocks'],
    }
}
{% endhighlight %}

## Summary

Four months into the game — works as a charm, easy to monitor. We are very happy with transition of our accounting logic into Lua scripts executed directly in the Redis.

## Links

1. [Lua Main Page](http://www.lua.org/)
1. [LuaRocks](http://github.com/keplerproject/luarocks)
1. [Installing Lua Rocks on Mac](http://luarocks.org/en/Installation_instructions_for_Mac_OS_X)
1. [Intro to Lua for Redis programmers](http://www.redisgreen.net/blog/intro-to-lua-for-redis-programmers/)
