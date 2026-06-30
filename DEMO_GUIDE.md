# CPTOS Frontend — ডেমো গাইড (Project Manager Reference)

> **দর্শক:** Project Manager (PM) এবং Stakeholder দল  
> **তারিখ:** ২০২৬ — বর্তমান development stage অনুযায়ী  
> **ভাষা-নীতি:** বাংলা গদ্যে লেখা; UI label, route, code, library নাম ইংরেজিতে রাখা হয়েছে।

---

## ১. পরিচিতি (Overview)

**CPTOS (Chittagong Port Terminal Operating System)** হলো চট্টগ্রাম বন্দরের টার্মিনাল অপারেশন পরিচালনার জন্য একটি আধুনিক web-based SPA (Single Page Application), যা React + TypeScript দিয়ে তৈরি। বর্তমানে **Authentication** এবং **Role Management** মডিউল দুটি সম্পূর্ণরূপে নির্মিত এবং live backend-এর সাথে সংযুক্ত — অর্থাৎ এই দুটি screen সম্পূর্ণ production-ready। বাকি **৬টি business module** (Vessel & Berthing, Pre-arrival & Discharge, Equipment Control, Intermodal Yard, Yard Planning, Gate Operations) ইতিমধ্যে কোডবেসে scaffold করা আছে এবং "Coming Soon" placeholder হিসেবে দেখায় — সেগুলোর backend microservice প্রস্তুত হওয়ামাত্র একটি করে সংযুক্ত করা হবে।

---

## ২. ডেমোর আগে প্রস্তুতি (Preparation)

ডেমো শুরুর আগে নিচের বিষয়গুলো নিশ্চিত করুন:

- **Backend চালু রাখুন।** Backend microservices আলাদা repository-তে আছে এবং `http://localhost:8080` এ চলমান থাকতে হবে। Frontend backend ছাড়া login করতে পারবে না।
- **Frontend চালু করুন।** Project root-এ টার্মিনাল খুলে চালান:
  ```bash
  npm run dev
  ```
  এরপর browser-এ যান: **http://localhost:5174** (port fixed, পরিবর্তন হবে না)।
- **ডেমোর আগে backend-এ দুটো account তৈরি করুন:**
  1. **Full System Administrator** — সব permission আছে (সব কিছু দেখাতে পারবেন)।
  2. **Limited account** — শুধু User Management-এ VIEW permission (security demo-র জন্য প্রয়োজন)।
- > **⚠️ গুরুত্বপূর্ণ নোট:** `nafis` / `hasib` account দুটো শুধু **offline mock mode**-এ কাজ করে। এই live demo-তে এগুলো ব্যবহার করবেন **না** — backend account ব্যবহার করুন।

---

## ৩. ধাপে ধাপে ডেমো ওয়াকথ্রু (Step-by-step Walkthrough)

### ধাপ ১ — Login

| কী করবেন | কী বলবেন / হাইলাইট করবেন |
|---|---|
| Browser-এ `http://localhost:5174` খুলুন। | Login screen দেখাবে — React Hook Form + Zod দিয়ে তৈরি, real-time validation আছে। |
| ইচ্ছাকৃতভাবে ভুল password দিন। | Field-level error message তাৎক্ষণিক দেখায়, form submit হয় না। |
| সঠিক full-admin credentials দিন, Submit করুন। | Authentication live backend-এ হচ্ছে — mock নয়। Session localStorage-এ store হয় এবং সব browser tab share করে। |
| লগইনের পর screen পরিবর্তন লক্ষ্য করুন। | Backend থেকে user-এর permission অনুযায়ী sidebar তৈরি হয় এবং সঠিক landing page-এ redirect হয়। |

> **বোনাস:** যদি session token expire হয়ে যায়, পরের বার login page-এ একটি "session expired" banner দেখাবে — এটি automatic।

---

### ধাপ ২ — Dashboard (`/dashboard`)

