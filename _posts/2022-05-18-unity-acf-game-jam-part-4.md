---
title: "How one game jam made me create a custom in-engine compiler tool for a brand-new narrative scripting language I made up in less than an hour. Part 4: Editor tooling"
layout: posts
toc_label: "Table of Contents"
toc_icon: "heart"
excerpt: "Unity 4 API woes, cries for UXML, and much more!"
tags: c# unity tool tooling narrative scripting custom scriptableobject
header:
    teaser: assets/posts/unity-acf-game-jam/teaser.png
---

## A quick recap

This is a continuation of [Part three]({% post_url 2022-05-18-unity-acf-game-jam-part-3 %}) of this series. In this post I'll be talking about the actual bit that most people came for - the compiler(???) thing!

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

## What I learned from all this

Writing tools for a game jam was something I never really considered doing until now. However, the length of the jam and my other (very awesome) teammates meant I got to experiment a bit with things. I think if I was to continue working on this tool, I'd probably want to rewrite the editor Window using UXML once I learn how that works, and also probably swap out the narrative language for [NovelRT's narrative scripting language, Fabulist](https://github.com/NovelRT/Fabulist), since its a far more terse and complete narrative scripting language. I could then bolt the additional Unity-related things on the top, and I think it would be a nicer experience overall.

...also probably more efficient too, looking at that `Interpret` method again. Oof. Oh well, just game jam things I suppose. I'll probably be writing a follow-up post on the Unity tooling side of things in the semi-near future, where I hopefully rewrite the UI to UXML. Hopefully UXML does a better job than these Unity 4 APIs. They still work...but...they feel very unfinished.

## Game Jam Bundle!

If what I have been working on in this series has interested you, or you just want to play the finished game, you can get the game as part of a small bundle or $3 as part of an autistic charity fundraiser. I'd really appreciate it if you spent the money for a good cause, as this is something that's very personal to me. Plus, the game is not finished, we have more content to add, so definitely more to look forward to from a gameplay perspecrtive! [You can find the bundle here, and we would really appreciate your financial suport if you can spare it!](https://itch.io/b/1369/autism-acceptance-month-game-jam)