---
title: "How one game jam made me create a custom in-engine compiler tool for a brand-new narrative scripting language I made up in less than an hour. Part 3: The Compiler/Interpreter/Thing???"
layout: posts
toc_label: "Table of Contents"
toc_icon: "heart"
excerpt: "This parsing logic with Unity reference asset binding is actually an adventure."
tags: c# unity tool tooling narrative scripting custom scriptableobject
header:
    teaser: assets/posts/unity-acf-game-jam/teaser.png
---

## A quick recap

This is a continuation of [Part two]({% post_url 2022-05-18-unity-acf-game-jam-part-2 %}) of this series. In this post I'll be talking about the actual bit that most people came for - the compiler(???) thing!

## The Compiler/Interpreter/Thing???

**Warning:** In case I hadn't inferred it hard enough already - this is absolutely not how compilers are usually written. This is merely my ancedotal experience of tokyo drifting out a solution that worked for our game's needs at the time of the jam. Please please please do not base any actual compilers off of this!
{: .notice--warning}

For a game jam, honestly this thing is a bit of a monster. It's also hacky as all heck. Brace yourselves for inefficient code. I chose chaotic evil when writing this so I could maintain the code easier and write it quicker. No refunds on this train ride.

So, lets take a look at the bit that really matters - the logic that takes the source script and turns it into a collection of `CharacterDialogueInfo` objects:

```cs
        public List<NarrativeSequence.CharacterDialogueInfo> Interpret(string sourceScript,
            PredefinedVariables predefinedVariables, int musicFilesCount, int soundEffectsCount)
        {
            var characterPaths = AssetDatabase.FindAssets("t:Character").Select(AssetDatabase.GUIDToAssetPath);
            var characterAssets = characterPaths.Select(AssetDatabase.LoadAssetAtPath<Character>);
            List<NarrativeSequence.CharacterDialogueInfo> returnList = new();

            // remove any invalid new line strings

            if (sourceScript.Contains("\r"))
            {
                sourceScript = sourceScript.Replace("\r", "\n");
            }

            var sourceSplit = sourceScript.Split("\n", StringSplitOptions.RemoveEmptyEntries);
            Character character = null;

            for (var index = 0; index < sourceSplit.Length; index++)
            {
                var line = sourceSplit[index];
                var splitLines = line.Split(": ", StringSplitOptions.RemoveEmptyEntries);

                if (splitLines.Length >= 6)
                {
                    throw new FormatException(
                        $"Invalid narrative script was provided to the interpreter. {splitLines.Length} arguments were provided when the maximum is 5. Invalid line number: {index + 1}");
                }

                var characterName = splitLines[0];
                characterName = ResolvePredefinedVariables(characterName, predefinedVariables);

                if (splitLines.Length > 1 && !string.IsNullOrWhiteSpace(characterName) &&
                    !characterName.All(char.IsNumber))
                {
                    character = characterAssets.FirstOrDefault(x =>
                        x.Name.Equals(characterName, StringComparison.OrdinalIgnoreCase));
                }

                if (character is null)
                {
                    throw new FileNotFoundException(
                        $"The character {(string.IsNullOrWhiteSpace(characterName) ? "NO_CHARACTER_NAME" : characterName)} cannot be found in the asset database. Please ensure the character has been created and that the name has been spelt correctly. Line number: {index + 1}");
                }

                var poseIndexString = splitLines.FirstOrDefault(x => x.All(char.IsNumber));
                int? poseIndex = null;

                if (!string.IsNullOrWhiteSpace(poseIndexString))
                {
                    poseIndexString = ResolvePredefinedVariables(poseIndexString, predefinedVariables);
                    poseIndex = int.Parse(poseIndexString);
                }
                
                if (poseIndex >= character.Poses.Count)
                {
                    throw new IndexOutOfRangeException(
                        $"Character Pose Index was outside the bounds of the Poses collection. Length: {character.Poses.Count}, Index: {poseIndex}. Line number: {index + 1}");
                }
                
                var playMusicIndexString = splitLines.FirstOrDefault(x => x.Contains(">>"));
                int? playMusicIndex = null;
                
                if (!string.IsNullOrWhiteSpace(playMusicIndexString))
                {
                    playMusicIndexString = ResolvePredefinedVariables(playMusicIndexString, predefinedVariables);
                    playMusicIndex = int.Parse(playMusicIndexString.Replace(">>", string.Empty));
                }

                if (playMusicIndex >= musicFilesCount)
                {
                    throw new IndexOutOfRangeException(
                        $"Music index was outside the bounds of the music collection. Length: {musicFilesCount}, Index: {playMusicIndex}. Line number: {index + 1}");
                }
                
                var playSoundEffectIndexString = splitLines.FirstOrDefault(x => x.Contains("#"));
                int? playSoundEffectIndex = null;
                
                if (!string.IsNullOrWhiteSpace(playSoundEffectIndexString))
                {
                    playSoundEffectIndexString = ResolvePredefinedVariables(playSoundEffectIndexString, predefinedVariables);
                    playSoundEffectIndex = int.Parse(playSoundEffectIndexString.Replace("#", string.Empty));
                }

                if (playSoundEffectIndex >= soundEffectsCount)
                {
                    throw new IndexOutOfRangeException(
                        $"Sound effect index was outside the bounds of the sound effects collection. Length: {soundEffectsCount}, Index: {playSoundEffectIndex}. Line number: {index + 1}");
                }

                var text = splitLines.Last();
                text = ResolvePredefinedVariables(text, predefinedVariables);

                NarrativeSequence.CharacterDialogueInfo info = new()
                {
                    Character = character,
                    PoseIndex = poseIndex,
                    PlayMusicIndex = playMusicIndex,
                    PlaySoundEffectIndex = playSoundEffectIndex,
                    Text = text
                };

                returnList.Add(info);
            }

            return returnList;
        }
```

