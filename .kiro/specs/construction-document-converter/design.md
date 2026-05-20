# Design Document: Construction Document Converter

## Architecture Overview

The Construction Document Converter adapts the existing book-to-skill extraction pipeline to process construction documents. The architecture reuses the proven extraction layer while adding construction-specific document type detection, parsing strategies, and output generation.

### Core Design Principle
**Reuse extraction, specialize parsing.** The PDF/EPUB extraction methods (pdftotext, PyPDF2, pdfminer, Docling, ebooklib) remain unchanged. New construction-specific modules handle document type detection, content parsing, and skill generation.

### High-Level Flow

```
Construction Document (PDF/EPUB)
    ↓
[Existing] Extract text + metadata
    ↓
[NEW] Detect document type (Specification, Safety Manual, Material Datasheet, etc.)
    ↓
[NEW] Parse by type (hierarchical sections, procedures, tables, compliance rules)
    ↓
[NEW] Organize into construction categories (procedures, specs, compliance, materials)
    ↓
[NEW] Generate construction-optimized skill files
    ↓
~/.claude/skills/<skill_name>/ (ready for on-site reference)
```

---

## Document Type Detection

### Detection Strategy

**Input**: Extracted text (first 10,000 characters) + optional user confirmation

**Analysis approach**:
1. Scan for structural patterns (section numbering, procedure keywords, table layouts, code references, checklist markers)
2. Analyze terminology (construction-specific terms, abbreviations, standards)
3. Examine formatting (hierarchical indentation, step numbering, hazard warnings)
4. Assign confidence score to each document type

**Document Types**:
- **Specification**: Hierarchical sections (3.2.1 format), technical requirements, tolerances, material grades
- **Safety Manual**: Procedures, hazard warnings (DANGER, WARNING, CAUTION), step-by-step instructions
- **Material Datasheet**: Tables, material properties, performance data, grades (ASTM, ACI, AWS, AISC)
- **Building Code**: Code section references (§), regulatory requirements, applicability, penalties
- **Compliance Checklist**: Checklist items (☐), verification criteria, sign-off requirements
- **Installation Guide**: Step-by-step tasks (1., 2., 3.), prerequisites, tools, troubleshooting
- **Blueprint**: Visual descriptions, reference information, cross-references (note: visual elements noted but not extracted as images)
- **Other**: Unknown or mixed-type documents

**User interaction**:
- If confidence > 80%, auto-select type and proceed
- If confidence < 80%, ask user: "What type of construction document is this?" with options
- Allow user to override auto-detection

### Pseudocode

```python
def detect_construction_document_type(text: str) -> Tuple[str, float, str]:
    """
    Classify document type based on structure and terminology.
    
    Returns:
        (document_type, confidence, extraction_mode)
        - document_type: One of [specification, safety_manual, material_datasheet, 
                                 building_code, compliance_checklist, installation_guide, 
                                 blueprint, other]
        - confidence: 0.0 to 1.0
        - extraction_mode: 'technical' or 'text'
    """
    scores = {}
    
    # Check for specification patterns
    if re.search(r'Section \d+\.\d+\.\d+', text):
        scores['specification'] += 0.3
    if re.search(r'(tolerance|dimension|grade|strength|performance)', text, re.I):
        scores['specification'] += 0.2
    
    # Check for safety manual patterns
    if re.search(r'(DANGER|WARNING|CAUTION|HAZARD)', text):
        scores['safety_manual'] += 0.4
    if re.search(r'(Step \d+|Procedure|Mitigation)', text, re.I):
        scores['safety_manual'] += 0.2
    
    # Check for material datasheet patterns
    if re.search(r'(Property|Value|Unit|Tolerance)', text):
        scores['material_datasheet'] += 0.3
    if re.search(r'(ASTM|ACI|AWS|AISC)', text):
        scores['material_datasheet'] += 0.2
    
    # Check for building code patterns
    if re.search(r'(§|Code Section|Requirement|Applicability)', text):
        scores['building_code'] += 0.3
    if re.search(r'(compliance|regulatory|jurisdiction)', text, re.I):
        scores['building_code'] += 0.2
    
    # Check for compliance checklist patterns
    if re.search(r'(☐|□|\[ \]|Checklist)', text):
        scores['compliance_checklist'] += 0.4
    if re.search(r'(Verify|Sign-off|Responsible)', text, re.I):
        scores['compliance_checklist'] += 0.2
    
    # Check for installation guide patterns
    if re.search(r'(\d+\.\s+[A-Z]|Prerequisites|Tools Required)', text):
        scores['installation_guide'] += 0.3
    if re.search(r'(Troubleshooting|Verification)', text, re.I):
        scores['installation_guide'] += 0.2
    
    # Determine highest score
    best_type = max(scores, key=scores.get)
    confidence = scores[best_type]
    
    # Determine extraction mode
    extraction_mode = 'technical' if has_tables_or_structured_data(text) else 'text'
    
    return (best_type, confidence, extraction_mode)
```

