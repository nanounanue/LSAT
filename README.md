# **Solving LSAT Logic Games: A Tale of Two Models** ##

## Objective
The Law School Admissions Test (LSAT), an exam designed to evaluate reading comprehension and logical reasoning skills, includes a section on [analytical reasoning](http://www.lsac.org/jd/lsat/prep/analytical-reasoning) better known as the "logic games" section.  The immediate goal of this project was to solve some of those logic games/puzzles and answer actual LSAT questions about them.  By testing two different solutions to this problem, however, I also explore larger questions about the roles that deep learning and linguistic expertise are likely to play in the future of Natural Language Processing (NLP).

## Motivation
Within NLP, semantic parsing or natural language understanding remains largely an unsolved problem.  Aside from corpora too vast for humans to read comfortably, the reading comprehension skills of the best NLP systems typically lag far behind those of humans.  This project explores the possibility that, even on a tiny corpus where size poses no challenge to a human reader, algorithms can sometimes outperform humans.  In the process, it investigates two different approaches to semantic parsing: one based loosely on syntactic parsers like Google's Parsey McParseface (building on decades of [research at Stanford](https://nlp.stanford.edu/software/lex-parser.shtml) and part of Google's larger [SyntaxNet](https://research.googleblog.com/2016/05/announcing-syntaxnet-worlds-most.html) project) and the other based on [Seq2Seq](https://research.googleblog.com/2017/04/introducing-tf-seq2seq-open-source.html), Google's state-of-the-art neural machine translation (NMT) engine, open-sourced on April 11, 2017.

## Data
My data consists of questions and answers from actual LSAT examinations like this:

![June2007](images/June_2007_sml.png)

Because those materials are copyrighted by [the company that produces them](https://www.lsac.org/), I have not included my data set in this repository.  Nor was I able to train my models on the entire corpus of publicly available LSAT tests.  For these initial prototypes, my data set was limited to a set of 50 puzzles (25 sequencing-type games and 25 grouping-type games).  When training a model to classify LSAT games as either sequencing or grouping puzzles, I trained on 45 games and held out 5 sets for testing.  For purposes of parsing the logical rules that accompany each game, I hand-labeled a set of 153 rules, with 138 rules in the training set and 15 held out for the test set.

##  The Seven Steps of the Puzzle-Solving Process
To answer LSAT questions about logic games, seven steps must be performed, whether the puzzle solver is a human or a machine.  Below, I describe how I approached each of these steps:

### Step 1: Classify the Game
Because most LSAT puzzles fall into one of two basic categories, sequencing games or grouping games, the solver's first task is to read the game's initial prompt and rules to determine which type it represents: a sequencing game, where the goal is to determine the permissible orderings for a set of variables, or a grouping game, where the object is to determine which variables may or may not belong within different sets.

I approached this step by building a stacked model, consisting of three linear classifiers, each of which feeds into a second-level random forest model.  Each of the three linear classifiers utilizes a different data source:

  1) the tokenized text of the prompt
  2) the tokenized text of the rules
  3) the Part of Speech (POS) tags for the rules

Each of these three models independently generates a probability that the game in question should be classified as a sequencing game.  Those three outputs are, in turn, fed into a random forest model that produces a final prediction.  Although I ran some grid searches to independently tune each component model, I did not test other configurations or classification models because my primary focus remained on Step 3 (below): parsing the logical rules that render the subsequent questions answerable.  In the future, I would nevertheless like to test Convolutional Neural Nets (CNN) for Step 1 because they have shown great promise as textual classifiers.

Note, too, that the remaining steps of my model(s) presuppose that the game represents a sequencing puzzle.  Given that my primary goal was to build prototypes of different models, I limited myself to one type of game (which comprises almost half of the LSAT logic games).

### Step 2: Set Up the Game
While I could have deployed spaCy's Named Entity Recognizer to identify the variables referenced in the prompts, I instead used spaCy's POS tagger and then extracted lists of the variable names from the prompts based on those POS tags and on the commas used to separate them.  Although my code can successfully handle various formats--such as compound variable names (most are single word or even a single letter) and lists that do or do not include the word "and" before the final element--I was not overly concerned with constructing a robust solution for this step of the problem.  

In addition, Step 2 populates a list of all the conceivable sequences of variables (before any rules have been applied to winnow that pool of candidates).  In other words, the initial pool consists of X factorial candidates, where X is the number of variables.  Thus, a game with 9 variables generates an initial pool of 9! = 362,880 permutations.

### Step 3: Parse the Rules
Step 3 represents the most challenging and interesting part of this project: What is the best way to convert an English-language statement of a logical rule into a Boolean expression that can be evaluated by Python's compiler (to produce a True or False result when tested on a candidate sequence of variables)?

