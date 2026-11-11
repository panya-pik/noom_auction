# Draft: User Onboarding Flow (4 steps)

## Context
Noom Auction (Rails 8, importmap + Stimulus, SQLite, no models/auth yet) needs a first-time user onboarding that collects:
1. Thai national ID  2. Occupation  3. Annual income  4. Years of experience.

Follow the existing draft style (`AuctionsController` uses mock data, no DB): answers are kept in the **Rails session**, UI is a **single-page stepper** driven by a Stimulus controller, with server-side validation on final submit. Easy to swap to a model later.

## Routes — `config/routes.rb`
```ruby
resource :onboarding, only: [:show, :create], controller: "onboarding"
# GET /onboarding (form or summary), POST /onboarding (submit)
```
Plus `delete :reset` member (optional, to clear session for demo) — `resource :onboarding ... do delete :reset end`.

## Controller — `app/controllers/onboarding_controller.rb` (new)
- Constants (Thai labels, same style as `AuctionsController::STATUSES`):
  - `OCCUPATIONS` = `{ "employee"=>"พนักงานบริษัท", "government"=>"ข้าราชการ/รัฐวิสาหกิจ", "business_owner"=>"เจ้าของธุรกิจ", "freelance"=>"ฟรีแลนซ์", "student"=>"นักเรียน/นักศึกษา", "other"=>"อื่นๆ" }`
  - `INCOME_RANGES` = `< 150,000`, `150,001–300,000`, `300,001–500,000`, `500,001–1,000,000`, `> 1,000,000` (บาท/ปี)
  - `EXPERIENCE_RANGES` = `0-1`, `1-3`, `3-5`, `5-10`, `10+` ปี
- `show`: `@data = session[:onboarding] || {}`; `@completed = @data["completed"]`; `@errors = {}`.
- `create`: permit `:national_id, :occupation, :occupation_other, :annual_income, :experience`; strip non-digits from ID; validate; on error → `render :show, status: :unprocessable_entity` with `@errors` and `@step` = first step with error (so the stepper opens there). On success → `session[:onboarding] = data.merge("completed"=>true)`, redirect to `onboarding_path` with flash notice.
- `reset`: clear session key, redirect.
- Private `valid_thai_id?(id)`: 13 digits; checksum = `(11 - (Σ digit[i]*(13-i) for i in 0..11) % 11) % 10 == digit[12]`.
- Store only a masked ID for display (`1-2345-xxxxx-xx-3`) helper; note in a comment that real storage needs encryption (PDPA).

## Param filtering — `config/initializers/filter_parameter_logging.rb`
Add `:national_id` so it isn't written to logs.

## View — `app/views/onboarding/show.html.erb` (new)
- Same head/font block as `auctions/index.html.erb` (Kanit).
- If `@completed`: summary card (masked ID, occupation, income, experience) + "ไปหน้ารายการประมูล" (`auctions_path`) + "แก้ไขข้อมูล" (reset).
- Else: `form_with url: onboarding_path, data: { controller: "onboarding", onboarding_start_value: @step || 0 }`
  - Progress header: 4 numbered dots + labels (บัตรประชาชน · อาชีพ · รายได้ · ประสบการณ์), `data-onboarding-target="dot"`.
  - 4 `<section data-onboarding-target="step">` panels, all rendered visible; JS hides the inactive ones on connect, so the form still works without JS:
    1. `national_id` text input, `inputmode="numeric"`, `maxlength=17`, placeholder `x-xxxx-xxxxx-xx-x`, auto-formats with dashes as you type.
    2. Occupation as radio cards; "อื่นๆ" reveals `occupation_other` text input.
    3. Annual income radio cards.
    4. Years of experience radio cards.
  - Server errors render under each field (`@errors[:field]`).
  - Footer buttons: ย้อนกลับ (`btn--ghost`, hidden on step 1), ถัดไป (`btn--primary`, steps 1–3), ยืนยัน (submit, step 4).

## Stimulus — `app/javascript/controllers/onboarding_controller.js` (new)
Auto-registered by `eagerLoadControllersFrom` in `controllers/index.js`.
- `static targets = ["step", "dot", "back", "next", "submit", "nationalId", "otherField"]`, `static values = { start: Number }`.
- `connect()` → `show(startValue)`.
- `next()` → validate current step client-side; if OK `show(i+1)`. `back()` → `show(i-1)`.
- `show(i)`: toggle `hidden` on steps, `is-active`/`is-done` on dots, button visibility, focus first input.
- `formatId()` on input: keep digits, format `1-2345-67890-12-3`, live checksum hint.
- Step validation: step 1 = same checksum algorithm in JS; steps 2–4 = a radio checked (and other-text filled if "other"). Show inline error text.
- Enter key on steps 1–3 → `next` instead of submitting.

## Styles — `app/assets/stylesheets/onboarding.css` (new)
Reuse `:root` tokens from `home.css` and `.btn*` from `auctions.css` (stylesheet_link_tag :app loads all). New: `.onboard` centered card (max-width 560px), `.stepper` dots (gold when active, teal when done), `.choice` radio cards (grid, checked = gold border), `.field__error` (rose), mobile-friendly.

## Entry points
- Link "เริ่มต้นใช้งาน" on `home/index.html.erb` and the profile menu in `auctions/index.html.erb` → `onboarding_path`.

## Test — `test/controllers/onboarding_controller_test.rb` (new)
- GET shows the form; POST valid data (valid checksum ID e.g. computed in test) → redirect + summary shows masked ID; POST invalid ID → 422 with error; invalid occupation → 422.

## Verification
1. `bin/rails test test/controllers/onboarding_controller_test.rb`
2. `bin/dev` → open `/onboarding`: step through, try a bad ID (blocked client-side), Back/Next, submit, see summary; disable JS → all panels still post and server errors show.
3. `bin/rubocop app/controllers/onboarding_controller.rb`
