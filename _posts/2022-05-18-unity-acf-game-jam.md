---
title: "How one game jam made me create a custom in-engine compiler tool for a brand-new narrative scripting language I made up in less than an hour."
layout: posts
toc_label: "Table of Contents"
toc_icon: "heart"
excerpt: "Compilers, languages and scriptable objects, oh my!"
tags: c# unity tool tooling narrative scripting custom scriptableobject
header:
    teaser: assets/posts/unity-acf-game-jam/teaser.png
---

## Setting the Scene...

It's 8th April. I'm sitting down for my first meeting with my game jam teammates, and friends, for the Autistic Creative Festival game jam (I'll update this with a relevant link in the future). We are discussing the game we want to work on for the month. We actually settled on an idea fairly quickly! One junior programmer, myself (so the make-do senior programmer of the group), two artists, one of which wrote all the narrative for the game - the other ended up working on the game's soundtrack with me.

The theme is autistic experiences. We decide on a game idea fairly fast.

There is just one problem - the only engine apart from [NovelRT](https://github.com/NovelRT/NovelRT) that comes integrated with a narrative scripting language that I am aware of is [Ren'Py](https://github.com/renpy/renpy). The former is not ready entirely and requires C++ knowledge, which the junior developer didn't really have enough of to make it work. The latter, I wrote the former to replace for myself, because I'm not a fan of how Ren'Py handles many things personally, and I doubt that is changing any time soon.

The junior did know basic C#.

The solution? We use Unity...with our own custom narrative scripting language.

"But Maaaaaaaaaatt why didn't you use Ink?!" I can hear some of you (probably?) crying out at me. A few reasons.

1 - I am not a fan of Ink syntax.

2 - I am not a fan of the Ink codebase - if it broke it'd be on me to fix it. Not fun to wade through.

3 - Writing game dev-related tools is kind of my specialty and I wanted to give it a real go in Unity again, after a few years off.

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

Matt: 2: #2: And I am playing the music data at index 2!
```

You could even have one single line of dialogue execute both in a single line! We also eventually added variable support, using `$`:

```
Matt: I am feeling $mood today.
```

Okay, everyone caught up on the rough design? Great! Because I spat this whole design out in about 15 minutes and its not changing now, so, hope you like it...not that you have a choice or anything!

## Implementation Details That Matter

So, onto the actual implementation of this lovely language. My first question was, "how do we get this into a format that Unity can actually understand?" - fortunately, given my love for driving games with data and not code as much as humanly possible, I was already aware that Unity supports a feature called ScriptableObjects. You can find more information about how those work [here](https://docs.unity3d.com/Manual/class-ScriptableObject.html), but for the sake of this post I shall summarise. A ScriptableObject is Unity's mechanism for supporting data containers as assets - you can create them, modify their values in the inspector, and have MonoBehaviours consume them and react to the data they contain. Since implementations of ScriptableObjects are also C# classes that inherit the base class, you can also add actual functionality to a ScriptableObject. However, this is rather rare due to the nature of ScriptableObjects being primarily just data containers.

Did I get the general idea across? Awesome! Let's look at the ScriptableObject I implemented to support narrative scripting then!

```cs
namespace ACHNarrativeDriver.ScriptableObjects
{
    [CreateAssetMenu(fileName = "NewNarrativeSequence", menuName = "ACH Narrative Driver / NarrativeSequence")]
    public class NarrativeSequence : ScriptableObject
    {
        [Serializable]
        public class CharacterDialogueInfo
        {
            [SerializeField] private Character _character;
            [SerializeField] private bool _hasPoseIndex;
            [SerializeField] private int _poseIndex;
            [SerializeField] private bool _hasPlayMusicIndex;
            [SerializeField] private int _playMusicIndex;
            [SerializeField] private bool _hasPlaySoundEffectIndex;
            [SerializeField] private int _playSoundEffectIndex;
            [SerializeField] private string _text;

            public Character Character
            {
                get => _character;
                set => _character = value;
            }

            public int? PoseIndex
            {
                get => _hasPoseIndex ? _poseIndex : null;
                set
                {
                    if (value.HasValue)
                    {
                        _poseIndex = value.Value;
                        _hasPoseIndex = true;
                    }
                    else
                    {
                        _hasPoseIndex = false;
                    }
                }
            }
            
            public int? PlayMusicIndex
            {
                get => _hasPlayMusicIndex ? _playMusicIndex : null;
                set
                {
                    if (value.HasValue)
                    {
                        _playMusicIndex = value.Value;
                        _hasPlayMusicIndex = true;
                    }
                    else
                    {
                        _hasPlayMusicIndex = false;
                    }
                }
            }
            
            public int? PlaySoundEffectIndex
            {
                get => _hasPlaySoundEffectIndex ? _playSoundEffectIndex : null;
                set
                {
                    if (value.HasValue)
                    {
                        _playSoundEffectIndex = value.Value;
                        _hasPlaySoundEffectIndex = true;
                    }
                    else
                    {
                        _hasPlaySoundEffectIndex = false;
                    }
                }
            }

            public string Text
            {
                get => _text;
                set => _text = value;
            }

            public override string ToString()
            {
                return $"Character: {Character.Name}, HasPoseIndex: {_hasPoseIndex}, {(_hasPoseIndex ? "PoseIndex: " + PoseIndex + ", " : string.Empty)}Text: {Text}";
            }
        }

        [Serializable]
        public class ChoiceInfo
        {
            [SerializeField] private string _choiceText;
            [SerializeField] private NarrativeSequence _narrativeResponse;

            public string ChoiceText
            {
                get => _choiceText;
                set => _choiceText = value;
            }

            public NarrativeSequence NarrativeResponse
            {
                get => _narrativeResponse;
                set => _narrativeResponse = value;
            }
        }
        
        [SerializeField] private Sprite _backgroundSprite;
        [SerializeField] private NarrativeSequence _nextSequence;
        [SerializeField] private List<AudioClip> _musicFiles;
        [SerializeField] private List<AudioClip> _soundEffectFiles;
        [SerializeField] private List<CharacterDialogueInfo> _characterDialoguePairs;
        [SerializeField] private List<ChoiceInfo> _choices;

        public List<ChoiceInfo> Choices
        {
            get => _choices;
            set => _choices = value;
        }

        public NarrativeSequence NextSequence
        {
            get => _nextSequence;
            set => _nextSequence = value;
        }

        public List<AudioClip> MusicFiles
        {
            get => _musicFiles;
            set => _musicFiles = value;
        }
        
        public List<AudioClip> SoundEffectFiles
        {
            get => _soundEffectFiles;
            set => _soundEffectFiles = value;
        }

        public List<CharacterDialogueInfo> CharacterDialoguePairs
        {
            get => _characterDialoguePairs;
            set => _characterDialoguePairs = value;
        }

        public Sprite BackgroundSprite
        {
            get => _backgroundSprite;
            set => _backgroundSprite = value;
        }

        [field: SerializeField, HideInInspector]
        public string SourceScript { get; set; }
    }
}

```

Woah. Okay. That's a lot of properties and other very cool modern C# stuff. Some of it possibly a hangover from the late night game jam debugging sessions where I was losing my mind, as all game developers do! So, lets break it down a bit.

```cs
        [SerializeField] private Sprite _backgroundSprite;
        [SerializeField] private NarrativeSequence _nextSequence;
        [SerializeField] private List<AudioClip> _musicFiles;
        [SerializeField] private List<AudioClip> _soundEffectFiles;
        [SerializeField] private List<CharacterDialogueInfo> _characterDialoguePairs;
        [SerializeField] private List<ChoiceInfo> _choices;
```

These are all the things that eventually made it into the narrative tool due to the requirements of the game jam; the current environment background, the next narrative sequence to execute, the dialogue and character information for the current sequence, and any music and sound effect files that the narrative file might rely on too! I also organised some of the data into nested serialised types, such as `CharacterDialogueInfo`, to help streamline and group data in a more organised fashion. You don't need these nested types, but it certainly made the inspector look nicer!

![The Unity inspector showing off an example of the Narrative Sequence scriptable object being used. It contains data for a sequence pertaining to a character called Flora, and uses a music file. The array of dialogue that the player and Flora say is also present.](/assets/posts/unity-acf-game-jam/narrative-sequence-inspector.png)

The reason we exposed these members to the inspector using `SerializeField` was so that if the UI tool I bolted on top of this thing broke, we could edit the scriptables directly. Thankfully, this didn't happen. Happy face.gif.

I also persisted the original source code into the asset itself using this:

```cs
        [field: SerializeField, HideInInspector]
        public string SourceScript { get; set; }
```

To not drag out the explanation of this piece of code too much, it is hidden from the inspector because the auto-generated field name is not only ugly, but also people shouldn't be directly modifying this. This is so Unity can track changes easier with asset dirtying, amongst other things. Like the other members, this property is directly set by the compiler-ish tool.

The rest of the code is mostly just property accessors so that intellisense didn't get totally clogged up with things I didn't need to care about. It might be a game jam but I like keeping my autocomplete as relevant as I can, and properties really help with that. If you're unsure on how these properties work, you can find more information [In this part of the Microsoft documentation.](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/expression-bodied-members)

The `Character` Scriptable is probably a little easier to digest:

```cs
namespace ACHNarrativeDriver.ScriptableObjects
{
    [CreateAssetMenu(fileName = "NewNarrativeSequence", menuName = "ACH Narrative Driver / Character")]
    public class Character : ScriptableObject
    {
        [SerializeField] private string _name;
        [SerializeField] private List<Sprite> _poses;
        [SerializeField] private Sprite _nameplateSprite;

        public string Name => _name;
        public IReadOnlyList<Sprite> Poses => _poses.AsReadOnly();
        public Sprite NameplateSprite => _nameplateSprite;
    }

```

This is mostly straightforward. The name is the name of the character and the list of sprites are the different poses the character can have. The nameplate sprite is something specific to the game jam - since I was writing the narrtive tooling, I was also responsible for handling UI rendering - and the character's nameplates were all different! So I just stored it in here for my canvas to use later.

And the last scriptable that matters to this tool, is `PredefinedVariables`:

```cs
namespace ACHNarrativeDriver.ScriptableObjects
{
    [CreateAssetMenu(fileName = "NewNarrativeSequence", menuName = "ACH Narrative Driver / PredefinedVariables")]
    public class PredefinedVariables : ScriptableObject
    {
        [Serializable]
        public class VariableNameValuePair
        {
            [SerializeField] private string _key;
            [SerializeField] private string _value;

            public string Key => _key;
            public string Value => _value;
        }
        
        [SerializeField] private List<VariableNameValuePair> _variables;

        public IReadOnlyList<VariableNameValuePair> Variables => _variables.AsReadOnly();
    }
}
```

This one does what it says on the tin, mostly - it is a list of predefined variables for the `NarrativeSequence` to consume. The name is a bit misleading however - I think a better name would've been `PredefinedConstants` since these values were mostly just baked into the scripts at compile time by the end of the jam. We'll explore this in a bit when we look at the compiler(?) internals.


Okay, great, we've gone over the data container stuff that matters - I should uh...probably start talking about the compiler/interpreter/thing? Honestly given the nature of a game jam I have no idea what to call this thing. Oh well, onward!

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

## So...how do I use this as a narrative nerd?

It's actually fairly straightforward! I wrote a Unity UI tool/extension that directly interfaces with all of this for you!

![A picture of the narrative editor UI. It has buttons for adding and removing assets such as sounds and music, a field for adding the background sprite, a massive box for adding the source script, and the ability to add choices to the sequence. The narrative script depicts a conversation between a character called Flora, and the player.](/assets/posts/unity-acf-game-jam/narrative-driver.png)

All that's actually needed to use this tool is to create the relevant scriptables, have your assets at the ready, and get to writing that narrative! When you're done, you just need to press the "Save Source Script" button at the bottom. You can do many things with a `NarrativeSequence` from this user interface. Very epic.

## And uh...how was this narrative editor UI programmed?

On the one hand, I am glad you asked that.

On the other, the code is not very epic. I really need to try UXML at some point, because this is partially the fault of the fact that the editor UI APIs are from Unity 4 or so. Today is not that day.

Still want to learn about it? Okay...guess I lied, this should be the last big blob...maybe.

```cs
namespace ACHNarrativeDriver.Editor
{
    public class NarrativeSequenceEditor : EditorWindow
    {
        private NarrativeSequence _currentNarrativeSequence;
        private readonly Interpreter _interpreter = new();
        private bool _currentChoicesValue = false;
        private bool _firstRead = true;
        private PredefinedVariables _predefinedVariables;

        private void OnGUI()
        {
            GUILayout.Label("Narrative Sequence Editor", EditorStyles.boldLabel);
            _currentNarrativeSequence = (NarrativeSequence)EditorGUILayout.ObjectField("Target",
                _currentNarrativeSequence, typeof(NarrativeSequence), false);

            if (_currentNarrativeSequence is null)
            {
                _currentChoicesValue = false;
                _firstRead = true;
                _predefinedVariables = null;
                return;
            }

            if (_firstRead)
            {
                _firstRead = false;
                if (_currentNarrativeSequence.Choices.Count > 0)
                {
                    _currentChoicesValue = true;
                }
            }

            _predefinedVariables = (PredefinedVariables)EditorGUILayout.ObjectField(
                "Predefined Variables", _predefinedVariables, typeof(PredefinedVariables),
                false);

            var previousSprite = _currentNarrativeSequence.BackgroundSprite;
            _currentNarrativeSequence.BackgroundSprite = (Sprite)EditorGUILayout.ObjectField("Background Image",
                _currentNarrativeSequence.BackgroundSprite, typeof(Sprite), false);
            var backgroundSpriteChanged = _currentNarrativeSequence.BackgroundSprite != previousSprite;
            
            GUILayout.Label("Music files", EditorStyles.label);

            bool musicCollectionModified = false;

            for (int index = 0; index < _currentNarrativeSequence.MusicFiles.Count; index++)
            {
                var previousClip = _currentNarrativeSequence.MusicFiles[index];
                _currentNarrativeSequence.MusicFiles[index] =
                    (AudioClip)EditorGUILayout.ObjectField($"Music {index}",
                        _currentNarrativeSequence.MusicFiles[index], typeof(AudioClip), false);

                if (previousClip != _currentNarrativeSequence.MusicFiles[index])
                {
                    musicCollectionModified = true;
                }
            }
            
            if (GUILayout.Button("Add new"))
            {
                _currentNarrativeSequence.MusicFiles.Add(null);
                musicCollectionModified = true;
            }

            if (GUILayout.Button("Remove last") && _currentNarrativeSequence.MusicFiles.Count > 0)
            {
                _currentNarrativeSequence.MusicFiles.RemoveAt(_currentNarrativeSequence.MusicFiles.Count - 1);
                musicCollectionModified = true;
            }
            
            GUILayout.Label("Music files", EditorStyles.label);

            bool soundEffectCollectionModified = false;

            for (int index = 0; index < _currentNarrativeSequence.SoundEffectFiles.Count; index++)
            {
                var previousClip = _currentNarrativeSequence.SoundEffectFiles[index];
                _currentNarrativeSequence.SoundEffectFiles[index] =
                    (AudioClip)EditorGUILayout.ObjectField($"Sound Effect {index}",
                        _currentNarrativeSequence.SoundEffectFiles[index], typeof(AudioClip), false);

                if (previousClip != _currentNarrativeSequence.SoundEffectFiles[index])
                {
                    soundEffectCollectionModified = true;
                }
            }
            
            if (GUILayout.Button("Add new"))
            {
                _currentNarrativeSequence.SoundEffectFiles.Add(null);
                soundEffectCollectionModified = true;
            }

            if (GUILayout.Button("Remove last") && _currentNarrativeSequence.SoundEffectFiles.Count > 0)
            {
                _currentNarrativeSequence.SoundEffectFiles.RemoveAt(_currentNarrativeSequence.SoundEffectFiles.Count - 1);
                soundEffectCollectionModified = true;
            }

            GUILayout.Label("Source Script", EditorStyles.label);
            var previousSourceScript = _currentNarrativeSequence.SourceScript;
            _currentNarrativeSequence.SourceScript = GUILayout.TextArea(_currentNarrativeSequence.SourceScript,
                GUILayout.ExpandWidth(true), GUILayout.ExpandHeight(true));
            var sourceScriptChanged = _currentNarrativeSequence.SourceScript != previousSourceScript;

            _currentChoicesValue = GUILayout.Toggle(_currentChoicesValue, "Has Choices");

            bool nextNarrativeSequenceModified = false;
            if (_currentChoicesValue)
            {
                for (var index = 0; index < _currentNarrativeSequence.Choices.Count; index++)
                {
                    var choice = _currentNarrativeSequence.Choices[index];
                    GUILayout.BeginHorizontal();
                    GUILayout.Label($"Choice text {index}");
                    var previousText = choice.ChoiceText;
                    choice.ChoiceText = GUILayout.TextField(choice.ChoiceText);
                    GUILayout.EndHorizontal();
                    var previousResponse = choice.NarrativeResponse;
                    choice.NarrativeResponse = (NarrativeSequence)EditorGUILayout.ObjectField(
                        "Narrative Response", choice.NarrativeResponse, typeof(NarrativeSequence),
                        false);
                    if (previousText != choice.ChoiceText || previousResponse != choice.NarrativeResponse)
                    {
                        nextNarrativeSequenceModified = true;
                    }
                }

                if (GUILayout.Button("Add new"))
                {
                    _currentNarrativeSequence.Choices.Add(new());
                    nextNarrativeSequenceModified = true;
                }

                if (GUILayout.Button("Remove last") && _currentNarrativeSequence.Choices.Count > 0)
                {
                    _currentNarrativeSequence.Choices.RemoveAt(_currentNarrativeSequence.Choices.Count - 1);
                    nextNarrativeSequenceModified = true;
                }
            }
            else
            {
                _currentNarrativeSequence.Choices.Clear();
                var previousSequence = _currentNarrativeSequence.NextSequence;
                _currentNarrativeSequence.NextSequence = (NarrativeSequence)EditorGUILayout.ObjectField(
                    "Next Narrative Sequence", _currentNarrativeSequence.NextSequence, typeof(NarrativeSequence),
                    false);

                if (previousSequence != _currentNarrativeSequence.NextSequence)
                {
                    nextNarrativeSequenceModified = true;
                }
            }

            bool compiledScriptChanged = false;
            if (GUILayout.Button("Save Source Script"))
            {
                compiledScriptChanged = true;
                var listOfStuff = _interpreter.Interpret(_currentNarrativeSequence.SourceScript, _predefinedVariables, _currentNarrativeSequence.MusicFiles.Count, _currentNarrativeSequence.SoundEffectFiles.Count);
                _currentNarrativeSequence.CharacterDialoguePairs = listOfStuff;

                if (_currentNarrativeSequence.Choices is not null && _predefinedVariables is not null)
                {
                    foreach (var choice in _currentNarrativeSequence.Choices)
                    {
                        choice.ChoiceText =
                            _interpreter.ResolvePredefinedVariables(choice.ChoiceText, _predefinedVariables);
                    }
                }
            }

            if (sourceScriptChanged || musicCollectionModified || soundEffectCollectionModified || nextNarrativeSequenceModified ||
                compiledScriptChanged || backgroundSpriteChanged)
            {
                EditorUtility.SetDirty(_currentNarrativeSequence);
            }
        }

        [MenuItem("Window / ACH Narrative Driver / Narrative Sequence Editor")]
        public static void ShowEditor()
        {
            var window = EditorWindow.GetWindow<NarrativeSequenceEditor>(title: "Narrative Sequence Editor");
            window.minSize = new Vector2(500, 500);
        }
    }
}
```

Once more, lets break this down.

```cs
    public class NarrativeSequenceEditor : EditorWindow
```

To make a custom window for the editor, your class must inherit `EditorWindow`. Fairly straightforward.

```cs
        private void OnGUI()
        {
            GUILayout.Label("Narrative Sequence Editor", EditorStyles.boldLabel);
            _currentNarrativeSequence = (NarrativeSequence)EditorGUILayout.ObjectField("Target",
                _currentNarrativeSequence, typeof(NarrativeSequence), false);

            if (_currentNarrativeSequence is null)
            {
                _currentChoicesValue = false;
                _firstRead = true;
                _predefinedVariables = null;
                return;
            }
```

`OnGUI` is the method called by the editor to render your GUI. Every button, label, textbox, etc. you want has to be drawn here. It is very reminiscent of Dear ImGUI...only worse.

Our first port of call is to give it a nice bold title in the window that says the name of the tool. Awesome!

The call to `ObjectField` renders a...well...an object field! We specify the type so Unity can filter for it and provide full editor support for the object field. It works the same way as object fields in the inspector (for example, when something wants you to drag and drop a reference to a Rigidbody). The `false` argument to this call is to prevent objects that only exist within a scene being used...not that I've ever tried to do that with scriptables, but hey, you never know.

If no `NarrativeSequence` is chosen, we bail out early as to not confuse end-users with a non-function UI. Might be obvious, but this is actually fairly important, as not only does it handle a good UX, but also allows me to reset the editor window to the "first time" state. Moving on!

```cs
            if (_firstRead)
            {
                _firstRead = false;
                if (_currentNarrativeSequence.Choices.Count > 0)
                {
                    _currentChoicesValue = true;
                }
            }
```

This just checks to see if there are any choices in the selected sequence on first read of the asset. This is to prevent destructive behaviour the editor tool performs later on.

```cs
            _predefinedVariables = (PredefinedVariables)EditorGUILayout.ObjectField(
                "Predefined Variables", _predefinedVariables, typeof(PredefinedVariables),
                false);
```

This object field is being used to obtain a `PredefinedVariables` scriptable. Once again we disable this field for objects that are in the currently open scene.

```cs
            var previousSprite = _currentNarrativeSequence.BackgroundSprite;
            _currentNarrativeSequence.BackgroundSprite = (Sprite)EditorGUILayout.ObjectField("Background Image",
                _currentNarrativeSequence.BackgroundSprite, typeof(Sprite), false);
            var backgroundSpriteChanged = _currentNarrativeSequence.BackgroundSprite != previousSprite;
```

This handles the background image I mentioned previously. Once again, no scene-only objects please!

```cs
            GUILayout.Label("Music files", EditorStyles.label);

            bool musicCollectionModified = false;

            for (int index = 0; index < _currentNarrativeSequence.MusicFiles.Count; index++)
            {
                var previousClip = _currentNarrativeSequence.MusicFiles[index];
                _currentNarrativeSequence.MusicFiles[index] =
                    (AudioClip)EditorGUILayout.ObjectField($"Music {index}",
                        _currentNarrativeSequence.MusicFiles[index], typeof(AudioClip), false);

                if (previousClip != _currentNarrativeSequence.MusicFiles[index])
                {
                    musicCollectionModified = true;
                }
            }
            
            if (GUILayout.Button("Add new"))
            {
                _currentNarrativeSequence.MusicFiles.Add(null);
                musicCollectionModified = true;
            }

            if (GUILayout.Button("Remove last") && _currentNarrativeSequence.MusicFiles.Count > 0)
            {
                _currentNarrativeSequence.MusicFiles.RemoveAt(_currentNarrativeSequence.MusicFiles.Count - 1);
                musicCollectionModified = true;
            }
```

This handles adding and removing references to music tracks to the `NarrativeSequence` object. It will check the collection and update the UI in real-time to ensure that the correct list of assets is always shown. This code also allows you to add and remove music assets. The sound effect code works the exact same way, so I'll just skip that....

```cs
            GUILayout.Label("Source Script", EditorStyles.label);
            var previousSourceScript = _currentNarrativeSequence.SourceScript;
            _currentNarrativeSequence.SourceScript = GUILayout.TextArea(_currentNarrativeSequence.SourceScript,
                GUILayout.ExpandWidth(true), GUILayout.ExpandHeight(true));
            var sourceScriptChanged = _currentNarrativeSequence.SourceScript != previousSourceScript;
```

This handles updating the source script of the `NarrativeSequence`. That's all this does, really? We use a text area to make that huge field that end-users can fill with their narrative code. Due to the layout settings we are passing in, it will also expand with the size of the window. Very handy for larger pieces of source code.

```cs
            _currentChoicesValue = GUILayout.Toggle(_currentChoicesValue, "Has Choices");

            bool nextNarrativeSequenceModified = false;
            if (_currentChoicesValue)
            {
```

Does this sequence support choices? If so, render the choices settings. The choices settings code is, again, similar to the music and sound effects code. I'll just skip over pasting this as well, due to the similarities.

```cs
            else
            {
                _currentNarrativeSequence.Choices.Clear();
                var previousSequence = _currentNarrativeSequence.NextSequence;
                _currentNarrativeSequence.NextSequence = (NarrativeSequence)EditorGUILayout.ObjectField(
                    "Next Narrative Sequence", _currentNarrativeSequence.NextSequence, typeof(NarrativeSequence),
                    false);

                if (previousSequence != _currentNarrativeSequence.NextSequence)
                {
                    nextNarrativeSequenceModified = true;
                }
            }
```

If it doesn't support choices, we need to render a different UI instead. This object field allows you to "daisy chain" sequences together - rather like a single-linked list! This means whatever is reading the sequence can only traverse one way.


```cs
            bool compiledScriptChanged = false;
            if (GUILayout.Button("Save Source Script"))
            {
                compiledScriptChanged = true;
                var listOfStuff = _interpreter.Interpret(_currentNarrativeSequence.SourceScript, _predefinedVariables, _currentNarrativeSequence.MusicFiles.Count, _currentNarrativeSequence.SoundEffectFiles.Count);
                _currentNarrativeSequence.CharacterDialoguePairs = listOfStuff;

                if (_currentNarrativeSequence.Choices is not null && _predefinedVariables is not null)
                {
                    foreach (var choice in _currentNarrativeSequence.Choices)
                    {
                        choice.ChoiceText =
                            _interpreter.ResolvePredefinedVariables(choice.ChoiceText, _predefinedVariables);
                    }
                }
            }
```

And now, we render the button that lets users compile their source script into scriptable data by calling `Interpret`! We also resolve any additional predefiend variables here on the choice text ahead of time. It's a workaround to a problem that never even came up - using variables in choice button labels. Oh well!

```cs
            if (sourceScriptChanged || musicCollectionModified || soundEffectCollectionModified || nextNarrativeSequenceModified ||
                compiledScriptChanged || backgroundSpriteChanged)
            {
                EditorUtility.SetDirty(_currentNarrativeSequence);
            }
```

Hey...so...after all that...did the scriptable change at all? Yes? No? Well, if it did, at least one of these booleans will be true. We tell unity to overwrite the asset on the next save by calling `SetDirty` on the asset. This step is very important, as without this your changes will forever and always be lost to the aether. Remember, Unity doesn't auto save your work. Save, save, save!

```cs
        [MenuItem("Window / ACH Narrative Driver / Narrative Sequence Editor")]
        public static void ShowEditor()
        {
            var window = EditorWindow.GetWindow<NarrativeSequenceEditor>(title: "Narrative Sequence Editor");
            window.minSize = new Vector2(500, 500);
        }
```

This method is called when it is time to create the window. The attribute describes under what Unity menu to put the tool under, and also gives the window a default size, too. Remember, this window can be docked to the main Unity edtior, too. Very slick...for such an ancient and unloved API.

## What I learned

Writing tools for a game jam was something I never really considered doing until now. However, the length of the jam and my other (very awesome) teammates meant I got to experiment a bit with things. I think if I was to continue working on this tool, I'd probably want to rewrite the editor Window using UXML once I learn how that works, and also probably swap out the narrative language for [NovelRT's narrative scripting language, Fabulist](https://github.com/NovelRT/Fabulist), since its a far more terse and complete narrative scripting language. I could then bolt the additional Unity-related things on the top, and I think it would be a nicer experience overall.

...also probably more efficient too, looking at that `Interpret` method again. Oof. Oh well, just game jam things I suppose. I'll probably be writing a follow-up post on the Unity tooling side of things in the semi-near future, where I hopefully rewrite the UI to UXML. Hopefully UXML does a better job than these Unity 4 APIs. They still work...but...they feel very unfinished.

## Game Jam Bundle!

If what I have been working on has interested you, you can get the game as part of a small bundle or $3 as part of an autistic charity fundraiser. I'd really appreciate it if you spent the money for a good cause, as this is something that's very personal to me. Plus, the game is not finished, we have more content to add, so definitely more to look forward to from a gameplay perspecrtive!

(Insert link when its up)