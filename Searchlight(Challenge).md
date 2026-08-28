# {{Searchlight}}

**Platform:** TryHackMe
**Date:** {{date:2026-03-17}}
**Difficulty:** {{easy}}
**Category:** {{OSINT}}
**Room URL:** https://tryhackme.com/room/searchlightosint)}}
**Status:**  ✅ Completed

---

## 📌 What Was This Room About?
> This room was about using IMINT/GEOINT(Image intelligence and geospatial intelligence) to geolocate any location in the world with only an image or a video.

---

## 🔍 Reconnaissance
> This entire room was about reconnaissance. The recon tools I used was Google and FFmpeg.

---

## 🧭 Steps I Took

### Step 1 — 
- **What I did:** For the first task all I did was just answer the question of the challenge. The first task was just talking what the room was about and what format to answer each question as.
- **Command/Tool used:**
```bash
# sl{answer here}
```


### Step 2 — 
- **What I did:** For Task 2 I had to complete my first geolocation challenge. This was about using my eyes to determine where the location in the picture was.
- Here is the image I had to use my eyes to find the location of:![[task2_1602089234031.jpg]]
The answer to this question was very easy to solve, the name of the street was Carnaby Street and that was the correct answer.
- **Command/Tool used:**
```bash
# no commands
```
- **What I found:** I found out that I don't have to use advanced hacker tools to find the location of a picture, sometimes I just have to use my good old trusty eyes.
- **Why it matters:** This matters because I don't want to get over-reliant on tools, I want to be able to use my human abilities to solve some problems. Another reason why this matters is because just using my eyes instead of a complicated OSINT tool will save me a lot of time.

### Step 3 — 
- **What I did:** In Task 3 things got a lot more interesting. I had to use Google to find the specific location of tube station in an image.
- Here is the image I had to use google to find its location:![[task3_1602089306375.jpg]]
At a glance this image does not give much info on the answer of the first question which was 'Which city is the tube station located in?'. It would have been easier to just dump this image in google lens and search where it was located in but for some reason I did not. I felt as if I had seen this place before, and as I looked past the tube station and the building behind it, suddenly it clicked, the tube station was located in London. I put sl{london} as the answer and it was correct. 

For the second question I used google this time to solve it. I dragged the image into the search part of google that used Google Lens and I asked which tube station do these stairs lead to, because that was the question I was trying to solve: 'Which tube station do these stairs lead to?'. I got Piccadilly Circus as my answer so I put in sl{piccadilly cirucs} as my answer and got it correct.

Question number 3 was quite easy I just typed which year did this station open into the search bar(Because that was what the question was) and I got 1906. So I inputed sl{1906} as my answer and yet again I got it correct.

Question number 4 was the easiest I just typed the same question as the on in TryHackMe - 'How many platforms are there in this station?' in the search bar and got 4 as the answer to how many platforms there where. So I put in sl{4} as my answer and I go it correct.

- **Command/Tool used:**
```bash
# no commands
```
- **What I found:** I found out that I can use google to do basic OSINT. This is what people call 'dorking' which is the art of using Google search queries to have Google return specific types of data.
- **Why it matters:** This matters because now I can do basic information gathering, and this can help because if I need to do research on a target now I know how to get my information.

### Step 4 — 
- **What I did:** What i did in Task 4 was pretty similar to what I did in Task 3 the only difference is that I had to answer a different set of questions:
- Questions:
	- Which building was this photo taken in?
	- Which country is this building located in?
	- Which city is this building located in?

	All of them where pretty easy to solve, and as I said before where very similar steps to Task 3
Here is the image I was using:
![[task4_1603353588780.jpg]]
- **Command/Tool used:**
```bash
# no commands
```
- **What I found:** Same as I did above in Task 3

### Step 5 — 
- **What I did:** Step 5 was a little different than the other 2 steps above, this one I ran into a small challenge with finding which city a coffee shop in a picture was located in. I tried the same technique above but the overall result was very vague and not helping me narrow down that city. So I used the greatest tool in my tool-belt > My Eyes. I looked at it a while before realising that in the image there was a shop that was right across the coffee shop, and i was able to see the shop's name it was 'The Edinburgh Woollen Mill' and It dawned on me that if I find the location of that shop I find where that coffee shop is. So I went straight to work trying to find that shop's location I found many dud locations, apperentaly that shop is located in many places around the world so I had to specifiy a specific regean that I had to look. I narrowed that reagon to Scotland since the challenge said that the location of the shop was in Scotland. So I searched it and I got the city of Blairgowrie, by the time I got this I had tried over 10 cities and I was hoping that this one was it, so I put sl{blairgowrie} in the text field and pressed enter and it was correct. 

- That was question number one now it was time for question number 2 to find 'Which street is this coffee shop located in'. Thankfully that was much easier since i had found that Woollen Mill the name of the coffee shop showed up its name was The Wee Coffee Shop, so I just typed for its location and I got Allan Street and that was what I put in as my answer: sl{allan street} and got correct. As for what their phone number, email address, and surname of the owners that was simple they had their details just written there so It was as easy as copy and paste. And I got them all correct.
- Here is the image I had to track down.
- ![[task5_1602347907147.jpg]].
- **Command/Tool used:**
```bash
# no command
```
- **What I found:** I found that for these kinds of information gathering require you to think outside the box sometimes. Because normally I would have tried to locate the coffee shop itself and not the shop right next to it, but because of my OSINT training I looked at the image and looked for anything that can help me locate the coffee shop.
- **Why it matters:** This matters because Ethical Hacking requires you to think outside the box.

