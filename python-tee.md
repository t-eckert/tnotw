# Python Tee

In Unix, there is a utility called [tee](<https://en.wikipedia.org/wiki/Tee_(command)>). It's perfectly named because it forks its input into a `T`, sending the data both to standard out and to the next process in a command. It's useful when you want to take a peek at a value in some intermediary stage of a script while allowing it to also be used for future processing.

When I learned this concept, I started seeing value for it everywhere in my development. It was particularly handy when working on the most recent [Advent of Code](https://adventofcode.com/) which I did entirely in Python.

The problems in Advent of Code all deal with some form of data transformation. Along the way, I wanted to debug the transformation by seeing the data and allowing it to continue being processed through my solution. For this, I wrote a simple `tee` function.

```python
def tee(v):
    print(v)
    return v
```

While I've used print debugging countless times to solve these types of puzzles, what was nice about this function was how I could just slip it in line with the existing solution. Take [my solution for Day 4](https://github.com/t-eckert/advent-2023/blob/2b4ed483c0536158fa8a563ba95c6265af180353/day_04.py) as an example. If I was testing my part 1 answer and needed to ensure I was deserializing the card information correctly, the modification is minimally invasive.

```python
# before
def deserialize_card(raw: str) -> Card:
    id = raw.split(":")[0]
    mine, winning = raw.split(":")[1].strip().split("|")
    return Card(id, to_set(mine), to_set(winning)))

# after
def deserialize_card(raw: str) -> Card:
    id = raw.split(":")[0]
    #                V it's tee!
    mine, winning = tee(raw.split(":")[1].strip().split("|"))
    return Card(id, to_set(mine), to_set(winning)))
```

The concept is so easy to implement and broadly useful once you know it, I definitely appreciate having it in my back pocket.