| কী করবেন | কী বলবেন / হাইলাইট করবেন |
|---|---|
| Dashboard খুলুন। | Greeting + আজকের তারিখ দেখায়। |
| Role-context card দেখুন। | User-এর role(s) দেখায়; একাধিক role থাকলে "access combines all N roles" বার্তা দেখায় — permission যোগ হয়, বিয়োগ নয়। |
| Module cards দেখুন। | শুধুমাত্র user-এর granted module-গুলো card হিসেবে দেখায়; প্রতিটি card-এ "Full access" বা "Limited access" badge এবং "G/T permissions · pages" সারসংক্ষেপ আছে। |
| System Administrator account-এ SYSTEM_ADMIN card দেখুন। | এই card-টি expand হয়ে live stat strip দেখায়: Users: Total / Active / Pending / Suspended + কতটি Role defined — এগুলো backend থেকে আসছে। |

---

### ধাপ ৩ — Sidebar (Left Navigation)

| কী করবেন | কী বলবেন / হাইলাইট করবেন |
|---|---|
| Sidebar-এর বিভিন্ন nav item দেখুন। | Menu সম্পূর্ণ **permission-driven** — backend থেকে আসা permission অনুযায়ী তৈরি; user যে module দেখতে পারেন না, সেটি sidebar-এ দেখায়ই না। |
| Search box-এ কিছু টাইপ করুন। | Sidebar-এর search box nav item filter করে। |
| Collapse toggle ক্লিক করুন। | Sidebar collapse/expand হয় — narrow screen-এ জায়গা বাঁচায়। |
| যেকোনো nav link-এ **right-click → "Open in new tab"** করুন। | এটি semantic `<a>` tag ব্যবহার করায় browser-এর নেটিভ behavior কাজ করে — Ctrl+click, middle-click, সব কিছু। অনেক React app-এ এটি কাজ করে না; এখানে ইচ্ছাকৃতভাবে সঠিকভাবে করা হয়েছে। |
| Logo-তে click করুন। | Home/landing page-এ ফিরে যায়। |

---

### ধাপ ৪ — Users Page (`/role-management/users`)

| কী করবেন | কী বলবেন / হাইলাইট করবেন |
|---|---|
| Users page খুলুন। | Paginated, sortable table। Column header-এ click করলে sort হয়। |
| Search box-এ কিছু টাইপ করুন। | 300ms debounce সহ live search — প্রতি keystroke-এ request যায় না, performance-friendly। |
| Department / Role / Status filter ব্যবহার করুন। | Filter combination কাজ করে। |
| **Add User** button ক্লিক করুন। | একটি drawer slide in করে। Username field লক্ষ্য করুন — Full Name টাইপ করলে username স্বয়ংক্রিয়ভাবে derive হয়। Password strength meter দেখান। কিছু না save করে বন্ধ করার চেষ্টা করুন — "discard changes?" confirm dialog দেখাবে। |
| যেকোনো row-এ ⋮ menu খুলুন। | Edit User, View Profile অপশন দেখায়। |
| **Export** button ক্লিক করুন। | PDF export হয় — `@react-pdf/renderer` lazy-load হয়, প্রথমবার একটু সময় নেয়, পরে instant। |
| Permission-based button visibility দেখান। | Add / Edit / Delete / Export button গুলো USER_MANAGEMENT permission অনুযায়ী দেখায় বা লুকায়। |

---

### ধাপ ৫ — Roles Page (`/role-management/roles`)

| কী করবেন | কী বলবেন / হাইলাইট করবেন |
|---|---|
| Roles page খুলুন। | বাঁ দিকে role switcher — প্রতিটি role-এ member count দেখায়। |
| একটি role select করুন। | Hero stats দেখায়: কতটি permission granted, কতজন member। |
| **Permissions tab** খুলুন। | Permission matrix দেখান: module tabs, প্রতিটি page-এর জন্য ৬টি action (view / create / update / delete / export / approve)। |
| Column header-এ tri-state toggle দেখান। | "All granted / some / none" — একটি ক্লিকে পুরো column toggle হয়। |
| Preset control দেখান। | "Read only", "Full access" ইত্যাদি preset একটি ক্লিকে পুরো role-এ apply হয়। |
| Module tab switch করুন। | প্রতিটি tab একটি backend module-এর permission দেখায়। |
| **Summary tab** দেখান। | Role-এর সব permission-এর সারসংক্ষেপ এক জায়গায়। |
| **Create role** (from scratch এবং duplicate) দেখান। | Existing role duplicate করে নতুন role তৈরি করা যায় — permission inherit করে, তারপর customize। |
| Rename এবং Delete (two-step) দেখান। | Delete-এ দুটো confirm step আছে — ভুলবশত delete হওয়া রোধ করতে। |
| **Members tab** খুলুন। | যে users-দের primary role এটি তারা এখানে দেখায়; ⋮ menu থেকে View Profile / Edit User সরাসরি যাওয়া যায়। Search কাজ করে। |
| **System role** select করুন (যেমন, System Administrator)। | Read-only info message দেখায় — system role edit করা যায় না। |

