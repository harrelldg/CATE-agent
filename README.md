# CATE: Cathay Automated Travel Expert

A personal booking manager for Gold and Diamond members, on the Cathay phone line.

**The problem: personal service stops at the top tier.** What premium members value most is someone who knows them, so only the top tiers (Diamond Exec and above) get a Relationship Manager. Gold and Diamond members, the large group of high-value flyers just below, get the same app, forms and queue as everyone else. Every trip they repeat the same seat, meal, baggage and details, and anything beyond a simple booking means the hotline, a wait and explaining from the start. A member with a disability explains their needs again at every booking.

**CATE** (Cathay Automated Travel Expert) answers the call, already knows the member, and books and manages the trip in one call. It fills in name, passport, seat, meal and baggage from the profile and past trips, so nothing is asked twice, and it remembers needs such as a wheelchair as the right airport service code. Accessibility is part of personalisation: a member who needs help describes it once. CATE handles the routine; people handle the exceptions.

Built for **Cathay Hackathon 2026**, theme *Premium Travel*, by team **E332 Coelas Hackers** — a finalist team selected for the 24-hour Cathay Hackathon On-day.

> **Status: working prototype.** A real phone call is answered on the laptop, Amazon Nova 2 Sonic talks with the caller, and the booking, the trip page sent by email, the pay page and the booked confirmation run end to end on Cathay's real schedule (OAG). The [Status](#status) table below shows what exists.

## How a call works

The demo follows Mr Wong, a Gold member who flew to Osaka in May with a wheelchair to the gate:

1. **Verify and check the tier.** Mr Wong calls from his own phone. CATE matches the caller ID to his membership and, as the call starts, emails a six-digit one-time code to the address on file; he reads it out. CATE serves Gold members and above, and any member with a registered assistance need.
2. **Which service?** Book a flight, change a booking (dates, seat, meal, baggage), or a special request.
3. **Say the trip.** "Osaka again, after Christmas, back before New Year."
4. **Options.** CATE reads up to 3 options from the schedule, ranked by his profile (he never leaves before 9:00), and emails one link to his trip page, where he can tap the option he wants.
5. **Seat, meal, assistance, baggage, requests.** "Your membership has a window seat near the front and a vegetarian meal. Still right?" He can ask for seat 21A instead. "Wheelchair to the gate, like Osaka?" One extra bag. "It's my wife's birthday on the 26th": noted for the crew. Name and passport come from his profile; nothing is asked again.
6. **Read-back and hold.** CATE reads the whole trip back, with the total, and holds the booking only after he says yes.
7. **Pay.** His open trip page turns into the payment page (Visa, Mastercard, Apple Pay, FPS and others). He pays and sees "Payment successful".
8. **Booked.** The page goes back to his trip, now booked, and he gets an email with the booking reference and his needs card for airport staff.
9. **Change it later.** On a later call: "Move my Osaka trip to the 28th", "Make it an aisle seat", "Add a bag." CATE reads the change back and any difference in price opens on the same payment page.

## Core rule: the AI talks, our code decides

The voice model only listens, talks and chooses which tool to call. Every action that matters runs as a tool in our code, with its rule enforced there:

- **Names and passport** come from the verified member profile, never from speech, so the ticket matches the passport. The passport number never reaches the model.
- **Who gets CATE** (Gold and above, or a registered assistance need) is checked in code after verification.
- **Seats, meals, baggage and fees** come from fixed lists and the Orders system; a seat number must exist on our planes.
- **Flights and fares** come from our data. The model can't invent a flight or a price.
- **Wheelchair codes** come from a fixed rule table, not from the model's judgement.
- **Nothing about the member** reaches the model before verification passes. The model never sees the one-time code or the email address it goes to, and a code the model makes up is refused.
- **Consent:** a booking is held only after the caller has heard the read-back up to "Shall I hold this booking?" and said yes.
- **General Cathay answers** come only from checked passages, each with its source. With no relevant passage, CATE says it does not have that information and offers a member of staff.

The 16 tools, with their inputs, results and rules, are documented in [`docs/TOOLS.md`](docs/TOOLS.md). How the code fits together, file by file: [`docs/CODE.md`](docs/CODE.md).

## Architecture

```
Caller's phone ──► the phone line (prototype: FaceTime Audio on the demo Mac, see below)
                     │ call audio, through virtual audio cables
                     ▼
                  audio bridge on the laptop (the /bridge page)
                     │ WebSocket /local/stream: 8 kHz PCM16 both ways
                     ▼
                  CATE server (Node.js, on the same laptop)
                    ├─ 16 tools: send_code · verify_caller · apply_profile · find_flights · choose_option ·
                    │   set_preferences · keep_as_last_time · set_assistance · set_baggage · set_special_request ·
                    │   hold_booking · text_options · send_pay_link · find_booking · change_booking · ask_cathay
                    ├─ cathay.js ──► mock Cathay systems (Membership · Orders · Schedule)
                    ├─ trip pages (view-only links) · projector view
                    │
                    └─ two-way audio stream ──► Amazon Bedrock: Nova 2 Sonic (speech to speech)
```

**The phone line in this prototype.** CATE is built for an ordinary phone call: any phone, no app. At the hackathon we could not run a public phone number (no hosted server was allowed, and a telephony provider could not be set up in time), so for the demo the call reaches CATE through FaceTime Audio on a Mac: the call's audio goes through virtual audio cables (BlackHole) into the `/bridge` page, and a small script on the demo Mac answers the call and tells the bridge to start. Everything after the audio (the phone line `/local/stream`, Nova 2 Sonic, the tools and the Cathay systems) is the code a real phone number would use; in production, FaceTime is replaced by a telephony line and nothing else changes. Trip and pay links reach the caller's phone through a tunnel to the laptop.

