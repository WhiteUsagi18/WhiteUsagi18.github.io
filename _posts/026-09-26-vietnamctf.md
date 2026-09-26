---
title: What on earth is happening to us in Da Nang??
date: 2026-09-26 00:00:00 +0000
categories: [Blog, CTF]
tags: [storyblog, capturetheflag, vietnam]     # TAG names should always be lowercase
image: /assets/img/vietnamctf/image%201.jpg
---

Recently, my team puasa6, got a chance to be one of the teams representing Malaysia in Vietnam and it was the first time for four of us to travel overseas. We managed to qualify for the finals "luckily" because we encountered a network issue at the hostel during the prelim round which almost cost us our qualification. Eventually, there was one more team from our college that qualified for the finals making it 2 teams from Malaysia.

Even though we were very excited for this CTF, we actually faced many issues from the organizer, the infrastructure, the players, and how the event was managed.

## What happened in the prelim round?
I will start by addressing the issues from the prelim round. During this round, the organizer promised 15 challenges in total based on their official website. But because the challenges were too "easy" for AI to solve, the organizer decided to add one more hard challenge without any announcement. This left us not only questioning why the organizer was silent about the new challenge, but also why the rules had suddenly changed.

![official website 15 challenge](/assets/img/vietnamctf/image%202.png)

Because of the last challenge is too hard and guessy to solve and it waste of time, many of participants are not satisfied with the challenge.

Original:

![participants not satisfied](/assets/img/vietnamctf/image%203.png)

Translated:

![participants not satisfied translate](/assets/img/vietnamctf/image%204.png)

Despite how bad the challenge design was, I kinda agree with the admin's reasoning.

Original:

![Admin reason](/assets/img/vietnamctf/image%206.png)

Translated:

![Admin reason translated](/assets/img/vietnamctf/image%207.png)

Fair reason, but the challenge is still guessy af and kinda nonsense. Not to mention how bad the infrastructure was (ignore my friend kecek kelate to the Vietnamese XD):

![participants not satisfied translate](/assets/img/vietnamctf/image%205.png)

The poor infra and rules that suddenly changed caused a little drama in the discord.

![bad bingo ctf](/assets/img/vietnamctf/image%208.png)

Seeing many Vietnamese condemning this CTF made me have a bad feeling about how the final was gonna be.

## THE FINALLSSS!!
Now, the finals had come and we had prepared ourselves mentally to compete. Especially me, since I often heard that Vietnamese players are really skilled in hacking and CTF, and their country's rank is even in the top 15 on HTB. Damnnnn, if we could get on the podium or at least in the top 10, we actually almost on the same level as them! I really really really really want to see how much I have improved! Everything seemed under our control until we got there one day before the competition, where they asked us to do a rehearsal to check if the network and setup were functioning or not.

![Final Pic](/assets/img/vietnamctf/image%209.jpg)

![Tentative](/assets/img/vietnamctf/image%2010.png)

Before we continue, I would like to clarify the rules that we got before arrived in Vietnam:
```
Dear team leaders,

Greetings from the DDC 2026 Organizing Committee! As we are approaching the DDC 2026 Final round, we would like to share some important information and preparation requirements with all participating teams.

1. Technical preparation
Use of AI tools is prohibited during the competition. Teams are not allowed to use any AI-powered tools or services to assist with solving challenges during the Final round.

Each team member is required to bring their own laptop for the competition.

Please make sure your laptop has an Ethernet/LAN port or bring a suitable LAN-to-USB adapter, as a wired network connection will be used during the competition.

2. Media materials
To support the Organizing Committee in preparing promotional materials for the Final round, we kindly ask each team to submit:

One team photo showing all team members. Please wear neat and appropriate attire. (Teams are encouraged to wear their university uniforms if available)

One high-quality logo file of your university.

These materials will be used by the Organizing Committee to design posters and other communication materials for the DDC 2026 Final round. Please submit the requested photo and logo no later than September 11, 2026.

3. Communication channels
To ensure timely communication and updates from the Organizing Committee, we kindly ask all team leaders to join the appropriate communication group:

Vietnamese teams: Please join the Zalo group via the following link: [Zalo Group Link]

International teams: Please join the WhatsApp group via the following link: [WhatsApp Group Link]

Please make sure that all the team leaders join the relevant group so that we can promptly share important announcements and updates regarding the Final round.

We greatly appreciate your cooperation and support in preparing for the event. We look forward to welcoming all teams to VKU for the DDC 2026 Final round!



Best regards,
DDC 2026 Organizing Committee
Vietnam-Korea University of Information and Communication Technology (VKU)
```

