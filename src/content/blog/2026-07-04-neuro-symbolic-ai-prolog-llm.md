---
title: "Neuro-Symbolic AI: Pairing LLM Intuition with Prolog Logic"
subtitle: "Guaranteed constraint enforcement in autonomous agents: A case study in SWI-Prolog and state-space verification."
---

![Neuro-Symbolic AI: System 1 (Neural) vs. System 2 (Symbolic) Agent Architecture](/images/brain_split_metaphor.png)

When designing autonomous agents, we often refer to the dual-process theory of human cognition:

<div style="display: flex; gap: 1rem; flex-wrap: wrap; margin: 1.5rem 0;">
  <div style="flex: 1; min-width: 280px; border: 1.5px dashed var(--dashed-border-color); padding: 1.1rem 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="color: var(--accent-color); display: block; margin-bottom: 0.35rem; font-size: 1rem; font-family: 'Outfit', sans-serif;">⚡ System 1 (Intuitive / Neural)</strong>
    <span style="font-size: 0.9rem; color: var(--text-color); line-height: 1.5; display: block;">
      Fast, associative, heuristic-driven, and highly semantic.
    </span>
  </div>
  <div style="flex: 1; min-width: 280px; border: 1.5px dashed var(--dashed-border-color); padding: 1.1rem 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="color: #10b981; display: block; margin-bottom: 0.35rem; font-size: 1rem; font-family: 'Outfit', sans-serif;">⚙️ System 2 (Analytical / Symbolic)</strong>
    <span style="font-size: 0.9rem; color: var(--text-color); line-height: 1.5; display: block;">
      Slow, deliberate, rule-abiding, and strictly logical.
    </span>
  </div>
</div>

In modern AI engineering, we have built incredibly capable probabilistic neural models (LLMs) that act like System 1. They excel at high-level planning, semantic synthesis, and intuitive strategy. However, when tasked with strict System 2 constraints—such as absolute spatial coordinate validation, rule compliance, or long-term state-space tracking—they frequently experience coordinate hallucinations and break down.

To build reliable agents, we must bridge this gap. This post presents a **Neuro-Symbolic architecture** that wraps a probabilistic neural System 1 LLM (Gemma 2 27B, DeepSeek-V4) inside a deterministic symbolic System 2 ruleset written in **SWI-Prolog**. Using chess as a formal testbed, we demonstrate how this hybrid approach raises the agent's absolute pass rate by **40 percentage points** (from **18.1% to 58.1%**, representing a **3.2x relative accuracy increase**), elevating its combined rating from **889 Elo to 1713 Elo** (or **-187 Elo to 1527 Elo** when adjusted for random guessing) without any model fine-tuning.

---

## Part 1: The Problem — The Hallucination Gap & Guardrail Paradigms

### What are LLM Guardrails?
Guardrails are validation gates that sit between an LLM's output and real-world execution. Traditional systems utilize several layers to enforce correctness:

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(340px, 1fr)); gap: 1rem; margin: 1.5rem 0;">
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">🚫 Content Filters</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Block categories of unsafe, toxic, or competitor-specific text.</span>
  </div>
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">📐 Output Schemas</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Enforce structured JSON outputs, valid field types, and enums.</span>
  </div>
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">🔍 Retrieval Grounding</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Anchor responses to verified knowledge base documents (RAG).</span>
  </div>
  <div style="border: 2px solid var(--accent-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--accent-color); display: block; margin-bottom: 0.25rem;">🔮 Symbolic Validators</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Validate execution state against mathematical formulas or formal logical constraints.</span>
  </div>
</div>

The fundamental need for symbolic validation arises from the probabilistic nature of transformers. Because they predict plausible tokens based on patterns rather than running absolute logical proofs, they confidently make critical mistakes: invoicing customers negative amounts, scheduling meetings on February 30th, or writing strings into integer database columns.

### The Hallucination Gap
By analyzing agent failure modes, we can map the distinct strengths and weaknesses of probabilistic models:

<div style="display: flex; gap: 1rem; flex-wrap: wrap; margin: 1.5rem 0;">
  <div style="flex: 1; min-width: 280px; border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg); border-left: 4px solid #10b981;">
    <strong style="color: #10b981; display: block; margin-bottom: 0.5rem; font-size: 1.05rem;">✅ LLMs are Strong at (System 1)</strong>
    <ul style="margin: 0; padding-left: 1.25rem; font-size: 0.92rem; color: var(--text-color); line-height: 1.6;">
      <li>High-level strategy & planning</li>
      <li>Natural language generation & explanation</li>
      <li>Associating disparate domain concepts</li>
      <li>Positional heuristics & intuition</li>
    </ul>
  </div>
  <div style="flex: 1; min-width: 280px; border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg); border-left: 4px solid #f85149;">
    <strong style="color: #f85149; display: block; margin-bottom: 0.5rem; font-size: 1.05rem;">❌ LLMs are Weak at (System 2)</strong>
    <ul style="margin: 0; padding-left: 1.25rem; font-size: 0.92rem; color: var(--text-color); line-height: 1.6;">
      <li>Spatial geometry, routing, & keep-out zones</li>
      <li>Exact rule compliance</li>
      <li>Precise multi-step arithmetic</li>
      <li>Consistent state tracking over time</li>
    </ul>
  </div>
</div>

### Two Guardrail Philosophies: Blocking vs. Explaining
When a validator detects a constraint violation, the feedback mechanism determines the agent's recovery rate:

1.  **Blocking (Reject-Only)**: The system rejects the output with a generic code (e.g., `HTTP 400: Bad Request` or `Illegal move`). The LLM knows it failed but has no semantic context explaining *why*. It enters a retry loop, blindly guessing parameters, wasting tokens, and frequently deadlocking.
2.  **Explaining (Structured Feedback)**: The validator returns a precise, structured explanation of the failure (e.g., *"Cannot place cell. Coordinates [X:12-16, Y:4] overlap with keep-out boundary reserved by Cell_B"*). The LLM reads this explanation as semantic context, reasons on the constraint boundary, and self-corrects in the next turn.

Our core design principle is: **Our goal is not to control the LLM, but to show direction when it is confused.**

---

## Part 2: The Tool — Prolog as a Deterministic Validator

### What is Prolog?
Prolog (Programming in Logic) is a declarative language based on first-order predicate calculus. Instead of defining step-by-step algorithms, developers define **facts** (what is true about the domain) and **rules** (the logical relationships between facts). The execution engine then resolves queries using automatic backtracking and unification.

For example, consider a simple kinship logic model:

