# Horn Clauses & Datalog

A practical guide to writing `Rule(phi => psi)` declarations against the
typed graph, and to what actually runs today versus what is still a design
pattern waiting on an inference engine.

> **Where this lives:** intended as `docs/datalog_rules.md` in the
> `ner_20260608` source repo, as a companion to `docs/cookbook.md`.

> **Status up front.** `formal_spec.md` defines `Rule(phi => psi)` as a
> first-class part of the trait vocabulary, and describes what a derived
> instance looks like once one fires. As of this writing, no rule engine
> exists in `ner_20260608`. The named traits (`Transitive`, `Symmetric`,
> `Inverse`) are inert marker classes you can introspect, and exactly one
> traversal (`Graph.transitive_closure`) answers a transitivity query --
> without asserting anything new into $V$. Nothing in the codebase sets
> `extraction_method="inferred"`. This guide shows you how to write rules
> that respect the Datalog discipline the formal spec requires, so they
> are correct today as manually-invoked Python functions and portable
> later into a real fixed-point engine.

---

## 1. What a Horn clause is, and what Datalog restricts it to

A **Horn clause** is a disjunction of literals with at most one positive
literal. Written as an implication, that is a conjunction of positive
conditions implying a single conclusion:

```
body_1 AND body_2 AND ... AND body_k  =>  head
```

**Datalog** is the fragment of Horn clauses used for database-style
inference. It adds three restrictions on top of the general Horn clause:

1. **No function symbols.** Terms are variables or constants only --
   never `f(x)` for some function `f`. This keeps the set of derivable
   facts finite.

   A **term** is just anything that can fill an argument slot in a
   predicate. In `LocatedIn(x, y)`, the slots are filled by `x` and
   `y`. Datalog allows exactly two kinds of term:

   - a **constant** -- a specific, already-existing thing, e.g. the
     actual id `"wiki:London"`;
   - a **variable** -- a placeholder like `x` that gets *bound* to
     some specific thing already in $V$ when the rule is evaluated.

   What's excluded is a third kind of term: a **function symbol**
   applied to another term, producing a brand-new term that names
   something which may not exist yet -- e.g. `f(x)`, read as "the
   thing you get by applying `f` to `x`." The classic textbook example
   of why this is dangerous is defining the natural numbers with a
   "successor" function:

   ```
   number(0).
   number(s(X))  :-  number(X).
   ```

   `s(X)` means "the successor of X." Start from the one fact
   `number(0)` and this rule never stops finding new things to derive:
   `number(s(0))`, then `number(s(s(0)))`, then
   `number(s(s(s(0))))`, forever. There is no smallest, finite set of
   facts closed under this rule -- so "iterate to a fixed point"
   doesn't terminate, and the whole approach of "just keep applying
   rules until nothing new appears" (which is how this system's
   fixed-point evaluation works) breaks down.

   The same danger shows up in this system's own domain if a rule
   invents an entity instead of pointing at one that already exists.
   Suppose you wanted every `Location` to have a "containing region,"
   and wrote (informally):

$$
\text{LocatedIn}(x, y) \Rightarrow
\text{LocatedIn}(x, \text{region\_containing}(x))
$$

   `region_containing(x)` is a function symbol: it manufactures a new
   term -- a synthetic region that is not one of the `Location`
   instances already in $V$ -- for every `x` the rule sees. And because
   the rule's own output (`LocatedIn(x, region_containing(x))`) matches
   its own body pattern, it fires again on the thing it just invented,
   nesting `region_containing` inside itself without limit:
   `region_containing(region_containing(x))`, and so on. This is
   disallowed for exactly the same reason `s(X)` is disallowed.

   Contrast that with the transitivity rule this system does use:

$$
\text{LocatedIn}(x, y) \wedge \text{LocatedIn}(y, z)
\Rightarrow \text{LocatedIn}(x, z)
$$

   Here `x`, `y`, and `z` are only ever *bound* to `Location` instances
   that already exist in $V$ -- Briony Lodge, St. John's Wood, London.
   Nothing new is manufactured; the rule just notices a new
   *relationship* among things that were already there. Since the
   graph has only finitely many `Location` instances to begin with,
   there are only finitely many ways to bind `x`, `y`, and `z`, and
   therefore only finitely many `LocatedIn` facts this rule could ever
   produce. That is what "no function symbols... keeps the set of
   derivable facts finite" means in concrete terms: without a way to
   manufacture new terms, a rule can only ever recombine the finitely
   many things that were already in the graph.

