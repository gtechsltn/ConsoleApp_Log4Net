# ConsoleApp and Log4Net Sample
* Handle exeption that log4net write to file that no more free space
* What ```<maxSizeRollBackups value="-1" />``` this mean?
* What ```<maxSizeRollBackups value="5" />``` this mean?

# packages.config
```
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <package id="Dapper" version="2.0.123" targetFramework="net48" />
  <package id="Dapper.Contrib" version="2.0.78" targetFramework="net48" />
  <package id="Dapper.FastCrud" version="3.1.46" targetFramework="net48" />
  <package id="log4net" version="3.0.4" targetFramework="net48" />
  <package id="Newtonsoft.Json" version="13.0.3" targetFramework="net48" />
</packages>
```

# Web.config or App.config
```
  <configSections>
    <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net" />
  </configSections>

  <system.diagnostics>
    <trace autoflush="true">
      <listeners>
        <add name="textWriterTraceListener" type="System.Diagnostics.TextWriterTraceListener" initializeData="logs\log4net.trace.txt" />
      </listeners>
    </trace>
  </system.diagnostics>
  <log4net>
    <appender name="ConsoleAppender" type="log4net.Appender.ConsoleAppender">
      <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%date [%thread] %-5level %logger [%ndc] &lt;%property{auth}&gt; - %message%newline" />
      </layout>
    </appender>
    <appender name="DailyRollingFileAppender" type="log4net.Appender.RollingFileAppender">
      <threshold value="ALL" />
      <file value="logs\traceroll.day.log" />
      <appendToFile value="true" />
      <rollingStyle value="Composite" />
      <datePattern value="yyyyMMdd" />
      <maximumFileSize value="10MB" />
      <maxSizeRollBackups value="-1" />
      <CountDirection value="1" />
      <preserveLogFileNameExtension value="true" />
      <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="[${COMPUTERNAME}] %d{ISO8601} %6r %-5p [%t] %c{2}.%M() - %m%n" />
      </layout>
    </appender>
    <root>
      <!-- ALL, DEBUG, INFO, WARN, ERROR, FATAL, OFF -->
      <level value="ALL" />
      <appender-ref ref="ConsoleAppender" />
      <appender-ref ref="DailyRollingFileAppender" />
    </root>
  </log4net>
```

# Program.cs
```
namespace ConsoleApp1
{
    public class Program
    {
        private static readonly ILog log = LogManager.GetLogger(MethodBase.GetCurrentMethod().DeclaringType);

        private static readonly string connectionString = ConfigurationManager.ConnectionStrings["DBConnectionString"].ConnectionString;

        public static void Main(string[] args)
        {
            //====================================================================================================
            log4net.Config.XmlConfigurator.Configure();
            //====================================================================================================
            log.Info("Hello World");
        }
    }
}
```

# A. Handle exeption that log4net write to file that no more free space

If log4net is writing logs to a file and runs into a "No more free space" error, you can handle this exception gracefully by configuring log4net to detect such errors and take appropriate action. Here's how you can do it:

## 1. Use an Error Handler in log4net
You can set up a log4net.Util.LogLog error handler to capture internal logging errors:
```
using log4net;
using log4net.Core;
using log4net.Util;
using System;
using System.IO;

public class Log4NetErrorHandler : IErrorHandler
{
    public void Error(string message, Exception e, ErrorCode errorCode)
    {
        if (e is IOException && e.Message.Contains("There is not enough space on the disk"))
        {
            Console.WriteLine("Logging error: Disk is full! Taking appropriate action.");
            // Handle this scenario, e.g., notify admin, clear logs, or switch storage
        }
        else
        {
            Console.WriteLine($"Logging error: {message}, Exception: {e?.Message}");
        }
    }

    public void Error(string message)
    {
        Console.WriteLine($"Logging error: {message}");
    }

    public void Error(string message, Exception e)
    {
        Console.WriteLine($"Logging error: {message}, Exception: {e?.Message}");
    }
}
```