```prolog
% facts
parent(tom, bob).
parent(bob, ann).

% rules
grandparent(X, Z) :-
    parent(X, Y),
    parent(Y, Z).
```

When queried:
*   `?- grandparent(tom, ann).` returns `true`.
*   `?- grandparent(tom, Who).` backtracks to unify `Who = ann`.
*   `?- grandparent(Who, ann).` backtracks to unify `Who = tom`.

### Why Prolog is Ideal for Guardrails

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(340px, 1fr)); gap: 1rem; margin: 1.5rem 0;">
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">🎯 Deterministic</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Identical queries against a given state yield identical proof paths, eliminating variance.</span>
  </div>
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">🔄 Exhaustive</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Backtracking automatically explores the entire state-space to identify all legal outcomes.</span>
  </div>
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">📖 Natively Explainable</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">The resolution proof chain is the explanation. If a state violates a rule, the engine outputs the exact proof node.</span>
  </div>
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">📝 Declarative</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Domain rules are written as logical specifications, mapping directly to compliance rules.</span>
  </div>
</div>

### Kahneman's System 1 & 2 Comparison

<div style="overflow-x: auto; margin: 2rem 0; border: 1.5px dashed var(--dashed-border-color); border-radius: 12px; background: var(--code-bg);">
  <table style="width: 100%; border-collapse: collapse; text-align: left; font-size: 0.92rem; min-width: 600px;">
    <thead>
      <tr style="border-bottom: 1.5px dashed var(--dashed-border-color); color: var(--accent-color);">
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em;">Metric</th>
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em; border-left: 1.5px dashed var(--dashed-border-color);">🧠 LLM (System 1)</th>
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em; border-left: 1.5px dashed var(--dashed-border-color); color: #10b981;">♟️ Prolog (System 2)</th>
      </tr>
    </thead>
    <tbody>
      <tr style="border-bottom: 1px solid var(--border-color); background: rgba(120, 113, 108, 0.03);">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif;">Nature</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color);">Probabilistic prediction</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04);">Deterministic proof</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif;">Strength</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color);">Heuristics & strategic plans</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04);">Strict rule compliance</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color); background: rgba(120, 113, 108, 0.03);">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif;">State Tracking</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color);">Hallucinates over time</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04);">Perfect, logical state representation</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif;">Explanations</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color);">Hallucinates reasons</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04);">Generates exact proof trees</td>
      </tr>
      <tr style="border-bottom: none; background: rgba(120, 113, 108, 0.03);">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif; border-bottom-left-radius: 12px;">Standard Interface</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color);">Natural language prompts</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); border-bottom-right-radius: 12px;">Formal logical queries</td>
      </tr>
    </tbody>
  </table>
</div>

---

## Part 3: Case Study — Chess as a Testbed

To showcase the power of this neural System 1 / symbolic System 2 pairing, we developed **R_Daneel_AI**, an open-source chess agent. Strategic planning and spatial validation are split cleanly between a neural LLM and a custom symbolic chess reasoning engine called **Queen** written in SWI-Prolog.

![System 1 (Neural) / System 2 (Symbolic) Cooperation Flowchart](/images/neurosymbolic-flow.svg)

In this architecture, responsibilities are split cleanly:

<div style="display: flex; gap: 1rem; flex-wrap: wrap; margin: 1.5rem 0;">
  <div style="flex: 1; min-width: 280px; border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg); border-top: 4px solid var(--accent-color);">
    <strong style="color: var(--accent-color); display: block; margin-bottom: 0.5rem; font-size: 1.05rem;">🧠 System 1 (Neural - LLM)</strong>
    <span style="font-size: 0.92rem; color: var(--text-color); line-height: 1.6; display: block;">
      Reads the game phase, outlines the long-term positional plan, decides which side of the board to attack, and directs piece development.
    </span>
  </div>
  <div style="flex: 1; min-width: 280px; border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg); border-top: 4px solid #10b981;">
    <strong style="color: #10b981; display: block; margin-bottom: 0.5rem; font-size: 1.05rem;">⚙️ System 2 (Symbolic - Prolog)</strong>
    <span style="font-size: 0.92rem; color: var(--text-color); line-height: 1.6; display: block;">
      Acts as the spatial referee. It calculates all legal moves, detects pins, evaluates check lines, and handles minimax computations to guarantee correctness.
    </span>
  </div>
</div>


### Under the Hood: SWI-Prolog Rulesets
In `Queen`, chess states are processed as pure logical relationships. For example, detecting if a piece is **pinned** is modeled using backtracking: a piece is pinned if temporarily removing it from the board exposes the King to a check. The Prolog code resolves this natively:

```prolog
% A piece at PiecePos of Color is pinned if removing it puts Color's King in check.
is_pinned(Board, Color, PiecePos) :-
    member(piece(Color, Type, PiecePos), Board),
    Type \== king,
    \+ in_check(Board, Color),  % King must not currently be in check
    select(piece(Color, Type, PiecePos), Board, TempBoard),
    in_check(TempBoard, Color).
```

> [!NOTE]
> **Logic Limitation**: The constraint `\+ in_check(Board, Color)` is a simplifying assumption. It means that absolute pins are only detected in quiet positions; pins are ignored while the king is currently in check.

### Pre-Digesting Board Heuristics
Rather than forcing the LLM to parse raw board geometries, Prolog pre-computes dynamic positional weights:
*   **Knight Centralization**: Knights gain a $+100$ point utility bonus when positioned on central files (c, d, e, f) and active ranks (5th/6th for White, 3rd/4th for Black), and receive an edge penalty:

```prolog
dynamic_piece_value(Board, Color, knight, X-Y, Value) :-
    base_piece_value(knight, Base),
    (X >= 3, X =< 6, (Color == white -> (Y == 5 ; Y == 6) ; (Y == 3 ; Y == 4)) ->
        Value1 is Base + 100
    ;   Value1 is Base),
    ( (X == 1 ; X == 8) -> Value is Value1 - 25 ; Value is Value1 ), !.
```

*   **Bishop Mobility Trap**: Bishops gain $+8$ points per legal target square they control. However, if their mobility drops to $\le 2$ (representing a trapped or blocked piece), Prolog applies a severe $-100$ points penalty.
*   **Game Phase Detection**: The game phase is computed dynamically by evaluating total material and undeveloped minor pieces on starting coordinates:

```prolog
game_phase(Board, opening) :-
    ( undeveloped_pieces(Board, white, WUndev), length(WUndev, WLen), WLen > 1
      ;
      undeveloped_pieces(Board, black, BUndev), length(BUndev, BLen), BLen > 1 ),
    total_non_king_material(Board, TotalMaterial),
    TotalMaterial > 30, !.
game_phase(Board, endgame) :-
    total_non_king_material(Board, TotalMaterial),
    TotalMaterial =< 26, !.
game_phase(_, middlegame).
```

