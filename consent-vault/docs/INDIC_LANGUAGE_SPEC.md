# Indic Language Support Specification — SahmatOS

**Version:** 1.0  
**Date:** 2026-04-22  

---

## Overview

SahmatOS is built for India's 900 million internet users — most of whom interact digitally in a language other than English. DPDPA Section 5 / Rule 3 requires consent notices to be delivered in the user's interface language. This specification defines how SahmatOS implements language support across all 22 Eighth Schedule languages mandated by the DPDP Rules 2025.

**Design principle:** Language is infrastructure, not a feature. Language support is not an add-on — every system component from DB schema to API response format to font loading is designed with 22-language support from day one.

---

## 1. The 22 Eighth Schedule Languages

| # | Language | Script | ISO 639-1 | ISO 639-3 | Direction | Primary Regions |
|---|----------|--------|-----------|-----------|-----------|-----------------|
| 1 | Assamese | Bengali (Assamese variant) | as | asm | LTR | Assam |
| 2 | Bengali | Bengali | bn | ben | LTR | West Bengal, Bangladesh border |
| 3 | Bodo | Devanagari | — | brx | LTR | Assam (Bodoland) |
| 4 | Dogri | Devanagari | — | doi | LTR | Jammu |
| 5 | Gujarati | Gujarati | gu | guj | LTR | Gujarat |
| 6 | Hindi | Devanagari | hi | hin | LTR | Most of India (link language) |
| 7 | Kannada | Kannada | kn | kan | LTR | Karnataka |
| 8 | Kashmiri | Nastaliq (Perso-Arabic) | ks | kas | **RTL** | Jammu & Kashmir |
| 9 | Konkani | Devanagari / Roman | kok | kok | LTR | Goa, coastal Karnataka |
| 10 | Maithili | Devanagari / Mithilakshar | — | mai | LTR | Bihar, Nepal border |
| 11 | Malayalam | Malayalam | ml | mal | LTR | Kerala |
| 12 | Manipuri (Meitei) | Meitei Mayek | — | mni | LTR | Manipur |
| 13 | Marathi | Devanagari | mr | mar | LTR | Maharashtra |
| 14 | Nepali | Devanagari | ne | nep | LTR | Sikkim, Darjeeling |
| 15 | Odia | Odia (Oriya) | or | ori | LTR | Odisha |
| 16 | Punjabi | Gurmukhi | pa | pan | LTR | Punjab |
| 17 | Sanskrit | Devanagari | sa | san | LTR | Pan-India (classical) |
| 18 | Santali | Ol Chiki | — | sat | LTR | Jharkhand, West Bengal |
| 19 | Sindhi | Perso-Arabic / Devanagari | sd | snd | **RTL** (Perso-Arabic) | Rajasthan, Gujarat |
| 20 | Tamil | Tamil | ta | tam | LTR | Tamil Nadu, Sri Lanka border |
| 21 | Telugu | Telugu | te | tel | LTR | Andhra Pradesh, Telangana |
| 22 | Urdu | Nastaliq (Perso-Arabic) | ur | urd | **RTL** | UP, Maharashtra, Hyderabad |

### RTL Languages (3)
Kashmiri, Sindhi (Perso-Arabic variant), and Urdu require right-to-left rendering. All UI layouts must mirror for these languages: close buttons move to left, back arrows reverse, text alignment changes.

### Complex Scripts Requiring Special Handling
- **Nastaliq (Urdu, Kashmiri):** Contextual letter forms, ligatures — requires HarfBuzz shaping
- **Meitei Mayek (Manipuri):** Relatively new Unicode block (U+ABC0–U+ABFF)
- **Ol Chiki (Santali):** Alphabetic script, Unicode block U+1C50–U+1C7F
- **Assamese:** Superficially identical to Bengali script but different characters (ক্ষ differs)

---

## 2. Language Detection Pipeline

When a Data Principal opens a consent widget, language is determined in this order:

```
1. Explicit user selection (highest priority — always override)
        ↓
2. Accept-Language HTTP header (browser setting)
        ↓
3. Device locale (extracted from User-Agent / native SDK)
        ↓
4. Geolocation-based inference (IP → state → primary language)
        ↓
5. Tenant-configured default language
        ↓
6. Hindi (fallback — most widely understood across India)
```

### Language Code Normalisation

Incoming language codes must be normalised to our internal format:

