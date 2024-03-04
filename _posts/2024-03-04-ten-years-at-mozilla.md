---
layout: post
title: Ten years at Mozilla
---
March 4th 2014 my first day working for Mozilla full time software engineer. I've now been on the Networking team (Necko) for 10 years. A bit more than that if you count the 3 months I spent as an intern in 2012.

These years have seen living in Romania, then Czechia and finally Sweden. I've gotten married and had two kids. I've grown so much as a person and as an engineer that I would barely recognize myself. I've had the great luck of having some of the most wonderful managers, I've worked with some of the best engineers in the world, I've gone to so many all-hands, and I've seen a few dozen people come and go on the Necko team.

## My journey with Mozilla

My journey with Mozilla started around 2004 or 2005 when I installed the first version of Firefox. I don't remember exactly how I found it - I just remember that I was frustrated with IE and went through a few browsers before I settled on Firefox. I even tried a version of Netscape if I recall correctly.

Fast forward a few years to 2009 when I was at university, and a few contributors to Firefox localization came to my university to give a talk about Firefox. They only had one T-shirt to give, so they asked how many years had Firefox been around, which I answered correctly. I still have that shirt, and I treasure it dearly.

My years at university were great. I had the great luck of getting involved with [ROSEdu](https://www.rosedu.org/en/) - a local open-source community. At some point I became a [Mozilla rep](https://mozilla.github.io/reps-archive/u/valentin_gosu/index.html). I organized events, talks and competitions. And in November of 2011 I was invited to MozCamp in Berlin. It was there where I first met face to face with my reps mentor, Brian King, one of the best people in the world, but also with some of Mozilla's Romanian employees and contributors. Dumitru Gherman, an SRE for Mozilla at the time, encouraged me to apply for an internship with Mozilla, which I did and in 2012 I went to Mountain View  and worked as an intern for the networking team. I worked on about:networking - a dashboard to display networking stats similar to Chrome's about:net-internals. I didn't manage to land the code before my internship completed, but it took another few months of back and forth reviews until it finally landed in Firefox.

By this time I had taken a job at Ixia (since acquired by Keysight). It was a nice gig. Working on the same team as my best friend - Stack Manager, and at the same company as my girlfriend. The work was interesting. I implemented and maintained protocols such as PPP and DHCP. But as all reorgs go, I was assigned to a different team. The new project wasn't exactly what I was interested in, so I started looking for other opportunities, and lucky for me, Necko had an opening.

The 14th of February was my last day at Ixia, and the 4th of March was my first at Mozilla. It was a quick onboarding, a lot of information from my internship was fresh in my mind. A few weeks later we had a Necko work week at the Paris office.

The next year I married my high-school sweet-heart. Soon after we moved to Czechia when my wife accepted a job at Skype. Even though I had two team mates in Prague, we only met IRL two or three times. The year I spend in Prague was quite memorable - I was a night owl, working mostly in the afternoon and late nights. I loved the work, but working from home was starting to take its toll, so I enrolled in a master's programme at KTH in Stockholm.

Around this time Mozilla had shut down the FirefoxOS project, so around 5 people from the Taipei office who'd previously worked on that joined Necko, which meant that the team had around 12 engineers at the time. It was a very nice period, and as we all settled into our roles, it felt like the team was rocking it. Unfortunately, about a year later a reorg made it so that the the Taiwanese people who worked on the team were required to relocate to a different office. Only two people chose to make that move.

2018 also saw the team's manager, lead and a couple of other engineers leave the team. While a new manager and another great engineer were hired, the team struggled to regain its previous momentum.

In September of 2018 I welcomed into the world my first daughter. My colleague Daniel told me that everything changes when you have a child. I thought I understood what he meant, but I didn't. I expected that the world goes on as usual, but there's also a child there. But no. The moment they put her in my arms, she became my world. While the work I do is still important to me, I no longer think it's the most important thing I've ever done.

Working from home is though at times, you feel disconnected from your colleagues, might miss the water-cooler talks with your team mates. But when you have a child, there's nothing better. Being able to be there and not miss my daughter's first steps, first words or just the occasional detour to take a peek at her playing whenever I took a bathroom break are worth everything.

2019 were a bit of a blur. I was now working on DNS over HTTPS, which was super interesting, and I had finally managed to make nsIURI threadsafe (a huge piece of technical debt). I also took a bit of paternity leave, which was just wonderful.

Then came 2020, the pandemic, the loss of two important people in my life. This year also saw big layoffs at Mozilla, which hit the networking team pretty hard. Necko was down to 3 engineers, including myself. The morale was pretty bad, and the team struggled to deal with the loss of engineering talent and expertise.

2021 saw the birth of my second daughter, a ray of sunshine in an otherwise pretty bleak pandemic. As far as I remember I mostly focused on DNS over HTTPS during this time.

2022 and 2023 saw the networking team slowly grow back to a more reasonable size. And me becoming the module owner of the Networking component - something I would have never expected or even hoped for. It's a big responsibility.

And finally, 2024, the present. I feel the Networking team is again in a place where we're not just putting out fires. We're building and tackling some hard problems. This year's focus is performance, and I'm quite optimistic about it.

## Code graph

[My commits](https://github.com/mozilla/gecko-dev/graphs/contributors?from=2014-01-01&to=2024-03-01&type=c) for the past 10 years.

<img src="/images/code-graph-moz-10years.png">
## So what have I learned?

**Mozilla is an amazing place to work.** A month or so into my internship I remember saying that "Mozilla has spoiled me for other companies." And I still think that's accurate. At a time when most tech companies only think about the bottom line, working for an *ethical* company is just fantastic. I could be making more money at other companies, but what would I have to compromise?

**Maintenance work is hard and sometimes thankless.** The networking code in Firefox is a beautiful mix of 20 year old C++ code we barely ever touch and modern C++ or rust code we just developed. It's like an intricate jenga tower - with a lot of assumptions built in. If you don't want to make any improvements it can probably keep running for years, but if you want to change anything you'd better have a capable team to deal with any regressions and breakage. And unfortunately the value of such team is most obvious when something goes wrong.

**Mozilla's leaders are fallible people.** I've seen a quite a few reorgs, layoffs and changes in leadership. While people hold Mozilla to a higher standard due to our mission, it's become apparent to me that the people in charge are as fallible as anyone. They make mistakes, they make the wrong choices, they don't know everything, and they're sometimes selfish. With all that, I wouldn't want to be in their shoes. Keeping a company financially viable is hard work. Doing that while also abiding by Mozilla's principles is an almost impossible task. And yet, the balance of the web rests on it.

**I really love all of the people I've had the luck of working with.** I can't stress this enough. While I'm sure might be people at Mozilla who are actual jerks, all of the people I've had the honour of working with over the years have been amazing. I've learnt so much from these wonderful humans and engineers. Though many of them I haven't seen in years, I still miss them every day. They've made this a great workplace, a home, a place where I can grow and be happy. While compared to most of them I'm but a mediocre engineer, I feel just working with them has made me better in so many ways.

**The work never ends**. This one took me a while to get. There's so many bugs, so much to do. Even if I were to work 24 hours a day, I would probably never reach a point where the codebase is as polished as I'd like it to be. And while the times I did try to push extra hard did lead to a small increase in productivity, burnout surely followed. Slow but steady beats fast at an unsustainable place.

**There are a lot of things that are not within our control.** Even if I did everything perfectly, even if all my colleagues did too, and even if we all worked overtime, it might not make as big a dent in Firefox's market-share as we'd like it to. It's hard to compete against browsers with 10X our funding and engineers. It's hard to compete with browsers that own the most visited page in the world and use it for advertising. It's hard to compete with browsers that come preinstalled. The best we can do is our best.

**I love what I do**. A friend once took offence to my use of the phrase "dream job". They worked to live, not lived to work. And I fully agree with their stance. But if I had to choose between doing something I enjoy for 8 hours a day, and something that I can't wait for it to be over... I'd choose to do what I love all day with people I respect.

## Conclusion

In conclusion, 10 years is a very long time. In this time I've changed, Mozilla has changed and the world has changed around us. I have no idea what the future holds. I hope Mozilla remains a beacon of hope for the open web, and I hope to be there to do my part towards the mission. But even if the future isn't that bright, I can't help but look back on the past 10 years with pride and joy, and be thankful to everyone that stood besides me in this wonderful journey.