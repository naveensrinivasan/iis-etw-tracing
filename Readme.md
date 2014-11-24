# IIS 8.5 ETW Logging

IIS 8.5 (Windows Server 2012 R2) supports writing W3SVC logs to ETW (Event Tracing for Windows) in addition (or as an alternative) to traditional text-based W3C log files.

You can read more about this feature [here](http://www.iis.net/learn/get-started/whats-new-in-iis-85/logging-to-etw-in-iis-85).

This offers a lot of interesting opportunities for processing log events in a more compact format, and also for processing log entries in near real-time!

This sample contains a simple TraceEventParser that can be used with the [Microsoft TraceEvent library](https://www.nuget.org/packages/Microsoft.Diagnostics.Tracing.TraceEvent) to process ETW events generated by the Microsoft-Windows-IIS-Logging provider in IIS 8.5.

## Using the Code
Included is also some sample code on how to use it. Here's how to create a real-time ETW session and log events from IIS as they come:


```C#
class Program {
  const String SessionName = "iis-etw";
  static void Main(string[] args) {
    // create a new real-time ETW trace session
    using ( var session = new TraceEventSession(SessionName) ) {
      // enable IIS ETW provider and set up a new trace source on it
      session.EnableProvider(IISLogTraceEventParser.ProviderName, TraceEventLevel.Verbose);

      using ( var traceSource = new ETWTraceEventSource(SessionName, TraceEventSourceType.Session) ) {
        Console.WriteLine("Session started, listening for events...");
        var parser = new IISLogTraceEventParser(traceSource);
        parser.IISLog += OnIISRequest;

        traceSource.Process();
        Console.ReadLine();
        traceSource.StopProcessing();
      }
    }
  }
  private static void OnIISRequest(IISLogTraceData request) {
    Console.WriteLine("{0} {1}", request.Cs_method, request.Cs_uri_stem);
  }
}
```

## Pushing events to Azure
The second part of this sample is a more interesting ETW event collector. This currently is a simple console application (IISLogCollector), but will later turn into a proper Windows NT service.

This collector will capture ETW events in near-real time and can push said events to an [Azure Event Hub](http://azure.microsoft.com/en-us/services/event-hubs/). To use this feature, you'll need to first configure a few things:

* Create a new Event Hub in the azure portal. You should also add a new Shared Access Policy (SAS) for the collector, which only requires "Send" permissions.
* Create an `appSettings.config` file to the IISLogCollector project and ensure it is copied to the output folder during the build.
* Edit the `appSettings.config` file to configure your Event Hub connection information:

```XML
<!-- EventHub connection string -->
<add key="EtwHubConnectionString"
     value="<event hub connection string>" />
<!-- EventHub Name -->
<add key="EtwEventHubName" value="<event hub name>"/>
<!-- 
   Maximum size in KB of the batch for sending data 
   to Event Hub (should be less than 256KB by default)
-->
<add key="MaxBatchSizeKB" value="<event hub name>"/>
<!--
  Number of concurrent EventHub client instances to use
  to send event batches. If not present, one sender
  per CPU will be used.
-->
<add key="EventHubSenderCount" value="4"/>
```

**Note:** This is just a sample, lacking most error handling and won't be able to handle a large number of events without some work. It is, however, useful for playing with Azure services :).

## About the code
The code for the TraceEventParser is hand-written. This was my first attempt at writing such a thing and I wanted to better understand how it all worked under the hood. I'm sure I missed a few things here and there.

One of the interesting aspects of it, however, is that ETW events generated by the Microsoft-Windows-IIS-Logging provider include some string fields in unicode and others in ANSI.

Note, however, that you wouldn't normally do it this way. Instead, the TraceParserGen tool can be used to generate this code directly from the ETW provider manifest. A pointer to the tool can be found [here](http://blogs.msdn.com/b/vancem/archive/2013/08/15/traceevent-etw-library-published-as-a-nuget-package.aspx#10473283), and the help for the tool is pretty self-explanatory. Note, however, that the generated code for the IIS provider is pretty ugly :).
