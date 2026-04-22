# Indic Language Detection and Rendering Flow — SahmatOS

## 1. Language Detection Pipeline

```mermaid
flowchart TD
    A[Consent request received] --> B{X-DP-Language header\nor languageCode param?}

    B -->|Yes| C[Use explicit override]
    B -->|No| D{Accept-Language header?}

    D -->|"e.g. ta-IN, hi-IN"| E[Parse Accept-Language\nNormalise to BCP 47]
    D -->|Missing| F{X-Device-Locale header?}

    F -->|"e.g. mr_IN"| G[Parse device locale]
    F -->|Missing| H{IP Geolocation available?}

    H -->|Yes| I[IP → State → Primary Language\ne.g. Karnataka → kn]
    H -->|No| J{DP has registered\nlanguage preference?}

    J -->|Yes| K[Use dp_identifiers.preferred_language]
    J -->|No| L{Tenant has\ndefault_language?}

    L -->|Yes| M[Use tenant.default_language]
    L -->|No| N[Fallback: hi\nHindi — national link language]

    C --> O[Normalise language code\nusing LANGUAGE_CODE_MAP]
    E --> O
    G --> O
    I --> O
    K --> O
    M --> O
    N --> O

    O --> P{Approved translation\nexists for this language?}

    P -->|Yes| Q[Retrieve from Redis cache\nor PostgreSQL]
    P -->|No| R[Fallback language chain:\n1. Tenant default language\n2. Hindi\n3. English]

    R --> Q

    Q --> S[Build LanguageContext:\n- code, script, direction\n- fontFamily, fontUrl\n- textDirection ltr/rtl]

    S --> T[Return to rendering engine]
```

---

## 2. Consent Widget Rendering Pipeline

```mermaid
flowchart LR
    A[Language detected:\ne.g. ur = Urdu] --> B[Look up font map]

    B --> C{Font already\nloaded in browser?}
    C -->|Yes| D[Skip font load]
    C -->|No| E{Is RTL language?}

    E -->|Yes: ur, ks-Arab, sd-Arab| F[Load Noto Nastaliq Urdu\nor Noto Sans Arabic\n~520KB lazy]
    E -->|No| G{Script type?}

    G -->|Devanagari| H[Already bundled:\nNoto Sans Devanagari]
    G -->|Tamil| I[Already bundled:\nNoto Sans Tamil]
    G -->|Other| J[Lazy load:\nScript-specific Noto font\nfrom CDN]

    D --> K[Retrieve translated consent content\nfrom API/cache]
    F --> K
    H --> K
    I --> K
    J --> K

    K --> L[Build widget DOM]

    L --> M{Is RTL?}
    M -->|Yes| N[Apply dir=rtl to root\nMirror layout:\n- Buttons right-to-left\n- Back arrow flipped\n- Progress bar RTL]
    M -->|No| O[Default dir=ltr layout]

    N --> P[Render consent widget]
    O --> P

    P --> Q[HarfBuzz text shaping\nfor Nastaliq / complex scripts]
    Q --> R[Display to user in\ntheir native script]
```

---

## 3. Translation Quality Pipeline

```mermaid
flowchart TD
    A[DF defines new Purpose\nin English] --> B[Template Engine\ngenerates English base text]

    B --> C[On-premise LLM\nfine-tuned for legal Indic translation\nHosted: AWS Mumbai]

    C --> D[Translates to all 22 languages\nin parallel]

    D --> E{Quality threshold check:\n- Legal term accuracy score\n- Reading level ≤ 6th grade\n- Length within bounds}

    E -->|Pass| F[Enter Human Review Queue]
    E -->|Fail| G[Flag for re-translation\nwith quality notes]
    G --> C

    F --> H[Native speaker review\n(per language):\n- Terminology accuracy\n- Cultural appropriateness\n- Clarity check]

    H --> I{Reviewer decision}

    I -->|Approved| J[Legal review\n(language-qualified lawyer\nper language group)]
    I -->|Rejected + notes| G

    J --> K{Legal decision}
    K -->|Approved| L[Mark translation: status=approved\nin consent_purpose_translations]
    K -->|Needs revision| M[Return to translator\nwith legal notes]
    M --> C

    L --> N[Translation available\nfor production use]

    N --> O[Cache in Redis:\nlang:notice:{purposeId}:{langCode}\nTTL: 24hr]

    O --> P[Served to users\nin their language]
```

