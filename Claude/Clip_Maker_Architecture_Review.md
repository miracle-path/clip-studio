# Clip Maker — Architecture Review & Data Contract v2

| รายการ | ค่า |
|---|---|
| ไฟล์ที่ตรวจ | `Clip_Maker_Enhanced.html` (130,657 bytes, 881 บรรทัด; `<script>` ≈ 124 KB, ส่วน HTML/CSS ≈ 86 บรรทัด) |
| บทบาท | Software Architect |
| วันที่ | 31 ต.ค. 2026 |
| ข้อจำกัดของโปรเจกต์ | HTML ไฟล์เดียว, Vanilla JS, Canvas 2D / SVG / WebGL เท่าที่เบราว์เซอร์มี, **ไม่มี Framework/Dependency ภายนอก, ไม่เขียนใหม่ทั้งระบบ** |

**ป้ายหลักฐานที่ใช้ในรายงาน**

- **[R]** อ่านจากโค้ดโดยตรง (ระบุเลขบรรทัดของไฟล์ที่แนบ)
- **[V]** ตรวจสอบโดยรันโค้ดที่ตัดออกมาจากไฟล์ใน Node.js (ทำซ้ำได้)
- **[S]** อ้างอิงพฤติกรรมตามสเปก Web API แต่ยังไม่ได้รันในเบราว์เซอร์

**ข้อจำกัดของการตรวจ:** ไม่ได้เปิดแอปในเบราว์เซอร์จริง จึง **ไม่มีตัวเลข performance ที่วัดได้** ข้อสรุปด้าน performance เป็นการประเมินจากโค้ด และ Acceptance Criteria จึงให้ "วัด baseline ก่อน" แทนการกำหนดตัวเลขลอยๆ

---

## 0. สรุปผู้บริหาร

**Architecture ปัจจุบัน** คือ pipeline ขนาดเล็กที่ทำงานได้จริงและมีแนวคิดดี: `JSON ซีน → normalize → คำนวณเวลาต่อซีน → frame(time) วาดด้วย Canvas 2D แบบ immediate-mode → rAF / MediaRecorder` ทั้งหมดอยู่ใน `<script>` เดียว 61 ฟังก์ชัน และตัวแปร global 22 ตัว

**จุดแข็งที่ต้องรักษาไว้**

1. `frame(time)` เป็นฟังก์ชันของเวลา ไม่ใช่ state machine ทำให้ scrub/seek/export แบบกำหนดเวลาเองทำได้ในหลักการ
2. คำศัพท์ภาพและพิกัดเป็นสัดส่วน 0–1 ทำให้ AI สร้าง JSON ได้ง่ายและย่อ/ขยายได้
3. ตัวละครทั้งหมดวาดด้วยโค้ด (procedural) ไม่ต้องมีไฟล์ภาพ
4. มี alias layer (`normalizeSceneData`) รับ JSON หลายสำเนียงจาก AI

**ความเสี่ยงสำคัญ 5 ข้อ (รายละเอียดในหัวข้อ 3)**

| # | ความเสี่ยง | หลักฐาน |
|---|---|---|
| R-01 | **บั๊กฟังก์ชันจริง:** ซีนที่ไม่ใส่ `theme`/`sky`/`atm` ไม่ได้ "สืบจากช็อตก่อน" ตามที่พรอมต์สัญญา เพราะ `normalize` เติมค่า default ก่อนที่ `build()` จะสืบทอด ตัวอย่างในพรอมต์เองก็โดนผลกระทบ | [V] |
| R-02 | ไม่มี fault isolation: JSON บางรูปแบบทำให้ `build()` throw เงียบๆ และค่าสี/ตัวเลขที่ผิดทำให้ `loop` หยุดกลางคลิป | [V] [S] |
| R-03 | `frame(t)` **ไม่ pure จริง**: พึ่งพา global ที่ค้างจากเฟรมก่อน (`DIRS`, `POSG`) และ `Math.random` ทำให้ seek, golden test และ offline export ให้ผลไม่แน่นอน | [R] |
| R-04 | Export เป็น realtime ผ่าน `MediaRecorder` ผูกกับ rAF (แท็บถูกพักแล้วเฟรมหาย) และไม่มีเสียง | [R] |
| R-05 | Schema มี "แหล่งความจริง" 3 แห่งที่ไม่ผูกกัน: ข้อความพรอมต์, `normalize`, และ painter ทำให้ค่าที่ไม่รู้จักถูกกลืนเงียบๆ | [R] |

**ข้อเสนอหลัก:** ไม่เขียนใหม่ แต่เพิ่ม **"compile pipeline" 4 ขั้น** คั่นกลางระหว่าง JSON v1 (ซึ่งคงเป็นรูปแบบที่ AI/ผู้ใช้เขียน) กับ painter เดิม:

```
v1 JSON ──parse──► normalize ──► validate ──► compile ──► ProjectV2 (immutable)
                                                              │
                                  evaluate(project, t) ──► FrameState (pure data)
                                                              │
                                      Renderer.render(FrameState) ──► Canvas 2D (painter เดิม)
```

- ตัววาดตัวละคร/ฉาก/เอฟเฟกต์ (`fig`, `robotActor`, `road`, `props` ฯลฯ) **ไม่ต้องเขียนใหม่** แค่ห่อด้วย registry
- v2 เป็น "ภาษากลางภายใน" ที่ผ่านการตรวจแล้ว ไม่ต้องแก้พรอมต์ AI ใน Phase แรก
- ทำเป็น 7 phase เล็กๆ ที่แต่ละ phase แอปยังรันได้ Phase 0–1 แก้ R-01/R-02 ได้โดยไม่แตะโครงสร้าง

---

## 1. สถาปัตยกรรมปัจจุบัน

### 1.1 แผนที่ไฟล์ (ตามเลขบรรทัด)

| บรรทัด | ส่วน | หน้าที่ | หมายเหตุ |
|---|---|---|---|
| 1–86 | HTML + CSS | UI 3 กล่อง (ไอเดีย / JSON / ตัวเลือก) + canvas 720×1280 | ฟอนต์ Kanit โหลดจาก Google Fonts |
| 88–97 | Constants + `SAMPLE` | `W,H,$,cv,g`, JSON ตัวอย่าง | `g` (ctx) เป็น global ที่ทุกฟังก์ชันใช้ตรงๆ |
| 98–101 | Helpers | `lerp, clamp, rr, glow, sky` | `rr` เรียก `g.roundRect` ตรงๆ |
| 103–148 | Themes | `road room city nature space` | วาดฉากแบบ hard-code |
| 149–153 | `customOld`, `seg` | โค้ดเก่า / helper แขนขา | `customOld` ไม่ถูกเรียก |
| 154–216 | `PAL`, `cat4`, `critter` | แมว/สุนัข, ปลา/นก/blob | แต่ละตัวมี animation inline |
| 218–263 | `robotActor`, `beastActor` | หุ่นยนต์, สัตว์ 4 ขา/มอนสเตอร์ | เก็บ anchor ลง `ANCH` |
| 264–316 | `actors()` | **ตัวกลางวาดตัวละคร**: merge cast, keys, หันหน้า, dispatch ตามชนิด, state machine ของ `act` ทั้งหมด | ฟังก์ชันเดียวทำ 5 หน้าที่ |
| 317–354 | `actorsEmoji`, `person`, `cat`, `drawChar` | ระบบตัวละครรุ่นเก่า | **ไม่ถูกเรียก** |
| 355–372 | `badge`, `subtitle` | overlay | `badge` ถูกใช้เฉพาะใน `frameOld` |
| 373–416 | `normalizeSceneData`, `build` | alias, inference, คำนวณเวลา, สืบทอดข้ามซีน | **ต้นเหตุ R-01, R-02** |
| 417–422 | `ANCH/CAMI/DIRS/IMG`, `wpt`, `ez`, `kf` | global state + keyframe interpolation | |
| 423–437 | `lake`, `mapbg` | ฉากทะเลสาบ, แผนที่ | |
| 438–507 | `props()` | prop + VFX รวมกันใน if-chain 15 แบบ (`rod rocket img pin route dots ring shape beam aura blast songthaew ball spark`) | VFX ปนกับ prop |
| 508–524 | `atmos`, `shadowAt`, `lights` | บรรยากาศ/แสง/เงา | ใช้ `Math.random` |
| 525–538 | `photobg`, `PRE`, `stylize` | ฉากจากรูปถ่าย | ไม่มีที่ใดเติม `PH` หรือโหลดภาพ `photo:*` |
| 539–598 | `hsh`, `tl`, `fig`, `shophouse`, `yamo` | ตัวละครมนุษย์/ซอมบี้ละเอียด, ฉากเฉพาะ | `fig` ≈ 30 บรรทัดยาว |
| 599–625 | `GEO`, `proj`, `geoLayers`, `kang` | แผนที่พิกัดจริง | |
| 626–689 | `ring`, `decor`, `custom`, `svgActor`, `crowdOf` | ฉากเวที, ชิ้นส่วนประกอบฉาก, ตัวละคร SVG, ฝูงชน | |
| 690–707 | `gfx` | การ์ด/ป้ายชื่อ | |
| 708–722 | **`frame(time)`** | กล้อง → ฉาก → prop หลัง → ตัวละคร → prop หน้า → แสง → บรรยากาศ → gfx → ซับ → ทรานซิชัน | อ่าน DOM (`$('oSub')`) |
| 723–731 | `frameOld` | เฟรมรุ่นเก่า | **ไม่ถูกเรียก** อ้าง `$('oBadge')` ที่ไม่มีใน DOM |
| 732–761 | `speak`, `loop`, `play`, `stop`, `record` | เสียง, เวลา, ส่งออก | rAF + MediaRecorder |
| 762–863 | `storyPrompt`, `reviewPrompt`, `promptText`, `promptText2`, `promptTextOld` | พรอมต์ AI (~100 บรรทัดข้อความ) | **เป็นเอกสาร schema ที่ไม่ผูกกับโค้ด**; `promptText2/Old` ไม่ถูกเรียก |
| 864–878 | Event wiring | ปุ่ม, `claude.use('sample')`, เสียง, init | |

### 1.2 Data flow

```
textarea#json ──► build() ──► JSON.parse(regex-fixed text)
                              └► normalizeSceneData(j)   (mutate ซีนในที่เดิม)
                              └► คำนวณ s.d / s.s ต่อซีน จาก w (น้ำหนัก)
                              └► สืบทอด actors/props/crowd/decor/geo/text/badge/gfx/atm/lights
                              └► เขียน field ที่ derive ได้กลับลง object ซีน (s.tx, s.am, s.lt, s.gx ...)
rAF loop ──► frame(time) ──► ตั้ง global (ANCH, CAMI, LT, GEO, POSG, DIRS) ──► THEMES[theme] → actors → props ...
                          └► speak() เมื่อ text เปลี่ยน (TTS ของเบราว์เซอร์)
record() ──► cv.captureStream(30) → MediaRecorder → ดาวน์โหลด (ไม่มีเสียง)
```

