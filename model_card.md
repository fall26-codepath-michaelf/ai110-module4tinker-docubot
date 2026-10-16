# DocuBot Model Card

This model card is a short reflection on your DocuBot system. Fill it out after you have implemented retrieval and experimented with all three modes:

1. Naive LLM over full docs  
2. Retrieval only  
3. RAG (retrieval plus LLM)

Use clear, honest descriptions. It is fine if your system is imperfect.

---

## 1. System Overview

**What is DocuBot trying to do?**  
Describe the overall goal in 2 to 3 sentences.

DocuBot helps developers find information in a small documentation corpus. It can show retrieved documentation sections directly or use Gemini to write an answer grounded in retrieved sections.

**What inputs does DocuBot take?**  
For example: user question, docs in folder, environment variables.

- A developer question.
- Markdown or text files in the `docs/` folder.
- An optional `GEMINI_API_KEY` for Naive LLM and RAG modes.

**What outputs does DocuBot produce?**

Depending on the mode, DocuBot produces a Gemini response, raw documentation sections with filenames, or a grounded Gemini response. When no snippet has enough evidence, Retrieval Only and RAG return `I do not know based on these docs.`

---

## 2. Retrieval Design

**How does your retrieval system work?**  
Describe your choices for indexing and scoring.

- How do you turn documents into an index?
- How do you score relevance for a query?
- How do you choose top snippets?

Documents are split into `##` Markdown sections so retrieval returns smaller sections instead of complete files. The index maps normalized, meaningful words to filenames. The scorer counts distinct meaningful query words shared by a query and section, then returns up to three sections in descending score order. `MIN_RELEVANCE_SCORE = 2` filters out weaker matches.

**What tradeoffs did you make?**  
For example: speed vs precision, simplicity vs accuracy.

This approach is fast, transparent, and easy to explain line by line. It is less precise than semantic search: it does not understand synonyms, intent, or the full meaning of a sentence. The threshold reduces weak matches, but a score of 2 is still a judgment call and can refuse a short but answerable query.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**  
Briefly describe how each mode behaves.

- Naive LLM mode:
- Retrieval only mode:
- RAG mode:

- Naive LLM mode calls Gemini with only the question. The current wrapper does not include the documentation corpus, so its answer is not grounded in project evidence.
- Retrieval only mode does not call Gemini. It displays matching sections and filenames for the reader to inspect.
- RAG mode retrieves sections first, then calls Gemini only when snippets were found. If there are no snippets, it refuses without calling Gemini.

**What instructions do you give the LLM to keep it grounded?**  
Summarize the rules from your prompt. For example: only use snippets, say "I do not know" when needed, cite files.

The RAG prompt tells Gemini to use only retrieved snippets, avoid inventing functions, endpoints, or configuration values, cite the files it used, and reply `I do not know based on the docs I have.` when the snippets do not contain enough evidence.

---

## 4. Experiments and Comparisons

Run the **same set of queries** in all three modes. Fill in the table with short notes.

You can reuse or adapt the queries from `dataset.py`.

| Query | Naive LLM: helpful or harmful? | Retrieval only: helpful or harmful? | RAG: helpful or harmful? | Notes |
|------|---------------------------------|--------------------------------------|---------------------------|-------|
| Which endpoint returns all users? | Mixed. In one run it suggested `GET /users`, which is wrong; in a later run it asked for more context. | Helpful but less readable: it returned the `API_REFERENCE.md` user-data section containing `GET /api/users`. | Helpful: it answered `GET /api/users` and cited `API_REFERENCE.md`. | RAG combined evidence with a direct answer. |
| Is there any mention of payment processing in these docs? | Not grounded because it has no documentation evidence. | Helpful: it returned `I do not know based on these docs.` | Helpful: the retrieval guardrail prevented a generated guess. | No section met the relevance threshold. |
| Can users edit projects? | Not tested in this mode. | Helpful but ambiguous: it showed roles including `editor`, without defining editor permissions. | Mixed: it inferred that the `editor` role implies actions can be taken, while admitting the docs do not define an editing endpoint or function. | RAG can still make a weak inference when retrieved text is incomplete. |

**What patterns did you notice?**  

- When does naive LLM look impressive but untrustworthy?  
- When is retrieval only clearly better?  
- When is RAG clearly better than both?

Naive generation can look fluent but changes between runs and has no evidence from this project. Retrieval Only is easier to audit because the source text is visible, but readers must interpret it themselves. RAG is clearest when the retrieved sections directly answer the question, such as the all-users endpoint question.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**  
For each one, say:

- What was the question?  
- What did the system do?  
- What should have happened instead?

For `Which endpoint returns all users?`, Naive LLM previously answered `GET /users`. The docs specify `GET /api/users`, so the answer sounded plausible but omitted the required `/api` base path. It should have used documentation evidence or refused to guess.

For `Can users edit projects?`, RAG retrieved a permissions table that lists an `editor` role. It cautiously noted that the docs do not define an editing endpoint, but it still inferred that the role implies actions can be taken. It should refuse a direct yes-or-no answer until the documentation explicitly defines editor permissions.

**When should DocuBot say “I do not know based on the docs I have”?**  
Give at least two specific situations.

- When no retrieved section reaches the minimum relevance score, such as the payment-processing question.
- When snippets mention related concepts but do not directly support the requested conclusion, such as whether an editor may edit a project.

**What guardrails did you implement?**  
Examples: refusal rules, thresholds, limits on snippets, safe defaults.

- Retrieve `##` sections instead of entire documents.
- Require a relevance score of at least 2.
- Return no snippets below that threshold.
- Make RAG refuse rather than call Gemini when retrieval has no evidence.
- Instruct Gemini to use only provided snippets and name its source files.

---

## 6. Limitations and Future Improvements

**Current limitations**  
List at least three limitations of your DocuBot system.

1. Keyword matching does not understand synonyms or semantic meaning.
2. The fixed threshold of 2 may reject valid short questions or include weak matches for broad questions.
3. Top results can include multiple sections from the same file and do not use a tie-breaker based on section specificity.
4. Even with grounded snippets, Gemini can make cautious but unsupported inferences from incomplete documentation.

**Future improvements**  
List two or three changes that would most improve reliability or usefulness.

1. Add semantic embeddings or a better ranking method that accounts for phrase meaning and rarity.
2. Index individual sections directly and diversify results across documents when useful.
3. Add evaluation cases that count a correct refusal as a successful outcome.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**  
Think about wrong answers, missing information, or over trusting the LLM.

Incorrect documentation answers can cause developers to use the wrong API endpoint, misconfigure authentication, expose data, or make unsafe database changes. Fluent answers are especially risky when people assume they are grounded without checking the evidence.

**What instructions would you give real developers who want to use DocuBot safely?**  
Write 2 to 4 short bullet points.

- Treat answers as guidance and verify important claims in the cited source files.
- Keep documentation current, specific, and complete before relying on RAG.
- Preserve refusal behavior when evidence is missing or ambiguous.
- Do not use DocuBot alone for security-sensitive or production-changing decisions.

---
