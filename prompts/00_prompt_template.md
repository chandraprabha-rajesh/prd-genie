# Prompt Template node

Sits between the extraction and Agent 2. Merges the approved extraction with
the PRD template file; the merged text is what Agent 2 receives.

```
APPROVED REQUIREMENTS:
{extraction}

---

PRD TEMPLATE TO FOLLOW:
{prd_template}
```