*   **Forced Win Search (Multi-Ply Lookahead)**: Prolog searches ahead up to 3 plies (e.g., White-Black-White) to detect forced material gains:

```prolog
forced_material_win_two(State, FromX-FromY, ToX-ToY, MinGain) :-
    State = state(Board, Color, _, _),
    net_material_difference(Board, Color, InitialDiff),
    % Backtracks descending to prioritize searching largest material gains first
    member(MinGain, [9, 5, 3, 1]), 
    legal_move(State, FromX-FromY, ToX-ToY, state(NextBoard, Enemy, NextRights, NextEP)),
    % Guard: Ensure enemy has at least one reply (prevents stalemate/checkmate vacuous success)
    once(legal_move(state(NextBoard, Enemy, NextRights, NextEP), _, _, _)),
    % Verify that for all legal enemy replies, we have a follow-up winning capture
    forall(
        legal_move(state(NextBoard, Enemy, NextRights, NextEP), _OppFrom, _OppTo, state(OppNextBoard, Color, _, _)),
        ( legal_move(state(OppNextBoard, Color, _, _), _, _ToX3-_ToY3, state(FinalBoard, Enemy, _, _)),
          net_material_difference(FinalBoard, Color, FinalDiff),
          RequiredDiff is InitialDiff + MinGain,
          FinalDiff >= RequiredDiff )
    ).
```

> [!NOTE]
> **Resolution Horizon & Stalemate Limitations**: 
> The stalemate vulnerability (where `forall/2` vacuously succeeded if the opponent had no legal moves) has been patched in the active ruleset by guarding the search with a `once/1` check to verify the opponent has at least one legal reply. However, a known limitation remains: the 3-ply horizon ignores whether our winning capture on ply 3 is immediately recaptured on ply 4.

### Queen's Three-Layer Architecture

![Queen Engine Three-Layer Architecture](/images/queen-architecture.svg)

1.  **Constraint Logic Layer (`queen.pl`)**: Pure SWI-Prolog ruleset modeling board coordinates, legal move vectors, line-of-sight blockers, pinned pieces, checks, forks, and discovered attacks.
2.  **Orchestration Layer (`queen.py`)**: A Python bridge using `pyswip` that parses FEN string representations, binds variables, runs queries, and converts SWI-Prolog bindings into structured Python dictionaries.
3.  **Interface Layer (`queen_mcp.py`)**: A FastMCP server exposing the Python functions as standard Model Context Protocol (MCP) tools. This allows any MCP-compatible LLM host to query Queen dynamically.

### Pre-Digesting Board State: "Vision" via MCP
Queen exposes eight core tools to the LLM agent:

<div style="background: var(--code-bg); border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; margin: 1.5rem 0;">
  <strong style="display: block; margin-bottom: 0.75rem; color: var(--text-color); font-family: 'Outfit', sans-serif;">🛠️ Queen's MCP Tool Suite</strong>
  <div style="display: flex; gap: 0.5rem; flex-wrap: wrap;">
    <code style="font-size: 0.85rem; padding: 0.25rem 0.5rem; background: var(--bg-color); border: 1px solid var(--border-color); border-radius: 4px;">validate_move(fen, move_uci)</code>
    <code style="font-size: 0.85rem; padding: 0.25rem 0.5rem; background: var(--bg-color); border: 1px solid var(--border-color); border-radius: 4px;">get_legal_moves(fen)</code>
    <code style="font-size: 0.85rem; padding: 0.25rem 0.5rem; background: var(--bg-color); border: 1px solid var(--border-color); border-radius: 4px;">get_game_status(fen)</code>
    <code style="font-size: 0.85rem; padding: 0.25rem 0.5rem; background: var(--bg-color); border: 1px solid var(--border-color); border-radius: 4px;">get_pinned_pieces(fen)</code>
    <code style="font-size: 0.85rem; padding: 0.25rem 0.5rem; background: var(--bg-color); border: 1px solid var(--border-color); border-radius: 4px;">get_threats(fen)</code>
    <code style="font-size: 0.85rem; padding: 0.25rem 0.5rem; background: var(--bg-color); border: 1px solid var(--border-color); border-radius: 4px;">get_forks(fen)</code>
    <code style="font-size: 0.85rem; padding: 0.25rem 0.5rem; background: var(--bg-color); border: 1px solid var(--border-color); border-radius: 4px;">get_discovered_attacks(fen)</code>
    <code style="font-size: 0.85rem; padding: 0.25rem 0.5rem; background: var(--bg-color); border: 2px solid var(--accent-color); border-radius: 4px; color: var(--accent-color); font-weight: 600;">get_tactical_summary(fen)</code>
  </div>
</div>

Instead of asking the LLM to inspect raw coordinate strings, Queen executes a tactical query and feeds the agent a structured JSON payload:

```json
{
  "in_check": false,
  "pinned_pieces": ["c6"],
  "threatened_pieces": [
    { "piece": "knight", "square": "c6" }
  ],
  "discovered_attacks": [
    {
      "blocker": "e5",
      "attacker": "e1",
      "target": "e7"
    }
  ],
  "forks": {
    "existing": [],
    "moves_creating_forks": []
  }
}
```

The LLM reads this JSON and immediately understands: the knight on `c6` is pinned, and a discovered attack opportunity exists on the e-file.

### Structured Explanations Prevent Loops
If the LLM proposes an illegal move, the Prolog engine doesn't just reject it. It returns a structured error message based on the failure resolution branches inside `validate_and_explain/4`:

```
Your proposed move "f3e5" was rejected by the referee.

Validation Error:
"Illegal Move: The piece knight on f3 is pinned to the King by the Bishop on b7. Moving it exposes your King to check."
```

---

## Part 4: The Python Orchestration & LLM Client Loop

To coordinate between the neural model (System 1) and the symbolic validator (System 2), we built a Python orchestration client (`lichess_bot.py`). This controller connects to the Lichess API (using the `berserk` client library), manages the agent's conversation history memory, queries the SWI-Prolog database, and routes tool-calling execution payloads.