OK I PROMISE this is the large code glob. I hope. Maybe. So, lets break this down:

```cs
            var characterPaths = AssetDatabase.FindAssets("t:Character").Select(AssetDatabase.GUIDToAssetPath);
            var characterAssets = characterPaths.Select(AssetDatabase.LoadAssetAtPath<Character>);
```

Up until now, I had completely glossed over how character scriptables are fetched from the projct to use in a `NarrativeSequence`. Well, this is how! This is actually a Unity Editor API, so we can't use it at runtime. However this works at compile time just fine. We get all possible characters using this logic, and a little bit of LINQ sugar in that call to `Select` I am making. The `"t:Character"` is special Unity editor string formatting stuff - basically, it tells `FindAssets` to return all assets of the specified type.

```cs
            List<NarrativeSequence.CharacterDialogueInfo> returnList = new();

            // remove any invalid new line strings

            if (sourceScript.Contains("\r"))
            {
                sourceScript = sourceScript.Replace("\r", "\n");
            }

            var sourceSplit = sourceScript.Split("\n", StringSplitOptions.RemoveEmptyEntries);
```

As the comment suggests, this was to stop new lines breaking code in weird ways. It stopped most of the cases, all but one, I think. There are still cases whereby the asset might have a rogue CRLF-style return that isn't rendered in the editor tool. I dunno why, but I am sure as heck not fixing it at the moment!

And now, in true monkey-at-a-typewriter-fashion, I give you, the parsing logic!

```cs

            var sourceSplit = sourceScript.Split("\n", StringSplitOptions.RemoveEmptyEntries);
            Character character = null;

            for (var index = 0; index < sourceSplit.Length; index++)
            {
                var line = sourceSplit[index];
                var splitLines = line.Split(": ", StringSplitOptions.RemoveEmptyEntries);

                if (splitLines.Length >= 6)
                {
                    throw new FormatException(
                        $"Invalid narrative script was provided to the interpreter. {splitLines.Length} arguments were provided when the maximum is 5. Invalid line number: {index + 1}");
                }
```

The way this works is honestly stupidly simple - it splits each line of the source script based on the remaining newline characters, and then splits the individual lines up based on the separator I mentioned earlier, `": "`. It then checks that each line doesn't have more than the valid amount of arguments - in this case, 5. If it has more than 5, it refuses to transform the narrative script into scriptable data.

Yes, I know magic numbers are bad.

Yes, I am aware I should've checked for `> 5` and not `>= 6`.

No, I don't care right now, this was for a game jam. If this was production code I would've done a metric ton of things differently. But its not!

```cs
               var characterName = splitLines[0];
                characterName = ResolvePredefinedVariables(characterName, predefinedVariables);

                if (splitLines.Length > 1 && !string.IsNullOrWhiteSpace(characterName) &&
                    !characterName.All(char.IsNumber))
                {
                    character = characterAssets.FirstOrDefault(x =>
                        x.Name.Equals(characterName, StringComparison.OrdinalIgnoreCase));
                }

                if (character is null)
                {
                    throw new FileNotFoundException(
                        $"The character {(string.IsNullOrWhiteSpace(characterName) ? "NO_CHARACTER_NAME" : characterName)} cannot be found in the asset database. Please ensure the character has been created and that the name has been spelt correctly. Line number: {index + 1}");
                }
```

