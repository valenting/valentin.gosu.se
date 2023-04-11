---
layout: post
title: "URL interop 2023 in Firefox. Guest starring ChatGPT"
---

It's been a while since I last wrote a blog post, so I wanted to do one about a topic that's close to my heart. When I joined Mozilla almost 9 years ago as a full time employee, one of the first important bugs that I got was a bug in our URL parser. Little did I know that by fixing that one small bug I'd soon come to own the URL parsing code in Gecko.

Fast forward to today: one of our teams goals for this year is the web-platform interop for URL. And because starting a blog post from scratch is frustrating, I decided to make it interesting. No, I did not use ChatGPT to generate this blog post, I used it as an interviewer, so it would ask me questions about the topic. It was a rather pleasant experience. My rubber duckies are rarely this interested in what I have to say. Here it is:

**ChatGPT**:

Can you start by explaining what the URL standard is and why it's important for web browsers like Firefox to implement it correctly?

**Valentin**:

The [URL standard](https://url.spec.whatwg.org/) is a WHATWG spec that describes how browsers should parse URL and the DOM APIs that can be used to manipulate URLs. 

**ChatGPT**:

Thanks for that explanation. Can you describe how Firefox implements the URL standard, and what specific challenges the team faced during implementation?

**Valentin**:

Well, the Firefox implementation has existed for longer than the URL standard. As such, there are [10 separate URL parsers](https://firefox-source-docs.mozilla.org/networking/url_parsers.html#implementations) in Firefox, each one parsing one or more different protocols.

**ChatGPT**:

That's interesting. Can you explain a bit more about the different URL parsers in Firefox and how they handle different protocols? How do these parsers work together to ensure proper URL handling across the browser?

**Valentin**:

Well, the core of things is the [nsIURI](https://searchfox.org/mozilla-central/rev/11dbac7f64f509b78037465cbb4427ed71f8b565/netwerk/base/nsIURI.idl) interface. Reference counted URL objects implementing this interface get passed around through method calls or get saved as member variables. A few years ago, when XPCOM addons were still a thing, nsIURI objects were created by calls to `nsIProtocolHandler.newURI`. Because you could have any number of protocol handlers, and they could return nsIURI objects backed by any implementation, at the time URL parsing was actually limited to the main thread (since those URLs could be implemented in JavaScript). Since then we've made nsIURI objects immutable thus threadsafe, and parsing is now centralized through the [NS_NewURI](https://bugzilla.mozilla.org/show_bug.cgi?id=1536744) method. That said we still have a number of URL parsers. The main ones are [nsStandardURL](https://searchfox.org/mozilla-central/rev/11dbac7f64f509b78037465cbb4427ed71f8b565/netwerk/base/nsStandardURL.h#112), for most common URL schemes, and [nsSimpleURI](https://searchfox.org/mozilla-central/rev/11dbac7f64f509b78037465cbb4427ed71f8b565/netwerk/base/nsSimpleURI.h) which is used for most [unknown schemes](https://searchfox.org/mozilla-central/rev/11dbac7f64f509b78037465cbb4427ed71f8b565/netwerk/base/nsNetUtil.cpp#2030-2037).

**ChatGPT**:

Thanks for the detailed explanation. Can you talk a bit about the interop efforts Firefox is currently undertaking in regards to the URL standard? What steps are being taken to ensure that Firefox's implementation of the URL standard is consistent with other browsers?

**Valentin**:

Well, we've been working on [spec compatibility bugs](https://bugzilla.mozilla.org/show_bug.cgi?id=url) for a while, but mostly chipping away at minor issues. Currently we're passing about 7500 from about 9000 URL web-platform tests. We're tracking a number of bugs to fix tests, but there are 3 that will make the most impact. First, there's a [bug](https://bugzilla.mozilla.org/show_bug.cgi?id=1723456) to reject non-IPv4 hostnames that end in numbers. Then there's a [bug](https://bugzilla.mozilla.org/show_bug.cgi?id=1553105) to make non-special scheme URLs have a null origin. Lastly there's a [bug](https://bugzilla.mozilla.org/show_bug.cgi?id=1603699) to use our DefaultURI implementation as the parser for any non-special schemes. The DefaultURI implementation is backed by [rust-url](https://github.com/servo/rust-url) which follows the URL standard much closely. Using it for non-special URLs is going to make a big difference in the [Web Platform Test pass rate](https://wpt.fyi/results/url?label=experimental&label=master&product=chrome&product=firefox&product=safari&aligned&view=subtest&q=label%3Ainterop-2023-url). At the same time, not using it for special URLs keeps it away from the hot path making performance less of an issue (rust-url is roughly twice as slow as nsStandardURL). At the same time we avoid changing behavior for the most common of use cases for URLs. I think the future involves using rust-url for most of our URL parsing needs, but that is a bigger effort to avoid regressions. In any case, fixing these 3 bugs will fix about 6-700 of the failing test cases.

**ChatGPT**:

Thanks for the detailed explanation of the interop efforts. It sounds like the Firefox team is actively working to ensure compatibility with the URL standard and other browsers. Can you talk a bit more about the impact that these changes will have on Firefox users and web developers who use Firefox to build web applications?

**Valentin**:

Well, URL parsing is a core task for most web applications. Usually parsing is fairly straightforward, but the more complex and niche an application gets, the odds of hitting one of these corner cases increases and it becomes important that all browsers behave in the same manner. For example, we're seeing more and more web applications that were actually written to run on a full operating system, but have been compiled to run on the web. Think SSH clients, databases, etc. Having a good webcompat story will make a big difference to the developers of that kind of webapps.

**ChatGPT**:

That's a great point. It's important for browsers to have a consistent implementation of the URL standard so that web developers can create applications that work across different browsers without encountering unexpected behavior. How does the rust-url library differ from Firefox's nsStandardURL implementation in terms of URL parsing and manipulation?

**Valentin**:

Well, there are a few. First one being that rust-url is written in the rust language with a small C++ wrapper for DefaultURI, while nsStandardURL is entirely written in C++. This means it's more likely there are security bugs still hiding in nsStandardURL even after more than 20 years than in rust-url. Another difference is the general direction. Rust-url was started for use in the servo browser with the goal of implenting the URL standard from the beginning. nsStandardURL initially implemented the URI RFCs, and throughout the years we've made changes to cover corner cases that were not specified in the RFCs, to address webcompat issues with different implementations, reduce the technical debt accumulated in those 20+ years. Finally I did mention performance. Some of our microbenchmarks showed rust-url was more than 2 times slower on basic URLs. URL parsing is a hot path, so if we ever want to replace nsStandardURL with rust-url, then we must focus on performance in rust-url. I believe there are some low hanging fruit, this is definitely doable, but that is not currently part of the interop effort.

EOF>