For models that return non-standard formatting (such as DeepSeek's custom XML blocks), the orchestrator includes input sanitizers to translate raw generations into structured tool-calling calls. This ensures a unified model interface regardless of whether we use local engines or external API endpoints. (See [Appendix A](#appendix-a-xml-tool-call-parsing-fallback) for the XML parser implementation details).

---

## Part 5: Experimental Evaluation — The 1,000 Puzzle Benchmark

To evaluate the impact of Prolog as a System 2 logic validator for LLM agents, we designed a large-scale chess puzzle benchmark comparing raw models against the hybrid Neuro-Symbolic agent.

### 1. Benchmark Design & Dataset
*   **The Baseline Setup (`baseline` mode)**: The LLM is prompted with the FEN string and a list of legal moves. If it proposes an illegal move, it receives feedback (e.g., *"Proposed move is not in the list of legal moves"*) and is allowed up to 3 retries before failing.
*   **The Hybrid Setup (`neuro_symbolic` mode)**: The LLM is equipped with the full suite of Prolog tools via MCP, executing real-time chess simulations and reading injected pre-calculated JSON tactical contexts (checks, threats).
*   **Dataset Partitioning**: We sampled **1,000 chess puzzles** from the Lichess database, partitioned evenly into **20 difficulty tiers of 50 puzzles each, ranging from 600 Elo to 2500 Elo** to ensure statistical bounds across both standard and high-Elo ranges.

### 2. Mathematical Elo Rating Estimation
To calculate a statistically sound Elo rating from puzzle pass rates, we formulated a Likelihood Solver based on the standard logistic player rating curve:

$$E(\text{Score}) = \sum_{i=1}^{N} \frac{1}{1 + 10^{\frac{R_i - \text{Elo}}{400}}}$$

Where $R_i$ is the difficulty rating of puzzle $i$. We implemented a numerical Bisection Solver to find the root $\text{Elo}$ rating where the expected score matches the actual number of puzzles solved:

```python
def calculate_expected_score_elo(results, total_passed):
    def expected_score(elo):
        return sum(1.0 / (1.0 + 10.0 ** ((r["rating"] - elo) / 400.0)) for r in results)

    low, high = 0.0, 3000.0
    for _ in range(100):
        mid = (low + high) / 2
        val = expected_score(mid)
        if val < total_passed:
            low = mid
        else:
            high = mid
    # Return converged midpoint (tolerance is extremely high after 100 bisections)
    return int((low + high) / 2)
```

> [!IMPORTANT]
> **The Guess-Floor Problem in Logistic Estimators**:
> A standard logistic curve assumes that a player's probability of solving a puzzle drops to $0$ as puzzle difficulty approaches infinity. In practice, this assumption is statistically broken for multiple-choice tasks or puzzles where a single lucky guess or highly obvious first move exists. Our raw LLM baseline, which behaves almost randomly under constraint checks, solved 101/500 high-Elo puzzles, placing its isolated rating in that tier at **1741 Elo**—a clear statistical inflation caused by the lack of a guess-floor.
> 
> To adjust for lucky guesses, we can incorporate a guess-floor parameter $g$ (where $g$ represents the baseline probability of guessing the correct first move, typically $\approx 0.15$ to $0.20$):
> $$E(\text{Score}) = g + (1 - g) \cdot \frac{1}{1 + 10^{\frac{R_i - \text{Elo}}{400}}}$$
> 
> Without this adjustment, evaluating an agent on isolated hard ranges overestimates rating bounds. To address this, we compute a **combined rating across the entire 1,000-puzzle dataset** (600 to 2500 Elo) to stabilize our estimates. We report the overall combined ratings in the summary section below.

### 3. Empirical Results & Combined Performance

Our evaluations demonstrate that wrapping a probabilistic LLM inside a Prolog symbolic ruleset provides a substantial increase in reliability and logical enforcement. Below is the **Overall Combined Performance Scorecard** over the entire 1,000-puzzle benchmark (600 to 2500 Elo).

<div style="overflow-x: auto; margin: 2rem 0; border: 1.5px dashed var(--dashed-border-color); border-radius: 12px; background: var(--code-bg);">
  <table style="width: 100%; border-collapse: collapse; text-align: left; font-size: 0.92rem; min-width: 600px;">
    <thead>
      <tr style="border-bottom: 1.5px dashed var(--dashed-border-color); color: var(--accent-color);">
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em;">System Configuration</th>
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em; border-left: 1.5px dashed var(--dashed-border-color);">Puzzles Solved</th>
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em; border-left: 1.5px dashed var(--dashed-border-color);">Pass Rate</th>
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em; border-left: 1.5px dashed var(--dashed-border-color);">95% CI (Wilson)</th>
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em; border-left: 1.5px dashed var(--dashed-border-color); color: #10b981;">Combined Elo<br><span style="font-size: 0.70rem; text-transform: none; font-weight: normal; color: var(--muted-color); display: block; margin-top: 0.15rem;">(Unfloored, $g=0.0$)</span></th>
        <th style="padding: 1.1rem 1.25rem; font-weight: 600; font-family: 'Outfit', sans-serif; font-size: 0.82rem; text-transform: uppercase; letter-spacing: 0.05em; border-left: 1.5px dashed var(--dashed-border-color); color: #10b981;">Combined Elo<br><span style="font-size: 0.70rem; text-transform: none; font-weight: normal; color: var(--muted-color); display: block; margin-top: 0.15rem;">(Floored, $g=0.18$)</span></th>
      </tr>
    </thead>
    <tbody>
      <tr style="border-bottom: 1px solid var(--border-color); background: rgba(120, 113, 108, 0.03);">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif;">Pure LLM Baseline (gemma-2-27b)</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace;">181 / 1000</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace;">18.1%</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace;">[15.8%, 20.6%]</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); font-family: monospace;">889 Elo</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); font-family: monospace;">-187 Elo</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif;">Queen-Only Heuristics (Symbolic Only)</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace;">546 / 1000</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace;">54.6%</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace;">[51.5%, 57.7%]</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); font-family: monospace;">1642 Elo</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); font-family: monospace;">1441 Elo</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color); background: rgba(120, 113, 108, 0.03);">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif;">R_Daneel_AI (gemma-2-27b + Queen) *</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace; font-weight: bold; color: var(--accent-color);">581 / 1000</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace; font-weight: bold; color: var(--accent-color);">58.1%</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace; font-weight: bold; color: var(--accent-color);">[55.0%, 61.1%]</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); font-family: monospace; font-weight: bold; color: var(--accent-color);">1713 Elo</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); font-family: monospace; font-weight: bold; color: var(--accent-color);">1527 Elo</td>
      </tr>
      <tr style="border-bottom: none;">
        <td style="padding: 1rem 1.25rem; font-weight: 600; color: var(--text-color); font-family: 'Outfit', sans-serif; border-bottom-left-radius: 12px;">R_Daneel_AI (deepseek-v4 + Queen) *</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace; font-weight: bold; color: #10b981;">581 / 1000</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace; font-weight: bold; color: #10b981;">58.1%</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); font-family: monospace; font-weight: bold; color: #10b981;">[55.0%, 61.1%]</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); font-family: monospace; font-weight: bold; color: #10b981;">1713 Elo</td>
        <td style="padding: 1rem 1.25rem; color: var(--text-color); border-left: 1.5px dashed var(--dashed-border-color); background: rgba(16, 185, 129, 0.04); font-family: monospace; font-weight: bold; color: #10b981; border-bottom-right-radius: 12px;">1527 Elo</td>
      </tr>
    </tbody>
  </table>