---

### ধাপ ৬ — Security Demo (Limited Account দিয়ে)

> এই ধাপটি সবচেয়ে গুরুত্বপূর্ণ stakeholder talking point — "permission enforcement"।

| কী করবেন | কী বলবেন / হাইলাইট করবেন |
|---|---|
| Logout করুন। Full-admin account থেকে বের হোন। | Logout একই সাথে সব browser tab logout করে — দেখান। |
| Limited account দিয়ে login করুন (শুধু VIEW permission)। | Sidebar ভিন্ন দেখাবে — permitted module-এ কম item। |
| Users page খুলুন। | Add / Edit / Delete / Export button নেই — hidden, কারণ permission নেই। User শুধু দেখতে পারছে। |
| Roles page খুলুন এবং Edit করার চেষ্টা করুন। | Permission matrix read-only mode-এ। Edit করার চেষ্টা করলে লাল toast দেখায়: **"You do not have permission to update roles."** |
| বলুন: | "Frontend permission enforce করছে। Backend-ও independently enforce করছে — double layer। UI button লুকালেও কেউ direct API call করলেও backend reject করবে।" |

---

### ধাপ ৭ — Profile Page (`/profile`)

| কী করবেন | কী বলবেন / হাইলাইট করবেন |
|---|---|
| Profile page খুলুন। | "Personal info" tab: avatar, full name (read-only — admin manage করে), username (read-only), editable email ও phone, read-only designation ও department। |
| Email বা phone edit করুন, Save করুন। | Save/Discard দেখায় — unsaved changes থাকলে Discard চাইলে confirm চায়। |

---

### ধাপ ৮ — Cross-tab Session Demo

| কী করবেন | কী বলবেন / হাইলাইট করবেন |
|---|---|
| Full-admin দিয়ে logged in থাকা অবস্থায় নতুন browser tab খুলুন এবং `http://localhost:5174` paste করুন। | নতুন tab সরাসরি Dashboard দেখাচ্ছে — আবার login করতে হয়নি। Session shared। |
| একটি tab-এ Logout করুন। | অন্য সব open tab স্বয়ংক্রিয়ভাবে login page-এ redirect হয়। একটি logout = সব জায়গায় logout। |

---

## ৪. আর্কিটেকচার ও বিজনেস ভ্যালু (Architecture & Business Value)

- **Microservice-aligned modules:** প্রতিটি backend microservice-এর জন্য `src/modules/` এর ভেতরে আলাদা folder আছে। নতুন service যোগ হলে নতুন folder — বাকি সব কোড অপরিবর্তিত থাকে। Risk isolated।
- **Mock-first strategy (one-line swap):** Backend তৈরির আগেই পুরো UI typed mock data দিয়ে build করা হয়েছে — তাই এত কাজ শেষ। Mock থেকে live backend-এ switch করতে শুধু `services/<module>Service.ts` ফাইলের একটি re-export line পরিবর্তন হয়। UI, hook, component — কিছু ছোঁয়া লাগে না।
- **Permission-driven security (double layer):** Sidebar/menu সম্পূর্ণ server-granted permission থেকে তৈরি। প্রতিটি UI action `hasPagePermission` দিয়ে gate করা। Backend independently প্রতিটি request validate করে। দুটো layer থাকায় কেউ browser console থেকে bypass করলেও backend reject করবে।
- **Production-grade stack:** React 19.2, TypeScript strict mode, TanStack Query v5 (server state), Tailwind CSS v4, Zod 4 validation, code-splitting — enterprise-level quality।
- **Responsive:** Desktop-এ full layout, tablet-এ adjusted, mobile-এ compact — breakpoint দিয়ে সব screen size handle।
- **Accessible:** Semantic HTML `<a>` link, ARIA roles, keyboard navigation সহ — right-click / middle-click / Ctrl+click সব কাজ করে।
- **Performance:** Server-side pagination, 300ms debounced search, lazy-loaded PDF renderer, route-level code-splitting।
- **Reporting:** User list PDF export built-in (`@react-pdf/renderer`) — ভবিষ্যতে অন্য module-এও extend করা যাবে।

