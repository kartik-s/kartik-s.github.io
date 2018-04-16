# How OEIS helped me write a PNG decoder

A little while back I wrote [a decoder for the PNG image format](https://github.com/kartik-s/lua-png) in Lua 5.1.[^1] I ran into a bunch of interesting problems along the way, but there's one that I want to discuss one in particular.[^2] But first, some background.

[^1]: For fun, of course.
[^2]: This problem is actually pretty orthogonal to the problem of decoding PNG files, but I figured it would be cool to write about anyway.

A PNG file contains data split into "chunks" which contain information about different aspects of the file. For instance, every PNG file starts with an `IHDR` chunk that contains things like the image resolution, bit depth, color type, and a few other items related to data storage. The actual pixels are stored in one or more `IDAT` chunks. The real challenge for this project was writing code to decode `IDAT` chunks.

The PNG file format supports lossless compression, in constrast to lossy compression methods such as JPEG. PNG takes your image's raw pixel data, compresses it using an algorithm called DEFLATE,[^3] and then sticks this compressed data into one or more `IDAT` chunks. The hard part of decoding an `IDAT` chunk is decompressing[^4] this DEFLATEd data to recover the original pixel data.

[^3]: This is the same compression algorithm used by the gzip and ZIP file formats. https://en.wikipedia.org/wiki/DEFLATE
[^4]: Inflating, if you will.

As I read through [RFC 1951](https://tools.ietf.org/html/rfc1951) to figure out exactly how to do this, I came across this table:[^5]

```
                  Extra           Extra               Extra
             Code Bits Dist  Code Bits   Dist     Code Bits Distance
             ---- ---- ----  ---- ----  ------    ---- ---- --------
               0   0    1     10   4     33-48    20    9   1025-1536
               1   0    2     11   4     49-64    21    9   1537-2048
               2   0    3     12   5     65-96    22   10   2049-3072
               3   0    4     13   5     97-128   23   10   3073-4096
               4   1   5,6    14   6    129-192   24   11   4097-6144
               5   1   7,8    15   6    193-256   25   11   6145-8192
               6   2   9-12   16   7    257-384   26   12  8193-12288
               7   2  13-16   17   7    385-512   27   12 12289-16384
               8   3  17-24   18   8    513-768   28   13 16385-24576
               9   3  25-32   19   8   769-1024   29   13 24577-32768
```

[^5]: I'm reluctant to explain what this table actually means, since that would probably double the size of this post. For more information, check out RFC 1951 or [this great explanation of the DEFLATE algorithm](https://zlib.net/feldspar.html).

I didn't want to copy that table down in my code manually.[^6] However, if I could find some sort of mapping from the "Code" column to the first number in the "Dist" column, then I would be all set:

[^6]: In hindsight, I probably would have saved a lot of time if I had just done that.

```
Code Dist
---- ----
  0   1
  1   2
  2   3
  3   4
  4   5
  5   7
  6   9
  7   13
  8   17
  9   25
 10   33
 11   49
 12   65
 13   97
 ...  ...
```

I tried racking my brain for some sort of pattern, but I couldn't come up with anything neat and clean. As a last-ditch effort, I tried searching for the sequence of numbers in the "Dist" column in the [On-Line Encyclopedia of Integer Sequences](https://http://oeis.org). I got no results. Then I tried searching again, except this time I didn't include the first four elements in the sequence. This time, I got [exactly one result](http://oeis.org/A209721).

It turns out that each number in the "Dist" column contains "1/4 the number of (n+1)X3 0..2 arrays with every 2X2 subblock having distinct clockwise edge differences."[^7] More relevant to me was that the sequence can be encoded by the following recurrence relation:

```
a(0) = 3
a(1) = 4
a(2) = 5
a(n) = a(n - 1) + 2a(n - 2) - 2a(n - 3)
```

[^7]: If someone knows what this means, please let me know.

Neat, clean, and really quick to implement.[^8] Thanks, OEIS!

[^8]: Clearly you'll run into performance issues if you use this straight recursive definition. You can mitigate this by using memoization, building the sequence iteratively from the bottom up, or just generating the sequence offline and pasting it into your code as a table or something.