Here, we are making an assumption the first argument in the dialogue string is the character's name. We run it through the predefined variables in case it is defined elsewhere, and then if a `Character` is found matching that name, we pull the reference from the collection we made earlier. If the `Character` doesn't exist and one wasn't previously assigned, we throw an exception saying as such and refuse to compile. Great!

```cs
                var poseIndexString = splitLines.FirstOrDefault(x => x.All(char.IsNumber));
                int? poseIndex = null;

                if (!string.IsNullOrWhiteSpace(poseIndexString))
                {
                    poseIndexString = ResolvePredefinedVariables(poseIndexString, predefinedVariables);
                    poseIndex = int.Parse(poseIndexString);
                }
                
                if (poseIndex >= character.Poses.Count)
                {
                    throw new IndexOutOfRangeException(
                        $"Character Pose Index was outside the bounds of the Poses collection. Length: {character.Poses.Count}, Index: {poseIndex}. Line number: {index + 1}");
                }
                
                var playMusicIndexString = splitLines.FirstOrDefault(x => x.Contains(">>"));
                int? playMusicIndex = null;
                
                if (!string.IsNullOrWhiteSpace(playMusicIndexString))
                {
                    playMusicIndexString = ResolvePredefinedVariables(playMusicIndexString, predefinedVariables);
                    playMusicIndex = int.Parse(playMusicIndexString.Replace(">>", string.Empty));
                }

                if (playMusicIndex >= musicFilesCount)
                {
                    throw new IndexOutOfRangeException(
                        $"Music index was outside the bounds of the music collection. Length: {musicFilesCount}, Index: {playMusicIndex}. Line number: {index + 1}");
                }
                
                var playSoundEffectIndexString = splitLines.FirstOrDefault(x => x.Contains("#"));
                int? playSoundEffectIndex = null;
                
                if (!string.IsNullOrWhiteSpace(playSoundEffectIndexString))
                {
                    playSoundEffectIndexString = ResolvePredefinedVariables(playSoundEffectIndexString, predefinedVariables);
                    playSoundEffectIndex = int.Parse(playSoundEffectIndexString.Replace("#", string.Empty));
                }

                if (playSoundEffectIndex >= soundEffectsCount)
                {
                    throw new IndexOutOfRangeException(
                        $"Sound effect index was outside the bounds of the sound effects collection. Length: {soundEffectsCount}, Index: {playSoundEffectIndex}. Line number: {index + 1}");
                }
```

This section is mostly ensuring the specified pose index exists, and to check for any requested asset-related actions that need performing - such as playing music. The relevant exceptions I feel somewhat explain themselves. Optional assets have their indices represetned as `int?` so that this is a well-defined operation in the parsing logic itself.

```cs
                var text = splitLines.Last();
                text = ResolvePredefinedVariables(text, predefinedVariables);

                NarrativeSequence.CharacterDialogueInfo info = new()
                {
                    Character = character,
                    PoseIndex = poseIndex,
                    PlayMusicIndex = playMusicIndex,
                    PlaySoundEffectIndex = playSoundEffectIndex,
                    Text = text
                };

                returnList.Add(info);
```

Resolve any predefined variables in the narrative dialogue itself, assemble the object, add it to the list we plan to return. Splendid!

## In Summary

This logic will turn any narrative script into a collection of character, dialogue and asset reference information, along with optional indices data for poses, sound effects and music files. The main reason we explicitly bind the asset references this way is because Unity loves asset references. The editor loves them, I love them, anyone who likes writing maintainable code loves them. The fact scriptables can store these references just enables us to use them to the extreme, and bind their usage to actual lines of dialogue, allowing for fine-grained control over things such as on what line a piece of music plays. Not bad for a language designed in 15 minutes!

In the last part, we will touch on my experience with editor tooling. Strap in, kids. This is gonna be a bumpy ride.

## Game Jam Bundle!

If what I have been working on in this series has interested you, or you just want to play the finished game, you can get the game as part of a small bundle or $3 as part of an autistic charity fundraiser. I'd really appreciate it if you spent the money for a good cause, as this is something that's very personal to me. Plus, the game is not finished, we have more content to add, so definitely more to look forward to from a gameplay perspecrtive! [You can find the bundle here, and we would really appreciate your financial suport if you can spare it!](https://itch.io/b/1369/autism-acceptance-month-game-jam)