---

## ৫. ডেমো ডেটা চিট-শিট (Demo Data Cheat-Sheet)

ডেমোর আগে এই table fill করুন এবং সাথে রাখুন:

| অ্যাকাউন্ট | পারমিশন লেভেল | ডেমোতে কী দেখায় |
|---|---|---|
| `<full-admin account>` | System Administrator — সব permission | পুরো sidebar, সব feature, SYSTEM_ADMIN stat strip, edit/create/delete সব কাজ করে |
| `<limited account>` | শুধু User Management VIEW | Restricted sidebar; Users page read-only; Roles page-এ "no permission" toast; Add/Edit/Delete/Export button hidden |

> **📝 ফুটনোট:** `nafis` ও `hasib` account শুধুমাত্র **offline mock mode**-এ কাজ করে (`authService.ts`-এ mock re-export active থাকলে)। এই live demo-তে backend account ব্যবহার করুন — mock account দিয়ে login হবে না।

---

## ৬. সম্ভাব্য প্রশ্ন ও উত্তর (Anticipated PM Q&A)

**প্র ১: Backend তৈরি হওয়ার আগেই এত UI কীভাবে সম্ভব হলো?**
> উত্তর: "Mock-first" strategy — UI টা backend-এর জন্য অপেক্ষা না করে typed mock data দিয়ে পুরোপুরি build করা হয়েছে। Mock এবং real API function-এর signature হুবহু এক রাখা হয়েছে; service layer-এর একটি line বদলালেই live হয়ে যায়। এ কারণে এত কাজ এগিয়ে আছে।

**প্র ২: বাকি ৬টি module কোথায়? Sidebar-এ তো দেখছি না।**
> উত্তর: প্রতিটি module `src/modules/` এ আলাদা folder হিসেবে scaffolded আছে। এখন "Coming Soon" placeholder দেখাচ্ছে। Sidebar permission-driven — যে module-এর permission নেই বা যেটা placeholder, সেটা দেখায় না। Backend microservice ready হলে সেই module-এর `services/` file-এর একটি line বদলে দিলেই live হয়ে যাবে।

**প্র ৩: Permission এবং security কীভাবে কাজ করে?**
> উত্তর: Login করলে backend থেকে user-এর permission map আসে, localStorage-এ store হয়। Sidebar সেই permission দিয়ে তৈরি হয়। প্রতিটি UI action (button, edit mode) `hasPagePermission` function দিয়ে check করে। Backend independently প্রতিটি API request validate করে। Double enforcement।

**প্র ৪: কেউ কি browser console ব্যবহার করে permission bypass করতে পারবে?**
> উত্তর: না। Frontend-এর permission check UI convenience-এর জন্য — button লুকানো, read-only mode। কিন্তু backend সব API request independently reject করে। Console থেকে localStorage edit করলেও backend request fail করবে। Security backend-এ enforce।

**প্র ৫: একজন user-এর একাধিক role থাকলে কী হয়?**
> উত্তর: সব role-এর permission-এর union কাজ করে — মানে যে কোনো role-এ যদি একটি permission থাকে, user সেটি পাবে। Dashboard-এ "access combines all N roles" বার্তা দেখায়। এটি backend থেকে নির্ধারিত।

**প্র ৬: Live backend-এ পুরো integrate করতে কত কাজ বাকি?**
> উত্তর: Auth এবং Role Management ইতিমধ্যে live। বাকি প্রতিটি module-এর জন্য কাজ: backend API ready হলে সেই module-এর `services/<module>Service.ts` file-এ mock re-export সরিয়ে real API export দিলেই হয়। UI, hook, component — কিছু পরিবর্তন লাগে না। Mock এবং real function-এর contract identical রাখা হয়েছে।

