# Coral Guide (coral.guide) – โครงสร้างเว็บและเนื้อหา

เอกสารนี้สรุปจากซอร์สของเว็บ <https://github.com/koenigderluegner/coral-island-guide> (fan-made guide ของเกม **Coral Island**, ผู้สร้าง Koenig)
ไม่ได้ดึงจากหน้าเว็บจริง เพราะ network ของ session บล็อก `coral.guide` ข้อมูลทั้งหมดมาจากไฟล์ JSON ที่เว็บใช้เอง
(`packages/guide/src/assets/live/database/`, อ้างอิงข้อความจาก `i18n/en.json`) เวอร์ชันข้อมูล: live v1.3-1247, app 0.30.0 (2026-06-18)

> หมายเหตุ: เว็บมี spoiler และข้อมูลบางส่วนที่ยังไม่ถูกใส่ในเกม ข้อความ/ข้อมูลเป็นของเกมและชุมชน ตรวจสอบสิทธิ์ก่อนนำไปเผยแพร่ต่อ

## Tech stack
- Angular (standalone components, lazy routes, SSR `server.ts`), Angular Material, Tailwind, ngsw (PWA), Nx + pnpm monorepo
- packages: `guide` (เว็บ), `data-types` (TypeScript types ของข้อมูล), `file-extraction` + `pak-files-parser` (ดึงข้อมูลจากไฟล์เกม), `util`
- รองรับ 10 ภาษา: en, de, es, fr, id, ja, ko, pt-BR, th-TH, zh-CN
- ข้อมูลผู้ใช้ (checklist, to-do, settings) เก็บใน local storage, import/export ได้
- มีโหมด Live / Beta (`/only-in-live`, `/only-in-beta`)

## Site map (เมนูหลัก 6 ส่วน + หน้าอื่น)

| เมนู | Route | หน้าย่อย / แท็บ | ข้อมูลใน repo |
|---|---|---|---|
| Home | `/` | ต้อนรับ + changelog | `CHANGELOG.md` |
| **Journal** | `/journal/...` | `produce/{crops, animalproducts, artisanproducts, ocean}`, `caught/{fish, insects, seacritters}`, `found/{artifacts, gems, fossils, scavangables}`, `notes/{letters, tornpages, treasuremap}`, `clothing/:tab`, `bestiary`, `achievements`, `achievements/:id` | [fish](data/fish.md) [bugs](data/bugs-and-insects.md) [ocean](data/ocean-critters.md) [bestiary](data/bestiary.md) [achievements](data/achievements.md) [mail](data/mail.md) [torn pages](data/torn-pages.md) |
| **Crafting** | `/crafting/...` | `inventory`, `cooking/{oven, blender, chefknife, fryingpan, skillet, grill, pot, seasoningset, ceramicbowl}`, `artisan/:machine` (20 เครื่อง เช่น barrel, beehouse, cheesepress, compostbin, dehydrator, dyemaker, furnace, keg, extractor, kiln, loom, mayonnaise, mill), `mixing` (underwater seeds) | [crafting](data/crafting-recipes.md) [cooking](data/cooking-recipes.md) [artisan](data/artisan-processing.md) |
| **People (NPCs)** | `/npcs/...` (`/people` redirect) | `overview`, `gifting`, `unlinked-images`, `:npcKey` (หน้า NPC รายคน: heart events, ของขวัญ, วันเกิด) | [npcs](data/npcs.md) |
| **Locations** | `/locations/...` | `lake-temple/:altar` (offerings 5 แท่น), ร้านค้า, เทศกาล, `merit-exchange`, `orchestra-zones`, `attractions` | [offerings](data/offerings.md) [shops](data/shops.md) [festivals](data/festivals.md) [attractions](data/attractions.md) |
| **My Guide** | `/my/...` | `to-do`, `museum-checklist/:tab`, `cooking-recipes-checklist/:utensil`, `offerings-checklist/:altar`, `orchestra-zones-checklist`, `clothing-checklist`, `settings`, dashboards (birthdays, museum fish/insects/ocean critters) | ใช้ข้อมูลชุดเดียวกับด้านบน |
| **Database** | `/database`, `/database/:itemId` | ค้นหา/กรองไอเทมทั้งหมด (5,169 รายการ) | [items](data/items.md) |
| อื่น ๆ | `/about`, `/only-in-live`, `/only-in-beta`, `**` = 404 | | |

