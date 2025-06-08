---
title: Even the Best Laid Plans...
date: 2025-05-26
categories: [blog, development]
tags: [plans]     # TAG names should always be lowercase
---

# Even the Best Laid Plans...

Recently I have been working on a Vim/Kakuone-like modal [TUI editor in Go](https://github.com/BoweFlex/vincent-vimgo). I don't expect it to ever really be used by anyone, but it seemed like a great exercise to better understand displaying content and tracking various states. This practice exercise has also led me to reflect on planning, and that despite planning being an important part of a project they almost never match what you end up with. As Robert Burns said:

> The best laid plans of mice and men often go awry.

Extensive planning is often prescribed for programming. For example, Test Driven Development encourages planning individual units (often functions) to the point that you can write a unit test for a specific unit, then write code and refactor until that test is passed.
However, especially if you are developing in a domain you are not intimately familiar with, or solving a problem that is not already solved, it is very possible that the solution you plan does not work, or that there are edge cases that you did not know to consider until they happened.
Either way, you will likely find yourself spending as much time calling audibles or rewriting code as you will planning ahead for that code.

In this article, I am going to walk through a hands on example of how I approached planning a new feature for my code editor, and how that plan went awry.

## Code Editor Basics

There are a lot of things about how a text editor operates that I had not considered before starting on this project. I won't cover all of them right now, but a couple are relevant for context:

### Cursor Positioning

Ignoring how you actually display things on the screen, not only does the cursor need to be shown and move around, but if you move down to a line that is shorter than your current cursor position
the cursor will go to the end of that new line, but if you move the cursor back to a line long enough to accommodate your original cursor position it will move back to its initial column.
You also have to consider the size constraints of the current window, i.e. moving the cursor down off the screen should scroll rather than it becoming invisible.

### Visible Text

Monitors do not have infinite space to display content, like text in your editor, and chances are you are not exclusively looking at the beginnings of the lines in a file you are editing.
This means you need to find a way to distinguish what content is visible on screen, what is not visible on screen, and how that changes as the cursor moves offscreen.

## The Gameplan

When I started trying to break down the problem of handling visible text, I pretty quickly settled on using a horizontal and vertical offset to determine what text should be shown.
Making this change also required some changes to cursor movement and positioning to account for the offset potentially changing where the cursor could be. Here's some pseudo code for my plan going into this:

```text
Add a bool value to InputProcessor struct
Bool value is initialized as false, and is set to true when a rune (displayable character) was entered in INSERT mode
If cursor moves offscreen, determine how far and increase by that much
    If bool is false, cursor position is subtracted by the same amount by offset being increased
        Preferred cursor column needs to be decreased by same amount as cursor position, and need to check if it's valid for the length of the current line
    If bool is true, cursor position is not subtracted, and bool is set to false
```

## The Plan Went Awry...

I started to develop this solution, and realized a few things:

- Even with not changing cursor position when bool was true, moving the cursor and adjusting the offset was not consistent
    - If the cursor was moved on existing line, the offset would adjust every time your cursor moved (aka pressing l/h or <-/->)
    - If you were adding more characters to a line, the offset would adjust every other time you typed a new character
- Preferred cursor column was being updated so often, and compared to things like current line length just as often

Worst of all, it felt overcomplicated. Even once I felt like I had a decent grasp on how things were operating, I had a feeling that things could be simpler.

## How I Adjusted

As I was trying to figure out why the cursor did not move every time you typed a character in the line, moving the cursor off screen, I realized something.

**The preferred cursor column, which was already being tracked, could solve all these problems itself.**

I had written the following, either early on in beginning this project or as part of trying to solve the offset problem.

```go
type position struct {
	x, y int
}

type cursorInfo struct {
	position     position
	preferredCol int
}

func (p *position) clampX(minX, maxX int) {
	if p.x < minX {
		p.x = minX
	} else if p.x >= maxX {
		p.x = maxX - 1
	}
}

func (c *cursorInfo) addDelta(xDelta, yDelta int, changePreferredCol bool) {
	c.position.x += xDelta
	c.position.y += yDelta
	if changePreferredCol && xDelta != 0 {
		c.preferredCol = c.position.x
	}
}
```

The `cursorInfo` struct was used to track both the current x,y coordinates for the cursor and the preferred column it wanted to be in (preparing for situations where the current line is not long enough).
There are functions to clamp both x and y coordinates, which is based on the following definition of [clamping](https://en.wikipedia.org/wiki/Clamp_(function)):

> the process of limiting a value to a range between a minimum and a maximum value.

Finally, there is a function used to move the cursor as requested, and optionally update the preferred column. This updates the preferred column almost any time the x coordinate is changed.

By better utilizing that preferred column, I could greatly simplify the following:

- Cursor positioning while calculating any changes to the offset
- Determining whether the preferred column is acceptable in the current line
- Determining whether the new line fits the preferred column when the previous line did not

The final solution for handlng cursor positioning with a horizontal offset became:

```go
if p.cursor.position.x >= p.screenWidth {
	diff := max(p.cursor.position.x-p.screenWidth, 1)
	p.offset.x += diff
} else if p.cursor.position.x < p.offset.x {
	diff := p.offset.x - max(p.cursor.position.x, 0)
	p.offset.x -= diff
}
// Determining how much of the current line is actually visible, but allowing the cursor to be one column in front of the text
currLineVisibleLength := len(lines[p.cursor.position.y]) - p.offset.x + 1
p.cursor.position.x = p.cursor.preferredCol
p.cursor.position.clampX(p.offset.x, currLineVisibleLength)
```

## The "Right" Answer (According to me)

Don't get me wrong, I am not saying you should never plan for projects; pseudocode or in some cases even writing tests before writing code can be a great tool. 
I find it incredibly useful to try to break down large problems into smaller chunks that are solvable, whether that is planning out individual functions and parameters or just the general flow of a process.
However, especially when you are developing in an unfamiliar area, whether it is writing backend code when you typically write frontends or writing a TUI when you typically handle data, there may be too many variables you are unaware of to properly plan a process or API before you have encountered some of those variables.
Even when you do properly plan a solid API for your program to interact with, it is possible that a refactor could greatly improve performance or readability after you're more deeply familiar with the ins and outs of your process.

I would encourage you to stay flexible in your development and ready to "call an audible" if you think there may be a better way to approach the problem you are trying to solve.
There is still a balance to find - you could likely rewrite any process almost infinitely and it would be "better", but you would give up future progress for increasingly small improvements to the existing process.
Time spent writing code is time not spent writing other code, so unless the problem you are trying to solve is actively broken I would consider if a potential improvement to the process is more valuable than writing new processes instead.
Conversely, consider if the new code you are writing will overcomplicate a future refactor that will be simpler if handled now.

Ultimately, no process is future-proof. Your goal, at least in the beginning, should be to get code written and find motivation as you make progress.
Once you have a working solution and a good grasp on both the big picture and the smaller moving pieces, it will be easier to determine where refactors are useful and where you need tests to confirm you're handling the correct edge cases.

There is likely not a one size fits all solution for planning, and for most everyone the answer is likely somewhere in the middle of planning every small detail of your project and winging it.
Experiment, try different things, and find what works for you to write the best code you can.