2. **No negation.** The body is a pure conjunction of positive
   conditions. (Some Datalog dialects allow *stratified* negation; this
   system does not use it.)
3. **No existential variables in the head.** Every variable in the head
   must already appear in the body. A rule cannot invent a new entity --
   it can only assert a new relationship among things that already
   exist.

Those three restrictions are exactly what make Datalog **decidable**:
starting from a finite set of facts, repeatedly applying rules can only
ever produce a finite set of new facts, so the process terminates at a
**least fixed point** -- the smallest set of facts closed under every
rule. `formal_spec.md` names this explicitly: "the Datalog restrictions
-- no function symbols, no negation, no existential variables in the
head -- keep inference decidable and the semantics clean."

---

## 2. Mapping the general form onto this schema

`formal_spec.md` gives the shape of a rule with $n$ variables as:

$$
p_1(x_{a_1}, x_{b_1}) \wedge \cdots \wedge p_k(x_{a_k}, x_{b_k})\
\Rightarrow\ p_0(x_{a_0}, x_{b_0})
$$

Translate each symbol into this codebase's vocabulary:

- Each $p_i$ is a **predicate type** -- a `BaseStatement` subclass such as
  `LocatedIn` or `Involves`.
- Each $x_j$ is a **variable ranging over $V$** -- in practice, over
  entity or statement instances reachable in a `Graph`.
- An atom $p_i(x_a, x_b)$ is **not** an abstract relation lookup; it
  means "there exists an asserted instance of type $p_i$ whose `subject`
  field is bound to $x_a$ and whose `object_` field is bound to $x_b$."
  Concretely, that is one element of the list returned by
  `graph.edges_from(x_a.id, pred_type=p_i, truth="asserted_true")` whose
  `object_.id == x_b.id`.
- The head $p_0(x_{a_0}, x_{b_0})$ is a single instance of predicate type
  $p_0$ to construct once every body atom is satisfied by some binding.

So "evaluating the body" is graph traversal over the asserted subgraph,
and "firing the head" is constructing a new `BaseStatement` instance and
adding it to $V$.

A rule is a **Datalog rule** precisely when:

- every body atom is a positive predicate-instance lookup (no
  `truth_status != asserted_true` checks disguised as negation),
- the head is a single atom, not a conjunction,
- and every head variable is bound by some body atom.

If a candidate rule needs to check the *absence* of a fact, or needs to
invent a new entity that isn't already in $V$, it is not a Datalog rule
-- write it as ordinary Python outside this framework, and say so.

---

## 3. The named traits are canned Datalog rules

Four of the six entries in the trait vocabulary are Datalog rules with a
fixed shape, spelled directly in `formal_spec.md`:

| Trait | Equivalent rule |
|---|---|
| `Transitive` | $p(x, y) \wedge p(y, z) \Rightarrow p(x, z)$ |
| `Symmetric` | $p(x, y) \Rightarrow p(y, x)$ |
| `Inverse(p')` | $p(x, y) \Rightarrow p'(y, x)$ |

`Functional` and `InverseFunctional` are not rules in this sense --
they are integrity constraints (at most one object per subject, and vice
versa), not inference patterns that derive new facts. `Rule(phi =>
psi)` is the sixth trait, and it is the **escape hatch**: it exists for
every inference pattern that doesn't fit one of the three named shapes
above -- cross-predicate rules, multi-hop chains, or bodies with more
than two literals.

Declaring a trait today only marks the predicate type; it does not make
anything fire:

```python
class LocatedIn(BaseStatement, ProvenanceMixin, Transitive):
    subject: Location
    object_: Location

class DisguisedAs(BaseStatement, ProvenanceMixin, EpistemicMixin,
                   Inverse['HasTrueIdentity']):
    subject: Person
    object_: Persona
```

`issubclass(LocatedIn, Transitive)` tells you the predicate *should*
obey the transitivity rule. Nothing currently walks the schema, finds
every `Transitive`-tagged predicate, and applies the rule to the
instance graph. That's the gap this guide works around.

---

## 4. What runs today

Be precise about the one piece of trait-adjacent code that does exist,
`Graph.transitive_closure`:

```python
def transitive_closure(self, entity_id, pred_type,
                        truth_values=('asserted_true',)) -> set[str]:
    """All entities reachable from entity_id by following
    pred_type transitively."""
    ...
```

