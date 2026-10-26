# Openly — Worklog

---
Task ID: 1
Agent: Main (orchestrator)
Task: Fondasi proyek Openly (audit, schema, theme, logo, lib, API, public views)

Work Log:
- Audit proyek: Next.js 16 + TS + Tailwind 4 + shadcn/ui + Prisma/SQLite. Hanya route `/` boleh terlihat user → single-page app dengan hash-based router (Zustand store).
- Adaptasi requirement Supabase → Prisma + SQLite (lingkungan hanya mendukung ini). Storage file lokal di `storage/` dengan akses terautentikasi via `/api/files/[id]`.
- Prisma schema lengkap: User, CustomerProfile, Service, ServicePackage, Order, OrderStatusHistory, OrderAttachment, OrderFile, ContentSection, SiteSetting, Notification, PushSubscription.
- Theme cobalt blue di globals.css (oklch), pola geometris, glass, shimmer, reduced-motion.
- Logo Openly (SVG wordmark + mark) di public/openly-logo.svg & openly-mark.svg.
- Lib: auth (bcrypt+JWT), format (IDR/date/phone mask), storage (file path+validasi), queue (order number + queue number + token), content (getContent/setContent caching), session (getSession/requireAdmin), site (whatsapp/address constants), types (status labels/colors/steps), icons (lucide map).
- Stores: router (hash-based view switching), ui (mobile nav + tts toggle).
- Providers: ThemeProvider + Sonner Toaster.
- layout.tsx: metadata lengkap SEO + OG, viewport, manifest, font Geist.
- Seed script (scripts/seed.ts): admin user (admin@openly.id/admin123), site settings, content sections (hero/advantages/services_summary/order_flow/cooperation/owner/faq/contact), 6 graphic services + 5 website services + packages.
- Public API routes: /api/services, /api/packages, /api/content, /api/auth/{login,logout,me}, /api/orders/create (multipart upload + validasi), /api/orders/track (token), /api/orders/[id], /api/files/[id] (authenticated download), /api/portal, /api/notifications.
- Admin API routes: /api/admin/stats, /api/admin/orders (list+PATCH status), /api/admin/services (CRUD), /api/admin/packages (CRUD), /api/admin/customers (list+invite+PATCH), /api/admin/files (upload/list/delete), /api/admin/content (GET+PATCH), /api/admin/settings (GET+PATCH+POST), /api/admin/notifications (GET+POST broadcast).
- Shared components: header (desktop nav + mobile drawer + admin/login buttons), footer, reveal (framer scroll reveal), section-heading, wa-button, service-card, package-card, package-detail-modal, status-timeline, file-uploader (preview+drag), service-explorer.
- Public views built: beranda (hero+advantages+services summary+order flow+cooperation+owner+FAQ+CTA WhatsApp), jasa-desain, jasa-website, pricelist (tabs+modal), cara-pemesanan, tentang, kontak, login, order (graphic+website forms + confirm modal), confirmation, track (token), portal (customer cooperation dashboard).
- page.tsx shell: hash router + view switch + sticky footer layout.

Stage Summary:
- Foundation + all public views + all admin APIs DONE and wired.
- Admin UI (shell + 14 pages) is the next big chunk — delegated to subagent.
- Notifications (in-app bell + toast + push SW + TTS) pending.
- Self-verification (lint + dev + agent browser) pending.

