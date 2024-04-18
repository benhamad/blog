# Reverse engineering illegal IPTV apk 

I recently came across an illegal IPTV streaming Android application popular in
my home country. This application offer pirated IPTV channels. Most of these
channels are FTA channels, but some are paid (Bein sports, OSN, etc). A lot of
these applications are available on the internet, but this one caught my
attention because it's available in the Google Play Store with more than 10M
downloads. So I went ahead and installed the application to figure out how it
works and where the content is coming from.

In the Play Store the application is advertised as a free IPTV player,
and that it doesn't have any content. And users will have to provide their own.
Well, we will see later on how that is not true. We will go through how to set
the application up and how it works in the next section. The same application is
available in the Apple App Store as well. But due to the restrictions on iOS, it
won't be easy to figure out how it works compared to Android.


## Setting up the application

As I already said, the application can be downloaded from the Google play store.
After installing it, you will be greeted with a prompt for m3u8 link. This is
where you will need to provide the link to the IPTV service. The idea is that
you will need to subscribe to a service on your own and get the m3u8 link to be
able to watch the content.  Hence why the application is advertised as a free IPTV
player.

That said, a particular link is heavily shared in the internet to be used with
this application, it's available to anyone searching for the name of the
application online. And with tiktok and other social media platforms, it's
shared widely.

This link is for a service called "fgcode". This service seems to be offering
some sort of playlist editor, where you provide your own m3u8 playlist and can
edit it and combine other playlists. The result will be a shortened link in the
form of `fgcode.org/PLAYLIST_ID`. That link will have the m3u8 playlist you
provided.

This is where things get interesting.  The heavily shared link
`fgcode.org/232425` is for an empty playlist empty m3u8 link.

```bash
[chaker@chaker-yoga drama_live]$ curl fgcode.org/232425 -L  -w "\n" 
	#EXTM3U
[chaker@chaker-yoga drama_live]$ 
```

But it's treated differently by the application. When you provide this link, the
application loads a huge list of channels from an API hardcoded in the
application. On the other hand when you provide a genuine M3U8 playlist, the
application will only load the channels from that playlist. 


Note on `fgcode.org`: This service seems to be some kind of url shortener for m3u8
links. But one flaw is that it doesn't rate limit requests and the id of the
playlist can be brute forced. But it's not out main focus here.


## Intercepting the traffic

To figure out how the application works, I started by intercepting the traffic
between the application and the server. I used HTTP Toolkit to intercept the
traffic. I will skip how to set up HTTP Toolkit, but you can find the
instructions on their website. 

All requests issued by this application were plain HTTP, which made it easy to
intercept them, without the need to install a custom CA certificate. Listening
to the traffic, I noticed that the payload of the requests issued by the
application were encrypted though. It was clear that it's base64 encoded string,
but I needed to figure out what kind of encryption it's. 

One of the requests's payload:
```
d/HSGvbVOH5fxLSGrEPAc5RM1RDVeNVFIIjnNoaP04ajqhOpKdLZw2QT+eFnlE9U2lL53TMtgDG9
D1k6b0EevkH+q02Sdpan2pn98LYvbWt4aXcE9t7XGD/jEuXwL/u36mt60OrLPKqx7HptibGM04e8
8xQJ0FsK/yUeIuuKFND2uNYia7DI7FTb5FlHaCwJP1FRLuhvjGeMTl7F8kk6/iEUbkAxc3SpHrVf
bW1x4YpmF+fnKrn2GLhxpuUbo2nnIqYbnScPd/rW3za//0wy4lVsRaI77U6HRnDjxQ6VBr7CinuR
RSBXb5KbMrkWUl5ORMpD7O+hBr3SXCEYqp5MzVeh/Y7gw3AtvoYPEozBbjhZ1BPsXNXXZ3GeSXsG
Odc+s7jVkkYWC+xP9cX2xnpHlmN04OTFs391Wd6bBe3yCeoTcCwpbj7NPtIsB3MGjSFBJl8MriYP
4+rcTqf2nWue8dquIc67utXjtBCBae28liGcL9ONun3Dd2YjTejWlP8oe2IiCxGD1khgDN+M34gd
TgK9lrNN1J1LnTANnPPctEKEiAbipFbL+RRjdAqIglsi5B+f9EkuXg5v9jnFSRhA8af4dq1kvbOo
6gKTKNDY3c3TXZSBaIL4/fkzReTydtPo/7rTwc98Lckl0bI5lp25k7mou2Fj1xHYY3qEbXLxJjEN
rZP7SgbhTNvhPmFs0Dls
:ZmVkY2JhOTg3NjU0MzIxMA==
```

So we need to reverse engineer the application to figure out how the requests
are encrypted and decrypted. We will go back to listening to the traffic later
on.

## Reverse engineering the application

