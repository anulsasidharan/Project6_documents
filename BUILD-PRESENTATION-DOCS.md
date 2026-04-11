# Build presentation Word export (optional)

Install [Pandoc](https://pandoc.org/installing.html), then from the **repository root**:

**Single Word file (study guide + speaker notes, in order):**

```bash
pandoc docs/VidShield-AI-Presentation-Study-Guide.md docs/VidShield-AI-Presentation-Speaker-Notes.md -o docs/VidShield-AI-Presentation-Bundle.docx --toc -s
```

**Study guide only:**

```bash
pandoc docs/VidShield-AI-Presentation-Study-Guide.md -o docs/VidShield-AI-Presentation-Study-Guide.docx -s
```

**Speaker notes only:**

```bash
pandoc docs/VidShield-AI-Presentation-Speaker-Notes.md -o docs/VidShield-AI-Presentation-Speaker-Notes.docx -s
```

**Q&A / interview prep (with embedded architecture images):**

```bash
pandoc docs/VidShield-AI-QA-Interview-Prep.md -o docs/VidShield-AI-QA-Interview-Prep.docx -s --toc --resource-path=docs
```

If Pandoc is not installed, use **Print to PDF** from a Markdown preview, or open the `.md` files in Word (recent Word versions can import Markdown).