Then, configure your log4net appender to use this custom error handler:
```
var fileAppender = new log4net.Appender.RollingFileAppender
{
    File = "logs/app.log",
    AppendToFile = true,
    RollingStyle = log4net.Appender.RollingFileAppender.RollingMode.Size,
    MaxSizeRollBackups = 5,
    MaximumFileSize = "10MB",
    Layout = new log4net.Layout.PatternLayout("%date [%thread] %-5level %logger - %message%newline"),
    LockingModel = new log4net.Appender.FileAppender.MinimalLock()
};

fileAppender.ErrorHandler = new Log4NetErrorHandler();
fileAppender.ActivateOptions();
log4net.Config.BasicConfigurator.Configure(fileAppender);
```

## 2. Catch Exceptions in Your Code
If you're manually logging messages, wrap log statements in a try-catch block:
```
try
{
    log.Info("This is a log message.");
}
catch (IOException ex) when (ex.Message.Contains("There is not enough space on the disk"))
{
    Console.WriteLine("Log file write failed due to low disk space. Consider clearing logs.");
}
```

## 3. Automatically Delete Old Logs
To prevent this issue, configure RollingFileAppender to delete old logs:
```
<appender name="RollingLogFileAppender" type="log4net.Appender.RollingFileAppender">
    <file value="logs/app.log" />
    <appendToFile value="true" />
    <rollingStyle value="Size" />
    <maxSizeRollBackups value="5" />
    <maximumFileSize value="10MB" />
    <staticLogFileName value="true" />
    <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%date [%thread] %-5level %logger - %message%newline" />
    </layout>
</appender>
```

With this setup, log4net will automatically delete older logs when new ones are created.

## 4. Monitor Disk Space
Periodically check disk space and disable logging if necessary:
```
public static bool IsDiskSpaceAvailable(string driveLetter, long requiredSpaceInBytes)
{
    DriveInfo drive = new DriveInfo(driveLetter);
    return drive.AvailableFreeSpace > requiredSpaceInBytes;
}

if (!IsDiskSpaceAvailable("C", 50 * 1024 * 1024)) // 50MB threshold
{
    Console.WriteLine("Low disk space detected! Stopping log writes.");
    log4net.LogManager.Shutdown();
}
```

This ensures that logging stops before the disk is completely full.

With these strategies, you can handle low disk space errors effectively while ensuring log4net behaves gracefully when encountering such issues. 🚀

# What <maxSizeRollBackups value="-1" /> this mean?

The <maxSizeRollBackups value="-1" /> setting in log4net's RollingFileAppender means "unlimited backup log files."

**Explanation**

maxSizeRollBackups controls how many old log files are kept when rolling occurs due to size or date.

When set to -1, log4net will not delete any old log files, allowing an unlimited number of backups.

This means that as logs grow, old logs will keep accumulating indefinitely, which may lead to disk space issues if not monitored.

Example:
```
<appender name="RollingFileAppender" type="log4net.Appender.RollingFileAppender">
    <file value="logs/app.log" />
    <appendToFile value="true" />
    <rollingStyle value="Size" />
    <maxSizeRollBackups value="-1" />
    <maximumFileSize value="10MB" />
    <staticLogFileName value="true" />
    <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%date [%thread] %-5level %logger - %message%newline" />
    </layout>
</appender>
```

**Potential Issues**

Over time, this can fill up the disk if logs grow continuously.

If you don't want unlimited logs, set a specific number, such as 5 (keeps the last 5 backups).

Better Alternative

To prevent disk space issues, limit the number of backups:

```
<maxSizeRollBackups value="5" />
```

This keeps only the last 5 log files, deleting older ones automatically.

Would you like help setting up a log cleanup strategy? 🚀

# References
* https://github.com/gtechsltn/NET9_SecureWebApp
* https://github.com/gtechsltn/NET48_WinSvc_DataExport
* https://github.com/gtechsltn/SqlCommandTimeout
* https://github.com/gtechsltn/ConsoleApp_Log4Net
* https://github.com/gtechsltn/SqlServerInsertFiles
* https://github.com/gtechsltn/ConsoleApp_SharpZipLib
