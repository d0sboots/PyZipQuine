# PyZipQuine
A polyglot quine for Python and PKZIP.

quine.py.zip is a Python program that prints its own source, commonly known as a [Quine](https://en.wikipedia.org/wiki/Quine_(computing)). It is also a functional .zip file that contains exactly one file, quine.py.zip, which is exactly the same as the source file. Programs which do the same thing when executed in different languages are known as [Polyglots](https://en.wikipedia.org/wiki/Polyglot_(computing)), so this is a two-language polyglot quine for the languages Python and PKZIP.

## How do you construct such a thing?

First, I recommend reading [Zip Files All The Way Down](https://research.swtch.com/zip), an in-depth writeup of (to the best of my knowledge) the first zip quine. It's a great read, a good primer on quines in general as well as the basics of LZ77, and also part of my inspiration for doing this. Then, forget most of what you read, because the underlying techniques here are totally different. :)

What makes this project possible is the looseness of the PKZIP format. Unlike almost every other file format, zip files do not have a fixed header, and due to an optional comment field they don't have a fixed footer either! This is a feature that is used for creating self-extracting archives; in our case it will allow for embedding a Python program at the start, before the zip data itself begins.

Initially, I thought the Python half of the quine could be relatively simple, by using r""" (a raw, triple-quoted string) to surround the zipped data. This would both protect it, so that Python wouldn't choke on the binary garbage characters, and also make it available to Python so it could print it out for the quine function. **However**, Python has an interesting *feature* where when it encounters a null byte in the source, it drops everything from there until the end of the line. For instance, this is a valid Python program, given ^@ represents null:

```
pri^@ Haha this is ignored
nt "Hello wo^@ This too. "Quotes don't matter"
rld!"
```

### Take 2

Because of this, we need to take more care. The triple-quote technique will still work, but sections of the data that involve nulls will need to be repeated in Python as strings with \0. This is where it becomes important to get familiar with the structure of a zip file. The key points are:
- The (compressed) data for each file is preceded by a local file header
- Near the end of the archive, each file also has a central directory entry
- These two headers are very similar with the local file header being a subset of the latter
- After the directory entries is an end-of-central-directory record, which is essentially the file's footer
- The EoCD has a variable-sized comment field, which allows arbitrary bytes to appear at the very end of the file

It's also important to know something about the DEFLATE algorithm: Although usually it operates in bitwise mode, decoding Huffman symbols into either literal charcters or back-references, it also has a "literal" block type which consists of a 4-byte length-prefix and then LEN many bytes which are copied verbatim to the output. This mode is always byte-aligned, which makes it very useful for our purposes.

We'll define a function that takes a string and does the work. This allows us to finish the file with a minimal footer: A newline (to end the line that's being ignored due to nulls), """ to close the string, closing parens to call the function, and a final newline to go with the general source convention of having a final newline.

At a high level, the structure of the quine looks like this:
```
|<python code>|a(r""|"|PK^C^D|common header|filename|begin-literal-block-1|
|<python code>|a(r""|begin-literal-block-2|"|PK^C^D|common header|filename|begin-literal-block-1|
|deflate-block-including-self-referential-stuff|
|PK^A^B^T^C|common header|directory-entry-specific-stuff|filename|end-of-central-directory-record|
|\n""")\n|
```

### Wheels within wheels

The problem with trying to create a zip quine is that at the beginning, you're writing output to a position before your input, and by the end you are writing past the input position. At some point, you have to cross the streams, and that's where the magic happens. In "Zip Files All The Way Down," he confronted and solved the problem head-on. In our case, we can sidestep the problem by embedding the deflate-string in the Python section, which we need to do anyway so that the Python quine can print it. As a result, in the deflate-block section we can output the entire deflate-block with a single back-reference to the copy we've already output, via the literal block section.
