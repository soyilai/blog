---
title: I Hacked My School
date: 16-06-2026
description: The Great Grade Heist of 2026
---

> "We changed the password, we're secure now!"
> -- Said no sysadmin ever

<caution>
Look, I'm putting this out there because the infosec community needs to laugh at how badly this was set up. But let's be real: what I did is illegal in every country on Earth. We're talking fines, criminal charges, jail time, the whole nine yards. This isn't a slap on the wrist situation. If you're thinking about doing this to your school, don't. I got incredibly lucky. You probably won't.
</caution>

## <pending header>

When I first enrolled at this school, I was impressed.

That probably sounds ridiculous to anyone from the United States or Europe, but here on Mexico, this was the most technologically advanced educational institution I had ever seen.

They had fiber internet. Multiple segmented Wi-Fi networks. Google Workspace accounts for every student. A custom student management portal.

The whole thing looked surprisingly professional. Which, naturally, made me want to take it apart.

The portal itself looked like it had survived several presidential administrations. It wasn't just old, it was *interesting* old.

One of the first things I noticed was the login form:
```html
<input name="UserName" type="password" id="UserName" class="password">
```

Yes.

The username field was a password field.

Not the password field.

The username field.

I'm not exaggerating when I say this detail drove me insane. It broke browser autocomplete, it made zero functional sense, and it was so fundamentally wrong that my autism couldn't unsee it.

Then I noticed the two domains hosting the system. Neither had SSL certificates. Not one. In a country where criminals know more cybersecurity than government officials, they're serving login credentials over plain HTTP.

At that point I wasn't even looking for vulnerabilities anymore, they were just kind of introducing themselves.

## Is that .NET I smell?

First, an `nmap`
```
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS/10.0
3306/tcp  open  mysql        MySQL 5.5.22
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Microsoft Windows Server (workstation)
```

MySQL 5.5.22 is so ancient it remembers when buffer overflows and `memcpy()` bugs were a personality trait, not a vulnerability class.

Wappalyzer revealed the full tech stack: ASP.NET WebForms powering the student/teacher portals, CodeIgniter with PHP 8.1.10 running the main website. The ASPX pages use MySQL on the backend, not MSSQL, which is... certainly a choice.

I was poking around when I found something:
```bash
$ curl -sI http://www.notleakingthis.edu.mx/application/config/database.php
HTTP/1.1 200 OK
```

Bingo! CodeIgniter's database configuration file existed and was accessible through the web server.

The framework's protection prevented its contents from being displayed, but the fact that the file was reachable at all suggested the deployment had not been configured particularly carefully.

Then I found /mysql/, which returned `403`, probably my favorite HTTP response since it literally says "There is absolutely something here and I don't want you looking at it." Naturally, I spent several hours trying to look at it. I never managed to enumerate anything useful.