This is a **query**, not **materialization**. It answers "what is
reachable" by doing the BFS at call time; it never constructs a new
`LocatedIn(x, z)` instance, never assigns it an id, never stamps
provenance, and never adds it to `graph.by_id`. Run it twice and it does
the same walk twice. It also isn't generic over the `Transitive` trait
-- it takes whatever `pred_type` you hand it, transitive or not, and
happens to be used in the cookbook only against `LocatedIn`.

`Symmetric` and `Inverse` have no runtime counterpart at all beyond
`get_inverse()`, which just answers "what predicate type is declared as
this one's inverse" for introspection -- it does not construct the
inverse instance.

The gap between this and the formal spec's promise -- "every derived
instance enters $V$ as a full member with its own id and provenance ...
the `extraction_method` is `inferred`" -- is exactly the gap
`cookbook.md` flags in its Bohemia walkthrough: assembling the evidence
for `Possesses(Irene, photograph)` is not the same operation as deriving
it, and "the latter requires rule declarations and an inference engine
that this project does not yet have."

The rest of this guide shows you how to write rules that are ready for
that engine, and how to run them by hand until it exists.

---

## 5. The discipline for writing a rule today

Since there is no engine to enforce Datalog restrictions for you, hold
yourself to them when writing the Python function that stands in for a
rule:

1. **Body = read-only graph queries.** Use `graph.edges_from`,
   `graph.edges_to`, or `graph.bfs` with `truth="asserted_true"` (or an
   explicit `truth_values` tuple). Never mutate the graph while
   evaluating the body.
2. **One head shape per rule.** A rule function constructs instances of
   exactly one predicate type. If you need two conclusions, write two
   rules.
3. **No invented entities.** Every `subject` and `object_` on the head
   instance must be an entity or statement instance the body already
   bound -- never a freshly-constructed entity.