---

## Extraction Strategies by Document Type

### 1. Specification Documents

**Parsing approach**: Hierarchical section extraction

**Key extraction**:
- Sections and subsections (e.g., "Section 3.2.1 — Concrete Specifications")
- Technical requirements and criteria
- Quantified specifications (dimensions, tolerances, grades, performance thresholds)
- Material specifications and standards
- Cross-references to related sections

**Output**: `specifications.md` + `sections/section-NN-*.md`

**Pseudocode**:
```python
def extract_hierarchical_sections(text: str) -> List[Section]:
    """Extract nested sections from specification documents."""
    sections = []
    current_section = None
    
    for line in text.split('\n'):
        # Pattern: "Section N.M.P — Title"
        match = re.match(r'Section (\d+\.\d+(?:\.\d+)?)\s*—?\s*(.+)', line)
        if match:
            section_num, title = match.groups()
            current_section = Section(
                number=section_num,
                title=title,
                content=[],
                subsections=[]
            )
            sections.append(current_section)
        elif current_section:
            # Extract quantified data from content
            specs = extract_quantified_data(line)
            current_section.content.append(line)
            current_section.specifications.extend(specs)
    
    return sections
```

### 2. Safety Manual Documents

**Parsing approach**: Procedural extraction with hazard identification

**Key extraction**:
- Procedure names and descriptions
- Hazard warnings (DANGER, WARNING, CAUTION)
- Step-by-step instructions
- Mitigation measures
- Compliance references
- Verification criteria

**Output**: `procedures.md`

**Pseudocode**:
```python
def extract_procedures(text: str) -> List[Procedure]:
    """Extract step-by-step procedures from safety/installation guides."""
    procedures = []
    current_procedure = None
    
    for line in text.split('\n'):
        # Detect procedure start
        if re.match(r'^(Procedure|Task|Process):\s*(.+)', line, re.I):
            if current_procedure:
                procedures.append(current_procedure)
            current_procedure = Procedure(
                name=line.split(':', 1)[1].strip(),
                hazards=[],
                steps=[],
                mitigations=[],
                compliance_refs=[]
            )
        elif current_procedure:
            # Extract hazards
            if re.match(r'(DANGER|WARNING|CAUTION):', line):
                current_procedure.hazards.append(line)
            # Extract steps
            elif re.match(r'^\d+\.\s+', line):
                current_procedure.steps.append(line)
            # Extract mitigations
            elif re.match(r'(Mitigation|Prevention):', line, re.I):
                current_procedure.mitigations.append(line)
    
    if current_procedure:
        procedures.append(current_procedure)
    
    return procedures
```

### 3. Material Datasheet Documents

**Parsing approach**: Tabular data extraction with property parsing

**Key extraction**:
- Material name and grade
- Material properties (tensile strength, fire rating, cost, etc.)
- Performance data and thresholds
- Usage guidelines and applications
- Standards and certifications (ASTM, ACI, AWS, AISC)