I tried using apktool to decompile the APK first. But it only gave me SMALI
files, and though that may be enough to figure out what was happing, jdx was
more suited in this case. I used `jadx` to decompile the APK. And although the
decompilation wasn't without errors, I had enough decompiled code to work with.

```bash
[nix-shell:~/Downloads/apk/drama_live]$ jadx drama_live.apk 
INFO  - loading ...
INFO  - processing ...
ERROR - finished with errors, count: 137      
```
Finding the name of the main package of the application:

```bash
[chaker@chaker-yoga drama_live]$ fd drama
sources/com/sneig/livedrama/
```

It's `com.sneig.livedrama`. I will be mostly searching in this package. Though
the decryption and encryption functions may be in another package. We will see
how the code path goes.

Our main goal is to figure out how the requests are issues and there we can see
how they're encrypted. One of the requests that I saw when intercepting the
traffic was for a `getSettings` endpoint

```bash
[chaker@chaker-yoga livedrama]$ ag getSettings -l 
a/f/c.java
a/c/a.java
a/b.java
j/b/e.java
j/b/n.java
g/x0.java
fragments/PlayerFragment.java
```

Checking the matched files, only `j/b/n` was issuing an HTTP request. The other
were references to a Java function called `getSettings`. 

```bash
[chaker@chaker-yoga livedrama]$ ag getSettings j/b/n.java 
134:        String str = com.sneig.livedrama.h.n.j(this.b).g().q() + "getSettings";
```

This is the particular function where the endpoint is being requested. Due to
decomplication we don't have the original variable names, but we can still
follow the code.

```java
    public void d() {
        h0.a.a.a("Lana_test: Networking: %s: run ", this.a);
        String str = com.sneig.livedrama.h.n.j(this.b).g().q() + "getSettings";
        JSONObject a2 = com.sneig.livedrama.j.a.a(this.b);
        b bVar = new b(a2.length() == 0 ? 0 : 1, str, null, new a(), com.sneig.livedrama.h.r.a(a2.toString()));
        bVar.M(new z.b.b.e(0, 1000, 1.0f));
        bVar.O(this.a);
        com.sneig.livedrama.h.j.c(this.b).a(bVar, this.a);
    }
```


The following line seems to be building the URL for the request. We can see the
URLs is dynamically built.

```java
	String str = com.sneig.livedrama.h.n.j(this.b).g().q() + "getSettings";
```
And `a2` seems to be the payload of the request.

```java
	JSONObject a2 = com.sneig.livedrama.j.a.a(this.b);
```

This is important since we can follow where `a2` is used. 

```java
	b bVar = new b(a2.length() == 0 ? 0 : 1, str, null, new a(), com.sneig.livedrama.h.r.a(a2.toString()));
```

So `h.r.a` seems maybe the function that encrypts the payload. Note how a2 is
converted to a string before being passed to `h.r.a`. 

```java
	com.sneig.livedrama.h.r.a(a2.toString())
```
Looking up the `h.r.a` function:

```java
package com.sneig.livedrama.h;

import android.util.Base64;
import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
/* compiled from: Zippi.java */
/* loaded from: classes5.dex */
public class r {
    private static String a = "0123456789abcdef";

    public static String a(String str) {
        try {
            if (a.length() < 16) {
                for (int i = 0; i < 16 - a.length(); i++) {
                    a += "0";
                }
            } else if (a.length() > 16) {
                a = a.substring(0, 16);
            }
            IvParameterSpec ivParameterSpec = new IvParameterSpec("fedcba9876543210".getBytes("ISO-8859-1"));
            SecretKeySpec secretKeySpec = new SecretKeySpec(a.getBytes("ISO-8859-1"), "AES");
            Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5PADDING");
            cipher.init(1, secretKeySpec, ivParameterSpec);
            return Base64.encodeToString(cipher.doFinal(str.getBytes()), 0) + ":" + Base64.encodeToString("fedcba9876543210".getBytes("ISO-8859-1"), 0);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    public static String b(String str) {
        try {
            if (a.length() < 16) {
                for (int i = 0; i < 16 - a.length(); i++) {
                    a += "0";
                }
            } else if (a.length() > 16) {
                a = a.substring(0, 16);
            }
            String[] split = str.split(":");
            IvParameterSpec ivParameterSpec = new IvParameterSpec(Base64.decode(split[1], 0));
            SecretKeySpec secretKeySpec = new SecretKeySpec(a.getBytes("ISO-8859-1"), "AES");
            Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5PADDING");
            cipher.init(2, secretKeySpec, ivParameterSpec);
            return new String(cipher.doFinal(Base64.decode(split[0], 0)));
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}

```

And we found it! This seems to be a simple AES encryption with a hardcoded key
and initialization vector. The key is `0123456789abcdef` and the IV is
`fedcba9876543210`. One interesting thing is that the IV is also base64 encoded
and appended to the encrypted string. In the format of
`base64(encrypted_string):base64(IV)`.


