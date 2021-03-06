## Introduction to table based Huffman decoding

There are many articles online that explain the basics of Huffman coding.
Most of them focus on two things,

 - constructing Huffman codes by building a Huffman tree,
 - bit-by-bit decoding while traversing that tree.

That is fine, introduction-level articles should exist. On the other end of the difficulty spectrum, there are academic papers about the theory behind efficient Huffman decoding and also less-formal cryptic notes written by compression experts on low-level implementation details of fast Huffman decoders. There is not so much in the middle.
This article is supposed to be in the middle, giving the reader some of the flavour of and intuition behind basic table based decoding, without immediately diving into the deep end.

### The standard Huffman tree is a bitwise trie

Of course, the standard Huffman tree is a bitwise trie.
Bit-by-bit decoding directly (though naively) uses that fact.
It also suggests a whole family of alternative decoding strategies: using higher degree (and thus shallower) tries.

Suppose the original Huffman tree was a perfect binary tree of height 8 (256 leaves, 255 internal nodes).
A group of three binary nodes, a parent and its two children, represents two successive choices both based on a single bit.
That group can be replaced by a single 4-way nodes, representing both choices at the same time, based on two bits.
Similarly, a group of three 4-way nodes can be combined into a single 16-way node, etc.
Starting with that tree of height 8, an equivalent 4-way trie would have a height of 4, an 16-way trie a height of 2, and finally, a 256-way trie would have only a single level, turning it into a plain old table.

The requirement that the original tree be a *perfect* binary tree is not met in any useful case.
This is easily side-stepped by duplicating leaves that are too shallow, effectively introducing some "phantom choices".
The bits that would conceptually be read by the phantom choices must not be consumed during decoding, the depth of a leaf is no longer equal to its code length.
In order to find the right code length, every leaf carries not only a symbol but also the number of bits to actually consume.
Below, the symbol `A` is too shallow and therefore duplicated. It still only corresponds to the code `1`, the bit that follows that `1` has been *peeked* by the decoder but since the code length is 1 it would not be consumed immediately.

![tree1](/tree1.png)

![tree2](/tree2.png)

That is just a way of thinking about what the decoding table means, not how the table would be created in practice.
Equivalently, one can take every `symbol <-> code` mapping and write the corresponding entries into every slot of the decoding table that has an index that *starts with* the `code`.
This is easily done by taking the `code` and padding it by every possible bit pattern of the remaining length, thereby mapping a code of length L to 2<sup>MAX_CODE_LENGTH - L</sup> slots in the table.
So in the above example, the final table could be created directly by:

 - writing `B,2` into `table[00]`
 - writing `C,2` into `table[01]`
 - writing `A,1` into both `table[10]` and `table[11]`, where the left `1` is the code corresponding to the `A` symbol, and the right bit is the padding.

In general there would be, for every code of length L, a loop from `0` up to but excluding `1 << (MAX_CODE_LENGTH - L)` which generates all the required padding bit strings in sequence.
No tree is involved.

The decoding procedure is conceptually simple: 
- read `MAX_CODE_LENGTH` bits into a `buffer` from the input at the current position (not byte-aligned),
- find `(symbol, length)` pair simply with `table[buffer]`,
- advance the position in the input by `length`. 

### Constructing length-limited codes

This style of table based decoding requires a table with 2<sup>MAX_CODE_LENGTH</sup> entries, making it important to ensure a reasonable maximum code length.
Building a Huffman tree does not in general guarantee that the longest code is no longer than your chosen maximum code length.
Finding an optimal set of codes given a constrained maximum length can be done using the package-merge algorithm, which is [the approach Zopfli takes](https://github.com/google/zopfli/blob/master/src/zopfli/katajainen.c).

A faster (not guaranteed optimal, but "good enough" for many purposes) approach is heuristically redistributing some bits, while being careful to keep the set of lengths valid overall.
Every code of length L corresponds to 2<sup>MAX_CODE_LENGTH - L</sup> slots of the decoding table, the sum of that over all lengths must not exceed the size of the table.
A set of lengths can be made valid by iteratively choosing the longest code that can be lengthened and lengthening it, until the set of lengths is valid.
That method can overshoot the target, there are more advanced algorithms that try to get closer to using 2<sup>MAX_CODE_LENGTH</sup> table slots.
Once a valid set of code lengths has been found, [canonical Huffman codes](https://en.wikipedia.org/wiki/Canonical_Huffman_code) of those lengths can be generated with a simple algorithm.

The lengths can be initially estimated based on the logarithms of the symbol probabilities, so no tree building needs to be done to construct the set of lengths.
So while Huffman coding is conceptually founded on concept of the Huffman tree, in practice neither the encoder nor the decoder need to build that tree.