**প্র ৭: Performance কেমন? অনেক user থাকলে সমস্যা হবে না?**
> উত্তর: Server-side pagination — একবারে সব data load হয় না। Search 300ms debounce করা — প্রতি keystroke-এ request যায় না। PDF renderer lazy-load। Route-level code-splitting দিয়ে initial bundle ছোট। TanStack Query v5 দিয়ে smart caching এবং background refetch।

**প্র ৮: Mobile বা tablet-এ কাজ করবে?**
> উত্তর: হ্যাঁ। Responsive design — desktop-এ full sidebar layout, tablet-এ adjusted, mobile-এ compact। Login screen-ও তিনটি breakpoint handle করে।

**প্র ৯: মোট কত শতাংশ কাজ শেষ?**
> উত্তর: Frontend-এর কাজ ধরলে — Auth + Role Management (Users, Roles, Profile, Dashboard, Sidebar) সম্পূর্ণ, live backend-connected। ৬টি business module scaffold করা, placeholder দেখাচ্ছে। Backend microservice ready হওয়ার গতির উপর বাকি কাজের timeline নির্ভর করছে।

**প্র ১০: ডেমোতে যে data দেখছি সেটা কি আসল?**
> উত্তর: হ্যাঁ — Auth এবং Role Management-এর সব data live backend থেকে আসছে। কোনো hardcoded mock নয়। Real user, real role, real permission।

**প্র ১১: Accessibility কেমন? Screen reader বা keyboard-only user-রা ব্যবহার করতে পারবে?**
> উত্তর: Semantic HTML ব্যবহার করা হয়েছে — nav link সব `<a>` tag, ARIA role আছে, keyboard navigation কাজ করে। Sidebar-এর nav item right-click / Ctrl+click / middle-click সব browser-native behavior কাজ করে। পুরো accessibility audit ভবিষ্যতে করা হবে।

**প্র ১২: Report বা export feature আছে?**
> উত্তর: এখন Users page-এ PDF export আছে — `@react-pdf/renderer` দিয়ে। Download button click করলে formatted PDF generate হয়। ভবিষ্যতে অন্য module-এও একই pattern দিয়ে extend করা সহজ।

**প্র ১৩: পরবর্তী ধাপ কী?**
> উত্তর: যত backend microservice ready হবে, তত module live হবে — Vessel & Berthing, Pre-arrival & Discharge, Equipment Control, ইত্যাদি। প্রতিটির জন্য frontend code প্রায় ready; শুধু service layer-এর swap এবং backend-নির্দিষ্ট type adjustment লাগবে।

**প্র ১৪: Code quality কেমন? Technical debt আছে?**
> উত্তর: TypeScript strict mode — `any` type ব্যবহার নেই। ESLint configured। Module boundary enforced — একটি module অন্যটির internal file import করতে পারে না। প্রতিটি module স্বাধীন, তাই একটিতে bug fix অন্যটিকে affect করে না। Code-split এবং lazy loading। পুরো codebase এই একটি architectural pattern follow করে।

**প্র ১৫: Session security কেমন? কেউ কি অন্যের session access করতে পারবে?**
> উত্তর: Auth HttpOnly cookie + localStorage permissions combination ব্যবহার করছে। Session সব tab share করে — যেটা port terminal-এর multi-monitor workflow-এ সুবিধাজনক। Logout একটি tab থেকে করলেই BroadcastChannel দিয়ে অন্য সব tab logout হয়। Token expire হলে "session expired" banner দেখায় এবং login page-এ redirect হয়।

---

## ৭. উপসংহার (Closing)

CPTOS frontend-এর এই demo-তে দেখানো হয়েছে যে — backend-এর জন্য অপেক্ষা না করেও একটি production-grade, secure, permission-driven UI সম্পূর্ণভাবে তৈরি করা সম্ভব এবং করা হয়েছে। Microservice-aligned architecture এবং mock-first strategy-র কারণে বাকি module-গুলো backend ready হওয়ামাত্র দ্রুত এবং নিরাপদে live করা যাবে — পুরো frontend পুনরায় তৈরির প্রয়োজন নেই।
