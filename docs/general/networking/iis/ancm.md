---
uid: network-ancm-iis
title: IIS (ANCM)
---

:::caution
Jellyfin on IIS is an unsupported feature.
:::

## Requirements

IIS with default selections, plus
    - Application Development->WebSocket Protocol
    - Application Development->Application Initialization

[URL Rewrite 2.1](https://www.iis.net/downloads/microsoft/url-rewrite)

[ANCM Hosting bundle](https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-aspnetcore-8.0.2-windows-hosting-bundle-installer)

## Configure

- App pool settings (ApplicationPools->[apppoolname]->Advanced Settings)
    - Under General set Start Mode to AlwaysRunning.
    - Under Process Model set load user profile to true, and idle timeout to 0.
    - Under Recycling set the Regular Time Interval to 0.
- [site_name]->Advanced Settings->Preload enabled set to true.
- long running streams may freeze due to limits within IIS, the following (powershell) command resolves it.
```powershell
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.applicationHost/webLimits" -name "minBytesPerSecond" -value 25
```

## web.config

```config
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <location path="." inheritInChildApplications="false">
        <system.webServer>
            <handlers>
                <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified"/>
            </handlers>
            <aspNetCore processPath="\path\to\jellyfin.exe" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="InProcess">
                <environmentVariables>
                    <environmentVariable name="JELLYFIN_ENABLE_IIS" value="true"/>
                </environmentVariables>
            </aspNetCore>
        </system.webServer>
    </location>
    <system.webServer>
        <rewrite>
            <rules><!-- for local clients that do not need https we can assign a port in IIS (8096) and exclude connections to this port -->
                <rule name="https" enabled="false" stopProcessing="true">
                    <match url=".*"/>
                    <conditions logicalGrouping="MatchAll">
                        <add input="{HTTPS}" pattern="^off$"/>
                        <add input="{SERVER_PORT}" pattern="^8096$" negate="true"/>
                        <add input="{REQUEST_URI}" pattern="^/.well-known" negate="true"/>
                    </conditions>
                    <action type="Redirect" url="https://{HTTP_HOST}{REQUEST_URI}"/>
                </rule>
            </rules>
        </rewrite>
        <applicationInitialization doAppInitAfterRestart="true"/>
    </system.webServer>
</configuration>
```

## SSL

[CertifytheWeb](https://certifytheweb.com/); a very easy to use UI for getting certificates
