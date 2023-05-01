# pokecomms

This challenge is pretty straight forward, a text file is given with some encoded text.

There are only two sequences of characters "PIKA" and "CHU!", this suggests that these represent binary data (0s or 1s)

These pieces of encoded data are split into lines, with each line containing 8 characters which also indicates that each line contains a representaion of a character over 8 bits.

Ascii characters are actually over 7 bits so the MSB is always a zero, so if we look into the file we can see that all lines start with a "CHU!" which confirms out theory and indicates that "CHU!" is 0.

with that we can script the solution to get the flag

```python
file = open("pokecomms.txt",'r')

crypt = file.readlines()

file.close()
chars = []
final = ['']*len(crypt)
count = 0
for i in crypt:
    for j in i.split():
        if j == 'CHU!':
            final[count] = final[count] + "0"
        else:
            final[count] = final[count] + "1"
    count += 1

for i in final:
    ascii_code = int(i, 2)
    ascii_char = chr(ascii_code)
    print(ascii_char, end='')

```