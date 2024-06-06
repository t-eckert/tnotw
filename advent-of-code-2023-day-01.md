---
title: Advent of Code 2023, Day 1
published: 1 December 2023
tags:
  - Python
  - Advent of Code
---

Today is the first day of [Advent of Code for 2023](https://adventofcode.com/)! This annual coding challenge
consists of puzzles increasing in difficulty each day from the start of December through Christmas.

My self-imposed rule this year is to use only the Python standard library. No external dependencies are allowed.
I can't get around this restriction by copy-pasting some existing A\* algorithm either. All code needs to be
written after the start time. I don't tend to over-index on runtime complexity. I'm keeping my solutions to this year in my
[advent-2023](https://github.com/t-eckert/advent-2023) repository.

Before I dive into my solution for Day 1, I thought I would share a few helpful tips and code snippets I developed
to use with Advent of Code.

## Downloading input files

Every problem in Advent of Code follows the same pattern, there is an input file and a desired outcome from running
a calculation on that file. The URL to download the input files follows a predictable pattern so I wrote a Python
script to download that file.

```python
from datetime import datetime

import os
import sys

session = os.environ["AOC_COOKIE"]


if len(sys.argv) > 1:
    day = sys.argv[1]
else:
    day = datetime.now().day


os.system(
    f'curl --cookie "session={session}" https://adventofcode.com/2023/day/{day}/input > day_{day:02d}.txt'
)
```

This script depends a user setting the `AOC_COOKIE` environment variable because the puzzle inputs are unique for
different users. This can be grabbed from application storage when authenticated to the website.

This script can be passed a number to download a specific date's input or will default to the current day if left empty.

## Reading input files

This function saves me a bit of time as I know that I just want a the contents of `filename` as a string.

```python
def read(filename: str) -> str:
    with open(filename, "r") as f:
        return f.read()
```

## Tee-ing output

I call this little helper `tee` after the [Unix program](https://man7.org/linux/man-pages/man1/tee.1.html) that inspired it.
This function is about as simple as they come, yet it can be incredibly helpful when debugging a problem using print statements.

```python
def tee(val):
    print(val)
    return val
```

By printing a value and returning it, this function can be placed inline with function calls, replacing the need for a separate
variable declaration to print a value out.

```python
# Before
sum(double_vals(filter_evens(input)))

# After
sum(tee(double_vals(tee(filter_evens(input)))))
```

## Day 1

I was up last night working on [Devy](https://devy.page), so I started this problem at midnight when it was released for me.
The problem asks you to look at a series of strings, each of which contains single-digit numbers. These numbers show up in the
string in both a numeric (`9`) and word (`nine`) form, but the first part of the problem only asks you to identify the numeric
form only. The answer to the problem is the sum of values produced by joining the first and last digits in the string to form a two-digit number.

- [Problem statement](https://adventofcode.com/2023/day/1)
- [Full solution on GitHub](https://github.com/t-eckert/advent-2023/blob/main/day_01.py)

I was able to solve part 1 rather quickly. Because we are looking at the numeric representation and the numbers are single-digits,
I wrote a function that would grab the first single-digit numeric value in a string.

```python
def is_number(c: str) -> bool:
    return c in "0123456789"


def first_number(line: str) -> str:
    for c in line:
        if is_number(c):
            return c
    return ""
```

This function can find us both the first and last digits in the string if we reverse the input. Joining these values and summing them
gives the answer to part 1.

```python
def part_1(lines: list[str]) -> int:
    return sum([int(first_number(line) + first_number(line[::-1])) for line in lines])
```

Moving on to part 2, I needed to find a way to effeciently grab the first and last word-form numbers. I decided to parse through the
characters in the strings until I found a character that was a candidate first letter for a word-form number. This worked well because
I have such a limited set of words, just representing 1 through 9. In my solution, I did include zero as a possibility, which was a mistake
but didn't cause my solution to fail.

To effeciently do this number word lookup, I built a dictionary that mapped the first letter of the word to the candidate words then to the
numeric forms they represented.

```python
number_words: dict[str, dict[str, str]] = {
    "z": {"zero": "0"},
    "o": {"one": "1"},
    "t": {
        "two": "2",
        "three": "3",
    },
    "f": {
        "four": "4",
        "five": "5",
    },
    "s": {
        "six": "6",
        "seven": "7",
    },
    "e": {"eight": "8"},
    "n": {"nine": "9"},
}
```

As I iterated through the string, I matched on the keys of this dictionary then iterated over the candidates to test for a match. This avoided an
issue many people ran into where number words could overlap other number words (e.g. `eightwo` which should resolve to `82`).

I wrote two very similar functions for getting the first and last values in the string. I could have instead created a second `number_words` dictionary
where the words were reversed, but I didn't.

```python
def first_number_or_word(line: str) -> str:
    for i, c in enumerate(line):
        if is_number(c):
            return c
        if matches := number_words.get(c):
            for number_word in matches:
                word = line[i : i + len(number_word)]
                if word == number_word:
                    return number_words[c][number_word]
    return ""


def last_number_or_word(line: str) -> str:
    for i, c in enumerate(line[::-1]):
        if is_number(c):
            return c
        if matches := number_words.get(c):
            for number_word in matches:
                offset = len(line) - i - 1
                word = line[offset : offset + len(number_word)]
                if word == number_word:
                    return number_words[c][number_word]
    return ""
```

The use of these functions was not too different from part 1.

```python
def part_2(lines: list[str]) -> int:
    return sum(
        [int(first_number_or_word(line) + last_number_or_word(line)) for line in lines]
    )
```