**Output**: `materials.md`

**Pseudocode**:
```python
def extract_material_properties(text: str) -> List[Material]:
    """Extract material data from datasheets."""
    materials = []
    
    # Use Docling if available for table extraction
    if extraction_mode == 'technical':
        tables = extract_tables_with_docling(text)
        for table in tables:
            material = parse_material_table(table)
            materials.append(material)
    else:
        # Fallback: parse structured text
        for line in text.split('\n'):
            # Pattern: "Property: Value Unit [Tolerance]"
            match = re.match(r'(.+?):\s*(.+?)\s*(PSI|MPa|°C|inches|mm|%)?(?:\s*\[(.+?)\])?', line)
            if match:
                prop, value, unit, tolerance = match.groups()
                materials.append(MaterialProperty(
                    name=prop.strip(),
                    value=value.strip(),
                    unit=unit or '',
                    tolerance=tolerance or ''
                ))
    
    return materials
```

### 4. Building Code Documents

**Parsing approach**: Regulatory section extraction

**Key extraction**:
- Code section references (§ X.Y.Z)
- Regulatory requirements and criteria
- Applicability (geographic jurisdiction, project type)
- Compliance verification criteria
- Penalties for non-compliance
- Cross-references to related codes

**Output**: `compliance.md`

**Pseudocode**:
```python
def extract_compliance_requirements(text: str) -> List[ComplianceItem]:
    """Extract regulatory requirements from building codes."""
    items = []
    
    for line in text.split('\n'):
        # Pattern: "§ X.Y.Z — Requirement"
        match = re.match(r'§\s*(\d+\.\d+(?:\.\d+)?)\s*—?\s*(.+)', line)
        if match:
            code_section, requirement = match.groups()
            item = ComplianceItem(
                code_section=code_section,
                requirement=requirement,
                applicability='',
                verification_criteria='',
                penalties=''
            )
            items.append(item)
    
    return items
```

### 5. Compliance Checklist Documents

**Parsing approach**: Item-based extraction

**Key extraction**:
- Checklist items
- Verification criteria for each item
- Responsible parties
- Sign-off requirements
- Related compliance rules

**Output**: `quick-reference.md`

**Pseudocode**:
```python
def extract_checklist_items(text: str) -> List[ChecklistItem]:
    """Extract checklist items from compliance documents."""
    items = []
    
    for line in text.split('\n'):
        # Pattern: "☐ Item description"
        if re.match(r'(☐|□|\[ \])\s*(.+)', line):
            item = ChecklistItem(
                description=line.split(None, 1)[1],
                verification_criteria='',
                responsible_party='',
                sign_off_required=False
            )
            items.append(item)
    
    return items
```

### 6. Installation Guide Documents

**Parsing approach**: Step-by-step extraction

**Key extraction**:
- Task names and descriptions
- Prerequisites and requirements
- Tools and materials needed
- Step-by-step instructions
- Verification and testing procedures
- Troubleshooting and common issues

**Output**: `procedures.md`

**Pseudocode**:
```python
def extract_installation_tasks(text: str) -> List[Task]:
    """Extract installation tasks from guides."""
    tasks = []
    current_task = None
    
    for line in text.split('\n'):
        # Detect task start
        if re.match(r'^(Task|Step|Procedure):\s*(.+)', line, re.I):
            if current_task:
                tasks.append(current_task)
            current_task = Task(
                name=line.split(':', 1)[1].strip(),
                prerequisites=[],
                tools=[],
                steps=[],
                verification=[],
                troubleshooting=[]
            )
        elif current_task:
            if re.match(r'Prerequisites?:', line, re.I):
                current_task.prerequisites.append(line)
            elif re.match(r'Tools?:', line, re.I):
                current_task.tools.append(line)
            elif re.match(r'^\d+\.\s+', line):
                current_task.steps.append(line)
            elif re.match(r'Verification:', line, re.I):
                current_task.verification.append(line)
            elif re.match(r'Troubleshooting:', line, re.I):
                current_task.troubleshooting.append(line)
    
    if current_task:
        tasks.append(current_task)
    
    return tasks
```

