---
date: "2019-10-16T21:55:46+01:00"
title: "A recap of PyCon UK 2019"
authors: []
categories:
  - conference
draft: false
---

On Friday 13th to Tuesday 17th September I had the pleasure of attending the PyCon programming conference in Cardiff City Hall for the third year in a row. Unlike the two previous years I 
wasn't giving a talk which was a welcome break from all the stress that comes with preparing and presesting a talk. This year I also focused less on going to every possible talk and spent 
more time socialising with delegates, as I can always catch up on talks later on YouTube but there's only one opportunity to have a conversation with people.

## Day -1

The conference started on Thursday for me as I arrived at ~1pm on that day to help setup. PyCon of course doesn't have infinite money and thus rely on volunteers to help make the conference 
possible. I enjoyed the conference so much in previous years and felt I should help make it as good a conference as possible for new and returning delegates. City Hall where as always happy 
to see us and provided the great service and venue we've come to expect for PyCon. Some of the more frequent delegates where present on Thursday and it was great to meet up with them for a 
chat before everyone else arrived for the conference on Friday. As usual there was lots to do to get the venue ready for the conference and not everything went to plan, so a lot of duct 
tape was used and there where a few last minute runs to shops for things that went missing between this year and last year.

## Day 0

On Friday morning I arrived a bit earlier than the offical start time to help with registration and help desk (bad idea), there where a lot of people arriving on the first day and there 
where the most badges to sort through. Every year the printers promise all the badges will arrive in alphabetical order, and every year they don't. However it was a great opportunity to 
meet the delegates I'd be interacting with during the conference. Doing this meant I missed the keynote and the first slot of talks, but they wheren't massively of interest to me so it 
didn't matter. After that was the break, and as always every year everyone but the Welsh rush in to get the welshcakes (apparently there very difficult to come by outside of Wales).

The first talk I attended was on the L* search algorithm by David R. Maciver (a regular at PyCon and one of the members of my Star Trek role play group). It is a way to map otherwise 
identical looking rooms/states by traversing paths (experiments) and seeing where you end up (observations). As David himself pointed out the full algorythm is useless to most if not all 
real world applications but variations on it and simplifications can be used to solve other problems. 

Next was a talk by Alex Chan (also a regular at PyCon and this year one of the organisers) on Sans I/O programming. Alex showed us a simple paradigm to help us simplify our programs and 
make them more adaptable. Alex advocaded for seperating the input/output code of a program from its business logic. I would agree with this viewpoint as I've tried in my programs since and 
it indeed does make my life easier.

Finally before lunch I attended a talk by Dustin Ingram on static typing in python. Python is a generally considered to be dynamically typed language which is excellent for quick 
prototyping but not so great for business critical applications. Dustin introduced us to python's type annotations and how we can use type checker such as MyPy to check said type 
annotations.

After lunch there wasn't much of note until a talk by Samathy Barratt on regular expressions. Samathy provided a mathematical explanation of what regular expressions are and how they are 
calculated by introducing us to deterministic/non-deterministic finite state machines. 

At the end of the conference proper for Friday was the highlight of PyCon; the lightning talks! Lightning talks are short 5 minute or less presentations on any topic whatsoever, related to 
python or not at all. As usual there was a lightning talk on first aid (always useful) and possibly my favourite was by Alex Chan on how a recruiter tried to hire automated script pushing 
code to github. 

The after conference event on Friday was board games and manual technology. I sat down with some new and old friends and played some fun card games. One was a bit like twister but with 
cards, and another was a satirical cat based card game. As always there was a pizza van outside, but I had food waiting for me at home so unfortunately I didn't get to have any this year. 
And that was it, the first day of PyCon over. I was already exhausted and there was still 4 days left to go.

## Day 1

On Saturday morning I again volunteered to operate the camera in the Assembly room for talks. Since the speakers didn't move around too much from the lecturn it gave me the opportunity to 
concentrate on what was being said and not miss too much of the content of the talks. The keynote on Saturday was on Python in Africa and how PyCons in Affrica are run and lead. I'm 
reminded every year at PyCon that I should really try to get to an Affrican PyCon and this keynote only emphasised this.

After the break Joel Collins who I'd met at the social event the previous evening gave a talk on robot microscopy and how his team created a cheap microscope with 3d printing, raspberry pis 
and flexible structures as opposed to expensive precision machined bearings. They're currently working on using it in less developed countries to help detect malaria easier and more cheaply. 

Next I was planning to go to a talk on extracting data from PDFs but unfortunately the speaker was nowhere to be seen so instead we got a impromptu talk from Alex Chan on the curb cut 
effect. The curb cut effect is how a adaptation for disabled or disadvantaged people (such a curb cuts/dipped kerbs) can end up making an improvment for everyone else.

