---
name: test-gpt-form
description: >
  TEST workflow — the smallest possible check of whether a workflow can render
  inline interactive `ask_user_input` forms in ChatGPT Work. Asks three trivial
  questions, each one only as a form, then echoes the answers. Writes nothing,
  reads nothing, calls no tools. Trigger on "test gpt form", "test form",
  "перевір форми", "тест форми".
allowed-tools:
---

# Test — GPT interactive form

Goal: prove that a workflow can render an inline user-input form. Nothing else
happens here. **Call no tools. Write nothing. Fetch nothing.**

## How to ask

Emit the directive **directly in your assistant message**:

```
genui{"ask_user_input":{"questions":[ … ]}}
```

- Do not check or discuss chat modes. Do not call `functions.request_user_input`.
  Do not use `show_widget`, `sendPrompt`, `visualize`, HTML, or `AskUserQuestion`.
- Never ask in prose, a numbered list, or "reply with 1/2/3" — that is the
  failure this test exists to catch.
- One short sentence before the form, then the form, then **end the turn**.
  No text after the directive.
- The next user message is the answer (including the `> Question` / answer
  format). Parse it and go to the next step.
- Empty answer (`Не вибрано`, `Not selected`, blank) → one sentence saying what
  is needed, then re-emit the **same** form.
- Match the language of the user's last message.

## Steps

**1.**

```
genui{"ask_user_input":{"questions":[
  {"question":"Pick a colour","options":["Red","Green","Blue"],"type":"single_select","free_text_placeholder":"Another colour"}
]}}
```

**2.**

```
genui{"ask_user_input":{"questions":[
  {"question":"Pick any fruits","options":["Apple","Banana","Cherry"],"type":"multi_select","free_text_placeholder":"Another fruit"}
]}}
```

**3.**

```
genui{"ask_user_input":{"questions":[
  {"question":"Done — did all three render as forms?","options":["Yes, all three","No, at least one was plain text"],"type":"single_select","free_text_placeholder":"What went wrong"}
]}}
```

## Result

Print one line:

```
✅ test-gpt-form — 3/3 forms rendered · colour: {answer1} · fruits: {answer2}
```

If the user answered `No` in step 3, or reported seeing raw `genui{…}` text,
print `❌ FAIL` instead and name which step did not render.