### 1.3 Global state ที่เป็น "ช่องทางลับ" ระหว่างฟังก์ชัน [R]

| ตัวแปร | ผู้เขียน | ผู้อ่าน | ปัญหา |
|---|---|---|---|
| `ANCH` | `frame` รีเซ็ตเป็น `{}`; painter ตัวละครเขียนระหว่างวาด | `props` (rod/shape/beam) | prop ที่วาด **ก่อน** ตัวละคร (`back:true`) จะไม่เห็น anchor เลย |
| `POSG` | `actors()` | `frame` (camera `follow`), `props` (aura), `P0` | กล้องตามตำแหน่งจากเฟรม/ซีน **ก่อนหน้า** |
| `DIRS` | `actors()`, `svgActor` | `kf`, `props` (rod) | ทิศหันสืบทอดจากประวัติเฟรม → seek แล้วผลต่างกัน |
| `CAMI` | `frame` | `wpt` | ต้องเรียกตามลำดับเดิมเท่านั้น |
| `LT`, `GEO` | `frame` | `shadowAt`, `proj`/`geoLayers` | ซีนไหนก็อ่านได้ถ้าลืมตั้ง |
| `CAST` | `build` | `actors` | |
| `IMG` | `build` | `svgActor`, `props(img)`, `photobg` | ไม่มีการ evict, ไม่รอ decode |
| `scenes,total,playing,t0,cur,rec,chunks,lastTx` | `build/play/loop/record` | ทั่วไป | state ของ player ปนกับ state ของ renderer |
| `PH` | — | `promptText` | ไม่มีโค้ดที่เติมค่า (ฟีเจอร์ภาพถ่ายค้างครึ่งทาง) |

---

## 2. จุดเสี่ยงและข้อบกพร่อง

ระดับความรุนแรง: **P0** กระทบผลลัพธ์/เสถียรภาพตอนนี้, **P1** ขวางการขยายและการทดสอบ, **P2** ควรจัดการเมื่อสะดวก

### P0

**R-01 การสืบทอดค่าข้ามซีนถูกทำลายโดย normalize [V]**

- พรอมต์ (บรรทัด 802, 820) สัญญาว่า "สิ่งที่ไม่ใส่จะสืบจากช็อตก่อน: theme sky ground sign" และ `atm` "สืบเนื่องข้ามช็อต" ตัวอย่างในพรอมต์เอง (บรรทัด 824) มีซีนที่ไม่ใส่ `theme`
- แต่ `normalizeSceneData` (บรรทัด 374–407) กำหนด `sc.theme = ... || inferTheme(sc)` (คืน `'custom'` เสมอ), `sc.sky`, `sc.ground`, `sc.atm` ให้ทุกซีน **ก่อน** `build()` ตรวจ `!s.theme && pv` ดังนั้นเงื่อนไขสืบทอดใน `build()` ไม่มีวันเป็นจริงสำหรับ `theme/sky/ground/decor/hz/sign`
- ผลที่ทดสอบ: ซีน 1 = `lake` + `atm.mood:'danger'`; ซีน 2 ไม่ใส่อะไรเลยนอกจาก `cam` → ได้ `theme:'custom'`, sky สี default, `atm.mood:'calm'`
- ผลกระทบ: ภาพกระโดดฉาก/อารมณ์กลางเรื่อง และ AI ที่ทำตามพรอมต์ได้ผลลัพธ์ผิด
- แก้: Phase 1 (แยกขั้น "authored → resolved" ให้ default เกิด **หลัง** การสืบทอด)

**R-02 ไม่มีการกันความผิดพลาด (fault isolation)**

- `normalizeSceneData` อยู่นอก `try` ใน `build()` ทดสอบแล้ว throw จาก `{"scenes":{...}}`, `null`, และ `{"scenes":[null]}` [V] เกิดใน click handler → ผู้ใช้เห็นเหมือนกดแล้วไม่เกิดอะไร
- `loop` ไม่มี try/catch (บรรทัด 737–742) และเรียก rAF **หลัง** `frame()` ถ้า `frame()` throw ลูปจะหยุดเงียบๆ โดย `playing` ยังเป็น `true` ตามสเปก Canvas [S] `addColorStop` throw `SyntaxError` เมื่อสีไม่ถูกต้อง และ `createRadialGradient` throw เมื่อรัศมีติดลบหรือไม่ใช่ตัวเลขจำกัด ค่า `core`, `eye`, `col` ของ props/actors ไหลเข้า `glow()` โดยไม่ผ่านการตรวจ (มีเพียง `sky`/`ground`/`lights.col` ที่มี `okc`/regex)
- แก้: Phase 0 (ครอบ try/catch + ตัวรายงานข้อผิดพลาด), Phase 2 (validator)

**R-03 `frame(t)` ไม่ pure และไม่ deterministic [R]**

- `kf` ใช้ `DIRS[...]` จากเฟรมก่อน (บรรทัด 422), camera `follow` อ่าน `POSG` ของเฟรม/ซีนก่อนหน้า (บรรทัด 711), `ANCH` ถูกรีเซ็ตทุกเฟรมแล้ววาด prop ที่ `back:true` ก่อนมี anchor
- `Math.random` ใน `atmos` (grain) และ `props(beam)` (ประกายไฟฟ้า) ทำให้ผลต่างกันทุกครั้งแม้เวลาเดียวกัน
- ผลกระทบ: seek/scrub ไม่ตรงกับการเล่นต่อเนื่อง, ทำ golden test แบบ pixel ไม่ได้, offline export แบบเฟรมต่อเฟรมจะให้ภาพไม่ตรงกับพรีวิว
- แก้: Phase 3 (RenderContext แทน global) และ Phase 5 (seeded RNG, facing คำนวณจากเวลา)

**R-04 ระบบส่งออกเปราะบาง [R]**

- `record()` เล่นแบบ realtime ผ่าน rAF แล้วอัดด้วย `MediaRecorder` ถ้าแท็บถูกพัก/ลดความถี่ เฟรมหายและความยาวไม่ตรง; ข้อความในแอปเตือนให้ "อย่าสลับแท็บ" อยู่แล้ว
- เสียงพากย์ใช้ `speechSynthesis` ซึ่งเบราว์เซอร์ไม่ให้ต่อเข้า WebAudio/MediaRecorder จึงอัดเสียงไม่ได้ [S] (แอปบอกผู้ใช้ไว้แล้ว)
- ความยาวถูกบังคับ `total = Math.max(60, ...)` (บรรทัด 412) คลิปทดสอบสั้นๆ ทำไม่ได้ และสเกล `w` ของซีนให้ครบ 60 วินาทีเสมอ
- แก้: Phase 5 (export ที่ทนทานขึ้น + ทางเลือกกำหนดเวลาเอง) ดูข้อจำกัดจริงในหัวข้อ 5.10

### P1

**R-05 Schema มีแหล่งความจริง 3 แห่ง**: ข้อความพรอมต์ (บรรทัด 770–834), alias ใน `normalizeSceneData`, และเงื่อนไขใน painter เช่น `act` ที่ไม่รู้จักถูกวาดเป็นท่าเริ่มต้นเงียบๆ, `c` ที่ไม่รู้จักตกเป็น `human` (บรรทัด 270: `PAL[a.c]?a.c:'human'`) เมื่อเพิ่มชนิดตัวละครหรือท่าใหม่ ต้องแก้ 3 ที่ และไม่มีอะไรบอกเมื่อพลาด
→ แก้: Phase 2 (SCHEMA registry เดียว สร้างทั้ง validator และส่วน "คลังเครื่องมือ" ของพรอมต์)

**R-06 Renderer ผูกกับ DOM**: `frame()` อ่าน `$('oSub').checked`; `build()` อ่าน `$('json')` และ `$('oDur')`; `speak()` อ่าน `$('oVoice')`, `$('oRate')`, `$('oVoiceSel')` ทำให้รัน headless/เทสต์ไม่ได้ → แก้ Phase 3 (ส่ง options เข้า renderer)

**R-07 ข้อมูลที่ผู้ใช้เขียนปนกับข้อมูลที่คำนวณ**: `build()` เขียน `s.d, s.s, s.tx, s.hl, s.ts, s.bd, s.am, s.lt, s.gx, s.gts` ลงใน object ซีนเดียวกับที่ normalize แก้ค่าไปแล้ว (ชื่อฟิลด์สั้นชนกับของผู้ใช้ได้ เช่น `s.s` กับ `a.s`, และ `d`) ทำให้ debug ยากและทำ snapshot ไม่ได้ → แก้ Phase 2

**R-08 ฟังก์ชันก้อนใหญ่ที่ต้องแก้หลายที่เมื่อเพิ่มของใหม่**

- `actors()` (≈50 บรรทัด): merge cast, keyframe, หาเป้าหมายที่หัน, dispatch ตามชนิด และ state machine ของ `act` ราว 20 แบบอยู่ในฟังก์ชันเดียว
- `props()` (≈70 บรรทัด): if-chain 15 แบบ ที่รวม prop, VFX, และ overlay แผนที่ไว้ด้วยกัน
- เพิ่มชนิดตัวละคร/ท่า/VFX = แก้ if-chain + พรอมต์ + alias
→ แก้ Phase 4 (registry)

**R-09 ต้นทุนต่อเฟรมที่เลี่ยงได้ [R — ยังไม่ได้วัด]**

- `subtitle()` สร้าง `Intl.Segmenter` ใหม่ + `measureText` ทุกคำ **ทุกเฟรม** (บรรทัด 361–371) ทั้งที่ข้อความเปลี่ยนเฉพาะตอนเปลี่ยนซีน
- `crowdOf(sc.crowd)` สร้างอาเรย์ใหม่ทุกเฟรม (บรรทัด 714)
- `actors()` merge object ด้วย spread สองรอบต่อตัวละครต่อเฟรม; ธีม `road` สร้าง radial gradient ≈ 36 อันต่อเฟรม; `atmos` วาด grain 260 สี่เหลี่ยมต่อเฟรม
- `ctx.filter = blur()` (dof) ราคาแพงและบางเบราว์เซอร์ไม่รองรับ [S]
→ แก้ Phase 3–4 (cache ที่ระดับ compile/evaluate) โดย **วัด baseline ก่อน**