The final talk I went to on Saturday was by Marco Bonzanini on how statistics can be used to lie and how to spot naughty numbers. The conference dinner was on Saturday night. This year I 
didn't decide to pay for a ticket for that, however in hindsight this wasn't the smartest choice as I missed out on a great socialising event. Nevertheless I went out with some other people 
not going to the conference dinner. 

## Day 2

On Sunday everyone was pretty exhausted from the night before and I didn't go to any talks before about 3pm. That talk was by Yeray Díaz Díaz on how dependency injection is a good thing to 
use in python even if it's seen as overengineering most of the time. This talked linked nicely to the Sans I/O talk by Alex Chan as both focused on decoupling different parts of code and how 
it makes our lives easier. 

At the end of the day was the lightning talks as usual, and the code dojo. A code dojo is where people split up into small groups with a single sentence task and try to program a solution 
to it. This year the program was "make a dance". Ostensibly the program was meant to be in python as this was a python conference but I decided to try and write part of it in Rust, a 
language I've been trying to learn more about recently. We only had 40 minutes this time because of a delay in getting a projector. Nevertheless out team got something working even if it 
wasn't remotely impressive compared to what other teams managed in the same time. All the code from all teams is available on [the pycon uk github](https://github.com/PyconUK/dojo19).

## Day 3

By Monday many people had to leave, or had just spent too much time out the previous evening so everything was a bit slow to get going, but the conference went on; there was a need for even 
more exchanging of knowlege! The keynote on this morning was possibly the best keynote I've ever had the pleasure of viewing live (from the front row too - advantages of getting there 
early). Tobias introduced us to some events in history where humans in their messy nature had to support multiple systems (such as calendars) or had to change from one to the other 
simultaneously (like the side of the road to drive on, which was changed in Sweden as recently as the late 20th century). He showed us some ways humans have dealt with these successfuly, 
and not so successfuly (for those concerned Sweden changed rather successfuly). 

The next talk I attended was on writing secure python code. This was one of the talks I live tweeted instead of taking notes on so I'll put the thread here instead. Live tweeting as a form 
of note taking was an interesting learning experience and required me to very actively think about what was being said. Having to condense every thought into less that 280 characters before 
the speaker moves onto the next slide is certainly a challenge!

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Secure python now!</p>&mdash; ⧖🔶🏳️‍🌈✨impl Cat for Q (it/its)✨🏳️‍🌈🔶⧖ (@TheEnbyperor) <a 
href="https://twitter.com/TheEnbyperor/status/1173547830339874817?ref_src=twsrc%5Etfw">September 16, 2019</a></blockquote>

Next was a talk on the game servers running the 'World of' series of games such as World of Warcraft. Even with my live tweeting this talk very much went over my head but I'll leave my 
tweets here for you to attempt to decipher.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Now massively multiplayer online games with python with Dmitry Karpov</p>&mdash; ⧖🔶🏳️‍🌈✨impl Cat for Q (it/its)✨🏳️‍🌈🔶⧖ 
(@TheEnbyperor) <a href="https://twitter.com/TheEnbyperor/status/1173552656150908928?ref_src=twsrc%5Etfw">September 16, 2019</a></blockquote>

After lunch was a talk I was very much interested in the details of (*cough* brexit *cough*) titled "How to use Python to expose politicians?" presented by Rafael. Rafael is from Brazil. He 
found some sources of open data about government and policitian expenditure in Brazil and showed us how to then graph these and more importantly add these graphs to websites in a way the 
general public can play with. He's already found some suspicious payments from policital parties, but more importantly it was useful to know how to present any data in an understandable and 
accessible format.

And that was it for Monday, I didn't go to any more talks at PyCon and instead spent the time talking to people. I can always catch up with talks on YouTube but I can't have conversations 
with people at a later date.

## Day 4

Tuesday is possibly the best day of the conference. The last day is a 'sprints' day, which counter to what the name suggests is actually more of a jog or walk. A sprint is a load of people 
getting around a table and working on a project together for a day (sprinting on a project). This year I worked with a friend from previous PyCons - John Chandler - on 'Tweets of Commons'; 
a project to fetch and analyze the tweets of all members of the Houses of Commons and Lords. We didn't get very far and only managed to fetch some tweets into a database, no anaysis took 
place but we can continue on it next year or maybe some other time. 

Unfortunately all good things have to come to an end and at the end of Tuesday PyCon came to an end and we all had to go back to our significantly more boring lives. Till next year!

<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 
