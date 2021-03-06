---
layout: post
title: An Exercise in Madness
date: 2007-12-20
tags: ron work
summary: My first time pair programming.
---

Most of the tenets of Extreme Programming seem most useful on small-ish
projects. Today I discovered that, despite my preference for working alone, pair
programming is undeniably useful, even on tiny projects. What qualifies as a
tiny project? I'm going to use it in the sense of one to two pages of code. This
story begins with an application called Foundstone.

The Manitoba Government has a habit of buying large, proprietary software suites
from big-name vendors. One such program is a vulnerability scanner, ticketing,
and reporting suite from McAfee called Foundstone. As far as I'm concerned,
Foundstone is poorly made. Since I'm an open source geek, I am admittedly
biased. Couple that bias with the typical programmer's ego, and you can probably
see where the next bit of this post is going.

Foundstone is an all-in-one, turnkey, 1U 'pizza box' (as Gary would say). This
box is running a *highly* modified build of Windows, to the point that the
top-level of the filesystem represents the [FSH][] as opposed to a modern
Windows machine. It runs on IIS and MSSQL, and uses PHP and GD to generate
reports and provide most of the web interface. All-in-all, no complaints
there. In fact, I am suitably impressed with their filesystem layout.

Sadly, there's more to Foundstone, and that's where I start getting annoyed, and
*fast*. One of the things that I'd been tasked with researching is automating
the reporting. No matter what job I get SBGH, IBM, Seccuris it always seems to
end up with me automating some report or another. After a few weeks of poking
about, I managed to get reports being automatically generated and emailed. These
reports include the details of all vulnerabilities discovered for hosts within
certain administrative boundaries. The only reason it took any appreciable
amount of time to generate was an arbitrary limit on the government's email
servers stating that ZIP files could only contain a certain number of
files. They set it to something like 100, and that wasn't *nearly* enough for
these reports. Oh, no, they needed that number jacked up ludicrously high to
accommodate for the method Foundstone uses for presenting scans. Whether it's an
HTML or a PDF report you ask for, it presents you with a ZIP file. This ZIP file
will contain a directory named with the unique number (in today's case, 20) that
represents the automatically incremented primary key for the scan, taken from
the MSSQL database. Once you go down another level into the HTML or PDF folder,
depending on the type of scan you generated, you'll find the file named
index.{html,pdf}. While quirky, so far, so good. And here's where it all breaks
down. The report is always broken down into innumerable small files listing
every subject that was to be included in the report. This makes sense for CSV
files, which are also an option, but having a slew of tiny cross-linked PDF
files is the sort of thing we need to "resurrect the punishment of stoning for"
(to quote Kyle).

Adding insult to injury, I noticed that in the report that we wanted to send to
the highest-up brass, each graph across two of its sections are ~750x16 pixels,
which is ridiculously narrow. I spent a day trying to find out if I had screwed
up the settings somewhere. Then I called for help:

<dl class="dl-horizontal">
  <dt>Me</dt><dd><em>*relays the last two sentences*</em></dd>
  <dt>McAfee</dt><dd>It's a known issue, which results from an error in the XML file used to generate the report.</dd>
  <dt>Me</dt><dd>Any idea when that'll be fixed?</dd>
  <dt>McAfee</dt><dd>In the next patch, which has no ETA, but I would expect it to come out sometime, hopefully early, next year.</dd>
</dl>

This answer from Technical Support made perfect sense to me, since it
characterized the odd behaviour of this issue. What I've yet to mention is that,
should one ask Foundstone for a live representation of the graphs generated for
the report, it would do so instantly and perfectly. "An error in an XML file," I
thought, "that's the sort of thing that I could fix in two seconds!" Tempting
fate is a hobby of mine. I can often be found saying either "What's the worst
that could happen?" or "There's no *possible* way that this could go wrong!"
and then standing around waiting for everything to go to pot.

Yesterday, Ron and I were asked to develop a workaround for this reporting
issue. It was believed that the easiest solution would be to generate our own
report by direct manipulation of the MSSQL back-end. Neither Ron nor I have any
appreciable experience with MSSQL, but we dove in, regardless. As we laboured to
decipher the meaning of the tables, and suppressed our revulsion toward their
organization (what's the difference between an asset and a host?) and backup
strategy (having a dozen tables named like 'Hosts' through 'Hosts_12'), we found
ourselves unable to find the correct arcane combination of SQL commands to
produce numbers that matched those shown on the known-good graphs. We attempted
to log all transactions to the database, but its extensive use of stored
procedures proved to be more than we were willing to reverse engineer.

Our next approach was to look at the PHP code to see if we could simply read the
SQL queries that are often contained within. The McAfee developers thought it
necessary to obfuscate all their code by running it through ionCube. This is the
point at which I started swearing. Frequently. Shit was broken, and the
responsible party saw fit to prevent others from fixing it. Few things annoy me,
or I'd guess any other programmer, like that. This is one major reason why I
favour open source.

Failing the PHP approach, we thought back to what Tech Support had said: there
must be a simple plain text XML file that we can edit to solve all our problems!
I'd thought all youthful, naive optimism had left me. I was wrong. Long story
short: XML files exist, but the one with the name of the report didn't appear to
have any control over image generation. Bugger. Drop curtain.

New scene, Noon today. Ron and I once again sitting at my desk, hacking away at
our SQL query from the day before. Interestingly, our numbers, though still off
by a factor of ten, made even less sense when compared to the reference graphs.
It was then we gave up on solving our problem the Right Way, and embraced the
madness.

Our first idea, generating the report automatically and having it emailed to our
manual-labour-grunt-zombie (read: the next co-op student to come around that
isn't me). Upon receiving it, our victim would then log into Foundstone, and
request the live representation of the data to be generated. They would then
download and rename each image that was missing from the report. As a
penultimate step, the victim would then unzip the file, integrate the graphs
into the (several levels deep) directory structure, and zip it up again.
Finally, they would email the report out to the people that *actually
matter*. Although we were amused by this plan, so long as we weren't the ones
going through these steps, it just didn't have that satisfyingly inelegant
flavour to it.

Our next few ideas involved everything from writing a series of chained
Greasemonkey scripts, to writing an entire Firefox extension from scratch. Then,
we came up with our Final Solution: manually (read: use Perl and LWP) run all
the HTTP requests ourselves. Since the Web interface uses cookies as a session
token and doesn't bother with SSL (why on Earth would an enterprise-grade
security suite *not* conduct all it's activity in plain text?) it should be
possible.

For the rest of the day, Ron and I programmed a two page Perl script that logs
into Foundstone, presses all the right buttons, rips out all the resulting
images, downloads the report's ZIP, and overwrites the bad graphs. I
appropriately named our script 'wrongness.pl'. There's still more to do on our
script, since they also want the report to be emailed out on completion, and a
few other things. The important things that this entire experience taught me
were: Ron is awesome (despite being a Vegan, Slackware user, and VIM lover), and
pair programming works no matter what the project is. I 'drove' all day, which is
to say that I was the one [windmilling][] at the keyboard, while he 'navigated'.
This sounds like a waste of manpower, but I assure you it isn't. While I was
typing, he was watching that I didn't make any spelling or logic errors. Ron was
also thinking ahead on how to solve the next issue, while I was busy
implementing our solution to the current one. Any time we disagreed on an
implementation issue, we'd simply argue until we came to a resolution, which
usually took about a minute.

The most enjoyable part of this pair programming experience, other than creating
a dirty solution to an ugly problem that works perfectly, was the chance to
spend an entire day talking, arguing, and insulting someone. Had I been told
that I could spend that much time ranting and questioning someone's breeding,
upbringing, sexual habits, hygiene, intelligence, attractiveness, creativity,
choice of editor, and just general life choices while still being productive,
I'd never have believed it. Admittedly, the amount of code we produced was
meager, but it uses libraries we had no experience with (LWP and Archive::Zip)
in a language that we rarely use. The most important thing, though, is that I
have *never* produced Perl code that nice. I look forward to pair programming as
much as possible in future, providing my partner isn't useless. That'd just be
no fun! Yes, I feel the need to counterbalance my optimism with the bitter,
jaded cynicism of experience.

[FSH]: https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard
[windmilling]: /img/windmilling.gif
