---
title: "In-Engine Narrative Compiler - Part 1: Language Design"
layout: posts
toc_label: "Table of Contents"
toc_icon: "heart"
excerpt: "Language design is not my forte."
tags: c# unity tool tooling narrative scripting custom scriptableobject
header:
    teaser: assets/posts/unity-acf-game-jam/teaser.png
---

## Setting the Scene...

It's 8th April. I'm sitting down for my first meeting with my game jam teammates, and friends, for the [Autism Acceptance Month Game Jam](https://itch.io/b/1369/autism-acceptance-month-game-jam). We are discussing the game we want to work on for the month. We actually settled on an idea fairly quickly! One junior programmer, myself (so the make-do senior programmer of the group), two artists, one of which wrote all the narrative for the game - the other ended up working on the game's soundtrack with me.

The theme is autistic experiences. We decide on a game idea fairly fast.

There is just one problem - the only engine apart from [NovelRT](https://github.com/NovelRT/NovelRT) that comes integrated with a narrative scripting language that I am aware of is [Ren'Py](https://github.com/renpy/renpy). The former is not ready entirely and requires C++ knowledge, which the junior developer didn't really have enough of to make it work. The latter, I wrote the former to replace for myself, because I'm not a fan of how Ren'Py handles many things personally, and I doubt that is changing any time soon.

The junior did know basic C#.

The solution? We use Unity...with our own custom narrative scripting language.

"But Maaaaaaaaaatt why didn't you use Ink?!" I can hear some of you (probably?) crying out at me. A few reasons.

- I am not a fan of Ink syntax.
- I am not a fan of the Ink codebase - if it broke it'd be on me to fix it. Not fun to wade through.
- Writing game dev-related tools is kind of my specialty and I wanted to give it a real go in Unity again, after a few years off.

## The Basics of the Language Design

The basic idea was to provide a primitive narrative scripting language, so that beginner Unity users could add narrative to our game. The core design was actually fairly straightforward; choose some way to define characters in the narrative, and be able to use poses for rendering in the game scene. On the first pass, I came up with this:

```
Mackie: 0: Oh, hello! This is a line of dialogue, with a pose index!
And it supports multiple lines of separate dialogue with implicitly provided character information from the previous line! Great!

It is not new-line sensitive, either! However, it is whitespace-sensitive.

Bex: 3: Hello! A new character enters the fray!

Matt: 2: I really hope this won't take me long to implement...it shouldn't. On paper. Maybe. Who knows?

Jo: 0: Where's the popcorn? I am here to see this tool explode!
```

Let's break this down a bit.

The person's name, for example "Matt" indicates that this should use character data where the character's name is "Matt", along with all the "pose" information so that Unity can render the correct character sprite. For the sake of conciseness, I will refer to character sprites simply as "poses" going forward. The colon and space act as a separator for the different arguments so the interpreter (compiler? thing??? It's a game jam, I don't think this slapdash implementation has a real name given how it works) knows what data is what. The last argument here is the actual dialogue itself. In the language I designed (which has no real name yet!), the dialogue is always assumed to be the last argument in a given line of narrative script. If it isn't, you'll get weird behaviour.

The design of this language also meant that we could support sound effect (the `#` symbol) and music files (the `>>` symbol) later on directly in the script, like so:

```
Mackie: 0: #4: I am playing the sound effect at index 4!

Matt: 2: >>2: And I am playing the music data at index 2!
```

You could even have one single line of dialogue execute both in a single line! We also eventually added variable support, using `$`:

```
Matt: I am feeling $mood today.
```

A lot of the design decisions made here were made so that it was very easy and quick to parse. In a more complete narrative scripting language such as [Fabulist](https://github.com/NovelRT/Fabulist), you most definitely would not do it this way, if you want something maintainable, terse and versatile. Many of the decisions here were made as a result of time constraints on the game jam itself.

Okay, everyone caught up on the rough design? Great! Because I spat this whole design out in about 15 minutes and its not changing now, so, hope you like it...not that you have a choice or anything!

In the next part, I'll be going over some Unity-specific implementation details that matter. All the parts should be up at the same time - I just needed to break the post up a bit. It got a bit long.

## Game Jam Bundle!

If what I have been working on in this series has interested you, or you just want to play the finished game, you can get the game as part of a small bundle or $3 as part of an autistic charity fundraiser. I'd really appreciate it if you spent the money for a good cause, as this is something that's very personal to me. Plus, the game is not finished, we have more content to add, so definitely more to look forward to from a gameplay perspecrtive! [You can find the bundle here, and we would really appreciate your financial suport if you can spare it!](https://itch.io/b/1369/autism-acceptance-month-game-jam)