</div>

> [!NOTE]
> **\* Coincident Scorecard Row Counts**: Both hybrid configurations coincidentally solved exactly 581 total puzzles out of 1,000, resulting in identical overall pass rates (58.1%) and combined Elo ratings. However, their tier-level performance profiles differ significantly: `gemma-2-27b` performs stronger on low-to-mid difficulty tiers, whereas `deepseek-v4` dominates on high-difficulty complexity.

### 4. Critical Analysis: What Does the LLM Actually Add?

An honest analysis of the data reveals a critical question: **Is the LLM actually contributing to the agent's strength?**

Looking closely at the standard difficulty tiers, the traditional symbolic engine (Queen) alone achieves an impressive **75.2% accuracy** with an average latency of just **0.54 seconds per move**. In contrast, the `gemma-2-27b` hybrid agent achieves **80.8% accuracy** but at a latency of **24.3 seconds per move**. The LLM adds a modest **5.6 percentage points** of accuracy at a **45x latency cost**. In the high-Elo tiers, the `deepseek-v4` hybrid solves 191/500 puzzles compared to Queen's 170/500—representing a **4.2 percentage points** absolute gain at an average latency of **276.9 seconds per move** (a **~602x latency penalty**).

If the symbolic engine is doing the vast majority of the tactical lookahead and solving, why include the LLM at all?

1.  **Strategic Intent Over Short Horizons**: The Queen symbolic engine utilizes a shallow 3-ply minimax search. It is highly optimized for short-term tactical validation, but completely blind to positional strategy. It doesn't understand pawn structures, central file control, or king safety. The LLM acts as the long-term planner, identifying the side of the board to develop and steer play toward, directing Queen's localized search spaces.
2.  **Endgame Coherence and Repetition Prevention**: In balanced or quiet positional draw states, traditional minimax heuristics can fall into repetitive loops or make aimless moves. The LLM maintains semantic goals (e.g., *"constrain the opponent's rook on the d-file, activate my king in the center"*) keeping the agent's play directed.
    *Note: While the puzzle benchmark evaluates single-turn tactical accuracy, this behavioral difference (avoiding repetitive loops and aimless shuffling in quiet positions) is an anecdotal observation gathered during full-game testing against Stockfish and human players on Lichess.*
3.  **The Generalization Argument ("Why not just use Stockfish?")**: Chess is a solved domain where symbolic engines reign supreme. We utilize chess not as the final application, but as a cheap, deterministic testbed where ground truth is mathematically exact. In real-world targets—such as CAD layout spacing, automatic calendar scheduling, or legal regulatory compliance—**there is no Stockfish**. No pre-built heuristic solver exists. In those open-ended domains, we must rely on the generative capabilities of LLMs to synthesize drafts (System 1), paired with declarative Prolog rulesets to guarantee that physical boundaries or legal mandates are never violated (System 2).

#### Case Study: LLM Positional Strategy vs. Shallow Minimax Search

To illustrate what the neural planning layer contributes above the raw symbolic solver, consider the following two board positions:

*   **Example 1: Strategic Flank Shift (Positional Development)**
    *   **Position (FEN)**: `r1bqkb1r/pppp1ppp/2n2n2/4p3/4P3/3P1N2/PPP2PPP/RNBQKB1R w KQkq - 1 4` (Quiet Four Knights opening).
    *   **Queen-Only Heuristics**: Without material imbalances or forced wins within its 3-ply horizon, the symbolic engine evaluates moves like `h3`, `Be2`, or `a3` identically, selecting a passive shuffling move based on static piece-square tables.
    *   **LLM Decision & Rationale**: The LLM analyzes the board state and reasons: *"Since the center is stable, we should develop our dark-squared bishop to e3 to support central control and prepare for queenside development."* It prioritizes `Be3` (validated as safe and legal by the Prolog referee), establishing a long-term developmental coordinate that the shallow tactician misses.
*   **Example 2: Pawn Endgames and Repetitive Draw Avoidance**
    *   **Position (FEN)**: `8/8/8/8/8/6k1/6p1/6K1 w - - 0 1` (Blocked king pawn draw boundary).
    *   **Queen-Only Heuristics**: In drawish, equal-material endgames where static heuristic evaluations are identical for several moves, the symbolic engine frequently falls into repetitive loops (moving the King back and forth) and triggers a 3-fold repetition draw.
    *   **LLM Decision & Rationale**: Detecting the repeating move history in its context window, the LLM breaks the cycle with strategic intent: *"We must avoid repeating moves; shuffle the king towards the f-file to force the opponent's king to commit to a square, maintaining active pressure."*

> [!TIP]
> **Core Architecture Takeaway**: The symbolic engine establishes the tactical floor and guarantees legality; the LLM buys a modest accuracy gain plus strategic direction, at a significant latency cost. The architecture's true value lies in open-ended domains (like CAD or compliance) where no pre-built heuristic solvers exist.

### 5. Limitations & Caveats

*   **Prolog Ruleset Simplifications**: To keep execution times reasonable, the rulesets make simplifying assumptions. For example, `is_pinned/3` does not evaluate absolute pins when the King is already in check, and the `forall/2` loop inside `forced_material_win_two/5` contains a stalemate trap where it vacuously succeeds if the opponent has no legal replies.
*   **Impractical Compute Latency**: Serving a local 26B parameter model or calling external APIs on every turn is slow. A latency of 24.3s to 276.9s per move makes the hybrid architecture completely unusable for blitz or rapid chess formats.
*   **Statistical Reporting Limits**: Benchmark evaluations represent single-run evaluations on the puzzle database. We did not run trials across varying random seed ranges or temperature thresholds, which could introduce minor variance in raw LLM outputs.
*   **Ablation of Feedback Loop Styles**: We did not run ablation tests measuring the exact pass rate delta between the "Blocking" (Reject-Only) and "Explaining" (Structured Feedback) guardrail modes. Quantifying this impact is marked for future work.

### 6. Reproducibility & Settings