**Mocked for the demo.** We have no access to Cathay's systems, so the server includes mock Membership, Orders and Schedule systems behind one adapter (`src/cathay.js`). Their operations are modelled on public IATA NDC message names (`OrderCreate`, `AirShopping`). In production, that adapter points at Cathay's membership and booking systems and nothing else changes. Flight schedules are a real Cathay schedule snapshot (OAG, November–December 2026); the demo flights in February–March 2027 are placeholders. Fares are illustrative. Payment is a demo pay page; no money moves.

## Status

| Part | State |
|---|---|
| Module contracts (`CONTRACTS.md`), configuration, event log, in-memory store, seed data | ✅ Done |
| Mock Cathay systems + adapter, the booking tools and their rules | ✅ Done, tested |
| Server: local phone line for the Bluetooth bridge, view-only pages, reset | ✅ Done, tested with real phone calls |
| Screens: projector view, trip page with needs card (choose a flight by button during the call), texts sent as email, booked email | ✅ Done, tested |
| Payment: the member pays himself on the demo pay page `/pay/<code>` (`PAYMENT=page`), which his trip page opens by itself once the booking is held; `POST /dev/pay` as a stage fallback | ✅ Done, tested |
| Scripted stand-in for the voice model (plays the golden path, no audio) | ✅ Done |
| General Cathay questions (`ask_cathay`): search over passages in `knowledge/`, source shown on the projector | ✅ Tool and keyword search done, tested. The 23 passages are drafts, not yet verified against Cathay's pages. Search by meaning (embeddings) is planned. |
| Voice model: Amazon Nova 2 Sonic over the call audio (`MODEL=nova`) | ✅ Done, run on real calls through Bedrock (Tokyo) |
| Live email, texts sent as email (`TEXTS=email`) | ✅ Done, tested |
| Real schedules (OAG snapshot): every Cathay route from Hong Kong, November–December 2026 | ✅ Done |
| Personal booking manager: tier gate, "which service?", seat by side, cabin area or number, meal choice, extra baggage, special requests, passport from the profile | ✅ Done, tested |
| Manage a booking: change dates, seat, meal or baggage; a higher price opens the payment page for the difference | ✅ Done, tested |
| Cancellations, upgrades, refunds of a lower fare | Next: handled by staff for now |
| Real card checkout (Stripe or Cathay's payment gateway) | Planned for production |

## Repo layout

```
CONTRACTS.md          interfaces between modules (read before writing code)
data/                 seed data for the mock Cathay systems (JSON)
docs/                 CODE.md (how the code works, file by file) · TOOLS.md (the 16 tools)
src/server.js         HTTP routes, projector feed, wiring
src/phone.js          the phone line: call audio from the bridge, life of a call, the one-time code at its start
src/tools.js          the 16 tools and their rules
src/knowledge.js      search over the knowledge passages (for ask_cathay)
knowledge/            general Cathay knowledge: short passages, each with source and date checked
src/cathay.js         the one adapter to Cathay's systems
src/mock-cathay/      mock Membership, Orders and Schedule systems
src/model/            voice model drivers (nova.js, scripted.js) and the system prompt (prompt.js)
src/screens/          projector view, trip page, texts, emails
src/payment.js        what happens when a booking is paid
src/messages.js       texts and email (log, or live by email)
src/config.js, events.js, store.js   settings, event bus + run log, in-memory state
test/                 node --test; golden.test.js plays the whole booking
```

## Run it

Requires Node.js 22 or later.

```sh
npm install
cp .env.example .env     # then fill in your own values
npm test                 # node --test
npm start                # node src/server.js; prints the projector address
```

To play the whole booking without a phone (`MODEL=scripted`), open the projector address in a browser, then:

```sh
curl -X POST -H "x-admin-key: <ADMIN_SECRET>" http://localhost:8080/dev/simulate          # the call
curl -X POST -H "x-admin-key: <ADMIN_SECRET>" http://localhost:8080/dev/pay               # the member pays (or pay on the page his trip link opens)
curl -X POST -H "x-admin-key: <ADMIN_SECRET>" http://localhost:8080/reset                 # start again
```

The trip page link is printed in the server console with the text message.

All settings are in `.env.example`. `MODEL=scripted` runs the golden path without a voice model, and `MESSAGES=log` prints texts and emails to the console instead of sending them.

For the live stage demo we used a shorter version of the prompt, so the call fits in about 90 seconds (the booking is checked on the trip page instead of read out); it is not part of this repository. This repository runs the full call, and the tools enforce the same rules either way.

## Privacy and security

- **No real personal data in git.** The demo caller's and family member's real phone numbers live only in `.env` (never committed); the seed data uses placeholders.
- **Minimum data to the model.** After verification the model gets name, preferences, needs and recent trips, never phone numbers, email addresses or the family member's contact details.
- **View-only links.** Trip pages open only from a 128-bit random link created during a call. They are read-only, never indexed, never cached, and show only name, flights and needs.
- **Local phone line.** Call audio is accepted only from this machine (not through a tunnel or proxy) and only with the admin secret. The projector view, reset and the `/dev` routes need the admin secret too.
- **One call at a time,** so a second caller can't change the demo's state.

## Licences

Direct dependencies: [`ws`](https://github.com/websockets/ws) (MIT) and the AWS SDK for JavaScript (`@aws-sdk/client-bedrock-runtime`, `@smithy/node-http-handler`; Apache-2.0). All 29 installed packages, including indirect ones, use permissive licences (Apache-2.0, MIT, 0BSD); none is copyleft. All code in this repo was written for this hackathon.
