BrowserLog
============
Use your browser as a live log viewer with this tiny .net library

[![Build status](https://ci.appveyor.com/api/projects/status/github/alexvictoor/BrowserLog?svg=true)](https://ci.appveyor.com/project/alexvictoor/BrowserLog)

![Server logs in your browser console](https://raw.githubusercontent.com/alexvictoor/BrowserLog/master/screenshot.png)

BrowserLog is a log appender leveraging on "HTML5 Server Sent Event" (SSE) to push logs on browser consoles.
It relies on .NET 4.5 and support log4net (old and new key), NLog and Serilog. No other external dependencies required!


Usage
-----

Activation requires 3 steps:  

1. configuration of your build to add a dependency to this project
2. configuration of the appender ( or NLog target) in the logger configuration
3. inclusion of a javascript snippet in your HTML code to open a SSE connection

First thing first, you need to add a nuget dependency to BrowserLog.

If you are using a recent version of log4net you can add a nuget reference to BrowserLog to your project as follow:

    PM> Install-Package BrowserLog.log4net

If you are using an old version of log4net (version<=1.2.10) you need to use the 'old' log4net package:

    PM> Install-Package BrowserLog.Old.log4net

There are two different packages because log4net 'public token' has changed between version 1.2.10 and 2.03... Hence it might be very difficult to upgrade log4net on some legacy projects.

If you are using NLog you need to add a nuget reference to BrowserLog to your project as follow:

    PM> Install-Package BrowserLog.NLog

If you are using NLog you need to add a nuget reference to BrowserLog to your project as follow:

    PM> Install-Package BrowserLog.Serilog


The next step is to add BrowserLog in your XML logger configuration.  


Below an XML fragment example that shows how to configure log4net on the server side:
```xml
<log4net>
  ...
  <appender name="WEB" type="BrowserLog.BrowserConsoleAppender, BrowserLog.log4net">
    <Host>192.168.0.7</Host> <!-- Optional, the IP address on which the SSE server will be bound. If not specified try to detect the local IP of the host by itself -->
    <Port>8082</Port> <!-- Optional, this is the port on which the HTTP SSE server will listen. Default port is 8765 -->
    <Active>true</Active> <!-- Optional, if false the appender is disabled. Default value is true -->
    <Buffer>10</Buffer> <!-- Optional, the size of the buffer used to replay logs on connection. Default value is 1 -->
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%date [%thread] %-5level %logger - %message%newline" /> <!-- Use whatever pattern you want -->
    </layout>
  </appender>
  ...
```

If you are using NLog, below a similar example:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog>
...
  <extensions>
    <add assembly="BrowserLog.NLog"/>
  </extensions>
  <targets>
    <target
      name="WEB"
      type="BrowserConsole"
      Host="192.168.0.7"
      Active="true"
      Buffer="100"
      Port="8082"
      layout="${date} [${threadid}] ${uppercase:${level}} ${logger} ${ndc} - ${message}${newline}" />
  </targets>
  <rules>
    <logger name="*" minlevel="Debug" writeTo="WEB" />
  </rules>
  ...
</nlog>
```

Last but not least, with Serilog you need to add a few config keys in your App.config files as shown below:

```xml
<appSettings>
  <add key="serilog:using:BrowserConsole" value="BrowserLog.Serilog" />
  <add key="serilog:write-to:BrowserConsole.active" value="true" />
  <add key="serilog:write-to:BrowserConsole.buffer" value="100" />
  <add key="serilog:write-to:BrowserConsole.port" value="8082" />
  <add key="serilog:write-to:BrowserConsole.outputTemplate" value="{Timestamp:yyyy-MM-dd HH:mm:ss} [{Level}] {Message}{NewLine}{Exception}" />
  <add key="serilog:write-to:BrowserConsole.logProperties" value="true" />
</appSettings>
```

**Warning:** using default configuration, without specifying HOST property, **the server is not reachable on http://localhost** or 127.0.0.1 . You need to use the "windows host name" of your box, the first one returned by command "ipconfig /all"

In the browser side, the easiest way to get the logs is to include in your HTML document javascript file BrowserLog.js. This script is delivered by the embedded HTTP SSE server at URL path "/BrowserLog.js":

    <script src="http://HOST:PORT/BrowserLog.js"></script>

It gets even simpler when using a bookmarklet. To do so use your browser to display the "homepage" of the embedded HTTP SSE server (at URL http:// HOST : PORT where HOST & PORT are the parameters you have used in the log4net configuration). The main purpose of this "homepage" is to test your BrowserLog configuration but it also brings a ready to use bookmarklet (named "Get Logs!"). This bookmarklet looks like code fragment below:

    (function () {
        var jsCode = document.createElement('script');
        jsCode.setAttribute('src', 'http://HOST:PORT/BrowserLog.js');
        document.body.appendChild(jsCode);
    }());

*Note: the name of the js file does not really matter, you use URL http://HOST:PORT/foo.bar.js it will work as well*

Why SSE?
--------
[Server Sent Events](https://en.wikipedia.org/wiki/Server-sent_events) while being quite simple to implement, allows to stream data form a server to a browser leveraging on the HTTP protocol, just the HTTP protocol. Resources required on the server side are also very low. Simply put, to stream text log it makes no sens to use WebSocket which is much more complex.   
All newest browsers implement SSE, except IE... (no troll intended). Chrome even implements a nice reconnection strategy. Hence when you are using a Chrome console as a log viewer, log streams will survive to a restart of a server side process, reconnection are handled automatically by the browser.

Multiple server-side process?
-----------------------------
If you want to watch logs coming from several process, you just need one single browser console!  
There is no way to generated a big fat bookmarklet but you can write one by yourself quite easily. Let's say you have 2 processes:

- process1 logs can be retrieved using HOST1 & PORT1
- process2 logs can be retrieved using HOST2 & PORT2

You can use a bookmarlet that looks like code fragment below to retrieve logs from both processes:

    (function () {
        var jsCode = document.createElement('script');
        jsCode.setAttribute('src', 'http://HOST1:PORT1/process1.js');
        document.body.appendChild(jsCode);
    }());
    (function () {
        var jsCode = document.createElement('script');
        jsCode.setAttribute('src', 'http://HOST2:PORT2/process2.js');
        document.body.appendChild(jsCode);
    }());

Not that hard, right? One tiny detail is important to have in mind: in the above example, the two js resources have different names, process1 an process2.js, this is done on purpose. Otherwise, the js code that opens the SSE log stream does not work properly and opens only a connection to the server declared last.  

To distinguish logs coming from one service to logs coming from another one, you can use different log4net pattern, use a log prefix, or use custom styles (see next section).


Custom colors and styles?
-------------------------
Once you are connected to several log streams, you will might want to get different visual appearance for those streams.  
By default all streams use default browser log styles. Styles can be customized by adding special attributed to BrowserLog.js script tag:

    <script
        src="http://HOST:PORT/BrowserLog.js"
        style="font-size: 18px; background: cyan" >
    </script>

Styles attrbutes can also be specific to a logging category:

    <script
        src="http://HOST:PORT/BrowserLog.js"
        style="color: black;"
        style-error="color: red; font-size: 18px;" >
    </script>

With above example, all logs are written in black on white, size 12px, except error logs written in red, size 18px.  
Below a bookmarklet code that gives similar results:

    (function () {
        var jsCode = document.createElement('script');
        jsCode.setAttribute('src', 'http://HOST:PORT/BrowserLog.js');
        jsCode.setAttribute('style' 'color: black;');
        jsCode.setAttribute('style-error' 'color: red; font-size: 18px;');
        document.body.appendChild(jsCode);
    }());

Custom styles can be specify for debug (style-debug), info (style-info) and warn (style-warn) logs as well.

Disclaimer
---------
1. For obvious security concerns, **do not activate it in production!**  
2. This is a first basic not opimized implementation, no batching of log messages, no buffering, etc