```typescript
// Internal format: BCP 47 with script subtag where needed
const LANGUAGE_CODE_MAP: Record<string, string> = {
  'hi': 'hi',           // Hindi (Devanagari implied)
  'hi-IN': 'hi',
  'ta': 'ta',           // Tamil
  'ta-IN': 'ta',
  'ur': 'ur',           // Urdu (Nastaliq)
  'ur-IN': 'ur',
  'ks': 'ks-Arab',      // Kashmiri — Nastaliq (Arab script) variant
  'ks-Arab': 'ks-Arab',
  'ks-Deva': 'ks-Deva', // Kashmiri — Devanagari variant (less common)
  'mni': 'mni-Mtei',    // Manipuri — Meitei Mayek
  'mni-Mtei': 'mni-Mtei',
  'sat': 'sat-Olck',    // Santali — Ol Chiki
  'sat-Olck': 'sat-Olck',
  'sd': 'sd-Arab',      // Sindhi — Perso-Arabic (RTL)
  'sd-Deva': 'sd-Deva', // Sindhi — Devanagari (LTR, Maharashtra variant)
  // ... full map in i18n/language-codes.ts
};
```

### IP-to-Language Mapping (Fallback Only)

```typescript
// State → primary language (approximate, for fallback only)
const STATE_PRIMARY_LANGUAGE: Record<string, string> = {
  'Maharashtra': 'mr',
  'Karnataka': 'kn',
  'Tamil Nadu': 'ta',
  'West Bengal': 'bn',
  'Andhra Pradesh': 'te',
  'Telangana': 'te',
  'Gujarat': 'gu',
  'Rajasthan': 'hi',
  'Uttar Pradesh': 'hi',
  'Madhya Pradesh': 'hi',
  'Jharkhand': 'hi', // Santali also significant but Hindi safer fallback
  'Assam': 'as',
  'Kerala': 'ml',
  'Punjab': 'pa',
  'Odisha': 'or',
  'Bihar': 'hi', // Maithili also spoken but Hindi safer
  'Jammu and Kashmir': 'ur', // Kashmiri also, but Urdu is official
  // ...
};
```

---

## 3. Translation Architecture

### 3.1 Why On-Premise LLM (Not External APIs)

Consent notice text contains the following sensitive information:
- Business name and identity
- Specific data categories being collected
- Purposes of data processing
- Retention periods
- Rights granted to the Data Principal

Sending this through Google Translate, Azure Translator, or similar external APIs:
1. Potentially transfers PII-adjacent data outside India (Section 16 risk)
2. Caches data on foreign servers
3. Creates discoverable evidence trail in foreign jurisdictions
4. Returns generic translations that may not be legally precise

**Decision: Fine-tuned LLM hosted on-premise in AWS Mumbai.**

### 3.2 Translation Quality Requirements

Consent translation is not content translation — it is legal-accuracy translation. Requirements:

| Requirement | Standard |
|-------------|---------|
| Legal terminology accuracy | Native-speaker lawyer review for each language |
| DPDPA-specific terms | Consistent glossary per language (see Appendix A) |
| Reading level | 6th-grade comprehension (simple sentences) |
| Tone | Respectful, not bureaucratic |
| Length | Must not exceed English by >40% (mobile screen constraint) |

### 3.3 Translation Pipeline

```
[Business registers purpose] 
        ↓
[Template engine creates English base consent text]
        ↓
[On-premise LLM translates to all 22 languages]
        ↓
[Human review queue: native speaker + legal review per language]
        ↓ (review gate — must pass before production)
[Approved translations stored in consent_templates DB table]
        ↓
[Translation cache (Redis, 24hr TTL) for hot-path rendering]
        ↓
[Widget renders in DP's language at consent collection time]
```

### 3.4 DPDPA Glossary (Critical Terms)

The following terms must be consistently translated — not ad-hoc per request:

