# Blender addons library

**Before downloading anything, check the [Addons database in Notion](https://app.notion.com/p/a899ed69265744e7855528f65d002731) first** (under the `Blender` page). That's the real answer to "do I already have this, even disabled/archived somewhere?" — `Status` covers exactly that: `Wishlist` → `Backlog` → `Testing` → `Using` → `Archived` → `Avoid`. An addon you disabled and forgot about should show up there as `Archived` or `Using` with `Owned: Yes`, not silently vanish — that's the whole reason it exists.

This local folder is the technical companion, not a replacement: it holds
the actual downloaded files (Notion isn't a great place for large zips) and
the machine-local detail Notion doesn't track — module names, exact file
paths, and which Blender *profile* on this computer has each one. Profiles
under `~/apps/blender/profiles/<profile>/` hold the *installed* copies (in
`portable/scripts/addons/` or as a registered Asset Library); this folder
holds the *originals* you'd reinstall from — useful for spinning up a new
isolated profile, rolling back a version, or working offline. `manifest.json`
entries cross-link to their Notion page via `notion_url`.

See [`../scripts/setup-blender-profile.sh`](../scripts/setup-blender-profile.sh)
for creating isolated per-plugin Blender profiles in the first place.

## Layout

```
addons-library/
├── manifest.json              # registry: type, notion_url, versions, which local profiles have each one
└── <addon-slug>/
    └── source/
        ├── <addon-slug>-1.0.0.zip   (or .py for single-file addons)
        └── <addon-slug>-1.1.0.zip   # keep old versions on upgrade, don't overwrite
```

## Adding a new addon

0. **Check Notion first.** Search the Addons database by name — if it's
   already there (even `Archived`/disabled), you have it; go install/enable
   it in the profile you need instead of re-downloading.
1. `mkdir -p <addon-slug>/source` and drop the downloaded `.zip`/`.py` in
   there, named `<addon-slug>-<version>.<ext>`.
2. Create/update its Notion page (`Name`, `Website`, `Category`, `Cost type`,
   `Owned: Yes`, `Store`, `Status`, `Last checked`).
3. Add an entry to `manifest.json`: `type` (`"addon"` or `"asset-library"`),
   `notion_url` (link back to the page from step 2), `display_name`,
   `source_url`, `install_method`, `module` (the Python module name Blender
   registers it under — needed to enable it headlessly via
   `bpy.ops.preferences.addon_enable(module=...)`; `null` for asset packs,
   which have no module), `current_version`, the `versions` map, and which
   profiles it's `installed_in`.
4. Install it into a profile from its portable addons folder, e.g.:
   ```bash
   PROFILE=claude-mcp
   ADDON=<addon-slug>
   VERSION=1.0.0
   cp addons-library/$ADDON/source/$ADDON-$VERSION.py \
      ~/apps/blender/profiles/$PROFILE/portable/scripts/addons/
   ~/apps/blender/profiles/$PROFILE/blender -b --python-expr "
   import bpy
   bpy.ops.preferences.addon_enable(module='<module_name>')
   bpy.ops.wm.save_userpref()
   "
   ```
   (For zipped/multi-file addons, extract into that same `addons/` folder
   instead of copying a single `.py`.)
5. Update `manifest.json`'s `installed_in` for that addon (or run
   `scripts/addon-status.sh set <slug> <profile> enabled <version>`), and
   flip its Notion `Status` to `Using`.

## Upgrading an addon

Add the new version file alongside the old one (don't delete the old one),
bump `current_version`, and repeat the install steps above into whichever
profiles you want upgraded. Profiles you don't touch keep running the old
version — that's the isolation this whole setup is for.

Some stores (SuperHive order pages are the common case) ship a separate file
per *Blender* version, not just per addon version — e.g. Speedsculpt's page
lists `speedsculpt_v_0_3_9_blender_5_2_0.zip`, `..._0_3_8_blender_5_1_0.zip`,
etc. Label the `versions` map key with both, addon version first:
`"0.3.9 (Blender 5.2.0)"`.

Only keep the build that matches the Blender version your profiles actually
run (5.2.0 LTS, as of this writing) — don't hoard every historical
per-Blender-version build. The store's own order/download page is already
your permanent archive of every version ever shipped; re-download from there
if you ever need an older one. The "don't delete the old one" rule above is
for addon-version upgrades you might want to roll back within the *same*
Blender version, not for keeping years of Blender-version-specific builds
around locally.

## Installing from SuperHive (Extensions Platform)

SuperHive extensions install through Blender's own Extensions system
(Preferences → Get Extensions), not the manual `addons/` folder above.
Deliberately **don't** point its "Custom Directory" option at this library —
each profile already resolves its extensions to its own isolated,
version-specific folder by default (e.g.
`profiles/<profile>/portable/extensions/user_default/`), and Blender manages
that folder with its own internal manifest. Redirecting it here would either
break that bookkeeping or, worse, share one folder across profiles and
defeat the whole point of this setup.

Instead, catalog it the same way as everything else:

1. Install the extension normally into the target profile — drag-and-drop
   the downloaded file onto that profile's running Blender, or
   Preferences → Get Extensions → Install from Disk. It lands in that
   profile's own isolated extensions folder, untouched by this library.
2. Copy the downloaded `.zip` into `<addon-slug>/source/` here, same as any
   other addon.
3. Add/update its `manifest.json` entry — set `install_method` to
   `"SuperHive (Extensions)"` so it's clear at a glance it's not a legacy
   `scripts/addons` install.

## Asset packs (not addons)

Some SuperHive/Gumroad/etc. downloads are `.blend` asset files (rigs,
models, materials), not code — no `bl_info`, no module, nothing to
enable/disable. They're `type: "asset-library"` in `manifest.json`. Install
means registering the folder as a Blender **Asset Library**
(Preferences → File Paths → Asset Libraries → point at
`<addon-slug>/source/` or an extracted copy of it), not touching
`scripts/addons`. Catalog it the same as any addon otherwise: Notion page
first, file copied into `<slug>/source/`, manifest entry with `module: null`.

## Currently tracked

| Item | Type | Installed in | Version | Notion |
|---|---|---|---|---|
| [Blender MCP](https://github.com/ahujasid/blender-mcp) | addon | `claude-mcp` (enabled) | 1.6 | [page](https://app.notion.com/p/3d386ab69e848198bc5edcf4af576a65) |
| [Adjustable Mannequin](https://superhivemarket.com/products/adjustable-mannequin) | asset-library | `claude-mcp`, `photo-pose` (registered) | 1.3 | [page](https://app.notion.com/p/3d386ab69e848186bb19e072190faeed) |
| Photo Pose (MediaPipe -> Rigify) | addon, custom-built | `photo-pose` (enabled) | 0.4.0 | [page](https://app.notion.com/p/3d386ab69e8481148911f4f895a7e9d0) |
| [FlowPose](https://github.com/Aar-os01/Flowpose-Addon-for-blender) | addon | `photo-pose`, `claude-mcp` (enabled) | 1.6 | [page](https://app.notion.com/p/3ee86ab69e848139a175e1557d39315f) |
| [Home Builder 5](https://extensions.blender.org/add-ons/home-builder-5/) | addon (Extension) | `claude-mcp` (enabled) | 5.2.4 | [page](https://app.notion.com/p/3f586ab69e84814ba62ad9dbaec099be) |
| [Import Images as Planes](https://extensions.blender.org/add-ons/import-images-as-planes/versions) | core Blender 4.2+ (no addon needed) | all profiles, built-in | n/a | [page](https://app.notion.com/p/3d586ab69e8481d094f4d099eb5f87c9) |
| Resin2FDM Advanced | addon (Extension) | `resin2fdm` (enabled) | 1.6.1 | [page](https://app.notion.com/p/3e386ab69e8481cca87ede7ba7262e78) |
| [3MF Import/Export](https://extensions.blender.org/add-ons/threemf-io/) | addon (Extension) | `resin2fdm` (enabled) | 2.7.7 | [page](https://app.notion.com/p/3e386ab69e8481809c22eacd9f25019f) |
| [Botaniq](https://superhivemarket.com/products/botaniq-trees) | asset-library | not installed yet | HB2024 | [page](https://app.notion.com/p/29986ab69e8480cb8f08fac53a386090) |
| True-Terrain 5 | addon | not installed yet | 5.0.3 | [page](https://app.notion.com/p/29986ab69e8480878f0fc113041460ec) |
| True-Sky | addon | not installed yet | 2.2.4 | [page](https://app.notion.com/p/29986ab69e8480bdbe52c0fe845df55e) |
| [True-VFX Undergrowth](https://truevfx.gumroad.com/l/undergrowth) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/3f586ab69e8481ff94bec79f122dc184) |
| [BlendShop](https://superhivemarket.com/products/blendshop-compositing-tool-for-blender) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e848098b022cc790cf5d616) |
| FloraPaint | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e8480b59cafe273dd43d6c2) |
| [RealCloud Generator - VDB Clouds Library](https://superhivemarket.com/products/realcloud) | addon | not installed yet | 1.0.4 | [page](https://app.notion.com/p/29986ab69e84807998ebe25b2717d602) |
| [Shaders Plus (SMOUSE)](https://superhivemarket.com/products/shaders-plus) | asset-library | not installed yet | v3 + Modules (BDay Gift) | [page](https://app.notion.com/p/29986ab69e8480e58fadc86823df805b) |
| [Gobos Plus (SMOUSE)](https://superhivemarket.com/products/gobos-plus) | asset-library | not installed yet | v2 (HB) | [page](https://app.notion.com/p/29986ab69e8480cb8b84e46f085a73fb) |
| Underwater Caustics Shader (SMOUSE) | asset-library | not installed yet | v2 (HB) | [page](https://app.notion.com/p/29986ab69e848024bdc4f2b78623c030) |
| [Clay Doh](https://blendermarket.com/products/clay-doh) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29286ab69e8480cca551fa3cf9cc55f0) |
| [FlipFluids](https://github.com/rlguy/Blender-FLIP-Fluids) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e848043a5d2d687f9795b5d) |
| [Procedural Alleys](https://superhivemarket.com/products/procedural-alleys) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e84804ba50df2af457b8671) |
| Summer Pack - People Set | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e848018b91df5a591e094ed) |
| [Procedural Signs](https://superhivemarket.com/products/procedural-signs) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e8480739dd1f84bce71ac5b) |
| [Spock](https://superhivemarket.com/products/spock) | addon | not installed yet | 1.1.2 (Win) / 1.1.1 (Mac/Linux) | [page](https://app.notion.com/p/29986ab69e84809dbddcc944688488ec) |
| [SciFiFlex](https://superhivemarket.com/products/sci-fi-flex) | addon | not installed yet | 2.4.7 | [page](https://app.notion.com/p/29986ab69e84801f9a2de9765d194ae5) |
| [Kit Ops Pro](https://superhivemarket.com/products/kit-ops-3-pro) | addon | not installed yet | 3.0.24 | [page](https://app.notion.com/p/29986ab69e8480b68bb8fcb180fa8921) |
| [Traffiq](https://superhivemarket.com/products/car-library-traffiq-vehicles-for-blender) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e848024accae74cfbc83b58) |
| CloudScapes Geo-Scatter Biomes Pack | asset-library (dependency: Geo-Scatter) | not installed yet | 1.0 | [page](https://app.notion.com/p/3f586ab69e8481a9b1a3fdbe9ba3509c) |
| [RetopoFlow](https://blendermarket.com/products/retopoflow) | addon | not installed yet | 3.4.4 (also keeping 3.4.3) | [page](https://app.notion.com/p/29986ab69e84802caeccd843febecb64) |
| [RetopoFlow 4 (Alpha)](https://superhivemarket.com/products/retopoflow) | addon | not installed yet | 4.0.0 | [page](https://app.notion.com/p/29286ab69e848009b4c9cfd9bef457ce) |
| [BoxCutter](https://superhivemarket.com/products/box-cutter) | addon | not installed yet | 720_5b | [page](https://app.notion.com/p/29986ab69e848041972fdd00d923cfa8) |
| [Cablerator](https://superhivemarket.com/products/cablerator) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e8480a792bfdbb5a27e3ea3) |
| [CloudScapes](https://superhivemarket.com/products/cloudscapes-v2-hyper-realistic-vdb-clouds-explosions) | addon | not installed yet | 2.0 | [page](https://app.notion.com/p/29986ab69e84808db887f3cd92c8cd31) |
| [Hard Ops](https://superhivemarket.com/products/hard-ops) | addon | not installed yet | 988_5b | [page](https://app.notion.com/p/29986ab69e848041972fdd00d923cfa8) |
| [Human Generator](https://superhivemarket.com/products/human-generator) | addon | not installed yet | HB | [page](https://app.notion.com/p/29986ab69e8480cca6c2f0b0b672e449) |
| [Malfunktion](https://superhivemarket.com/products/malfunktion) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e8480afa5aec6db2c4a1520) |
| [Physical Open Waters](https://superhivemarket.com/products/physical-open-waters) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e848019b6c7dbca6db864b7) |
| [Physical Celestial Objects](https://superhivemarket.com/products/physical-celestial-objects) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e8480dc9ae4f6030eb32363) |
| [Realistic Touch](https://superhivemarket.com/products/realistic-touch-surface-imperfection) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/bb401917c365497aaebc520a2fd60f95) |
| [Real-time Materials](https://blendermarket.com/products/realtime-materials-for-blender) | asset-library | not installed yet | 2.1.6 | [page](https://app.notion.com/p/12f86ab69e848019b7a3cad4e87dc1fb) |
| [Conform Object](https://www.blendermarket.com/products/conform-object) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29986ab69e8480cfacd9f62f0d31f55e) |
| [Perspective Plotter](https://blendermarket.com/products/perspective-plotter) | addon | not installed yet | 1.3.2 | [page](https://app.notion.com/p/29986ab69e848014baffe1792796688d) |
| [Speedsculpt](https://superhivemarket.com/creators/pitiwazou) | addon | not installed yet | 0.3.9 (Blender 5.2.0) | [page](https://app.notion.com/p/12f86ab69e84809aa32bcad04ce41d27) |
| PBR Materials 1000+ | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/3f586ab69e8481798f3ef53232514db2) |
| Dermis | addon | not installed yet | 1.3 | [page](https://app.notion.com/p/12f86ab69e848010b958eb12706f46d0) |
| [Simply Cloth Studio](https://superhivemarket.com/creators/vjaceslavt) | addon + asset-library | not installed yet | 1.5.5 (Blender 5.0) | [page](https://app.notion.com/p/12f86ab69e848033a5cfc5a97e0a06f6) |
| [Dynamic Pose Library Free](https://gumroad.com/d/bac0ea58268eae05479f6b548c9b03e1) | asset-library | not useful -- locked to Human Generator's own pose system | v6.03 | [page](https://app.notion.com/p/3f386ab69e848116b499dba78969fbfc) |
| [The Plant Library](https://blendermarket.com/products/the-plant-library) | asset-library | not installed yet | unversioned | [page](https://app.notion.com/p/a8facdcabf78436e9250591c49318be2) |
| [Blenrig 6](https://blendermarket.com/products/blenrig) | addon | not installed yet | 5.0.0 | [page](https://app.notion.com/p/12f86ab69e8480daafc9cf59b75921fa) |
| [Sanctus Library](https://blendermarket.com/products/sanctus-library-addon---procedural-shaders-collection-for-blender) | addon | not installed yet | 3.4.20 (Commercial) | [page](https://app.notion.com/p/12f86ab69e8480f38d54ddbc1a45f8e1) |
| [Minecraft World Generator](https://superhivemarket.com/products/free-addon-mc-world-generator) | addon | not installed yet | 1.0.0 | [page](https://app.notion.com/p/3d586ab69e848187b596d687e2fd395a) |
| [Ultimate IMM Anatomy Primitives](https://georgezaky108.gumroad.com/l/rwewi) | asset-library | not installed yet (reference only) | 2.0 | [page](https://app.notion.com/p/3f486ab69e848197b65dc8403c671aa7) |
| [Quick Brush Selector](https://superhivemarket.com/products/quick-brush-selector) | addon | not installed yet | 3.0.0 | [page](https://app.notion.com/p/3f686ab69e848137be6beb96a300a11c) |
| [Free Spring Grass Set](https://superhivemarket.com/products/free-spring-grass-set) | asset-library | not installed yet | unversioned | [page](https://app.notion.com/p/3f686ab69e8481259331e2b5b1bd0d2c) |
| [Bumarin for Sculptor](https://superhivemarket.com/products/bumarin-for-sculptor) | addon | not installed yet | 1.0 (Blender 4.3) | [page](https://app.notion.com/p/3f686ab69e84813e9aaee96e6dcae705) |
| [Sculpture Pencils](https://superhivemarket.com/products/sculpture-pencils) | addon | not useful -- no downloadable files available | n/a | [page](https://app.notion.com/p/3f686ab69e8481f3a100fcebd376811e) |
| [Sculpt+Paint Wheel](https://superhivemarket.com/products/sculpt-paint-wheel) | addon | not useful -- confirmed incompatible with Blender 4.3+/5.2.0 | 3.0.4_b4.2_hotfix | [page](https://app.notion.com/p/3d986ab69e8481e1b465efabf0ee5953) |
| [Cracks and Damage Maker](https://superhivemarket.com/products/cracks-and-damage-maker-add-on) | addon | not installed yet | 1.4.0 | [page](https://app.notion.com/p/3d986ab69e8481549ed1d97f81256fcc) |
| [Smartify Nodes](https://blendermarket.com/products/smartify-nodes) | addon | not installed yet | 1.04 | [page](https://app.notion.com/p/3d586ab69e8481ec92c4da4f98fd35fa) |
| [Tradigitalist Toolbag](https://bazaar.blendernation.com/listing/tradigitalist-toolbag/) | asset-library | not installed yet | bundle (unversioned) | [page](https://app.notion.com/p/3f686ab69e848153a35ffff677ec2fce) |
| [Alt Tab Lighting](https://blendermarket.com/products/alt-tab-lighting) | asset-library | not installed yet | 1.1.0 | [page](https://app.notion.com/p/12f86ab69e848005b4d0ec8512a73484) |
| [Real Water - Waters Shader](https://blendermarket.com/products/realwater1) | asset-library | not installed yet | 2.0.3 | [page](https://app.notion.com/p/12f86ab69e84801cafc3d9d1f6a2a15c) |
| [Alt Tab Easy Fog](https://blendermarket.com/products/alt-tab-easy-fog) | asset-library | not installed yet | Blender 3.4+ | [page](https://app.notion.com/p/12f86ab69e84808da53ed02e35c66b23) |
| [BagaPie V11](https://extensions.blender.org/add-ons/bagapie/) | addon | not installed yet | 11.0.11 (Blender 5.2) | [page](https://app.notion.com/p/a9a6fff6f01c4afc8493badee9726fa5) |
| [Better Lighting V2](https://blendermarket.com/products/better-lighting/) | asset-library | not installed yet | 2.1.0 | [page](https://app.notion.com/p/3d586ab69e848141976deccdf22be1aa) |
| [Divine Cut \| Smart Cloth Generator V4](https://superhivemarket.com/products/divine-cut-smart-cloth-generator) | addon | not installed yet (expired trial) | 2.5.5 (Vanilla Trial, expired) | [page](https://app.notion.com/p/3d986ab69e848169a94ac12955a1ab89) |
| [Auto-Rig Pro](https://superhivemarket.com/products/auto-rig-pro) | addon | not installed yet | 3.70.38 | [page](https://app.notion.com/p/3d586ab69e8481acae56cf0bfe4670b5) |
| [PETZ Library](https://superhivemarket.com/products/petz-library) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/3f686ab69e848174bb48d30808fd433f) |
| [Ultimate Basemesh Collection](https://superhivemarket.com/products/ultimate-basemesh) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e8480fabb79ca1460f3bb95) |
| [Quick Baker](https://superhivemarket.com/products/quick-baker) | addon | not installed yet | 2.7.1 | [page](https://app.notion.com/p/29a86ab69e8480c0a0d9d009f553017f) |
| [Auto Bake 2](https://superhivemarket.com/products/autobake) | addon | not installed yet | 2 | [page](https://app.notion.com/p/29a86ab69e8480838cede2225c141f03) |
| [Action Commander - Game Animation Tool](https://superhivemarket.com/products/action-commander) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e848035865fea709796c880) |
| [Bakemaster](https://superhivemarket.com/products/bakemaster) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e8480cf83baedc14908ce4b) |
| [Chat Companion](https://superhivemarket.com/products/chat-companion) | addon | not installed yet | 1.5.14 | [page](https://app.notion.com/p/3f686ab69e8481358ca7e31910cbfa9d) |
| [Population - Human Crowd Simulation](https://superhivemarket.com/products/population) | addon | not installed yet (compat unverified on 5.2) | 1.1.2 (Blender 4.1) | [page](https://app.notion.com/p/3b4a9aac209448a2a322922458023e40) |
| [Cubicity: Creating & Utilizing Asset Libraries With Blender](https://superhivemarket.com/products/cubicity) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/3f786ab69e84812e9daece9d44794748) |
| [Better FBX Importer & Exporter](https://superhivemarket.com/products/better-fbx-importer--exporter) | addon | not installed yet | unknown | [page](https://app.notion.com/p/29a86ab69e8480418d9be169e63f2dd2) |
| [Blender To Godot 4 Pipeline Addon](https://superhivemarket.com/products/blender-godot-pipeline-addon) | addon | not installed yet | unknown | [page](https://app.notion.com/p/29a86ab69e8480859d4df5ee17e0b887) |
| [Blender To Unity](https://superhivemarket.com/products/blender-to-unity) | addon | not installed yet | unknown | [page](https://app.notion.com/p/29a86ab69e8480a89535e4de8cf9021b) |
| [Blender to Unreal](https://superhivemarket.com/products/blender-to-unreal) | addon | not installed yet | 2.5.3 | [page](https://app.notion.com/p/29a86ab69e84806eb2d5d92defcb4808) |
| [Empire of the Sands](https://superhivemarket.com/products/empire-of-the-sands---desert-structures-kitbash-greeble-assets) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e8480a8a7e3f447fb6ab86c) |
| [Faceit : Facial Expressions And Performance Capture](https://superhivemarket.com/products/faceit) | addon | not installed yet | unknown | [page](https://app.notion.com/p/29a86ab69e8480338104cef01ec3af8c) |
| [FastTrackTutorials Hero Asset Course](https://superhivemarket.com/products/hero-asset-creation---in-depth-tutorial-course) | asset-library | not installed yet (course complete, all 3 files present) | 1.0 | [page](https://app.notion.com/p/0b57dc73c90f4df78f8209a414bec275) |
| [Fantasy Cemetery 2](https://superhivemarket.com/products/fantasy-cemetery-2) | asset-library | not installed yet (environment asset pack) | 1.0 | [page](https://app.notion.com/p/29a86ab69e848019b435c1abdad69292) |
| [Game and Render Ready Rock Pack](https://superhivemarket.com/products/realistic-low-poly-rock-pack) | asset-library | not installed yet (environment asset pack) | 1.0 | [page](https://app.notion.com/p/29a86ab69e848033913ce12260c62bb5) |
| [RIG: Blender's Character Rigging Playbook](https://superhivemarket.com/products/rig-blenders-character-rigging-playbook) | asset-library | not installed yet (video training course) | 1.0 | [page](https://app.notion.com/p/2f575e93fbe44925a1c20b2b652935bf) |
| [Full Environment Creation In UE5 & Blender - In-Depth Course](https://superhivemarket.com/products/western-game-environment-course) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/06a7c5a75c6b4c9d94ec9bc720411fbb) |
| [Game Rig Tools (Addon) - Game-Ready Rigs In Seconds](https://superhivemarket.com/products/game-rig-tools-blender-addon-game-rigs) | addon | not installed yet | unknown | [page](https://app.notion.com/p/29a86ab69e84805aac57c483e1929dc3) |
| [Massive 240-Piece Modular Mega Pack](https://superhivemarket.com/products/blender-massive-250-piece-modular-mega-pack-for---kitbash-immersive-castles-and-towns-for-game-dev-animation-and-archviz) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e8480188eb3efbbd6ada2a9) |
| [Sci-Fi Modular Panel Kit Vol -2](https://superhivemarket.com/products/sci-fi-modular-panel-kit-vol--2) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e848026ae83d7739646f613) |
| [Trash Kit - 3D Assetkit](https://superhivemarket.com/products/trashkit---3d-assetkit) | asset-library | not installed yet (all 4 volumes present, not yet extracted) | 1.0 | [page](https://app.notion.com/p/29a86ab69e848045ae8afcd9cb778c77) |
| [Tread: Hard Surface Asset Creation For Video Games](https://superhivemarket.com/products/tread-hard-surface-asset-creation-for-video-games) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/17355de780224b55ae065dd5bfa35252) |
| [Ultimate Game Asset Course With Blender](https://superhivemarket.com/products/blender-game-asset-course) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/82dedf5504774f20bc9d27b065083075) |
| [Uvpackmaster 3](https://superhivemarket.com/products/uvpackmaster) | addon | not installed yet | unknown | [page](https://app.notion.com/p/29a86ab69e84803786bee0512ec88b9a) |
| [Blender Game Development Tutorial](https://superhivemarket.com/products/blender-game-development-tutorial) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/77d2431f0b57455c8ef3e4a613c985ec) |
| [Blender & Unreal Engine Massive Stylized Sci-Fi Crate Pack](https://superhivemarket.com/products/blender--unreal-engine-massive-stylized-sci-fi-crate-pack) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e848012b2b8f043d5c05929) |
| [Stylized Low Poly Food Pack 01](https://superhivemarket.com/products/stylized-low-poly-food-pack-01) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e84800292f1cf8e05488bd8) |
| [REV Model Muscle Car](https://superhivemarket.com/products/rev-model-muscle-car) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/509ec371e9ac41f3b37bf226a790a178) |
| [Rigging Transforming Rifle](https://superhivemarket.com/products/rigging-transforming-rifle) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/88a8eeb9b8314524941bc5ee440a3530) |
| [Scifi Station Vol.2 - Stylized Low Poly 3D Art Asset Pack](https://superhivemarket.com/products/scifi-station-vol-2) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e8480cea4d9eddfce9b3c75) |
| [Family - Cartoon Modular Fully Rigged 3D Family](https://superhivemarket.com/products/family---cartoon-modular-fully-rigged-3d-family) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e84805ea2acf392f192ccbf) |
| [Uv Flow](https://superhivemarket.com/products/uv-flow) | addon | not installed yet | unknown | [page](https://app.notion.com/p/29a86ab69e8480c39be7fbe3f327215a) |
| [50 Sword Base Mesh - Vol 05 ( Game Ready )](https://superhivemarket.com/products/swordbase) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e8480498d2fe1e649e8dfeb) |
| [AK-47 Game Ready](https://superhivemarket.com/products/ak-47-game-ready) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e8480e9a570e3f9887436e0) |
| [Animating First Person Character Weapons in Blender](https://superhivemarket.com/products/animating-first-person-character-weapons-in-blender) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/a7296afe4dc14662b11fb390ebf55d17) |
| [Blender Animation Bootcamp](https://superhivemarket.com/products/blender-animation-bootcamp) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/97230fb947774809a75183ed09db0f56) |
| [Cairo City Cyberpunk Streets - Animated Neon Signs - Day And Night - Egyptian Cyberpunk](https://superhivemarket.com/products/egyptian-cyberpunk) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e848016a491f2cf292026c3) |
| [Modeling A Robot Game Character With Blender](https://superhivemarket.com/products/modeling-a-robot-game-character-with-blender) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/2c780bcb3ad649bab172a3110bea1059) |
| [Outpost - Medieval Military Buildings 3D Kitbash Pack](https://superhivemarket.com/products/outpost---medieval-military-buildings-3d-kitbash-pack-) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e848054b173d21c07d1ab64) |
| [Scattershot - Pbr Texture Bombing For Blender](https://superhivemarket.com/products/scattershot---procedural-image-texture-scattering--tiling-with-voronoi) | asset-library | not installed yet | 1-12 | [page](https://app.notion.com/p/29a86ab69e8480c48b42e67f6473f747) |
| [Sci-Fi Modular Asset Pack](https://superhivemarket.com/products/sfy-sci-fi-modular-asset-pack) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e84809d8e39f65262ddcee5) |
| [Stylized Castle Ruins Asset Pack](https://superhivemarket.com/products/stylized-castle-ruins-asset-pack) | asset-library | not installed yet | 1.0 | [page](https://app.notion.com/p/29a86ab69e8480fca604db2c917302f8) |
| [BrushFlow - 3D Normal Map Painting](https://blendermarket.com/products/brushflow) | asset-library | not installed yet | unversioned | [page](https://app.notion.com/p/944921b226b24179b2f22738724d7d44) |
| [AutoUV](https://blendermarket.com/products/free-addon-autouv) | addon | not installed yet | 1.0.1 | [page](https://app.notion.com/p/3d586ab69e84813992f1d3ac52d9371b) |
| [Quick Turnaround Camera](https://superhivemarket.com/products/quick-turnaround-camera-addons) | addon | not installed yet | V4 | [page](https://app.notion.com/p/27086ab69e8480638453ce05b3d1d8fc) |
| [Ultimate Character Kit](https://superhivemarket.com/products/ultimate-character-kit) | asset-library | not installed yet | unversioned | [page](https://app.notion.com/p/27086ab69e8480ad8c52dc607fcdcd12) |
| [Simple Tabs](https://blendermarket.com/products/simple-tabs) | addon | not installed yet | 1.2.5 | [page](https://app.notion.com/p/3d586ab69e848103aa6bc2ac3ad08bda) |
| [Simplicage](https://superhivemarket.com/products/simplicage) | addon | not yet downloaded | n/a | [page](https://app.notion.com/p/27086ab69e848066985cfc2fdd7e52f7) |
| [Level Of Detail Generator \| LODs Maker](https://blendermarket.com/products/lods-maker--create-level-of-details-in-1-click/) | addon | not installed yet | v4 | [page](https://app.notion.com/p/503462de77aa49f5b0ded09862189aa9) |
| [Printlab \| Your Guide to 3D Printing With Blender and a Prusa Printer](https://blendermarket.com/products/printlab) | asset-library | not installed yet | 01 | [page](https://app.notion.com/p/3f786ab69e84813ba611ed4581179569) |
| Furnishing 1.4 - Furniture Pack - Ultimate Furniture Collection (no source URL given) | asset-library | not installed yet | 1.4 | [page](https://app.notion.com/p/3f786ab69e84816a99d6cd1916407997) |
| ZenShaders Free Library (no source URL given) | asset-library | not installed yet | unversioned | [page](https://app.notion.com/p/3f786ab69e84816a8e83e3a7438fb4da) |
| Lazy Animation (no source URL given) | addon | not installed yet | 1.0 | [page](https://app.notion.com/p/3f786ab69e848138b2cec6eae876be43) |
| Sanctus Vault (no source URL given) | asset-library | not installed yet | 1.2.0 | [page](https://app.notion.com/p/3f786ab69e8481cf82a7d85f995d48d6) |
| [Kin Kollection - Procedural Clay Materials](https://superhivemarket.com/products/kin-klei) | asset-library | not installed yet | unversioned | [page](https://app.notion.com/p/2fe86ab69e8480459706f69b2bdf6e13) |
| [Customisable Female Basemesh](https://superhivemarket.com/products/customisable-female-basemesh) | asset-library | not installed yet | 2.1.2 | [page](https://app.notion.com/p/2fe86ab69e8480218726d78488a12218) |
| Casey_Sheep Free Bundle (no source URL given) | asset-library | not installed yet | unversioned | [page](https://app.notion.com/p/3f786ab69e8481788920e50784fda87e) |