### 7. Blueprint Documents

**Parsing approach**: Metadata and description extraction

**Key extraction**:
- Visual descriptions (floor plans, elevations, sections, details)
- Reference information and dimensions
- Cross-references to specifications
- Legend and symbol explanations
- Note: Visual elements are noted but not extracted as images

**Output**: `sections/blueprint-*.md`

---

## Knowledge Organization

### Categories

1. **Procedures** (`procedures.md`)
   - Step-by-step construction tasks
   - Hazards and mitigations
   - Verification steps
   - Compliance references

2. **Specifications** (`specifications.md`)
   - Quantified requirements (dimensions, tolerances, grades)
   - Performance criteria
   - Material specifications
   - Standards and certifications

3. **Compliance** (`compliance.md`)
   - Regulatory requirements
   - Code sections and references
   - Verification criteria
   - Applicability and jurisdiction

4. **Materials** (`materials.md`)
   - Material properties and performance data
   - Selection criteria
   - Cost information
   - Standards (ASTM, ACI, AWS, AISC)

5. **Glossary** (`glossary.md`)
   - Construction-specific terms
   - Abbreviations and acronyms
   - Material grades and standards
   - Cross-references to sections

6. **Quick Reference** (`quick-reference.md`)
   - Decision tables and selection matrices
   - Material selection guides
   - Troubleshooting guides
   - Compliance checklists

### Cross-Referencing

All extracted content includes cross-references:
- Procedures reference compliance requirements
- Specifications reference material standards
- Compliance items reference applicable sections
- Glossary terms reference sections where used

---

## Output File Structure

```
~/.claude/skills/<skill_name>/
├── SKILL.md                    # Master file: overview, quick procedures, compliance checklist
├── sections/                   # One file per major section
│   ├── section-01-concrete.md
│   ├── section-02-steel.md
│   ├── section-03-safety.md
│   └── ...
├── procedures.md               # All step-by-step procedures with hazards/mitigations
├── specifications.md           # All quantified requirements with units/tolerances
├── compliance.md               # All regulatory requirements with verification criteria
├── materials.md                # All material properties with selection criteria
├── glossary.md                 # Construction terms, abbreviations, acronyms, standards
└── quick-reference.md          # Decision tables, material selection, troubleshooting
```

### SKILL.md Structure (Construction-Optimized)

```markdown
---
name: <skill_name>
description: Construction knowledge from "<Document Title>" by <Issuing Body>. 
             Use when <construction use case: on-site reference, compliance checking, 
             safety procedures, material selection>.
when_to_use: <trigger phrases based on document type and content>
allowed-tools: Read Grep
argument-hint: [procedure name, compliance requirement, material type, or section number]
---

# <Document Title>
**Issuing Body**: <Body> | **Version**: <Version> | **Effective Date**: <Date> | **Jurisdiction**: <Jurisdiction>

## How to Use This Skill

- **Without arguments** — `/skill-name` loads quick procedures and compliance checklist
- **With a procedure** — `/skill-name concrete-placement` → I find and explain the procedure
- **With compliance** — `/skill-name compliance` → I retrieve applicable code requirements
- **With material** — `/skill-name concrete` → I retrieve material properties and selection criteria
- **Browse** — ask "what procedures do you have?" to see the full index

---

## Quick Procedures
<!-- ~1,000 tokens: most common procedures with hazards and steps -->

---

## Compliance Checklist
<!-- ~500 tokens: key compliance items and verification criteria -->

---

## Section Index

| # | Title | Key Content |
|---|-------|-------------|
| [section-01](sections/section-01-*.md) | <Title> | <procedures>, <specs>, <compliance> |
| [section-02](sections/section-02-*.md) | <Title> | <procedures>, <specs>, <compliance> |

## Topic Index

- **<Procedure Name>** → procedures.md
- **<Material Type>** → materials.md
- **<Compliance Requirement>** → compliance.md
- **<Specification>** → specifications.md

## Supporting Files

- [procedures.md](procedures.md) — all step-by-step procedures
- [specifications.md](specifications.md) — all quantified requirements
- [compliance.md](compliance.md) — all regulatory requirements
- [materials.md](materials.md) — all material properties
- [glossary.md](glossary.md) — construction terms and abbreviations
- [quick-reference.md](quick-reference.md) — decision tables and troubleshooting

---

## Document Metadata

- **Title**: <Document Title>
- **Issuing Body**: <Body>
- **Version**: <Version>
- **Effective Date**: <Date>
- **Jurisdiction**: <Jurisdiction>
- **Applicability**: <Project types, geographic scope>
- **Standards Referenced**: <ASTM, ACI, AWS, AISC, etc.>
- **Related Documents**: <Cross-references>

---

## Scope & Limits

This skill covers the document content only. For hands-on implementation on your project,
combine with project-specific tools and local regulations. For topics beyond this document,
check related skills or consult with compliance officers.
```