**R-10 Asset pipeline**: `build()` สร้าง `Image` แล้วเริ่มเล่นทันที ส่วน `svgActor`/`props(img)` ข้ามการวาดถ้า `!im.complete` → เฟรมแรกๆ (และเฟรมแรกๆ ของไฟล์ที่ส่งออก) อาจไม่มี asset; ไม่มีเพดานขนาด/จำนวน; `IMG` ไม่ถูกล้าง; เส้นทาง `photo:*`/`stylize` ไม่มีทางเข้า (dead end)

**R-11 ฟอนต์และเครือข่าย**: ลิงก์ Google Fonts เป็นทรัพยากรภายนอกเพียงรายการเดียว (ไม่ใช่ dependency โค้ด แต่ทำให้ "ไม่ต้องติดตั้ง/ออฟไลน์" ไม่จริง 100%) `document.fonts.load(...).then(...)` (บรรทัด 878) ไม่มี `.catch` ถ้าโหลดฟอนต์ล้มเหลว promise อาจ reject ตามสเปก [S] ทำให้เฟรมแรกไม่ถูกวาด และทุกจุดใช้ชื่อ `Kanit` ตรงๆ ไม่มี font stack กลาง

**R-12 ใช้ Web API ใหม่โดยไม่ตรวจความสามารถ**: `g.roundRect` (ถูกเรียกผ่าน `rr()` ทั่วทั้งไฟล์), `ctx.filter` (dof), `captureStream`, `Intl.Segmenter` (มี fallback ดีอยู่แล้ว), `MediaRecorder` (ตรวจแล้ว), `window.claude.use('sample')` (ตรวจแล้ว แต่เป็น API เฉพาะโฮสต์ ควรอยู่หลัง adapter)

### P2

| # | เรื่อง | รายละเอียด |
|---|---|---|
| R-13 | โค้ดตาย | `customOld, frameOld, promptText2, promptTextOld, actorsEmoji, person, cat, drawChar, badge, stylize, PRE, photobg` (ดูภาคผนวก B) ใช้พื้นที่และทำให้ผู้อ่านสับสนว่าเส้นทางไหนคือของจริง ลบได้เมื่อมี golden test |
| R-14 | regex ซ่อมตัวเลขแก้ **เนื้อหา** ด้วย | `build()` รัน `replace(/([:\[, ]\s*)(-?)\.(\d)/g, ...)` กับทั้งข้อความ JSON ทดสอบแล้ว `"ราคา 1, .5 บาท"` กลายเป็น `"ราคา 1, 0.5 บาท"` [V] ควรซ่อมเฉพาะนอก string literal หรือใช้ตัว parse ที่ทนกว่า |
| R-15 | ขอบเวลาซีนสุดท้าย | ผลรวม `s.d` ของซีนอาจน้อยกว่า `total` เล็กน้อยจาก floating point (ทดสอบสุ่ม 80,000 ชุด พบ ≈ 20% ที่ผลรวมต่ำกว่า `total` [V]) ช่วงนั้น `findIndex` คืน −1 แล้ว `Math.max(0, …)` ตกไปซีน 0 ผลจริงเล็กมาก (ช่องว่าง ~1e-14 วินาที) แต่ควรแก้เมื่อทำ export แบบกำหนดเวลาเอง (จับที่ t = total ตรงๆ) |
| R-16 | เสียงกับซีนไม่ซิงก์ | `speak()` เรียกเมื่อข้อความเปลี่ยน และ `speechSynthesis.cancel()` ก่อนทุกครั้ง ถ้าซีนสั้นเกินกว่าบทพูดจะจบ เสียงถูกตัด พรอมต์ขอ ≈6 ตัวอักษร/วินาที แต่ไม่มีการตรวจ |
| R-17 | `innerHTML` กับชื่อเสียง | `voices()` ใส่ `v.name` ลง `innerHTML` (บรรทัด 875) ชื่อมาจากระบบ ความเสี่ยงต่ำ แต่ควรใช้ `textContent`/`new Option` |
| R-18 | ความปลอดภัยของข้อมูลจาก AI | ไม่มี `eval`/`new Function`, ซับไตเติลวาดลง canvas (ไม่ผ่าน DOM), SVG โหลดผ่าน `Image` + data URI (สคริปต์ใน SVG ไม่ทำงานในบริบท `<img>`) → **ความเสี่ยงต่ำ** แต่ผลจาก `sample.json` ยังเป็นข้อมูลที่ไม่น่าเชื่อถือ ต้องผ่าน validator + เพดานขนาด |

---

## 3. แผนแยก Module (ไม่เขียนใหม่)

### 3.1 หลักการ

1. **ย้ายโค้ดวาดเดิมเข้าที่ใหม่ ไม่แก้เนื้อใน** การเปลี่ยนแปลงอยู่ที่ "ใครเรียกใคร และข้อมูลไหลทางไหน"
2. **ไฟล์เดียวเหมือนเดิม** แบ่งด้วย IIFE + namespace `CM` ภายใน `<script>` เดียว (ES Modules ที่ import ไฟล์ภายนอกใช้ไม่ได้เมื่อเปิดผ่าน `file://` และขัดกับเป้าหมาย "ไฟล์เดียว")
3. **แต่ละ phase ต้องทำให้แอปรันได้และผ่านเทสต์เดิม** (golden / selftest ในหัวข้อ 6)
4. **v1 ยังเป็น authoring format** พรอมต์ AI ไม่ต้องแก้จนกว่าจะถึง Phase 6

### 3.2 รูปแบบ namespace

```js
(function (root) {
  'use strict';
  const CM = root.CM = root.CM || {};

  CM.util = (function () {            // ไม่มี DOM/canvas
    const lerp = (a, b, t) => a + (b - a) * t;
    const clamp = (v, a = 0, b = 1) => Math.min(b, Math.max(a, v));
    const ease = u => u * u * (3 - 2 * u);
    function rng(seed) { /* mulberry32 */ }
    return { lerp, clamp, ease, rng };
  })();

  CM.schema = (function () { /* registry ของ enum/field/limit */ return { /* ... */ }; })();
  // ... modules ตามลำดับในตารางด้านล่าง
})(window);
```

ลำดับประกาศ: `util → schema → normalize → validate → compile → evaluate → painters.* → renderer → player → tts → exporter → ui → selftest`

### 3.3 ตาราง Module

| Module | หน้าที่ (เป็นเจ้าของ) | พึ่งพาได้ | **ห้ามแตะ** |
|---|---|---|---|
| `util` | math, easing, seeded RNG, hash | — | DOM, canvas |
| `schema` | enum (species/act/theme/vfx/transition), ขอบเขตค่า, alias, ค่า default, เพดานขนาด | `util` | DOM, canvas |
| `normalize` | แปลงสำเนียง v1 → v1 มาตรฐาน (ย้ายจาก `normalizeSceneData`) **ไม่เติม default** | `schema` | state ใดๆ |
| `validate` | ตรวจ/จำกัดค่า คืน `{value, errors[], warnings[]}` ไม่ throw | `schema` | canvas |
| `compile` | v1 → `ProjectV2`: สืบทอดข้ามซีน, เติม default, คำนวณเวลาสัมบูรณ์, สร้าง track ซับ/เสียง/overlay | `schema, validate` | canvas, DOM |
| `evaluate` | `evaluate(project, t) → FrameState` ฟังก์ชัน pure ไม่เก็บ state | `util, schema` | canvas, DOM, global |
| `painters.*` | `theme`, `actor`, `prop`, `vfx`, `overlay`: โค้ดวาดเดิมที่ห่อเป็น registry | `util` + `RenderContext` | project, DOM |
| `renderer` | เป็นเจ้าของ canvas/ctx, `RenderContext`, ลำดับชั้นการวาด, กล้อง, ทรานซิชัน | `painters, util` | DOM อื่นนอก canvas |
| `player` | นาฬิกา (rAF), play/stop/seek, เรียก `evaluate → render`, จับข้อผิดพลาดต่อเฟรม | `evaluate, renderer` | โค้ดวาด |
| `tts` | `speechSynthesis` (voice, rate, cancel) | — | canvas |
| `exporter` | `MediaRecorder` / (ตัวเลือก) WebCodecs | `player, renderer` | โค้ดวาด |
| `ui` | **ส่วนเดียวที่แตะ DOM/ปุ่ม/textarea** ประกอบ options ส่งให้ module อื่น | ทุก module | — |
| `ai` | พรอมต์ + `claude.use('sample')` (adapter เฉพาะโฮสต์) | `schema` | canvas |
| `selftest` | รัน `?selftest=1` | ทุก module | — |

**กฎการพึ่งพา:** ลูกศรไหลลงตามลำดับตารางเท่านั้น (ไม่ย้อน) และตรวจได้ด้วย grep ในหัวข้อ 6 (AC-G4)

### 3.4 Registry (จุดขยายแทน if-chain)

```js
CM.painters.actor.register('robot', { actions: ['idle','walk','run','punch'/*...*/],
  anchors: ['hand', 'center'],
  draw(rc, actor, pose) { /* ตัวเนื้อเดิมของ robotActor */ } });

CM.painters.vfx.register('beam', { layer: 'front', blend: 'lighter',
  draw(rc, fx, st) { /* ตัวเนื้อเดิมของ p=='beam' */ } });
```

- `registry.get(id)` ที่ไม่พบ → คืน painter สำรอง + บันทึก warning (ไม่กลืนเงียบ และไม่ throw)
- `schema` สร้างรายการ "คลังเครื่องมือ" ในพรอมต์จาก registry จึงมีแหล่งความจริงเดียว (แก้ R-05)

### 3.5 สิ่งที่ **ไม่ต้องแตะ** ในช่วงแรก

เนื้อใน `fig, robotActor, beastActor, cat4, critter, kang, shophouse, yamo, ring, decor, road/room/city/nature/space/lake, gfx, subtitle` ย้ายและห่อเท่านั้น

---

## 4. Data Contract v2

> สัญลักษณ์ชนิดข้อมูลด้านล่างเป็น **notation สำหรับเอกสาร** (คล้าย TypeScript) ไม่ได้หมายถึงการนำ TypeScript มาใช้ ตัวจริงเป็น JSON ธรรมดา + validator Vanilla JS

### 4.1 หลักการร่วม