| English Term | Hindi | Tamil | Telugu | Kannada | Bengali |
|-------------|-------|-------|--------|---------|---------|
| Consent | सहमति (sahmatī) | சம்மதம் (cammatam) | అంగీకారం (aṅgīkāraṁ) | ಒಪ್ಪಿಗೆ (oppige) | সম্মতি (sammati) |
| Data Principal | डेटा मालिक (ḍeṭā mālik) | தரவு உரிமையாளர் | డేటా యజమాని | ಡೇಟಾ ಮಾಲೀಕ | ডেটা মালিক |
| Data Fiduciary | डेटा कस्टोडियन | தரவு பாதுகாவலர் | డేటా కస్టోడియన్ | ಡೇಟಾ ಕಸ್ಟೋಡಿಯನ್ | ডেটা কাস্টোডিয়ান |
| Withdrawal | वापसी (vāpsī) | திரும்பப் பெறல் | ఉపసంహరణ | ಹಿಂಪಡೆಯುವಿಕೆ | প্রত্যাহার |
| Erasure | मिटाना (miṭānā) | அழித்தல் | తొలగింపు | ಅಳಿಸುವಿಕೆ | মুছে ফেলা |
| Purpose | उद्देश्य (uddeśya) | நோக்கம் | ప్రయోజనం | ಉದ್ದೇಶ | উদ্দেশ্য |
| Personal Data | व्यक्तिगत डेटा | தனிப்பட்ட தரவு | వ్యక్తిగత డేటా | ವೈಯಕ್ತಿಕ ಡೇಟಾ | ব্যক্তিগত তথ্য |

*Full glossary for all 22 languages in `i18n/glossary/` directory.*

---

## 4. Font Loading Strategy

Each Indic script requires specific fonts. Loading all fonts upfront (>2MB) is unacceptable on 4G.

### 4.1 Font Matrix

| Script | Font | Size | Languages | Loading |
|--------|------|------|-----------|---------|
| Devanagari | Noto Sans Devanagari | 380KB | Hindi, Marathi, Bodo, Dogri, Konkani, Maithili, Nepali, Sanskrit | Bundle (most common) |
| Tamil | Noto Sans Tamil | 220KB | Tamil | Bundle |
| Telugu | Noto Sans Telugu | 280KB | Telugu | Lazy |
| Kannada | Noto Sans Kannada | 260KB | Kannada | Lazy |
| Bengali | Noto Sans Bengali | 320KB | Bengali, Assamese | Lazy |
| Gujarati | Noto Sans Gujarati | 200KB | Gujarati | Lazy |
| Gurmukhi | Noto Sans Gurmukhi | 180KB | Punjabi | Lazy |
| Odia | Noto Sans Oriya | 190KB | Odia | Lazy |
| Malayalam | Noto Sans Malayalam | 290KB | Malayalam | Lazy |
| Nastaliq (RTL) | Noto Nastaliq Urdu | 520KB | Urdu, Kashmiri (Arab) | Lazy (trigger on RTL detect) |
| Perso-Arabic | Noto Sans Arabic | 300KB | Sindhi (Arab) | Lazy (trigger on RTL detect) |
| Meitei Mayek | Noto Sans Meitei Mayek | 85KB | Manipuri | Lazy |
| Ol Chiki | Noto Sans Ol Chiki | 45KB | Santali | Lazy |

### 4.2 Font Loading Sequence

```typescript
// Called immediately when language is determined
async function loadFontForLanguage(langCode: string): Promise<void> {
  const fontConfig = FONT_MAP[langCode];
  if (!fontConfig) return; // Latin fallback already loaded
  
  // Check if already loaded
  if (document.fonts.check(`12px "${fontConfig.fontFamily}"`)) return;
  
  // Load from CDN (fonts.gstatic.com mirror on Indian CDN for latency)
  const font = new FontFace(fontConfig.fontFamily, `url(${fontConfig.url})`);
  await font.load();
  document.fonts.add(font);
  
  // Apply RTL direction if needed
  if (RTL_LANGUAGES.includes(langCode)) {
    document.querySelector('.consent-widget')?.setAttribute('dir', 'rtl');
  }
}
```

### 4.3 Bundled vs Lazy Loading Decision

**Bundled (included in initial JS bundle):**
- Devanagari — covers Hindi, Marathi, and 6 other languages (highest combined reach)
- Tamil — second-largest regional language online
- Latin — fallback for English interface elements

**Lazy loaded (fetched on demand):**
- All other scripts — loaded only when user selects or is detected for that language
- RTL scripts (Nastaliq, Perso-Arabic) — loaded on RTL language detection
- Target: font available within 800ms on 4G after language detection

---

## 5. UI Layout Requirements

### 5.1 RTL Layout Mirroring

For Urdu, Kashmiri (Nastaliq), and Sindhi (Perso-Arabic):

