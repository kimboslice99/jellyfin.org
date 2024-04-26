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

## web.config - Basic

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
        </rewrite>
        <caching enabled="false" enableKernelCache="false" />
        <httpProtocol>
            <customHeaders>
                <clear />
                <add name="X-XSS-Protection" value="0" />
                <add name="X-Content-Type-Options" value="nosniff" />
                <add name="Referrer-Policy" value="no-referrer" />
                <add name="X-Robots-Tag" value="noindex, nofollow, noarchive" />
            </customHeaders>
        </httpProtocol>
    </system.webServer>
</configuration>
```

## web.config - Advanced
You may wish to utilize some security features, this configuration is a good start. It may break some things so be prepared to debug.

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
                <rule name="xframeoptions" enabled="true" preCondition="excludeua">
                    <match serverVariable="RESPONSE_X_Frame_Options" pattern=".*" />
                    <action type="Rewrite" value="SAMEORIGIN" />
                </rule
                <rule name="csp" enabled="true" preCondition="securable">
                    <match serverVariable="RESPONSE_Content_Security_Policy" pattern=".*" />
                    <action type="Rewrite" value="default-src 'none'; img-src 'self' https://i.ytimg.com https://image.tmdb.org https://m.media-amazon.com data:; script-src 'self' blob: https://www.youtube.com https://www.gstatic.com; media-src 'self' blob:; style-src 'self' 'unsafe-inline'; manifest-src 'self' ; font-src 'self'; connect-src 'self'; frame-src 'self' https://www.youtube.com; frame-ancestors 'none'; base-uri 'self'" />
                </rule>
                <rule name="permissionspolicy" enabled="true">
                    <match serverVariable="RESPONSE_Permissions_Policy" pattern=".*" />
                    <action type="Rewrite" value="serial=(), geolocation=(), accelerometer=(), battery=(), ambient-light-sensor=(), clipboard-read=(), clipboard-write=(), display-capture=(), xr-spatial-tracking=(), interest-cohort=(), keyboard-map=(), local-fonts=(), magnetometer=(), sync-xhr=(), microphone=(), payment=(), publickey-credentials-get=(), document-domain=(), bluetooth=(), camera=(), gamepad=(self), usb=(self), encrypted-media=(), autoplay=(self &quot;https://www.youtube.com&quot;), fullscreen=(self), cast=(self), picture-in-picture=(self &quot;https://www.youtube.com&quot;)" />
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
                <preConditions><!-- Pre-Condition for images -->
                    <preCondition name="images" logicalGrouping="MatchAny">
                        <add input="{REQUEST_URI}" pattern="Items/.+/Images/.*" />
                        <add input="{RESPONSE_CONTENT_TYPE}" pattern="^image/.+" />
                    </preCondition>
                    <preCondition name="excludeua"><!--exclude specific clients from X-Frame-Options jellyfin/jellyfin-webos/issues/63 -->
                        <add input="{HTTP_USER_AGENT}" pattern="webOS|playstation|Dalvik" negate="true" />
                        <add input="{RESPONSE_STATUS}" pattern="^20" />
                    </preCondition>
                    <preCondition name="securable"><!-- precondition so CSP is only added to clients which are not effected -->
                        <add input="{HTTP_USER_AGENT}" pattern="playstation" negate="true" />
                        <add input="{RESPONSE_CONTENT_TYPE}" pattern="^text/html" />
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
        <caching enabled="false" enableKernelCache="false" />
        <httpProtocol>
            <customHeaders>
                <clear />
                <add name="X-XSS-Protection" value="0" />
                <add name="X-Content-Type-Options" value="nosniff" />
                <add name="Referrer-Policy" value="no-referrer" />
                <add name="X-Robots-Tag" value="noindex, nofollow, noarchive" />
            </customHeaders>
        </httpProtocol>
    </system.webServer>
</configuration>
```

## SSL

[CertifytheWeb](https://certifytheweb.com/); a very easy to use UI for getting certificates