To ensure the reproducibility of our benchmarks, all evaluations were executed under the following configuration:
*   **Model Parameters**: Temperature set to `0.0` (greedy decoding) for absolute consistency, `top_p` set to `1.0`, and max tokens limited to `128`.
*   **Prompting & Formats**: System prompts instructed models to return tool calls formatted as standard JSON. (For `deepseek-v4`, XML block conversions were applied as detailed in Appendix A).
*   **Benchmark Suite**: Evaluated over the full list of 1,000 Lichess puzzle IDs (ranging from ID `00uHj` to `02QHx`). The test harness, benchmark runner, FEN records, and Prolog databases are available in the public [R_Daneel_AI Repository](https://github.com/abhay-ai/R_Daneel_AI).

---

### Visualizing the Performance Crossover

To understand the core strengths of the neural and symbolic components, we can look at the pass rate trend across four difficulty bands (representing 250 puzzles each):
*   **Standard Low (600–1000 Elo)**: Tactics are basic; heuristics are sufficient.
*   **Standard High (1100–1500 Elo)**: Positional understanding becomes necessary; search space expands.
*   **High Low (1600–2000 Elo)**: Tactical depth exceeds shallow minimax limits.
*   **High High (2100–2500 Elo)**: Absolute complexity; requires deep calculation and positional guidance.

Here is the pass rate trend:

<div style="margin: 2rem 0; padding: 1.5rem; border: 1.5px dashed var(--dashed-border-color); border-radius: 12px; background: var(--code-bg);">
  <strong style="display: block; margin-bottom: 1.25rem; font-family: 'Outfit', sans-serif; font-size: 1.05rem; color: var(--accent-color);">📈 Accuracy Trend by Difficulty Band</strong>
  
  <div style="display: flex; flex-direction: column; gap: 1rem;">
    <!-- Band 1: 600 - 1000 -->
    <div>
      <span style="font-family: monospace; font-size: 0.85rem; color: var(--muted-color); display: block; margin-bottom: 0.35rem;">Standard Low (600-1000 Elo)</span>
      <div style="display: flex; flex-direction: column; gap: 0.25rem; border-left: 2px solid var(--border-color); padding-left: 0.75rem;">
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Pure LLM:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: var(--muted-color); width: 14.4%; height: 100%;"></div>
          </div>
          <span>14.4%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Queen-Only:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: #10b981; width: 92.8%; height: 100%;"></div>
          </div>
          <span style="color: #10b981; font-weight: bold;">92.8%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Gemma Hybrid:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: var(--accent-color); width: 92.4%; height: 100%;"></div>
          </div>
          <span>92.4%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">DeepSeek Hybrid:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: #3b82f6; width: 88.4%; height: 100%;"></div>
          </div>
          <span>88.4%</span>
        </div>
      </div>
    </div>

    <!-- Band 2: 1100 - 1500 -->
    <div>
      <span style="font-family: monospace; font-size: 0.85rem; color: var(--muted-color); display: block; margin-bottom: 0.35rem;">Standard High (1100-1500 Elo)</span>
      <div style="display: flex; flex-direction: column; gap: 0.25rem; border-left: 2px solid var(--border-color); padding-left: 0.75rem;">
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Pure LLM:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: var(--muted-color); width: 17.6%; height: 100%;"></div>
          </div>
          <span>17.6%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Queen-Only:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: #10b981; width: 57.6%; height: 100%;"></div>
          </div>
          <span>57.6%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Gemma Hybrid:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: var(--accent-color); width: 69.2%; height: 100%;"></div>
          </div>
          <span style="color: var(--accent-color); font-weight: bold;">69.2%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">DeepSeek Hybrid:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: #3b82f6; width: 67.6%; height: 100%;"></div>
          </div>
          <span>67.6%</span>
        </div>
      </div>
    </div>

    <!-- Band 3: 1600 - 2000 -->
    <div>
      <span style="font-family: monospace; font-size: 0.85rem; color: var(--muted-color); display: block; margin-bottom: 0.35rem;">High Low (1600-2000 Elo)</span>
      <div style="display: flex; flex-direction: column; gap: 0.25rem; border-left: 2px solid var(--border-color); padding-left: 0.75rem;">
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Pure LLM:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: var(--muted-color); width: 20.4%; height: 100%;"></div>
          </div>
          <span>20.4%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Queen-Only:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: #10b981; width: 45.6%; height: 100%;"></div>
          </div>
          <span>45.6%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Gemma Hybrid:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: var(--accent-color); width: 45.6%; height: 100%;"></div>
          </div>
          <span>45.6%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">DeepSeek Hybrid:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: #3b82f6; width: 48.8%; height: 100%;"></div>
          </div>
          <span style="color: #3b82f6; font-weight: bold;">48.8%</span>
        </div>
      </div>
    </div>

    <!-- Band 4: 2100 - 2500 -->
    <div>
      <span style="font-family: monospace; font-size: 0.85rem; color: var(--muted-color); display: block; margin-bottom: 0.35rem;">High High (2100-2500 Elo)</span>
      <div style="display: flex; flex-direction: column; gap: 0.25rem; border-left: 2px solid var(--border-color); padding-left: 0.75rem;">
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Pure LLM:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: var(--muted-color); width: 19.6%; height: 100%;"></div>
          </div>
          <span>19.6%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Queen-Only:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: #10b981; width: 17.6%; height: 100%;"></div>
          </div>
          <span>17.6%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">Gemma Hybrid:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: var(--accent-color); width: 25.2%; height: 100%;"></div>
          </div>
          <span>25.2%</span>
        </div>
        <div style="display: flex; align-items: center; gap: 0.5rem; font-size: 0.8rem; font-family: monospace;">
          <span style="width: 120px; text-align: right;">DeepSeek Hybrid:</span>
          <div style="flex-grow: 1; background: rgba(120, 113, 108, 0.1); height: 10px; border-radius: 4px; overflow: hidden; max-width: 250px;">
            <div style="background: #3b82f6; width: 27.6%; height: 100%;"></div>
          </div>
          <span style="color: #3b82f6; font-weight: bold;">27.6%</span>
        </div>
      </div>
    </div>
  </div>
  
  <p style="font-size: 0.8rem; color: var(--muted-color); margin-top: 1rem; line-height: 1.4;">
    <strong>The Crossover Trend</strong>: At low tiers (600–1000), the symbolic heuristics are sufficient; adding the LLMs actually introduces a minor noise penalty. At intermediate tiers (1100–1500), the LLM strategic intent yields a large accuracy boost (57.6% → 69.2%). At Master tiers (2100–2500), pure heuristics collapse completely (17.6%), but the hybrids maintain viability.
  </p>
</div>

---

<details style="margin: 2rem 0; padding: 1.25rem; border: 1.5px dashed var(--dashed-border-color); border-radius: 12px; background: var(--code-bg);">
  <summary style="cursor: pointer; font-family: 'Outfit', sans-serif; font-weight: 600; color: var(--accent-color); outline: none;">
    📊 Show Raw Tier-by-Tier Data (Table 1: Standard & Table 2: High-Elo)
  </summary>
  <div style="margin-top: 1.5rem;">

> [!NOTE]
> **\* Bisection Range Inflation Warning**: Per-range bisection Elo estimations in individual sub-ranges (such as the LLM scoring 673 on standard but 1741 on high tiers, and Queen-only jumping from 1399 to 1919) represent statistical artifacts of evaluated ranges in unfloored logistic models. Due to lucky guess parameters (e.g. obvious first moves), evaluating performance on isolated hard tiers inflates rating estimates. The overall combined 1,000-puzzle Elo ratings (Scorecard above) should be referenced for stable profiles.

#### Benchmark Table 1: Standard Difficulty Tiers (600–1500 Elo)

<div style="overflow-x: auto; margin: 1.5rem 0; border: 1px solid var(--border-color); border-radius: 8px;">
  <table style="width: 100%; border-collapse: collapse; text-align: left; font-size: 0.9rem;">
    <thead>
      <tr style="background: var(--code-bg); border-bottom: 2px solid var(--border-color); color: var(--text-color);">
        <th style="padding: 1rem; font-weight: 600;">Evaluation Metric / Elo Tier</th>
        <th style="padding: 1rem; font-weight: 600;">Pure LLM Baseline<br><span style="font-size: 0.75rem; color: var(--muted-color); font-weight: normal; font-family: monospace;">gemma-2-27b</span></th>
        <th style="padding: 1rem; font-weight: 600;">Queen-Only Baseline<br><span style="font-size: 0.75rem; color: var(--muted-color); font-weight: normal; font-family: monospace;">Queen Heuristics (Symbolic)</span></th>
        <th style="padding: 1rem; font-weight: 600; color: var(--accent-color);">R_Daneel_AI (Ours)<br><span style="font-size: 0.75rem; color: var(--accent-color); font-weight: normal; font-family: monospace;">gemma-2-27b + Queen</span></th>
        <th style="padding: 1rem; font-weight: 600; color: #10b981;">R_Daneel_AI (Ours)<br><span style="font-size: 0.75rem; color: #10b981; font-weight: normal; font-family: monospace;">deepseek-v4 + Queen</span></th>
      </tr>
    </thead>
    <tbody>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>Puzzles Solved</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">80 / 500</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">376 / 500</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">404 / 500</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">390 / 500</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>Accuracy / Pass Rate</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">16.0%</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">75.2%</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">80.8%</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">78.0%</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>Puzzle Rating (Bisection)</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">673 Rating</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">1399 Rating</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">1478 Rating</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">1437 Rating</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>Inference Retries / Loops</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">0.24 avg</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">0.00 avg</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">0.38 avg</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">0.16 avg</td>
      </tr>
      <tr style="border-bottom: 2px solid var(--border-color); background: rgba(255, 255, 255, 0.02);">
        <td style="padding: 0.85rem 1rem;"><strong>Average Move Latency</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">17.5s</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">0.54s</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">24.3s</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">133.9s</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>600s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">10.0% (5/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">98.0% (49/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">96.0% (48/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">90.0% (45/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>700s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">16.0% (8/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">96.0% (48/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">96.0% (48/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">92.0% (46/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>800s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">10.0% (5/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">94.0% (47/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">98.0% (49/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">84.0% (42/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>900s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">20.0% (10/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">82.0% (41/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">84.0% (42/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">92.0% (46/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1000s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">16.0% (8/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">86.0% (43/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">88.0% (44/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">86.0% (43/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1100s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">16.0% (8/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">70.0% (35/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">82.0% (41/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">72.0% (36/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1200s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">20.0% (10/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">66.0% (33/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">80.0% (40/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">74.0% (37/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1300s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">24.0% (12/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">52.0% (26/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">68.0% (34/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">64.0% (32/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1400s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">10.0% (5/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">58.0% (29/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">52.0% (26/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">56.0% (28/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1500s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">18.0% (9/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">50.0% (25/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">64.0% (32/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">70.0% (35/50)</td>
      </tr>
    </tbody>
  </table>
</div>

#### Benchmark Table 2: High Difficulty Tiers (1600–2500 Elo)

<div style="overflow-x: auto; margin: 1.5rem 0; border: 1px solid var(--border-color); border-radius: 8px;">
  <table style="width: 100%; border-collapse: collapse; text-align: left; font-size: 0.9rem;">
    <thead>
      <tr style="background: var(--code-bg); border-bottom: 2px solid var(--border-color); color: var(--text-color);">
        <th style="padding: 1rem; font-weight: 600;">Evaluation Metric / Elo Tier</th>
        <th style="padding: 1rem; font-weight: 600;">Pure LLM Baseline<br><span style="font-size: 0.75rem; color: var(--muted-color); font-weight: normal; font-family: monospace;">gemma-2-27b</span></th>
        <th style="padding: 1rem; font-weight: 600;">Queen-Only Baseline<br><span style="font-size: 0.75rem; color: var(--muted-color); font-weight: normal; font-family: monospace;">Queen Heuristics (Symbolic)</span></th>
        <th style="padding: 1rem; font-weight: 600; color: var(--accent-color);">R_Daneel_AI (Ours)<br><span style="font-size: 0.75rem; color: var(--accent-color); font-weight: normal; font-family: monospace;">gemma-2-27b + Queen</span></th>
        <th style="padding: 1rem; font-weight: 600; color: #10b981;">R_Daneel_AI (Ours)<br><span style="font-size: 0.75rem; color: #10b981; font-weight: normal; font-family: monospace;">deepseek-v4 + Queen</span></th>
      </tr>
    </thead>
    <tbody>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>Puzzles Solved</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">101 / 500</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">170 / 500</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">177 / 500</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">191 / 500</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>Accuracy / Pass Rate</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">20.2%</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">34.0%</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">35.4%</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">38.2%</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>Puzzle Rating (Bisection)</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">1741 Rating</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">1919 Rating</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">1936 Rating</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">1968 Rating</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>Inference Retries / Loops</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">0.22 avg</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">0.00 avg</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">1.40 avg</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">0.45 avg</td>
      </tr>
      <tr style="border-bottom: 2px solid var(--border-color); background: rgba(255, 255, 255, 0.02);">
        <td style="padding: 0.85rem 1rem;"><strong>Average Move Latency</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">38.8s</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">0.46s</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">29.6s</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">276.9s</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1600s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">20.0% (10/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">46.0% (23/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">42.0% (21/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">56.0% (28/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1700s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">26.0% (13/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">50.0% (25/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">50.0% (25/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">48.0% (24/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1800s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">24.0% (12/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">44.0% (22/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">48.0% (24/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">50.0% (25/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>1900s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">12.0% (6/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">38.0% (19/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">44.0% (22/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">46.0% (23/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>2000s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">26.0% (13/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">40.0% (20/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">44.0% (22/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">40.0% (20/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>2100s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">26.0% (13/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">22.0% (11/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">30.0% (15/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">26.0% (13/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>2200s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">18.0% (9/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">36.0% (18/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">28.0% (14/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">40.0% (20/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>2300s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">8.0% (4/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">26.0% (13/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">22.0% (11/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">22.0% (11/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>2400s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">24.0% (12/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">20.0% (10/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">30.0% (15/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">40.0% (20/50)</td>
      </tr>
      <tr style="border-bottom: 1px solid var(--border-color);">
        <td style="padding: 0.85rem 1rem;"><strong>2500s Tier Accuracy</strong></td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">18.0% (9/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace;">18.0% (9/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: var(--accent-color);">16.0% (8/50)</td>
        <td style="padding: 0.85rem 1rem; font-family: monospace; font-weight: bold; color: #10b981;">14.0% (7/50)</td>
      </tr>
    </tbody>
  </table>
  </div>
</details>

---

## Part 6: Future Plans: Automated Prompt Optimization & Meta-Harness

To scale this design, we are decoupling active gameplay execution from prompt optimization cycles:

![Inner Loop vs. Outer Loop Feedback loops](/images/feedback-loops.svg)

1.  **Inner Loop (Real-Time)**: Executes the live game transaction. Runs tool calling and self-correction to produce a valid move.
2.  **Outer Loop (Planned)**: Analyzes multi-turn failure logs. When rejections for specific constraint categories (like pins) cross a threshold, a meta-reasoning model synthesizes prompt amendments and tests them against the puzzle suite to measure Elo delta.

---

## Generalising Beyond Chess
While chess provides a clean testbed, the **LLM Proposes $\rightarrow$ Symbolic Validates & Explains $\rightarrow$ LLM Revises** pattern is completely domain-general. 

Any structured domain governed by strict rules can be modeled similarly:

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(340px, 1fr)); gap: 1rem; margin: 1.5rem 0;">
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">🗄️ SQL Generation</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Prolog validates database schemas and foreign key constraints, preventing column name hallucinations.</span>
  </div>
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">📅 Scheduling</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Enforces business hours, resource availability, and timezone offsets, preventing impossible bookings.</span>
  </div>
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">📐 CAD & Physical Layout</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Validates design rule checks (DRC), pitch spacing, and keep-out zones, preventing cell overlaps.</span>
  </div>
  <div style="border: 1px solid var(--border-color); padding: 1.25rem; border-radius: 8px; background: var(--code-bg);">
    <strong style="font-size: 1.05rem; color: var(--text-color); display: block; margin-bottom: 0.25rem;">📜 Compliance</strong>
    <span style="font-size: 0.9rem; color: var(--muted-color); line-height: 1.4; display: block;">Validates financial transactions or legal drafts against declarative compliance rulebooks.</span>
  </div>
</div>

By marrying neural System 1 intuition with symbolic System 2 logic, we can construct autonomous agents that balance creative strategy with guaranteed constraint enforcement.

### Related Work & Citations

The integration of neural models with symbolic reasoning engines is an active area of machine learning research. Key paradigms in this space include:

*   **LLM-Modulo Frameworks**: Subbarao Kambhampati's research (Kambhampati et al., 2024) formalizes the "LLM-Modulo" architecture, demonstrating that LLMs cannot act as their own verifiers for planning tasks and must be paired with external, deterministic verification modules.
*   **AlphaGeometry**: Developed by Google DeepMind (Trinh et al., *Nature* 2024), AlphaGeometry integrates a neural language model (System 1) with a symbolic deduction engine (System 2) to solve complex Olympiad-level geometry problems, proving the efficacy of hybrid systems in rigorous logical domains.
*   **Logic-LM & PAL**: Program-Aided Language Models (PAL) by Gao et al. (ACL 2023) and Logic-LM by Pan et al. (EMNLP 2023) delegate algebraic and logical reasoning tasks from the LLM to programmatic interpreters (Python, Prolog) to guarantee state and mathematical accuracy.

---

## Appendix A: XML Tool-Call Parsing Fallback

When invoking reasoning models like `deepseek-v4`, tool calls may be generated inside raw XML blocks (e.g. `<｜｜DSML｜｜tool_calls>`) rather than structured SDK objects. The following Python function is used inside the `R_Daneel_AI` client loop to parse these blocks and unify the model interfaces:

```python
import re

class MockFunction:
    def __init__(self, name, arguments):
        self.name = name
        self.arguments = arguments

class MockToolCall:
    def __init__(self, id, function):
        self.id = id
        self.function = function

def parse_deepseek_xml_tool_calls(assistant_message):
    content = assistant_message.content or ""
    parsed_xml_tool_calls = []
    
    if "<｜｜DSML｜｜tool_calls>" in content:
        invoke_blocks = re.findall(
            r'<｜｜DSML｜｜invoke name="([^"]+)"\s*>(.*?)</｜｜DSML｜｜invoke\s*>', 
            content, re.DOTALL
        )
        for idx, (func_name, block_content) in enumerate(invoke_blocks):
            param_matches = re.findall(
                r'<｜｜DSML｜｜parameter name="([^"]+)"[^>]*>(.*?)</｜｜DSML｜｜parameter\s*>', 
                block_content, re.DOTALL
            )
            func_args = {p_name: p_val.strip() for p_name, p_val in param_matches}
            
            # Instantiate mock structure for compatibility with SDK handlers
            mock_call = MockToolCall(f"call_ds_{idx}", MockFunction(func_name, func_args))
            parsed_xml_tool_calls.append(mock_call)
            
    return parsed_xml_tool_calls
```

---

## Explore the Projects
*   **[R_Daneel_AI](https://abhay-ai.github.io/R_Daneel_AI)**: A neuro-symbolic AI chess player combining LLM heuristic reasoning with symbolic constraints.
*   **[Queen](https://abhay-ai.github.io/Queen/)**: The SWI-Prolog spatial reasoning engine and ruleset validating agent action parameters.