### ร้านค้า (Locations > shops, 22 ร้าน)
general-store, blacksmith, lab, carpenter, bos, ranch, pet-shop, bens-caravan, socket-and-pan, beach-shack, concerned-monkey, merfolk-general-store, merfolk-oracle-tail-store, tidal-threads, underwater-ranch, taco-truck, ramen-shop, sales-cart-stall, furniture-store, white-flamingo, tavern, coffee
(ร้านที่มี item-upgrade/process: blacksmith, carpenter, lab, beach-shack)

### เทศกาล (8)
winter-fair, cherry-blossom, animal, beach-clean-up, harvest, spooky, new-year-eve, tree-planting

### Lake Temple altars
cropaltar และอีก 4 แท่น (รายละเอียดใน [offerings](data/offerings.md))

## หมวดหมู่ข้อมูล (จำนวนรายการ)

| ชุดข้อมูล | จำนวน | ไฟล์ md |
|---|---|---|
| Items (inventoryCategory: Decor 716, Outfit 667, Decoration 459, ArtisanProduct 431, Seeds 374, Wallpapers 355, Floors 345, Furniture 329, Consumable 180, ฯลฯ) | 5,169 | [items.md](data/items.md) |
| NPCs (CoralFolks 69, MerfolkKingdom 53, Baby 32, DomesticAnimals 31, OceanAnimals 13, GiantFolks 12, None 14) | 224 | [npcs.md](data/npcs.md) |
| Fish / Insects / Sea critters | 69 / 57 / 45 | [fish](data/fish.md), [bugs](data/bugs-and-insects.md), [ocean](data/ocean-critters.md) |
| Crops / Fruit trees / Fruit plants | 71 / 7 / 13 | [farming.md](data/farming.md) |
| Cooking recipes (9 utensils) / Crafting recipes / Artisan processing | – / 158 / 20 machines | [cooking](data/cooking-recipes.md), [crafting](data/crafting-recipes.md), [artisan](data/artisan-processing.md) |
| Shops | 22 ไฟล์ | [shops.md](data/shops.md) |
| Achievements / Mail / Torn pages / Treasure maps | 51 / 279 / 30 / 10 | [achievements](data/achievements.md), [mail](data/mail.md), [torn-pages](data/torn-pages.md) |
| Bestiary / Attractions / Animals | 23 / 10 / 18 | [bestiary](data/bestiary.md), [attractions](data/attractions.md), [animals](data/animals.md) |
| ข้อมูลที่ยังไม่ได้แตกเป็น md | heart-events (ต่อ NPC), gift-preferences (73 กลุ่ม), consumables (3,640 buff), museum/journal order, mastery unlocks, tag-based-items, pet-shop-adoptions, merit-exchange | อ่านจาก JSON ต้นทางโดยตรง |

## หมายเหตุสำหรับพัฒนาต่อ (React + MUI)
- ข้อความในข้อมูลเป็น key (เช่น `DT_InventoryItems.item_72030_name`, หรือ hash 32 หลัก) ต้อง lookup จาก `i18n/<lang>.json` (12,678 keys ใน en)
- ไอเทมอ้างกันด้วย `id` แบบ `item_72030`; รูปไอคอนอ้างด้วย `iconName` (อยู่ใน `packages/guide/src/assets`, repo ทั้งหมด ~9.8GB เพราะมี pak-assets)
- ไฟล์ shop ทุกร้านใช้ schema เดียวกัน (`allowedSeasons/Days/Weather`, `townRank`, `priceOverride`, `enabled`) → ทำ component เดียวได้
- ใน items.json มี category ที่ยังไม่ map (hash/`TBD`/`Test`) ซึ่งรวมไว้เป็น `Other(unmapped)` ใน md
- Types อยู่ที่ `packages/data-types/src`, ลิสต์ route ทั้งหมดสร้างจาก `static-routes-list-generator.ts` (271 routes ที่ prerender)
- ไฟล์ md ใน `data/` สร้างด้วยสคริปต์ (ตัดคำอธิบายยาวเหลือ ~140–300 ตัวอักษร ส่วน festivals เป็น excerpt) ถ้าต้องการข้อมูลเต็มให้ใช้ JSON ต้นทาง
