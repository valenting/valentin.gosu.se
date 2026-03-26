---
layout: post
title: "getaddrinfo sucks. everything else is much worse"
---
DNS is one of the critical building blocks of the internet and of the modern web.
For the longest time the only way for Firefox to resolve a DNS domain was by using `getaddrinfo`. 
What's remarkable about this function is that it's implemented on [Linux](https://man7.org/linux/man-pages/man3/getaddrinfo.3.html), [Windows](https://learn.microsoft.com/en-us/windows/win32/api/ws2tcpip/nf-ws2tcpip-getaddrinfo), [MacOS](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/getaddrinfo.3.html) - even Android. It has the same signature, and works in roughly the same way, even though the implementation in these operating systems doesn't share the same code base.

getaddrinfo was brought into the IETF in [RFC 2133 - Basic Socket Interface Extensions for IPv6](https://datatracker.ietf.org/doc/html/rfc2133#section-6.3) along with the rest of the socket API from [Protocol Independent Interfaces, IEEE Std 1003.1g, DRAFT 6.3, November 1995](https://www.open-std.org/jtc1/sc22/open/n4217.pdf). This RFC was later updated in [RFC 2553](https://datatracker.ietf.org/doc/html/rfc2553) and finally [RFC 3493](https://datatracker.ietf.org/doc/html/rfc3493#section-6.1).

Being defined in a IETF RFC meant everyone was incentivized to implement it in the same way. 

There are a few ways in which `getaddrinfo` kinda sucks. Most of these are well known, but it's important to recap:
- it's a synchronous API. That means the thread calling this function is blocked until the function returns. This means you need a separate thread for DNS resolution to avoid blocking the main thread and UI of your application, or ideally multiple threads when you need to resolve multiple domain names quickly, in parallel.
- it's very limited. It gives you a way to retrieve IPv4 and IPv6 addresses, and the CNAME of the final domain in the CNAME chain. There's no way to get the TTL of a record, no way to resolve other kinds of DNS records, no control over OS caching of the domain names.
- If the failure happens due to a specific DNS error (e.g., SERVFAIL, NXDOMAIN, etc), getaddrinfo doesn’t expose these raw DNS details.
- Implementation bugs and differences: [1870496 - DNS resolution fails on Android if any label in a CNAME target begins with a hyphen or an underscore](https://bugzilla.mozilla.org/show_bug.cgi?id=1870496). This is a bionic bug in Androids glibc library.

In Firefox `nsHostResolver` has a thread pool dedicated to DNS resolution. DNS requests are put into queues for different priorities and each thread will wake up and pick up the highest priority request and call `getaddrinfo` for that domain.

For other issues, like the missing TTL info there's no real workaround. That means we don't know for how long the DNS record is valid for. We could call `getaddrinfo` every time, but the OS cache isn't unlimited, so it will unnecessarily generate DNS requests for entries that haven't expired yet.
The way we worked around that in Firefox was to use a different API [DnsQuery_A](https://learn.microsoft.com/en-us/windows/win32/api/windns/nf-windns-dnsquery_a). This one is only available on Windows, which isn't great, but it did improve caching for most of our users. The way this was implemented was that we'd call getaddrinfo, then we'd call DNSQuery_A and only extract the TTL.

A few years back we implemented [RFC8484 - DNS Queries over HTTPS (DoH)](https://datatracker.ietf.org/doc/html/rfc8484). That means the raw bytes of the DNS request and response are send via a HTTPS request to a DoH resolver. Now Firefox needed to be capable of parsing the DNS packet from the wire representation, but that did give us some additional benefits. TTL values were now available on all platforms, and we could also parse other record types, such as CNAME, OPT and TXT.

And of course, the most recent and useful of record types, [SVCB and HTTPS Resource Records - RFC 9460](https://datatracker.ietf.org/doc/html/rfc9460).
Why are HTTPS records so great? Let's say you want to connect to `http://www.example.com`
Before that connection is made, Firefox will resolve the A, AAAA and HTTPS records.
What the HTTPS record could contain is something like this:
```
$ORIGIN example.com.  ; domain for CDN 1
www  7200 IN HTTPS 1 . alpn=h2
www  7200 IN HTTPS 2 . alpn=h3 port=8443
```

So even before connecting, Firefox will know a few things:
- This host has a HTTPS record, so there's no need to use plain-text HTTP. We should always use HTTPS.
- The host supports HTTP/2 on the default port
- If we want to use HTTP/3, we can connect to port 8443, while the URL bar will still show https://www.example.com (note that the origin will still be www.example.com:443 - default HTTPS port).

Additionally the HTTPS record may hold other parameters such as ipv4hint and ipv6hint if we're resolving in parallel with A and AAAA. And the `ech` parameter for [TLS Encrypted Client Hello](https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni-22), which allows you to hide the SNI field in the TLS handshake from any network observers.

Now this is all great, but most Firefox users don't have DNS over HTTPS turned on by default. So what do we do to get them some of this technological and privacy goodness?
We try to resolve HTTPS records using the different system APIs available on each platform.

But why use native APIs instead of implementing a DNS/TCP resolver?

- using the same resolver as the rest of the OS
- platform specific quirks and configuration
- caching of responses

When I started out this endeavour I had a few expectations for these APIs.
- they exist
- they are well documented
- they actually work

I was really surprised by how often these APIs failed my expectations.
# Resolving HTTPS records on different operating systems
## Linux - res_nquery

This function is [well documented](https://man7.org/linux/man-pages/man3/resolver.3.html), thread-safe, and works as expected.

```cpp
int res_ninit(res_state statep);
void res_nclose(res_state statep);

int res_nquery(res_state statep,
          const char *dname, int class, int type,
          unsigned char answer[.anslen], int anslen);
```

The [implementation in Firefox](https://searchfox.org/mozilla-central/rev/dd8b64a6198ff599a5eb2ca096845ebd6997457f/netwerk/dns/PlatformDNSUnix.cpp#62) works on all Linux systems as far as we can tell.
The `answer` buffer will be filled with the response bytes, that can then be parsed using the same function we use for DNS over HTTPS.

## Android - res_query

`res_query` was our first find. It just doesn't work. At all. It exists in libresolv, but calling it always returns -1, regardless on what parameters you pass to it.

I did think about whether it matters if this is called by a simple NDK binary, or an app with permissions. Regardless, whatever I tried it never seemed to work. It didn't resolve anything, A, AAAA, HTTPS.
I didn’t find much documentation, apart from a lonely stack overflow question and a few other forum posts.

[Stack Overflow: Android NDK DNS resolution with libresolv](https://stackoverflow.com/questions/63486579/android-ndk-dns-resolution-with-libresolv)

So the implementation is completely borked. Did it ever work? Unclear.
But that did give us a good idea where to look next.
## Android - android_res_nquery

[Networking #android_res_nquery > Android NDK > Android Developers](https://developer.android.com/ndk/reference/group/networking#android_res_nquery)

Here's a short gist with a [sample implementation](https://gist.github.com/valenting/1aee0a1209da05118a2a3fa24762f5e1), and a [link to the Firefox implementation](https://searchfox.org/mozilla-central/rev/d1fbe983fb7720f0a4aca0e748817af11c1a374e/netwerk/dns/PlatformDNSAndroid.cpp#53-90)

There are a few things to note about this API:
- It needs dynamic linking. It was first added in Android 10, so you need to look for the `android_res_nquery` symbol in `libandroid_net.so`
- The call returns a file descriptor. You can then poll the file descriptor and read from the file descriptor.
- Interesting observation: the returned buffer also contains the UDP header in the first 8 bytes.

My main gripe with this API, it’s only available on Android 10 and up.
Fortunately most Firefox users on Android are using these newer Android versions, but it would have been nice to support users with older phones too.
It’s relatively well documented, and you can specify some caching flags too, and the network interface.
I haven’t actually checked if these flags work as expected, but I haven’t seen anything that indicated that’s not the case.

## Windows - DNSQuery_A

[DnsQuery_A function (windns.h) - Win32 apps](https://learn.microsoft.com/en-us/windows/win32/api/windns/nf-windns-dnsquery_a)

Since Firefox was already using this function, I didn't expect any surprises.
Well, while DNSQuery_A does a good job at resolving A and AAAA records, when testing it out for HTTPS records we had a crash on Windows 10 platforms.
**It seems even though it returns `ERROR_SUCCESS` on Windows 10, and Wireshark shows the request and response, the `ppQueryResults` argument gets set to `NULL`.**

```c++
DNS_STATUS DnsQuery_A(
  [in]                PCSTR       pszName,
  [in]                WORD        wType,
  [in]                DWORD       Options,
  [in, out, optional] PVOID       pExtra,
  [out, optional]     PDNS_RECORD *ppQueryResults,
  [out, optional]     PVOID       *pReserved
);
```


You can build this yourself using the code in this [gist](https://gist.github.com/valenting/64ff50a3d86557ea0289d15cb55c8bef). Our solution was not use this function on Windows 10. Fortunately it works fine on Windows 11.

If you're interested in pushing Microsoft to get this fixed, feel free to upvote this [feedback hub issue](https://aka.ms/AAo4y7b)... assuming you're currently using Windows (rant: why aren't you able to view issues in a web browsers but instead you need to use Microsoft's Feedback hub?)

Some things I like about the windows API:
- It's really powerful. It has [lots of options to control how the Query is made](https://learn.microsoft.com/en-us/windows/win32/dns/dns-constants#dns-query-options), caching, use of hosts file, recursion.
- The response is a linked list of records similar to getaddrinfo.
- The `DNS_RECORD.Data` field is [a union](https://learn.microsoft.com/en-us/windows/win32/api/windnsdef/ns-windnsdef-dns_recorda), so even if the windows library doesn’t know how to parse the record, you still get a buffer of bytes and the length of the record, which is definitely enough to parse any DNS record type you want.

If it weren’t for the Windows 10 implementation bug this API would be my favourite.
Note that the Windows 10 vs 11 quirk may or may not apply for other record types.

## MacOS - res_query

[Mac OS X Manual Page For res_query(3)](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/res_query.3.html)

Initially quite optimistic about this. It seemed to be rather close to res_nquery.
But for some reason we were seeing random failures in CI.
[1882856 - Crash in @ dns_res_send  with network.dns.native_https_query on MacOS](https://bugzilla.mozilla.org/show_bug.cgi?id=1882856)

One wonderful thing about getaddrinfo:
> Functions getaddrinfo() and freeaddrinfo() must be thread-safe. [rfc3493]

It turns out res_query isn't thread-safe, though that is entirely absent from Apple's man page.
That's not to say that it can't be thread-safe. But of course, it's not, and probably it won't be seeing any work done in the near future.

If your application only resolves HTTPS records on a single thread, you may use this one without much worry.

## MacOS - DNSServiceQueryRecord

[DNSServiceQueryRecord  Apple Developer Documentation](https://developer.apple.com/documentation/dnssd/dnsservicequeryrecord(_:_:_:_:_:_:_:_:)?language=objc)

I don't have much experience developing for MacOS, but compared to all other platforms, Apple's documentation really really sucks! I couldn't find any examples of this function being used on official Apple sites. Maybe there is one, hidden behind an Apple developer account login that you have to pay for, but none that my google-fu could locate.

Similar to `android_res_nquery` - returns `sdRef` which can then passed to `DNSServiceRefSockFD` to get a filedescriptor. Then we can poll or select on that fd. Once we wake up we can call `DNSServiceProcessResult` which causes the callback to be called. If we call `DNSServiceProcessResult` directly, it will hang until a response comes back.

To my surprise, what I found out is that if the DNS response doesn't contain any records, `select` and `DNSServiceProcessResult` just hang forever.

> [!WARNING]
> 😱 I really hope this is a bug, because this makes literally no sense. I don't know a lot about MacOS, but this can't be what the person designing this API intended to happen. Do I really have to set a timeout because the API isn't smart enough to continue when the records I'm looking for don't exist? Is there an undocumented flag I'm missing? Is there yet another API I could be using, because this super common use case is broken in this one, which makes me think nobody actually uses it.

Anyway, here's a [gist](https://gist.github.com/valenting/dd1125978313320ee0ab7e52768ea48f) that shows this bug. If you definitely do need HTTPS records, make sure to set a timeout so this call doesn't block forever. (I'll also file a bug with Apple about this eventually).

> [!NOTE]
> (2026-03-26) After a long time I went back to investigate potential solutions. It turns out there is a workaround, which is to pass the [kDNSServiceFlagsReturnIntermediates](https://developer.apple.com/documentation/dnssd/kdnsserviceflagsreturnintermediates?language=objc) flag to the `DNSServiceQueryRecord` call. This makes it so the NXDOMAIN result is immediately returned, and also intermediate CNAMEs. In my opinion this should have been the default, but at least we eventually fixed this in [bug 2026152](https://bugzilla.mozilla.org/show_bug.cgi?id=2026152).

---

Anyway, the main takeaway from this experience is that after all this time DNS is not a solved problem, even for a coder that just wants to use it without actually understanding the protocol.

If I were to make any recommendation to other folks that need HTTPS records over unencrypted DNS (Do53) I'd say:
- Maybe implement your own TCP/UDP resolver or use an existing implementation.
- Stick to one OS if possible. Dealing with all of them is painful.
- Make note of my warnings.
- Document any corner cases you encounter.

So why did I say getaddrinfo sucks? It's because it was designed for POSIX. Do one thing, and do it well. And in that sense it succeeded without a doubt. Could it be better, more flexible and more extensible? Sure. But if that were the case it would also probably be more buggy, and have a bunch more differences across platforms.
But what of the other APIs we looked at today? Well, these are the results of each platform doing their own thing. Some had a common starting point - see res_query - but slowly diverged to accommodate the specific requirements of their users and platform.
What we now have is a buffet of choices, each with their quirks and buggy implementations.

I think there is still some value in using these OS specific APIs. Especially if your requirements are limited to one OS or a limited set of use cases that manage to avoid all the bugs.

What I would like to have in the future? Some sort of library or implementation that provides a common API that looks and asks the same (within reason) on all platforms. Some things I'd like:
- A form factor similar to DnsQuery_A/Ex - lots of options for caching,  /etc/hosts, network, protocol
- Like android_res_query - access to the socket and response bytes
- Better control over request bytes as well. Being able to add OPT padding for example would be a nice feature to have.
- Like getaddrinfo - I’d like something that works well everywhere!

---

I gave a talk based on the above at FOSDEM 2025 in the DNS room.

[getaddrinfo sucks, everything else is much worse](https://fosdem.org/2025/schedule/event/fosdem-2025-4229-getaddrinfo-sucks-everything-else-is-much-worse/)

The video should be up soon, and the [slides](https://fosdem.org/2025/events/attachments/fosdem-2025-4229-getaddrinfo-sucks-everything-else-is-much-worse/slides/236940/getaddrin_TExgCsb.pdf) are already available.
It was a great experience, my first FOSDEM talk, and I got a bunch of good questions.
One of them was from Stéphane Bortzmeyer, who pointed me to [getdnsapi.net](https://getdnsapi.net/)
My first impression was that it would be a much better API than what we currently have, though I'd still like to have access to the DNS response buffer. And of course, have this available on all operating systems. I'm not holding my breath 🙂.
