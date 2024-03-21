---
uid: network-reverse-proxy-iis
title: IIS Reverse Proxy (ARR)
---

## Requirements

IIS with default selections plus Application Development->WebSocket Protocol

[URL Rewrite 2.1](https://www.iis.net/downloads/microsoft/url-rewrite)

[Application Request Routing 3.0](https://www.iis.net/downloads/microsoft/application-request-routing)

## Configure

- These commands must be run from an elevated powershell prompt, this will perform the following tasks.
    - Enable the proxy.
    - Disable proxy caching.
    - Preserve host header.
    - Add the nessessary allowedServerVariables.

```powershell
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.webServer/proxy" -name "enabled" -value "True"
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.webServer/proxy/cache" -name "enabled" -value "False"
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.webServer/proxy" -name "preserveHostHeader" -value "True"
Add-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.webServer/rewrite/allowedServerVariables" -name "." -value @{name='HTTP_X_FORWARDED_PROTOCOL'}
Add-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.webServer/rewrite/allowedServerVariables" -name "." -value @{name='HTTP_X_FORWARDED_PROTO'}
Add-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.webServer/rewrite/allowedServerVariables" -name "." -value @{name='HTTP_X_REAL_IP'}
Add-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.webServer/rewrite/allowedServerVariables" -name "." -value @{name='HTTP_X_FORWARDED_HOST'}
Add-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.webServer/rewrite/allowedServerVariables" -name "." -value @{name='HTTP_X_FORWARDED_PORT'}
```

long running streams may freeze due to limits within IIS, the following (powershell) command resolves it.

```powershell
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST'  -filter "system.applicationHost/webLimits" -name "minBytesPerSecond" -value 25
```

## web.config

```config
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <clear />
                <rule name="Redirect to https" enabled="false" stopProcessing="true">
                    <match url=".*" negate="false" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
                        <add input="{HTTPS}" pattern="off" />
                        <add input="{REQUEST_URI}" pattern="^/.well-known" negate="true" />
                    </conditions>
                    <action type="Redirect" url="https://{HTTP_HOST}{REQUEST_URI}" redirectType="Found" />
                </rule>
                <!-- These rules add X-Forwarded-Protocol -->
                <rule name="ForwardedHttps">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
                        <add input="{HTTPS}" pattern="On" />
                    </conditions>
                    <serverVariables>
                        <set name="HTTP_X_FORWARDED_PROTOCOL" value="https" />
                        <set name="HTTP_X_FORWARDED_PROTO" value="https" />
                    </serverVariables>
                </rule>
                <rule name="ForwardedHttp">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
                        <add input="{HTTPS}" pattern="Off" />
                    </conditions>
                    <serverVariables>
                        <set name="HTTP_X_FORWARDED_PROTOCOL" value="http" />
                        <set name="HTTP_X_FORWARDED_PROTO" value="http" />
                    </serverVariables>
                </rule><!-- proxy to Jellyfin -->
                <rule name="Proxy">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll" trackAllCaptures="false">
                        <add input="{R:0}" pattern="^\.well-known" negate="true" />
                    </conditions>
                    <serverVariables>
                        <set name="HTTP_X_REAL_IP" value="{REMOTE_ADDR}" />
                        <set name="HTTP_X_FORWARDED_HOST" value="{HTTP_HOST}" />
                        <set name="HTTP_X_FORWARDED_PORT" value="{SERVER_PORT}" />
                    </serverVariables>
                    <action type="Rewrite" url="http://localhost:8096/{R:0}" logRewrittenUrl="true" />
                </rule>
            </rules>
            <outboundRules><!-- Add Cache -->
                <rule name="Add Cache" preCondition="images" enabled="true" patternSyntax="ECMAScript">
                    <match serverVariable="RESPONSE_Cache_Control" pattern="(.*)" />
                    <action type="Rewrite" value="public, max-age=604800" />
                </rule>
                <preConditions><!-- Pre-Condition for images -->
                    <preCondition name="images" logicalGrouping="MatchAny">
                        <add input="{REQUEST_URI}" pattern="Items/.+/Images/.*" />
                        <add input="{RESPONSE_CONTENT_TYPE}" pattern="^image/.+" />
                    </preCondition>
                </preConditions>
            </outboundRules>
        </rewrite>
        <caching enabled="false" enableKernelCache="false" />
        <httpProtocol>
            <customHeaders>
                <clear />
                <add name="X-XSS-Protection" value="0" />
                <add name="X-Content-Type-Options" value="nosniff" />
                <add name="Cache-Control" value="no-cache" />
                <add name="X-Frame-Options" value="SAMEORIGIN" />
                <add name="X-Robots-Tag" value="noindex, nofollow" />
            </customHeaders>
        </httpProtocol>
    </system.webServer>
</configuration>

```

## SSL

[CertifytheWeb](https://certifytheweb.com/); a very easy to use UI for getting certificates