### Step 6 — 
- **What I did:** This one was very similar to the one in task 5. Not really much of a difference really, what I had to do was to find out 'Which restaurant was this picture taken at?' and 'What is the name of the Bon Appetit editor that worked 24 hours at this restaurant?'. Those questions where just one pretty easy google search away. I got sl{katz's deli} for the answer to question 1 and for question 2 I got sl{andrew knowlton}.
- Here is the image:
- ![[task6_1602348602115.jpg]]

- **Command/Tool used:**
```bash
# no command
```
- **What I found:** Same thing in Task 5

### Step 7 — 
- **What I did:** This one again was similar to task 6 and 5 the only difference was that i had to use a different image and that the questions where: 'What is the name of this statue?' and 'Who took this image?'. The answer to both was again just one google search away.
- Here is the image:
- ![[task7_1602636111226.png]]
- **Command/Tool used:**
```bash
# command here
```


### Step 8 — 
- **What I did:** This one, oh this one. This one was the most challenging. I had to think completely hard and sneaky to get this one right. The first question was 'What is the name of the character that the statue depicts?' that one was pretty easy I just dragged the image into google lens and asked the same question and got sl{lady justice} as my answer. For the second question: 'where is this statue located?', that was more challenging. Since that statue was located in many places I had to be more specific. I asked many questions, one generic like where is the statue located, and others more specific. It took a while but I used my head and i realised that in the image there was a building in the reflection. So I was like "If I could find where that building is located then I could find the location of the statue." And doing some more digging i eventually found that the location of the statue was: sl{alexandria, virginia}. For the third and final question(the most challenging one), I had to answer: 'What is the name of the building opposite from this statue?' I thought it was easy since I used the building in the reflection to get the first answer right, I should just find out the name of the building and bada-bing bada-boom I should be done in an instant. WRONG. It turns out it was harder than I thought. Apparently when I searched it I wasn't 'clear enough' and I had to go through a lot of thinking to come to this conclusion to ask google. I know that the location is Alexandria, Virginia and I know the name of the statue. All I need to know is the name of the building opposite the statue. While I was thinking I realised that the answer to the question has 5 words in total so I searched this: 'what is the name of the building opposite the lady justice statue located in alexandria, virginia. The name has to have 5 words'. That was my search query, the results that it yielded had promises the name that fit my query the most was some hospital, but alas the answer was wrong and I had to search again. If google wanted me to be more specific, specific I shall be. This was my fullproof search query that is that will work: 'what is the name of the building opposite the lady justice statue located in alexandria, virginia. The name has to have 5 words. The first word has 3 letters, the second word has 6 letters, the third word has 10 words, the fourth word has 3 letters, and the fith had 4 letters.' This search query was the one I just knew it and low and behold it was there the answer to the question = 'the westin alexandria old town'. I put it in there and it was correct. By the time I was done with that I had only 10 minutes on the clock. It was time to do the last challenge: Task 9.
- Here was the image that gave me so much trouble:
- ![[task8_1603365958159.png]]
- **Command/Tool used:**
```bash
# no command
```
- **What I found:** In short, I found that I have sometimes when you are googling something and you are not getting your desired result, you have to get specific and i mean really specific.
- **Why it matters:** This is important because It will help me get more accurate google responses.

### Step 9 — 
- **What I did:** This is it the last one and the image for Task 9 is, a video? I am supposed to locate the location of a video. Well that is interesting and sounds difficlult right, well not really videos are just a string of images and using a tool called [[FFmpeg]] I was able to get those frames. I used `brew install ffmpeg` to install it on my mac and using the `ffmpeg -i task9_1602643917499.mp4 img%06d.png -hide_banner` command I was able to extract the frames I needed. And grabbing only 3 images from the 1 thousand frames(in which I accedentaly downloaded) and put it in google and asked the same question that i was asked: 'What is the name of the hotel that my friend stayed in a few years ago?'. Of course I didn't ask that exact question I asked 'What is the name of this hotel'. And I got sl{novotel singapore clarke quay} as my answer and it was correct.
- Here is the video:
- ![[task9_1602643917499.mp4]]
- **Command/Tool used:**
```bash
# brew install ffmpeg
# ffmpeg -i task9_1602643917499.mp4 img%06d.png -hide_banner
```
- **What I found:** I learned how to use a tool called Ffmpeg to grap frames from a video and then using google to locate where the video was taken.
- **Why it matters:** This matters because images are not the only media in which I have to use OSINT to locate, there are also videos and this task helped me understand how to geolocate the location of video.

---

## 🛠 Commands Used

| Command                                                    | What It Does                                          |
| ---------------------------------------------------------- | ----------------------------------------------------- |
| brew install ffmpeg                                        | This installs the video to image tool called ffmpeg   |
| ffmpeg -i task9_1602643917499.mp4 img%06d.png -hide_banner | This tool extracts the frames from the selected video |
|                                                            |                                                       |

---
---

## 💡 What I Learned
> Most important section. Write at least 3 bullet points.

- I found out that I can use google to do basic OSINT. This is what people call 'dorking' which is the art of using Google search queries to have Google return specific types of data.
-  I found that for this kind of information gathering, it requires you to think outside the box.
- I learned how to use a tool called Ffmpeg to grap frames from a video and then using google to locate where the video was taken.

---
---

## 🔗 Linked Notes
> Connect this report to your concept and tool notes.

- [[FFmpeg]]

---
---
*Report written in [[TryHackMe Vault]] — part of my ethical hacking journey 🛡️*