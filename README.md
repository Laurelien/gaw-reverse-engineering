# Galaxy At War reverse engineering

## 1. Introduction

Galaxy at War is a real time sci-fi battle strategy online game on mobile where players have to manage planets and fleets in order to rule the galaxy.

I have been playing this game for a decade, and I was always wondering how does the player gets informations from the server, and what information does the server actually give to the player.

## 2. Environment and tools

I was playing on an iPhone 16, and could not have access to the IPA file. I had to use the Android version of the game, the APK, to go further into my understanding.

During this whole project, I used different tools in order to achieve my goals :

- I used Burp Suite to intercept traffic between my iPhone and the server, along with the [NoPE](https://github.com/summitt/Nope-Proxy) extension
- I used [Jadx](https://github.com/skylot/jadx) to decompile the game from its Android APK form to readable Java files
- I used [Unluac](https://github.com/HansWessels/unluac) to decompile Lua bytecode files found in the decompiled APK

## 3. My step by step methodology

### A) Traffic interception

In order to intercept traffic between my phone and the game server, I had to configure Burp Suite proxy.
I made the proxy listening on all interfaces and on a defined port. On my phone, I added a proxy on my network parameters at my computer local address at the chosen port. If this basic configuration works for web browsers such as Safari on my iPhone, it didn't work for the application that I wanted to analyze. Galaxy At War was bypassing the proxy, and was requesting the server IP address directly through my DNS, opening a direct TCP connection with the server.

So I also changed my network DNS configuration to use my computer as a DNS server. NoPE acting as a DNS server, routing all DNS request to Burp, tricked the application believing that it was in direct communication with the game server.

Last but not least, I had to generate a certificate in Burp and to install it on my phone. It ensures that my phone trust the connection to the game server (with the wrong certificate) and can negotiate TLS connections. Burp and NoPE are acting as a MitM (Man in the Middle), a transparent proxy chains with their own TLS connection.

Starting the game, I immediately gathered traffic produced and noticed that the game was actually communicating with the server through HTTPs with PHP 5.3.8 and 5.6.4. PHP 5.3.8 is used in the main menu, where the player is automatically connected. PHP 5.6.4 is then used in the game for the actual gameplay.

In the beginning, the client is sending two important values : `account_key` and `client_key` . Later on, when player is logged in, the client is also sending a `client_secret` to the server.

 Most of the server responses were in plain text in JSON, except a few API endpoints who were sending responses in base64. I tried to decode the base64, but could not get anything readable, so I understood that it was encrypted. Here are the 4 API endpoints that were responding encrypted base64 payload :

```
/ING004/n/WebServer/Web/sogame/newControl/nmUniverse/getUniverse
/ING004/n/WebServer/Web/sogame/newControl/nmFriendEx/getFrientList
/ING004/n/WebServer/Web/sogame/newControl/nmFleet/getRadarFleets
/ING004/n/WebServer/Web/sogame/newControl/nmAlliance/getMemberList
```

I decided to push my investigation further to understand why some views were encrypted and if they were hiding more informations that the mobile application was not displaying.

### B) Decompilation and encryption

There are numerous ways to encrypt data, so I had to find which algorithm the game was actually using.
I decided to download the Android version, the APK file, and decompile it with the command line tool **Jadx**. The idea behind it was to find in the code anything related to the encryption and the algorithm used.

The decompiled APK was separated in two distinct folders :

- the folder `resources` containing graphic assets, the game business logic, the gameplay though Lua scripts
- another folder `sources` containing Java files acting as an integration layer for Android (handle notifications, third parties SDK, permissions, etc.)

Investigating bytecode Lua files, I discovered the critical piece of the encryption : a file named `rc4.lua` . RC4 is a fast symmetric encryption algorithm. The most interesting part was that the developers had fully reimplemented the algorithm themselves. I used the command line tool **Unluac** to decompile Lua's bytecode file to readable Lua.

Now that I had the algorithm, I had to find where it was called, and which key was used.
When I intercepted the traffic, I was able to note the endpoint, so searching for that specific endpoint in the Lua files drove me to the function `base64_decode_rc4`. That function was created in another file, and I was able to identify the key structure.

The RC4 key was set up like this in this function :


```lua
local _rc4 = RC4.RC4(key .. AccountLib.client_secret, _base64_decode)
```

with the `key` variable directly hardcoded inside the function.

The next thing to do, was to find the `client_secret` inside the `AccountLib` in order to find the complete key. As the name implied, it is related to the account, and when I analyzed traffic I discovered that upon first connection, the application was sending in plain text to the server the wanted `client_secret` !

So the picture was set :

1. The application is generating a `client_secret` and send it to the server.
2. The server is then sending the response in an encrypted base64 with RC4 algorithm
3. The application decrypt the payload thanks to the hardcoded key and the `client_key` generated and decode the base64

Another file worth mentioning, was the `gameNet.lua` and especially this line :

```lua
t_sign = ezMD5_Lua(t_user_key .. t_ex_data_json .. AccountLib.client_key .. AccountLib.client_secret)
```

This line acts as a request signature using MD5 hashing over a concatenation of session values and the request payload. Note that this is not a proper HMAC construction, it lacks a secret key processed through a keyed hash function, making it weaker than standard HMAC-MD5.

## 4. Findings

Here are the interesting findings that I made :

- Session values ()`client_key` , `client_secret` and `account_key`) transmitted in plaintext during initial connection.
- RC4 key partially hardcoded inside Lua's bytecode
- Reimplementation of the RC4 algorithm
- Custom MD5 signature scheme (non-standard, weaker than HMAC)
- Old version on PHP (5.3.8 and 5.6.4) obsolete and vulnerable

## 5. Ethical considerations

During my investigation, I allowed myself to only understand how the game was communicating with the server and what was the data that was encrypted thanks to the RC4 algorithm. On the other hand, I choose not to try to forge request by manipulating the HMAC. I don't want to harm the game and the players. I did not share any secret keys or secrets found along the way.

## 6. Conclusion

This whole analysis helped me to understand how the game was actually communicating with the server and how the RC4 algorithm works under the hood. Funny enough, when I managed to decrypt the server response I was a bit disappointed that nothing was really hidden. The output JSON was not hiding any sensitive or any more information.

So I wondered why bother with the encryption ? Why use it in just a few API endpoints ? The most probable answer is that maybe it's to prevent bots from easily reading transmitted data and it's acting as an anti-cheat.

As a developer, I would have kept the encryption, but for all the API endpoints for coherence. I also would have made the key slightly differently, including a rotation schedule. As for the HMAC, I would add a time component, such as the age of the request : on the server side, I would reject requests older than one minute. I would also change the MD5 hash algorithm for a more secure one, like SHA-256. Obviously, PHP would also be updated to a more secure and modern version
