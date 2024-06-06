# Advent of Code 2023, Day 3

This problem asks us to process a multi-line string representing the engine of a gondola.
The input is made up of digits and symbols with `.`s representing empty space in the engine.
This input is given as an example.

```text
467..114..
...*......
..35..633.
......#...
617*......
.....+.58.
..592.....
......755.
...$.*....
.664.598..
```

For part 1, we need to sum all of the numbers that have a symbol adjacent to them. For part 2,
we need to sum the product of all numbers adjacent to a `"*"` if the `"*"` has exactly two numbers
adjacent to it.

- [Problem statement](https://adventofcode.com/2023/day/3)
- [Full solution on GitHub](https://github.com/t-eckert/advent-2023/blob/main/day_03.py)

### Part 1

I didn't need to do much to the puzzle input in order to parse it for the information I needed. I just split
it up into a list by newlines and iterated over the values. When I encountered a digit I began feeding those digits
into a buffer until a non-digit value was encountered.

```python
def part_1(engine_rows: list[str], symbols: set[str]) -> int:
    engine_dimensions = (len(engine_rows), len(engine_rows[0]))

    reading_number = False
    subject_buffer = set()
    number_buffer = ""

    total = 0

    for row, vals in enumerate(engine_rows):
        for col, char in enumerate(vals):
            if char in digits:
                subject_buffer.add((row, col))
                number_buffer += char
                reading_number = True
            else:
                if reading_number:
                    if has_symbol(
                        engine_rows, border(subject_buffer, engine_dimensions), symbols
                    ):
                        total += int(number_buffer)
                    subject_buffer.clear()
                    number_buffer = ""
                reading_number = False

    return total
```

This approach is one I've used before when parsing in data and trying to grab chunks of it.
There is some global toggle that determines if the data should be read into the buffer.
When a piece of data is encountered that ends the chunk that should be passed in, in this case
a non-digit, the toggle is switched off, the chunk is stored from the buffer, and the buffer is cleared.

In this implementation, when the end of the number is reached, I add the number to the running total
if the bordering characters include symbols.

I do this in two parts. First I find the set of locations in the engine which border the number, then
I check those locations to see if they include a symbol.

```python
#[...]
                    if has_symbol(
                        engine_rows, border(subject_buffer, engine_dimensions), symbols
                    ):
                        total += int(number_buffer)
#[...]


def border(
    subject: set[tuple[int, int]], engine_dimensions: tuple[int, int]
) -> set[tuple[int, int]]:
    height, width = engine_dimensions

    borders = set()

    for cell in subject:
        row, col = cell[0], cell[1]

        # Iterate clockwise around the location
        if row > 0 and col > 0:
            borders.add((row - 1, col - 1))  # above left
        if row > 0:
            borders.add((row - 1, col))  # above center
        if row > 0 and col < width - 1:
            borders.add((row - 1, col + 1))  # above right
        if col < width - 1:
            borders.add((row, col + 1))  # center right
        if row < height - 1 and col < width - 1:
            borders.add((row + 1, col + 1))  # below right
        if row < height - 1:
            borders.add((row + 1, col))  # below center
        if row < height - 1 and col > 0:
            borders.add((row + 1, col - 1))  # below left
        if col > 0:
            borders.add((row, col - 1))  # center left

    return borders - subject


def has_symbol(
    engine_rows: list[str], locations: set[tuple[int, int]], symbols: set[str]
) -> bool:
    for loc in locations:
        if engine_rows[loc[0]][loc[1]] in symbols:
            return True
    return False
```

I was particularly happy with the `border` function because I took an approach I hadn't thought of when I did a similar
problem several years ago. I took the 8 cells that surround each digit in the number as a set and then subtracted from that
the values that make up the number. This leads to a fairly clean implementation that would work generally on any shape.

### Part 2

I took the wrong approach initially with part 2. I first went looking for all of the gears in the puzzle, then used my border
finding code to grab digits adjacent to the gears. The problem here is that I had to add a lot of edge-case logic for handling
if an adjacent digit was part of a number in another adjacent digit or if it represented a separate number. I got deep into some
globbing of numbers by iterating back to the start of the number and then forward from the center of the number. It was a mess.

I took some time away from the problem and decided to approach it in the opposite manner. I created a class called `PartNumber`
that stores the full number and the cells that make up that number. A cell here is just a location in the engine. I call this same
concept multiple things in the code depending on when I wrote it.

```python
class PartNumber:
    def __init__(self, value: int, cells: set[tuple[int, int]]):
        self.value = value
        self.cells = cells

    def __repr__(self) -> str:
        return f"{self.value}\t{self.cells=}"
```

This class allowed me to store the numerical value and the full location of every part number. This was all I needed to then
map the location of every gear in the engine to its adjacent numbers. This allowed me to reuse the shape of my solution to part 1
and avoided all the nasty number globbing.

```python
def part_2(engine_rows: list[str]) -> int:
    engine_dimensions = (len(engine_rows), len(engine_rows[0]))

    reading_number = False
    subject_buffer = set()
    number_buffer = ""

    numbers = []

    for row, vals in enumerate(engine_rows):
        for col, char in enumerate(vals):
            if char in digits:
                subject_buffer.add((row, col))
                number_buffer += char
                reading_number = True
            else:
                if reading_number:
                    numbers.append(
                        PartNumber(int(number_buffer), subject_buffer.copy())
                    )
                    subject_buffer.clear()
                    number_buffer = ""
                reading_number = False

    gears = {}
    for number in numbers:
        for neighbor in border(number.cells, engine_dimensions):
            if engine_rows[neighbor[0]][neighbor[1]] == "*":
                if neighbor in gears.keys():
                    gears[neighbor].append(number)
                else:
                    gears[neighbor] = [number]

    total = 0
    for _, numbers in gears.items():
        if len(numbers) == 2:
            total += numbers[0].value * numbers[1].value

    return total
```

To get the answer, I iterated over all of the gears summed the product of numbers for gears that were adjacent to exactly two numbers.
