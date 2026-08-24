# Orientation: coming from React Native

You have not built this kind of system before, so here is an honest account of
which parts you already know and which parts are actually new.

## What carries over

More than you would expect. Roughly two thirds of the code in this project is
JavaScript.

The backend is Node.js. The ETL is Node.js. The frontend is almost certainly
going to be React. You already write JavaScript daily, you already know npm and
`package.json`, and you already know promises and async/await. None of that
changes.

If issue #3 picks React for the frontend, the component model is identical to
what you know. Hooks, state, props, and the render cycle work the same way.
What changes is the surface: `div` instead of `View`, CSS instead of
`StyleSheet.create`, react-router instead of react-navigation, and a browser
instead of Metro. Your instincts about component structure and state carry
over intact.

You have spent years calling REST APIs from a mobile client. In M3 you write
the other side of that same contract. The mental model does not change, only
the direction.

Git, branches, and pull requests are the same as anywhere.

## What is genuinely new

Two areas, and it is worth being clear about which is which.

### Running a Linux server

nginx, pm2, SSH hardening, backups, TLS. Most mobile engineers never touch
any of it, and this is the larger of the two gaps.

The good news is that it is not conceptually hard. It is procedure work:
install a thing, edit a config file, restart the service, check that it
responds. There is very little to reason about. The bad news is that it is
unforgiving of carelessness, which is a different problem from being difficult.
See the risks section below.

Milestones M1 and M2 are almost entirely this.

### RDF, SPARQL, and ontology modeling

This looks scarier than it is, and it is the part worth spending Step 0 of the
roadmap on.

You know objects and you know SQL tables. RDF is neither. Instead of storing a
record with fields, you store individual facts, each one a three-part
statement: subject, predicate, object. These are called triples.

Where you would write this in JavaScript:

```js
{ id: "pc-42", type: "Desktop", inLab: "lab-b", ram: 16 }
```

RDF stores four separate facts:

```turtle
:pc-42  rdf:type   :Desktop .
:pc-42  :inLab     :lab-b .
:pc-42  :ramGb     16 .
:lab-b  rdf:type   :Lab .
```

Every subject and predicate is a URI rather than a local string key, which is
what lets two datasets refer to the same thing without coordinating first.
There are no tables. Adding a new property to one asset means adding one more
line, not running a migration.

The ontology you design in issue #2 declares which classes and properties
exist and how they relate. Think of it as a type definition that the data
store actually knows about, rather than one that only your editor knows about.

SPARQL is the query language, and it is closer to SQL than you are probably
fearing. You write a pattern with variables in it and the engine finds every
subgraph that matches:

```sparql
SELECT ?asset ?labName WHERE {
  ?asset rdf:type :Desktop .
  ?asset :inLab   ?lab .
  ?lab   :name    ?labName .
}
LIMIT 20
```

Read that as: find everything that is a Desktop, follow its `inLab` link,
and get the name of whatever it points at. `?asset`, `?lab`, and `?labName`
are variables. `SELECT`, `LIMIT`, `OFFSET`, `ORDER BY`, `COUNT`, and `FILTER`
all mean what they mean in SQL.

The queries are not the hard part. The modeling is. That is why issue #2 comes
before issue #4, and why the roadmap suggests playing with sample data before
either of them.

## Where the real risk is

Three places in this project where a mistake actually costs you something.
None of them are about being clever.

Issue #5 asks you to turn off SSH password authentication. Get the key setup
wrong and you lock yourself out of the container with no console to recover
from. Keep a second SSH session open while you make the change, and confirm
key login works from a third session before you close either one.

Issue #20 has you writing SPARQL updates. A `DELETE WHERE` with a pattern that
is more general than you intended will delete more than you meant, and there is
no undo. Count the affected triples with a `SELECT` before you run the
`DELETE`, and make sure the backup script from issue #9 actually works before
you get here.

Issue #8 handles secrets. Add `.env` to `.gitignore` before you create the
file, not after. Removing a committed password from git history is a separate
job you do not want.

## Debugging habits worth having early

Test SPARQL in the GraphDB workbench before putting it in code. The workbench
has a query editor with syntax highlighting and readable error messages. Your
Node SPARQL client will just hand you a 400.

When a Turtle file will not load, GraphDB's error names the line number. Turtle
syntax is mostly about the periods at the end of statements, and that is
usually what is wrong.

When a page does not load in your browser, do not debug nginx and the SSH
tunnel at the same time. First run `curl http://localhost:80` from inside the
container. If that works, the problem is the tunnel. If it does not, the
problem is nginx. Isolating those two will save you more time than anything
else in this document.

## A realistic expectation

M0 is writing and will feel slow because you are not producing anything that
runs. M1 and M2 are the unfamiliar part and will take longer than you estimate,
mostly spent reading documentation for tools you have not used. M3 is the
longest milestone but the most familiar one, since it is mostly JavaScript and
React. M4 is Node scripting plus the data modeling you already settled in M0.

If a milestone is taking much longer than the one before it, that is normal
here rather than a sign something is wrong.