```css
/* Applied when dir="rtl" is set on widget root */
[dir="rtl"] .consent-widget {
  direction: rtl;
  text-align: right;
}

[dir="rtl"] .consent-actions {
  flex-direction: row-reverse; /* Grant/Deny button order mirrors */
}

[dir="rtl"] .consent-back-arrow {
  transform: scaleX(-1); /* Arrow flips */
}

[dir="rtl"] .consent-progress {
  direction: rtl; /* Progress bar fills right-to-left */
}
```

### 5.2 Text Length Accommodation

Indic translations can be significantly longer than English. UI must accommodate:

| Language | Typical expansion vs English | Max line count (consent notice) |
|----------|-----------------------------|---------------------------------|
| Hindi | +15–25% | 8 lines |
| Tamil | +20–30% | 9 lines |
| Bengali | +10–20% | 8 lines |
| Telugu | +25–35% | 10 lines |
| Urdu | +20–30% | 9 lines (RTL) |

**Rule:** Consent notice container is scrollable, not fixed-height. No text truncation.

### 5.3 Language Selector Component

```
┌────────────────────────────────┐
│  अपनी भाषा चुनें               │  ← Always shown in Hindi first
│  Choose your language          │  ← English subtitle
│                                │
│  🔍  खोजें / Search            │  ← Search box (for 22 options)
│                                │
│  Hindi (हिंदी)            ✓   │  ← Current language
│  Tamil (தமிழ்)                 │
│  Telugu (తెలుగు)               │
│  Bengali (বাংলা)               │
│  Marathi (मराठी)               │
│  ─────────────────────────     │
│  [See all 22 languages ↓]      │
└────────────────────────────────┘
```

Rules:
- Language names always displayed in **the language itself** (not transliterated)
- Current language shown with checkmark
- Top 5 most common languages shown first, then divider, then remaining
- Search available for the full 22-language list
- Language selection persists in user profile (if authenticated) or localStorage

### 5.4 Consent Widget Layout (Mobile-First)

Designed for 5.5" Android screen at 360×800 viewport:

```
┌──────────────────────────────────────┐ ─┐
│  [🏢 Company Logo]                   │  │ Header (60px)
│  [Language: हिंदी ▾]                 │  │
├──────────────────────────────────────┤ ─┤
│                                      │  │
│  [Business Name] आपसे यह डेटा       │  │
│  इस उद्देश्य के लिए माँग रहा है:   │  │
│                                      │  │ Notice (scrollable)
│  📊 [Purpose Name]                  │  │
│  [Purpose Description in Hindi]      │  │
│                                      │  │
│  📅 कितने समय के लिए: [Duration]    │  │
│  🗑️ मिटाने का अधिकार: हाँ          │  │
│                                      │  │
├──────────────────────────────────────┤ ─┤
│  [और जानें ▾]  [पूरी नीति देखें]   │  │ Expandable detail
├──────────────────────────────────────┤ ─┤
│                                      │  │
│  ✅ [मैं सहमत हूँ]  ❌ [मैं मना करता हूँ] │  │ Action buttons (56px each)
│                                      │  │
└──────────────────────────────────────┘ ─┘
```

Key rules:
- Primary action buttons minimum 56px height (touch target)
- "I agree" and "I decline" equally prominent (no dark pattern)
- "I decline" never grey, hidden, or smaller than "I agree"
- Scroll indicator if notice text > 3 lines

---

## 6. WhatsApp Consent Flow (Conversational)

For users who interact via WhatsApp Business API (primary channel for Tier 2-3):

### 6.1 Conversation Template (Hindi)

```
[Business Name] आपका संदेश:

नमस्ते [User Name] जी! 🙏

[Business Name] आपसे यह जानकारी एकत्र करना चाहता है:
📊 [Data Category]
🎯 उद्देश्य: [Purpose in Hindi]
📅 अवधि: [Duration in Hindi]

क्या आप सहमत हैं?

1️⃣ हाँ, मैं सहमत हूँ
2️⃣ नहीं, मैं सहमत नहीं
3️⃣ और जानकारी चाहिए

जवाब दें: 1, 2, या 3
```

### 6.2 Language Selection in WhatsApp

1. First message always in the business's default language
2. If user replies in a different language, detect language and switch
3. "भाषा बदलें / Change Language" always available as menu option
4. Language preference stored against phone number

### 6.3 Consent Artifact for WhatsApp

