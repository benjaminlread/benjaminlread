---
layout: page
title: VBA Macros for Microsoft Word / Annotation
---
When I write documents in Word, I like to mark them up with notes that are flagged with customized character styles and usually enclosed in brackets. The VBA macros on this page were designed to help manipulate such notes. They make it easy, for example, to create one, select one, jump to the next one, and convert one into regular text. When a clean version of the document is needed, another macro can delete them all instantly.

True, Word has its own annotation features like comments and hidden text. But I like this way better.

<div style="text-align:center" markdown="1">
![annotation illustration](WordDocumentWithNotes.gif)
</div>

In my experience, using the keyboard instead of the mouse (as much as possible) is essential for avoiding forearm and wrist pain. It is also faster. Most of these macros are really only useful if [assigned](http://word.mvps.org/faqs/customization/AsgnCmdOrMacroToHotkey.htm) to keystrokes. General information on Word macros [here](http://word.mvps.org/FAQs/MacrosVBA.htm).

Some of the macros assume that you have character styles named, e.g., "Flag 1". This [document](SampleDocument-Annotation.docm) contains sample styles by way of illustration. Note that styles, too, can be assigned directly to keystrokes, but in some cases I use macros instead (see "Set style to 'Flag 1'").

My general macros for Word are [here](macros-word-general.html).

## Select text within nearest brackets from insertion point

(not including the brackets themselves)  
Suggested keystroke: Ctrl-[

```
Sub SelectToBracketsExclusive()
  With Selection.Find
    .ClearFormatting
    .Text = "["
    .Forward = False
    .MatchWildcards = False
    .Wrap = wdFindStop
    .Execute
  End With
  Selection.Extend
  With Selection.Find
    .Text = "]"
    .Forward = True
    .Execute
    .Text = ""
  End With
  Selection.MoveStart Unit:=wdCharacter, Count:=1
  Selection.MoveEnd Unit:=wdCharacter, Count:=-1
End Sub
```

## Select text within nearest brackets from insertion point

(including the brackets themselves)  
Suggested keystroke: Ctrl-Shift-[

```
Sub SelectToBracketsInclusive()
  With Selection.Find
    .ClearFormatting
    .Text = "["
    .Forward = False
    .MatchWildcards = False
    .Wrap = wdFindStop
    .Execute
  End With
  Selection.Extend
  With Selection.Find
    .Text = "]"
    .Forward = True
    .Execute
    .Text = ""
  End With
End Sub
```

## Add brackets around selection

Suggested keystroke: Ctrl-]

```
Sub AddBracketsAroundSelection()
  ' Simply inserts adjacent brackets if nothing is selected
  If Selection.Type = wdSelectionIP Then
    Selection.InsertBefore "["
    Selection.InsertAfter "]"
    Selection.MoveStart Unit:=wdCharacter, Count:=1
    Selection.Collapse Direction:=wdCollapseStart
  Else
    ' Shrinks selection to exclude leading or trailing spaces
    ' Also excludes trailing paragraph break
    While Selection.Characters.First = " "
      Selection.MoveStart Unit:=wdCharacter, Count:=1
    Wend
    While Selection.Characters.Last = " " Or Selection.Characters.Last = vbCr
      Selection.MoveEnd Unit:=wdCharacter, Count:=-1
    Wend
    Selection.InsertBefore "["
    Selection.InsertAfter "]"
    ' Brackets shed any character style acquired from selection
    Selection.Characters.First.Font.Reset
    Selection.Characters.Last.Font.Reset
  End If
End Sub
```

## Clear formatting from text within brackets

(and delete the brackets themselves)  
Suggested keystroke: Ctrl-Shift-]

```
Sub ClearTextWithinBrackets()
  ' Removes char style / formatting within brackets, deletes the brackets
  Application.ScreenUpdating = False
  ActiveDocument.Bookmarks.Add Name:="LastPosition"
  ' First step: select text within brackets
  With Selection.Find
    .ClearFormatting
    .Text = "["
    .Forward = False
    .MatchWildcards = False
    .Wrap = wdFindStop
    .Execute
  End With
  Selection.Extend
  With Selection.Find
    .Text = "]"
    .Forward = True
    .Execute
    .Text = ""
  End With
  ' Second step: clear formatting in selected text and collapse selection
  With Selection
    .ClearFormatting
    .Collapse Direction:=wdCollapseStart
  End With
  ' Third step: delete the initial bracket
  With Selection
    .MoveEnd Unit:=wdCharacter, Count:=1
    .Text = ""
  End With
  ' Fourth step: find and delete the final bracket
  With Selection.Find
    .Text = "]"
    .Forward = True
    .Replacement.Text = ""
    .Execute Replace:=wdReplaceOne
    .Text = ""
  End With
  Selection.GoTo What:=wdGoToBookmark, Name:="LastPosition"
  ActiveDocument.Bookmarks(Index:="LastPosition").Delete
  Application.ScreenUpdating = True
End Sub
```

## Delete text in brackets, and the brackets

(put insertion point inside the note first)  
Suggested keystroke: Alt-Ctrl-Shift-]

