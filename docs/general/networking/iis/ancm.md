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

## web.config - Basic

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
        <caching enabled="false" enableKernelCache="false" />
        <httpProtocol>
            <customHeaders>
                <clear />
                <add name="X-XSS-Protection" value="0" />
                <add name="X-Content-Type-Options" value="nosniff" />
                <add name="X-Robots-Tag" value="noindex, nofollow, noarchive" />
            </customHeaders>
        </httpProtocol>
        <applicationInitialization doAppInitAfterRestart="true"/>
    </system.webServer>
</configuration>
```

## web.config - Advanced
You may wish to utilize some security features, this configuration is a good start. It may break some things so be prepared to debug.

```config
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="C:\Jellyfin\Server\jellyfin.exe" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="InProcess">
          <environmentVariables>
            <environmentVariable name="JELLYFIN_ENABLE_IIS" value="true" />
          </environmentVariables>
    </aspNetCore>
    </system.webServer>
  </location>
    <system.webServer>
        <rewrite>
            <rules>
                <rule name="https" enabled="false" stopProcessing="true">
                    <match url=".*" />
                    <conditions>
                        <add input="{HTTPS}" pattern="^off$" />
                        <add input="{SERVER_PORT}" pattern="^8097$" negate="true" />
                        <add input="{REQUEST_URI}" pattern="^/.well-known" negate="true"/>
                    </conditions>
                    <action type="Redirect" url="https://{HTTP_HOST}{REQUEST_URI}" />
                </rule>
                <rule name="RequestBlockingRule1" stopProcessing="true">
                    <match url=".*" />
                    <conditions>
                        <add input="{URL}" pattern="^(/metrics|/api-docs)" />
                        <add input="{REMOTE_ADDR}" pattern="^(127.0.0.1|::1)$" negate="true" />
                    </conditions>
                    <action type="CustomResponse" statusCode="404" statusReason="File or directory not found." statusDescription="The resource you are looking for might have been removed, had its name changed, or is temporarily unavailable." />
                </rule>
            </rules>
          <outboundRules><!-- Add Cache -->
                <rule name="Add Cache" preCondition="static">
                    <match serverVariable="RESPONSE_Cache_Control" pattern="(.*)" />
                    <action type="Rewrite" value="public, max-age=2592000" />
                </rule>=
                <rule name="csp" enabled="true" preCondition="securable">
                    <match serverVariable="RESPONSE_Content_Security_Policy" pattern=".*" />
                    <action type="Rewrite" value="default-src 'none'; img-src 'self' https://i.ytimg.com https://image.tmdb.org https://m.media-amazon.com data:; script-src 'self' blob: https://www.youtube.com https://www.gstatic.com; media-src 'self' blob:; style-src 'self' 'unsafe-inline'; manifest-src 'self' ; font-src 'self'; connect-src 'self'; frame-src 'self' https://www.youtube.com; frame-ancestors 'none'; base-uri 'self'" />
                </rule>
                <rule name="permissionspolicy" enabled="true">
                    <match serverVariable="RESPONSE_Permissions_Policy" pattern=".*" />
                    <action type="Rewrite" value="geolocation=(), accelerometer=(), battery=(), ambient-light-sensor=(), clipboard-read=(), clipboard-write=(), display-capture=(), xr-spatial-tracking=(), interest-cohort=(), keyboard-map=(), local-fonts=(), magnetometer=(), sync-xhr=(), microphone=(), payment=(), publickey-credentials-get=(), document-domain=(), bluetooth=(), camera=(), gamepad=(self), usb=(self), encrypted-media=(self), autoplay=(self &quot;https://www.youtube.com&quot;), fullscreen=(self), cast=(self), picture-in-picture=(self &quot;https://www.youtube.com&quot;)" />
                </rule>
                <rule name="xframeoptions" enabled="true" preCondition="excludeua">
                    <match serverVariable="RESPONSE_X_Frame_Options" pattern=".*" />
                    <action type="Rewrite" value="SAMEORIGIN" />
                </rule><!-- CORP, COES, COOP, this breaks trailers and some external plugins/images/assets -->
                <rule name="corp" enabled="false" preCondition="successcode">
                    <match serverVariable="RESPONSE_Cross_Origin_Resource_Policy" pattern=".*" />
                    <action type="Rewrite" value="same-origin" />
                </rule>
                <rule name="coop" enabled="false" preCondition="successcode">
                    <match serverVariable="RESPONSE_Cross_Origin_Opener_Policy" pattern=".*" />
                    <action type="Rewrite" value="same-origin" />
                </rule>
                <rule name="coep" enabled="false" preCondition="successcode">
                    <match serverVariable="RESPONSE_Cross_Origin_Embedder_Policy" pattern=".*" />
                    <action type="Rewrite" value="require-corp" />
                </rule>
                <rule name="weboscorp" enabled="false" preCondition="webos">
                    <match serverVariable="RESPONSE_Cross_Origin_Resource_Policy" pattern=".*" />
                    <action type="Rewrite" value="cross-origin" />
                </rule>
            <preConditions>
                <preCondition name="static" logicalGrouping="MatchAny">
                    <add input="{RESPONSE_CONTENT_TYPE}" pattern="^image/" />
                    <add input="{RESPONSE_CONTENT_TYPE}" pattern="^application/javascript" />
                </preCondition>
                <preCondition name="securable"><!-- precondition to remove csp from some clients which I have had issues with -->
                    <add input="{HTTP_USER_AGENT}" pattern="playstation" negate="true" />
                    <add input="{RESPONSE_CONTENT_TYPE}" pattern="^text/html" />
                    <add input="{RESPONSE_STATUS}" pattern="^20" />
                </preCondition>
                <preCondition name="excludeua"><!--pre condition for excluding certain clients from X-Frame-Options jellyfin/jellyfin-webos/issues/63 -->
                    <add input="{HTTP_USER_AGENT}" pattern="webOS|playstation|Dalvik" negate="true" />
                    <add input="{RESPONSE_STATUS}" pattern="^20" />
                </preCondition>
                <preCondition name="webos">
                    <add input="{HTTP_USER_AGENT}" pattern="webOS|JellyfinMediaPlayer|Dalvik" />
                    <add input="{RESPONSE_STATUS}" pattern="^20" />
                </preCondition>
                <preCondition name="successcode">
                    <add input="{RESPONSE_STATUS}" pattern="^20" />
                </preCondition>
            </preConditions>
          </outboundRules>
        </rewrite>
        <httpProtocol>
            <customHeaders>
                <clear />
                <add name="X-XSS-Protection" value="0" />
                <add name="X-Content-Type-Options" value="nosniff" />
                <add name="Referrer-Policy" value="no-referrer" />
                <add name="X-Robots-Tag" value="noindex, nofollow, noarchive" />
            </customHeaders>
        </httpProtocol>
        <applicationInitialization doAppInitAfterRestart="true" />
    </system.webServer>
</configuration>
```

## SSL

[CertifytheWeb](https://certifytheweb.com/); a very easy to use UI for getting certificates
