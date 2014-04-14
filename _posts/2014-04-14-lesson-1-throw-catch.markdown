---
layout: post
title: "Juggling Lesson 1 &mdash; Basic Throw / Catch"
date: 2014-04-13 15:44:02
categories: juggling
---

This is the first lesson of my juggling class designed for developers, coders, engineers. Over the next weeks (months perhaps?) you will learn how to apply and transfer your knowledge of computer science principles to the art of juggling with 3 balls. You only need to invest a few minutes each day. We will focus on the basic pattern for now, and add tricks and variants later.

Why is this important? Anyone working on a computer all day should learn how to juggle. Not only do you take your mind off your programming problems for a few minutes, you also concentrate on something completely different, which you will soon see helps to re-gain focus. You also stretch your finger muscles and stand up from your desk, and your whole body will thank you for that break. 

Each of the exercises (which will be released semi-regularly whenever I have time) will add a new element to what you've already learned, but we will start from zero (as any good software engineer would). All you need is a few juggling balls: stress balls, tennis balls, mandarines or any other spherical objects of similar size and weight will do.

### Let's try this...

In today's exercise, instead of throwing an exception, you will instead learn how to throw a single ball from left to right and back. Let's get into the correct position:

- Stand up
- Arms relaxed on your sides, forearms in 90&deg; angle
- Your elbows remain in this position, they don't move
- Imagine two points L and R, they are on the same height as your head and above your left and right palms

Now for the interesting part:

1. Take a ball in your left hand and throw it so it just touches point R and then drops into your right hand[^1], as shown in pictures a) and b)
2. Throw the ball back so it just touches point L and falls into your left hand again, as in c) and d)

Then repeat: throw, catch, re-throw. 

![Juggling Instructions]({{ site.url }}/assets/juggling/juggling_1.png)

Here is the algorithm in Java code: 

~~~java
void juggle1() {
    while (true) {
        try {
            throwBallLeft();
        }
        catch (JugglingBall b) {
            closeHandRight();
        }

        try {
            throwBallRight();
        }
        catch (JugglingBall b) {
            closeHandLeft();
        }
    }
}
~~~

Some gotchas:

* Your elbows don't move. Most work is done from your forearm and wrist. 
* Catch gently. The ball just falls into your palm and you simply close it. 
* Don't reach for the ball. It has to fall into your hand exactly where the hand is.

Practice for 15 minutes total a day and make sure your trajectories are always exactly the same. No exceptions.


[^1]: it has been pointed out to me that this is physically impossible and that the ball would always follow a parabolic trajectory. This is, of course, completely irrelevant as this is a Juggling class and not a Physics class. Believe me, it will work. 