```
Sub SelectToBracketsDelete()
  With Selection.Find
    .ClearFormatting
    .Text = "["
    .Forward = False
    .Wrap = wdFindStop
    .Execute
  End With
  Selection.Extend
  With Selection.Find
    .Text = "]"
    .Forward = True
    .Execute
    .Text = ""
  End With
  ' By not using "delete" method we get around Word's habit of adding spaces
  Selection.Text = ""
End Sub
```

## Select to boundaries of style

Suggested keystroke: Ctrl-\

```
Sub SelectToStyleBoundaries()
  Dim StyleName As Variant
  Application.ScreenUpdating = False
  Set StyleName = Selection.Style
  While Selection.Characters.First.Previous.Style = StyleName
    Selection.MoveStart Unit:=wdCharacter, Count:=-1
  Wend
  While Selection.Characters.Last.Next.Style = StyleName
    Selection.MoveEnd Unit:=wdCharacter, Count:=1
  Wend
  Application.ScreenUpdating = True
End Sub
```

## Select to boundaries of style, then delete

Suggested keystroke: Ctrl-Shift-\

```
Sub SelectToStyleBoundariesAndDelete()
  Dim StyleName As Variant
  Application.ScreenUpdating = False
  Set StyleName = Selection.Style
  While Selection.Characters.First.Previous.Style = StyleName
    Selection.MoveStart Unit:=wdCharacter, Count:=-1
  Wend
  While Selection.Characters.Last.Next.Style = StyleName
    Selection.MoveEnd Unit:=wdCharacter, Count:=1
  Wend
  Application.ScreenUpdating = True
  Selection.Delete
End Sub
```

## Set style to "Flag 1"

Suggested keystroke: Alt-Shift-F

```
Sub SetStyleToFlag1()
  ' If no text selected, select text within the nearest brackets
  If Selection.Start = Selection.End Then SelectToBracketsExclusive
  Selection.Style = ActiveDocument.Styles("Flag 1")
End Sub
```

## Find next instance of "Flag 1" style

Suggested keystroke: Ctrl-Shift-F

```
Sub FindNextInstanceOfFlag1()
  Application.ScreenUpdating = False
  ' First, moves to end of Flag1 range if insertion point is in Flag1
  While Selection.Characters.Last.Next.Style = "Flag 1"
    Selection.MoveEnd Unit:=wdCharacter, Count:=1
  Wend
  Selection.Collapse Direction:=wdCollapseEnd
  With Selection.Find
  .ClearFormatting
  .Forward = True
  .Text = ""
  .MatchWildcards = False
  .Style = ActiveDocument.Styles("Flag 1")
  .Execute
  .ClearFormatting
  End With
  Selection.Collapse Direction:=wdCollapseStart
  Application.ScreenUpdating = True
End Sub
```

## Delete all text with "Flag 1" style

(This affects the entire document; I do not assign it to a keystroke.)

```
Sub DeleteTextWithFlag1Style()
  ' First, confirm that user wants to do this
  Dim varResponse As Variant
  varResponse = MsgBox("Delete all text styled Flag 1?", vbYesNo, "Selection")
  If varResponse <> vbYes Then Exit Sub
  Dim oRng As Range
  Set oRng = ActiveDocument.Range(Start:=0, End:=0)
  With oRng.Find
    ' Preparation
    .ClearFormatting
    .Replacement.ClearFormatting
    .Text = ""
    .Replacement.Text = ""
    .MatchWildcards = False
    .Wrap = wdFindContinue
    ' Remove Flag 1 text
    .Style = ActiveDocument.Styles("Flag 1")
    .Execute Replace:=wdReplaceAll
    ' Clean out empty bracket pairs that once held flagged text
    .ClearFormatting
    .Text = "[]"
    .Execute Replace:=wdReplaceAll
    ' Clean up
    .Text = ""
    .Wrap = wdFindAsk
  End With
End Sub
```

## Delete all flagged text

(This affects the entire document; I do not assign it to a keystroke.)

```
Sub DeleteAllFlaggedText()
  ' First, confirm that user wants to do this
  Dim varResponse As Variant
  varResponse = MsgBox("Delete all flagged text?", vbYesNo, "Selection")
  If varResponse <> vbYes Then Exit Sub
  Dim oRng As Range
  Set oRng = ActiveDocument.Range(Start:=0, End:=0)
  With oRng.Find
    ' Preparation
    .ClearFormatting
    .Replacement.ClearFormatting
    .Text = ""
    .Replacement.Text = ""
    .MatchWildcards = False
    .Wrap = wdFindContinue
    ' Remove Flag 1 text
    .Style = ActiveDocument.Styles("Flag 1")
    .Execute Replace:=wdReplaceAll
    ' Remove Flag 2 text
    .Style = ActiveDocument.Styles("Flag 2")
    .Execute Replace:=wdReplaceAll
    ' Remove Flag 3 text
    .Style = ActiveDocument.Styles("Flag 3")
    .Execute Replace:=wdReplaceAll
    ' Clean out empty bracket pairs that once held flagged text
    .ClearFormatting
    .Text = "[]"
    .Execute Replace:=wdReplaceAll
    ' Clean up
    .Text = ""
    .Wrap = wdFindAsk
  End With
End Sub
```

Comments welcome. Updated November 20, 2013.
