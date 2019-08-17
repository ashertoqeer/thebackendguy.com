---
layout: post
title: "Messy CodeBase, Who to BLAME?"
permalink: "messy-codebase-who-to-blame"
last_modified_at: 2019-07-24T00:00:00
excerpt: "How a codebase becomes a mess and how to prevent it."
category: "Best Practices"
---
Whenever a new project is starts, It starts with an ambition of building a simple nice architecture. It usually follows a pattern like MVC, business logic usually goes to services, controllers are nicely set, model layer is nothing less than a piece of art. Design patterns are being followed, SOLID principles are not getting compromised. It is nice, simple, elegant, AT THE START!

After sometime, the beauty is gone, nobody knows exactly what architecture is like, layers are leaking, SOLID principals are now no where to be seen, strange bugs are popping here and there, reading code is now a nightmare, It becomes a MESS.

### What makes a Messy codebase?

We occasionally deal with Messy code-bases. The developers don't initially plan for a mess, it happens automatically. There are many factors that can contribute to a messy code base, i.e unrealistic deadlines, initial developers left the project, no documentation for new joiners, etc. 

The most important factor is **Psychology**.

Developer psychology plays a vital role in success or failure of a project. As I said earlier, The developers don't initially plan for a mess, It starts with a tiny, little, small portion. Perhaps there is a class doing multiple things, or one method in controller is directly accessing database layer without going through proper route. A public method is unnecessarily way too large, or there are bunch of nested `if-else` which needs refactoring.

These tiny bad portions of code are perhaps due to some hot fix, or meeting a deadline, or simply developer was in hurry. It is normal and must be fixed as soon as possible.

The problem starts when these small portions are left untreated. First there was only one bad class, now there are three bad classes, two methods are directly calling database layer, multiple methods with nested `if-else`. Perhaps, due to same reasons.

These bad portions starts growing, why do you care when nobody else seems to care? It grows and grows and eventually it eats up all the beauty, elegance, simplicity of a code base.

### How to Prevent Messy code base?

The solution is rather very simple. Don't let it start. It all starts small and if we want to prevent a Mess, we must not allow even a tiny, little, small portion of bad code to live. Eliminate bad code as soon as possible. Repeatedly review your code with in your team and mark portions which needs refactoring, no matter how small it may seem. This is a continuous process, a habit, a way of doing quality work. Make standards and make sure everyone follow them strictly. Read your code base in light of design patterns, SOLID principles, good practices. Keep an eye on logical view of code, don't let it slip. Make a checklist for iteration, add proper comments for what needs to be changed. 

Treat your code like a baby, keep it clean. :)

Inspired by: "The Pragmatic Programmer"
