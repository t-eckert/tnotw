# Advent of Code 2023, Day 2

In this problem, an elf is pulling sets of colored cubes from a bag. In part 1, we need to determine how many of those sets of handfuls pulled
from the bag are possible given a predetermined count of each colored cube. In part 2, we need to determine the minimum number of cubes that
are required for the given sets of handfuls to be possible.

- [Problem statement](https://adventofcode.com/2023/day/2)
- [Full solution on Github](https://github.com/t-eckert/advent-2023/blob/main/day_02.py)

## Part 1

Each set of cubes pulled from the bag is referred to as a handful in the problem statement. Multiple handfuls make up a single "game".
Each game is presented as a string in the input file.

```text
Game 1: 2 blue, 4 green; 7 blue, 1 red, 14 green; 5 blue, 13 green, 1 red; 1 red, 7 blue, 11 green
```

After each handful is presented, the cubes are returned to the bag and may be reused.

In part 1, I needed to determine whether a given game was possible with the following set of cubes being in the bag:

- `12 Red`
- `13 Green`
- `14 Blue`

A possible game is one where the cubes presented are **never greater** than the cubes provided.
The answer to the puzzle is the sum of game ids for possible games.

This problem breaks down into two parts, a string parsing part and an evaluation of possible games.

I know all of the inputs so I can take a very uncareful and quick appraoch to getting the two pieces of information I need from each game: the game id
and the list of handfuls presented.

```python
def game_id(s: str) -> int:
    return int(s.strip().split(":")[0].split(" ")[1])


def deserialize_handfuls(s: str) -> list[tuple[int, int, int]]:
    return [count_cubes(handful) for handful in s.strip().split(":")[1].split(";")]


def count_cubes(handful: str) -> tuple[int, int, int]:
    r, g, b = 0, 0, 0

    cubes = handful.strip().split(",")
    for cube_color in cubes:
        count = int(cube_color.strip().split(" ")[0])
        if "red" in cube_color:
            r = count
        elif "green" in cube_color:
            g = count
        elif "blue" in cube_color:
            b = count

    return (r, g, b)
```

I have encoded the handfuls as tuples with 3 integers. They correspond to red, green, and blue respectively.

This produces well-structed data from each game that can be evaluated. Take the form of "Game 1" listed above, which is now much more readable for the program.

```text
id: 1
handfuls: [(0, 4, 2), (1, 14, 7), (1, 13, 5), (1, 11, 7)]
```

I wrote a function to check each handful against the set of cubes provided.

```python
def is_allowed(reqs: tuple[int, int, int], handful: tuple[int, int, int]) -> bool:
    for i, color in enumerate(handful):
        if reqs[i] < color:
            return False
    return True
```

Then I used the parsing and evaluating functions together to sum the ids of games that were possible.

```python
def part_1(games: list[str]) -> int:
    reqs = (12, 13, 14)

    return sum(
        game_id(game)
        * all(is_allowed(reqs, handful) for handful in deserialize_handfuls(game))
        for game in games
    )
```

## Part 2

For part 2, I didn't need any new parsing code. I did need a way to evaluate the minimum set of cubes that would make a given game possible.
This can be found by iterating over every handful shown and taking the maximum value that we ever observe for each cube color to be the minimum
we need of that color for the game to be possible.

For the example Game 1,

```text
Game 1: 2 blue, 4 green; 7 blue, 1 red, 14 green; 5 blue, 13 green, 1 red; 1 red, 7 blue, 11 green
```

the maximum value of each color is 1 red, 14 green, and 7 blue. Therefore, the minimum set of colored cubes that make this game possible is this same set.

This function takes a list of the deserialized handfuls and makes the same determination.

```python
def min_cubes(handfuls: list[tuple[int, int, int]]) -> tuple[int, int, int]:
    return tuple(max(x) for x in zip(*handfuls))
```

The answer to the puzzle is the sum of the product of cubes that make each game possible. So we iterate over the games, evaluate the minimum possible set of cubes,
then we multiply those cube counts together and sum it all up. I used the `prod` function from the `math` package in the standard library to get the product of the
minimum count of cubes.

```python
from math import prod

def part_2(games: list[str]) -> int:
    return sum(prod(min_cubes(deserialize_handfuls(game))) for game in games)
```