I researched a while and discovered that this version of MySQL posses [`CVE-2012-2122`](https://nvd.nist.gov/vuln/detail/CVE-2012-2122).

An infamous authentication bypass bug. It's just hilarious. You essentially keep trying to log in until MySQL gets mad at you and just lets you in.

I got my hands dirty, and wrote a small script that fires a ton of requests until it gets access, which should be around the ~257 attempts... but I reached the 500 attempts and it didn't work. 

SMB was next. Completely locked down. No anonymous access. My ADHD told me to gave up and I went to sleep.

## The most dangerous character in web history

The next morning I returned with a fresh perspective.

And by "fresh perspective" I mean I started throwing apostrophes at the login form.

For decades, entire industries have been brought to their knees by a single character: `'`

Apostrophes have destroyed databases, exposed secrets, and ruined weekends for developers across the globe.

Maybe this portal would be no different.
```bash
$ curl -s "http://.../maestros/login.aspx" \
--data "UserName='&Password=test&entrar=Entrar"
```

The result? `error500.aspx`... Interesting.

So I tried something slightly more offensive: `' OR 1=1 --`

Effectively, it worked and bypassed the password check. Lots of funsies!

Now, let's try the inverse: `' OR 1=2 --`.

We're now back to the error page. Now things become extremely interesting... the application can answer yes-or-no questions.

I had accidentally built an oracle. And once you have an oracle, databases become *surprisingly talkative*.

## Building the oracle

A faster way would be error-based extraction, but it wasn't possible. The application swallowed every exception and redirected users to a generic error page.

Annoying.

Instead, I weaponized the boolean logic with MySQL's `IF()` function:
```sql
OR (SELECT IF((condition), 1, (SELECT 1 UNION SELECT 2))) -- 
```
- When **TRUE:** `IF` returns `1` -> subquery returns 1 row -> login page loads normally
- When **FALSE:** `IF` runs `SELECT 1 UNION SELECT 2` -> subquery returns 2 rows -> crash

Now we can ask the database!

First discovery: the application was running as the MySQL user `college_collegenotleakingthis@%`. Now I had a real username to work with.

I tried the ancient CVE-2012-2122 exploit again with this username.

...

Well, it didn't work. I doubled checked my script, the versions... but the server simply didn't bulge.

Then I remembered something just embarrassing. I already had database access! I had spent hours trying to enter through the front door after accidentally finding an open window.

So I just started extracting data instead.

## Binary Search comes in clutch

Dumping data one character at a time sounds awful because it is. But binary search makes it slighly faster, not slightly less awful, just faster.

Instead of testing every possible character, I could cut the search space in half repeatedly.

Seven requests per character, a few hundred requests per record, thousands upon thousands for everything else.

```python
 for position in range(1, length+1):
    lo, hi = 32, 126
    while lo < hi:
         mid = (lo + hi) // 2
        if oracle(f"ASCII(SUBSTRING(({query}),{position},1))>{mid}"):
            lo = mid + 1
        else:
           hi = mid
   character = chr(lo)  # Got one!
```

Eventually the database began revealing its contents. I exfiltrated:
- Teacher records.
- Student records (most of them minors!).
- Personal information.
- Phone numbers.
- Government identifiers (including citizenship things and social security numbers).
- Physical Addresses.
- Credentials.

And apparently hashing wasn't a thing back then:

- Plain-text passwords.

Every time I thought I had found the worst part, another table appeared, so for my mental sanity I decided to leave it there.

## The great grade heist

At this point there was only one question left.

Could grades be modified? The answer is yes.

Finding the correct records took much longer than changing them, though. Grades weren't attached directly to classes, they were attached to evaluations.

Another 5 minutes researching evaluations, and I discovered I needed to know the class too.

Another 10 minutes researching classes, and I discovered I needed to know the subject too.

Another 15 minutes researching subjects, and I discovered- Oh, well, here it ends. I finally located the record I needed.

The actual modification took way less than a second.

I spent hours of reconnaissance, hours of understanding the schema for just one query and about 150 miliseconds.

Done.


## Blue team strikes back

Several hundred requests later, the website suddenly died.

`mysqli_sql_exception: Access denied for user 'college_collegenotleakingthis'@'localhost' (using password: YES)`

For a moment I thought someone had noticed... then I remembered where I was.

This wasn't a Fortune 500 company. This was your standard mexican educational infrastructure.

The most likely explanation was that something broke on its own... or not.. or maybe.

### The Waiting Game

~2 days later:
```bash
$ curl -sI http://www.notleakingthis.edu.mx/
HTTP/1.1 200 OK
```
 
She's back. Test the SQLi:
```bash
$ ./blind_sqli.sh "' OR (SELECT IF((1=1), 1, (SELECT 1 UNION SELECT 2))) -- "
OK
$ ./blind_sqli.sh "' OR (SELECT IF((1=2), 1, (SELECT 1 UNION SELECT 2))) -- "
ERROR
```
**They didn't patch the hole.**

## The Cover Up

By the time I'd done all this, grades had already been printed and published. My modified grade made it through. But I'd left *traces*: test queries, accidental data changes from when I was learning how the schema worked.

Modifying things mid-grading period was low-risk (changes to grades during evaluation windows aren't inherently suspicious). But now that grades were finalized and public, any new modifications would stand out.

So I buried the evidence. I queried random tables, updated random student records, threw SQL noise into the logs to make the trail harder to follow. It wasn't perfect obfuscation, just enough clutter that it would take at least some basic skill to reconstruct what actually happened.

This was just modifying some other rows, querying some more things, and in the middle of everything changing the artifact.

## So... What Now?

I don't usually do red team work. This wasn't some grand ethical hacking mission either, it was just a kid with a bad grade, autism-driven rage over broken HTML, and too much free time. The sysadmin, whoever they are, had no malice. They just didn't know what they didn't know.

The real takeaway: your school's infrastructure is probably held together with duct tape and prayers. So is everyone else's. The only difference between a vulnerability and a feature is whether someone's paying attention.

I'm publishing this because infosec is supposed to be about learning from failures, not laughing behind closed doors. And because the person responsible for this mess deserves to know exactly how bad it was.

Somewhere out there, a system administrator probably still believes the biggest issue was the password.

Stay curious. Don't be reckless. And for God's sake, use `type="text"` on username fields.

<important>
I was never caught. This writeup is published months after the fact. Please don't use this as a blueprint for your own school. The legal consequences are real. I got lucky.
</important>

<note>
I've anonymized the school, domain names, and identifying details. You don't need to know who they are to understand what went wrong.
</note>
