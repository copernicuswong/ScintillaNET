# ScintillaNET

ScintillaNET is a Windows Forms control, wrapper, and bindings for the versatile [Scintilla](http://www.scintilla.org/) source code editing component.

> "As well as features found in standard text editing components, Scintilla includes features especially useful when editing and debugging source code. These include support for syntax styling, error indicators, code completion and call tips. The selection margin can contain markers like those used in debuggers to indicate breakpoints and the current line. Styling choices are more open than with many editors, allowing the use of proportional fonts, bold and italics, multiple foreground and background colours and multiple fonts." � [scintilla.org](http://www.scintilla.org/)

ScintillaNET can also be used with WPF using the <a href="https://msdn.microsoft.com/en-us/library/ms751761.aspx">WindowsFormsHost</a>.

### Project Status

ScintillaNET is currently in active development and is not considered ready for general use.

## Background

This project is a rewrite of the [ScintillaNET project hosted at CodePlex](http://scintillanet.codeplex.com/) and maintained by myself and others. After many years of contributing to that project I decided to think differently about the API we had created and felt I could make better one if I was willing to go back to a blank canvas. Thus, this project is the spiritual successor to the original ScintillaNET but has been written from scratch.

### First Class Characters

One of the issues that ScintillaNET has historically suffered from is the fact that the native Scintilla control operates on bytes, not characters. Prior versions of ScintillaNET did not account for this, and when you're dealing with Unicode, [one byte doesn't always equal one character](http://www.joelonsoftware.com/articles/Unicode.html). The result was an API that sometimes expected byte offsets and at other times expected character offsets. Sometimes things would work as expected and other times random failures and out-of-range exceptions would occur.

No more. **One of the major focuses of this rewrite was to give ScintillaNET an understanding of Unicode from the ground up.** Every API now consistently works with character-based offsets and ranges just like .NET developers expect. Internally we maintain a mapping of character to byte offsets (and vice versa) and do all the translation for you so you never need to worry about it. No more out-of-range exceptions. No more confusion. No more pain. [It just works](http://en.wikipedia.org/wiki/List_of_Apple_Inc._slogans).

### One Library

The second most popular ScintillaNET issue was confusion distributing the ScintillaNET DLL and its native component, the SciLexer DLL. ScintillaNET is a wrapper. Without the SciLexer.dll containing the core Scintilla functionality it is nothing. As a native component, SciLexer.dll has to be compiled separately for 32 and 64-bit versions of Windows. So it was actually three DLLs that developers had to ship with their applications.

This proved a pain point because developers often didn't want to distribute so many libraries or wanted to place them in alternate locations which would break the DLL loading mechanisms used by PInvoke and ScintillaNET. It also causes headaches during design-time in Visual Studio for the same reasons.

To address this ScintillaNET now embeds a 32 and 64-bit version of SciLexer.dll in the ScintillaNET DLL. **Everything you need to run ScintillaNET in one library.** In addition to soothing the pain mentioned above this now makes it possible for us to create a ScintillaNET NuGet package.

### Keeping it Consistent

Another goal of the rewrite was to accept the original Scintilla API for what it is and not try to coerce it into a .NET-style API when it should not or could not be. A good example of this is how ScintillaNET uses indexers to access lines, but not treat them as a .NET collection. Lines in a Scintilla control are not items in a collection. There is no API to Add, Insert, or Remove a line in Scintilla and thus we don't try to create one in ScintillaNET. These deviations from .NET convention are rare, but are done to keep any native Scintilla documentation relevant to the managed wrapper and to avoid situations where trying to force the original API into a more familiar one is more detrimental than helpful.

*NOTE: This is not to say that ScintillaNET cannot add, insert, or remove lines. Those operations, however, are handled as text changes, not line changes.*

## Conventions

ScintillaNET is fully documented, but since there has been such an effort made to keep the ScintillaNET API consist with the native Scintilla API we encourage you to continue using [their documentation](http://www.scintilla.org/scintilladoc.html) to fill in any gaps you find in ours.

Generally speaking, their API will map to ours in the following ways:

+ A call that has an associated 'get' and 'set' such as `SCI_GETTEXT` and `SCI_SETTEXT(value)`, will map to a similarly named property such as `Text`.
+ A call that requires a number argument to access an item in a 'collection' such as `SCI_INDICSETSTYLE(indicatorNumber, ...)` or `SCI_STYLEGETSIZE(styleNumber, ...)`, will be accessed through an indexer such as `Indicators[0].Style` or `Styles[0].Size`.

The native Scintilla control has a habit of clamping input values to within acceptable ranges rather than throwing exceptions and so we've kept that behavior in ScintillaNET. For example, the `GotoPosition` method requires a character `position` argument. If that value is less than zero or past the end of the document it will be clamped to either `0` or the `TextLength` rather than throw an `OutOfRangeException`. This tends to result in less exceptions, but the same desired outcome.

## Recipes

1. [Basic Text Retrieval and Modification](#basic-text)
  1. [Retrieving Text](#retrieve-text)
  2. [Insert, Append, Delete](#modify-text)
2. [Syntax Highlighting](#syntax-highlighting)
  1. [Selecting a Lexer](#lexer)
  2. [Defining Styles](#styles)
  3. [Setting Keywords](#keywords)
  4. [Complete Recipe](#syntax-highlighting-recipe)
3. [Intercepting Inserted Text](#insert-check)
4. [Displaying Line Numbers](#line-numbers)
5. [Zooming](#zooming)
6. [Updating Dependent Controls](#update-ui)
7. [Find and Highlight Words](#find-highlight)
8. [View Whitespace](#whitespace)
9. [Increase Line Spacing](#line-spacing)
10. [Bookmark Lines](#bookmarks)
11. [Using a Custom SciLexer.dll](#scilexer)

### <a name="basic-text"></a>Basic Text Retrieval and Modification

At its most basic level ScintillaNET is a text editor. The following recipes demonstrate basic text I/O.

#### <a name="retrieve-text"></a>Retrieving Text

The `Text` property is the obvious choice if you want to get a string that represents all the text currently in a Scintilla control. Internally the Scintilla control will copy the entire contents of the document into a new string. Depending on the amount of text in the editor, this could be an expensive operation. Scintilla is designed to work efficiently with large amounts of text, but using the `Text` property to retrieve only a substring negates that.

Instead, it's usually best to identify the range of text you're interested in (through search, selection, or some other means) and use the `GetTextRange` method to copy only what you need into a string:

```cs
// Get the first 256 characters of the document
var text = scintilla.GetTextRange(0, Math.Min(256, scintilla.TextLength));
Console.WriteLine(text);
```

#### <a name="modify-text"></a>Insert, Append, and Delete

Modifications usually come in the form of insert, append, and delete operations. As was discussed above, using the `Text` property to make a small change in the document contents is highly inefficient. Instead try one of the following options:

```cs
scintilla.Text = "Hello";
scintilla.AppendText(" World"); // 'Hello' -> 'Hello World'
scintilla.DeleteRange(0, 5); // 'Hello World' -> ' World'
scintilla.InsertText(0, "Goodbye"); // ' World' -> 'Goodbye World'
```

*NOTE: It may help to think of a Scintilla control as a `StringBuilder`.*

### <a name="syntax-highlighting"></a>Syntax Highlighting

Far and away the most popular use for Scintilla is to display and edit source code. Out-of-the-box, Scintilla comes with syntax highlighting support for over 100 different languages. Chances are that the language you want to edit is already supported.

#### <a name="lexer"></a>Selecting a Lexer

A language processor is referred to as a 'lexer' in Scintilla. Without going too much into parser theory, it's important to know that a lexer performs [lexical analysis](http://en.wikipedia.org/wiki/Lexical_analysis) of a language, not [syntatic analysis (parsing)](http://en.wikipedia.org/wiki/Parsing). In short, this means that the language support provided by Scintilla is enough to break the text into tokens and provide syntax highlighting, but not interpret what those tokens mean or whether they form an actual program. The distinction is important because developers using Scintilla often want to know how they can highlight incorrect code. Scintilla doesn't do that. If you want more than basic syntax highlighting, you'll need to couple Scintilla with a parser or even a background compiler.

To inform Scintilla what the current language is you must set the `Lexer` property to the appropriate enum value. In some cases multiple languages share the same `Lexer` enumeration value because these language share the same lexical grammar. For example, the `Cpp` lexer not only provides language support for C++, but all for C, C#, Java, JavaScript and others�because they are lexically similar. In our example we want to do C# syntax highlighting so we'll use the `Cpp` lexer.

```cs
scintilla.Lexer = Lexer.Cpp;
```

#### <a name="styles"></a>Defining Styles

The process of doing syntax highlighting in Scintilla is referred to as styling. When the text is styled, runs of text are assigned a numeric style definition in the `Styles` collection. For example, keywords may be assigned the style definition `1`, while operators may be assigned the definition `2`. It's entire up to the lexer how this is done. Once done, however, you are then free to determine what style `1` or `2` look like. The lexer assigns the styles, but you define the style appearance. To make it easier to know which styles definitions a lexer will use, the `Style` object contains static constants that coincide with each `Lexer` enumeration value. For example, if we were using the `Cpp` lexer and wanted to set the style for single-line comments (//...) we would use the `Style.Cpp.CommentLine` constant to set the appropriate style in the `Styles` collection:

```cs
scintilla.Styles[Style.Cpp.CommentLine].Font = "Consolas";
scintilla.Styles[Style.Cpp.CommentLine].Size = 10;
scintilla.Styles[Style.Cpp.CommentLine].ForeColor = Color.FromArgb(0, 128, 0); // Green
```

To set the string style we would:

```cs
scintilla.Styles[Style.Cpp.String].Font = "Consolas";
scintilla.Styles[Style.Cpp.String].Size = 10;
scintilla.Styles[Style.Cpp.String].ForeColor = Color.FromArgb(163, 21, 21); // Red
```

To set the styles for number tokens we would do the same thing using the `Style.Cpp.Number` constant. For operators, we would use `Style.Cpp.Operator`, and so on.

If you use your imagination you will begin to see how doing this for each possible lexer token could be tedious. There is a lot of repetition. To reduce the amount of code you have to write Scintilla provides a way of setting a single style and then applying its appearance to every style in the collection. The general process is to:

* Reset the `Default` style using `StyleResetDefault`.
* Configure the `Default` style with all common properties.
* Use the `StyleClearAll` method to apply the `Default` style to all styles.
* Set any individual style properties

Using that time saving approach, we can set the appearance of our C# lexer styles like so:

```cs
// Configuring the default style with properties
// we have common to every lexer style saves time.
scintilla.StyleResetDefault();
scintilla.Styles[Style.Default].Font = "Consolas";
scintilla.Styles[Style.Default].Size = 10;
scintilla.StyleClearAll();

// Configure the CPP (C#) lexer styles
scintilla.Styles[Style.Cpp.Default].ForeColor = Color.Silver;
scintilla.Styles[Style.Cpp.Comment].ForeColor = Color.FromArgb(0, 128, 0); // Green
scintilla.Styles[Style.Cpp.CommentLine].ForeColor = Color.FromArgb(0, 128, 0); // Green
scintilla.Styles[Style.Cpp.CommentLineDoc].ForeColor = Color.FromArgb(128, 128, 128); // Gray
scintilla.Styles[Style.Cpp.Number].ForeColor = Color.Olive;
scintilla.Styles[Style.Cpp.Word].ForeColor = Color.Blue;
scintilla.Styles[Style.Cpp.Word2].ForeColor = Color.Blue;
scintilla.Styles[Style.Cpp.String].ForeColor = Color.FromArgb(163, 21, 21); // Red
scintilla.Styles[Style.Cpp.Character].ForeColor = Color.FromArgb(163, 21, 21); // Red
scintilla.Styles[Style.Cpp.Verbatim].ForeColor = Color.FromArgb(163, 21, 21); // Red
scintilla.Styles[Style.Cpp.StringEol].BackColor = Color.Pink;
scintilla.Styles[Style.Cpp.Operator].ForeColor = Color.Purple;
scintilla.Styles[Style.Cpp.Preprocessor].ForeColor = Color.Maroon;
```

#### <a name="keywords"></a>Setting Keywords

The last thing we need to do to provide syntax highlighting is to inform the lexer what the language keywords and identifiers are. Since languages can often add keywords year after year, or because a lexer may sometimes be used for more than one language, it makes sense to make the keyword list configurable.

Since each Scintilla lexer is like a program until itself the number of keyword sets and the definition of each one varies from lexer to lexer. To determine what keyword sets a lexer supports you can call the `DescribeKeywordSets` method. This prints a human readable explanation of how many sets the current `Lexer` supports and what each means:

```cs
scintilla.Lexer = Lexer.Cpp;
Console.WriteLine(scintilla.DescribeKeywordSets());

// Outputs:
// Primary keywords and identifiers
// Secondary keywords and identifiers
// Documentation comment keywords
// Global classes and typedefs
// Preprocessor definitions
// Task marker and error marker keywords
```

Based on the output of `DescribeKeywordSets` I can determine that the first two sets are what I'm interested in for supporting general purpose C# syntax highlighting. To set a set of keywords you call the `SetKeywords` method. What 'primary' and 'secondary' means in the keyword set description is up to a bit of interpretation, but I'll break it down so that primary keywords are C# language keywords and secondary keywords are known .NET types. To set those I would call:

```cs
scintilla.SetKeywords(0, "abstract as base break case catch checked continue default delegate do else event explicit extern false finally fixed for foreach goto if implicit in interface internal is lock namespace new null object operator out override params private protected public readonly ref return sealed sizeof stackalloc switch this throw true try typeof unchecked unsafe using virtual while");
scintilla.SetKeywords(1, "bool byte char class const decimal double enum float int long sbyte short static string struct uint ulong ushort void");
```

*NOTE: Keywords in a keyword set can be separated by any combination of whitespace (space, tab, '\r', '\n') characters.*

#### <a name="syntax-highlighting-recipe"></a>Complete Recipe

The complete recipe below will give you C# syntax highlighting using colors roughly equivalent to the Visual Studio defaults.

```cs
// Configuring the default style with properties
// we have common to every lexer style saves time.
scintilla.StyleResetDefault();
scintilla.Styles[Style.Default].Font = "Consolas";
scintilla.Styles[Style.Default].Size = 10;
scintilla.StyleClearAll();

// Configure the CPP (C#) lexer styles
scintilla.Styles[Style.Cpp.Default].ForeColor = Color.Silver;
scintilla.Styles[Style.Cpp.Comment].ForeColor = Color.FromArgb(0, 128, 0); // Green
scintilla.Styles[Style.Cpp.CommentLine].ForeColor = Color.FromArgb(0, 128, 0); // Green
scintilla.Styles[Style.Cpp.CommentLineDoc].ForeColor = Color.FromArgb(128, 128, 128); // Gray
scintilla.Styles[Style.Cpp.Number].ForeColor = Color.Olive;
scintilla.Styles[Style.Cpp.Word].ForeColor = Color.Blue;
scintilla.Styles[Style.Cpp.Word2].ForeColor = Color.Blue;
scintilla.Styles[Style.Cpp.String].ForeColor = Color.FromArgb(163, 21, 21); // Red
scintilla.Styles[Style.Cpp.Character].ForeColor = Color.FromArgb(163, 21, 21); // Red
scintilla.Styles[Style.Cpp.Verbatim].ForeColor = Color.FromArgb(163, 21, 21); // Red
scintilla.Styles[Style.Cpp.StringEol].BackColor = Color.Pink;
scintilla.Styles[Style.Cpp.Operator].ForeColor = Color.Purple;
scintilla.Styles[Style.Cpp.Preprocessor].ForeColor = Color.Maroon;
scintilla.Lexer = Lexer.Cpp;

// Set the keywords
scintilla.SetKeywords(0, "abstract as base break case catch checked continue default delegate do else event explicit extern false finally fixed for foreach goto if implicit in interface internal is lock namespace new null object operator out override params private protected public readonly ref return sealed sizeof stackalloc switch this throw true try typeof unchecked unsafe using virtual while");
scintilla.SetKeywords(1, "bool byte char class const decimal double enum float int long sbyte short static string struct uint ulong ushort void");
```

### <a name="insert-check"></a>Intercepting Inserted Text

There are numerous events to inform you of when text has changed. In addition to the `TextChanged` event provided by almost all Windows Forms controls, Scintilla also provides events for `Insert`, `Delete`, `BeforeInsert`, and `BeforeDelete`. By using these events you can trigger other changes in your application.

These events are all read-only, however. Changes made by a user to the text can be observed, but not modified�with one exception. The `InsertCheck` event occurs before text is inserted (and earlier than the `BeforeInsert` event) and is provided for the express purpose of giving you an option to modify the text being inserted. This can be used to simply cancel/prevent unwanted user input. Or in more advanced situations, it could be used to replace user input.

The following code snippet illustrates how you might handle the `InsertCheck` event to transform user input to HTML encode input:

```cs
private void scintilla_InsertCheck(object sender, InsertCheckEventArgs e)
{
    e.Text = WebUtility.HtmlEncode(e.Text);
}
```

### <a name="line-numbers"></a>Displaying Line Numbers

Someone new to Scintilla might wonder why displaying line numbers gets its own recipe when one would assume it's as simple as flipping a single Boolean property. Well it's not. The subject of line numbers touches on the much larger subject of margins. In Scintilla there can be up to five margins (0 through 4) on the left edge of the control, of which, line numbers is just one of those. By convention Scintilla sets the `Margin.Type` property of margin 0 to `MarginType.Number`, making it the de facto line number margin. Any margin can display line numbers though if its `Type` property is set to `MarginType.Number`. Scintilla also hides line numbers by default by setting the `Width` of margin 0 to zero. To display the default line number margin, increase its width:

```cs
scintilla.Margins[0].Width = 16;
```

You'll quickly find, however, that once you reach lines numbers in the 100's range the width of your line number margin is no longer sufficient. Scintilla doesn't automatically increase or decrease the width of a margin�including a line number margin. Why? It goes back to the fact that a margin could display line numbers or it could display something else entirely where dynamically growing and shrinking would be an undesirable trait.

The line number margin can be made to grow or shrink dynamically, it just requires a little extra code your part. In the recipe below we handle the `TextChanged` event so we can know when the number of lines changes. (*There are several other events we could use to determine the content has changed, but `TextChanged` will do just fine.*) Then we measure the width of the last line number in the document (or equivalent number of characters) using the `TextWidth` method. Finally, set the `Width` of the line number margin. Some caching of the calculation is thrown in for good measure since the number of lines will change far less than the `TextChanged` event will fire.

```cs
private int maxLineNumberCharLength;
private void scintilla_TextChanged(object sender, EventArgs e)
{
    // Did the number of characters in the line number display change?
    // i.e. nnn VS nn, or nnnn VS nn, etc...
    var maxLineNumberCharLength = scintilla.Lines.Count.ToString().Length;
    if (maxLineNumberCharLength == this.maxLineNumberCharLength)
        return;

    // Calculate the width required to display the last line number
    // and include some padding for good measure.
    const int padding = 2;
    scintilla.Margins[0].Width = scintilla.TextWidth(Style.LineNumber, new string('9', maxLineNumberCharLength + 1)) + padding;
    this.maxLineNumberCharLength = maxLineNumberCharLength;
}
```

*NOTE: The color of the text displayed in a line number margin can be controlled via the `Style.LineNumber` style definition.*

### <a name="zooming"></a>Zooming

Scintilla can increase or decrease the size of the displayed text by a "zoom factor":

```cs
scintilla.ZoomIn(); // Increase
scintilla.ZoomOut(); // Decrease
scintilla.Zoom = 15; // "I like big 'text' and I cannot lie..."
```

*NOTE: The default key bindings set `CTRL+NUMPLUS` and `CTRL+NUMMINUS` to zoom in and zoom out, respectively.*

### <a name="update-ui"></a>Updating Dependent Controls

A common feature most developers wish to provide with their Scintilla-based IDEs is to indicate where the caret (i.e. cursor) is at all times by perhaps displaying its location in the status bar. The `UpdateUI` event is well suited to this. It is fired any time there is a change to text content or styling, the selection, or scroll positions and provides a way for identifying which of those changes caused the event to fire. This can be used to update any dependent controls or even synchronize the scrolling of one Scintilla control with another.

To display the current caret position and selection range in the status bar, try:

```cs
private void scintilla_UpdateUI(object sender, UpdateUIEventArgs e)
{
    if ((e.Change & UpdateChange.Selection) > 0)
    {
        // The caret/selection changed
        var currentPos = scintilla.CurrentPosition;
        var anchorPos = scintilla.AnchorPosition;
        toolStripStatusLabel.Text = "Ch: " + currentPos + " Sel: " + Math.Abs(anchorPos - currentPos);
    }
}
```

### <a name="find-highlight"></a>Find and Highlight Words

The following example will find all occurrences of the string specified (case-insensitive) and highlight them with a light-green indicator:

```cs
private void HighlightWord(string text)
{
    // Indicators 0-7 could be in use by a lexer
    // so we'll use indicator 8 to highlight words.
    const int NUM = 8;

    // Remove all uses of our indicator
    scintilla.Indicators.Current = NUM;
    scintilla.Indicators.ClearRange(0, scintilla.TextLength);

    // Update indicator appearance
    scintilla.Indicators[NUM].Style = IndicatorStyle.StraightBox;
    scintilla.Indicators[NUM].ForeColor = Color.Green;
    scintilla.Indicators[NUM].OutlineAlpha = 50;
    scintilla.Indicators[NUM].Alpha = 30;

    // Search the document
    scintilla.TargetStart = 0;
    scintilla.TargetEnd = scintilla.TextLength;
    scintilla.SearchFlags = SearchFlags.None;
    while (scintilla.SearchInTarget(text) != -1)
    {
        // Mark the search results with the current indicator
        scintilla.Indicators.FillRange(scintilla.TargetStart, scintilla.TargetEnd - scintilla.TargetStart);

        // Search the remainder of the document
        scintilla.TargetStart = scintilla.TargetEnd;
        scintilla.TargetEnd = scintilla.TextLength;
    }
}
```

This example also illustrates the "set-once, run-many" style API that Scintilla is know for. When performing a search, the `TargetStart` and `TargetEnd` properties are set to indicate the search range prior to calling `SearchInTarget`. The indicators API is similar. The `Indicators.Current` property is first set and then subsequent calls to `Indicators.ClearRange` and `Indicators.FillRange` make use of that value.

*NOTE: Indicators and styles can be used simultaneously.*

### <a name="whitespace"></a>View Whitespace

Scintilla has several properties for controlling the display and color of whitespace (space and tab characters). By default, whitespace is not visible. It can be made visible by setting the `ViewWhitespace` property. Since whitespace can be significant to some programming languages the default behavior is for the current lexer to set the color of whitespace. To override the default behavior the `SetWhitespaceForeColor` and `SetWhitespaceBackColor` methods can be used. To make whitespace visible and always display in an orange color (regardless of the current lexer), try:

```cs
// Display whitespace in orange
scintilla.WhitespaceSize = 2;
scintilla.ViewWhitespace = WhitespaceMode.VisibleAlways;
scintilla.SetWhitespaceForeColor(true, Color.Orange);
```

### <a name="line-spacing"></a>Increase Line Spacing

Feeling like your text is a little too cramped? Having a little extra space between lines can sometimes help the readability of code. In some text editors and IDEs you would need to use a different font which has a larger ascent and/or descent (I'm looking at you Visual Studio). In Scintilla you can increase the text ascent and decent independently of the font using the `ExtraAscent` and `ExtraDescent` properties.

```cs
// Increase line spacing
scintilla.ExtraAscent = 5;
scintilla.ExtraDescent = 5;
```

### <a name="bookmarks"></a>Bookmark Lines

This recipe shows how markers can be used to indicate bookmarked lines and iterate through them. Markers are symbols displayed in the left margins and are typically used for things like showing breakpoints, search results, the current line of execution, or in this case, bookmarks.

In the `Form.Load` event we'll prepare a margin to be used for our bookmarks and we'll configure one of the markers to use the `Bookmark` symbol:

```cs
private const int BOOKMARK_MARGIN = 1; // Conventionally the symbol margin
private const int BOOKMARK_MARKER = 3; // Arbitrary. Any valid index would work.

private void MainForm_Load(object sender, EventArgs e)
{
    var margin = scintilla.Margins[BOOKMARK_MARGIN];
    margin.Width = 16;
    margin.Sensitive = true;
    margin.Type = MarginType.Symbol;
    margin.Mask = Marker.MaskAll;
    margin.Cursor = MarginCursor.Arrow;

    var marker = scintilla.Markers[BOOKMARK_MARKER];
    marker.Symbol = MarkerSymbol.Bookmark;
    marker.SetBackColor(Color.DeepSkyBlue);
    marker.SetForeColor(Color.Black);
}
```

The margin is flagged as `Sensitive` so we can receive mouse click notifications from it (and because this margin can sometimes get emotional). The margin `Mask` is a way of restricting which marker symbols can appear in the margin. For the purposes of this example we'll configure it so that all marker symbols assigned to any given line will be displayed.

By handling the `MarginClick` event we can toggle a bookmark marker for each line:

```cs
private void scintilla_MarginClick(object sender, MarginClickEventArgs e)
{
    if (e.Margin == BOOKMARK_MARGIN)
    {
        // Do we have a marker for this line?
        const uint mask = (1 << BOOKMARK_MARKER);
        var line = scintilla.Lines[scintilla.LineFromPosition(e.Position)];
        if ((line.MarkerGet() & mask) > 0)
        {
            // Remove existing bookmark
            line.MarkerDelete(BOOKMARK_MARKER);
        }
        else
        {
            // Add bookmark
            line.MarkerAdd(BOOKMARK_MARKER);
        }
    }
}
```

The code above makes use of the `MarkerGet` method to get a bitmask indicating which markers are set for the current line. This mask is a 32-bit value where each bit corresponds to one of the 32 marker indexes. If marker `0` were set, bit `0` would be set. If marker `1` where set, bit `1` would be set and so on. To determine if our maker has been set for a line we would want to check if the `1 << 3` bit is set which is what the statement `(line.MarkerGet() & mask) > 0` does.

At this point, if you run the code you should be able to add and remove pretty blue bookmarks from document lines. To let a user jump between bookmarks we'll add 'Next' and 'Previous' buttons and wire-up their click events like this:

```cs
private void buttonPrevious_Click(object sender, EventArgs e)
{
    var line = scintilla.LineFromPosition(scintilla.CurrentPosition);
    var prevLine = scintilla.Lines[--line].MarkerPrevious(1 << BOOKMARK_MARKER);
    if (prevLine != -1)
        scintilla.Lines[prevLine].Goto();
}

private void buttonNext_Click(object sender, EventArgs e)
{
    var line = scintilla.LineFromPosition(scintilla.CurrentPosition);
    var nextLine = scintilla.Lines[++line].MarkerNext(1 << BOOKMARK_MARKER);
    if (nextLine != -1)
        scintilla.Lines[nextLine].Goto();
}
```

### <a name="scilexer"></a>Using a Custom SciLexer.dll

This is an advanced topic. On rare occasions you may wish to provide your own build of the SciLexer DLL used by ScintillaNET instead of the one we embed for you. By default ScintillaNET will use the embedded one but you can override the default behavior by calling `SetModulePath` prior to instantiating any Scintilla controls:

```cs
// Call prior to creating any controls
Scintilla.SetModulePath("AltSciLexer.dll");

// Load a control and get the SciLexer.dll info
var scintilla = new Scintilla();
var version = scintilla.GetVersionInfo();
```

The path provide can be an absolute or relative path to SciLexer.dll.

## License

The MIT License (MIT)

Copyright (c) 2015, Jacob Slusser, https://github.com/jacobslusser

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