| หัวข้อ | ข้อกำหนด |
|---|---|
| เวอร์ชัน | ฟิลด์ `schema: "clipmaker.project/2"` ถ้าไม่มี → ถือเป็น v1 และผ่าน `migrate` |
| พิกัด | สัดส่วนหน้าจอ 0–1 (x ซ้าย→ขวา, y บน→ล่าง, y ของตัวละคร = เท้า) ตามของเดิม; พิกัดภูมิศาสตร์ระบุเป็น `ll:[lat,lon]` เฉพาะในซีนที่มี `stage.geo` |
| เวลา | หน่วย **วินาที** ในระดับ timeline (สัมบูรณ์); ภายในซีน choreography ใช้ **สัดส่วนซีน `t∈[0,1]`** (เหมือน v1) เพื่อให้ปรับความยาวซีนแล้วจังหวะยังสัมพันธ์กัน |
| `id` | ทุกวัตถุที่ถูกอ้างอิงต้องมี `id` (`/^[\w\u0E00-\u0E7F-]{1,40}$/`) และไม่ซ้ำในขอบเขตของมัน |
| ค่าเริ่มต้น | ถูกเติมโดย `compile` **หลัง** การสืบทอดเท่านั้น (แก้ R-01) |
| ฟิลด์ไม่รู้จัก | ไม่ทิ้งเงียบๆ: เก็บใน `extra` พร้อม warning `UNKNOWN_FIELD` และ renderer ไม่ใช้ |
| ค่านอกช่วง | `validate` บีบเข้าช่วงพร้อม warning `CLAMPED` (ไม่ throw) |
| ค่าที่ทำให้วาดพัง | สี → ต้องผ่าน `/^#[0-9a-f]{6}$/i` ไม่งั้นใช้สีสำรอง; ตัวเลข → `Number.isFinite` ไม่งั้นใช้ default (กัน R-02) |
| Immutability | `ProjectV2` ถูก `Object.freeze` (ลึก) หลัง compile; ห้ามเขียน field ที่ derive ลงไป (แก้ R-07) |

### 4.2 ผลของ validator

```ts
type Issue = { code: string; path: string; message: string; severity: 'error'|'warning' };
type ValidateResult = { value: ProjectV2 | null; errors: Issue[]; warnings: Issue[] };
```

- `errors` = สร้างโปรเจกต์ไม่ได้ (เช่น ไม่มีซีน) → UI แสดงข้อความภาษาไทยพร้อม `path` เช่น `scenes[3].actors[1].c`
- `warnings` = สร้างได้แต่มีการแก้ค่า → UI แสดงจำนวน คลิกดูรายการ
- `validate` **ห้าม throw** ไม่ว่า input จะเป็นอะไร (รวม `null`, อาร์เรย์, ตัวเลข)

### 4.3 Project (root)

```ts
type ProjectV2 = {
  schema: 'clipmaker.project/2';
  meta: { title?: string; lang: 'th'|'en'; fps: 30; size: { w: 720; h: 1280 }; seed: number };
  assets: Record<string, AssetDef>;
  cast:   Record<string, CharacterDef>;
  scenes: SceneDef[];              // เรียงตามลำดับเล่น; id ไม่ซ้ำ
  timeline: TimelineDef;           // derive จาก scenes ตอน compile (ผู้ใช้ไม่ต้องเขียนเอง)
};

type AssetDef = {
  kind: 'svg';                     // (ขยายเป็น 'png' ภายหลังได้)
  source: string;                  // ข้อความ SVG
  size?: { w: number; h: number }; // จาก viewBox
  baseline: 'bottom';              // ขอบล่างของรูป = พื้น (ตามพรอมต์เดิม)
  facing: 'right';
};
```

### 4.4 Character

```ts
type Species =
  | 'human'|'zombie'|'robot'|'astronaut'|'kangaroo'
  | 'cat'|'dog'|'wolf'|'lion'|'bear'|'horse'|'dinosaur'|'dragon'|'elephant'|'monster'
  | 'fish'|'bird'|'blob'|'svg';

type CharacterDef = {
  id: string;
  species: Species;
  color?: string;                   // v1: col
  scale?: number;                   // v1: s   ช่วง 0.3–3, default 1
  appearance?: {                    // ใช้ตามชนิด; ฟิลด์ที่ชนิดนั้นไม่ใช้ → warning IGNORED_FIELD
    seed?: number;                  // หน้าตา/สีผิว/เสื้อที่สุ่มแบบคงที่ (human/zombie)
    outfit?: 'casual'|'officer'|'doctor';
    hair?: 'short'|'long';
    skin?: string; top?: string; bottom?: string;
    build?: number;                 // 0.7–1.8
    height?: number;                // v1: h
    accessories?: { cape?: string; mask?: string; horns?: boolean; glasses?: boolean; suit?: boolean; shirtless?: boolean };
    robot?: { core?: string; eye?: string; shoulder?: boolean };
  };
  svg?: { asset: string; widthRatio?: number };   // species 'svg' เท่านั้น (v1: svg, w)
};
```

**Contract ของ painter ตัวละคร**

- ประกาศ `actions: string[]` ที่รองรับ และ `anchors: ('hand'|'mouth'|'center')[]` ที่ **ต้องเผยแพร่** ลง `rc.scratch.anchors[actorId]`
- anchor ที่เป็นมาตรฐาน (อ้างอิงจากโค้ดเดิม): มนุษย์/ซอมบี้/หุ่นยนต์/จิงโจ้ = `hand` (มือซ้าย), สัตว์ใหญ่/ปลา = `mouth`, SVG = `center`
- ชนิด+ท่าที่ไม่รองรับ → ใช้ `idle` + warning `ACTION_UNSUPPORTED` (v1 ปล่อยเงียบ)

### 4.5 Scene

```ts
type SceneDef = {
  id: string;
  weight: number;                       // v1: w (เก็บไว้เพื่อ migrate; ความยาวจริงอยู่ที่ timeline)
  stage: {
    theme: 'road'|'room'|'city'|'nature'|'space'|'lake'|'custom'|'map'|'ring'|'arena'|'shophouse'|'yamo'|'photo';
    sky?: [string, string];             // เฉพาะ custom (และโค้ดเดิมใช้กับธีมอื่นบางส่วน)
    ground?: string;                    // สี hex หรือ 'grass'|'sand'|'snow'|'wood'|'tile'|'road'|'water'|'track'|'dark'
    horizon?: number;                   // v1: hz  0.3–0.85
    decor?: DecorDef[];                 // ≤ 40; 'front' วาดหลังพื้น
    sign?: [string, string?];           // theme road
    geo?: { center: [number, number]; widthKm: number; roads?: RoadDef[]; areas?: AreaDef[] };
    zones?: ZoneDef[]; water?: number[][];   // theme map
    background?: { asset: string };     // theme photo (สงวนไว้; ดู R-10)
  };
  actors: ActorPlacement[];             // ≤ 12
  crowd?: CrowdDef[];                   // รวมไม่เกิน 60 ตัว
  props: PropDef[];                     // ≤ 24 (รวมกับ vfx ≤ 40)
  vfx: VfxDef[];
  camera: CameraDef;
  look: { lights: LightDef[]; atmosphere: AtmosphereDef };
  transition: 'cut'|'fade'|'flash'|'wipe'|'zoom';   // v1: tr ; default: ซีนแรก 'fade' ที่เหลือ 'cut'
  inherit?: string[];                   // สิ่งที่สืบจากซีนก่อน เช่น ['stage','look','actors'] (v1: cont + กฎเดิม)
};

type LightDef = { type: 'glow'|'lamp'|'beam'; x: number; y: number; r: number; color: string;
                  flicker?: number; length?: number; spread?: number };   // ตัวแรก = แสงหลักที่กำหนดทิศเงา
type AtmosphereDef = { mood: 'calm'|'night'|'danger'|'warm'|'cold'; fog: number; vignette: number; grain: number };
type PropDef = { id: string; kind: 'rod'|'rocket'|'img'|'ball'|'songthaew'|'pin'|'route'|'dots'|'ring'; layer: 'back'|'front'; /* + ฟิลด์ตามชนิด */ };
```

**กฎการสืบทอด (ต้องเขียนเป็นเทสต์ — แก้ R-01)**

| ฟิลด์ | ถ้าซีนไม่ระบุ |
|---|---|
| `stage.theme/sky/ground/horizon/sign/decor/geo` | สืบจากซีนก่อน |
| `look.atmosphere`, `look.lights` | สืบจากซีนก่อน (ซีนแรกใช้ default: calm, fog .12, vig .42, grain .18) |
| `actors`, `props`, `crowd` | **ไม่สืบ** เว้นแต่ `inherit` มี `'actors'` หรือ v1 `cont:true` (ตำแหน่งปลายซีนก่อนเป็นตำแหน่งเริ่ม) |
| `camera` | ไม่สืบ (default: from=to={z:1,x:.5,y:.5}) |
| ซับ/เสียง/gfx/badge | ดู Timeline (ข้อความคงอยู่จนมีข้อความใหม่ → compile แปลงเป็นช่วงเวลาชัดเจน) |

### 4.6 Timeline

```ts
type TimelineDef = {
  duration: number;                // วินาที (≥ 1) ไม่บังคับขั้นต่ำ 60 เหมือน v1; ดูหัวข้อ 8 ข้อ 2
  tracks: [
    { id: 'shots';    type: 'shots';   clips: ShotClip[] },
    { id: 'subtitle'; type: 'text';    clips: SubtitleClip[] },
    { id: 'narration';type: 'speech';  clips: SpeechClip[] },
    { id: 'overlay';  type: 'overlay'; clips: OverlayClip[] },
  ];
};
type ShotClip     = { id: string; scene: string; start: number; dur: number };   // เรียงต่อเนื่อง ไม่ทับกัน
type SubtitleClip = { start: number; dur: number; text: string; highlight?: string };
type SpeechClip   = { start: number; text: string; voice?: string; rate?: number; estDur: number };
type OverlayClip  = { start: number; dur: number; kind: 'card'|'title'; /* ฟิลด์ตามชนิดจาก gfx เดิม */ };
```

**กฎ**

- `compile` คำนวณ `start/dur` ของ `ShotClip` จาก `weight` ด้วยสูตรเดิม `dur = duration * w / Σw` แล้ว **ปรับให้ผลรวมเท่ากับ `duration` พอดี** (ซีนสุดท้ายรับเศษ → แก้ R-15)
- v1 `text` ที่ซีนใด → `SubtitleClip` เริ่มที่ต้นซีนนั้นและ **ยาวถึงข้อความถัดไป** (พฤติกรรมเดิมแบบ explicit)
- `SpeechClip.estDur = จำนวนตัวอักษร / (6 × rate)` ถ้าเกินช่วงที่ครอบ → warning `SPEECH_TOO_LONG` (แก้ R-16 ฝั่งตรวจสอบ)
- `evaluate` หา clip ด้วยการค้นหาแบบ half-open `[start, start+dur)` และ `t ≥ duration` → เฟรมสุดท้าย (ไม่ตกไปซีน 0)

