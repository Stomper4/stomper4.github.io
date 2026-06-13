---
layout: distill
title: Another Firefox Vulnerability
description: The most creative way to kill my laptop battery.
tags: Finding
date: 2026-06-12
featured: false
pretty_table: true
related_posts: false
d-appendix: false

toc:
  - name: Background
  - name: Testing Setup
  - name: Vulnerability Discovery
  - name: Impact
  - name: What is Firefox supposed to do?
  - name: Disclosure
  - name: Lessons
  - name: Links

# Below is an example of injecting additional post-specific styles.
# If you use this post as a template, delete this _styles block.
_styles: >
  .table td {
  border-left: 1px solid var(--global-divider-color);
  border-right: 1px solid var(--global-divider-color);
  }
  .table th {
  border-left: 1px solid var(--global-divider-color);
  border-right: 1px solid var(--global-divider-color);
  }
  d-appendix {
  display: none !important;
  }
  pre, code {
  white-space: pre-wrap !important;
  word-wrap: break-word !important;
  word-break: break-word !important;
  }
  pre.red-code, pre.red-code code, pre.red-code code * {
  color: red !important;
  }
  figcaption {
  display: block;
  text-align: center;
  font-size: 1.0rem;
  font-weight: medium;
  color: #000000;
  margin-top: 0 rem;
  background-color: #adacac;
  }
  figure {
  margin-top: 0;
  }

---

If you don't think browser vulnerabilities are cool I don't know what kind of hook I could put here to make you keep reading. 

## Background

Late 2025 Elysee Franchuk and I took on a project to perform security testing of service worker implementation in Firefox. This came about as part of a broader goal to get a CVE in a browser. 

A <a href="https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API">Service Worker</a> runs as a background process in your browser, separate from the content process that renders your web page. It is able to cache content, and intercept requests to speed up web page performance. It is also capable of running with no tab open to sync data or receive notifications after the tab is closed. 

Service workers have more capability than the document (or web page) rendered in your tab. They are written to your disk, they persist across sessions, and can run in the background. Service workers and web  pages (content processes) run in sub processes of Firefox, but we found that the initial import of a service worker is handled by Firefox's parent process. 

Firefox, like other major browsers, implements a multi-process design as a security feature, to keep untrusted content away from the execution and memory of the parent process. Any time that we can get our input handled by the parent process deserves special attention. 

## Testing Setup

I hosted a local Apache server and had AI draft a few test pages to get started. I used them to start experimenting with registering, triggering, updating, and removing service workers. The rest of the pages were made on the fly to test certain features.

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox7.png">

Note that TLS is required to register service workers unless from a local server. For resource exhaustion tests I recommend using multiple machines and separating the client and server so they don't compete with each other for resources.

## Vulnerability Discovery
My <a href="/blog/2026/FF-crash/">last post</a>  is a root cause analysis of a vulnerability we found while boundary testing the registration function.

While doing that analysis I came across some key facts which helped direct me to discover a new vulnerability in the parent process of Firefox.
1. The service worker is imported by the parent process.
2. It was loaded into memory immediately, regardless of size, and parsed front to back repeatedly by different functions.
3. The service worker import functions appear to use a *lot* of computer resources. (Keep in mind we are using larger service workers than anyone reasonable would ever code!)
4. There were no additional security checks to determine whether or not the service worker should be accepted before passing it to the string handling functions.

This caused me to wonder how much of the computers resources could I use if I registered as many large service workers as possible. (spoiler: all of them)