WhatsApp conversations do not have cryptographic signatures natively. For WhatsApp-collected consents:
- `collection_method = 'WHATSAPP'`
- `artifact.channel_evidence` contains: WhatsApp WABA ID, message ID, timestamp, read receipt
- ECDSA signature applied by SahmatOS backend after collecting response
- Strength: weaker than widget (no client-side attestation) — must be disclosed in audit

---

## 7. Testing Requirements for Language Support

### 7.1 Automated Tests

```typescript
// Every supported language must pass these tests
describe('Language: {language}', () => {
  it('renders consent notice without character corruption', async () => {
    // Check for replacement characters (U+FFFD)
    const rendered = await renderConsentNotice('hi', testPurpose);
    expect(rendered).not.toContain('\uFFFD');
  });
  
  it('applies correct text direction', async () => {
    const widget = await mountWidget('ur'); // Urdu - RTL
    expect(widget.root.getAttribute('dir')).toBe('rtl');
  });
  
  it('loads correct font family', async () => {
    await renderConsentNotice('ta', testPurpose); // Tamil
    const fontLoaded = document.fonts.check('12px "Noto Sans Tamil"');
    expect(fontLoaded).toBe(true);
  });
  
  it('stores language code in consent artifact', async () => {
    const artifact = await grantConsent('bn', testPrincipal, testPurpose);
    expect(artifact.language_code).toBe('bn');
  });
});
```

### 7.2 Human QA Checklist (Per Language, Pre-Production)

- [ ] Native speaker review of all consent templates
- [ ] Legal translation review by language-qualified lawyer
- [ ] Rendering on lowest-spec supported device (2GB RAM Android)
- [ ] Font rendering check (no tofu boxes □)
- [ ] RTL layout check (for Urdu, Kashmiri, Sindhi)
- [ ] 6th-grade reading level verification
- [ ] Text length accommodation (no truncation)
- [ ] Consent withdrawal text equally understandable as grant

### 7.3 Language Coverage Targets

| Phase | Languages | Deadline |
|-------|-----------|----------|
| MVP | Hindi, Tamil, Telugu, Kannada, Bengali, Marathi, Gujarati | Nov 2026 (CM registration) |
| v1.1 | + Punjabi, Malayalam, Odia, Urdu, Assamese | Jan 2027 |
| v1.2 | + Konkani, Kashmiri, Maithili, Nepali, Sindhi | Mar 2027 |
| v1.3 | + Bodo, Dogri, Manipuri, Santali, Sanskrit | May 2027 (full enforcement) |

---

## Appendix A: Language-Specific Encoding Notes

### Bengali vs Assamese
Despite using visually similar scripts, Bengali and Assamese have different Unicode characters. Use `as` locale for Assamese — never use Bengali font/locale for Assamese text. Key distinction: ক্ষ in Bengali ≠ ক্ষ in Assamese.

### Kashmiri Script Selection
Kashmiri is written in two scripts:
- **Nastaliq (Perso-Arabic)** — official, used by majority, RTL
- **Devanagari** — used in Jammu region, LTR

Widget must detect/ask which script variant. Default to Nastaliq (`ks-Arab`).

### Sindhi Script Selection
Similarly, Sindhi in India uses:
- **Perso-Arabic** — Sindhi diaspora, RTL
- **Devanagari** — Kutch region variant, LTR

Default to Perso-Arabic (`sd-Arab`) for Gujarat/Rajasthan users.

### Santali (Ol Chiki)
Ol Chiki was created specifically for Santali in 1925. Unicode block U+1C50–U+1C7F. Font: Noto Sans Ol Chiki. Some older Santali users may use Devanagari or Bengali script for Santali — widget should allow this fallback.

### Sanskrit
Sanskrit in SahmatOS is used for formal/legal context only (not primary UI). Devanagari script. ISO code: `sa`.

---

## Appendix B: Browser and Device Support

| Environment | Indic Script Support | Notes |
|-------------|---------------------|-------|
| Chrome 90+ (Android) | Full | Most common Tier 2-3 browser |
| Safari 14+ (iOS) | Full | System fonts include most Indic scripts |
| Firefox 88+ | Full | |
| Samsung Internet 14+ | Full | Common on Samsung devices (large market share) |
| UC Browser | Limited | Avoid — poor Indic rendering; detect and warn |
| Android 8+ (WebView) | Full | |
| Android 5-7 (WebView) | Partial | May lack Meitei Mayek, Ol Chiki — fallback to Devanagari transliteration |
| Feature phones (KaiOS) | Basic | Hindi only via SMS fallback |
