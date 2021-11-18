---
layout: page
title: VBA Macros for Microsoft Word / General
---
These macros are very simple but they make working in Word quite a bit snappier.

In my experience, using the keyboard instead of the mouse (as much as possible) is essential for avoiding forearm and wrist pain. It is also faster. Most of these macros are really only useful if [assigned](http://word.mvps.org/faqs/customization/AsgnCmdOrMacroToHotkey.htm) to keystrokes. General information on Word macros [here](http://word.mvps.org/FAQs/MacrosVBA.htm).

My macros for annotating Word documents are [here](macros-word-annotation.html).

## Delete current line

Suggested keystroke: Alt-Ctrl-Y

```
Sub DeleteCurrentLine()
  Selection.HomeKey Unit:=wdLine
  Selection.MoveEnd Unit:=wdLine
  Selection.Text = ""
End Sub
```

## Delete to end of line

Without also deleting a possible end-of-paragraph marker.  
Suggested keystroke: Alt-Delete

```
Sub DeleteToEndOfLine()
  Selection.MoveEnd Unit:=wdLine
  If Selection.Characters.Last = vbCr Then
    Selection.MoveEnd Unit:=wdCharacter, Count:=-1
  End If
  Selection.Text = ""
End Sub
```

## Insert QuickMark

Quickly mark a point in the document to return to later. WordPerfect had a feature like this, as does every decent text editor. I couldn't live without this. In fact, I keep two sets of these, QuickMark1 and QuickMark2.  
Suggested keystroke: Ctrl-Shift-Q

```
Sub QuickMarkInsert()
  If ActiveDocument.Bookmarks.Exists("QuickMark") = True Then
    ActiveDocument.Bookmarks("QuickMark").Delete
  End If
  With ActiveDocument.Bookmarks
    .Add Range:=Selection.Range, Name:="QuickMark"
  End With
End Sub
```

## Go to QuickMark

Suggested keystroke: Ctrl-Q

```
Sub QuickMarkGoTo()
  If ActiveDocument.Bookmarks.Exists("QuickMark") = True Then
    Selection.GoTo What:=wdGoToBookmark, Name:="QuickMark"
  End If
End Sub
```

## Delete QuickMark

Suggested keystroke: Alt-Ctrl-Shift-Q

```
Sub QuickMarkDelete()
  If ActiveDocument.Bookmarks.Exists("QuickMark") = True Then
    ActiveDocument.Bookmarks("QuickMark").Delete
  End If
End Sub
```

## Select paragraph

An alternative to the F8 x 4 then Escape method.  
Suggested keystroke: Ctrl-Shift-P

```
Sub SelectParagraph()
  Selection.StartOf Unit:=wdParagraph
  Selection.MoveEnd Unit:=wdParagraph
End Sub
```

## Go to next heading

Suggested keystroke: Alt-Shift-RightArrow

```
Sub GoToNextHeading()
  Selection.GoTo What:=wdGoToHeading, Which:=wdGoToNext
End Sub
```

## Go to previous heading

Suggested keystroke: Alt-Shift-LeftArrow

```
Sub GoToPreviousHeading()
  Selection.GoTo What:=wdGoToHeading, Which:=wdGoToPrevious
End Sub
```

## Go to next "Heading 1"

Suggested keystroke: Alt-Ctrl-Shift-RightArrow

```
Sub GoToNextHeading1()
  Application.ScreenUpdating = False
  ' First, moves to end of Heading 1 range if insertion point is in H1
  While Selection.Characters.Last.Next.Style = "Heading 1"
    Selection.MoveEnd Unit:=wdCharacter, Count:=1
  Wend
  Selection.MoveStart Unit:=wdCharacter, Count:=1
  Selection.Collapse Direction:=wdCollapseEnd
  With Selection.Find
    .ClearFormatting
    .Text = ""
    .MatchWildcards = False
    .Forward = True
    .Style = ActiveDocument.Styles("Heading 1")
    .Execute
    .ClearFormatting
  End With
  Selection.Collapse Direction:=wdCollapseStart
  Application.ScreenUpdating = True
End Sub
```

## Go to previous "Heading 1"

Suggested keystroke: Alt-Ctrl-Shift-LeftArrow

```
Sub GoToPreviousHeading1()
  Application.ScreenUpdating = False
  ' First, move to start of Heading 1 range if insertion point is in H1
  While Selection.Characters.First.Style = "Heading 1"
    Selection.MoveStart Unit:=wdCharacter, Count:=-1
  Wend
  Selection.Collapse Direction:=wdCollapseStart
  With Selection.Find
    .ClearFormatting
    .Text = ""
    .MatchWildcards = False
    .Forward = False
    .Style = ActiveDocument.Styles("Heading 1")
    .Execute
    .ClearFormatting
    .Forward = True
  End With
  Selection.Collapse Direction:=wdCollapseStart
  Application.ScreenUpdating = True
End Sub
```

## Scroll up one line in document

Without moving the insertion point. Equivalent to flicking the mouse control wheel up a notch.  
Suggested keystroke: Alt-PageUp

```
Sub ScrollUp()
  ActiveDocument.ActiveWindow.SmallScroll Up:=1
End Sub
```

## Scroll down one line in document

Without moving the insertion point. Equivalent to flicking the mouse control wheel down a notch.  
Suggested keystroke: Alt-PageDown

```
Sub ScrollDown()
  ActiveDocument.ActiveWindow.SmallScroll Down:=1
End Sub
```

## Zoom in

Courtesy of [Allen Wyatt](http://word.tips.net/T001734_Zooming_with_the_Keyboard.html)  
Suggested keystroke: Shift-Ctrl-PageDown

```
Sub ZoomIn()
    Dim ZP As Integer
    ZP = Int(ActiveWindow.ActivePane.View.Zoom.Percentage * 1.1)
    If ZP > 200 Then ZP = 200
    ActiveWindow.ActivePane.View.Zoom.Percentage = ZP
End Sub
```

## Zoom out

Courtesy of [Allen Wyatt](http://word.tips.net/T001734_Zooming_with_the_Keyboard.html)  
Suggested keystroke: Shift-Ctrl-PageUp

```
Sub ZoomOut()
    Dim ZP As Integer
    ZP = Int(ActiveWindow.ActivePane.View.Zoom.Percentage * 0.9)
    If ZP < 10 Then ZP = 10
    ActiveWindow.ActivePane.View.Zoom.Percentage = ZP
End Sub
```

## Remove all hyperlinks from document

(I do not assign this to a keystroke.)

```
Sub RemoveHyperlinks()
  Dim oField As Field
  For Each oField In ActiveDocument.Fields
  If oField.Type = wdFieldHyperlink Then
    oField.Unlink
  End If
  Next
  Set oField = Nothing
End Sub
```

Comments welcome. Updated November 17, 2021.