So as stated, we can't use any AI tools during the finals including web/chat based AI, and it seems we need to use a LAN cable to access the CTFd platform locally. Yeah, its completely normal and predictable for CTF competitions these days to prevent players from "AI slopping" all the challenges. Also, theres nothing stated in the rules that internet is not allowed. Again, I mentioned theres no rule stating that internet is not allowed. So, I thought that they would probably set up a proxy and block AI domains like in the UMCS Final 2026. Okay, nothing weird and we could still get information from the internet to solve the challenges.

Tapi realiti tidak seindah itu...

When we got into the room where the rehearsal started, the LAN cable was not functioning and we saw that their technician was still troubleshooting the network. We asked for help and what we got was just to wait until a further update. Then suddenly in the evening, we got a new update to the rules that said we couldn't use the internet during the competition but they would provide a "local AI" on their network so we could access their AI later.

Original:

![local AI](/assets/img/vietnamctf/image%2014.png)

Translated:

![local ai translated](/assets/img/vietnamctf/image%2015.png)

You know what came to my mind when I heard that? They probably had an issue with their network setup so their solution was to not allow participants to use the internet at all and they just wanted us as a dataset to train their AI XD. Like, isnt it obvious? From "we can't use any AI tools to we can use their local AI". Haha what a funny joke. This is just my assumption, the real reason we still don't know but it still sucks to suddenly change the rules lol.

Alright, no internet but we got their AI to use. But the "local AI" always be our question that night. Can the "local AI" support being used by all teams? I mean, we had seen how bad the infrastructure was in the prelim round sooooo....

Nevermind, we don't want to gamble with this "local AI" shit so we prepared for a worst case where the "local AI" will down during the competition and discussed our strategy again. My strategy to solve web challenges is kinda simple: download every cheat sheet and documentation, including PayloadsAllTheThings, HackTricks, and other docs for source code review in various languages. Make my laptop like a wikipedia for me to refer to for information. Also, I prepare for an automation script to extract data in case the challenge is blind. Other than that, just trust in my experience identify vulnerability and source code review through training from hundreds of hours I have invested.

Another thing that need to clarify is their official website say there will be 15 challenges across 5 categories. Again, I mentioned only 15 challenges!

![Final 15 challenges](/assets/img/vietnamctf/image%2011.png)

Now all the preparation has finalized and its time to test my limits competing with the Vietnamese students and see how my skills have gone so far. LETSS GOOOO!!!!

As usual for the opening ceremony, they will yap about their sponsors, what country that going to finals, the teams, and the RULES.

![Final Round Rules](/assets/img/vietnamctf/image%2013.jpeg)

Normal rules, but I want to highlight for the first one where we can't use a mobile phones during the competition. Also, there is a new update for the category where they add 2 more categories, Forensics and Hardware. Adehhhhh why they love to announce last minute eh?

Straight to when the competition start:

![Pray](/assets/img/vietnamctf/image%2016.jpg)

In the first 2 hours, puasa6 reached the top 10 and we were trying to maintain our position. Mind you, we started 30 minutes late after the competition began because of a technical issue with the LAN cable and their "local AI". Yeah, we could not access their AI from the start. It seems that their server could not handle too many requests hahaha. After the organizer noticed that there were teams that could not access the AI, they took down their "local AI" on the spot. Haha, like what we guessed is correct. But its still crazy that we could reach almost into the top 5 without using AI and internet.

Now all teams have same condition to solve the challenges, NO INTERNET NO AI.