Now that we know how the requests are encrypted, we can decrypt them. With the
help of ChatGPT I rewrote this to a python script with streamlit to create a web
app to decrypt the requests. 

This image is an example of the web app in action.

![Web app](./images/drama_live.png)

## Back to intercepting the traffic

Now that we have a decryption function, we can see how the requests are issued.
I'm going to ignore all the requests that list the channels and focus on one
particular endpoint
`http://live.backendcoreapi.com/api/live/livedrama/v13.0.0/getLiveAllStreamsById`.
This endpoints get an id of a channel (e.g. `live_tv_beinsport1`) and return
links to the live stream.

The requests payload is 
```json
{
  "type": "tv",
  "id_live": "live_tv_beinsport1",
  "name": "Bein S. 1",
  "url": "http://.LS.V2LOAD_BALANCERlive_tv_beinsport1/s",
  "agent": "redirect",
  "backup": "{\"url\":\"https://qt2.dwasat.com/upload/images/logo1.m3u8\",\"agent\":\"Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Mobile Safari/537.36\",\"acceptSSL\":\"1\",\"headers\":{\"User-Agent\":\"Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Mobile Safari/537.36\",\"referer\":\"https://www.yariga.live/\",\"Origin\":\"https://www.yariga.live/\"}} -- advanced -;- {\"url\":\"https://webhdrus.onlinehdhls.ru/lb/premium91/index.m3u8\",\"agent\":\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.1 Safari/605.1.15\",\"acceptSSL\":\"1\",\"headers\":{\"User-Agent\":\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.1 Safari/605.1.15\",\"referer\":\"https://lewblivehdplay.ru/\",\"Origin\":\"https://lewblivehdplay.ru/\"}} -- advanced -;- {\"url\":\"https:\\/\\/www.aflam4you.pro\\/aflam85.json?vid=68&aflam_s=1\",\"agent\":\"Mozilla\",\"headers\":{\"Content-Type\":\"application\\/x-www-form-urlencoded\",\"Referer\":\"https:\\/\\/www.aflam4you.pro\\/\",\"User-Agent\":\"Mozilla\\/5.0 (Linux; Android 6.0; Nexus 5 Build\\/MRA58N) AppleWebKit\\/537.36 (KHTML, like Gecko) Chrome\\/121.0.0.0 Mobile Safari\\/537.36\"},\"data\":\" \"} -- double_redirect -;- ",
  "img_url": "http://3.66.87.188/img/channels/live_tv_beinsport1.png"
}
```

In this case, the main url is `http://.LS.V2LOAD_BALANCERlive_tv_beinsport1/s`.
This is not an actual URL, but the application requests the actual URL from the
server using this link through the `getLiveByRedirect` endpoint. The payload for
that endoint is:

```json
{
  "user_id": "......",
  "device_id": ".....",
  "device_api": "30",
  "version_name": "174",
  "language": "en",
  "timezone": "....",
  "device_type": "phone",
  "KEY_ACTIVATED_TYPE": "232425",
  "store": "playStore",
  "isStoreVersion": true,
  "isPremium": false,
  "isCoupon_active": false,
  "hideAds": false,
  "appCount": "{\"adsFailed\":150,\"adsLoaded\":97,\"adsShowed\":13,\"runCount\":31}",
  "mainServer": "http://main.backendcoreapi.com/api/live/livedrama/v13.0.0/",
  "id": "live_tv_beinsport1",
  "url": "http://.LS.V2LOAD_BALANCERlive_tv_beinsport1/s",
  "agent": "redirect"
}

I changed the values for `user_id`, `device_id` and `timezone` for privacy reasons.

The response is a bit interesting. 

```json
{
  "result": 0,
  "message": {
    "en": "operation succeeded",
    "ar": "تمت العملية بنجاح"
  },
  "data": {
    "url": "{\"url\":\"https://webhdrus.onlinehdhls.ru/lb/premium91/index.m3u8\",\"agent\":\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.1 Safari/605.1.15\",\"acceptSSL\":\"1\",\"headers\":{\"User-Agent\":\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.1 Safari/605.1.15\",\"referer\":\"https://lewblivehdplay.ru/\",\"Origin\":\"https://lewblivehdplay.ru/\"}}",
    "agent": "advanced"
  }
}
```

The `url` field is a JSON string. It does contain a link to a m3u8 file. But
what's more interesting are those HTTP headers. If I understood correctly this
is because the application is using a custom player to play the streams. And
that it uses those headers to bypass some restrictions on the server side. 

## Conclusion

Though the application is advertised itself as only a player, we saw how a
single empty link can be used to activate a huge list of channels. Those
channels are provided by a hardcoded API in the application. 