---

## Reuse of Existing Code

### Reusable Components from extract.py

1. **PDF extraction methods**
   - `extract_with_pdftotext()` — fast text extraction
   - `extract_with_pypdf2()` — fallback extraction
   - `extract_with_pdfminer()` — fallback extraction
   - `extract_with_docling()` — structure-aware extraction (tables, code blocks)

2. **EPUB extraction methods**
   - `extract_with_ebooklib()` — primary EPUB extraction
   - `extract_with_zipfile()` — fallback EPUB extraction
   - `extract_epub()` — wrapper for EPUB extraction

3. **Utility functions**
   - `estimate_tokens()` — token counting
   - `count_pages()` — page counting for PDFs
   - `count_epub_chapters()` — chapter counting for EPUBs
   - `detect_structure()` — basic structure detection

4. **File format detection**
   - Magic byte checking (PDF: `%PDF`, EPUB: `PK` zip header)
   - File extension validation

### New Modules Required

1. **`detect_construction_type.py`**
   - `detect_construction_document_type()` — classify document type
   - `analyze_structure_patterns()` — detect structural patterns
   - `analyze_terminology()` — detect construction terminology

2. **`parse_specification.py`**
   - `extract_hierarchical_sections()` — parse nested sections
   - `extract_quantified_data()` — extract dimensions, tolerances, grades

3. **`parse_safety_manual.py`**
   - `extract_procedures()` — parse step-by-step procedures
   - `extract_hazards()` — identify hazard warnings
   - `extract_mitigations()` — extract mitigation measures

4. **`parse_material_datasheet.py`**
   - `extract_material_properties()` — parse material data
   - `parse_material_table()` — parse tabular material data
   - `extract_material_grades()` — identify material standards

5. **`parse_building_code.py`**
   - `extract_compliance_requirements()` — parse regulatory requirements
   - `extract_code_sections()` — identify code references

6. **`parse_compliance_checklist.py`**
   - `extract_checklist_items()` — parse checklist items
   - `extract_verification_criteria()` — parse verification steps

7. **`parse_installation_guide.py`**
   - `extract_installation_tasks()` — parse installation steps
   - `extract_prerequisites()` — identify prerequisites
   - `extract_troubleshooting()` — parse troubleshooting guides

8. **`extract_construction_metadata.py`**
   - `extract_metadata()` — extract title, version, effective date, jurisdiction
   - `extract_applicability()` — extract geographic scope, project types
   - `extract_standards_referenced()` — identify standards (ASTM, ACI, AWS, AISC)

9. **`extract_construction_terminology.py`**
   - `extract_construction_terms()` — identify construction-specific terms
   - `extract_abbreviations()` — identify abbreviations and acronyms
   - `extract_material_standards()` — identify material grades and standards
   - `build_glossary()` — create glossary with cross-references