---

## 4. Language Coverage Status and Rollout

```mermaid
gantt
    title SahmatOS Language Rollout Timeline
    dateFormat YYYY-MM
    axisFormat %b %Y

    section Phase 1 (MVP — CM Registration)
    Hindi (hi)             :done, 2026-01, 2026-11
    Tamil (ta)             :done, 2026-01, 2026-11
    Telugu (te)            :done, 2026-02, 2026-11
    Kannada (kn)           :done, 2026-02, 2026-11
    Bengali (bn)           :done, 2026-03, 2026-11
    Marathi (mr)           :done, 2026-03, 2026-11
    Gujarati (gu)          :done, 2026-04, 2026-11

    section Phase 2 (v1.1 — Jan 2027)
    Punjabi (pa)           :active, 2026-08, 2027-01
    Malayalam (ml)         :active, 2026-08, 2027-01
    Odia (or)              :active, 2026-09, 2027-01
    Urdu (ur) — RTL        :active, 2026-09, 2027-01
    Assamese (as)          :active, 2026-10, 2027-01

    section Phase 3 (v1.2 — Mar 2027)
    Konkani (kok)          :2026-11, 2027-03
    Kashmiri/Nastaliq (ks-Arab) — RTL :2026-11, 2027-03
    Maithili (mai)         :2026-12, 2027-03
    Nepali (ne)            :2026-12, 2027-03
    Sindhi/Perso-Arabic (sd-Arab) — RTL :2026-12, 2027-03

    section Phase 4 (v1.3 — May 2027 Full Enforcement)
    Bodo (brx)             :2027-01, 2027-05
    Dogri (doi)            :2027-01, 2027-05
    Manipuri/Meitei Mayek (mni-Mtei) :2027-02, 2027-05
    Santali/Ol Chiki (sat-Olck)      :2027-02, 2027-05
    Sanskrit (sa)          :2027-03, 2027-05
```

---

## 5. Script Coverage Map

```mermaid
graph TD
    subgraph devanagari_group["Devanagari Script (9 languages)"]
        hi["Hindi (hi)"]
        mr["Marathi (mr)"]
        bodo["Bodo (brx)"]
        dogri["Dogri (doi)"]
        konkani["Konkani (kok)"]
        maithili["Maithili (mai)"]
        nepali["Nepali (ne)"]
        sanskrit["Sanskrit (sa)"]
        sindhi_deva["Sindhi/Devanagari (sd-Deva)"]
    end

    subgraph dravidian["Dravidian Scripts"]
        ta["Tamil (ta)"]
        te["Telugu (te)"]
        kn["Kannada (kn)"]
        ml["Malayalam (ml)"]
    end

    subgraph eastern["Eastern/Northeast Scripts"]
        bn["Bengali (bn)"]
        as_lang["Assamese (as)"]
        or_lang["Odia (or)"]
        mni["Manipuri — Meitei Mayek (mni-Mtei)"]
        sat["Santali — Ol Chiki (sat-Olck)"]
    end

    subgraph northwestern["Northwestern Scripts"]
        pa["Punjabi — Gurmukhi (pa)"]
        gu["Gujarati (gu)"]
    end

    subgraph rtl_scripts["RTL Scripts (Perso-Arabic family)"]
        ur["Urdu — Nastaliq (ur)"]
        ks["Kashmiri — Nastaliq (ks-Arab)"]
        sd_arab["Sindhi — Perso-Arabic (sd-Arab)"]
    end

    font_deva["Noto Sans Devanagari\n~380KB"]
    font_tamil["Noto Sans Tamil\n~220KB"]
    font_nastaliq["Noto Nastaliq Urdu\n~520KB (RTL)"]

    devanagari_group --> font_deva
    ta --> font_tamil
    ur --> font_nastaliq
    ks --> font_nastaliq

    style rtl_scripts fill:#ffe4b5,stroke:#ff9900
    style devanagari_group fill:#e6f3ff,stroke:#4499ff
```