Key conventions for subagents:
- Theme: cobalt blue primary (oklch 0.55 0.22 258), use `openly-gradient`, `openly-text-gradient`, `openly-pattern`, `openly-pattern-mesh`, `openly-scroll`.
- Router: `useRouter` from `@/store/router` with `navigate(view, params)`. Views: beranda|jasa-desain|jasa-website|pricelist|cara-pemesanan|tentang|kontak|login|order|track|confirmation|portal|admin.
- Auth: `useSession()` from `@/hooks/use-session` (client). Admin guard: redirect non-admin to login.
- DB: `import { db } from '@/lib/db'`. Session server: `getSession()`, `requireAdmin()` from `@/lib/session`.
- Toast: `import { toast } from 'sonner'`.
- Status labels/colors/steps: `import { ORDER_STATUSES, ORDER_STATUS_LABEL, ORDER_STATUS_COLOR, ORDER_STATUS_STEP } from '@/lib/types'`.
- Admin endpoints under `/api/admin/*` (see list above). All require admin cookie session.
- Sticky footer: root `<div className="flex min-h-screen flex-col">` with `<main className="flex-1">`.
- Only `/` route visible — all "pages" are client views switched by the router store.

---
Task ID: 9-13
Agent: full-stack-developer (admin UI)
Task: Build complete admin dashboard (shell + 14 pages)

Work Log:
- Read worklog Task 1 to understand foundation (theme, router, auth, APIs, status types).
- Verified existing admin API contracts (`/api/admin/stats|orders|services|packages|customers|files|content|settings|notifications`) and `/api/orders/[id]`.
- Created `src/components/admin/admin-view.tsx` shell:
  - Auth guard via `useSession()` → centered skeleton while loading, login card (POST `/api/auth/login` + reload) if not admin, full dashboard UI if admin.
  - Layout: `lg:grid lg:grid-cols-[260px_1fr]` with sticky sidebar (desktop) + Sheet drawer (mobile) + sticky top bar with page title, "Kembali ke Situs" button (calls `useRouter().navigate('beranda')`), admin avatar with initial.
  - Local `useState<AdminPageId>` for internal admin sub-navigation (no public router usage).
  - 13 nav items + logout button (POST `/api/auth/logout` + reload).
  - Active item: `bg-primary/10 text-primary` with left accent bar (`openly-gradient`).
  - Mobile bottom bar with quick logout for accessibility.
  - Content area scrolls independently (`overflow-y-auto openly-scroll`).
- Created `src/components/admin/pages/dashboard.tsx`:
  - GET `/api/admin/stats` → 10 stat cards (Total Pesanan, Menunggu Konfirmasi, Sedang Dikerjakan, Selesai, Dibatalkan, Pesanan Hari Ini, Layanan Desain, Layanan Website, Pelanggan, Pelanggan Kerja sama). Each card is clickable → navigates to relevant page via `onNavigate`.
  - recharts BarChart of last 7 days orders, primary fill `#1E63FF`.
  - Status queue summary card (Menunggu / Dikerjakan badges).
  - Recent orders table (clickable rows → orders page).
  - Skeletons while loading.
- Created `src/components/admin/pages/orders.tsx`:
  - GET `/api/admin/orders?status=&type=&q=` with debounced search + status Select + type Select + reset.
  - Table: orderNumber, queueNumber, customerName, phone, service+type, status badge, createdAt, Detail button.
  - Detail Dialog (desktop) / Sheet (mobile) showing: full order info card, WhatsApp pelanggan link (normalize 0→62), OrderDetailsView (renders GRAPHIC + WEBSITE fields nicely), StatusTimeline (compact + history list), status changer (Select + adminNote textarea + Simpan), attachments (read-only with "tersimpan aman di server" note), result files (download via `/api/files/[id]`).
  - Status PATCH + toast + refresh list + detail.
- Created `src/components/admin/pages/queue.tsx`:
  - GET `/api/admin/orders?status=QUEUED` + `?status=IN_PROGRESS` → queue board cards sorted by queueNumber.
  - Each card: big queueNumber (`openly-text-gradient`), orderNumber, customer, service, status badge, StatusTimeline compact, quick action button to advance status (Start→Approve→Complete) + skip.
  - TTS toggle button (uses `useUI().ttsEnabled` / `toggleTts`). On status change while TTS on, speaks Indonesian announcement via `window.speechSynthesis` ("Nomor antrean 012, pesanan sedang diproses.").
  - Status note: "TTS diaktifkan/dinonaktifkan".
  - Today's queue count summary.