4. **Stamp provenance honestly.** Per R10, the head instance still needs
   `source` and `extraction_method`. Use `extraction_method="inferred"`
   and set `source` to identify the rule itself (e.g. the rule's name),
   so a derived fact's origin is auditable exactly like an extracted
   one.
5. **Respect truth_status.** A freshly-derived fact is a new claim the
   graph is committing to, not a hypothesis -- construct it with
   `truth_status=TruthStatus.ASSERTED_TRUE` unless the rule is
   explicitly modeling something weaker.
6. **Prefer `statement_id()` when the rule should be idempotent.**
   Content-addressed ids on `(subject, predicate_name, object_)` mean
   re-running the rule against an unchanged graph reconstructs the same
   id instead of piling up duplicate derived facts -- this is what makes
   "iterate to a fixed point" safe to implement as "keep calling the
   rule and merging by id until nothing new appears."

---

## 6. Worked example: materializing `LocatedIn` transitivity

The existing `transitive_closure` only answers reachability. Here is the
Datalog rule it corresponds to, written to actually assert new
`LocatedIn` facts:

$$
\text{LocatedIn}(x, y) \wedge \text{LocatedIn}(y, z)
\ \Rightarrow\ \text{LocatedIn}(x, z)
$$

```python
from ner_20260608.holmes_schema import (
    LocatedIn, TruthStatus, statement_id,
)

def rule_located_in_transitive(graph):
    """LocatedIn(x, y) AND LocatedIn(y, z) => LocatedIn(x, z).

    Returns newly-derived LocatedIn instances not already present
    in the graph (by content-addressed id).
    """
    derived = []
    for xy in list(graph.by_id.values()):
        if not isinstance(xy, LocatedIn):
            continue
        if xy.truth_status != TruthStatus.ASSERTED_TRUE:
            continue
        x, y = xy.subject, xy.object_
        for yz in graph.edges_from(
            y.id, pred_type=LocatedIn, truth="asserted_true"
        ):
            z = yz.object_
            if z.id == x.id:
                continue  # skip trivial self-loops
            new_id = statement_id(x.id, "LocatedIn", z.id)
            if new_id in graph.by_id:
                continue  # already derived (or already asserted)
            derived.append(LocatedIn(
                id=new_id,
                subject=x,
                object_=z,
                truth_status=TruthStatus.ASSERTED_TRUE,
                story_id=xy.story_id,
                paragraph_index=xy.paragraph_index,
                extraction_method="inferred",
                extraction_confidence=min(
                    xy.extraction_confidence, yz.extraction_confidence
                ),
            ))
    return derived
```

Note the two-hop chain the cookbook mentions in passing (Briony Lodge ->
St. John's Wood -> London) is exactly what this rule derives on its
first pass, and materializes as a real instance you can later query with
`edges_from`/`edges_to` like any extracted fact -- instead of only
through a bespoke closure method.

---

## 7. Worked example: the rule the cookbook leaves undone

`cookbook.md` walks through assembling every piece of evidence for
`Possesses(Irene, photograph)` from three `Involves` events, then
states plainly that deriving the statement itself "would require a rule
... and an inference engine to fire it. Neither exists yet." That rule
does not reduce to `Transitive`, `Symmetric`, or `Inverse` -- it is
exactly what the `Rule(phi => psi)` escape hatch is for: a
cross-predicate pattern specific to this domain, expressed in prose on
the predicate class and realized as a callable.

Stated informally as a Horn clause over events:

$$
\begin{aligned}
\text{Involves}(e_1, \text{King})\ \wedge\ &\text{Involves}(e_1,
\text{Irene})\ \wedge\ \\
&\text{"photograph" appears in } e_1.\text{description}\\
\Rightarrow\ &\text{Possesses}(\text{Irene}, \text{photograph})
\end{aligned}
$$

This is deliberately looser than pure Datalog -- "photograph appears in
the description" is a string test, not a predicate-instance lookup, so
it is not function-symbol-free in the strict sense. That is normal for
`Rule`: `formal_spec.md` says it "has no clean Python type-level
expression; it is declared in prose on the predicate class and realized
as a callable that the inference engine invokes." Keep the *structural*
part Datalog-clean (positive conjunction, single head, no negation) and
isolate the domain-specific test as an ordinary predicate function:

```python
from ner_20260608.holmes_schema import (
    Involves, Possesses, TruthStatus, statement_id,
)

def rule_possession_from_joint_photograph_event(graph, photograph_id):
    """If an Event involves both a person and Irene Adler, and the
    event's description mentions the photograph, assert that Irene
    possesses it.

    This is a Rule(phi => psi) declaration on Possesses, not one of
    the named traits -- it is specific to the Bohemia domain and has
    no generic Python expression.
    """
    derived = []
    for e in graph.edges_to(
        "wiki:Irene_Adler", pred_type=Involves, truth="asserted_true"
    ):
        event = e.subject
        if "photograph" not in event.description.lower():
            continue
        new_id = statement_id(
            "wiki:Irene_Adler", "Possesses", photograph_id
        )
        if new_id in graph.by_id:
            continue
        photograph = graph.get(photograph_id)
        derived.append(Possesses(
            id=new_id,
            subject=graph.get("wiki:Irene_Adler"),
            object_=photograph,
            truth_status=TruthStatus.ASSERTED_TRUE,
            story_id=event.story_id,
            paragraph_index=e.paragraph_index,
            extraction_method="inferred",
            extraction_confidence=e.extraction_confidence,
        ))
    return derived
```

Run against the pre-cutoff subgraph from the cookbook's own example
(`sentence_cutoff=485`), this rule fires on
`sib:event:joint_photograph_revealed` and produces the same
`Possesses(Irene, photograph)` fact that only appears in the real
extraction pipeline at sentence 511 -- turning the cookbook's "spoiler
check, not a proof" into an actual derivation, with `extraction_method`
honestly marked `"inferred"` so it is never confused with a
directly-extracted claim.

---

## 8. Worked example: materializing `Inverse`

$$
\text{DisguisedAs}(x, y) \Rightarrow \text{HasTrueIdentity}(y, x)
$$

```python
from ner_20260608.holmes_schema import (
    DisguisedAs, HasTrueIdentity, TruthStatus, statement_id, get_inverse,
)

def rule_inverse(graph, pred_type):
    """p(x, y) => p'(y, x), where p' = get_inverse(p).

    Generic over any predicate type declared with Inverse[...].
    """
    inverse_type = get_inverse(pred_type)
    if inverse_type is None:
        raise ValueError(f"{pred_type.__name__} has no declared inverse")

    derived = []
    for stmt in list(graph.by_id.values()):
        if not isinstance(stmt, pred_type):
            continue
        if stmt.truth_status != TruthStatus.ASSERTED_TRUE:
            continue
        new_id = statement_id(
            stmt.object_.id, inverse_type.__name__, stmt.subject.id
        )
        if new_id in graph.by_id:
            continue
        derived.append(inverse_type(
            id=new_id,
            subject=stmt.object_,
            object_=stmt.subject,
            truth_status=TruthStatus.ASSERTED_TRUE,
            story_id=stmt.story_id,
            paragraph_index=stmt.paragraph_index,
            extraction_method="inferred",
            extraction_confidence=stmt.extraction_confidence,
        ))
    return derived
```

Unlike the `LocatedIn` example, this one is written generically against
`get_inverse` rather than hardcoded to `DisguisedAs`/`HasTrueIdentity`
-- because `Inverse`, like `Transitive` and `Symmetric`, is a *named*
rule shape, and a rule runner should be able to discover and apply it
to every predicate type that declares the trait, not just one.

---

## 9. A naive fixed-point runner

"Rule application is iterated to a least fixed point" cashes out as:
keep calling every rule function against the current graph, merge
whatever comes back by id (so re-derivations collapse instead of
duplicating), and stop when a full pass adds nothing new.

```python
def run_to_fixed_point(graph, rules, max_rounds=50):
    """Apply every rule in `rules` repeatedly until no rule produces
    a new instance, or max_rounds is hit.

    Each rule is a callable(graph) -> list[BaseStatement]. Safe to
    call repeatedly because every rule above dedupes via
    statement_id() before constructing a new instance.
    """
    for round_num in range(max_rounds):
        new_this_round = []
        for rule in rules:
            for inst in rule(graph):
                if inst.id not in graph.by_id:
                    new_this_round.append(inst)

        if not new_this_round:
            return round_num  # reached the fixed point

        for inst in new_this_round:
            graph.by_id[inst.id] = inst
            graph.out_edges[inst.subject.id].append(inst)
            graph.in_edges[inst.object_.id].append(inst)

    raise RuntimeError(f"did not reach a fixed point in {max_rounds} rounds")
```

`max_rounds` is a safety valve, not part of the semantics -- for a
genuinely Datalog-restricted rule set (no function symbols, finite
domain of constants) the loop is guaranteed to terminate on its own,
because there are only finitely many possible ground atoms to derive.
The `rule_possession_from_joint_photograph_event` example above is not
strictly Datalog (it takes an extra `photograph_id` argument and does a
string test), so treat rules like it as reviewed-by-hand additions to
the fixed point, not as something a fully generic engine should
discover and apply on its own.

This is deliberately a sketch, not a proposal for `ner_20260608`'s
actual architecture -- it exists to show what "least fixed point" means
operationally, using only the `Graph` API that already exists.

---

## 10. Testing a rule

Follow the cookbook's synthetic-fixture pattern (`tests/conftest.py`) so
rule tests don't depend on the LLM-extracted Bohemia data and don't
break when an extraction changes:

```python
from ner_20260608.graph import Graph
from ner_20260608.holmes_schema import Location, LocatedIn, TruthStatus

_PROV = dict(
    story_id="test", paragraph_index=0,
    extraction_method="manual", extraction_confidence=1.0,
)

def test_located_in_transitive_derives_two_hop_chain():
    briony = Location(id="place:briony_lodge", display_name="Briony Lodge")
    st_johns = Location(id="place:st_johns_wood", display_name="St. John's Wood")
    london = Location(id="wiki:London", display_name="London")

    l1 = LocatedIn(
        id="stmt:l1", subject=briony, object_=st_johns,
        truth_status=TruthStatus.ASSERTED_TRUE, **_PROV,
    )
    l2 = LocatedIn(
        id="stmt:l2", subject=st_johns, object_=london,
        truth_status=TruthStatus.ASSERTED_TRUE, **_PROV,
    )
    g = Graph([briony, st_johns, london, l1, l2])

    derived = rule_located_in_transitive(g)

    assert len(derived) == 1
    assert derived[0].subject.id == "place:briony_lodge"
    assert derived[0].object_.id == "wiki:London"
    assert derived[0].extraction_method == "inferred"

def test_rule_is_idempotent_against_its_own_output():
    # ... build graph, run the rule, add the derived facts, run again ...
    # asserts the second pass returns an empty list.
    ...
```

Test the same three things for every rule you write: (1) it derives the
fact you expect from a minimal fixture, (2) it does not derive anything
when the body is unsatisfied, and (3) running it again after merging its
own output produces nothing new -- the fixed-point property, checked at
the unit level.

---

## 11. Where this fits in the book

`formal_spec.md` section "Rule(phi => psi) -- Datalog rules" (reproduced
in the Appendix) is the normative definition; this document is its
practitioner's companion, in the same relationship `cookbook.md` has to
the rest of the formal spec. If a working inference engine is added to
`ner_20260608` later, the rule functions and fixed-point runner sketched
here are the intended shape for it to formalize -- until then, treat
every "derived" fact in this system as something a human ran a rule
function to produce, not something the graph maintains on its own.
