# ConsoleApp and Log4Net Sample

* Handle exeption that log4net write to file that no more free space

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
