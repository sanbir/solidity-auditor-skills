# Report Formatting

## Report Path

Save the report to `assets/findings/{project-name}-audit-report-{timestamp}.md`
where `{project-name}` is the repo root basename and `{timestamp}` is
`YYYYMMDD-HHMMSS` at scan time.

## Output Format

````
# Security Review — <ContractName or repo name>

---

## Scope

|                                  |                                                        |
| -------------------------------- | ------------------------------------------------------ |
| **Mode**                         | ALL / default / filename                               |
| **Files reviewed**               | `File1.sol` · `File2.sol`<br>`File3.sol` · `File4.sol` |
| **Attack vectors checked**       | 210                                                    |
| **Agents deployed**              | 5 / 7 (deep)                                           |
| **Confidence threshold (1-100)** | N                                                      |
| **Constraints**                  | _if `.pashov-skills-constraints.yaml` found, list declared values; otherwise omit this row_ |

---

## Findings

[95] **1. <Title>**

`ContractName.functionName` · Confidence: 95

**Description**
<The vulnerable code pattern and why it is exploitable, in 1 short sentence>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```
---

[82] **2. <Title>**

`ContractName.functionName` · Confidence: 82

**Description**
<The vulnerable code pattern and why it is exploitable, in 1 short sentence>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```
---

< ... all findings >

---

Findings List

| # | Confidence | Title |
|---|---|---|
| 1 | [95] | <title> |
| 2 | [82] | <title> |
| | | **Below Confidence Threshold** |
| 3 | [75] | <title> |
| 4 | [60] | <title> |

---

> ⚠️ This review was performed by an AI assistant. AI analysis can never
verify the complete absence of vulnerabilities and no guarantee of security
is given. Team security reviews, bug bounty programs, and on-chain monitoring
are strongly recommended.

````

**Rules:** Follow the template above exactly. Sort findings by confidence
(highest first). Findings below the threshold get a description but no
**Fix** block. Draft findings directly in report format — do not re-generate.