### 4.7 Animation

```ts
type ActorPlacement = {
  id: string;                      // v1: who
  cast: string;                    // อ้าง CharacterDef.id (v1: who ↔ cast key)
  anim: AnimDef;
  face?: string;                   // id ของอีกฝ่าย → หันหาอัตโนมัติ (v1: face)
  facing?: -1 | 1;                 // บังคับทิศ (v1: dir)
};

type AnimDef = {
  base: { x: number; y: number; scale?: number; rot?: number; action: ActionId };   // 0..1 ตามพิกัด
  to?: { x: number };              // เคลื่อนที่เชิงเส้นตลอดซีน (v1: to)
  keys?: Keyframe[];               // ถ้ามี → ใช้แทน base/to
};
type Keyframe = { t: number; x?: number; y?: number; scale?: number; rot?: number; action?: ActionId; ease?: 'linear'|'smooth' };

type ActionId =
  | 'idle'|'walk'|'run'|'jump'|'wave'|'dance'|'float'|'cheer'|'point'
  | 'hold'|'pull'|'cast'|'channel'
  | 'punch'|'guard'|'hit'|'ko'|'kick'
  | 'sit'|'sleep'|'swim'|'fly';
```

**กฎ**

- `keys` เรียงตาม `t` เพิ่มขึ้น, `t∈[0,1]`, ≤ 24 ต่อตัว; key แรกที่ `t>0` ให้ล็อกค่าจาก `base` (พฤติกรรมเดิมของ `kf` ใช้ key แรกเป็นฐาน)
- `action` เปลี่ยน **ทันที** ที่ key (ไม่ interpolate) ส่วน `x,y,scale,rot` interpolate ด้วย `smooth` (smoothstep เดิม)
- alias จาก v1 (`stand→idle, walking→walk, attack→punch, shoot→cast, dead→ko …`) อยู่ใน `schema.aliasAction` ที่เดียว
- ถ้า action เป็น `walk/run/fly/swim` แต่ไม่มี `to`/`keys` → `compile` เติม `to = x + facing*0.18` (ตามของเดิม) และบันทึกเป็น `DEFAULT_APPLIED`
- **ทิศหัน** (`facing`) ต้องคำนวณจาก `(project, t)` ล้วนๆ: ลำดับความสำคัญ `facing` ที่ผู้ใช้ระบุ → `face` → ทิศการเคลื่อนที่ของ `keys/to` → ทิศของ key ก่อนหน้า (compile สร้างตารางทิศล่วงหน้าต่อซีน) เพื่อเลิกพึ่ง `DIRS` (แก้ R-03)
- ท่าต่อสู้ (`punch/guard/hit/ko/kick`) หันหาตัวที่ใกล้สุดอัตโนมัติ เมื่อไม่มี `face/facing` (พฤติกรรมเดิม) → ทำใน `evaluate` ไม่ใช่ใน painter

### 4.8 Camera

```ts
type CameraDef = {
  from: { x: number; y: number; z: number };    // x,y ∈ [0,1]; z ∈ [1,4]  (default {0.5,0.5,1})
  to:   { x: number; y: number; z: number };    // default = from
  ease: 'smooth' | 'linear';                    // default 'smooth'
  roll: number;                                 // องศา, -15..15 (v1 เป็นค่าคงที่ต่อซีน)
  shake: number;                                // พิกเซล, 0..30
  dof: number;                                  // blur px, 0..12 (renderer ข้ามถ้าไม่รองรับ ctx.filter)
  follow?: { actor: string; offsetY?: number }; // เมื่อมี → ทับ x,y (ใช้ตำแหน่งจาก evaluate ของเวลาเดียวกัน)
};
```

**กฎ**

- `compile` แปลง v1 ทั้ง `cam:{from,to,shake,roll,dof,follow}` และ `camera:{zoomFrom,zoomTo,focusX,...}` (alias เดิม)
- Renderer บีบ `x,y` ให้อยู่ใน `[0.5/z, 1-0.5/z]` เสมอ (ของเดิม) และ `z ≥ 1 + |roll|×0.03` (ของเดิม)
- `follow` ใช้ตำแหน่งตัวละครที่ `evaluate` คำนวณในเฟรมเดียวกัน → ไม่ lag และข้ามซีนได้ถูกต้อง (แก้ R-03)
- `shake` ใช้ `sin/cos(time·k)` แบบเดิม (deterministic อยู่แล้ว)

### 4.9 VFX

แยก VFX ออกจาก prop (ปัจจุบันปนกันใน `props()`):

```ts
type VfxDef = {
  id: string;
  kind: 'bolt'|'wave'|'slash'|'flame'|'ice'|'water'    // v1: p:'shape'|'path' + form
      | 'beam' | 'aura' | 'blast' | 'spark' | 'ring' | 'particles';   // particles = v1 'dots'
  layer: 'back'|'front';                // default 'front'; anchored VFX ห้ามเป็น 'back' (ดูกฎ)
  from?: Ref; to?: Ref;                 // anchor ของ beam/shape; aura ใช้ at: Ref
  at?: Ref;                             // ตำแหน่งสำหรับ blast/spark/ring/particles
  start: number;                        // t∈[0,1] ภายในซีน (v1: at)
  duration?: number;                    // สัดส่วนซีน; default ตามชนิด (shape .22, beam .12, blast .25)
  loop?: boolean;
  style: { color: string; edge?: string; inner?: string; width?: number; scale?: number; fade?: number; outline?: boolean };
  blend?: 'source-over'|'lighter';      // default ตามชนิด (beam/aura/blast = 'lighter')
  seed?: number;                        // สำหรับความสุ่มที่คงที่ (ประกายของ beam, อนุภาค)
};
type Ref = { actor: string; anchor?: 'hand'|'mouth'|'center' } | { point: [number, number] };
```

**กฎ**

- VFX ที่อ้าง `Ref.actor` ต้อง `layer:'front'` (วาดหลังตัวละครซึ่งเผยแพร่ anchor แล้ว) → validator แปลง `back` เป็น `front` พร้อม warning `LAYER_FORCED` (แก้ปัญหา ANCH ของ back props)
- ความสุ่มทั้งหมดมาจาก `rc.rng(seed, frameIndex)` ห้ามใช้ `Math.random` ตรงๆ (AC-G5)
- ชนิดใหม่ = `painters.vfx.register(kind, {...})` + เพิ่มในรายการ enum ของ `schema`

### 4.10 Renderer และ FrameState

**`evaluate` เป็นฟังก์ชัน pure; renderer รับผลลัพธ์ไปวาดเท่านั้น**

```ts
type FrameState = {
  time: number; frameIndex: number;
  shot: { sceneId: string; local: number /*วินาทีในซีน*/; k: number /*0..1*/; transition: { type: string; p: number } };
  camera: { x: number; y: number; z: number; roll: number; shake: number; dof: number };
  stage: SceneDef['stage'];
  actors: ResolvedActor[];                      // เรียงตาม y แล้ว (depth)
  props: ResolvedProp[]; vfx: ResolvedVfx[];
  look: { lights: LightDef[]; atmosphere: AtmosphereDef };
  overlays: { subtitle?: { text: string; highlight?: string; alpha: number }; gfx: ResolvedGfx[] };
  fade: { color: string; alpha: number };       // ทรานซิชัน + fade-out ท้ายคลิป
};
type ResolvedActor = { id: string; species: Species; x: number; y: number; scale: number; rot: number;
                       facing: -1|1; action: ActionId; depthScale: number; character: CharacterDef };
```

```ts
interface Renderer {
  readonly id: 'canvas2d' | 'canvas2d+webgl-post';
  readonly capabilities: { roundRect: boolean; filter: boolean; webgl2: boolean };
  init(canvas: HTMLCanvasElement, opts: { subtitles: boolean; fontStack: string }): void;
  render(fs: FrameState): void;               // ต้องไม่ throw; ถ้า painter พังให้ข้ามชั้นนั้นและรายงาน
  dispose(): void;
}

type RenderContext = {                         // ส่งเข้า painter ทุกตัว แทน global
  g: CanvasRenderingContext2D; W: number; H: number;
  rng: (seed: number, i?: number) => number;   // deterministic
  scratch: { anchors: Record<string, {x:number;y:number}>; positions: Record<string, {x:number;y:number}> };  // รีเซ็ตทุกเฟรมโดย renderer
  assets: { get(id: string): HTMLImageElement | null };
  caps: Renderer['capabilities'];
};
```

**ลำดับการวาดที่ต้องรักษา (ตามโค้ดเดิม)**: กล้อง(transform) → `theme` → props `back` → `actors` (+ crowd) → props/VFX `front` → `lights` → ยกเลิกกล้อง → `atmosphere` → `gfx` → `subtitle` → ทรานซิชัน/fade

**เรื่อง Canvas 2D / WebGL / SVG**

- **Canvas 2D**: renderer หลักต่อไป (ผ่านการใช้งานจริงแล้ว)
- **SVG**: คงบทบาท asset (`Image` ผ่าน data URI) ซึ่งปลอดภัยกับสคริปต์ใน SVG และไม่ต้องเพิ่ม DOM layer; ต้อง `await img.decode()` ก่อนเริ่มเล่น/อัด (แก้ R-10)
- **WebGL (ทางเลือก, Phase 6)**: ใช้เฉพาะ post-processing (vignette/grain/fog/dof) โดยอ่านผลจาก canvas 2D เป็น texture แล้วผ่าน fragment shader, เปิดเมื่อ `webgl2` พร้อมและตรวจ `webglcontextlost`; ถ้าไม่พร้อมให้ใช้ `atmos()` เดิม **ห้ามเป็นข้อบังคับ**

**เรื่อง export (ข้อจำกัดจริง)**

- `MediaRecorder` ประทับเวลาตามนาฬิกาจริง จึง **เร่งเกินเรียลไทม์ไม่ได้** แม้ step เฟรมเองด้วย `captureStream(0)` + `requestFrame()`
- ตัวเลือก A (แนะนำเริ่มก่อน): คง realtime แต่ทำให้ทนขึ้น: ตรวจ `document.visibilityState`, วัด drift แล้วเตือน/หยุด, pace ที่ 30 fps, preload asset ก่อนอัด
- ตัวเลือก B (ใหญ่กว่า, ตรวจ `VideoEncoder` ก่อน): WebCodecs เข้ารหัสทีละเฟรมด้วย timestamp ที่กำหนดเอง ต้องเขียน muxer (WebM/MP4) เอง เป็นงานก้อนใหญ่ ควรแยกเป็น milestone ถ้าต้องการ export เร็วกว่าเรียลไทม์/frame-accurate
- เสียงพากย์จาก TTS อัดลงไฟล์ไม่ได้ (ตามที่แอปแจ้งผู้ใช้อยู่แล้ว); contract จึงเก็บ `narration` เป็นข้อมูลแยก เพื่อรองรับเสียงที่ผู้ใช้อัด/นำเข้าภายหลัง

