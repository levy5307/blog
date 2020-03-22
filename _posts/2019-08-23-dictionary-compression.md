Dictionary compression

The compression techniques we have seen so far replace individual symbols with a variable length codewords. 
In dictionary compression, variable length substrings are replaced by short, possibly even fixed length codewords.
Compression is achieved by replacing long strings with shorter codewords.
The general scheme is as follows:
The general scheme is as follows:
    • The dictionary D is a collection of strings, often called phrases. For completeness, the dictionary includes all single symbols.
    • The text T is parsed into a sequence of phrases: T = T1T2 . . . Tz, Ti ∈ D. The sequence is called a parsing or a factorization of T with respect to D.
    • The text is encoded by replacing each phrase Ti with a code that acts as a pointer to the dictionary.
Here is a simple static dictionary encoding for English text:
    • The dictionary consists of some set of English words plus individual symbols.
    • Compute the frequencies of the words in some corpus of English texts. Compute the frequencies of symbols in the corpus from which the dictionary words have been removed.
    • Number the words and symbols in descending order of their frequencies.
    • To encode a text, replace each dictionary word and each symbol that does not belong to a word with its corresponding number. Encode the sequence of number using γ coding.
