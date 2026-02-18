# resume-mcp Spec: Multi-Format Resume & CV Generator

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft — Ship on Demand
**Created:** 2026-02-18
**Phase:** 3 (lower priority; trigger-based shipping)
**Grounded In:** Module strategy scout (2026-02-17)

---

## 1. Vision

Transform structured resume markdown into multiple formats optimized for different audiences: ATS-plain-text (for applicant tracking systems), professional LaTeX/PDF (for human review), and HTML (for personal website)—enabling job seekers to maintain a single source-of-truth resume while generating format-specific variants.

### The Core Insight

Resumes are universal artifacts, but most people maintain multiple versions: a plain-text ATS-safe version (to bypass automated screening), a beautifully designed PDF (for human hiring managers), and an HTML version (for personal website). This creates friction: edit one version, then manually replicate across others. A ZenSci module captures resume structure (contact, summary, experience, education, skills, projects) and renders it intelligently for each context: ATS formatting strips fancy symbols and maintains simple structure; PDF applies professional typography; HTML enables interactivity and links.

### What Makes This Different

Unlike CV-focused tools (which emphasize chronological academic history), resume-mcp is *job-seeker-first*, supporting both résumé (career progression) and CV (academic record) formats. Unlike generic resume builders (Canva, Indeed), ZenSci's approach is *markdown-native* and supports multi-format output without vendor lock-in. The module treats ATS compatibility seriously: validates keyword density, avoids symbols that break parsing, and generates stripped-down plain-text variant.

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Single-source resume maintenance** — write once in markdown; generate ATS-plain, PDF, HTML, JSON without manual re-editing
2. **ATS optimization** — ensure plain-text variant bypasses automated screening while remaining human-readable
3. **Format flexibility** — support US/international formats, academic CVs, industry resumes, portfolio-focused presentations

### Success Criteria

✅ Generated ATS-plain-text passes 100% of major ATS parsers (LinkedIn, Workday, Taleo, BambooHR test suite)
✅ PDF output is visually comparable to professionally designed resumes (design review 8+/10)
✅ Single markdown source generates valid PDF, HTML, and ATS-plain with 0 manual corrections required
✅ HTML variant includes working links, skill tags, and is SEO-optimized for personal portfolio site

### Non-Goals

