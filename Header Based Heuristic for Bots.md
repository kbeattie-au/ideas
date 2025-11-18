# Header-based Heuristic for Bots

A lot of applications these days have some sort of internal JSON API they use. How can those applications detect if it's a browser that's talking to them vs. a desktop client pretending to be a browser? **Silent anomaly detection.**

### Telemetry Mechanism

Use anomaly buckets tied to a user, login session, and point in time. For each type of anomaly, have a separate counter. When an issue is encountered, increment the counter for the correct period, user, login session, and anomaly type. If this is implemented with a relational database table, make sure to use atomic increments.

A max threshold must exist, where if a bucket gets too incremented, no further increments happen for the time/user/type. This helps protect against this being an abuse vector by attackers if they become aware of it.

This bucket approach is superior to a log-all-the-values one, since it prevents users from flooding the logs with garbage values/noise.

**⚠️ Word of Caution:** Login session tokens are useful short-term to distinguish different sessions / devices. That said, a number of systems eventually may purge those, so they're not a good long-term reference / FK candidate.

## An Example Anomaly

Let's say the application has a seemingly pointless custom HTTP header.

**`X-WebClient`** *(but it can be named anything)*

* A random string, not unique to specific users, that we keep track of including creation date.
* This cycles daily or even more frequently.
* Should (not MUST) be present for all v2 API calls.

### The Process

1. On certain intervals OR after logging in, send back JS logic in the layout / master page that updates a "WebClient" value in `localStorage`, adding or replacing any older value. Note:`localStorage` was chosen over `sessionStorage` since it has less irritating behavior for implementors when dealing with incognito / private tabs.
2. When the client makes a call an API endpoint, it uses the value that was read from `localStorage` and sets the **`X-WebClient`** header to it.
3. The server does not typically reject invalid or missing **`X-WebClient`** values. But, this is a silent failure that is tracked.

**🛟 Storage Alternative / Fallback:** Just always include a correct value in a `<script>` tag via a variable in global/window scope and use that. You can also use a different fallback value for that, as it would be useful metric to know how many of our users have broken or locked down `localStorage` for some reason or another.

### Purpose

The silent failures are an "indicator light" that something is up with a client and maybe they're not a normal browser user.

* Our application always sends it, so if someone else never does, it's not coming from the proper application.
* Likewise, if it consistently uses a stale value for a long period of time, that's a flag someone just copied it from another request and didn't care that it changes since it doesn't break anything they can see.

### Using the Telemetry

A staff dashboard that can flag several different problems if they occur frequently enough:

1. Long drift/lag updating the sent **`X-WebClient`** value. If a client holds on to the same **`X-WebClient`** too long, that's a strong indicator somebody hardcoded it.
2. User sessions that don't bother send **`X-WebClient`** values at all. Likely an unauthorized client created by someone who decided the header was "pointless" since it doesn't cause any observable trouble for them to omit it. Could be Postman, could be a desktop app, could be a bot, etc.
3. Very frequent invalid values. Our application doesn't randomize this every request, so if someone is doing that, it's probably not a legitimate browser client.
   **🛸 Edge case:** It's possible some privacy-protecting browser or extension might see a header like this with a random string value and suspect it's tracking-related, so it could interfere on the users behalf to "protect" them. Given that the string value is not user specific, it randomizing the header is not only pointless but disruptive. Don't know how probable this actually is, but considered it at least as a possibility.

## Questions and Answers

**Q:** **How does the outlined example apply to [headless](https://developer.mozilla.org/en-US/docs/Web/WebDriver) browser users (WebDriver)?**
**A:** It doesn't and isn't intended to. [Different heuristics can do that](https://stackoverflow.com/questions/55364643/headless-browser-detection). You may want to detect as an anomaly that since it's unusual for random users to be doing in the first place, but that's beyond the scope of this document.

**Q:** **But isn't that example easy for someone to defeat? Isn't this just security through obscurity?**
**A:** It's a digital tripwire and a heuristic to be combined with others, not an access control mechanism. If you have a map that tells you where all the tripwires in a maze were, you might avoid them easily enough if you're not careless. The important thing is that you don't hand people the map! *Regardless*, these aren't part of the application authorization or authentication code. They don't filter content or prevent injection attacks. They just silently work in the background to track unusual user account behavior.

**Q:** **Why not include IP addresses with the bucket tracking?**
**A:** You can, but it complicates the bucket approach a bit and doesn't add a lot of value. IPs are typically a lower value heuristic thanks to the easy availability of VPNs and botnets.

## Other Mechanisms

Here's some additional anomalies to monitor:

1. **🧭 Sequence Breaking:** Requests that seemingly skipped screens to get where they are (e.g. user logged in, then immediately saved, without first doing a `GET` for an page that should have required the `GET` before the `PUT`/`POST` was made). **You must establish a baseline of normal behavior for each of your website internal APIs to detect this sort of anomaly accurately.**

2. **⛔ Excessive Errors:** Repeated attempts to invoke APIs not available to the user are a red flag someone is poking around trying to find vulnerabilities in authorization code. Outside of a programming mistake in your application, users should not be making frequent requests that result in 403 or 404 errors. Tracking the # of 500s isn't bad either, since it's a good way to detect which users are getting the most errors and having a bad experience in the application.

3. **🎛️ Parameter Fuzzing:** A lot of APIs in web apps are not going to be given incrementing or random identifiers repeatedly for endpoints. This is a red flag someone is trying to find Insecure Indirect Object References (IDOR).

4. **🍯 Decoy Endpoints:** Misleading endpoint URLs in methods that are never triggered. These are a strong indicator someone or something scraped your client code without understanding it. Just make sure comments are present explaining why they exist for *your developers*, and that maybe *you're not serving those comments publically* to users who view source. This is a variation of honeypot endpoints people place in the robots.txt file.

## The Takeaway

You can use heuristics and anomaly buckets to detect people using or abusing your private APIs outside of normal application use.