10. **`generate_construction_skill.py`**
    - `generate_skill_md()` — generate master SKILL.md
    - `generate_section_files()` — generate section-specific files
    - `generate_procedures_md()` — generate procedures.md
    - `generate_specifications_md()` — generate specifications.md
    - `generate_compliance_md()` — generate compliance.md
    - `generate_materials_md()` — generate materials.md
    - `generate_glossary_md()` — generate glossary.md
    - `generate_quick_reference_md()` — generate quick-reference.md

---

## Construction-Specific Parsing Functions

### Quantified Data Extraction

```python
def extract_quantified_data(text: str) -> List[Specification]:
    """
    Extract all quantified specifications from text.
    
    Patterns:
    - "Dimension: 12 inches ±0.5 inches"
    - "Strength: 4000 PSI minimum"
    - "Temperature: 20–25°C"
    - "Cost: $50 per unit"
    """
    specs = []
    
    # Pattern: "Property: Value Unit [Tolerance]"
    pattern = r'(.+?):\s*(.+?)\s*(PSI|MPa|°C|°F|inches|mm|feet|meters|%)?(?:\s*\[(.+?)\])?'
    
    for match in re.finditer(pattern, text):
        prop, value, unit, tolerance = match.groups()
        specs.append(Specification(
            name=prop.strip(),
            value=value.strip(),
            unit=unit or '',
            tolerance=tolerance or ''
        ))
    
    return specs
```

### Construction Terminology Extraction

```python
def extract_construction_terminology(text: str) -> Dict[str, Term]:
    """
    Extract construction-specific terms and abbreviations.
    
    Includes:
    - Common abbreviations: PSI, OSHA, ADA, LEED, BIM, MEP, RFI, PCO
    - Material standards: ASTM, ACI, AWS, AISC
    - Construction terms: definitions from context
    """
    glossary = {}
    
    # Common construction abbreviations
    abbreviations = {
        'PSI': 'Pounds per Square Inch',
        'OSHA': 'Occupational Safety and Health Administration',
        'ADA': 'Americans with Disabilities Act',
        'LEED': 'Leadership in Energy and Environmental Design',
        'BIM': 'Building Information Modeling',
        'MEP': 'Mechanical, Electrical, Plumbing',
        'RFI': 'Request for Information',
        'PCO': 'Potential Change Order',
        'ASTM': 'American Society for Testing and Materials',
        'ACI': 'American Concrete Institute',
        'AWS': 'American Welding Society',
        'AISC': 'American Institute of Steel Construction'
    }
    
    for abbr, definition in abbreviations.items():
        if abbr in text:
            glossary[abbr] = Term(
                name=abbr,
                definition=definition,
                context='construction',
                related_terms=[]
            )
    
    return glossary
```

### Metadata Extraction

```python
def extract_metadata(text: str) -> Dict:
    """
    Extract document metadata.
    
    Extracts:
    - Title, author/issuing body, version, effective date
    - Revision history
    - Jurisdiction, applicability, regulatory body
    - Standards referenced
    - Cross-references
    """
    metadata = {
        'title': '',
        'issuing_body': '',
        'version': '',
        'effective_date': '',
        'revision_history': [],
        'jurisdiction': '',
        'applicability': '',
        'regulatory_body': '',
        'standards_referenced': [],
        'cross_references': []
    }
    
    # Extract title (usually first line or in header)
    lines = text.split('\n')
    metadata['title'] = lines[0].strip() if lines else ''
    
    # Extract version (pattern: "Version X.Y" or "Rev. X")
    version_match = re.search(r'(?:Version|Rev\.?)\s*(\d+\.\d+|\d+)', text, re.I)
    if version_match:
        metadata['version'] = version_match.group(1)
    
    # Extract effective date
    date_match = re.search(r'(?:Effective|Issued).*?(\d{1,2}/\d{1,2}/\d{4}|\w+\s+\d{1,2},\s+\d{4})', text, re.I)
    if date_match:
        metadata['effective_date'] = date_match.group(1)
    
    # Extract standards referenced
    standards = re.findall(r'(ASTM|ACI|AWS|AISC)\s+[\w\-]+', text)
    metadata['standards_referenced'] = list(set(standards))
    
    return metadata
```

