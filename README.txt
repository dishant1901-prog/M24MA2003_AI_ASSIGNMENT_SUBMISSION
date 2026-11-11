Reasoning Agent - AI Assignment
Dishant Kaushik, M24MA2003
===============================

Files
-----
reasoning_agent.ipynb   whole implementation, experiments, traces, final run
report.pdf              short report (representation, algorithms, experiments, traces)
submission.csv          predictions, columns: id, flag (True / False)
README.txt              this file


Requirements
------------
Python 3.10 or newer (I used 3.13)
pandas
nltk (only the Porter stemmer is used, no downloads needed)
spacy + the small English model     pip install spacy
                                    python -m spacy download en_core_web_sm
    (optional: spaCy and nltk are only loaded if a problem comes in plain
     English. For logic input they are never imported, and if they are
     missing an English FOLIO style problem is simply answered False.)
datasets, huggingface_hub           only needed if the data files are not already in DATA_DIR

No GPU is required. Internet is only needed for the first download of the data.


How to run
----------
1. Set TEST_FILE to the test file before starting jupyter
       Windows:   set TEST_FILE=C:\path\to\test.csv
       Linux/Mac: export TEST_FILE=/path/to/test.csv
   and run all cells (Kernel -> Restart & Run All). The last cell prints the
   number of queries, the total inference time, the accuracy (if the file has
   labels) and writes submission.csv (columns id, flag) next to the notebook.
   With TEST_FILE set, the experiments on the public data are skipped, nothing
   is downloaded, and the whole notebook runs in a few seconds plus the test
   itself. Set RUN_EXPERIMENTS=1 as well to run the experiments anyway.
2. The file can be csv, json or jsonl. Column names that are recognised:
       context   : context / premises / theory / given_info / facts
       statement : statement / question / query / conclusion / hypothesis
       label     : flag / label / answer / gold         (optional; True/False,
                   entailment/not entailment, yes/no or 1/0)
       id        : id / example_id / question_id / qid  (row number if missing)
   premises-FOL / conclusion-FOL columns are used if they are there.
   The context can be one formula per line, formulas separated by ';' or by
   '. ', or a JSON list of formulas.
3. Without TEST_FILE the last cell runs on a public evaluation file in logic
   form (8000 RuleTaker test questions, 203 FOLIO validation problems with
   their FOL, 1200 CounterBench questions as equations).
4. Data is read from DATA_DIR (default ./data) and downloaded from Hugging Face
   if missing: tasksource/ruletaker, tasksource/folio, CounterBench/CounterBench.

Without TEST_FILE the whole notebook takes about 10 minutes, almost all of
it is the experiments section. The final cell alone takes under 10 seconds.


Logic notation that is accepted
-------------------------------
The instructor said the test will give the logic directly, but not in which
notation, so the parser takes all the usual ones:

    FOLIO / Unicode   ∀x (Young(x) ∧ ¬Kind(x) → Smart(x))     Young(bob)
    ASCII             forall x (young(x) & ~kind(x) -> smart(x))
                      all x. ..., exists x ..., !p, -p, not p, |, or, and,
                      =>, <->, <=>, ^ / xor, !=
    Prolog style      smart(X) :- young(X), \+ kind(X).        young(bob).
    causal equations  Vork = Blaf ∧ Trune      Blaf ↔ ¬Ziklo      Trune
    ProofWriter       ("Bob" "is" "young" "+")
                      ((("someone" "is" "young" "+")) -> ("someone" "is" "smart" "+"))

- Free x, y, z, u, v, w are read as universally quantified.
- In Prolog clauses capitalised arguments are variables.
- Predicate names are not case sensitive, CamelCase names are split into words
  (FemaleTennisPlayer -> female_tennis_player).
- The query can be a literal, a conjunction or any formula.
- Lines may have numbers or labels in front (1. / (2) / P1: / Rule 3: / - ),
  several facts may share a line separated by commas, and a line that does
  not parse is skipped instead of failing the whole problem.


How it works
------------
All reasoning is classical; nothing generative is used anywhere.

1. Logic input, Horn theories -> forward chaining (closed world)
   If every formula is a ground literal, a conjunction of ground literals, or
   "for all x: conjunction of literals -> literal(s)" (negation allowed), and
   the query is a ground literal or a conjunction of them, the theory goes to
   the forward chainer. "not X" means X cannot be derived (negation as
   failure), which is how RuleTaker is labelled. Rules are run stratum by
   stratum; when there is a cycle through negation the rules are grounded and
   the well-founded model is computed with the alternating fixpoint.
   Why closed world for Horn theories: on the Horn problems inside FOLIO
   (open world labels) it costs 3 of 132 on train and 0 of 20 on validation,
   while open world gets RuleTaker's negated questions wrong (numbers in the
   report and notebook section 7).

2. Logic input, anything else -> resolution (open world)
   Clause form (remove ->, <->, xor; negation normal form; rename variables;
   Skolemise; distribute), then resolution refutation with unification and
   occurs check, set of support, and the cheapest clause (number of literals +
   term depth) is expanded next - a uniform cost / best-first search.
   A proof stops after 500 derived clauses (2 s cap only as a safety net),
   so the answers do not depend on the speed of the machine. Answer True
   only if the goal is proved and its negation is not. A premise that does
   not parse is skipped instead of failing the whole problem.

3. Counterfactual questions -> structural equations
   "Would Y occur if not X instead of X?", "Would Y occur if not X and not Z?",
   "Assume not X, and based on this assumption, further suppose not Z. Would Y
   occur?" over a model given as English ("A and B together cause not C") or as
   equations. Abduction: enumerate root values that agree with the
   observations and with X being true in the actual world. Action: set the
   intervened variables. Prediction: propagate forward through the equations.
   True if Y comes out true in every consistent world.

4. English input (fallback)
   - Simple theories (RuleTaker sentences): hand written patterns into the
     same facts and rules as in 1.
   - Other English (FOLIO sentences): spaCy for lemmas / POS tags / subject,
     then rules for quantifiers, if-then, either-or, neither-nor, relative
     clauses and pronouns, into first order formulas for the prover in 2.
     Here word subset axioms are added: if the words of predicate B are a
     strict subset of the words of predicate A, then A(x) -> B(x). This
     route uses a bigger search budget (3000 clauses, 1 s).

There is also a backward chainer (depth first SLD with negation as failure,
loop check and memo) that is used for comparison and to print proof trees.

Labels: the task is entailment, so "Unknown"/"Uncertain" (FOLIO),
"not entailment" (RuleTaker) and "no" (CounterBench) all count as False.


Results on the public data
--------------------------
Final cell, public evaluation file in logic form (9403 queries):
    total inference time 7.66 s (0.81 ms per query), accuracy 99.03 %
    RuleTaker 100.00 %, CounterBench 95.00 %, FOLIO 84.73 %

RuleTaker test as logic, all 151,911 questions   100.00 % in Unicode, ASCII and Prolog notation (about 8 s)
RuleTaker test from English                       100.00 % (about 4.4 s)
FOLIO validation, gold FOL                         84.73 % binary (always-False baseline 64.53 %)
FOLIO validation from English                      67.49 % binary
CounterBench, English or equations                 95.00 % (1140 / 1200)

The experiments, the closed/open world comparison and the reasoning traces
are in sections 7 and 8 of the notebook and in the report.
