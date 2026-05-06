# OpenMontage — Prompt Yazma Yeteneği & LLM Entegrasyonu

> Araştırma notu — `/home/user/OpenMontage` repo'su üzerinde yapılan derinlemesine inceleme.
> Soru: "Bu repodan tam otomatik olmasa bile yarı-otomatik bir şekilde AI video yapabilmek için prompt yazabilecek bir kabiliyet var mı? Prompt yazma şeklini ve LLM entegrasyonunu inceler misin?"

---

## TL;DR

**Evet** — OpenMontage tam olarak yarı-otomatik AI video üretimi için tasarlanmış. Üç ana özellik:

- **Çok katmanlı prompt yazma altyapısı** (skill-tabanlı LLM yönlendirmesi + deterministik Python builder + provider'a-özel rehberler)
- **Pipeline + checkpoint mimarisi** ile insan onay kapıları (creative aşamalarda HUMAN, teknik aşamalarda AUTO)
- **Sıfır gömülü LLM API çağrısı** — agent'ın kendisi (Claude/GPT) LLM'dir; Python sadece araç + persistence katmanı

Bu mimaride "tam otomatik" yapmak da mümkün ama **tasarım amacı yarı-otomatik**: insan creative kararlarda devrede, sistem teknik üretimi sürüyor.

---

## 1. Prompt Yazma Kabiliyeti — Üç Katmanlı Mimari

OpenMontage prompt yazmayı tek bir yerde yapmıyor; **üç katmanda** üretiyor:

### Katman A — Universal Prompt İskelet (Agent-Driven)

**Dosya:** `skills/creative/video-gen-prompting.md`

Tüm video gen modelleri için kanonik 5-aspect skeleton (line 42-51):

```
[Subject]        tip + ayırıcı görsel özellikler
[Subject Motion] zaman sıralı eylemler, etkileşimler
[Scene]          POV + setting + günün zamanı + dinamikler
[Spatial]        shot size + frame pozisyonu + derinlik (FG/MG/BG)
[Camera]         playback hızı → lens → yükseklik → açı → focus → hareket
```

Her modelin **prompt uzunluk sweet spot**'u tablolaştırılmış (line 59-65):

| Model | Sweet Spot |
|---|---|
| Seedance 2.0 | 200–400 kelime hero shot, 80–150 insert |
| Wan 2.2 | 200–400 kelime |
| Sora 2 / VEO 3.1 | 100–250 kelime |
| LTX-2 | ≤80 kelime |
| Runway Gen-4 | ≤60 kelime |

Camera shot tipleri ve hareket primitives tabloları (line 73-99). Bu agent'ın okuyup uyguladığı "prompt grameri".

### Katman B — Provider'a Özel Prompt Bilgisi (Agent-Driven, Layer 3)

**Dizin:** `.agents/skills/<provider>/SKILL.md`

Kayıtlı provider skill'leri:

- `seedance-2-0/` — preferred premium default; identity-lock, beat-by-beat choreography, lip-sync için `Character says: "..."` quoting
- `ai-video-gen/` — multi-gateway router (HeyGen + fal.ai)
- `ltx2/` — strict 80-word format, layered dimensions
- `flux-best-practices/`, `bfl-api/` — image gen
- Ek: `heygen`, `create-video`, `avatar-video`, `faceswap`, `video-translate`, `manim-composer`, `manimce-best-practices`, `gsap-*`, `framer-motion`, `lottie-bodymovin`

Bu skill'ler **her provider'ın kendine özgü prompt vocabulary'sini** öğretiyor. AGENT_GUIDE.md (line 629-631):

> "The difference between a generic prompt and a skill-informed prompt is the difference between 'usable' and 'cinematic.'"

### Katman C — Deterministik Python Prompt Builder (Automatic)

**Dosya:** `lib/shot_prompt_builder.py:82-144`

`build_shot_prompt(scene, style_context)` fonksiyonu, structured `scene_plan` JSON'ından doğal dil prompt üretir. 5 layer:

```python
Layer 1: Camera     — lens_mm + depth_of_field
Layer 2: Movement   — shot_size + camera_movement
Layer 3: Subject    — description + texture_keywords
Layer 4: Lighting   — lighting_key + color_temperature
Layer 5: Style      — playbook'tan adapt (verbatim prefix DEĞİL)
```

Enum→cümle eşlemeleri kod sabit:

```python
"dolly_in"     → "slow dolly in toward subject"
"shallow"      → "shallow depth of field with bokeh"
"low_key"      → "dramatic low-key lighting with deep shadows"
"medium_wide"  → "medium-wide shot framing subject with surroundings"
```

(line 33-79). **Bu sayede prompt yapılandırması deterministik kalıyor**; agent sadece `description` + `texture_keywords` kısmına narrative katkı yapıyor.

### Scene Plan Şeması — Prompt'un Girdisi

**Dosya:** `schemas/artifacts/scene_plan.schema.json:31-62`

Her sahne şu alanları taşır → builder bunları okur:

- `shot_language` (enum'lar: `shot_size`, `camera_movement`, `lens_mm`, `lighting_key`, `depth_of_field`, `color_temperature`)
- `description`, `texture_keywords`, `shot_intent`, `narrative_role`
- `character_actions` (animation için emotion + action_sequence)
- `required_assets` (`video_selector` / `image_selector` / `diagram_gen`)

### Stage Director'ler — Prompt Self-Review Loop

Asset director her sahne için agent'a 3 adımlı CHAI prompt review yaptırır:

**`skills/pipelines/explainer/asset-director.md:232-263`**:

1. **Pre-caption** — prompt'u draft et
2. **Critique** — 5-aspect checklist'e karşı puanla
3. **Post-caption** — boşlukları doldurarak yeniden yaz; "duygusal sıfatları görsel sebeplerle değiştir"

Sonra `image_selector` veya `video_selector` çağrılır.

Diğer önemli director'lar:

- `skills/pipelines/cinematic/scene-director.md`
- `skills/pipelines/animation/scene-director.md:41-97`
- `skills/pipelines/<pipeline>/asset-director.md` her pipeline için

---

## 2. LLM Entegrasyonu — "Agent IS the LLM" Modeli

**Bulgu:** Python kodunda **sıfır LLM API çağrısı** var. `anthropic`, `openai`, `litellm`, `instructor` gibi hiçbir LLM SDK import edilmiyor.

### Kanıtlar

- `lib/config_model.py:28-32` — `LLMConfig` sınıfı `provider: "anthropic"` default'la tanımlı, ama bu sadece **konfigürasyon konteyneri**; gerçek çağrı yok
- `lib/checkpoint.py`, `lib/pipeline_loader.py`, `tools/tool_registry.py` — saf data plumbing; LLM yok
- `AGENT_GUIDE.md:71-81`'de açık tanım:

> "OpenMontage is an instruction-driven video production system. The AI agent IS the intelligence... **Python = tools + persistence.** No orchestration logic, creative decisions, review logic, or checkpoint policy in Python code."

### Mimari

```
┌─────────────────────────────────────────────────────────────┐
│ Claude/GPT (kullanıcının agent'ı — Claude Code, Cursor vb)  │
│   ↓ okur                                                     │
│ skills/*.md + pipeline_defs/*.yaml (orchestration intel)    │
│   ↓ çağırır                                                  │
│ tools/*.py (BaseTool subclasses — provider API'larını sarar)│
│   ↓ persist eder                                             │
│ projects/<name>/artifacts + checkpoints (resume için)       │
│   ↓ insan onayı gerektiğinde                                 │
│ Pause → user input → resume                                  │
└─────────────────────────────────────────────────────────────┘
```

**Yani:** Bu repo "LLM entegrasyonu olan bir Python kütüphanesi" değil; **"LLM agent'ları için yazılmış bir araç + talimat seti"**. Sen Claude'u (veya GPT'yi) bir CLI ile çalıştırıyorsun, Claude bu repo'yu okuyup kendini yönetiyor, Python araçlarını çağırıyor.

### Agent-driven Self-Review Döngüsü

**`skills/meta/reviewer.md:68-76`** — agent kendi ürettiği artifact'ı 2 tur revize edebilir, sonra "pass with warnings" der. Bu da Python'da değil, agent'ın markdown'a göre kendisini denetleme döngüsü.

### Reference Workflow (Inspirational)

**`skills/meta/video-reference-analyst.md:28-89`**:

> "Run VideoAnalyzer with `analysis_depth: 'standard'`... Present a summary to the user. This is NOT a raw dump. It's a conversational interpretation..."

Kullanıcı bir referans video gönderirse (URL veya yerel), agent bu skill'i okuyup analiz yürütür, 2-3 farklı concept önerir.

---

## 3. Yarı-Otomatik İş Akışı — Checkpoint Kapıları

**Pipeline manifesti örneği:** `pipeline_defs/cinematic.yaml:59-267`

```
research      (auto)            ← internet search, source review
proposal      HUMAN APPROVAL    ← 4-5 concept varyasyonu sun, kullanıcı seçsin
script        HUMAN APPROVAL    ← script'i göster, onay al
scene_plan    HUMAN APPROVAL    ← her sahne için shot_language, kullanıcı onaylasın
assets        (auto)            ← prompt yaz → video/image gen → audio gen
edit          (auto)            ← edit_decisions üret
compose       (auto)            ← Remotion / HyperFrames / FFmpeg ile render
publish       HUMAN APPROVAL    ← final video onayı
```

### Checkpoint Mantığı

**`lib/checkpoint.py:194-272, 328-341`**:

- Her stage tamamlandığında `pipelines/<project_id>/checkpoint_<stage>.json` yazılır
- `get_next_stage()` "kaldığın yerden devam" sağlar
- Compose'da fail olursa baştan değil, compose'dan resume

### Cost Gating

**`tools/cost_tracker.py:117-150`**:

- Her aksiyon öncesi cost estimate gösterilir
- Default eşik: $0.50/aksiyon, $10/toplam bütçe
- Approval threshold aşılırsa insan onayı zorunlu

### Onboarding

**`skills/meta/onboarding.md`** — kullanıcının makinesindeki tool envanterini keşfeder, "tier" çıkarır:

- `zero-key` — sadece ücretsiz/yerel araçlar
- `starter` — 1-2 API key
- `standard` — yaygın provider'lar
- `full` — tüm cloud provider'lar
- `full+GPU` — yerel GPU modelleri dahil

O tier'a uygun 3 starter prompt sunar.

### Available Pipelines

| Pipeline | Best For | Stability |
|----------|----------|-----------|
| `animated-explainer` | Topic to fully generated explainer | production |
| `talking-head` | Footage-led speaker videos | beta |
| `screen-demo` | Screen recordings and walkthroughs | production |
| `clip-factory` | Many clips from one long source | beta |
| `podcast-repurpose` | Podcast highlights and derivatives | beta |
| `cinematic` | Trailer, teaser, mood-led edits | production |
| `animation` | Motion-graphics and animation-first | production |
| `character-animation` | Local rigged cartoon characters | beta |
| `hybrid` | Source footage plus support visuals | production |
| `avatar-spokesperson` | Presenter-led avatar/lip-sync | production |
| `localization-dub` | Subtitle, dub, translated variants | beta |

### Konfigüre Edilebilir Video Gen Tool'lar

`tools/video/` altında 30+ video tool kayıtlı:

- **Cloud APIs** (paid): Kling, Google Veo, Runway, MiniMax, HeyGen, Higgsfield, Grok, Seedance 2.0, Sora 2
- **Local GPU** (free, GPU gerekli): WAN 2.1, Hunyuan, CogVideo, LTX-Video
- **Stock sources** (free): Pexels, Pixabay, Wikimedia Commons, Archive.org, Unsplash

`video_selector` 7 boyutta puanlama yapar (task fit, quality, control, reliability, cost, latency, continuity) ve mevcut olanlardan en iyiyi seçer.

---

## 4. Senin İçin Pratik Sonuç

### Yarı-otomatik AI video üretimi için kullanmak istiyorsan

1. **Doğru pipeline'ı seç:**
   - `cinematic` — trailer, teaser, mood-led (production stable)
   - `animation` — motion-graphics ağırlıklı (production stable)
   - `animated-explainer` — topic-to-explainer (production stable)
   - `clip-factory` — uzun kaynaktan çoklu kısa klip (beta)

2. **AI video gen API key'leri ayarla** (tier'ını yükseltmek için):
   - Cloud (paid): `FAL_KEY` (Seedance, Kling, LTX), `GOOGLE_API_KEY` (Veo), `RUNWAY_API_KEY`, `HEYGEN_API_KEY`, `OPENAI_API_KEY` (Sora 2 erişimi varsa)
   - Local GPU (free): WAN 2.1, Hunyuan, CogVideo, LTX-Video — GPU varsa
   - Stock (free): Pexels, Pixabay (zero-key tier'da bile çalışır)

3. **Bir Claude Code / Cursor session'ı aç**, repo dizinine gel, doğal dilde brief ver:

   > "60 saniyelik sinematik bir trailer yap: konu — 'denizaltı yapay zekanın evrimi'"

4. **Agent şunları yapar:**
   - Pipeline'ı seçer (cinematic)
   - Preflight: hangi tool'ların available olduğunu listeler
   - 4-5 concept önerir → **sen onay verirsin** (Stage 1 gate)
   - Script yazar → **sen onay verirsin** (Stage 2 gate)
   - Scene plan üretir (her sahne için 5-aspect prompt) → **sen onay verirsin** (Stage 3 gate)
   - Asset'leri otomatik üretir (prompt → video/image/TTS gen)
   - Compose ve render eder
   - Final'i sana sunar (Stage 4 gate)

### Daha fazla otomasyon istersen

- `human_approval_default: true` olan stage'leri YAML'da `false` yapabilirsin (`cinematic.yaml` line 89, 127, 151)
- Ama uyarı: skill'ler tarafından tasarlanan creative review insan içindir; tamamen kapatırsan kaliteyi agent'ın self-review'una emanet edersin

### Sadece prompt yazma yeteneğini kendi sisteminde kullanmak istersen

Üç dosya yeterli — taşınabilir bir paket oluştururlar:

- **`lib/shot_prompt_builder.py`** — saf Python, dependency yok (78-160 satırı)
- **`schemas/artifacts/scene_plan.schema.json`** — JSON schema; structured input kontratı
- **`skills/creative/video-gen-prompting.md`** + **`.agents/skills/seedance-2-0/SKILL.md`** — kendi LLM agent'ına system prompt olarak verilebilir

Akış:

```
Kullanıcı brief    →  LLM scene_plan üretir    →  shot_prompt_builder
                       (schema'ya göre)            (deterministic)
                                                          ↓
                                                  doğal dil prompt
                                                          ↓
                                                 video gen API çağrısı
```

---

## 5. Kritik Dosya Listesi

| Amaç | Dosya |
|---|---|
| Universal prompt grameri | `skills/creative/video-gen-prompting.md` |
| Provider-özel prompt knowledge | `.agents/skills/seedance-2-0/SKILL.md`, `.agents/skills/ltx2/SKILL.md`, `.agents/skills/ai-video-gen/SKILL.md` |
| Deterministik prompt builder | `lib/shot_prompt_builder.py:82-144` |
| Scene plan kontratı | `schemas/artifacts/scene_plan.schema.json` |
| Pipeline tanımları | `pipeline_defs/cinematic.yaml`, `animation.yaml`, `animated-explainer.yaml` |
| Asset director (prompt-to-call) | `skills/pipelines/<pipeline>/asset-director.md` |
| Self-review meta | `skills/meta/reviewer.md` |
| Reference video analyst | `skills/meta/video-reference-analyst.md` |
| Onboarding | `skills/meta/onboarding.md` |
| Checkpoint motoru | `lib/checkpoint.py` |
| Tool kayıt sistemi | `tools/tool_registry.py` |
| Cost gating | `tools/cost_tracker.py` |
| Agent kontratı | `AGENT_GUIDE.md`, `CLAUDE.md` |

---

## 6. Doğrulama Komutları

Bu analizi kendi terminalinde teyit etmek için:

```bash
# 1. Tool envanterini gör
cd /home/user/OpenMontage
python -c "from tools.tool_registry import registry; import json; registry.discover(); print(json.dumps(registry.provider_menu_summary(), indent=2))"

# 2. Hangi video gen tool'lar kayıtlı?
python -c "
from tools.tool_registry import registry
registry.discover()
for t in registry.get_by_capability('video_generation'):
    info = t.get_info()
    print(f\"{t.name:30} runtime={info.get('runtime'):10} status={info.get('status')}\")
"

# 3. Prompt builder'ı çalıştır
python -c "
from lib.shot_prompt_builder import build_shot_prompt
scene = {
    'description': 'A submarine explorer descends into bioluminescent abyss',
    'texture_keywords': ['volumetric haze', 'cyan glow', 'anamorphic'],
    'shot_language': {
        'shot_size': 'medium_wide',
        'camera_movement': 'dolly_in',
        'lens_mm': 35,
        'lighting_key': 'low_key',
        'color_temperature': 'cool',
        'depth_of_field': 'shallow',
    }
}
print(build_shot_prompt(scene, {'mood': 'mysterious cinematic'}))
"

# 4. Cinematic pipeline'ı incele
grep -n 'human_approval_default\|stages:\|skill:' pipeline_defs/cinematic.yaml | head -30

# 5. Bir gerçek session başlat (Claude Code / Cursor / Copilot içinde):
#    "Make me a 30-second cinematic teaser about deep-sea AI"
#    → agent preflight çalıştırır, concept'ler sunar, onay bekler
```

### Beklenen Çıktılar

- **#2** → 13+ video gen tool, status'larıyla beraber
- **#3** → `35mm lens, shallow depth of field with bokeh. medium-wide shot framing subject with surroundings, slow dolly in toward subject. A submarine explorer descends into bioluminescent abyss. volumetric haze, cyan glow, anamorphic. dramatic low-key lighting with deep shadows, cool blue-toned color palette. Style: mysterious cinematic`
- **#4** → `human_approval_default: true` olan creative stage'ler (proposal, script, scene_plan, publish) ve `false` olan teknik stage'ler (research, assets, edit, compose) görülmeli

---

## 7. Önemli Uyarılar

- Bu repo bir "library" değil — bir **agent operating manual**. Eğer Claude Code / Cursor / Copilot gibi bir LLM agent ortamından çalıştırmıyorsan, içindeki "intelligence" çoğunlukla devre dışı kalır (Python araçları çalışır ama orchestration kaybolur). Yarı-otomatik akışı görmek için **mutlaka bir LLM agent ile birlikte** kullan.

- **Beta pipelines** (`talking-head`, `clip-factory`, `podcast-repurpose`, `character-animation`, `localization-dub`) tam denetlenmemiş — production'da kullanılabilir ama sürtünme bekle.

- `render_runtime` kararı (Remotion vs HyperFrames vs FFmpeg) **proposal aşamasında kilitlenir**; agent silently swap yapamaz. Bu, motion-bağımlı briefler için kritik bir governance kuralı.

- Cost tracker default $10 toplam bütçe ile gelir; büyük projeler için `cost_tracker.py`'de ayarla.

---

## 8. Bu Sessiondan Notlar

### Kullanılan Araştırma Yöntemi

3 paralel Explore agent'ı kullanıldı:

1. **Prompt-yazma altyapısı** — skill dizinleri, prompt templateleri, stage director'lar, tool input şemaları
2. **LLM entegrasyon mimarisi** — Python'daki LLM API çağrıları (yok!), skill yükleme, agent kontratı
3. **Yarı-otomatik iş akışı** — entry points, checkpoint protokolü, pipeline manifestleri, cost tracking

Sonuçlar `lib/shot_prompt_builder.py`, `skills/creative/video-gen-prompting.md` ve `pipeline_defs/cinematic.yaml` üzerinden manuel olarak doğrulandı.

### Sonraki Adımlar (öneriler)

- **A) Bir pipeline'ı çalıştırmak** — Claude Code ile burada bir brief verip yarı-otomatik akışı sonuna kadar görmek
- **B) Prompt builder'ı izole etmek** — `lib/shot_prompt_builder.py` + `schemas/artifacts/scene_plan.schema.json` kendi sistemine taşımak
- **C) Daha derin inceleme** — belirli bir pipeline (ör. cinematic) veya provider skill'ini (ör. seedance-2-0) detaylı analiz

---

*Hazırlık: OpenMontage repo'sunda yapılan multi-agent araştırma, 2026-05-06.*