But after a while of us trying to push, our position suddenly dropped to the bottom. At that time I was suspicious because in the first 2 hours we could climb the scoreboard, so why did we suddenly drop? So I tried to inspect the "solved teams" for the challenge that I struggled with. I was shocked to discover that the timestamps for every team were really close together. Imagine the first blood on the challenge is at 10:30 AM, then the next team solves at 11:09 AM, then continuing at 11:15 AM, 11:18 AM, 11:23 AM, 11:26 AM, and even 2 teams that solved at the same time 11:29 AM! The actual timestamps are not correct because Im just trying to remember them, but the time between their solves was really short. I even asked Aliff to see that this kind of activity was suspicious. But I still think positive that they actually playing clean heh.

If you are curious on how the difficult the challenges are, I put it here the list of attack path that I remember for each web challenge (I could be wrong because there are challenges that I didn't have enough time to solve and just doing recon on the surface):

1. BAC in GraphQL Request
2. Protoype Pollution
3. Bypassing Middleware/Proxy Header checking
4. Blind PostgreSQL Injection with blacklisted character
5. SSRF with blacklisted character
6. SSTI to RCE with blacklist class and object
7. CORS misconfiguration

There are more than this, and I don't know what level of difficulty for this kind of challenge but definitely not for beginner lol. Important to remember that all web challenges are BLACKBOX!! NO SOURCE CODE REVIEW

Also, the total of the challenges is more than 15!!!!! Wtf is going on with this CTF??

### How we discovered the cheat
This one is kinda funny how we actually found this. One of my teammates was trying to solve the hardware challenge and the description said to scan the WiFi or something. So he assumed that we needed to use Wireshark for this and capture the network traffic in the room. Then suddenly we found that there were many packets coming from AI domains like claude and chatgpt HAHAHA. Actually, if we took a look at the WiFi discovery, there were many open hotspots from other teams haha. We also saw a team behind us using chatgpt, but the examiner was not doing anything lollll. So basically:

1. There were a lot of hotspots open.
2. We saw other teams with AI windows open on their laptops.
3. The captured traffic revealed packets from AI domains.

When we addressed this issue to the organizer, they said that we needed to make a report and not to make assumption that the top teams were cheating (the champion is their host team).

Original:

![generalized](/assets/img/vietnamctf/image%2017.png)

Translated:

![generalized translated](/assets/img/vietnamctf/image%2018.png)

I laughed so hard when I read this, because what kind of "skilled hackers" don't implement anti-forensics? XD

Don't get me wrong, if you win with your own skill then don't feel offended by my words. But if you are cheating, this post is exactly for you.

After the competition ended, I heard from another Malaysian team that they were warned for using their mobile phones during the competition, while other teams could use theirs freely. This also happened to ciko when he tried to text our lecturers about the halal food we needed to receive. Also they complained that all teams in their room are using AI to solve the challenges lmao.

There are more issue that we faced like the language barrier (the volunteers can't speak in english even the event is international level) making the communication difficult and we miss important information + lack of technical support.

## The Closing Ceremony
So that's what happened to us on the competition day. Then we requested clarification from the organizer, or more specifically from the one who handled the infrastructure and challenges. But as usual, dorang pusing selagi boleh. No comment on that sebab mengarut gila.

But not all the Vietnamese players are like this. We were also approached by the local teams and they said its better for us to participate in another CTF like in Ho Chi Minh or Hanoi rather than this one, since this CTF is trash. Meaning, not only are we not the only ones who are unsatisfied with the results and how they played, but the local players feel the same way too.

We also got support from the Vietnamese players on discord:

![vietnam agree 1](/assets/img/vietnamctf/image%2019.png)

![vietnam agree 2](/assets/img/vietnamctf/image%2020.png)

![vietnam agree 3](/assets/img/vietnamctf/image%2022.png)

![vietnam agree 4](/assets/img/vietnamctf/image%2021.png)

![vietnam agree 5](/assets/img/vietnamctf/image%2023.png)

![vietnam agree 6](/assets/img/vietnamctf/image%2024.png)

![vietnam agree 7](/assets/img/vietnamctf/image%2025.png)

What a crazy experience...