This problem can be conceptualized in two different ways:

1) as a semantic parsing problem (for which a syntactic parser like Parsey McParseface provides a useful analogy)

2) as a translation problem, where the source language is English and the target language is Python (and Google's NMT engine, [Seq2Seq](https://research.googleblog.com/2017/04/introducing-tf-seq2seq-open-source.html), provides an elegant solution)

This project thus afforded me an opportunity to learn more about Google's state-of-the-art solutions to these two different problems, and challenged me to consider--as both a practical and a theoretical matter--which approach would work best for my problem.  To test both options, I coded a model for each.

#### Model #1: The Semantic Parser
The most interesting part of building the Parser was working out the unique grammar that governs the LSAT's sequencing games and considering how best to translate those sentence structures into a syntax that can be read by Python's compiler.  With the aid of LSAT test prep materials produced by companies like PowerScore and Kaplan, I discovered that the structures defining LSAT sequencing rules constitute a Context-Free Grammar.  

A Context-Free Grammar defines the ways in which grammatical units can be decomposed into subunits.  The phrase "context-free" signifies that a unit's permissible decompositions are not affected by its surrounding context.  At their root, all LSAT sequencing rules take one of two forms.  They either consist of a Boolean expresion like **A>B**, or they form a conditional statement like: **if A>B, then B>C**. In other words, every LSAT rule can be decomposed in one of those two ways.  But the Boolean expressions that define those two, still-generic types of sentence can themselves be decomposed in different ways.  **B** might be **greater than A**, **less than A**, **equal to A**, **not equal to A**, etc.  Nevertheless, each of those different comparisons assumes the same form: **(Set of Variables) (Some Kind of Comparison) (Another Set of Variables)**.  As a result, that structure represents one CFG transformation that can be applied to decompose a Boolean.  But it's not the only way.  A Boolean might also be decomposed into still more Booleans, for example: **((A>B) and (A>C))**.  

In fact, there is in theory no limit to the number of times that a Boolean can be decomposed in this nested manner.  For example, we could utilize these same basic CFG rules to compose a more complicated expression like **(((A and D> B) and (A>C)) or ((A and D)<B) and (A<C))**.  In other words, CFG's are powerful for the same reason that recursive functions are powerful: they enable you to combine and nest operations as many times as you want.  Through recursion, a finite set of grammatical operations becomes capable of generating an infinite variety of grammatical structures.  

In comparison to the extraordinary richness and diversity of ordinary English sentences, the possibilities for LSAT rule statements remain very narrowly circumscribed.  Whereas the CFG's used to train the state-of-the-art syntactical parsers produced by Stanford and Google must accommodate thousands of permissible CFG transformations/decompositions, only nine CFG transformations are needed for my problem:

![CFG](images/CFG_list_sml.png)

Those 9 CFG transformations are nevertheless capable of generating an infinite variety of logical/grammatical structures.  And, given more time, I would have preferred to implement those 9 rules within a properly recursive parser, capable of nesting structures many layers deep.  Such a model could, in theory, handle any number of branches and any level of depth.  Armed with fully recursive capabilities, a machine parser could likely outperform humans on complicated logical structures.

For purposes of handling the types of questions that typically appear on actual LSAT examinations, however, such recursive structures were largely unnecessary.  The rules that define most LSAT puzzles are relatively straightforward.  Casting LSAT rules as CFG's, one discovers that most rules can be parsed using only a handful of CFG expansions; the corresponding grammatical trees typically run only two to four layers deep.  Put differently, my Parser adds a pair of parentheses each time it applies a CFG transformation, yet its Python output rarely contains more than three or four nested sets of parentheses.  More importantly, the LSAT rarely requires that a given CFG transformation be applied more than once on a particular rule.  As a result, I was able to obtain relatively robust results even without implementing a genuinely recursive, full-featured parser.  

For purposes of prototyping, I used a much simpler model, which assumes limited nesting and relies on crude textual heuristics when applying the CFG transformations.  By expanding the number of those heuristics (crafting an ever-longer set of if-then alternatives), one could, of course, readily expand the model's reach.  But without proper recursion or any machine learning, such heuristics remain exceedingly fragile and limited in scope.  And even with recursion, the task of devising appropriate heuristics remains labor-intensive at best and ungeneralizable at worst.

#### Model #2: The Translator (Seq2Seq)
Precisely for that reason, I built a second model harnessing the incredible power and flexibility of deep learning.  Instead of painstakingly crafting a complex web of if-then heuristics, why not outsource that task to a neural net?  Specifically, why not see if one of the most promising new tools in NLP deep learning--Google's Seq2Seq--can be trained to transform English sentences into Python code entirely on its own?  Although Seq2Seq was initially created with natural language translation in mind, many researchers are already busy exploring its potential applicability in other contexts, including (at Google itself) image captioning and content summarization.  Indeed, my doubts about Seq2Seq's viability in the context of translating English to Python stemmed less from its inherent power than from my own ability, given the time constraints of this project, to hand label a sufficiently large training set.  Would 138 examples be enough for Seq2Seq to replicate my own attempts to map out the grammar governing the LSAT's sequencing games?

#### Step 4: Apply the Rules to Winnow the Pool of Possible Solutions to the Game  
Step 4 simply takes the set of logical rules extracted in Step 3 and applies those rules to the set of permutations that was generated in Step 2.  For example, if a game's first rule dictates that **variable A must come after variable B**, one would expect (ignoring any additional constraints) that half of the original permutations would comply with that rule and half would not.  Casting the rule as a test, half the permutations would pass and half would fail.  For instance, a game with 9 variables would start off with 362,880 conceivable permutations (9!).  But if a rule mandates that **A must come after B**, then only 181,440 permutations will survive application of that initial rule.  And if a second rule requires that **B come after C**, then another half of the candidates will be eliminated.  

Taken in combination, a game's rules thus serve to limit its permissable solutions.  For most games, the initial set-up rules winnow the pool of permissible solutions to fewer than 100 candidates.  On occasion, however, the set of acceptable candidates might include only a dozen or so permutations, which enables the test makers to pose questions like, "What is a complete and accurate list of the variables that may occupy the fifth position?"

#### Step 5: Parse the Questions  
Precisely because the pool of candidates emerging from Step 4 typically includes more than a dozen permissible orderings, many LSAT questions begin by imposing additional constraints that apply only in the context of that particular question.  Such questions are sometimes said to impose 'local' rules, as opposed to the 'global' rules parsed in Step 4.  To handle these local rules, Step 5 checks to determine whether the question begins with "If..." or with a verb like "Assume that..." since those are the ways that the test makers typically flag local rules.  

In addition to looking for local rules, one must also determine what kind of information the so-called "question stem" seeks.  Most often, solvers are asked to determine what "must be true", "must be false", "could be true", or "could be false", but various negations like "CANNOT be true" and exceptions like "could be false EXCEPT" must also be accommodated, along with a handful of more esoteric types.  Again, since Step 3 absorbed most of my attention, I did not implement functions to handle the less common question types; I concentrated on the most frequent types.

#### Step 6: Parse the Multiple Choice Answers
Like the LSAT's questions, the answers require some understanding of their potential formats.  Here, too, I relied on my domain knowledge respecting those formats to hard-code some straightforward parsing heuristics.  Just as human test takers quickly learn that an "If..." question imposes an additional, local rule on the game, so too do they quickly learn to recognize the potential answer formats.  Most of the answers consist of lists of variables (which require only rudimentary processing) or sentences (which require handling by the more sophisticated models connected to Step 3).  But, for my prototyping purposes, I again found it unnecessary to tackle the more obscure answer formats; I stuck to the types that define 90% of the LSAT's games.

#### Step 7: Pick the Correct Answer
The final step of the process is identifying the correct answer from among the 5 multiple-choice candidates.  Here, both humans and machines must carefully keep in mind the nature of the question being asked and test each answer with that particular question-type in mind.  Am I searching for an answer that "could be true", "could be false EXCEPT", etc.? 

## Results
### The Classification Problem from Step 1 
As described above, I was not overly concerned with getting great results on Step 1.  My initial efforts yielded 94% accuracy, which was sufficient to get me started:  

![CFG](images/step1_results.png)

### Step 3-Model #1: Using Grammar-Based Heuristics to Parse Rules
As described above, my grammar-based, heuristic parser remains very crude since it does not include the recursive functionality that would render it a proper parser.  Even without any machine learning or recursion, however, my algorithm sufficed to parse many of the rules because the nesting typically only involves a handful of CFG transformations and thus a handful of nested layers.

![parser_results](images/parser_results.png)

### Step 3-Model #2: Using Seq2Seq to Translate Rules from English into Python
For my Seq2Seq model, I simply adapted one of the architectures provided by Google in an example.  Although I played with a few of the parameters, like bucketing and batch size, I was unable to improve on the out-of-the-box results produced by Seq2Seq's default settings for its attention-based model.  Understanding the different parameters, models, and architectures associated with Seq2Seq will take some time.  As Google provides additional tutorials and as others begin to explore these new tools (which are, after all, only 2 months old), the community of people experimenting with them will no doubt grow.  For purposes of this project, I was happy simply to get a sense of Seq2Seq's power:

![seq2seq](images/seq2seq_results.png)

The most remarkable aspect of my results was not their (mediocre) accuracy, but rather evidence that Seq2Seq was genuinely learning and not merely memorizing.  After mapping Seq2Seq's integer outputs back onto my original vocabulary, I discovered that Seq2Seq was sometimes generating labels that were different from mine, but equally valid.  In other words, it was *creatively generating alternative solutions!*
In my labels, for example, I used **abs(A-B)==x** to express the difference between A and B.  Seq2Seq took a different approach.  It treated A>B and B<A as separate cases: **(((B-A)==x) or ((A-B)==x))**.  Its solution was thus equally valid and equally effective, but nevertheless different from the exact labeling system on which it was trained.  I was amazed that Seq2Seq was able to learn so much from only a small training set of 138 examples.  Its ability to generalize beyond the examples provided to it was truly impressive and stands in stark contrast to the fragility of my hand-crafted parsing model.

## Has Deep Learning Replaced Linguistic Expertise in NLP Research?

Some might conclude from this little experiment that linguistic expertise has little to offer NLP and that today's best neural nets can easily infer whatever patterns a human linguist is capable of identifying.  For a nuanced but skeptical discussion by a leading NLP researcher, Stanford's Chris Manning, see ["Computational Linguistics and Deep Learning: 
A look at the importance of Natural Language Processing"](http://mitp.nautil.us/article/170/last-words-computational-linguistics-and-deep-learning).  Google shares Manning's doubts that deep learning will replace human linguistic expertise anytime soon.  The best performing syntactic parsers in the world, Parsey McParseface and its siblings in Google's SyntaxNet project, continue to rely on Context-Free Grammars devised by human linguists.  And while some researchers at Google have achieved impressive results building syntactic parsers without any CFG's whatsoever (substituting enormous, synthetically generated training sets of 11M labeled sentences!), Google continues to invest heavily in tools that both exploit and increase our understanding of syntax.  Despite the undeniable power of neural nets like Seq2Seq to learn grammatical structures entirely on their own, Google is not yet convinced that grammar is dead, and neither am I.  Indeed, [recent research](https://arxiv.org/pdf/1702.01147.pdf) in Machine Translation suggests that neural nets trained with the benefit of syntactic information perform better than neural nets trained without such information.  Accordingly, rather than simply abandoning my parsing model in favor of deep learning, I might in the future try to combine the two and enjoy the best of both worlds: the interpretability of a CFG combined with the power of a neural net.

## References
### Textbooks
[_Foundations of Statistical Natural Language Processing_](https://nlp.stanford.edu/fsnlp/)  
Christopher D. Manning, Prabhakar Raghavan and Hinrich Schütze

[_Speech and Language Processing_](https://web.stanford.edu/~jurafsky/slp3/)  
Dan Jurafsky and James Martin

[_Introduction to Information Retrieval_](https://nlp.stanford.edu/IR-book/)  
Christopher D. Manning, Prabhakar Raghavan and Hinrich Schütze

### Articles
[Globally Normalized Transition-Based Neural Networks](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45377.pdf)  
Daniel Andor, Chris Alberti, David Weiss, Aliasksei Severyn, Alessandro Presta, Kuzman Ganchev, Slav Petrov and Michael Collins

[Neural Machine Translation by Jointly Learning to Align and Translate](https://arxiv.org/pdf/1409.0473.pdf)  
Dzimitry Bahdanau, KyungHyun Cho, Yoshua Bengio

[Announcing SyntaxNet: The World’s Most Accurate Parser Goes Open Source](https://research.googleblog.com/2016/05/announcing-syntaxnet-worlds-most.html)  
Google Research

[Introducing tf-seq2seq: An Open Source Sequence-to-Sequence Framework in TensorFlow]
(https://research.googleblog.com/2017/04/introducing-tf-seq2seq-open-source.html)  
Google Research

[Syntax-aware Neural Machine Translation Using CCG](https://arxiv.org/pdf/1702.01147.pdf)  
Maria Nadejde, Siva Reddy, Rico Sennrich, Tomasz Dwojak, Marcin Junczys-Dowmunt, Philipp Koehn, Alexandra Birch

[Improved Semantic Representations From Tree-Structured Long Short-Term Memory Networks](https://arxiv.org/pdf/1503.00075.pdf)  
Kai Sheng Tai, Richard Socher, Christopher D. Manning

[Grammar as a Foreign Language](https://arxiv.org/pdf/1412.7449.pdf)  
Oriol Vinyals, Lukasz Kaiser, Terry Koo, Slav Petrov, Ilya Sutskever, Geoffrey Hinton


### Courses Covering Parsing and Machine Translation
[Natural Language Processing (Columbia)](http://www.cs.columbia.edu/~cs4705/)    
Michael Collins

[Natural Language Processing with Deep Learning (Stanford)](http://web.stanford.edu/class/cs224n/)    
Richard Socher and Chris Manning
