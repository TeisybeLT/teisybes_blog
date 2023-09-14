---
title: "Crackme: Cracknkeygen by 0xdecaf"
date: 2023-09-14T00:44:14+03:00
categories: writeups
tags: crackme
---

This is a write-up of a crackme titled `cracknkeygen` by `0xdecaf` for Linux which can be found on [crackmes.one](https://crackmes.one/crackme/64bf4185fc4ca2e6ca3d0f25). On crackmes.one, this crackme has a difficulty score of 2.0. Tools used while solving this crackme are Ghidra and Python.

The binary expects the user to pass in the key via the first argument as such: `./cracknkeygen <key>`. Upon execution, it either prints a success or failure message. Our task is to figure out how the key is checked and write a key generator to generate valid keys for this binary.

## Analysing the binary

I loaded the binary in Ghidra and performed automated analysis with the default options. Ghidra had no trouble parsing it and figured out where the `main` function is. All important code is located inside the `main` function so cleaning up variable names and data types was very quick and simple. I will not be providing full decompilation output as it is pretty simple and executes these steps:

1. Generates a key using an algorithm, which will be explained below. I named this generated key `source_key` during my analysis.
1. Using the same algorithm, a key is generated from the first argument passed into the binary. I named this computed key `user_key`.
1. Compares `source_key` with `user_key`. If they match - you win.

So let's analyze the first part - `source_key` generation, code for which is provided below:

```c
while( true ) {
  length_of_secret_6 = strlen("secret");

  if (length_of_secret_6 <= index_a)
    break;

  source_key = source_key + ((long)(int)"secret"[index_a] ^ index_a);
  index_a = index_a + 1;
}
```

This is a pretty simple algorithm, which xors each character's ASCII code in the word `secret` with its position and accumulates the result in the `source_key` variable. This code basically computes the constant 635.

One more thing of note is the code snippet below which happens after `source_key` calculation. The author of this crackme anticipated that people solving it may use `strings` (or a similar tool) to try and find the key, which would of course make the solution way too simple.

```c
return_value = strcmp(argv[1],"secret");
if (return_value == 0) {
  printf("sorry...\n");
  // Code to handle return from main
}
```

It explicitly prohibits using `secret` as the key supplied to the application.

Let's look at what happens next:

```c
while( true ) {
  length_of_arg = strlen(argv[1]);
  if (length_of_arg <= index_b)
    break;

  user_key = user_key + ((long)(int)argv[1][index_b] ^ index_b);
  index_b = index_b + 1;
}
```

This is the same algorithm again, but instead it uses the user-supplied key (via `argv[1]`) to compute the `user_key`. It then performs a comparison of them and prints the win/lose text.

```c
if (source_key == user_key) {
  printf("you did it!\n");  // Win condition
  // Return handling
}
else {
  printf("sorry...\n");
  // Return handling
}
```

So we basically figured we need to do in our keygen - write an algorithm that creates a string, whose each character xored with its position and summed up equals 635. Let's do that!


## Writing a keygen

The simplest solution for this problem would be brute forcing. Given capabilities of modern hardware, it would very quickly generate the key, but I didn't want to go that route, because there are more elegant ways of doing this. This is what I came up with.

Before I delve in to the key generation algorithm, I would like to introduce two helper functions that will be important down the line. The first one is responsible for generating a random printable character. It uses Python's [randint](https://docs.python.org/3/library/random.html#random.randint) function to generate a number in a specified range. The lowest printable character, excluding space and other junk, is `!` (0x20 ASCII) and the highest one is `~` (0x7E ASCII), so that is the range I chose. Ideally, I would have written a more complicated algorithm to exclude characters that need escaping (such as `&`, `$` and the likes), but I will be solving this problem differently at the end of this section.

```python
def generate_random_char() -> chr:
    return chr(random.randint(ord('!'), ord('~')))
```

Since we will be computing a character in the key generation algorithm I also need a function to check if that computed character is printable. That's where the second helper function comes in - it checks if a supplied ASCII character can be printed. This uses the same inclusive range between `!` and `~` mentioned earlier.

```python
def is_printable(character : chr) -> bool:
    return ord(character) in range(ord('!'), ord('~'))
```

One more thing I'd like to add before discussing the key generation algorithm is the code snippet below. Previously I mentioned the magic number of 635, which is used when validating the key. Well I didn't calculate it by hand, I wrote a function to do that for me:

```python
def create_key() -> int:
    secret_string = "secret"
    key = 0
    for index, char in enumerate(secret_string):
        key += ord(char) ^ index

    return key
```

Now for the key generation algorithm in its entirety:

```python
def generate_key(target_value : int) -> str:
    while True:
        out = ""
        index = 0
        count = 0

        while count < (target_value - (ord('~') ^ index)):
            char = generate_random_char()
            xored_char = ord(char) ^ index
            count += xored_char
            index = index + 1
            out += char

        char = chr((target_value - count) ^ index)
        if not is_printable(char):
            continue

        out += char
        return out
```

It contains an outer loop which runs util a valid key is generated using the algorithm within it. Generally, I prefer to avoid unbounded loops in scenarios such as this, since it can run away if coded poorly, but in practice, this algorithm produces a valid key within an iteration or two.

The fun stuff happens inside this loop. It contains another loop which generates `n-1` characters of the key (`n` being the number of characters in the generated key), which is accomplished by generating a random printable character then performing an `xor` operation with its index and storing the necessary information in housekeeping variables. The terminating condition of this loop is ` count < (target_value - (ord('~') ^ index))`, which translated into human speak means "generate another character until the next character that is to be generated is within printable range".

Once that loop finishes, we can use the properties of the `xor` operation to compute the final character. We know what the `xor`ed value of the character is and we know it's index. We just `xor` them together to get the actual character. It may also happen that this computed character falls below the specified printable range. In that case - restart the entire algorithm. But it usually succeeds, so we add that to a housekeeping variable responsible for storing the generated key and return it back to the caller.

And finally, the `main` function that glues it all together:

```python
def main():
    source_key = create_key()
    generated_key = generate_key(source_key)

    print(f"Your key is: {shlex.quote(generated_key)}")

if __name__ == "__main__":
    main()
```

The only thing of note here is the use of Python's [shlex.quote](https://docs.python.org/3/library/shlex.html#shlex.quote), which handles escaping of characters, so that the resulting key can be copied and pasted for the crackme.

You can find the full code for this keygen in my [github repo](https://github.com/TeisybeLT/Writeups/blob/master/cracknkeygen_0xdecaf/keygen.py)

Here are some examples of keys generated by this keygen:
* 'b'"'"'f"iCaS'
* '6t(0nlfG'
* '5AUTxz['
* 'DVo_`:r'

## How would I improve on this crackme

For me, the give-away for this crackme was the initial `source_key` generation bit. This basically computes a constant (635) and very explicitly shows the algorithm that is being used to compute it. If I were creating this challenge, I would have either hardcoded the number 635 or written a `constexpr` function (assuming C++ is used) to generate it. This would not increase the complexity by that much, but it would allow me to get rid of the if condition designed to safeguard against the usage of `strings` command to solve this without reverse-engineering, since the string `secret` would no longer be stored in the compiled binary.

## Conclusion

Overall, this was a very enjoyable crackme to solve. It took around 30 mins from downloading till a working keygen and I consider that time well spent.