>To denote the subsequence of natural numbers `2, 3, ..., 12` without the pernicious three dots, four conventions are open to us

为了避免使用 `...` 来表示自然数的子序列 `2, 4, ..., 12`， 有四种惯例


```
a) 		2 ≤ i < 13
b) 		1 < i ≤ 12
c) 		2 ≤ i ≤ 12
d) 		1 < i < 13
```

>Are there reasons to prefer one convention to the other? Yes, there are. The observation that conventions a) and b) have the advantage that the difference between the bounds as mentioned equals the length of the subsequence is valid. So is the observation that, as a consequence, in either convention two subsequences are adjacent means that the upper bound of the one equals the lower bound of the other. Valid as these observations are, they don't enable us to choose between a) and b); so let us start afresh.

有理由让我们更青睐于一种么？答案是肯定的。经过观察，a) 和 b) 的明显优势在于他们上界和下界的差等于子序列的长度。因此结果是，对于两者中的任意一种，两个相邻子序列意味着一个的上界等于另一个的下界。刚才的观察结果并不能使我们在 a) 和 b) 中做出选择；所以我们重新开始

>There is a smallest natural number. Exclusion of the lower bound —as in b) and d)— forces for a subsequence starting at the smallest natural number the lower bound as mentioned into the realm of the unnatural numbers. That is ugly, so for the lower bound we prefer the ≤ as in a) and c). Consider now the subsequences starting at the smallest natural number: inclusion of the upper bound would then force the latter to be unnatural by the time the sequence has shrunk to the empty one. That is ugly, so for the upper bound we prefer < as in a) and d). We conclude that convention a) is to be preferred.

存在最小的自然数。若像 b) 和 d) 中那样排除下界，从最小的自然数开始的子序列的下界会是一个非自然数。所以更倾向于使用 a) 与 c) 中的 `≤` 表示。现在考虑从最小自然数开始的子序列：当序列收缩至一个空序列时，包含上界会迫使后者变得不自然。所以对于上界更倾向于使用 a) 和 d) 中的 `<`。我们得出 a) 是更好的方式。

>Remark  The programming language Mesa, developed at Xerox PARC, has special notations for intervals of integers in all four conventions. Extensive experience with Mesa has shown that the use of the other three conventions has been a constant source of clumsiness and mistakes, and on account of that experience Mesa programmers are now strongly advised not to use the latter three available features. I mention this experimental evidence —for what it is worth— because some people feel uncomfortable with conclusions that have not been confirmed in practice. (End of Remark.)

备注【Xerox PARC 开发的 Mesa 编程语言有特殊的符号对应四种惯例里的整数间隔。Mesa 的丰富经验表明，使用其他三种惯例一直是笨拙和错误的根源，并且由于这种经验，Mesa 程序员强烈建议不要使用后三种。不管有沒有用，我提及这个实验证据是因为有些人对于在实践中没有得到确认的结论会感到不舒服。】

>When dealing with a sequence of length N, the elements of which we wish to distinguish by subscript, the next vexing question is what subscript value to assign to its starting element. Adhering to convention a) yields, when starting with subscript 1, the subscript range 1 ≤ i < N+1; starting with 0, however, gives the nicer range 0 ≤  i < N. So let us let our ordinals start at zero: an element's ordinal (subscript) equals the number of elements preceding it in the sequence. And the moral of the story is that we had better regard —after all those centuries!— zero as a most natural number.

当处理长度为 N 的序列时，我们希望通过下标来区分元素，下一个伤脑筋的问题是赋给起始元素什么样的下标值。采取惯例 a)，当从下标 1 开始时，下标范围便是 `1 ≤ i < N + 1`；然而从 0 开始，可以给出更好的范围 `0 ≤ i < N`。因此让我们的序数从零开始：一个元素的序数（下标）等于序列中它之前的元素的数量。

>Remark  Many programming languages have been designed without due attention to this detail. In FORTRAN subscripts always start at 1; in ALGOL 60 and in PASCAL, convention c) has been adopted; the more recent SASL has fallen back on the FORTRAN convention: a sequence in SASL is at the same time a function on the positive integers. Pity! (End of Remark.)

备注【许多已经被设计出来的编程语言设计没有对这个细节给予适当的关注。在 FORTRAN 语言中下标始终从 1 开始；ALGOL 60 和 PASCAL 中采用的是 c)】

>The above has been triggered by a recent incident, when, in an emotional outburst, one of my mathematical colleagues at the University —not a computing scientist— accused a number of younger computing scientists of "pedantry" because —as they do by habit— they started numbering at zero. He took consciously adopting the most sensible convention as a provocation. (Also the "End of ..." convention is viewed of as provocative; but the convention is useful: I know of a student who almost failed at an examination by the tacit assumption that the questions ended at the bottom of the first page.) I think Antony Jay is right when he states: "In corporate religions as in others, the heretic must be cast out not because of the probability that he is wrong but because of the possibility that he is right."

...