- Created `src/components/admin/pages/services.tsx` (with `type: GRAPHIC | WEBSITE` prop):
  - GET `/api/admin/services`, filter client-side by type.
  - Service row with expand chevron (toggle packages inline), Switch active, Edit/Delete buttons, package count + priceLabel badges.
  - "Tambah Layanan" dialog form (name, description, priceLabel, priceFrom, estDuration, revision, options for WEBSITE only, sortOrder, active). POST `/api/admin/services`.
  - Edit Dialog pre-filled. PATCH. Delete via AlertDialog confirm.
  - PackageManager: inline list of service's packages with popular toggle, edit/delete. Add/Edit dialog (name, price, priceLabel, description, features textarea one-per-line, estDuration, revision, popular Switch, active Switch, sortOrder). POST/PATCH/DELETE `/api/admin/packages`.
- Created `src/components/admin/pages/packages.tsx`:
  - GET `/api/admin/services`, flatten all packages with serviceName + serviceType.
  - Filter by type (ALL/GRAPHIC/WEBSITE).
  - Table: name, service, price (formatIDR), priceLabel, popular (Switch inline), active badge, sortOrder, edit/delete.
  - Edit Dialog with full package form. Inline popular toggle. Delete via AlertDialog.
- Created `src/components/admin/pages/customers.tsx`:
  - GET `/api/admin/customers`, search input (name/email/phone).
  - Table: name (with company if any), email, phone, ordersCount, Switch active, createdAt, edit + WhatsApp buttons.
  - "Tambah Pelanggan" dialog (name, email, phone, password, company, agreement, active). POST.
  - Edit Dialog (email locked). PATCH. Toggle active inline.
- Created `src/components/admin/pages/cooperation.tsx`:
  - GET `/api/admin/customers`, filter to those with profile.
  - Table: name+company, contact (email+phone), agreement (clamped), ordersCount, activatedAt, Switch active, edit + WhatsApp.
  - "Undang Pelanggan Kerja sama" dialog with default agreement text + info banner explaining portal access.
- Created `src/components/admin/pages/files.tsx`:
  - Order picker: search by orderNumber/customerName, click to select.
  - Upload form (note + multi-file input with client-side validation: 15MB max, allowed ext jpg/png/webp/gif/pdf/zip/ai/psd/doc/docx). POST multipart `/api/admin/files`.
  - Files list: name, version badge, size, date, note, Unduh (link to `/api/files/[id]`), Hapus (DELETE with inline confirm dialog).
  - Attachments read-only list ("file tersimpan aman di server").
- Created `src/components/admin/pages/content.tsx`:
  - GET `/api/admin/content`. Section editors for: hero (slogan, description, ctas[], badges[]), advantages (array icon/title/desc with datalist of icon hints), services_summary_graphic/website (title, desc, points[], cta), order_flow (title, steps[]), cooperation, owner (name, role, welcome, vision, mission, values[]), faq (q/a array), contact (title, whatsapp, address, hours, desc).
  - Each section in Card with independent Save button (PATCH `/api/admin/content` with `{ key, value }` JSON). Toast on success.
  - Generic `ArrayEditor` with add/remove/move up-down; `StringListEditor` for badges/points; `JSONEditor` fallback for unknown sections.
  - Used React 19 recommended pattern (prevValue tracking + setState-during-render) instead of useEffect to avoid cascading render lint errors.
- Created `src/components/admin/pages/settings.tsx`:
  - GET `/api/admin/settings`. Grouped form (Identitas Merek, Hero/Beranda, Kontak, Media Sosial) covering brand_name, parent_company, owner_name, footer_text, hero_slogan, hero_description, announcement, nav_login_text, whatsapp, whatsapp_display, address, email_contact, instagram, tiktok, facebook, behance.
  - Sticky save-all card at bottom; POST `/api/admin/settings` with `{ settings }`. Toast on success.
  - Amber notice: "Pengaturan rahasia server (VAPID, JWT secret) TIDAK dapat diubah melalui editor ini — gunakan environment variables."