### 4.11 ตัวอย่าง `ProjectV2` ขั้นต่ำ

```json
{
  "schema": "clipmaker.project/2",
  "meta": { "title": "ตกปลา", "lang": "th", "fps": 30, "size": { "w": 720, "h": 1280 }, "seed": 1 },
  "assets": {},
  "cast": {
    "ลุง": { "id": "ลุง", "species": "human", "color": "#2f6fb5", "appearance": { "seed": 7 } },
    "ปลา": { "id": "ปลา", "species": "fish" }
  },
  "scenes": [
    {
      "id": "s1", "weight": 2,
      "stage": { "theme": "lake" },
      "actors": [
        { "id": "ลุง", "cast": "ลุง", "anim": { "base": { "x": 0.25, "y": 0.63, "action": "hold" } } },
        { "id": "ปลา", "cast": "ปลา", "anim": { "base": { "x": 0.7, "y": 0.85, "action": "swim" }, "to": { "x": 0.5 } } }
      ],
      "props": [ { "id": "rod1", "kind": "rod", "layer": "front", "who": "ลุง", "target": "ปลา", "ang": [45, 55], "taut": [0.2, 0.9] } ],
      "vfx": [],
      "camera": { "from": { "x": 0.5, "y": 0.5, "z": 1 }, "to": { "x": 0.3, "y": 0.6, "z": 1.3 }, "ease": "smooth", "roll": 0, "shake": 0, "dof": 0 },
      "look": { "lights": [], "atmosphere": { "mood": "calm", "fog": 0.12, "vignette": 0.42, "grain": 0.18 } },
      "transition": "fade"
    },
    {
      "id": "s2", "weight": 1,
      "stage": { "theme": "lake" },
      "actors": [], "props": [], "vfx": [],
      "camera": { "from": { "x": 0.45, "y": 0.7, "z": 1.8 }, "to": { "x": 0.45, "y": 0.7, "z": 1.8 }, "ease": "smooth", "roll": 0, "shake": 0, "dof": 0 },
      "look": { "lights": [], "atmosphere": { "mood": "calm", "fog": 0.12, "vignette": 0.42, "grain": 0.18 } },
      "transition": "zoom",
      "inherit": ["actors"]
    }
  ],
  "timeline": {
    "duration": 60,
    "tracks": [
      { "id": "shots", "type": "shots", "clips": [
        { "id": "c1", "scene": "s1", "start": 0, "dur": 40 },
        { "id": "c2", "scene": "s2", "start": 40, "dur": 20 } ] },
      { "id": "subtitle", "type": "text", "clips": [
        { "start": 0, "dur": 60, "text": "ลุงนั่งตกปลาอยู่บนสะพาน", "highlight": "ตกปลา" } ] },
      { "id": "narration", "type": "speech", "clips": [
        { "start": 0, "text": "ลุงนั่งตกปลาอยู่บนสะพาน", "estDur": 3.5 } ] },
      { "id": "overlay", "type": "overlay", "clips": [] }
    ]
  }
}
```

(ตัวอย่างย่อจากตัวอย่างตกปลาในพรอมต์ ซีน s2 สืบ `stage` จาก s1 ผ่านกฎสืบทอด และคงตัวละครผ่าน `inherit:["actors"]`)

### 4.12 ตาราง map v1 → v2

| v1 | v2 | หมายเหตุ |
|---|---|---|
| `cast` / `characters` / `characterBible` | `cast` | alias เดิมคงไว้ใน `schema.alias` |
| `scenes` / `shots` / `sequence` / `beats` | `scenes` | |
| `scene.w` / `duration` / `weight` | `scene.weight` → `timeline.shots[].dur` | สัมบูรณ์ที่ compile |
| `scene.text` + `highlight` | `timeline.subtitle[]` + `narration[]` | คงอยู่ถึงข้อความถัดไป → ระบุช่วงชัด |
| `scene.badge`, `scene.gfx` | `timeline.overlay[]` | `badge` เดิมไม่ได้ถูกวาดแล้วใน `frame` (มีแต่ใน `frameOld`) |
| `scene.theme/sky/ground/hz/sign/decor/geo` | `scene.stage.*` | `hz` → `horizon` |
| `actor.who` | `actor.id` + `actor.cast` | |
| `actor.c`/`type`/`species` | `character.species` | |
| `actor.s`/`scale`, `col`, `build`, `h`, `outfit`, `hair`, `seed`, … | `character.scale/color/appearance.*` | ค่าใน scene ทับค่าใน cast (พฤติกรรมเดิม `{...CAST[who], ...a}`) |
| `actor.act`/`action`/`movement` | `anim.base.action` | ผ่าน `aliasAction` |
| `actor.x,y,to,rot` | `anim.base.{x,y,rot}`, `anim.to.x` | |
| `actor.keys[{t,x,y,s,rot,act}]` | `anim.keys[]` | `s`→`scale`, `act`→`action` |
| `actor.dir`/`facing`, `face` | `facing`, `face` | |
| `scene.cam`, `scene.camera` | `scene.camera` | ทั้งสองสำเนียง |
| `props[p:'rod'|'rocket'|'img'|'ball'|'songthaew'|'pin'|'route'|'dots'|'ring']` | `props[kind]` / `vfx` (`dots`→`particles`) | |
| `props[p:'shape'|'path', form]` | `vfx[kind=form]` | `from/to` → `Ref` |
| `props[p:'beam'|'aura'|'blast'|'spark']` | `vfx[kind]` | |
| `scene.lights`, `scene.atm` | `scene.look.lights`, `scene.look.atmosphere` | `vig`→`vignette` |
| `scene.tr` | `scene.transition` | |
| `scene.cont` | `scene.inherit:['actors',...]` | |
| `scene.crowd` | `scene.crowd` | ขยายเป็นตัวละคร id `cr0..` ตอน compile (ไม่สร้างทุกเฟรม) |
| `assets{id:svgText}` | `assets[id]={kind:'svg',source}` | |
| `j.plan`, `genre`, `title` … | `meta.*` / `extra` | |

### 4.13 เพดานขนาด (เสนอ — ปรับหลังวัด baseline)

| รายการ | เพดาน | เมื่อเกิน |
|---|---|---|
| ขนาด JSON ทั้งก้อน | 2 MB | error |
| ซีน | 60 | error |
| actors / ซีน | 12 | ตัดส่วนเกิน + warning |
| ฝูงชนรวม / ซีน | 60 | ตัดส่วนเกิน + warning |
| props + vfx / ซีน | 40 | ตัดส่วนเกิน + warning |
| decor / ซีน | 40 | ตัดส่วนเกิน + warning |
| keys / actor | 24 | ตัดส่วนเกิน + warning |
| asset SVG | 20 KB ต่อชิ้น, 40 ชิ้น, รวม 400 KB | ข้ามชิ้นนั้น + warning |
| ข้อความซับ | แนะนำ ≤ 45 ตัวอักษร; อัตราพูด > 8 ตัวอักษร/วินาที | warning |
| ความยาวคลิป | 1–300 วินาที | บีบเข้าช่วง |

---

## 5. Roadmap แบบค่อยเป็นค่อยไป

ขนาดงาน: **S** ≈ ครึ่งวัน, **M** ≈ 1–2 วัน, **L** ≈ 3–5 วัน (ประมาณการคร่าวๆ สำหรับนักพัฒนาที่คุ้นโค้ดนี้)

| Phase | เป้าหมาย | สิ่งที่ทำ | ห้ามทำ | ขนาด | ปิดงานเมื่อ |
|---|---|---|---|---|---|
| **0** Safety net | ไม่พังเงียบ + มีฐานวัด | ย้าย `normalizeSceneData` เข้า `try`; ครอบ `frame()` ใน `loop` ด้วย try/catch แล้วแสดงข้อผิดพลาดใน `#msg` (หยุดเล่นอย่างสะอาด); `.catch` ให้ `document.fonts.load`; `selftest` รุ่นแรก; บันทึก baseline เวลาเฟรม (ดู AC-P0-3) | แตะโค้ดวาด | S | AC-P0-* |
| **1** แก้ R-01 | สืบทอดตามที่พรอมต์สัญญา | แยกขั้น normalize (ไม่เติม default) → สืบทอด → เติม default; เพิ่มเทสต์การสืบทอด | เปลี่ยนรูปแบบ v1 | S–M | AC-P1-* |
| **2** Schema + Validate + Compile | ภาษากลาง v2 | `schema` registry, `validate`, `compile` (ยังไม่ส่งให้ renderer — สร้าง v2 คู่ขนานแล้วเทียบกับผลเดิม); ย้าย regex ซ่อมตัวเลข (R-14) | ให้ renderer อ่าน v2 | M–L | AC-P2-* |
| **3** Evaluate + RenderContext | `frame` pure | `evaluate(project,t)`; สร้าง `RenderContext` แทน `ANCH/CAMI/POSG/DIRS/LT/GEO`; renderer อ่าน options (เลิกอ่าน DOM); cache ซับ/ฝูงชน | แก้ท่าทางตัวละคร | L | AC-P3-* |
| **4** Painter registry | เพิ่มของใหม่ที่เดียว | ห่อ theme/actor/prop/vfx เป็น registry; แยก `actors()` เป็น pose resolver + dispatch; แยก VFX ออกจาก prop | เขียน painter ใหม่ | M–L | AC-P4-* |
| **5** Determinism + Export | seek ถูกต้อง, export ทนขึ้น | seeded RNG แทน `Math.random`; facing จากเวลา; preload asset; visibility guard ใน record; ตัวเลือกความยาว < 60 วินาที | ทำ muxer เอง (แยก milestone) | M | AC-P5-* |
| **6** (ทางเลือก) พรอมต์ + WebGL post | ลดความไม่ตรงของ AI, ภาพหลังประมวลผล | สร้างส่วนคลังเครื่องมือของพรอมต์จาก `schema`; `webgl2` post-fx หลัง feature detect | บังคับ WebGL | M | AC-P6-* |
| **7** กวาดโค้ดตาย | ลดภาระ | ลบรายการในภาคผนวก B เมื่อ golden ผ่านต่อเนื่อง | ลบก่อนมีเทสต์ | S | AC-P7-* |