---

## Metadata Handling

### Extraction

- **Document identity**: Title, author/issuing body, version, effective date
- **Revision history**: Track changes and updates
- **Applicability**: Geographic jurisdiction, project type, regulatory body
- **Standards**: Standards referenced (ASTM, ACI, AWS, AISC)
- **Cross-references**: Related documents, codes, internal references

### Warnings

- Flag superseded documents (if newer version detected)
- Warn if document is outdated (effective date > 5 years ago)
- Alert if jurisdiction-specific content may not apply

### Storage

- Include metadata in master SKILL.md for reference
- Store in structured format for easy retrieval
- Enable users to verify document version and applicability

---

## Skill Generation Differences from Book-to-Skill

| Aspect | Book-to-Skill | Construction Converter |
|--------|---------------|------------------------|
| **Organization** | By chapters | By procedures, specs, compliance, materials |
| **Core content** | Frameworks & mental models | Quick procedures & compliance checklist |
| **Glossary** | Key concepts | Construction terms, abbreviations, standards |
| **Patterns file** | Techniques & design patterns | Step-by-step procedures |
| **Cheatsheet file** | Decision tables | Material selection, troubleshooting, compliance |
| **Use case** | Study & reference | On-site lookup, compliance verification |
| **Metadata** | Title, author, pages | Version, effective date, jurisdiction, applicability |
| **Trigger phrases** | Study, frameworks, concepts | Procedures, compliance, materials, on-site |

---

## Integration Points

### User Invocation

```bash
/construction-converter /path/to/document.pdf [skill-name]
/construction-converter /path/to/document.epub [skill-name]
```

### Workflow Steps

1. **Validate input**: Check file exists and is PDF/EPUB
2. **Extract text**: Use existing extraction pipeline
3. **Detect document type**: Analyze structure and terminology
4. **Confirm type**: Ask user if confidence < 80%
5. **Select extraction mode**: Docling for technical, text for procedural
6. **Parse by type**: Apply document-specific parsing
7. **Organize content**: Group into procedures, specs, compliance, materials
8. **Generate skill**: Create SKILL.md and supporting files
9. **Report**: Show generated files and usage instructions

### On-Site Usage

```bash
/skill-name                           # Load quick procedures and compliance checklist
/skill-name concrete-placement        # Find and explain a specific procedure
/skill-name compliance                # Retrieve applicable code requirements
/skill-name concrete                  # Retrieve material properties and selection criteria
/skill-name "what procedures do you have?"  # Browse available procedures
```

---

## Implementation Phases

### Phase 1: Foundation
- Reuse existing extraction pipeline
- Implement document type detection
- Create basic parsing for Specification and Safety Manual types

### Phase 2: Expansion
- Add Material Datasheet, Building Code, Compliance Checklist parsing
- Implement construction terminology extraction
- Add metadata handling

### Phase 3: Optimization
- Add Installation Guide and Blueprint parsing
- Implement cross-referencing between files
- Optimize token usage and file generation

### Phase 4: Polish
- Add user interface improvements
- Implement validation and error handling
- Create comprehensive documentation

---

## Quality Criteria

1. **Accuracy**: Extracted content matches source document exactly
2. **Completeness**: All relevant content extracted (procedures, specs, compliance, materials)
3. **Organization**: Content organized logically for on-site reference
4. **Usability**: Skill files are easy to search and navigate
5. **Precision**: Quantified data (dimensions, tolerances, units) preserved exactly
6. **Compliance**: Regulatory requirements extracted without interpretation
7. **Terminology**: Construction terms and abbreviations clearly defined
8. **Cross-referencing**: Related content linked across files

