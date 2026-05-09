# RegexToFA

A web tool that converts regular expressions into finite automata. Enter any regex and instantly get the NFA, DFA, or minimized DFA with an interactive graph visualization.



## Features

- **Regex → NFA / DFA / Min DFA** — type a regular expression and pick which automaton you want to see
- **Interactive graph** — the generated automaton is rendered as a draggable, zoomable graph with labeled transitions
- **String tester** — test whether a string is accepted or rejected, step-by-step or automatically, with adjustable speed
- **Null string support** — type `""` in the regex input or the tester input to represent the empty string (λ)
- **Spaces are ignored** — write `(a + b)*` or `(a+b)*`, both work the same

## Supported Syntax

| Symbol | Meaning |
|--------|---------|
| `a`, `b`, `0`, `1`, … | alphabet characters |
| `+` | union (or) |
| `*` | Kleene star (zero or more) |
| juxtaposition (`ab`) | concatenation |
| `( )` | grouping |
| `""` or `λ` | null / empty string |

### Examples

| Expression | Language |
|------------|---------|
| `(a+b)*ab` | strings over {a,b} ending with ab |
| `(aa+bb)*` | even-length strings of all a's or all b's |
| `(aa+bb+(ab+ba)(aa+bb)*(ab+ba))*` | even number of both a's and b's |
| `(0+1+"")*` | any binary string including the empty string |

## Running Locally

Make sure you have Node.js installed (v16+).

```bash
# install dependencies
npm install

# start dev server
npm run dev
```

Open `http://localhost:5173` in your browser.

## Building for Production

```bash
npm run build
```

Output goes to the `dist/` folder.

## Project Structure

```
src/
├── utilities/
│   ├── finiteAutomata.js    # core regex parser + NFA/DFA/MinDFA engine
│   └── array.js             # array comparison helper
├── components/
│   ├── Automata/
│   │   ├── AutomataBox.vue      # main container (input + graph + tester)
│   │   ├── AutomataGraph.vue    # vis-network graph rendering
│   │   └── AutomataTester.vue   # string tester with step/play controls
│   └── Base/
│       ├── BaseInput.vue        # reusable input component
│       └── BaseSelect.vue       # reusable select component
├── views/
│   └── HomePage.vue         # landing page
├── App.vue
└── main.js
```

## Tech Stack

- **Vue 3** (Composition API)
- **Vite** (dev server + bundler)
- **vis-network** (graph rendering)
- **SCSS** (styling)