**ลำดับที่แนะนำสำหรับ PR แรก:** Phase 0 + Phase 1 (แก้ได้ทั้ง R-02 และ R-01 โดยไม่เปลี่ยนโครงสร้าง) แล้วค่อยเริ่ม Phase 2

---

## 6. Acceptance Criteria

หลักการ: ทุกข้อ **ทดสอบได้โดยไม่ต้องใช้เครื่องมือภายนอก** รันผ่านหน้าเดียวกันด้วย `?selftest=1` (ผลเป็นตาราง ผ่าน/ไม่ผ่าน บนหน้าเว็บ) และ/หรือ DevTools

### 6.1 เกณฑ์ร่วม (Global)

| ID | เกณฑ์ | วิธีตรวจ |
|---|---|---|
| AC-G1 | เปิดไฟล์ด้วยการดับเบิลคลิก (`file://`) ได้ ไม่ใช้ framework/CDN script/`import`/`require`; หากปิดลิงก์ฟอนต์ แอปยังเล่นได้ด้วยฟอนต์สำรอง | DevTools › Network ไม่มีคำขอสคริปต์ภายนอก; ค้น `import `, `require(`, `<script src` ต้องไม่พบ |
| AC-G2 | **ความเข้ากันได้ย้อนหลัง:** fixtures v1 อย่างน้อย 6 ชุด (SAMPLE, ตัวอย่างตกปลาในพรอมต์, มวย/ring, แผนที่ `geo`, `assets` SVG, `crowd`) ให้ `evaluate()` snapshot เท่ากับ baseline ยกเว้นรายการที่ระบุว่าเป็นการแก้บั๊ก (R-01) | `selftest` เทียบ hash ของ `JSON.stringify(FrameState)` ที่เวลา 0.5, กลางคลิป, `duration−0.1` |
| AC-G3 | ไม่มีตัวแปร/ฟังก์ชันระดับบนสุดใหม่นอก `CM` (หลัง Phase 2 เป็นต้นไป: ไม่มีเลย ยกเว้น `CM`) | `selftest` เทียบ `Object.keys(window)` ก่อน/หลังโหลดสคริปต์ |
| AC-G4 | กฎทิศการพึ่งพา: `$(`/`document.`/`getElementById` อยู่ใน `ui` เท่านั้น; `getContext`/`g.` อยู่ใน `renderer`/`painters` เท่านั้น; `evaluate/compile/validate/schema/util` ไม่มีทั้งสองอย่าง | สคริปต์ grep ใน `selftest` แบ่งตามบล็อก module |
| AC-G5 | ไม่มี `Math.random` นอก `util.rng` (หลัง Phase 5) | grep |
| AC-G6 | ไม่มี exception ที่ไม่ถูกจับระหว่างเล่น fixtures ทั้งหมดจนจบ (รวมกรณีข้อมูลเพี้ยน AC-V*) | `window.onerror`/`unhandledrejection` เก็บเป็น 0 |
| AC-G7 | เบราว์เซอร์เป้าหมาย Chrome/Edge รุ่นปัจจุบันครบทุกฟีเจอร์; Firefox/Safari: เล่นและพรีวิวได้ ส่วน `roundRect`, `ctx.filter`, `MediaRecorder` ที่ไม่มีต้อง degrade พร้อมข้อความ ไม่ throw | ทดสอบมือ + `caps` ถูกรายงานใน `selftest` |

### 6.2 รายเฟส

**Phase 0**

| ID | เกณฑ์ |
|---|---|
| AC-P0-1 | ใส่ `{"scenes":{"a":1}}`, `null`, `{"scenes":[null]}` แล้วกด "เล่น": ไม่มี exception หลุดคอนโซล, `#msg` แสดงข้อความไทยบอกว่า JSON ใช้ไม่ได้ |
| AC-P0-2 | ฉีดสีผิด (`"core":"red!"`) หรือรัศมีติดลบในซีนกลางคลิป: การเล่นไม่ค้างเงียบ แสดงข้อผิดพลาด ระบุซีนที่เกิด และหยุดอย่างสะอาด (`playing=false`, `speechSynthesis.cancel()`) |
| AC-P0-3 | `selftest` วัดเวลา `frame()` 600 เฟรมต่อ fixture แล้วรายงาน median/p95 และบันทึกเป็น **baseline** (ยังไม่มีเกณฑ์ผ่าน/ไม่ผ่านตัวเลข) |
| AC-P0-4 | ปิด/บล็อกลิงก์ฟอนต์: เฟรมแรกยังถูกวาด |

**Phase 1**

| ID | เกณฑ์ |
|---|---|
| AC-P1-1 | ซีน 2 ที่ไม่ใส่ `theme/sky/ground/atm` ได้ค่าเดียวกับซีน 1 (ทดสอบด้วยตัวอย่างตกปลา ซีน 2 ต้องเป็น `lake`) |
| AC-P1-2 | ซีนแรกที่ไม่ใส่อะไรเลย ได้ default เดิม (`custom`, sky `#18243c→#51405f`, atm calm/.12/.42/.18) |
| AC-P1-3 | ซีนที่ระบุ `atm` ของตัวเอง ไม่ถูกทับ และซีนถัดไปที่ไม่ระบุ สืบ `atm` นั้น |
| AC-P1-4 | fixtures ที่ **ทุกซีนระบุ theme ครบ** ให้ผลเท่า baseline เดิมทุกประการ (ยืนยันว่าไม่มี regression) |

**Phase 2 (schema/validate/compile)**

| ID | เกณฑ์ |
|---|---|
| AC-P2-1 | `validate()` ไม่ throw ต่อ input ใดๆ: ชุดทดสอบ ≥ 30 ค่า (`null`, `[]`, `0`, สตริง, อ็อบเจ็กต์ซ้อนลึก, `NaN`, `Infinity`, key `__proto__`) |
| AC-P2-2 | ทุก `Issue` มี `code`, `path`, `message` ภาษาไทย; `path` ชี้ตำแหน่งจริง (เช่น `scenes[2].actors[0].act`) |
| AC-P2-3 | enum ที่ไม่รู้จัก (`c`, `act`, `theme`, `tr`, `form`) → ใช้ค่าสำรองและมี warning (ไม่มีการกลืนเงียบ) |
| AC-P2-4 | สีไม่ผ่าน `#rrggbb` และตัวเลขไม่จำกัด ไม่เข้าสู่ `ProjectV2` เลย |
| AC-P2-5 | ผลรวม `ShotClip.dur` = `timeline.duration` เป๊ะ (เทียบด้วย `===` หลังกำหนดให้ซีนสุดท้ายรับเศษ) ทดสอบสุ่ม ≥ 10,000 ชุดน้ำหนัก |
| AC-P2-6 | `ProjectV2` ถูก freeze ลึก: พยายามเขียนแล้วใน strict mode ต้อง throw ใน `selftest` |
| AC-P2-7 | ทุก fixture v1 → `compile` ได้ → `migrate` → `compile` ซ้ำได้ผลเดิม (idempotent) |
| AC-P2-8 | ข้อความที่มี `".5"` ในสตริง ไม่ถูกแก้เนื้อหา (R-14) แต่ตัวเลข `.5` นอกสตริงยังถูกซ่อมเหมือนเดิม |
| AC-P2-9 | เกินเพดานในหัวข้อ 4.13: ตัดและมี warning ไม่ error เฉพาะรายการที่ระบุว่า error |

**Phase 3 (evaluate/RenderContext)**

| ID | เกณฑ์ |
|---|---|
| AC-P3-1 | `evaluate(p,t)` เรียกซ้ำด้วยอินพุตเดียวกัน 100 ครั้ง ผล `JSON.stringify` เหมือนกันทุกครั้ง |
| AC-P3-2 | **seek-equivalence:** ผล `evaluate(p,t)` ที่ได้จากการ seek ตรง กับที่ได้หลังเล่นผ่านเฟรมก่อนหน้า 0..t ต่อเนื่อง ต้องเท่ากันทุกฟิลด์ (ทดสอบ 50 ค่า t สุ่มต่อ fixture) |
| AC-P3-3 | camera `follow` ตรงกับตำแหน่งตัวละครในเฟรมเดียวกัน (ระยะต่างสูงสุด = 0) รวมเฟรมแรกของซีน |
| AC-P3-4 | renderer ทำงานได้โดยไม่เรียก DOM: รันใน `selftest` โดยส่ง canvas ที่สร้างจากโค้ด (`document.createElement('canvas')` นอกหน้า) |
| AC-P3-5 | `subtitle` ไม่สร้าง `Intl.Segmenter` ซ้ำต่อเฟรม (นับการสร้างได้ ≤ 1 ครั้งต่อข้อความ) และ `crowd` ถูกขยายตอน compile ครั้งเดียว |
| AC-P3-6 | median เวลาเฟรมไม่แย่กว่า baseline เกิน 10% ต่อ fixture (เมื่อวัดบนเครื่อง/เบราว์เซอร์เดิม) |

**Phase 4 (registry)**

| ID | เกณฑ์ |
|---|---|
| AC-P4-1 | เพิ่ม VFX ชนิดใหม่ 1 ชนิดได้โดยแก้ไม่เกิน 2 ที่: ไฟล์ painter (register) และ enum ใน `schema` (วัดจากตัวอย่างจริงตอนทำ PR) |
| AC-P4-2 | painter ที่ throw ถูกจับ: เฟรมยังวาดชั้นอื่นต่อ และ `selftest` เห็น warning พร้อม id ของ painter |
| AC-P4-3 | registry ที่ไม่พบ `id` → painter สำรอง + warning (ไม่ throw) |
| AC-P4-4 | ทุกชนิดตัวละครประกาศ `actions` และ `anchors`; ท่าที่ไม่รองรับถูก map เป็น `idle` + warning |
| AC-P4-5 | VFX ที่อ้าง anchor ของตัวละครแสดงผลถูกต้องแม้ประกาศ `layer:'back'` (ถูกบังคับเป็น `front` + warning) |
| AC-P4-6 | ภาพ L2 (pixel smoke) ของ fixtures ยังผ่าน (ดูหัวข้อ 6.3) |

**Phase 5 (determinism/export)**

