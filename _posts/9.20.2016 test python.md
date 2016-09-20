---
title: Test Python Code
layout: post
comments: True


---

Here is some python code for making a simple number guessing game.

```python
import random


guessesTaken = 0

print("Hello! What is your name?")
my_name = input("Name: ", )

number = random.randint(1, 20)
print("Well, " + my_name + ", I am thinking  of a number between 1 and 20.")

while guessesTaken < 6:
    print("Take a guess.")
    guess = int(input())

    guessesTaken = guessesTaken + 1

    if guess  < number:
        print("Your guess is too low!")

    if guess > number:
        print("Your guess is too high!")

    if guess == number:
        break

if guess == number:
    guessesTaken = str(guessesTaken)
    print("Good job, " + my_name + "! You guessed my number in " + guessesTaken + ' guesses!')

if guess != number:
    number = str(number)
    print("Nope. The number I was thinking of  was " + number)

```