- Created `src/components/admin/pages/notifications.tsx`:
  - GET `/api/admin/notifications` (title, message, type, read, userName, orderNumber, createdAt).
  - "Kirim Notifikasi" dialog: title, message, recipient Select (broadcast or specific customer from `/api/admin/customers`), type Select (INFO/STATUS/SYSTEM). POST.
  - History table with type badge icons, recipient info, read status, time.
- Created `src/components/admin/pages/account.tsx`:
  - Account header card with avatar initial + email + admin badge.
  - Edit form: name (editable), email (locked), new password + confirm (min 6 chars, with validation). PATCH `/api/admin/account`.
- Created `src/app/api/admin/account/route.ts`:
  - GET: `requireAdmin()` + return admin user (id, email, name, role, phone, createdAt).
  - PATCH: `requireAdmin()` + accept `{ name?, password? }`. Hash new password with `hashPassword`. Returns updated user.
- Verified all 13 admin pages and shell integrated correctly. Ran `bun run lint` — fixed my 2 errors in content.tsx (setState-in-effect lint rule, replaced with React 19 prevValue-tracking pattern). Remaining 2 errors (`use-session.ts`, `lib/queue.ts`) are pre-existing foundation code from Task 1, untouched.
- Verified dev server compiles cleanly: `GET / 200 in 1576ms`. Admin endpoints return 401 for unauthenticated requests as expected.

Stage Summary:
- Deliverables completed:
  1. `src/components/admin/admin-view.tsx` — AdminView shell (auth guard + sidebar + top bar + mobile Sheet + internal page state).
  2. `src/components/admin/pages/*.tsx` — 12 page components (dashboard, orders, queue, services, packages, customers, cooperation, files, content, settings, notifications, account). ServicesPage handles both GRAPHIC & WEBSITE via `type` prop.
  3. `src/app/api/admin/account/route.ts` — admin self-update endpoint (GET + PATCH name/password).
- All pages use shadcn/ui components (Card, Table, Button, Input, Textarea, Label, Select, Badge, Dialog, Sheet, Skeleton, Switch, AlertDialog) and follow cobalt blue theme (`openly-gradient`, `openly-text-gradient`, `primary`).
- Tables wrapped in Card, horizontally scrollable, with sticky header on long lists (`max-h-[60-65vh] overflow-y-auto openly-scroll`).
- Forms in scrollable Dialog (`max-h-[70-88vh] overflow-y-auto openly-scroll`).
- Responsive: sidebar collapses to Sheet drawer on mobile; grids use `grid-cols-1 md:grid-cols-2 xl:grid-cols-3` patterns; mobile bottom logout bar for accessibility.
- TTS announcement works via `window.speechSynthesis` with `id-ID` locale fallback.
- WhatsApp link normalization (0→62 prefix) shared across pages.
- File uploads validated client-side (max 15MB; allowed extensions enforced).
- Content editor uses React 19 idiomatic setState-during-render pattern to avoid lint errors.
- Lint clean for all admin files (`npx eslint src/components/admin src/app/api/admin` exit 0).
- Pre-existing lint errors in `src/hooks/use-session.ts` and `src/lib/queue.ts` (foundation Task 1 code) not modified per instructions.
- Admin login: `admin@openly.id` / `admin123` (seeded). Navigate to admin view via Header "Admin" link or hash `#/admin`.
- No manual configuration needed beyond what was already set up in Task 1.

---
Task ID: 15-16
Agent: Main (orchestrator)
Task: Notifikasi (bell + toast + push + TTS), SEO files, self-verification end-to-end