| ID | เกณฑ์ |
|---|---|
| AC-P5-1 | เรนเดอร์เฟรมเดียวกัน (fixture, t เดียวกัน) 2 ครั้งในเบราว์เซอร์เดียวกัน ได้ `ImageData` เหมือนกันทุกไบต์ (ครอบคลุม grain และ beam) |
| AC-P5-2 | เริ่มอัดก่อนที่ asset จะ decode เสร็จไม่ได้: `record()` รอ `decode()` ครบก่อน; เฟรมแรกของไฟล์มี asset ครบ |
| AC-P5-3 | ถ้าแท็บถูกซ่อนระหว่างอัด: แอปแสดงคำเตือนและไม่ส่งออกไฟล์ที่ความยาวผิดโดยไม่แจ้ง (ระบุว่าไฟล์อาจสั้นกว่าที่ตั้ง) |
| AC-P5-4 | ตั้งความยาวได้ต่ำกว่า 60 วินาที (เช่น 10 วินาที) และคลิปที่เล่น/อัดมีความยาวตามนั้น ±0.1 วินาที |
| AC-P5-5 | เฟรมที่ `t = duration` และ `t = duration − ε` ไม่ตกไปซีนแรก |

**Phase 6 (ทางเลือก)**

| ID | เกณฑ์ |
|---|---|
| AC-P6-1 | ส่วน "คลังเครื่องมือ" ของพรอมต์ถูกสร้างจาก `schema`/registry: เพิ่ม enum 1 ค่า → ปรากฏในพรอมต์โดยไม่แก้ข้อความ |
| AC-P6-2 | `webgl2` ไม่พร้อม/ปิดใช้ → ผลภาพมาจาก `atmos()` เดิม และ `selftest` รายงาน `renderer=canvas2d` |
| AC-P6-3 | จำลอง `webglcontextlost` แล้วเล่นต่อได้ (กลับไป canvas2d) ไม่ค้าง |
| AC-P6-4 | ผล post-fx ไม่เปลี่ยน `FrameState` (เฉพาะชั้นภาพ) |

**Phase 7**

| ID | เกณฑ์ |
|---|---|
| AC-P7-1 | รายการในภาคผนวก B ถูกลบ; ไม่มีการอ้างถึงชื่อเหล่านั้นเหลืออยู่ (grep = 0) |
| AC-P7-2 | AC-G2 และ AC-P4-6 ผ่านเหมือนเดิมหลังลบ |

### 6.3 ระดับของเทสต์ภาพ

- **L1 — FrameState snapshot** (ข้ามเครื่องได้): hash ของ `JSON.stringify(evaluate(...))` ใช้เป็นเกณฑ์หลักของ AC-G2
- **L2 — Pixel smoke** (เบราว์เซอร์เดียวกัน): เฟรมไม่ว่างเปล่า (ความแปรปรวนของพิกเซลเกินเกณฑ์ในแต่ละแถบแนวตั้ง 4 ส่วน) และ AC-P5-1 ความเหมือนเป๊ะ; **ไม่เทียบพิกเซลข้ามเครื่อง** เพราะฟอนต์/การ anti-alias ต่างกัน

### 6.4 ชุดข้อมูลเพี้ยน (AC-V) สำหรับ validator

อย่างน้อย: `scenes` เป็นอ็อบเจ็กต์แทนอาร์เรย์, ซีนเป็น `null`, `actors` เป็นสตริง, `keys` ไม่เรียง `t`, `t` นอก 0–1, สี 3 หลัก/ชื่อสี/สตริงว่าง, รัศมีติดลบ, `NaN`, `Infinity`, `to` ชี้นอกจอ (−5, 9), id ซ้ำ, `follow` อ้างตัวละครที่ไม่มี, `face` อ้างตัวละครที่ไม่มี, VFX อ้าง anchor ที่ชนิดตัวละครไม่มี, asset id ไม่มี, SVG > เพดาน, `__proto__` เป็น key — ทุกกรณี: ไม่ throw, ได้ warning/error ที่มี `path`

---

## 7. ลำดับความเสี่ยงที่ยังเหลือหลังทำตามแผน

| ความเสี่ยง | เหตุผล | ทางลด |
|---|---|---|
| Export เร็วกว่าเรียลไทม์/เฟรมแม่นยำยังไม่ได้ (ถ้าไม่ทำตัวเลือก B) | `MediaRecorder` ใช้เวลาจริง | ตัดสินใจเรื่อง WebCodecs + muxer เป็น milestone แยก (ข้อ 8.3) |
| เสียงพากย์อัดลงไฟล์ไม่ได้ | ข้อจำกัดของ `speechSynthesis` | contract เก็บ narration แยก เพื่อรองรับเสียงนำเข้า |
| ความหลากหลายของฟอนต์/ฟีเจอร์ข้ามเบราว์เซอร์ | ใช้ API ใหม่ (`roundRect`, `filter`) | feature detect + degrade (AC-G7) |
| พรอมต์ยาวมาก (~100 บรรทัด) ทำให้ AI พลาดฟิลด์ได้ | เป็นธรรมชาติของงาน LLM | validator ป้องกันฝั่งรับ; Phase 6 ลดความไม่สอดคล้อง |
| ตัวเลข performance ยังไม่ทราบ | ยังไม่ได้วัด | AC-P0-3 วัด baseline ก่อนตั้งเกณฑ์ |

---

## 8. คำถามที่ต้องตัดสินใจ

1. **ความยาวขั้นต่ำ 60 วินาที** (`total = Math.max(60, …)`) เป็นข้อกำหนดผลิตภัณฑ์หรือค่าที่ติดมา? ถ้าเป็นข้อกำหนด Phase 5 (AC-P5-4) ต้องเปลี่ยนเป็น "ตั้งได้ แต่ preset เริ่มต้น 60"
2. **ขอบเขตของ v2:** ต้องการให้ผู้ใช้/AI เขียน v2 ตรงๆ ในอนาคตหรือไม่ หรือ v2 เป็น runtime ภายในอย่างเดียว (ข้อเสนอนี้ตั้งต้นที่แบบหลัง)
3. **Export:** ยอมรับ realtime ที่ทนขึ้น (ตัวเลือก A) หรือต้องการ frame-accurate/เร็วกว่าเรียลไทม์ (ตัวเลือก B ที่เป็นงานก้อนใหญ่)?
4. **ฟีเจอร์ภาพถ่าย (`photo`/`stylize`/`PH`):** จะทำให้ครบหรือลบ? (ปัจจุบันไม่มีทางเข้า)
5. **เบราว์เซอร์เป้าหมาย:** ยืนยัน Chrome/Edge เป็นหลักตามข้อความในแอป และ Firefox/Safari เป็น best-effort?
6. **`window.claude.use('sample')`:** เป็น API ของโฮสต์ที่แอปนี้รันอยู่ ถ้าแอปจะแจกจ่ายนอกโฮสต์นั้น ต้องการ adapter ทางเลือก (วาง JSON เอง ซึ่งมีอยู่แล้ว) เท่านั้นหรือไม่?

---

## ภาคผนวก A — ฟิลด์ v1 ที่ renderer ใช้จริง (สำหรับสร้าง `schema` registry)

- **root:** `cast|characters|characterBible`, `scenes|shots|sequence|beats`, `assets`, `duration`, `plan` (ไม่ถูกใช้ตอนวาด)
- **scene:** `theme|environmentType|sceneType`, `sky|skyTop,skyBottom`, `ground|groundColor|floor`, `hz`, `decor`, `sign`, `w|duration|weight`, `text`, `highlight`, `badge`, `gfx`, `cam|camera`, `tr`, `cont`, `lights`, `atm|mood,weather,details`, `actors`, `props`, `crowd`, `geo`, `roads`, `areas`, `water`, `zones`, `bg`, `char` (รูปแบบเก่า: ตัวละครเดี่ยว), `location|setting|place|environment|background|details` (ใช้อนุมาน theme/decor/atm)
- **actor:** `who|name|id`, `c|type|species`, `s|scale`, `x`, `y`, `to|endX`, `position{x,y}`, `dir|facing`, `face`, `act|action|movement`, `rot`, `keys[{t,x,y,s,rot,act|action}]`, `col`, `build`, `h`, `skin`, `hair`, `outfit`, `seed`, `suit`, `shirtless`, `cape`, `mask`, `horns`, `glasses`, `bot`, `core`, `eye`, `shoulder`, `svg`, `w`
- **act ที่ painter รองรับ:** idle walk run jump wave dance float cheer point hold pull cast channel punch guard hit ko kick sit sleep swim fly (และ alias ใน `aliasAct`)
- **prop `p`:** rod rocket img pin route dots ring shape path beam aura blast songthaew ball spark
- **shape `form`:** bolt (ค่าเริ่มต้น) wave slash flame ice water
- **decor `k`:** tree pine building mountain cloud sun moon lamp fence flag stands finish box circle star castle pillar torch arch
- **ground preset:** grass sand snow wood tile road water track dark
- **transition `tr`:** cut fade flash wipe zoom
- **mood:** night danger calm warm cold

## ภาคผนวก B — โค้ดที่ไม่ถูกเรียกในเส้นทางปัจจุบัน (ลบได้หลังมี golden test)

| รายการ | บรรทัดโดยประมาณ | เหตุผล |
|---|---|---|
| `customOld` | 149 | แทนด้วย `custom` |
| `actorsEmoji` | 317 | ไม่มีผู้เรียก |
| `person`, `cat`, `drawChar` | 327–354 | ระบบตัวละครรุ่นแรก |
| `badge` | 355 | เรียกเฉพาะใน `frameOld` (`badge` ถูกแทนด้วย `gfx`) |
| `frameOld` | 723 | ไม่มีผู้เรียก; อ้าง `$('oBadge')` ที่ไม่มีใน DOM |
| `PRE`, `stylize`, `photobg`, `PH` | 525–538 | เส้นทางภาพถ่ายไม่มีตัวโหลด; `PH` ไม่เคยถูกเติม (ยังถูกอ่านในพรอมต์) |
| `promptText2`, `promptTextOld` | 835–863 | ไม่มีผู้เรียก |
| ตัวแปร `lastTx` ซ้ำซ้อนบางส่วน | 417 | หลังย้าย TTS เข้า `tts` |

**หมายเหตุ:** ก่อนลบให้ `grep` ชื่อทั้งหมดอีกครั้ง (ผมนับการอ้างอิงจากข้อความทั้งไฟล์และเห็นเพียงนิยามและการเรียกภายในโค้ดตายด้วยกันเอง) แล้วรัน AC-G2 ทุกครั้งหลังลบ

---

*รายงานนี้อิงจากการอ่านโค้ดทั้งไฟล์ และการรันชิ้นส่วนที่ตัดมาทดสอบใน Node 4 เรื่อง (R-01 การสืบทอดค่า, R-02 กรณี throw ใน normalize, R-14 regex ตัวเลข, R-15 ผลรวมเวลา) ยังไม่มีผลวัดบนเบราว์เซอร์จริง เกณฑ์ที่เป็นตัวเลขด้าน performance จึงอ้างอิง baseline ที่ต้องวัดใน AC-P0-3*