❌ Job application automation or job board integration (separate service)
❌ AI-driven resume optimization or keyword suggestions (out of scope)
❌ Biographical narrative generation (humans write their own stories)
❌ Salary negotiation or benefits comparison (not resume's domain)

---

## 3. Strategic Fit Assessment

**Why Phase 3 (not Phase 1 or 2):**
Resumes are *extremely commoditized*—thousands of tools exist (Zety, Indeed Resume, LinkedIn, Canva, Overleaf CV templates). ZenSci's differentiation is markdown + multi-format output, which matters to *technically sophisticated* job seekers (engineers, designers, academics) but has limited mass-market appeal. Without strong demand signals from that subset, this module is speculative. Additionally, ATS compatibility is a moving target (ATS systems change parsing rules frequently), making long-term maintenance uncertain.

**Market Trigger Conditions:**

- **Trigger 1:** 15+ GitHub issues or feature requests explicitly requesting resume + ATS-plain-text + PDF generation
- **Trigger 2:** Job seeker interviews (3+ candidates) confirm that ATS-plain-text generation eliminates their manual resume maintenance overhead
- **Trigger 3:** Data shows that ZenSci-generated ATS-plain-text resumes pass LinkedIn, Workday, and Taleo automated screening at >90% rate (real-world validation)

**Risk of NOT shipping:**
**Low.** Job seekers have many alternatives. If we don't ship, no strategic impact to ZenSci. If we do ship without demand validation, the module becomes maintenance burden (ATS parsing rules change ~2x/year, requiring updates).

---

## 4. Technical Architecture (simplified)

### 4.1 MCP Tools

**`convertResume`**
- **Input:** `DocumentRequest` (source markdown, output format: 'resume' | 'cv')
- **Output:** `DocumentResponse` (multi-format: PDF + HTML + ATS-plain-text + JSON)
- **Processing:** Parse resume structure → validate completeness → apply formatting rules per output type → generate all variants

**`validateATSCompliance`**
- **Input:** resume markdown or ATS-plain-text
- **Output:** `ValidationResult` with compliance warnings (symbols that break ATS, keyword density, formatting issues)

**`generateATSPlainText`**
- **Input:** resume markdown
- **Output:** plain-text version stripped of symbols, colors, formatting; optimized for ATS parsing

**`extractKeywords`**
- **Input:** resume markdown
- **Output:** array of skills, job titles, and experience tags (for ATS matching + SEO)

### 4.2 Infrastructure Reuse

- **MarkdownParser** — parse resume section hierarchy and frontmatter
- **SchemaValidator** — validate `DocumentRequest` against resume schema
- **Pandoc Python engine** — LaTeX → PDF compilation for professional resume output

### 4.3 Novel Infrastructure Required

**Resume Schema (`ResumeRequest`)**
```typescript
interface ResumeRequest extends DocumentRequest {
  format: 'resume' | 'cv';
  moduleOptions: {
    variant: 'technical' | 'corporate' | 'academic' | 'creative';
    includeContact: boolean;
    includeSummary: boolean;
    includeProjects: boolean;
    hideReference: boolean; // for review-only copies
    targetRole?: string; // optional: for keyword optimization
    locale?: 'US' | 'UK' | 'EU' | 'INT'; // formatting differences
  };
}
```

**Resume AST Structure** — specialized nodes for resume sections:
```typescript
type ResumeNode =
  | { type: 'contact'; name: string; email: string; phone?: string; location?: string; links?: { label: string; url: string }[] }
  | { type: 'summary'; text: string }
  | { type: 'experience'; entries: ExperienceEntry[] }
  | { type: 'education'; entries: EducationEntry[] }
  | { type: 'skills'; categories: { name: string; items: string[] }[] }
  | { type: 'projects'; entries: ProjectEntry[] }
  | { type: 'certifications'; entries: CertificationEntry[] }
  | { type: 'publications'; entries: PublicationEntry[] };

interface ExperienceEntry {
  title: string;
  company: string;
  location?: string;
  startDate: string; // YYYY-MM
  endDate?: string;
  current?: boolean;
  description: string;
  highlights?: string[];
}

interface EducationEntry {
  degree: string;
  institution: string;
  location?: string;
  graduationDate: string; // YYYY-MM
  gpa?: number;
  honors?: string;
}

interface ProjectEntry {
  title: string;
  description: string;
  url?: string;
  technologies?: string[];
  date?: string;
}

interface CertificationEntry {
  name: string;
  issuer: string;
  date: string; // YYYY-MM
  credentialId?: string;
  credentialUrl?: string;
}

interface PublicationEntry {
  title: string;
  authors: string[];
  publication: string;
  date: string; // YYYY-MM
  url?: string;
  doi?: string;
}
```

**ATS Compliance Validator** — checks for common ATS-breaking patterns (symbols, complex formatting, unusual fonts)

**Resume JSON Schema** — for structured data export (compatible with JSON Resume standard or custom schema)

### 4.4 Key Schemas

```typescript
interface ResumeStructure {
  contact: ContactInfo;
  summary?: string;
  experience: ExperienceEntry[];
  education: EducationEntry[];
  skills: SkillCategory[];
  projects?: ProjectEntry[];
  certifications?: CertificationEntry[];
  publications?: PublicationEntry[];
}

interface ContactInfo {
  name: string;
  email: string;
  phone?: string;
  location?: string;
  links?: { label: string; url: string }[];
}

interface SkillCategory {
  name: string;
  items: string[];
}

interface ResumeOutputFormats {
  pdf: Buffer;
  html: string;
  atsPlainText: string;
  json: Record<string, unknown>;
}

interface ATSComplianceReport {
  compliant: boolean;
  score: number; // 0-100
  warnings: {
    category: 'symbol' | 'formatting' | 'keyword' | 'structure';
    message: string;
    severity: 'low' | 'medium' | 'high';
  }[];
  keywordDensity?: { keyword: string; count: number; density: number }[];
}
```

---

## 5. Implementation Estimate

| Track | Duration (weeks) | Dependencies |
|-------|---------|-------------|
| Resume schema + parser | 1 | MarkdownParser |
| ATS plain-text generator + compliance validator | 1.5 | Parser; ATS spec research |
| LaTeX/PDF resume templates (3 variants) | 2 | Pandoc; design review; template markup |
| HTML resume generator + styling | 1.5 | HTML template library; CSS framework |
| JSON Resume export adapter | 0.5 | Schema definition |
| Keyword extraction + density analysis | 1 | NLP library (optional) or regex-based |
| MCP tool registration & testing | 1 | packages/sdk; ATS test suite (Workday, LinkedIn, Taleo) |
| **Total** | **9** | New: ATS compatibility research; design templates; ATS test systems |

---

## 6. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| ATS systems reject generated plain-text due to parsing rule changes | Medium | High | Build ATS test suite early (validate against Workday, LinkedIn, Taleo, BambooHR); test quarterly for rule changes |
| Generated PDF resume design quality insufficient for professional use | Medium | High | Hire designer for 2–3 professional templates; conduct design review with 5+ hiring managers before shipping |
| Resume domain too commoditized; low demand relative to maintenance burden | Medium | Medium | This is acceptable risk for Phase 3. Monitor GitHub issues; if <5 requests/quarter, consider deprecation. |
| JSON Resume compatibility not valuable (users don't export to third-party tools) | Low | Low | Make JSON export optional feature; focus on PDF/HTML as primary outputs |
| ATS keyword optimization is ineffective (algorithm can't improve keyword density) | Medium | Medium | Shift focus from keyword optimization to compliance validation; let humans optimize keywords |

---

## 7. Open Questions & Future Considerations

1. **Variant formats:** Should resume-mcp support ATS-optimized variants for specific job boards (LinkedIn, Indeed, specific ATS systems like Workday)?

2. **Cover letter integration:** Should we extend resume-mcp to also generate cover letters with consistent branding, or keep that separate?

3. **Anonymized version:** Should resume-mcp auto-generate "blind resume" variants (no name, age indicators, gender) for bias-free hiring?

4. **Multi-language support:** Should resumes support bilingual output (e.g., English + Spanish) for international job seekers?

5. **Activity tracking:** Should the module track which resume variant was used for which job application, linking back to application outcomes?

6. **Skills endorsement metrics:** Could we integrate with LinkedIn Skills endorsements to auto-populate skill importance/credibility?

7. **Salary transparency:** Should resume-mcp include optional salary history or desired salary range sections?

8. **Video resume variant:** Should we support generating a "visual resume" with embedded video or interactive timeline?

9. **Accessibility & screen reader optimization:** Should generated HTML resumes be rigorously tested for screen reader compatibility?

10. **Reference sheet generation:** Should resume-mcp auto-generate a separate "references" sheet (contact info for recommenders) with consistent formatting?

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **HIGH-1:** Tool names use camelCase (`convertResume`, `validateATSCompliance`, `generateATSPlainText`, `extractKeywords`). Should be snake_case: `convert_resume`, `validate_ats_compliance`, `generate_ats_plain_text`, `extract_keywords`.
- **ADVISORY-2:** Uses `moduleOptions` field.
- ATS compliance validation is a strong differentiator for this module.
- Strategic Fit Assessment correctly identifies commoditized market with niche technical appeal.