Work Log:
- use-notifications hook: poll /api/notifications setiap 30s, deteksi notifikasi baru, toast otomatis untuk notifikasi baru (skip first load).
- NotificationBell component: popover dengan unread count badge, mark-all-read, list notifikasi (timeAgo). Di-embed di header untuk user login.
- Push notification system:
  - API: /api/push/vapid-public (GET, enabled flag), /api/push/subscribe, /api/push/unsubscribe.
  - Service worker public/sw.js (push event + notification click).
  - PushSubscribe component dengan graceful degradation: cek dukungan browser, cek VAPID config, izin permission, subscribe/unsubscribe. Pesan jelas saat push belum dikonfigurasi / diblokir / tidak didukung (termasuk catatan iOS terbatas). Di-embed di portal view.
- TTS antrean: toggle di admin QueuePage (useUI store), speechSynthesis Indonesian untuk pengumuman antrean saat status berubah.
- SEO: manifest.webmanifest, robots.txt, sitemap.ts, .env.example, layout metadata lengkap (OG, twitter, themeColor).
- Lint fix: use-session.ts, lib/queue.ts (require->import), use-notifications.ts, push-subscribe.tsx, portal.tsx (set-state-in-effect). Final: 0 errors, 0 warnings.

Self-verification (agent-browser):
- Home page render: title benar, hero "Crafted For Your Growth", semua nav (8 menu), CTA, advantages (6), services summary, order flow, cooperation, owner, FAQ, CTA WhatsApp. 0 console error.
- Jasa Desain: 6 service cards + package buttons. Pricelist: tabs (Desain Grafis/Pembuatan Website) + package cards + detail buttons.
- Order form GRAPHIC: semua field render (name, phone, service combobox, package, kebutuhan, deadline, file uploader, catatan). Submit flow end-to-end BERHASIL: isi form -> pilih "Desain Logo" -> konfirmasi modal -> submit -> confirmation page (OPL-20261024-7502, antrean #003, token). 0 console error.
- Admin login (admin@openly.id/admin123) -> dashboard render dengan sidebar 14 menu + stat cards (Total Pesanan, Layanan Desain 6, dll). 0 console error.
- Track page dengan token nyata: render order details + status timeline + history + WhatsApp link. 0 console error.
- Backend API verified via curl: services, packages, content, auth/login, auth/me, admin/stats (401 tanpa auth), admin/orders, orders/create (returns orderNumber+queue+token). Queue number incrementing (001, 002, 003).

Stage Summary:
- SELURUH sistem berfungsi end-to-end & terverifikasi via browser. Lint 0 error.
- Admin: admin@openly.id / admin123. Seed sudah dijalankan (services, packages, content, settings).
- Push notification: graceful degradation — tanpa VAPID env, tampilkan pesan "belum dikonfigurasi". In-app + toast tetap berfungsi.
- File storage lokal di storage/ (di luar public) dengan akses terautentikasi via /api/files/[id].
- Adaptasi dari requirement Supabase -> Prisma+SQLite (lingkungan hanya mendukung ini) + storage lokal terautentikasi menggantikan Supabase Storage.

---
Task ID: A13
Agent: admin-ui-updater (full-stack-developer)
Task: Update admin pages — support `niche` field, `BUNDLE` service type, and `testimonials` + `review_section` content sections

Work Log:
- Read prior worklog (Task 1, 9-13, 15-16) to understand foundation, admin UI shell, and conventions (cobalt blue theme, `openly-gradient`, shadcn/ui, React 19 setState-during-render pattern).
- Confirmed API contracts: `/api/admin/services` GET returns `niche`, POST/PATCH accept `niche` in body (empty string → null). `/api/admin/content` PATCH `{ key, value }`. `NICHES` array defined in `src/lib/types.ts` (7 niches: baby-kids, mom-care, klinik, salon, kuliner, toko, custom).

1. `src/components/admin/pages/services.tsx`
   - Added import: `NICHES, nicheInfo` from `@/lib/types` (already imported `ServiceType`, `ServiceDTO`, `PackageDTO`).
   - Added `niche: string` to `ServiceFormState` + `EMPTY_FORM` (default `""` = umum).
   - Added helper functions `typeLabel(type)` / `typeLabelShort(type)` returning proper Indonesian labels for GRAPHIC / WEBSITE / BUNDLE.
   - In `openEdit`: prefill `niche: s.niche || ""`.
   - In `submit`: include `niche: form.niche.trim() || null` in POST/PATCH body.
   - Added Niche Select field in service form dialog (after Opsi for WEBSITE, before Urutan/Aktif). Options: `"Umum (tanpa niche)"` (value `""`) + all `NICHES` mapped to `${emoji} ${short}` with value = niche id. Includes helper description text.
   - In `ServiceRow`: render niche as a Badge (using `nicheInfo(service.niche)`). If niche set: `border-primary/30 bg-primary/5 text-primary` with emoji+short; if not set: muted "Umum". Title attr shows niche desc.
   - Updated header/empty-state copy to use `typeLabel`/`typeLabelShort` so BUNDLE shows "Paket Bundle" properly.

2. `src/components/admin/pages/content.tsx`
   - Imported `Select, SelectContent, SelectItem, SelectTrigger, SelectValue` from shadcn + `Star, MessageSquareQuote` icons from lucide.
   - Added `testimonials` + `review_section` to `SECTION_LABELS` with title/desc.
   - Added both to `ORDER` array (placed after `owner`, before `faq`, before `contact`).
   - Added Editor switch cases routing to new editors.
   - New `TestimonialsEditor`: array of `{ name, role, text, rating }`. Card list with header (Testimoni #N + Hapus button), Name/Role grid, Testimoni Textarea, Rating Select (5..1, default 5), "Tambah Testimoni" button.
   - New `ReviewSectionEditor`: form for `{ title, desc, googleRating, googleCount, googleUrl }`. Judul Input, Deskripsi Textarea, Rating Google (number step 0.1, max 5), Jumlah Ulasan (number), URL Ulasan Google Input.
   - Both editors write through `onChange` so the parent `SectionEditor` `saveSection(key, value)` PATCHes `/api/admin/content` with `{ key, value }` and toasts success.
   - Reused existing React 19 `prevValue` pattern (no useEffect cascades introduced).

3. `src/components/admin/admin-view.tsx`
   - Added `"services-bundle"` to `AdminPageId` union (after `services-website`, before `packages`).
   - Added nav item `{ id: "services-bundle", label: "Paket Bundle", icon: Package }` (Package icon already imported) in same position.
   - Added `"services-bundle": "Paket Bundle"` to `PAGE_TITLES`.
   - Added `case "services-bundle": return <ServicesPage type="BUNDLE" />;` to `renderPage` switch.

4. Lint + dev verification
   - `bun run lint` → exit 0, no errors introduced.
   - Dev server log clean; `/api/content` and `/api/services?type=BUNDLE` routes return 200.

Stage Summary:
- 3 admin files updated (`services.tsx`, `content.tsx`, `admin-view.tsx`). No public views, header, footer, or API routes touched.
- Admin can now create/edit GRAPHIC / WEBSITE / BUNDLE services and tag each with a niche (or leave as Umum). Niche shows as a colored badge in the service list.
- Admin content page now has editors for `testimonials` (array of customer quotes with 1-5 star rating) and `review_section` (Google rating summary card).
- Sidebar has new "Paket Bundle" entry between Layanan Website and Paket & Pricelist.
- All styling consistent with existing cobalt blue theme (`openly-gradient`, `bg-primary/10`, `text-primary`, `border-primary/30`). Reused existing shadcn Select / Input / Textarea / Label / Button / Badge / Card components.

---
Task ID: A1-A14
Agent: Main (orchestrator) + subagent A13
Task: Perubahan besar v2 — admin login terpisah premium, login customer pendek, niche menu, harga website baru, bundle, testimoni, review, social media

Work Log:
- Schema: tambah field `niche String?` ke Service. db push sukses.
- Types: tambah `BUNDLE` ke ServiceType, NICHES array (7 niche: baby-kids, mom-care, klinik, salon, kuliner, toko, custom) + nicheInfo helper, ServiceDTO.niche.
- Router: tambah view `niche` dan `review` ke RouteView.
- API services: support filter `?type=&niche=` (niche=general => niche IS NULL). Return niche field.
- API admin services: POST/PATCH handle niche (empty string => null).
- Content API: tambah sections `testimonials` + `review_section`.
- Seed v2 rewrite: 6 general website services dengan harga baru (HTML 150rb, custom 250rb+online, lengkap 350rb, reservasi 500-850rb, jangka panjang 650rb-2jt, bisnis premium 1.5-3.5jt). MAX 3.5jt. 7 niche × (1 desain + 1 website + 1 bundle) = 21 niche services. Testimoni (6 entri). Social media: TikTok + Instagram asli. google_review setting.
- Header rebuild: dropdown "Layanan" (desktop, 7 niche) + collapsible "Layanan" (mobile). Menu "Review" baru. HAPUS tombol "Admin" dari header publik (hidden). Teks "Login" pendek.
- page.tsx: view `admin` dirender TANPA header/footer (terpisah penuh).
- AdminView AdminLoginCard: premium full-screen standalone (gradient bg, glassmorphism, animated logo via framer-motion, show/hide password). No public header/footer.
- LoginView: judul "Login" (bukan "Login Pelanggan Kerja sama"), hapus link "Login Admin".
- NicheView baru: hero niche + 3 section (Desain Grafis / Website / Paket Lengkap Desain+Website) + niche pills.
- ReviewView baru: Google review highlight card (link g.page/r/CU_Ah0Evi1nwEAI/review, rating 4.9, 50+ ulasan) + auto-scroll marquee row + alternating card carousel + social media (TikTok, IG, dll).
- TestimonialCarousel component: 2 variant — "row" (marquee auto-scroll CSS) + "card" (fade alternating with dots). Respect prefers-reduced-motion. Pause on hover.
- BerandaView: tambah section Testimoni di PALING BAWAH (marquee + card carousel + tombol "Lihat Semua Review").
- Subagent A13: admin services page (niche field + BUNDLE type), admin content page (testimonials + review_section editors), admin-view (menu "Paket Bundle" baru).

Self-verification (agent-browser):
- Beranda: nav "Layanan" dropdown + "Review", tombol "Login" pendek, TIDAK ada tombol Admin. Section testimoni di bawah dengan carousel. 0 error.
- Desktop dropdown "Layanan": semua 7 niche tampil, klik Baby & Kids -> navigasi ke niche view.
- Niche view: render 🍼 Baby & Kids dengan section Desain Grafis + Website + Paket Lengkap.
- Review view: Google review card + testimoni carousel + social media (TikTok/IG).
- Admin login premium: full-screen gradient, glass card, "Masuk" button. Setelah login -> dashboard dengan menu "Paket Bundle" baru.
- Mobile (375px): hamburger menu, collapsible "Layanan" menampilkan semua niche, tombol "Login" pendek.
- Harga website: min 150.000, max 3.500.000 (sesuai permintaan). Total 31 service (6 general website + 5 general graphic + 21 niche).
- Lint: 0 error, 0 warning.

Stage Summary:
- Admin login terpisah (hidden dari publik, akses via #/admin saja), UI premium glassmorphism.
- 7 niche menu (Baby & Kids, Mom Care, Klinik, Salon, Kuliner, Toko, Custom) dengan desain + website + bundle terpisah per niche.
- Harga website max 3.5jt, mulai 150rb HTML, 250rb custom+online, 350rb lengkap, 650rb jangka panjang, reservasi bervariasi.
- Bundle desain+web per niche (lebih hemat).
- Testimoni di beranda (bawah) + menu Review dengan Google review embed + social media TikTok & IG.
- Semua terverifikasi via agent-browser, 0 console error.

---
Task ID: C1-C5
Agent: Main
Task: Perbaikan total — hapus SEMUA harga lama, layout full-width desktop, beranda minimal, navigasi rapih label pendek

Work Log:
- Investigasi database: ternyata masih ada 12 layanan GRAPHIC lama (Desain Logo 250rb, Banner 150rb, Poster 175rb, Kartu Nama 125rb, + 7 niche graphic Desain Baby&Kids 275rb dll) dan 7 BUNDLE services (550rb-750rb) yang TIDAK dihapus oleh migrate script sebelumnya.
- Script clean-all-services.ts: hapus SEMUA services + packages. Rebuild hanya dengan: 4 kategori GRAPHIC dari poster (31 item, harga satuan persis) + 6 WEBSITE umum (HTML 150rb, Custom 250rb, Lengkap 350rb, Reservasi 500rb, Jangka Panjang 650rb, Bisnis Premium 1.5jt). BUNDLE = 0. Niche graphic + niche website dihapus total.
- Verifikasi database: GRAPHIC max price 1.35jt (Brand Identity), min 10rb (Daftar Harga). TIDAK ada lagi harga ratusan ribu acak untuk desain grafis. BUNDLE = 0.
- Header rebuild: label PENDAK & rapih. Menu utama: Layanan (dropdown), Beranda, Pricelist, Review, Tentang, Kontak, Login. Dropdown "Layanan" berisi: Desain Grafis, Website (label pendek, bukan "Jasa Desain Grafis"/"Jasa Pembuatan Website") + grid 7 kategori bisnis. Tombol "Akun" (bukan "Akun Saya") bila sudah login.
- Beranda SUPER minimal: HANYA hero section (slogan + desc + 3 CTA + 6 shortcut navigasi tombol). TIDAK ada section menumpuk (advantages, services cards, CTA whatsapp, testimoni semua dihapus dari beranda — accessible via nav menu). 0 h2 headings = tidak numpuk.
- Layout full-width desktop: ganti semua `max-w-7xl` (1280px) dengan `max-w-[1600px]` + padding `px-4 sm:px-6 lg:px-10` di seluruh view (footer, jasa-desain, jasa-website, pricelist, niche, cara-pemesanan, tentang, kontak, track, confirmation, review, service-explorer). Hilangkan space kanan-kiri berlebih di desktop besar.

Self-verification (agent-browser):
- Database: GRAPHIC 4 kategori (Design Sosial Media, Design Promosi, Design Branding, Design UMKM & Bisnis) dengan 31 item harga poster. BUNDLE = 0. Max graphic price 1.35jt (Brand Identity), min 10rb (Daftar Harga). TIDAK ada harga ratusan ribu acak lagi.
- Beranda (1440px desktop): hero only, 0 h2 headings (tidak numpuk), nav rapih (Layanan, Beranda, Pricelist, Review, Tentang, Kontak, Login), 6 shortcut tombol (Desain Grafis, Website, Pricelist, Review, Tentang, Kontak). Full-width container.
- Pricelist: 4 kategori poster rapih dengan item harga satuan (Feed IG 15rb, Logo Simple 25rb, Daftar Harga 10rb, dll).
- Jasa Desain: 4 kategori rapih (tidak ada lagi layanan lama seperti Desain Logo 250rb, Banner 150rb).
- Mobile (375px): nav drawer dengan menu pendek + shortcuts.
- Lint: 0 error, 0 warning. 0 console error.

Stage Summary:
- SEMUA harga lama dihapus. Hanya pricelist poster yang tersisa (4 kategori, 31 item, harga satuan persis).
- Layout full-width desktop (max-w-[1600px]), tidak ada space kanan-kiri berlebih.
- Beranda minimal (hero + shortcuts), tidak numpuk, semua detail via navigasi.
- Navigasi rapih: label pendek (Layanan/Beranda/Pricelist/Review/Tentang/Kontak/Login), dropdown "Layanan" dengan "Desain Grafis" & "Website" (bukan panjang).