I created several different service workers of different sizes programmatically so that I didn't have to open and read files over a GB in size. An example of the commands is below:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1000000000 > pattern.txt
truncate -s -1 pattern.txt
echo -n \" >> pattern.txt
(echo -n "const largepayload =\"";cat pattern.txt) > payload.txt
(cat swbase.js;cat payload.txt) > medium.js
```

The following command will show the final worker:
```bash
head -c 500 medium.js;echo -n " ... ";tail -c 50 medium.js
```
<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox8.png">

The inflated size simply came from a large constant variable. This ensures that it is valid JavaScript, must be parsed, and won't be removed from the file.

The next step is simply to create an HTML file which registers multiple workers in a row with unique scopes.  Partial JavaScript shown below:

```javascript
for (let i = 0; i < 500; i++) {
  try {
    const workerId = randomId();
    // Register medium.js with a unique scope for each instance
    navigator.serviceWorker.register('/medium.js',{scope:`/sw/${workerId}/`});
  } catch (err) {
    console.error('Error during registration:', err);
  }
}
```
(worlds most annoying XSS escalation anyone?)

I found that the service workers are all added to a single queue, and the parent processes greedily uses all resources it can get until they are imported. The attacker can control the size of service worker as well as the rate of import to exhaust an intended amount of resources. Closing Firefox does not end this process, it will attempt to continue importing service workers for several minutes or until the process is killed. 

The following images show results of different test cases against a Windows 11 VM with 16GB of RAM and 4 logical processes.

Setting up the test page to import one service worker (around 1GB) allowed me to see the shape of memory usage for this process:

<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox9.png" alt="image">
  <figcaption><b>Figure:</b> Memory usage graphed in task manager during 1 service worker import</figcaption>
</figure>

Then I tried to import a bunch!

<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox10.png" alt="image">
  <figcaption><b>Figure:</b> The CPU spikes when the process runs out of memory to allocate</figcaption>
</figure>

You can see from the next image that the memory and CPU usage spike faster if larger service worker files are used:

<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox11.png" alt="image">
  <figcaption><b>Figure:</b> Memory usage spiking during an optimized test</figcaption>
</figure>

Reducing the rate and/or size of service workers to throttle this attack causes a cycle of memory allocation and release:

<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox12.png" alt="image">
  <figcaption><b>Figure:</b> Memory usage persisting cyclically during a reduced test</figcaption>
</figure>

## Impact

This is a resource exhaustion attack that an adversary can tune. It can be made to crash the Firefox browser, or run for an extended amount of time and cause adverse effects to other processes on the same machine.

To demonstrate impact across processes, I opened 2 tabs in Firefox, one to simulate a malicious page and one to simulate a normal page. I reduced the test case on my malicious website to allow Firefox to continue running. The browser soon started to lag and not respond but would not crash. In the normal tab I attempted to open developer tools. This action causes ASAN logs to print for the non-responsive tab. The following image shows logs from several minutes of unresponsive behaviour. 

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox13.png">

On linux, closing and restarting Firefox is not effective, to end this attack the process needs to be killed or the computer restarted. The following image is shown when attempting to restart the browser. 

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox14.png">

Note: Eventually the process will be killed without user interaction, this error occurs if a user tries to open Firefox before that occurs. 

Exhaustion attacks can be non-deterministic, meaning the impacted system does not always react the same way. For example, once during this attack Firefox's user interface crashed but every other process kept going. The following is an excerpt from the asan logs:

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox15.png">

In my testing I crashed the Ubuntu and Windows operating systems (once each) and caused other apps to stop responding, demonstrating additional cross process impact. Crashing a different application should be theoretically impossible, the operating system should kill Firefox before that. I witnessed it a couple times, but was not able to tune an attack that would cause it deterministically (every time). 

I was able to deterministically crash Firefox itself. Once you run out of memory the CPU spikes, these images are from real time monitoring on my Ubuntu host machine. 

<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox16.png">
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox16.5.png" alt="image">
  <figcaption><b>Figure:</b> Usage of 12 CPU processes, 32GB memory, and 8GB swap</figcaption>
</figure>

Then something crashes! 

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox17.png">

<br>
The following image shows a larger excerpt from operating system logs, showing the memory usage at the time the operating system forcibly killed Firefox.

<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox18.png" alt="image">
  <figcaption><b>Figure:</b> The highlighted line shows anon-rss (the memory used by the process at the time of the kill) was over 25GB.  More than a normal user has to spare to a browser exploit.</figcaption>
</figure>

This impact is significantly higher than a denial of service in the JavaScript engine or DOM. Firefox uses a <a href="https://blog.mozilla.org/en/firefox/expanded-multi-process-desktop-android-updates/">multi-process arcitecture</a> which separates each process that is meant to execute or render untrusted content from the internet. This allows Firefox to gracefully handle large, slow, and annoying websites. 

Even if you code a webpage to recursively allocate as much memory as possible, the browser will carefully terminate it's own subprocess before anything crashes or other tabs are affected. 


```html
<!DOCTYPE html>
<html>
<head><title>Content Process OOM Test</title></head>
<body>
<h2>Content Process OOM Test</h2>
<p>Click the button. This tab's content process will OOM and crash. The browser should survive.</p>
<button onclick="triggerOOM()">Trigger OOM</button>
<script>
function triggerOOM() {
  const chunks = [];
  try {
    while (true) {
      // Allocate memory indefinately
      chunks.push(new ArrayBuffer(256 * 1024 * 1024));
    }
  } catch(e) {
    // View the error in the browser
    document.body.innerHTML += "<p>Caught Error: \"" + e + "\" </p>";
  }
}
</script>
</body>
</html>
```

Firefox handles this reliably enough that you don't need any tricks like opening dev tools to create logs, you can just append them them to the document and view the error from the page. 

<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox19.png" alt="image">
  <figcaption><b>Figure:</b> An out of memory error served to the JavaScript engine by process management before affecting the local system</figcaption>
</figure>

The fundamental difference between these 2 attacks is which process is targeted. The browser has extra controls over the content process so that a malicious website (or a major ecommerce retailer, take your pick) can only DOS itself. When we target the parent process through service worker registration there are no additional protections. The CVSS for this vulnerability is **8.6, AV:N/AC:L/PR:N/UI:N/S:C/C:N/I:N/A:H**. 

## What is Firefox supposed to do?
*Short answer: I don't know?*

Check out the <a href="https://www.w3.org/TR/service-workers/#service-worker-registration-concept">official specification for the Service workers API</a>. (Don't rat me out for <a href="https://www.w3.org/standards/types/#CRD">calling it a standard</a>.) It has no suggestions on how to prevent this attack. I think this leaves the Mozilla team in a little bit of a pickle because how can they know what a developer is assuming? Would adding a protection that is not in the specification disrupt a real use case?

<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox20.png" alt="image">
  <figcaption><b>Figure:</b> Comment from Mozilla team on the bugzilla report.</figcaption>
</figure>

I have tried this exact attack against chromium browsers with no success, but haven't investigated why it doesn't work (or whether I could make it work).  I guess the only suggestion is to do whatever Chrome did? 

Actually the fix I have advocated for is to rate limit the import of new service workers. This exploit depends entirely on resources not being returned to the operating system between registrations of service workers for a given domain. I don't think you would need to prevent a certain number or size of worker for a domain (which could negatively impact the functionality of a web page). A few second delay between registrations should be enough. Or a rate limit by size, say 10 seconds after 1GB of service worker registrations. *I desperately hope there aren't devs writing websites that would break.* What I don't know is whether this is the most annoying potential fix to code? 

We also noted that if you register 10s of thousands of service workers Firefox loses track of them. Some but not all workers can be seen and managed in `about:serviceworkers`. This appears to not follow the specification requirement "A user agent _must_ persistently keep a list of registered service worker registrations unless otherwise they are explicitly unregistered" but I am not sure if this failing is in the backend or the UI. You can still delete all workers by clearing site data. It does not appear to be exploitable so we did not report it. The Mozilla team came across the same behaviour in their own testing after we submitted these vulnerabilities.

## Disclosure

This vulnerability was disclosed in November 2025. I also submitted this vulnerability for bounty consideration under the following criteria:
1. Denial of Service issues that merely crash the browser are not eligible for a bounty.
2. Persistent-DOS of browser across restarts or a DOS requiring reboot of user’s computer typically pay $1000 at the discretion of the committee.

I reasoned that this vulnerability was somewhere in the middle, it wasn't exactly persistent but could have wider impact than a browser crash. The Mozilla team decided that it was not eligible, but clarified their requirements and allowed me to attempt to escalate the vulnerability to meet them. 

The conversation was open and positive. Despite the lack of payment this is one of the best interactions I have had with security teams or bug bounty triagers after submitting a vulnerability. I exchanged multiple emails with a member of the Mozilla team and was able to share all the impact quirks I had found during my testing. They then brought it up before the committee, represented my side, and there was a vote about whether or not to pay a bounty. I would encourage other researchers to find mature disclosure programs that don't use bug bounty platforms if they, like me, care more about avoiding a headache than earning a paycheck. This is a hobby after all. 

The Mozilla team set this issue aside as a potential duplicate until the patch for CVE-2026-0889 came out, to see if they if it would reduce the impact. It did not. They re-opened my submission as an accepted but low priority vulnerability in January 2026. Since then, they have been responsive and have discussed potential fixes, but have not yet patched. Anthropic submitted a large number of bug reports found by Claude earlier this year so I expect the team at Mozilla has been quite busy. I also don't know what <a href="https://github.com/v12-security/pocs/tree/main/firefox">other vulnerabilities</a> have been submitted which may have a higher priority. I don't disagree with their assessment, I think this is more interesting than it is likely to be weaponized.

I gave the Mozilla team a courtesy notice on May 10th that I intended to share some details of this vulnerability at BSides Calgary 2026.

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox21.png">

Their response was quite polite and open to this.

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox22.png">

I also emailed a draft version of this article to the Mozilla security team a couple weeks before publishing and they did not respond with any concerns.

## Lessons

If anyone else is hunting against Firefox they shared this helpful tidbit:
<figure>
  <img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox23.png" alt="image">
  <figcaption><b>Figure:</b> <a href="https://bugzilla.mozilla.org/show_bug.cgi?id=1999084#c19">Comment from Mozilla team</a></figcaption>
</figure>

The bar for a CVE appears clear, does this qualify? I guess only if they fix it. I found that browser crashes were easy enough to duplicate I would say it qualifies under that criteria alone, regardless of the other impacts. 

**Fun facts about CVE-2026-0889:** CISA added the weakness CWE-400: *uncontrollable resource consumption* to the CVE record. This is factually incorrect about the CVE yet amusingly accurate as an arbitrarily predicted weakness in this component. (Lets not talk about how that CWE is <a href="https://cwe.mitre.org/data/definitions/400.html#Vulnerability_Mapping_Notes_400">discouraged from use mapping to real world vulnerabilities</a>)

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox24.png">

This has lead to many articles about the CVE which host entirely made up content.

<img class="img-fluid z-dept-1 rounded shadow" src="/assets/img/Firefox25.png">

It seems we have a vulnerability database to help defenders keep track of new risks... yet a significant amount of content around it cannot be trusted. Maybe one day Google will index my blog.

Also, congrats to Mozilla for <a href="https://www.linkedin.com/posts/anthonyed_pwn2own-ugcPost-7461133019907760128-Isez/?utm_source=share&utm_medium=member_desktop&rcm=ACoAADQnBusBRsJa9tg-84o7f1LFjVaifVEfGRQ">a great performance at Pwn2Own Berlin 2026</a>!


## Links

- <a href=" https://cwe.mitre.org/data/definitions/770.html">CWE 770: Allocation of Resources Without Limits or Throttling</a>
- <a href="https://bugzilla.mozilla.org/show_bug.cgi?id=1999578">Bugzilla Vulnerability Submission</a> (Likely won't be public until patched)